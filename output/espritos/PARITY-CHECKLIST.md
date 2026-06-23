# EspritOS — Parity Checklist vs CRM-ERP v4

> **Propósito:** Antes de apagar el CRM-ERP v4 y declarar a EspritOS como sistema oficial, **cada uno de estos checkboxes debe estar marcado**. Este checklist está derivado de las 27 fases GSD del v4 (ver `context/phases/`).
>
> **Cómo usar:** Conforme completes cada ola del Build Order del blueprint, regresa a este archivo y marca `[x]`. Cada item referencia el VERIFICATION.md de la fase correspondiente del v4 — lee esos docs para entender el criterio exacto.
>
> **Regla:** Si un item no aplica (porque Beto decidió no necesitarlo en EspritOS), márcalo como `[~]` y documenta la razón en una línea debajo.

---

## Fase 00 — Deuda técnica y hardening
Ref: `context/phases/00-deuda-tecnica-y-hardening/`

- [ ] Tests ejecutándose con pytest (no vitest/jest — Python)
- [ ] Linter configurado (ruff, no ESLint)
- [ ] Type checking (opcional con mypy, Django usa duck typing)
- [ ] CI/CD corre tests en cada PR (GitHub Actions + Coolify)
- [ ] Pre-commit hooks instalados (ruff, djlint, secrets scanner)
- [ ] `test_app_isolation.py` pasa (bloquea cross-imports entre apps)

## Fase 01 — Migración datos catálogo + ETL Punto Zero
Ref: `context/phases/01-migracion-datos-catalogo/`, **Blueprint Apéndice E**, `context/revision-servidor/`

> **Este es el Paso 6 del Build Order.** La estrategia completa está en el Apéndice E del Blueprint. Las referencias probadas están en `context/revision-servidor/` (ETL que ya migró Abarrotera exitosamente). No reinventes nada — copia patrones y adapta.

### Infraestructura del ETL

- [x] **Checklist del servidor completado** ✅ (2026-04-09, Beto + Bernardo):
  - [x] Versión de MySQL: **5.1.48** documentada
  - [x] Puerto 3306 escuchando en `0.0.0.0` y alcanzable desde `192.168.0.152`
  - [x] Usuario `espritos_reader@'192.168.%'` creado con `GRANT SELECT` en `datos1` y `datos9`
  - [x] Sin firewall bloqueando — `bind-address` no definido = acepta toda la LAN
  - [x] Tamaños documentados: datos1 = 4.5 GB / 343 tablas / 2,788 productos / 17.4M filas
  - [x] datos9: 93 MB / 334 tablas / 277K filas
  - [x] Ventana 9pm-7am confirmada libre para ETL (10 horas)
- [x] **Decisión C1 vs C2 tomada**: **Plan C1** — conexión LAN directa al servidor
- [x] **Conexión validada físicamente** desde `192.168.0.152` a ambas bases con cliente MySQL 8.4 ✅
- [ ] **Variables de entorno configuradas** en `.env` del repo `espritos/`:
  - [ ] `ETL_SOURCE=puntozero_mysql_direct`
  - [ ] `PUNTOZERO_HOST=192.168.0.200`
  - [ ] `PUNTOZERO_USER=espritos_reader`
  - [ ] `PUNTOZERO_PASSWORD=EspritReader2025!` (ver Blueprint E.14.7)
  - [ ] `PUNTOZERO_CHARSET=latin1` — **CRÍTICO**
  - [ ] `PUNTOZERO_DB_CREMERIA=datos1`
  - [ ] `PUNTOZERO_DB_ABARROTERA=datos9`
- [ ] **Apps/etl scaffold aplicado** desde `bootstrap/apps/etl/`
- [ ] **Migraciones de ETL ejecutadas**: `etl_watermarks`, `etl_entity_mappings`, `etl_run_logs`
- [ ] **Mappings de RevisionServidor importados**: `python manage.py etl_import_mappings`
- [ ] **Pre-flight check ejecutado**: `python manage.py etl_preflight` → status `READY`

