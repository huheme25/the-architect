# Cambios propuestos al BLUEPRINT.md de EspritOS
**Fecha:** 2026-04-11
**Autor:** Beto (con asistencia de Claude Code)
**Contexto:** Plan de limpieza y organización 2026-04-11 — Fase 2A

Este documento enumera los cambios concretos que deben aplicarse al `BLUEPRINT.md` autoritativo (archivo hermano en esta misma carpeta) para soportar:

1. Absorción completa de **POS Abarrotera HM** en EspritOS (multi-bodega real)
2. Validación de **Paso 29 (Comisiones)** contra la funcionalidad completa de VentasHM
3. Validación de **Paso 30 (Gastos)** contra la funcionalidad completa de Expenses (CRM-ERP Phase 13)
4. **Expansión del modelo de catálogo** de 3 niveles a **5 niveles jerárquicos + SKU + atributos ortogonales**, validado contra mejores prácticas de SAP Retail, Oracle RMS, NetSuite, Microsoft Dynamics 365 y Odoo.

---

## Cambio 1 — Expandir jerarquía de catálogo (Sección 4 Data Model + Apéndice D + Paso 14)

### Estado actual del Blueprint

El modelo `Categoria` actual está definido en líneas 474-479 de `BLUEPRINT.md` como árbol self-referential de **3 niveles** (Division / Linea / Grupo). El Apéndice D lo confirma en la línea ~2327:

> `Category` — 3 niveles (Division=1, Linea=2, Grupo=3), árbol self-referential

### Cambio propuesto

Reemplazar por un modelo de **5 niveles jerárquicos estrictos**, con SKU como hoja y **atributos ortogonales** en el Producto.

#### 1.1 Nuevo modelo `Categoria`

```
Categoria (hereda AuditedModel + SoftDeleteModel)
| id         | BigAutoField | PK                                              |
| nombre     | CharField(100) |                                               |
| codigo     | CharField(20)  | unique, slug/clave corta (ej: "LAC-QUE-FRE-PAN") |
| parent     | FK self        | nullable                                      |
| nivel      | SmallIntegerField | 1..5, validated                            |
| descripcion| TextField      | nullable                                      |
| activa     | BooleanField   | default True                                  |

Constraints:
- nivel debe ser consistente con parent.nivel + 1
- nivel 1 requiere parent NULL
- nivel 5 NO puede tener hijos Categoria (solo SKUs)
- unique_together: (parent, nombre)
```

**Semántica de los 5 niveles:**

| # | Nivel | Ejemplo Cremería HM | Cardinalidad esperada |
|---|---|---|---|
| 1 | División | Perecederos / Abarrotes / No-alimentos | 3–5 |
| 2 | Departamento | Lácteos, Embutidos, Cárnicos, Secos, Limpieza | 10–20 |
| 3 | Categoría | Quesos, Cremas, Yogures, Jamones | 40–80 |
| 4 | Familia | Queso fresco, Queso madurado, Queso fundido | 100–200 |
| 5 | Subfamilia | Panela, Oaxaca, Adobera, Cotija | 300–600 |

**SKU** (modelo `Producto`) es la **hoja del árbol**, no un sexto nivel jerárquico.

#### 1.2 Modelo `Producto` — cambios

Agregar/ajustar:

```
Producto (hereda AuditedModel + SoftDeleteModel)
| id             | BigAutoField       | PK                                      |
| codigo         | CharField(50)      | unique, código interno                  |
| codigo_pz      | CharField(50)      | nullable, código en Punto Zero          |
| nombre         | CharField(255)     |                                         |
| subfamilia     | FK Categoria       | REQUIRED, must be nivel=5               |
| marca          | FK Marca           | nullable — ATRIBUTO, no nivel jerárquico|
| unidad_medida  | CharField(20)      | "kg", "pieza", "litro", "caja"          |
| precio_lista   | DecimalField(12,2) | precio base                             |
| activo         | BooleanField       | default True                            |
| busqueda       | SearchVectorField  | full-text index español                 |

Atributos ortogonales (nuevos campos en Producto o tabla ProductoAtributo):
| presentacion       | CharField    | "granel" / "pieza" / "paquete" / "caja"  |
| catch_weight       | Boolean      | true si peso variable por corte          |
| cadena_frio        | CharField    | "ambiente" / "refrigerado" / "congelado" |
| bodega_disponible  | CharField    | "CR" / "AB" / "CR,AB" (ambas)            |
| proveedor_principal| FK Proveedor | nullable                                 |
| lote_obligatorio   | Boolean      | default false                            |
| dias_caducidad     | Integer      | nullable, para perecederos               |
| temporada          | CharField    | nullable, ej: "navidad" / "cuaresma"     |
| clase_abc_cr       | Char(1)      | "A"/"B"/"C" para bodega Cremería         |
| clase_abc_ab       | Char(1)      | "A"/"B"/"C" para bodega Abarrotera       |
| estado             | CharField    | "activo"/"descontinuado"/"estacional"    |
| es_comisionable    | Boolean      | mantener (para Paso 29)                  |
| sat_product_code   | CharField    | mantener (para CFDI)                     |
| sat_unit_code      | CharField    | mantener (para CFDI)                     |
```

