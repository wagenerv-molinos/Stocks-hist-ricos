# Contexto: Reconstrucción de stock histórico de insumos (SAP S4)

## Objetivo
Armar un histórico mensual (foto de fin de mes, últimos 2 años) de stock de
materiales/insumos por Centro, a partir de SAP S4, sin depender de Excel/AFO
manual. La extracción debe hacerse en una sola corrida de script (no múltiples
consultas/archivos manuales).

## Fuente de datos descubierta
Se investigó primero un query de BW en AFO (Analysis for Office) que traía
las medidas: **Cantidad de stock**, **Cantidad de consumo**, **Ctd.increm.stock**,
**Cantidad sal.stock**, agrupadas por Centro / Material stock / Fecha
contabilización. Sirvió para confirmar qué datos existen y su granularidad,
pero no es la vía de extracción final (Excel se traba con el volumen: muchos
centros x muchos insumos x 24 fechas).

Se encontró un **servicio OData** que expone el mismo tipo de dato de forma
programática:

- **Metadata:** `https://fiori.mrp.com.ar/sap/opu/odata/sap/ZZ1_BI_MM_STOCK_Q_CDS/$metadata`
- **Entity set a consultar:** `ZZ1_BI_MM_STOCK_Q` (OJO: no usar las entidades
  `*Results` ni las de "Master Data", esas son solo para listas de valores/F4)
- **Autenticación:** Basic Auth (usuario/contraseña de servicio SAP), devuelve
  401 sin credenciales — requiere probarse desde la red/VPN de Molinos.

### Campos relevantes del CDS view
| Campo OData | Significado |
|---|---|
| `Material` | Material stock |
| `Plant` | Centro |
| `MatlDocLatestPostgDate` | Fecha de contabilización (string, formato a confirmar — se está probando `'YYYYMMDD'`, ej `'20260630'`) |
| `MatlWrhsStkQtyInMatlBaseUnit` | **Cantidad de stock** (única medida — `sap:aggregation-role="measure"`) |
| `MaterialBaseUnit` | Unidad de medida base |
| `ProductType` | Tipo de producto (dimensión, NO filtrar por esto — se descartó filtrar por `ProductType eq '5000'`) |
| `InventoryStockType` | Tipo de stock |
| `Batch`, `Supplier`, `StorageLocation` | Disponibles pero no usados por ahora |

### ⚠️ Limitación importante — pendiente de resolver
Este CDS view (`ZZ1_BI_MM_STOCK_Q_CDS`) **solo trae `Cantidad de stock`**, no
consumo. En AFO sí existía la medida "Cantidad de consumo" pero no se encontró
todavía el CDS/OData equivalente que la exponga. Posibles caminos a explorar
en Claude Code:
1. Buscar otro CDS view expuesto (patrón de nombre similar,
   `ZZ1_BI_MM_CONSUMPTION...` o similar) — pedir a Basis/IT el catálogo de
   servicios OData disponibles, o revisar el catálogo de servicios en
   `/sap/opu/odata/sap/` (transacción `/IWFND/MAINT_SERVICE` del lado SAP).
2. Reconstruir el consumo mes a mes por diferencia de `cantidad_stock` entre
   fechas consecutivas, ajustando por entradas conocidas si hace falta.

## Alternativas evaluadas y descartadas (para no repetirlas)
- **Datalake Athena (`molinos_datalake_datapreparation`):** se sugirió
  revisar si existían tablas `sap_mm_*` replicadas (patrón `sap_sd_*` que ya
  usa el pipeline de IBP), pero no se llegó a verificar en esta conversación.
  Vale la pena chequear antes de asumir que el OData es la única vía.
- **Open Hub Destination (BW):** opción "nativa" de BW para extraer cubos
  completos sin pasar por Excel, pero requiere que Basis/IT lo configure —
  no se avanzó por ahora, se priorizó el OData porque ya hay precedente de
  uso (pipeline IBP `EXTRACT_ODATA_SRV`).

## Script ya armado
Se armó `extract_stock_historico.py` (Python) con esta lógica:
- Genera automáticamente las 24 fechas de fin de mes de los últimos 2 años.
- Función `test_una_fecha(fecha)` para validar formato de fecha y volumen
  antes de correr todo (**pendiente de ejecutar y confirmar** — no se llegó
  a validar el `status_code` ni el formato real de `MatlDocLatestPostgDate`
  en esta conversación).
- Función `extraer_fecha(fecha)` pagina con `$skip`/`$top` (páginas de 5000
  filas) por si el servidor tiene límite de resultados por request.
- Función `extraer_historico(fechas)` recorre las 24 fechas y consolida todo
  en un único DataFrame → un único CSV de salida
  (`stock_historico_mensual.csv`), columnas: `material, centro, fecha,
  cantidad_stock, unidad`.
- Requiere `.env` con `SAP_USER` / `SAP_PASSWORD`, y el paquete
  `python-dotenv` (`pip install python-dotenv` — es el nombre correcto del
  paquete, no `dotenv` a secas).

**Último error visto:** `ModuleNotFoundError: No module named 'dotenv'` — se
resolvió indicando instalar `python-dotenv`. No se confirmó si después de
instalar el script corrió bien ni qué devolvió el test de una fecha.

## Próximos pasos sugeridos para Claude Code
1. Instalar dependencias y correr `test_una_fecha()` contra el endpoint real
   (desde la red de Molinos/VPN) para confirmar formato de fecha y volumen
   de filas por día.
2. Si el filtro de fecha con `'YYYYMMDD'` falla, probar
   `datetime'2026-06-30T00:00:00'`.
3. Correr la extracción completa (24 fechas) y validar el CSV resultante
   (sin fechas duplicadas, sin huecos, totales por centro razonables).
4. Buscar el CDS/OData de consumo, o decidir reconstruirlo por diferencia de
   stock entre meses.
5. Evaluar si conviene además chequear tablas `sap_mm_*` en el datalake
   Athena (`molinos_datalake_datapreparation`) como fuente alternativa/cruce
   de validación.
6. Integrar la extracción al repositorio de GitHub correspondiente (mismo
   patrón que `ibp-forecast-centro-boca`: carga de credenciales con
   `dotenv_values()`, estructura de proyecto similar).

## Contexto general del entorno (Federico / Molinos)
- Rol: datos y analytics, foco en demand planning, forecast, supply chain.
- Stack habitual: SAP IBP (OData `EXTRACT_ODATA_SRV`), AWS (Athena,
  SageMaker, Lambda), Python/pandas, notebooks Jupyter con `dotenv_values()`
  para credenciales.
- Ya tiene precedente de consumir OData de SAP para IBP en el pipeline
  `ibp-forecast-centro-boca`; mismo patrón aplica acá para MM/stock.
- Prefiere código completo y listo para pegar, lenguaje directo, y trabaja de
  forma iterativa corrigiendo sobre la marcha.