### Transformers (tests críticos deben pasar)

- [ ] `test_encoding.py` — latin1 → utf8 correcto, nombres con ñ y acentos
- [ ] `test_fechas.py` — `0000-00-00` → NULL, timezones correctos
- [ ] `test_rfc.py` — RFCs válidos SAT, genéricos aceptados, inválidos rechazados
- [ ] `test_folios.py` — parseo de folios legacy de PZ

### Extractors (40 tablas, ver Apéndice E.6 del Blueprint)

**Catálogos base:**
- [ ] `productos` — ~2,724 productos de `datos1` con IEPS, SAT code, costo promedio
  - [ ] Códigos de barras alternos (`clavesalternas`) vinculados
  - [ ] Marcas (`marcas`) fusionadas correctamente
  - [ ] Líneas/categorías (`lineas`) mapeadas a Category level=2
- [ ] `clientes` — ~405 clientes con RFC normalizado, límite de crédito, canal
  - [ ] Asignación vendedor↔cliente respetada
  - [ ] Canales de distribución (`clasificactes`) → Channel
  - [ ] Wallet balance inicial = 0 (histórico no existe en PZ)
- [ ] `proveedores` — ~126 proveedores con días de crédito
- [ ] `vendedores` — 8 vendedores creados como User + group `vendedor`

**Transacciones de venta:**
- [ ] `tickets + ticketsdetalle` — ~9,026 ventas POS históricas → `Sale` + `SaleItem`
  - [ ] `priceOrigin` registrado desde PZ cuando existe
  - [ ] Métodos de pago mapeados (efectivo/tarjeta/transferencia)
- [ ] `facturas + facturasdetalle` — 148 facturas con XML timbrado
- [ ] `facturasglobales` — ~24,982 facturas globales históricas
- [ ] `remisiones + remisionesdetalle` → `DeliveryNote`
- [ ] `cotizaciones + cotizacionesdetalle` → `Quotation` (puede estar vacío)
- [ ] `devolucionpdv + devolucionpdvdet` → `SaleReturn`
- [ ] `notascredito + notascreditodetalle` → `Cfdi` tipo E + `SaleReturn` vinculados
- [ ] `notascreditoglobales` vinculadas a facturas globales correspondientes

**Compras y finanzas:**
- [ ] `compras + comprasdetalle` → `PurchaseOrder` + items
- [ ] `entradas + entradasdetalle` → `GoodsReceipt`
- [ ] `cxp` + `edoctaprov` → `AccountsPayable` con pagos
- [ ] `anexopagosprov` → `PayablePayment`
- [ ] `cxc` + `edoctacli` → `AccountsReceivable` con pagos
- [ ] `anexopagosctes` → `ReceivablePayment`
- [ ] `gastos` → `OperationalExpense`
- [ ] `comisiones` → `CommissionResult` (puede estar vacío en PZ)

**Inventario:**
- [ ] `historicoalmacen` → `StockLedgerEntry` (~36,291 filas, el más voluminoso)
  - [ ] Append-only respetado
  - [ ] Batch insert optimizado (COPY, no INSERT one-by-one)
  - [ ] `valuation_rate` calculado correctamente
- [ ] `traspasos + traspasosdetalle` → `StockTransfer`
- [ ] `ajustes` → `StockAdjustment` con razones (MERMA, ROTURA, CADUCIDAD, etc.)
- [ ] `kits + kitskardex + entradaskit + movtoskit` → `Bundle` + movimientos de inventario

**CFDI recibidos y fiscal:**
- [ ] `xmlcompras` → `SupplierCfdi` con parser XML
- [ ] `xmls` → importados como referencia de timbrado
- [ ] `certificados` → `FiscalConfig`

**Cortes y auditoría:**
- [ ] `cortes + cortesdetalle` → `CashCut` con denominación
- [ ] `bitacorausuarios` → `AuditLog` histórico (~38,450 entradas) marcado como `source='punto_zero_legacy'`