**Regla clave**: `Producto.subfamilia` siempre apunta a `Categoria.nivel=5`. El árbol completo se obtiene navegando `parent` hacia arriba.

#### 1.3 Catálogo compartido entre bodegas

**Decisión:** una sola jerarquía global, no duplicar por bodega.

La diferenciación Cremería/Abarrotera se resuelve con el atributo `bodega_disponible` en `Producto` (puede ser "CR", "AB" o "CR,AB"). Las categorías son corporativas. Los reportes que necesiten vista por bodega pivotan `Producto.bodega_disponible × Categoria.nivel_N`.

**Justificación:** SAP, Oracle, NetSuite, Wisys y Davanti coinciden en que duplicar jerarquías por bodega rompe la consolidación de reportes y duplica master data innecesariamente. La alternativa correcta es una jerarquía maestra única + asignación N:M producto↔bodega.

#### 1.4 Paso 14 del Build Order — ajustes

El Paso 14 actual de la Ola A (Catálogo base) debe:
- Implementar `Categoria` con los 5 niveles y sus constraints
- Implementar `Producto` con FK obligatoria a subfamilia (nivel=5)
- Implementar `Marca` como modelo separado **NO jerárquico** (es atributo del producto)
- Crear migración inicial con seed mínimo de las 3 Divisiones (Perecederos / Abarrotes / No-alimentos)
- Tests: validar que no se pueda asignar un Producto a una Categoría de nivel < 5; validar que no se pueda crear un hijo bajo nivel 5

#### 1.5 ETL Paso 6 — ajustes

El Paso 6 (ETL desde Punto Zero) debe mapear los datos legacy de PZ a la nueva estructura:
- `Lineas` de PZ → Departamento o Categoría (según granularidad real del dato)
- `SubLineas` de PZ → Familia o Subfamilia
- Productos sin clasificar completa → asignar a "Subfamilia genérica" de su Familia, marcar con flag `requiere_recategorizacion`
- Documentar en el ETL que la categorización fina (Subfamilia) es un paso de data quality posterior, no automático

---

## Cambio 2 — Validación Paso 29 (Comisiones, absorbe VentasHM)

### Estado actual del Blueprint

El Paso 29 en Ola D ya define `apps.comisiones` con 8 modelos:

- `CommissionProfile` (modos progresivo/plano/fijo, con tasas y thresholds NC)
- `CommissionTier` (tramos con lowerLimit/upperLimit/rate)
- `CommissionMonthlyTarget` (metas por vendedor × año × mes)
- `CommissionResult` (resultado mensual congelado)
- `NonCommissionableFamily` (HUEVO, MARGARINA DELICIA, etc. con patterns JSON)
- `SeasonalWeight` (pesos estacionales por mes)
- `ProductTarget` (metas por producto × mes)
- `LegacyCsvUpload` + `LegacyCsvRow` (importer de PZ, 4 formatos)

### Validación contra VentasHM original

| Funcionalidad VentasHM | En Paso 29 Blueprint | Acción |
|---|---|---|
| 3 modos cálculo (progresivo/plano/fijo) | ✅ Definido | Ninguna |
| Motor matemático puro | ⚠️ Implícito pero no explicitado | Agregar nota en Paso 29: "el motor de cálculo debe ser funciones puras en `apps.comisiones.engine` sin dependencia de ORM — facilita tests y portabilidad" |
| Tramos progresivos con desglose | ✅ Via CommissionTier + breakdownJson | Ninguna |
| Clasificador NC por patterns | ✅ Via NonCommissionableFamily | Ninguna |
| 5 familias NC seed (HUEVO, MARGARINA DELICIA, SAN ANTONIO, YAKULT, SAN MILLAN) | ⚠️ Solo menciona HUEVO, MARGARINA DELICIA como ejemplo | Agregar lista completa explícita como seed: HUEVO, MARGARINA DELICIA, SALCHICHA SAN ANTONIO, YAKULT, SALCHICHA SAN MILLAN |
| Importador CSV 4 formatos PZ | ✅ Via LegacyCsvUpload.format (remisiones_v1/v2, facturas_v1/v2/v2b/v3) | Ninguna — el blueprint incluso menciona 6 formatos (v2b y v3 extras) |
| Dedup por SHA256 de archivo | ⚠️ No explícito | Agregar: "LegacyCsvUpload tiene campo `file_hash` (SHA256) con unique constraint para evitar re-import del mismo archivo" |
| Dashboard vendedor (mis comisiones) | ⚠️ No explícito | Agregar en Paso 29: endpoint `/api/comisiones/mi-resultado/` (auth vendedor) + endpoint `/api/comisiones/resultados/` (auth admin) |
| Simulador "qué pasaría si vendo $X más" | ❌ No mencionado | **Decisión:** diferir a backlog — no estaba en CRM-ERP Phase 14, solo en VentasHM standalone. No bloquea migración. |
| Exportar resultado Excel | ⚠️ No explícito | Agregar endpoint `/api/comisiones/exportar/` |
| Permisos granulares (vendedor ve solo lo suyo) | ✅ Cubierto en Paso 38 (Permisos granulares Ola G) | Verificar en Paso 38 que incluye `comisiones.view_own` |

