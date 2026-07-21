# Contexto: Stock histórico y consumo mensual de insumos (SAP S4)

## Objetivo
Armar un histórico mensual (foto al 1° de cada mes, desde septiembre 2024) de stock y consumo de materiales/insumos por Centro, a partir de SAP S4, sin depender de Excel/AFO manual.

## Fuente de datos

**Tabla Athena:** `molinos_datalake_datapreparation.sap_pp_docordpro`
- Acceso confirmado con credenciales SSO de la cuenta datalake (`296062593156`).
- Rango disponible: **2024-09-16 → presente** (~22 meses).
- Nota: los primeros 15 días de septiembre 2024 no están en la tabla — el stock de oct-2024 puede tener un desvío menor por ese período.

**Campos clave:**
| Campo | Descripción |
|---|---|
| `sku` | Material |
| `planta` | Centro |
| `fecha_contabilizacion` | Fecha del movimiento |
| `codigo_tipo_movimiento_mercancia` | Tipo de movimiento SAP |
| `cantidad_produccion_umb` | Cantidad (ya viene con signo: + entrada, − salida) |
| `unidad_base_material` | Unidad de medida |

## Arquitectura de la solución

### Stock al 1° de cada mes
**Lógica:** stock inicial (CSV externo al 2024-09-01) + movimientos reales acumulados mes a mes.

```
stock[mes N+1] = stock[mes N] + SUM(movimientos reales del mes N)
```

**Movimientos incluidos** (flujos reales de entrada/salida):

| Código | Descripción | Efecto |
|---|---|---|
| 101 / 102 | EM compras / anulación | + / − |
| 105 / 106 | EM/DM desde stock bloqueado | + / − |
| 122 / 123 | Devolución a proveedor / anulación | − / + |
| 161 / 162 | EM devolución / anulación devolución | + / − |
| **261** | **Consumo para orden de producción** *(a agregar)* | **−** |
| 501 / 502 | Entrada/salida sin pedido | + / − |
| 521 / 522 | Entrada/salida sin orden de fabricación | + / − |
| 531 / 532 | Entrada/salida subproducto | + / − |
| 545 / 546 | Entrada/salida subproducto subcontratación | + / − |
| 561 / 562 | Entrada/salida inicial de stocks | + / − |
| 631 / 632 | Stock en consignación | + / − |
| 651–658 | Devoluciones varios | + / − |

**Movimientos excluidos** (traslados y reclasificaciones internas — solo tienen una punta registrada en la tabla, inflarían el saldo):
- 311 / 312: traslado en centro
- 301 / 302: traslado centro a centro
- 321 / 322 / 323 / 325 / 326: calidad ↔ libre
- 343 / 344 / 349 / 350: bloqueado ↔ libre
- 411 / 412: almacén a almacén
- 641 / 642 / 671 / 672: tránsito
- Z27 / Z28: movimientos custom

### Consumo mensual
Filtro exclusivo por **movimiento 261**, con `ABS(SUM(cantidad_produccion_umb))` para obtener valores positivos.

### Output
Archivo `stock_consumo_mensual.csv` con columnas:
`material, centro, mes, stock_inicio_mes, unidad, cantidad_consumo`

## Pendientes para correr el notebook

1. **`stock_inicial_20240901.csv`** — Federico lo genera desde AFO/BW con columnas: `material, centro, cantidad_stock, unidad`. Colocarlo en la carpeta raíz del repo.
2. **Movimiento 261** — el equipo de datos de Molinos lo tiene que agregar a `sap_pp_docordpro` en Athena.

## Servicios OData relevantes (investigados, acceso pendiente)

| Servicio | Datos | Estado |
|---|---|---|
| `ZZ1_BI_MM_STOCK_Q_CDS_0001` | Stock mensual por Material/Centro | 403 — incidente levantado a Basis |
| `ZZ1_BI_PP_DOCORDPRO_Q_CDS_0001` | Consumo (mov. 261) por Material/Centro | 403 — idem |

Endpoint base: `https://fiori.mrp.com.ar/sap/opu/odata/sap/<servicio>/<entity_set>`

Se priorizó la vía Athena por acceso inmediato. Los OData quedan como opción alternativa/validación.

## Credenciales y entorno

- **SAP:** usuario `rollandf`, contraseña en `.env`. Requiere VPN/red de Molinos.
- **AWS:** cuenta datalake `296062593156`, credenciales temporales SSO en `.env` (se vencen — renovar desde consola AWS → nombre de usuario → *Command line or programmatic access*).
- **Output Athena:** `s3://molinos-datalake-prod-us-east-1-296062593156-athena/`
- **Stack:** Python 3.14, `boto3`, `pandas`, `python-dotenv`.

## Contexto general del entorno (Federico / Molinos)
- Rol: datos y analytics, foco en demand planning, forecast, supply chain.
- Stack habitual: SAP IBP (OData), AWS (Athena, SageMaker, Lambda), Python/pandas, Jupyter con `dotenv_values()`.
- Cuenta AWS de IBP (`399583244247`) ≠ cuenta datalake (`296062593156`) — usar credenciales de la cuenta datalake para acceder a Athena.