**Precios:**
- [ ] `cambiosdeprecios` → `PriceChangeLog` (~5,167 cambios históricos)

### Sistema de watermarks y deltas

- [ ] Tabla `etl_watermarks` existe y tiene entrada por cada extractor
- [ ] Re-correr `etl_incremental` 5 veces consecutivas produce 0 filas afectadas (idempotencia)
- [ ] Buffer de 2 días aplicado correctamente (correcciones retroactivas)
- [ ] `etl_reset_watermark` funciona para reprocesar período específico

### Celery Beat

- [ ] Task `etl_incremental_nocturno` programada cada noche a las 2am
- [ ] Task `backup_postgres_to_b2` programada 30min antes del ETL (1:30am)
- [ ] Retry con backoff: 5min, 15min, 45min
- [ ] Circuit breaker activa después de 3 fallos consecutivos
- [ ] `python manage.py etl_resume` reactiva después de circuit breaker

### Verificación cruzada

- [ ] `python manage.py etl_verify` corre sin errores
- [ ] Diferencia entre PZ y EspritOS < 0.1% por cada tabla crítica
- [ ] Tres preguntas spot-check al chat IA dan respuestas correctas:
  - [ ] *"¿Cuántos productos activos tenemos?"* → coincide con `SELECT COUNT(*) FROM productos WHERE Activo=1` en PZ
  - [ ] *"¿Cuánta Salchicha San Antonio vendimos en marzo 2026?"* → coincide con suma manual
  - [ ] *"¿Cuál es el top 5 de clientes del último trimestre?"* → coincide con reporte manual de Beto

### Rollback probado

- [ ] `pg_dump` pre-ETL guardado en Backblaze B2 diariamente
- [ ] Restauración desde B2 probada al menos 1 vez
- [ ] Documentado el procedimiento en `docs/runbook.md`

---

## OLA A — Catálogo base (Build Order Pasos 14-17)

### Fase 02 — Precios con volumen (base del sistema)
Ref: `context/phases/02-precios-volumen-descuentos-pos/02-VERIFICATION.md`

- [ ] Cascada de precios 4 niveles: pactado → lista cliente → lista default → costo × factor
- [ ] Precios por volumen: al agregar cantidad ≥ minQty, precio cambia automáticamente al tier correcto
- [ ] VolumeTierBadge equivalente: indicador visual del tier activo
- [ ] Precio `origin` rastreable en cada venta (ej: `"pactado:CLI-0042"`, `"lista:Mayoreo"`)
- [ ] Descuento por línea (% o monto fijo) con preview
- [ ] Descuento global del documento distribuido proporcionalmente
- [ ] Límites de descuento por rol:
  - CAJERO: 5%
  - VENDEDOR: 10%
  - GERENTE/ADMIN: 100%
  - ALMACENISTA: 0%
- [ ] Validación server-side de descuentos (double check, no solo frontend)

### Fase 15 — Listas de precios (UI)
Ref: `context/phases/15-listas-de-precios/`

- [ ] CRUD de listas (Público, Tendero, Mayoreo, Rutero)
- [ ] Editor de precios por item con minQty
- [ ] Historial de cambios de precio (`PriceChangeLog` append-only)
- [ ] Precios programados con fecha efectiva futura (lazy materialization)
- [ ] Fórmula de margen por categoría × lista (matriz PuntoZero)
- [ ] Recalculador masivo con preview antes de aplicar

### Fase 17 — Ficha de producto
Ref: `context/phases/17-ficha-de-producto/`

- [ ] Vista 360° del producto: precios en todas las listas, stock por sucursal, ventas históricas, proveedores
- [ ] Gráfica de ventas del producto en el tiempo
- [ ] Top clientes del producto
- [ ] Margen promedio calculado
- [ ] Múltiples códigos de barras (EAN13, UPC, INTERNAL, SCALE) editables

### Fase 18 — Clientes mejorado
Ref: `context/phases/18-clientes-mejorado/`