**Conclusión Cambio 2:** El Paso 29 del blueprint está **~90% completo**. Se agregan 4 clarificaciones menores:
- Motor como funciones puras en `apps.comisiones.engine`
- Seed completo de 5 familias NC
- `file_hash` SHA256 en LegacyCsvUpload
- Endpoints explícitos de dashboard y exportación

---

## Cambio 3 — Validación Paso 30 (Gastos, absorbe Expenses)

### Estado actual del Blueprint

El Paso 30 en Ola D define `apps.gastos` con 3 modelos:

- `OperationalExpense` (folio GAS-CREM-2026-000001, categoría, método pago, tags JSON, accountsPayableId opcional)
- `ExpenseTemplate` (gasto recurrente con frequency mensual/semanal)
- `ExpenseBudget` (category × year × month + amount)

### Validación contra Expenses original + CRM-ERP Phase 13

| Funcionalidad | En Paso 30 Blueprint | Acción |
|---|---|---|
| 9 categorías (operativo, nómina, mercancía, administrativo, transporte, ventas, impuestos, mantenimiento, servicios bancarios) | ⚠️ No listadas explícitamente | Agregar lista explícita como Django TextChoices |
| 3 métodos pago (transferencia, efectivo, tarjeta) | ✅ Mencionado | Ninguna |
| Folio GAS-{BRANCH}-{YEAR}-{SEQ} | ✅ Ejemplo dado | Confirmar uso de `apps.core.DocumentSequence` |
| Tags JSON hasta 5 por gasto | ✅ Via tags JSON field | Agregar constraint: max 5 tags por gasto |
| CRUD completo con filtros (categoría, método, proveedor, fechas, tags) | ⚠️ Implícito | Agregar en Paso 30: ViewSet con django-filter, ordering por fecha/monto |
| Plantillas recurrentes (registrar desde plantilla, auto nextDueDate, flag vencidas) | ✅ Via ExpenseTemplate | Agregar custom action `POST /api/gastos/plantillas/{id}/registrar/` |
| Presupuestos con umbrales de color | ⚠️ Modelo existe, UX no | Agregar en Paso 30: endpoint `/api/gastos/dashboard/` con cálculo de % gastado y thresholds (verde <60%, amarillo 60-80%, naranja 80-100%, rojo >100%) |
| Exportar CSV/Excel | ❌ No mencionado | Agregar endpoint `/api/gastos/exportar/` |
| Link a CxP (accountsPayableId) | ✅ Mencionado | Ninguna |
| Integración con CashShift (gastos en efectivo) | ⚠️ No explícito | Nota: ver si CashMovement type=PAY_OUT cubre esto. Si sí, agregar FK opcional `gasto` en CashMovement para trazabilidad. Si no, diferir a backlog. |
| Auditoría completa (quién, cuándo) | ✅ Via AuditedModel | Ninguna |
| Permisos (ADMIN elimina, GERENTE edita su sucursal) | ✅ Cubierto Paso 38 | Verificar en Paso 38 |

**Conclusión Cambio 3:** El Paso 30 del blueprint está **~85% completo**. Se agregan:
- Lista explícita de 9 categorías como TextChoices
- Max 5 tags por gasto
- Endpoint dashboard con thresholds
- Endpoint exportar CSV/Excel
- Nota sobre trazabilidad CashMovement↔Gasto

---

## Cambio 4 — Absorción explícita de POS Abarrotera HM

### Estado actual del Blueprint

El Paso 19 (POS fullscreen) y Paso 21 (Hardware POS) ya contemplan:
- `CashShift` con `branchId` → multi-sucursal por diseño
- `Sale` con `branch` y `warehouse` → multi-sucursal por diseño
- `Sucursal` con código CREM/ABAR
- `Warehouse` ALM-CREM y ALM-ABAR
- Hardware (impresora ESC/POS, báscula serial, lector barcode, cajón)