- [ ] Tags de cliente (VIP, Moroso, Frecuente) con colores
- [ ] Notas cronológicas por tipo (COMERCIAL/OPERATIVO/COBRANZA/GENERAL)
- [ ] Campos enriquecidos: visitDay, preferredPayment, keyProducts, walletBalance
- [ ] Ficha 360° del cliente: ventas históricas, saldo CxC, devoluciones, comisiones generadas
- [ ] Asignación vendedor→cliente editable por admin

---

## OLA B — POS + Hardware (Build Order Pasos 18-22)

### Fase 02/21 — POS base + Entradas/salidas de caja
Ref: `context/phases/02-precios-volumen-descuentos-pos/` y `context/phases/21-entradas-salidas-caja/`

- [ ] Página POS fullscreen (sin sidebar ni topbar normal)
- [ ] Abrir turno con fondo inicial
- [ ] Búsqueda de productos por código de barras o nombre
- [ ] Carrito con items, cantidades editables, descuentos por línea
- [ ] Selección de cliente (o venta público general)
- [ ] Métodos de pago: CASH, CARD, TRANSFER, MIXED
- [ ] Cálculo de cambio en pago efectivo
- [ ] Impresión de ticket en impresora térmica
- [ ] Venta a crédito (isCredit=true) genera CxC automática
- [ ] Movimientos de caja intraturno (pay_in / pay_out con motivos)
- [ ] Cierre de turno con arqueo:
  - Conteo por denominación (JSON `{"500":3,"200":5}`)
  - Declarado vs sistema por método de pago
  - Diferencia calculada automáticamente
- [ ] **Atomicidad**: crear venta actualiza Sale + SaleItems + InventoryBin + StockLedger en una transacción
- [ ] Notas de venta abiertas (reopenable draft sales)
- [ ] Keyboard shortcuts (F2 producto, F3 descuento, F4 cliente, F12 pagar, etc.)

### Fase 03 — Hardware POS
Ref: `context/phases/03-hardware-pos/`

- [ ] Impresora ESC/POS térmica 80mm funcionando (test físico)
- [ ] Apertura del cajón de efectivo vía impresora
- [ ] Lector de código de barras USB HID (emulación de teclado)
- [ ] Báscula serial: productos con barcode type=SCALE reciben peso automáticamente
- [ ] Logo de Cremería HM impreso en el ticket
- [ ] Hardware configurable por sucursal

### Fase 05 — Inventario avanzado
Ref: `context/phases/05-inventario-avanzado/`

- [ ] `InventoryBin` por producto × almacén con qty, reserved, valuation_rate
- [ ] `StockLedgerEntry` append-only con posting date, qty change, valuation rate
- [ ] Costo promedio ponderado (WAC) recalculado en cada recepción
- [ ] Transferencias entre almacenes (CREM ↔ ABAR) con confirmación
- [ ] Ajustes de inventario con razones: MERMA, ROTURA, CADUCIDAD, CONTEO_FISICO, DONACION
- [ ] Kardex por producto/almacén con paginación

### Fase 24 — Restock automático
Ref: `context/phases/24-restock-automatico/`

- [ ] Cálculo de sugerencias de reposición basado en ventas históricas + reorderPoint
- [ ] Generación automática de PO borradores agrupados por proveedor
- [ ] Alertas de productos bajo mínimo

---

## OLA C — Facturación y cuentas (Build Order Pasos 23-27)

### Fase 04 — CFDI 4.0 base
Ref: `context/phases/04-cfdi-40-base/`

- [ ] `FiscalConfig` con RFC, régimen, certificados CSD (.cer/.key) y password
- [ ] Integración con Facturama (sandbox y producción)
- [ ] Timbrar factura individual desde Sale
- [ ] Descargar XML timbrado y PDF legal
- [ ] Cancelar CFDI con motivos 01/02/03/04
- [ ] Status flow: PENDIENTE → TIMBRADO → CANCELADO / ERROR
- [ ] Email del CFDI al cliente
- [ ] Manejo de errores del PAC (timeouts, rechazos del SAT)

### Fase 07 — CxC, devoluciones, remisiones
Ref: `context/phases/07-cxc-devoluciones-remisiones/`

- [ ] `AccountsReceivable` auto-creada al confirmar venta a crédito
- [ ] Pagos parciales con tracking de balance
- [ ] Aging por cliente: corriente, 1-15, 16-30, 31-60, 60+ días
- [ ] Vista de cuentas vencidas con priorización
- [ ] `SaleReturn` con razón y refund method (CASH/WALLET/NC_CFDI)
- [ ] Devolución actualiza inventario (append a ledger)
- [ ] Notas de crédito CFDI al devolver con refundMethod=NC_CFDI

### Fase 16 — Remisiones dedicado
Ref: `context/phases/16-remisiones-dedicado/`

- [ ] CRUD de remisiones independiente de ventas POS
- [ ] Plantillas de remisión por cliente ("Pedido semanal Restaurante X")
- [ ] Materializar plantilla → nueva remisión con items pre-llenados
- [ ] Convertir remisión → factura CFDI
- [ ] Facturar a público en general (invoiceToPublic flag)

### Fase 08 — CFDI avanzado
Ref: `context/phases/08-cfdi-avanzado/`

- [ ] Factura global (diaria/mensual de ventas público)
- [ ] Complementos de pago (tipo P)
- [ ] Factura standalone (sin venta vinculada, requiere justificación)
- [ ] Sustitución con uuidSustitucion (motivo 01)
- [ ] Relaciones entre CFDIs (DoctoRelacionado)
- [ ] Re-timbrado si el primer intento falló

### Fase 19 — Hub de facturación
Ref: `context/phases/19-hub-de-facturacion/`

- [ ] Panel único donde Beto/Montserrat ven: por timbrar, timbradas hoy, errores, cancelaciones pendientes
- [ ] Filtros por status, cliente, rango de fechas
- [ ] Acciones bulk: timbrar varias a la vez

### Fase 22 — CFDI operativo
Ref: `context/phases/22-cfdi-operativo/`

- [ ] Mejoras basadas en uso real post-Fase 08
- [ ] Dashboards de facturación (timbre/mes, errores recurrentes, costos Facturama)

---

## OLA D — Compras, comisiones, gastos (Build Order Pasos 28-30)

### Fase 06 — Compras y proveedores
Ref: `context/phases/06-compras-y-proveedores/`

- [ ] CRUD de proveedores con RFC, días de crédito, descuento por pronto pago
- [ ] Asignación primario/secundario por producto (ProductSupplier priority)
- [ ] PO: borrador → confirmado → completado / cancelado
- [ ] Recepción de mercancía parcial (qtyOrdered vs qtyReceived)
- [ ] Recepción actualiza InventoryBin + recalcula WAC en StockLedgerEntry
- [ ] `AccountsPayable` auto-generada al recibir
- [ ] Pagos con descuento por pronto pago aplicado
- [ ] Importación de CFDIs recibidos de proveedores (parser XML)

### Fase 13 — Gastos operativos
Ref: `context/phases/13-gastos-operativos/`

- [ ] CRUD de gastos con categorías, métodos de pago, tags (JSON)
- [ ] Plantillas recurrentes (mensual, semanal) con `nextDueDate`
- [ ] Materializar plantilla → nuevo gasto automáticamente
- [ ] Presupuesto por categoría × mes
- [ ] Vista de ejecución vs presupuesto
- [ ] Vincular gasto opcionalmente a `AccountsPayable`

### Fase 14 — Comisiones de ventas
Ref: `context/phases/14-comisiones-de-ventas/14-01-SUMMARY.md` (motor), `14-02/03/04-SUMMARY.md` (UI/cálculo/CSV)