**Infraestructura multi-sucursal ya está.** Pero no hay mención explícita de "absorbe POS Abarrotera HM".

### Cambio propuesto

#### 4.1 Agregar nota en Paso 19

Al final del Paso 19 agregar:

> **(absorbe POS Abarrotera HM)** — El repositorio independiente `POS Abarrotera HM` (Next.js + Prisma + SQLite, con 4 productos de testing y sin histórico real) queda absorbido en este paso. La infraestructura multi-sucursal definida aquí (CashShift × branch, Sale × branch, Warehouse ALM-CREM/ALM-ABAR) cumple los casos de uso del POS standalone. Ningún dato histórico real requiere migración (verificado 2026-04-11 vía export a `RevisionServidor/exports/abarrotera-pos/`).

#### 4.2 Agregar en Paso 15 o 16 (Ola A) — PosConfig multi-sucursal

Si el Blueprint no lo especifica hoy, asegurar que:

- `PosConfig` (si existe en el blueprint) es **multi-sucursal** (FK Sucursal) o bien que la config se resuelve vía atributos de `Sucursal` directamente
- Selector de sucursal en el header del POS frontend

#### 4.3 Tests obligatorios del Paso 19

Agregar como criterio de cierre del Paso 19:
- Test de extremo a extremo: abrir turno en sucursal CREM, vender, cerrar turno
- Test de extremo a extremo: abrir turno en sucursal ABAR en paralelo al turno CREM, vender, cerrar turno
- Verificar que el folio `VTA-CREM-2026-000001` y `VTA-ABAR-2026-000001` se generan en secuencias independientes vía `DocumentSequence`

---

## Cambio 5 — Aclaración del Catálogo y compras inter-bodega (nueva nota)

### Contexto

Cremería HM opera 2 bodegas que comparten proveedores, precios de compra y en muchos casos los mismos productos físicos. La diferencia principal es el **catálogo disponible** por bodega (algunos productos solo se venden en una de las dos).

### Cambio propuesto

Agregar al final de la Sección 4 (Data Model) una nota:

> **Modelo de bodega en el catálogo**
>
> El catálogo de productos es **global**. La disponibilidad por bodega se modela como atributo multi-valor (`Producto.bodega_disponible` ∈ {"CR", "AB", "CR,AB"}), no como duplicación de jerarquía. Esto sigue la best practice de SAP Retail, Oracle RMS, NetSuite, Microsoft Dynamics 365 y Wisys WMS: *"una sola jerarquía maestra con asignación producto↔bodega, para preservar consolidación de reportes y evitar duplicación de master data"*.
>
> Los reportes que requieran vista por bodega pivotan `Producto.bodega_disponible × Categoria.nivel_N` sin requerir duplicación del árbol.

---

## Impacto de estos cambios

| Área | Antes | Después |
|---|---|---|
| Niveles de categoría | 3 (Division/Linea/Grupo) | 5 (División→Departamento→Categoría→Familia→Subfamilia) + SKU + 12 atributos |
| Marca | Entidad separada implícitamente jerárquica | Atributo ortogonal explícito |
| Bodega en catálogo | Ausente | Atributo ortogonal `bodega_disponible` con valores CR/AB/ambas |
| Paso 29 Comisiones | 90% listo | 100% listo — 4 clarificaciones menores |
| Paso 30 Gastos | 85% listo | 100% listo — 5 clarificaciones menores |
| Paso 19 POS | Multi-sucursal por diseño, sin mención de Abarrotera | Absorción explícita de POS Abarrotera HM + tests E2E |
| Pasos afectados | N/A | Paso 6 (ETL), 14 (Catálogo), 15-16 (PosConfig), 19 (POS), 29 (Comisiones), 30 (Gastos), 38 (Permisos) |

## Efecto en el Build Order

Ninguno de estos cambios agrega pasos nuevos ni reordena los existentes. Son **todos modificaciones dentro de pasos ya definidos**. El Builder puede seguir ejecutando en orden, simplemente con specs más completas.

---

## Próximo paso

Una vez que Beto valide este documento:

1. **Opción B del flujo the-architect:** abrir sesión nueva en `E:/ClaudeWorks/repos-referencia/the-architect/`, pasar este documento como input y pedir regeneración consolidada de las secciones afectadas del BLUEPRINT.
2. Validar el diff antes de sobrescribir `BLUEPRINT.md`.
3. El Builder de EspritOS (sesión separada en `E:/ClaudeWorks/proyectos/EspritOS/`) retoma el trabajo con el BLUEPRINT actualizado cuando llegue a la Ola A (Catálogo), Ola B (POS), u Ola D (Comisiones+Gastos).