- [ ] 3 modos de cálculo: progresivo, plano, fijo
- [ ] Tiers con lowerLimit/upperLimit + rate (upper null = infinito)
- [ ] Metas mensuales por vendedor: baseSales, targetMin, targetExpected
- [ ] Familias no comisionables (HUEVO, MARGARINA DELICIA) con patrones regex
- [ ] Pesos estacionales por mes (suman ~100)
- [ ] Metas por producto específico (Croquetas 45 costales)
- [ ] Cálculo mensual congelado en `CommissionResult` con breakdown JSON
- [ ] Re-cálculo manual (override) con audit trail
- [ ] Importación de CSVs legacy de PuntoZero como fallback (6 formatos soportados)
- [ ] Dashboard por vendedor: resultado del mes, histórico, concentración top 3 clientes
- [ ] Reporte PDF mensual con firma/logo HM

---

## OLA E — CRM, promos, lealtad (Build Order Pasos 31-34)

### Fase 09 — CRM mayorista
Ref: `context/phases/09-crm-mayorista/`

- [ ] CRUD de prospectos con pipeline NUEVO → CONTACTADO → COTIZADO → NEGOCIACION → CLIENTE/PERDIDO
- [ ] Actividades: LLAMADA, VISITA, EMAIL, WHATSAPP, COMPROMISO_PAGO
- [ ] Calendario de nextAction por vendedor
- [ ] Cotizaciones con vigencia (validDays, expiresAt)
- [ ] Conversión prospect → customer (crea Customer, mantiene historial)
- [ ] Conversión quotation → sale / deliveryNote
- [ ] Reporte de pipeline de ventas (embudo de conversión)
- [ ] Alertas de seguimiento vencido

### Fase 11 — Promociones y bundles
Ref: `context/phases/11-promociones-y-bundles/`

- [ ] CRUD de promociones con tipos: NxM, SPECIAL_PRICE, ACCUMULATED_DISCOUNT
- [ ] Targets: producto específico o categoría completa
- [ ] Prioridad en caso de conflicto entre promos
- [ ] Vigencia con startDate/endDate (endDate null = indefinida)
- [ ] Motor aplica promociones al carrito POS automáticamente
- [ ] Bundles: FIXED_PRICE (precio total), COMBO_DISCOUNT (% al comprar todos), KIT (SKU propio con stock)
- [ ] KIT puede tener inventario propio o descontar de componentes
- [ ] Visualización en POS: "ahorro aplicado: $X"

### Fase 25 — Programa de lealtad
Ref: `context/phases/25-programa-de-lealtad/`

- [ ] `LoyaltyConfig` con accrualRate (2% por default), minPurchaseAmount
- [ ] Al confirmar venta: generar `LoyaltyTransaction` ACCRUAL, actualizar customer.walletBalance
- [ ] Al hacer devolución: REVERSAL del accrual correspondiente
- [ ] Redención en POS: usar walletBalance como método de pago (REDEMPTION)
- [ ] Historial de transacciones por cliente
- [ ] Ajustes manuales (ADJUSTMENT) con razón

---

## OLA F — Dashboard, reportes, móvil (Build Order Pasos 35-37)

### Fase 10 — Reportes y analítica
Ref: `context/phases/10-reportes-y-analitica/`

- [ ] Reporte de ventas (por vendedor, cliente, producto, periodo)
- [ ] Reporte de utilidad (con costo promedio al momento de venta)
- [ ] Reporte de inventario (stock actual + valuación)
- [ ] Reporte de antigüedad de saldos
- [ ] Exports CSV y PDF
- [ ] Filtros guardables por usuario

### Fase 23 — Dashboard y costos
Ref: `context/phases/23-dashboard-y-costos/`

- [ ] Dashboard diferenciado por rol:
  - Vendedor: sus ventas del día/mes, sus comisiones, sus top clientes
  - Supervisor: equipo completo + chat IA
  - Cobranza: aging + prioridades del día
  - Admin: todo + KPIs globales
- [ ] Gráficas con ECharts (ventas por día, por producto, por vendedor)
- [ ] Auto-refresh cada N segundos (configurable)
- [ ] Widgets drag-and-drop (opcional)

### Fase 12 — PWA offline (Rutero)
Ref: `context/phases/12-pwa-offline/`

- [ ] PWA instalable en celular/tablet Android
- [ ] Catálogo offline (IndexedDB) con precios del día
- [ ] Carrito funcional sin internet
- [ ] Sync al reconectar: queue de pedidos pendientes → server
- [ ] Resolución de conflictos (precios cambiaron, stock insuficiente)
- [ ] UI grande, táctil, legible bajo el sol (contraste alto)
- [ ] Búsqueda por voz (opcional)
- [ ] Escáner de QR/barcode con cámara del teléfono

---

## OLA G — Cierre y migración (Build Order Pasos 38-40)

### Fase 20 — Permisos granulares
Ref: `context/phases/20-permisos-granulares-tech-debt/`

- [ ] `RolePermission` matriz: role × module × (can_view, can_create, can_edit, can_delete)
- [ ] Panel `/admin/permisos/` para editar matriz
- [ ] Enforcement en todas las vistas (decorator/middleware)
- [ ] Tests E2E verificando que vendedor no puede acceder a rutas admin por URL directa

### Migración de datos
- [ ] Scripts en `apps/etl/management/commands/migrate_from_crm_erp_v4.py`
- [ ] Migración read-only primero (verificar counts y sumas)
- [ ] Reconciliación de folios (no duplicar secuencias)
- [ ] Validación de integridad referencial post-migración
- [ ] Backup pre-migración verificado
- [ ] Rollback plan documentado

### Cutover
- [ ] Ventana de mantenimiento agendada (fin de semana)
- [ ] Comunicación al equipo con antelación
- [ ] CRM-ERP v4 puesto en read-only
- [ ] Delta final migrado (ventas del sábado)
- [ ] EspritOS encendido y probado con 3 supervisores
- [ ] CRM-ERP v4 mantenido read-only por 30 días
- [ ] Apagado formal del v4 después de 30 días sin issues

---

## Golden paths — deben funcionar siempre (smoke test pre-deploy)

- [ ] **Login admin → dashboard** con KPIs correctos
- [ ] **Login vendedor → /clientes** → solo ve sus clientes asignados
- [ ] **Login cobranza → /cobranza/aging** → ve buckets de todos los vendedores
- [ ] **Login supervisor → /chat** → pregunta "ventas de hoy" → recibe respuesta con tabla + SQL auditable
- [ ] **Login cajero → /pos** → abre turno → escanea producto → cobra efectivo → imprime ticket → cierra turno con arqueo correcto
- [ ] **Admin → /admin/facturacion** → timbra factura del día → CFDI descarga XML y PDF válidos
- [ ] **Admin → /comisiones/cierre-mensual** → corre cálculo → resultados coinciden con cálculo manual Excel

---

## Métricas de éxito (del Blueprint sección 1)

- [ ] **M1**: 5 supervisores usan el chat IA ≥10 veces/día durante semana 6
- [ ] **M2**: 0 deploys por USB después de semana 4
- [ ] **M3**: <1 bug regresivo por mes en módulos estables
- [ ] **M4**: Costo mensual real ≤ $110 USD durante primeros 3 meses
- [ ] **M5**: Chat IA p95 < 8s queries simples, < 20s complejas
- [ ] **M6**: 100% de los 27 SUMMARY.md del v4 tienen equivalente verificable
- [ ] **M7**: Venta POS completa (primer escaneo → ticket impreso) < 15 segundos
- [ ] **M8**: Cobertura de tests ≥ 80% por app, 100% en `sql_validator.py`

---

## Cuando todos los checkboxes estén marcados

1. Semana de operación paralela (v4 y EspritOS al mismo tiempo, comparación diaria)
2. Reunión con Beto + Humberto + Montserrat para sign-off
3. Apagar CRM-ERP v4 definitivamente
4. Escribir `context/phases/99-cierre-v4/SUMMARY.md` documentando el cierre
5. Celebrar con una torta en Cremería HM 🎂
