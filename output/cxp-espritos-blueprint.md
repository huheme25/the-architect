# Control de Pagos y Proveedores (CxP) — Blueprint para EspritOS

> Generado por The Architect el 01/09/2026
> Arquetipo: Internal Tool — **embebido en el monolito Django 5.1 de EspritOS** (NO greenfield)
> Repo anfitrión: `E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS` · GitHub `huheme25/espritos` · prod https://espritos.app
> Construye: **The Builder**, en rama `feature/cxp`. Estrategia: **reactivar y extender `apps/compras` + `apps/proveedores`** (apps en standby desde el pivote 09/06/2026 — ver `apps/core/sidebar.py:242-249`). Cero apps nuevas.

---

## 0. Cómo usar este blueprint (para The Builder)

Este documento es **100% autocontenido**. Todo patrón citado está calcado de código real en prod — las rutas `archivo:línea` apuntan al repo EspritOS para verificar el original si dudas. Regla de oro: **calca el patrón, no lo reinventes**.

La diferencia clave contra otros builds: aquí **NO se construye desde cero**. Los modelos `Proveedor`, `CuentaPorPagar` y `PagoCxP` ya existen y tienen migraciones aplicadas en prod. El trabajo es: (a) migraciones aditivas sobre esos modelos, (b) dos modelos nuevos, (c) servicios nuevos, (d) las vistas/UI que nunca se construyeron, (e) permisos + sidebar, (f) activar el ETL de proveedores sin que pise datos editados en EspritOS.

Orden de lectura: §1 (qué es) → §3 (modelo de datos) → §4 (servicios) → §5 (ETL) → §6 (vistas/UI) → §7 (permisos) → **§9 (build order — sigue paso por paso)** → §10 (tests) → §11 (reglas).

---

## 1. Visión, alcance y métricas

### Visión

Cremería HM compra a ~126 proveedores. Hoy no hay un lugar único donde se vea cuánto se debe, a quién, y qué toca pagar cada día. **CxP** cierra ese hueco con un ciclo de 4 pasos y 4 actores:

1. **Almacén** recibe al proveedor y captura la compra: monto total, foto de la nota/factura y notas libres. Si después llegan notas de crédito o ajustes de precio, también los captura — el saldo se ajusta solo.
2. **Beto (admin)** ve las cuentas por vencer y arma la corrida: qué cuenta se paga qué día y por cuánto.
3. **Tesorería** entra a "pagos de hoy", paga, ajusta el monto real pagado si difiere del programado, y sube foto/PDF del comprobante de transferencia.
4. **Contabilidad/Montse** ve la evidencia completa: la nota que capturó almacén y el comprobante que subió tesorería, lado a lado.

El catálogo de proveedores viene de PuntoZero (vía el ETL existente), pero **las condiciones comerciales (días de crédito, límite) se administran en EspritOS** porque PZ está en desorden. EspritOS es el punto de captura, no espejo.

### Goals

- Capturar una compra en < 60 segundos desde el celular en el andén de recepción (proveedor + monto + foto + listo).
- El vencimiento se calcula solo: `fecha_compra + proveedor.dias_credito`.
- Notas de crédito bajan el saldo ANTES del pago — tesorería siempre ve el monto correcto a pagar.
- Corrida de pagos: programar N pagos con fecha; tesorería ve exactamente su lista del día.
- Evidencia fotográfica de punta a punta: foto de la nota (almacén) + foto del comprobante (tesorería), servidas con control de permisos.
- Dashboard: total adeudado, por proveedor, por vencer esta semana, vencidas.

### Success metrics (la rúbrica de Tomás sale de aquí)

- Ciclo completo demostrable: captura compra con foto → NC que baja saldo → programación → tesorería paga con monto ajustado + comprobante → CxP en estado PAGADO con evidencia visible. Todo por UI, sin tocar el admin de Django.
- `saldo == monto_total - monto_pagado - Σ(notas de crédito activas)` se sostiene bajo cualquier orden de operaciones (probado con tests).
- Matriz de permisos verde: almacenista NO ve programación ni tesorería; tesorería NO captura compras; un usuario sin módulo recibe 403 (incluidas las fotos).
- El ETL de proveedores corre y NO pisa `dias_credito`/`limite_credito`/`descuento_pronto_pago` editados en EspritOS (test explícito).
- Suite completa del repo verde (`scripts/test_docker.ps1`) y vistas de lista sin N+1 (< 20 queries).

### Anti-alcance (NO se construye en v1)

- ❌ **Conciliación bancaria** (cargar estado de cuenta y cruzar) — fuera por decisión de Beto 01/09/2026.
- ❌ **CFDI/XML** — la captura es manual con foto. El módulo `apps/cfdi` existe pero no se toca.
- ❌ **Órdenes de compra con productos línea-por-línea** — el flujo pesado (`OrdenCompra → Recepcion → WAC → inventario`) queda **dormido e intacto**. Esta v1 NO toca inventario ni costos promedio. Si algún día se quiere detalle por producto, ese flujo ya existe (`apps/compras/services.py:123-397`).
- ❌ Multi-moneda, anticipos sin factura, intereses moratorios.
- ❌ App nueva — todo vive en `apps/compras` (vistas/servicios/modelos CxP) y `apps/proveedores` (catálogo).

---

## 2. Stack (FIJO — es el de EspritOS, no negociable)

Django 5.1 · Python 3.12 · PostgreSQL 16 · Redis 7 · Celery 5 · **HTMX 2 + Alpine.js 3 + Tailwind v4 compilado (NO Play CDN) + DaisyUI** · django-filter · crispy-tailwind. Server-rendered first: HTMX intercambia **HTML, no JSON**. Sin SPA. Clases CSS: convención `hm-*` del repo (`hm-input`, `hm-textarea`, `hm-select`, `hm-badge`).

**Sin dependencias nuevas.** En particular: NO hay Pillow en `pyproject.toml` → usar `FileField` (no `ImageField`) para las fotos, con validación propia (§3.6). Las fotos se muestran con `<img src>` normal; tesorería puede subir PDF y se muestra con `<embed>` o link.

**Diseño**: hereda el design system de EspritOS tal cual (sidebar, `base.html`, componentes `hm-*`, DaisyUI). Cero diseño nuevo. Las vistas de captura (almacén) y tesorería son **mobile-first** — se usan en piso con el celular.

---

## 3. Modelo de datos

### 3.1 Lo que YA existe (no recrear — migrar encima)

| Modelo | Dónde | Qué trae ya |
|---|---|---|
| `Proveedor` | `apps/proveedores/models.py:27` | `codigo`, `codigo_pz` (liga con PZ), `dias_credito`, `limite_credito`, `descuento_pronto_pago`, fiscal, contacto |
| `CuentaPorPagar` | `apps/compras/models.py:306` | folio, proveedor, `monto_total`, `monto_pagado`, `saldo`, estados PENDIENTE/PARCIAL/PAGADO, `fecha_vencimiento`, properties `esta_vencida`/`dias_vencida`, índices por estado+vencimiento |
| `PagoCxP` | `apps/compras/models.py:422` | monto, método (EFECTIVO/TRANSFERENCIA/CHEQUE), `referencia_bancaria`, pagos parciales ya soportados |
| `DocumentSequence` | `apps/core/models.py:194` | folios atómicos `CXP-CREM-2026-000001` vía `DocumentSequence.next("CXP", sucursal.codigo)` |
| `AuditedModel` | `apps/core/models.py` | `created_by/updated_by/created_at/updated_at` — heredado por los modelos de arriba |

### 3.2 Migración sobre `CuentaPorPagar` (aditiva)

Hoy la CxP **exige** una `OrdenCompra` (FK non-null). La compra directa la crea sin OC. Cambios:

```python
# apps/compras/models.py — cambios a CuentaPorPagar

class OrigenCxP(models.TextChoices):
    COMPRA_DIRECTA = "DIRECTA", "Compra directa"      # flujo nuevo (v1)
    RECEPCION_OC = "RECEPCION", "Recepción de OC"     # flujo pesado dormido

class CuentaPorPagar(AuditedModel):
    # ... campos existentes intactos ...

    # CAMBIO: orden pasa a nullable (las CxP de compra directa no tienen OC)
    orden = models.ForeignKey(
        OrdenCompra, on_delete=models.PROTECT,
        null=True, blank=True,                        # ← antes non-null
        related_name="cuentas_por_pagar",
    )

    # NUEVOS CAMPOS
    origen = models.CharField(
        max_length=10, choices=OrigenCxP.choices,
        default=OrigenCxP.COMPRA_DIRECTA, db_index=True,
    )
    sucursal = models.ForeignKey(
        "core.Sucursal", on_delete=models.PROTECT,
        null=True, blank=True, related_name="cuentas_por_pagar",
        help_text="Sucursal que recibió la compra. Null solo en filas legacy.",
    )
    fecha_compra = models.DateField(
        null=True, blank=True,
        help_text="Fecha de la compra/nota. Base del vencimiento en compra directa.",
    )
    folio_factura = models.CharField(
        max_length=60, blank=True,
        help_text="Folio de la nota/factura del proveedor (opcional).",
    )
    foto_nota = models.FileField(
        upload_to="cxp/notas/%Y/%m/", blank=True,
        help_text="Foto de la nota/factura que capturó almacén.",
    )
    monto_notas_credito = models.DecimalField(
        max_digits=14, decimal_places=2, default=Decimal("0"),
        help_text="Suma de notas de crédito activas. Mantenida por servicios.",
    )
```

En la data migration: filas existentes (si las hay) → `origen=RECEPCION`. La fórmula del saldo cambia en TODO el módulo a:

```
saldo = max(0, monto_total - monto_pagado - monto_notas_credito)
```

y la mantiene **un solo lugar**: `_recalcular_cxp()` (§4.1). Nadie más escribe `saldo`/`estado` a mano.

### 3.3 `PagoCxP` — un campo nuevo

```python
    foto_comprobante = models.FileField(
        upload_to="cxp/comprobantes/%Y/%m/", blank=True,
        help_text="Foto/PDF del comprobante de transferencia. La sube tesorería.",
    )
```

### 3.4 `NotaCredito` (modelo NUEVO en `apps/compras/models.py`)

```python
class NotaCredito(AuditedModel):
    """
    Nota de crédito / descuento del proveedor contra una CxP.
    Baja el saldo ANTES del pago. Almacén la captura y puede cancelarla
    (soft-cancel — nunca se borra, la evidencia queda).
    """
    folio = models.CharField(max_length=30, unique=True)   # NC-CREM-2026-000001
    cuenta = models.ForeignKey(
        CuentaPorPagar, on_delete=models.PROTECT, related_name="notas_credito",
    )
    monto = models.DecimalField(max_digits=14, decimal_places=2)
    motivo = models.CharField(
        max_length=255,
        help_text="Ej: ajuste de precio, devolución, merma, descuento.",
    )
    foto = models.FileField(
        upload_to="cxp/notas_credito/%Y/%m/", blank=True,
        help_text="Foto de la nota de crédito física (opcional).",
    )
    cancelada = models.BooleanField(default=False, db_index=True)
    cancelada_motivo = models.CharField(max_length=255, blank=True)

    class Meta:
        db_table = "notas_credito_cxp"
        verbose_name = "Nota de crédito"
        verbose_name_plural = "Notas de crédito"
        ordering = ["-created_at"]
        constraints = [
            models.CheckConstraint(
                check=models.Q(monto__gt=0),
                name="ck_nc_monto_positivo",
            ),
        ]
```

### 3.5 `PagoProgramado` (modelo NUEVO en `apps/compras/models.py`)

```python
class EstadoPagoProgramado(models.TextChoices):
    PROGRAMADO = "PROGRAMADO", "Programado"
    EJECUTADO = "EJECUTADO", "Ejecutado"
    CANCELADO = "CANCELADO", "Cancelado"

class PagoProgramado(AuditedModel):
    """
    La corrida de pagos: Beto decide qué CxP se paga qué día y por cuánto.
    Tesorería ejecuta contra esto. El monto real pagado puede diferir del
    programado (queda en el PagoCxP ligado).
    """
    cuenta = models.ForeignKey(
        CuentaPorPagar, on_delete=models.PROTECT, related_name="pagos_programados",
    )
    fecha_programada = models.DateField(db_index=True)
    monto_programado = models.DecimalField(max_digits=14, decimal_places=2)
    estado = models.CharField(
        max_length=12, choices=EstadoPagoProgramado.choices,
        default=EstadoPagoProgramado.PROGRAMADO, db_index=True,
    )
    pago = models.OneToOneField(
        PagoCxP, on_delete=models.PROTECT,
        null=True, blank=True, related_name="programacion",
        help_text="El pago real que ejecutó esta programación.",
    )
    notas = models.TextField(blank=True)

    class Meta:
        db_table = "pagos_programados_cxp"
        verbose_name = "Pago programado"
        verbose_name_plural = "Pagos programados"
        ordering = ["fecha_programada", "pk"]
        indexes = [
            models.Index(fields=["estado", "fecha_programada"]),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(monto_programado__gt=0),
                name="ck_pp_monto_positivo",
            ),
        ]
```

### 3.6 Validación de archivos (sin Pillow)

Helper compartido en `apps/compras/validators.py`:

- Extensiones permitidas: `.jpg .jpeg .png .webp .pdf` (case-insensitive).
- Tamaño máximo: **10 MB** (`file.size`).
- Aplicar como `validators=[validar_evidencia]` en los tres `FileField` y re-validar en el form.
- En los forms de captura móvil: `<input type="file" accept="image/*,application/pdf" capture="environment">` — abre la cámara directo en el celular.

### 3.7 Diagrama

```
Proveedor (catálogo desde PZ; dias_credito/limite editados en EspritOS)
  └── CuentaPorPagar (origen=DIRECTA, monto_total, foto_nota, notas,
      │                vencimiento = fecha_compra + dias_credito,
      │                saldo = total - pagado - notas_credito)
      ├── NotaCredito (0..N — baja el saldo, soft-cancel, foto)
      ├── PagoProgramado (0..N — la corrida: fecha + monto)
      │     └── PagoCxP (1:1 al ejecutar — monto REAL + foto_comprobante)
      └── PagoCxP (1..N — pagos parciales soportados)
```

---

## 4. Servicios (`apps/compras/services.py` — extender el archivo existente)

El archivo ya trae `registrar_pago_cxp` (`services.py:434`) y el patrón: `@transaction.atomic`, `select_for_update`, excepciones de dominio propias, folios vía `DocumentSequence`. Calca ese estilo. Servicios nuevos:

### 4.1 `_recalcular_cxp(cuenta)` — el corazón

```
monto_nc   = Σ notas_credito activas (cancelada=False)
saldo      = max(0, monto_total - monto_pagado - monto_nc)
estado     = PAGADO  si saldo <= 0.01 (y saldo := 0)
             PARCIAL si monto_pagado > 0 o monto_nc > 0
             PENDIENTE en otro caso
```

Único escritor de `saldo`/`estado`/`monto_notas_credito`. **Refactor obligatorio**: `registrar_pago_cxp` existente (`services.py:476-490`) recalcula saldo a mano — cambiarlo para delegar en `_recalcular_cxp` (con test de regresión del comportamiento previo: sobrepago permitido, tolerancia 0.01).

### 4.2 `registrar_compra_directa(...)`

Args: `proveedor_id, sucursal_id, monto_total, fecha_compra, folio_factura="", foto_nota=None, notas="", user`.
- Folio: `DocumentSequence.next("CXP", sucursal.codigo)`.
- `fecha_vencimiento = fecha_compra + timedelta(days=proveedor.dias_credito)` (calca `services.py:410-411`).
- `origen=DIRECTA`, `orden=None`, `recepcion=None`.
- Validar `monto_total > 0`. Warning no-bloqueante si el adeudo total del proveedor supera `limite_credito` (se muestra en la UI, no impide capturar).

### 4.3 `editar_compra_directa(cuenta_id, ...)`

- Sin pagos ejecutados: almacén puede editar `monto_total`, `fecha_compra`, `folio_factura`, `foto_nota`, `notas` → recalcular vencimiento y saldo.
- Con pagos: solo `folio_factura`, `foto_nota`, `notas`. El monto ya no se toca (los ajustes van por nota de crédito).
- Solo cuentas `origen=DIRECTA` — las de recepción no se editan por aquí.

### 4.4 `registrar_nota_credito(cuenta_id, monto, motivo, foto=None, user)`

- Folio `DocumentSequence.next("NC", sucursal.codigo)`.
- Validar `monto > 0` y `monto <= cuenta.saldo` (una NC no puede dejar saldo negativo).
- Rechazar sobre cuenta PAGADO.
- → `_recalcular_cxp`.

### 4.5 `cancelar_nota_credito(nc_id, motivo, user)`

- `cancelada=True` + motivo → `_recalcular_cxp`. Rechazar si al revertir la cuenta quedaría con `monto_pagado > monto_total - monto_nc` en estado inconsistente… no: el sobrepago ya es legal en el módulo; simplemente recalcular (el saldo se clampa a ≥ 0).

### 4.6 `programar_pago(cuenta_id, fecha_programada, monto_programado, notas="", user)`

- Validar cuenta no PAGADO, `monto > 0`, `monto <= saldo` (default en UI: el saldo completo).
- Permitir varias programaciones activas por cuenta solo si la suma de `monto_programado` de las PROGRAMADO ≤ saldo.

### 4.7 `cancelar_pago_programado(pp_id, user)` → estado CANCELADO (solo si PROGRAMADO).

### 4.8 `ejecutar_pago_programado(pp_id, monto_real, metodo_pago, referencia_bancaria="", foto_comprobante=None, notas="", user)`

- Solo sobre PROGRAMADO. `monto_real` editable por tesorería (default: `monto_programado`, o el saldo si es menor).
- Llama `registrar_pago_cxp` (extendido con `foto_comprobante`) → liga `pp.pago`, `pp.estado=EJECUTADO`.
- Todo en una transacción.

Excepciones nuevas siguen el patrón existente (`PagoCxPInvalidoError`, `services.py:86`): `NotaCreditoInvalidaError`, `PagoProgramadoInvalidoError`, `CompraDirectaInvalidaError`.

---

## 5. ETL de proveedores desde PuntoZero

El extractor **ya existe y funciona**: `apps/etl/extractors/proveedores.py` (full load ~126 proveedores, upsert por `codigo`, columnas reales de PZ EVO 5.1.48 documentadas ahí mismo). Trabajo:

1. **Protección anti-clobber (crítico)**: hoy el upsert actualizaría TODOS los campos, pisando `dias_credito`/`limite_credito`/`descuento_pronto_pago` editados en EspritOS. Cambiar el extractor para que en **conflicto (fila ya existe)** actualice SOLO identidad: `nombre, rfc, direccion, codigo_postal, telefono, email, codigo_pz`. En **insert (proveedor nuevo)** sí siembra `dias_credito`/`limite_credito` desde PZ como valor inicial. Mecanismo: revisar `apps/etl/loaders/postgres_upsert.py` y `extractors/base.py:322` — si el loader no soporta subset de update, hacer override de `run()` como `ventas_pos.py:283`. **Test obligatorio**: editar `dias_credito` en EspritOS → correr extractor → el valor NO cambia.
2. **Activarlo en el corrido programado** del ETL (donde corren los demás extractores — ver `apps/etl/management/commands/` y el beat de Celery; calcar cómo está registrado `clientes`).
3. **UI de condiciones comerciales**: `apps/proveedores/views.py` ya existe — verificar qué trae. Se necesita: lista de proveedores (búsqueda por nombre/código) + edición de `dias_credito`, `limite_credito`, `descuento_pronto_pago`, contacto. Permiso: módulo `cxp_captura` para ver, solo admin edita condiciones (§7).

---

## 6. Vistas, URLs y UI

Namespace existente `compras` (`apps/compras/urls.py` ya está incluido o se incluye en `config/urls.py` — verificar y agregar si falta, patrón de las demás apps). Prefijo `/compras/cxp/`.

| URL | Vista | Quién | Qué hace |
|---|---|---|---|
| `cxp/captura/` | `CapturaCompraView` | almacén | Form mobile-first: proveedor (select con búsqueda), monto, fecha (default hoy), folio factura, foto (cámara), notas. Al guardar muestra vencimiento calculado. |
| `cxp/` | `CxPListaView` | todos los del módulo | Lista con filtros (django-filter): estado, proveedor, vencidas, rango fechas. Badges `hm-badge` por estado. Orden default: vencimiento asc. |
| `cxp/<pk>/` | `CxPDetalleView` | todos | Timeline: compra + foto, NCs, programaciones, pagos + comprobantes. Botones según permisos. |
| `cxp/<pk>/editar/` | `EditarCompraView` | almacén | Reglas §4.3. |
| `cxp/<pk>/nota-credito/` | `NotaCreditoCreateView` | almacén | Modal HTMX: monto, motivo, foto. |
| `cxp/nc/<pk>/cancelar/` | POST | almacén/admin | Soft-cancel con motivo. |
| `cxp/programacion/` | `ProgramacionView` | Beto/admin | Dos paneles: CxPs con saldo (ordenadas por vencimiento, semáforo vencidas) → botón "programar" (modal: fecha + monto, default saldo). Abajo: corrida programada agrupada por fecha con totales por día. |
| `cxp/tesoreria/` | `TesoreriaHoyView` | tesorería | Pagos programados de HOY + atrasados (fecha < hoy, aún PROGRAMADO). Cada fila: proveedor, monto programado, link a la foto de la nota. Botón "pagar" → modal: monto real (editable), método, referencia, foto comprobante. |
| `cxp/dashboard/` | `CxPDashboardView` | admin/tesorería | KPIs: total adeudado, por vencer 7 días, vencido, # cuentas abiertas. Tabla top proveedores por saldo. |
| `cxp/evidencia/<tipo>/<pk>/` | `EvidenciaView` | según módulo | Sirve la foto con `FileResponse` + check de permisos. `tipo ∈ {nota, nc, comprobante}`. |
| `proveedores/` | reactivar/completar | ver §5.3 | Catálogo + condiciones comerciales. |

**Patrones obligatorios:**

- HTMX para modales y acciones (patrón del repo — intercambio de HTML parcial, no JSON).
- `select_related("proveedor", "sucursal")` + `prefetch_related("notas_credito", "pagos")` en listas — success metric de < 20 queries.
- **Fotos NUNCA por `/media/` directo**: `config/urls.py:62-68` solo sirve media en DEBUG; en prod el volumen `espritos_prod_media` existe pero el acceso va por vista con `FileResponse` + permiso (calca `apps/aprendizaje/views.py:204` y `apps/reportes/views.py:499`). Así un link filtrado no expone comprobantes bancarios.
- Templates en `templates/compras/` extendiendo el `base.html` del repo; clases `hm-*`; captura y tesorería probadas a 375px de ancho.
- Selector de proveedor: los ~126 caben en un `<select>` con Alpine para filtrado client-side; no hace falta autocomplete server-side.

---

## 7. Permisos y roles

Sistema existente: enum `Modulo` en `apps/core/models.py` + matriz `RolePermission` (rol × módulo × ver/crear/editar/eliminar) + `check_permission()`/`ModuloRequeridoMixin` (`apps/core/permissions.py:238,370`) + seeds por data migration (calca `apps/core/migrations/0027_seed_tareas_permissions.py` y `0031_add_rh_modulo.py`).

**Módulos nuevos** (2, siguiendo el patrón `pulso`/`pulso_comisiones` de granularidad por audiencia):

- `cxp_captura` — captura de compras, NCs, lista/detalle de CxP, catálogo proveedores.
- `cxp_pagos` — programación, tesorería, dashboard.

**Grupos**: `almacenista` YA existe (`apps/core/permissions.py:74`). Crear grupo `tesoreria` en la data migration.

| Grupo | cxp_captura | cxp_pagos | Notas |
|---|---|---|---|
| `almacenista` | ver/crear/editar | — | No elimina. No ve programación ni pagos. |
| `tesoreria` | ver | ver/crear | Ve la nota que va a pagar; ejecuta pagos. No captura ni programa. |
| `admin` (Beto, Montse) | todo | todo | Programación es de admin. Superuser bypasea (fail-closed si matriz vacía — ya resuelto en core). |

- Vistas con `ModuloRequeridoMixin` (`modulo_permiso` + `accion_permiso`).
- `EvidenciaView`: `tipo=comprobante` exige `cxp_pagos:ver` O `cxp_captura:ver` (contabilidad/Montse entra por admin); `nota`/`nc` exigen `cxp_captura:ver` o `cxp_pagos:ver` (tesorería necesita ver la nota).
- Editar condiciones comerciales del proveedor (días de crédito, límite): acción `editar` de `cxp_pagos` (admin) — almacén ve pero no cambia el crédito.
- La migración siembra `RolePermission` para los 2 módulos × 3 grupos y llama `invalidate_permission_cache()`.

## 8. Sidebar

En `apps/core/sidebar.py` (los módulos CxP salen del bloque de apps ocultas `sidebar.py:242-249`):

```python
SidebarGrupo(
    key="pagos", label="Pagos", icon="dollar", seccion="principal",
    items=[
        SidebarItem(label="Capturar compra", icon="box", url_name="compras:cxp_captura", modulo="cxp_captura"),
        SidebarItem(label="Cuentas por pagar", icon="dollar", url_name="compras:cxp_lista", modulo="cxp_captura"),
        SidebarItem(label="Programación", icon="calendar", url_name="compras:cxp_programacion", modulo="cxp_pagos"),
        SidebarItem(label="Tesorería — hoy", icon="lightning", url_name="compras:cxp_tesoreria", modulo="cxp_pagos"),
        SidebarItem(label="Dashboard CxP", icon="chart", url_name="compras:cxp_dashboard", modulo="cxp_pagos"),
        SidebarItem(label="Proveedores", icon="users", url_name="proveedores:lista", modulo="cxp_captura"),
    ],
),
```

(`build_sidebar` ya aplana el grupo a items sueltos si el usuario solo ve uno — comportamiento heredado, ver `sidebar.py:228-230`.)

---

## 9. Build Order (sigue paso por paso)

**Paso 1 — Modelos y migraciones.** Cambios §3.2-3.5 en `apps/compras/models.py` + validador §3.6. Migraciones: (a) schema aditiva, (b) data migration `origen=RECEPCION` para filas con orden. Verificar `makemigrations --check` limpio y que las migraciones de `compras` no toquen otras apps.

**Paso 2 — Permisos.** Módulos `cxp_captura`/`cxp_pagos` en el enum `Modulo`, data migration: grupo `tesoreria` + matriz §7 + `invalidate_permission_cache()`. Tests de matriz (calca `apps/core/tests/test_permissions_granular.py`).

**Paso 3 — Servicios + tests unitarios.** §4 completo, incluido el refactor de `registrar_pago_cxp` hacia `_recalcular_cxp` con tests de regresión (sobrepago, tolerancia 0.01, estados). Es el paso más denso: la lógica de dinero queda cerrada y probada ANTES de cualquier UI.

**Paso 4 — Captura de compra.** `CapturaCompraView` + template mobile-first + foto con `capture="environment"`. Registrar URLs de `compras` en `config/urls.py` si falta.

**Paso 5 — Lista + detalle + edición + NC.** `CxPListaView` (filtros), `CxPDetalleView` (timeline), `EditarCompraView`, modales de NC y cancelación.

**Paso 6 — Evidencias.** `EvidenciaView` con `FileResponse` + matriz de acceso §7. Test: usuario sin módulo → 403 en cada tipo.

**Paso 7 — Programación.** `ProgramacionView` con los dos paneles y validación de suma de programaciones ≤ saldo.

**Paso 8 — Tesorería.** `TesoreriaHoyView` + modal de pago (monto real editable + comprobante) + atrasados.

**Paso 9 — Dashboard + sidebar.** KPIs agregados (queries con `aggregate`, sin loops) + bloque §8.

**Paso 10 — ETL proveedores.** Anti-clobber + activación en el corrido + test de no-clobber (§5). Completar UI de proveedores/condiciones.

**Paso 11 — E2E + cierre.** Test del ciclo completo (success metric #1) vía client de Django. Suite completa del repo verde en `scripts/test_docker.ps1`. Smoke manual en preview con los 4 roles.

---

## 10. Tests (pytest — convenciones del repo, `apps/compras/tests/`)

| Archivo | Cubre |
|---|---|
| `test_compra_directa.py` | alta (folio, vencimiento = fecha + días crédito), validaciones, edición con/sin pagos, warning límite de crédito |
| `test_notas_credito.py` | NC baja saldo, NC > saldo rechazada, NC sobre PAGADO rechazada, cancelación recalcula, estados PARCIAL correctos |
| `test_pagos_programados.py` | programar ≤ saldo, suma de programaciones ≤ saldo, cancelar, ejecutar con monto real ≠ programado, liga 1:1 con PagoCxP |
| `test_recalculo_saldo.py` | la invariante `saldo = total - pagado - NC` bajo órdenes de operación mezcladas; regresión del `registrar_pago_cxp` refactorizado |
| `test_permisos_cxp.py` | matriz completa: almacenista 403 en programación/tesorería, tesorería 403 en captura, anónimo 302/403, evidencias por tipo |
| `test_evidencias.py` | FileResponse sirve el archivo correcto, extensión/tamaño inválidos rechazados en el form |
| `test_etl_proveedores_no_clobber.py` (en `apps/etl/tests/`) | editar `dias_credito` en EspritOS → extractor corre → NO lo pisa; proveedor nuevo → sí siembra desde PZ |
| `test_e2e_ciclo_cxp.py` | el ciclo completo del success metric #1 |

Regla del repo: correr TODO con `scripts/test_docker.ps1` antes de cerrar cada ola. Vistas de lista: assert de conteo de queries (`django_assert_num_queries` o `CaptureQueriesContext`, patrón existente en tests de pulso).

---

## 11. Reglas No Negociables

1. **No tocar inventario, WAC ni el flujo `OrdenCompra → Recepcion`.** La v1 vive 100% al nivel de CxP. `registrar_movimiento`, `apps/inventario` y `apps/catalogo` no se importan ni se modifican.
2. **Dinero = `Decimal` con `quantize(Decimal("0.01"), ROUND_HALF_UP)`.** Jamás float. Calca `services.py:96-115`.
3. **Un solo escritor de saldo/estado**: `_recalcular_cxp()`. Cualquier vista/servicio que necesite ajustar saldo pasa por ahí.
4. **Migraciones aditivas.** No borrar columnas ni tablas del módulo dormido; `orden` pasa a nullable, nada más se altera de lo existente.
5. **HTMX intercambia HTML, no JSON. Sin SPA. Clases `hm-*`.**
6. **Fotos/comprobantes solo vía `FileResponse` con check de permiso.** Nunca links directos a `/media/`. Validación de archivo en modelo Y form (10 MB, jpg/jpeg/png/webp/pdf).
7. **El ETL nunca pisa condiciones comerciales editadas en EspritOS** (`dias_credito`, `limite_credito`, `descuento_pronto_pago`). EspritOS es canónico para crédito; PZ solo siembra proveedores nuevos.
8. **Sin dependencias nuevas** (`pyproject.toml` intacto — por eso `FileField`, no `ImageField`).
9. **Permisos fail-closed**: toda vista nueva lleva `ModuloRequeridoMixin` o `@modulo_requerido`. Nada queda protegido "solo porque no está en el sidebar".
10. **Suite completa del repo verde antes de cerrar** — no solo los tests nuevos.

---

## 12. Documentación de la app (crear `apps/compras/CLAUDE.md`)

```markdown
# apps/compras — Compras y Cuentas por Pagar

Dos flujos sobre los mismos modelos:

1. **CxP directa (ACTIVO desde 09/2026)** — control de pagos a proveedores.
   Almacén captura compra (monto + foto de nota) → NC ajustan saldo →
   admin programa pagos (PagoProgramado) → tesorería ejecuta (PagoCxP con
   monto real + foto comprobante). `origen=DIRECTA`, sin OC.
2. **Flujo pesado (DORMIDO)** — OrdenCompra → Recepcion → inventario/WAC →
   CxP `origen=RECEPCION`. Existe desde Ola D, sin UI. NO tocar sin blueprint.

## Invariantes
- `saldo = max(0, monto_total - monto_pagado - monto_notas_credito)`.
  Único escritor: `services._recalcular_cxp()`.
- Vencimiento compra directa: `fecha_compra + proveedor.dias_credito`.
- NC: soft-cancel (`cancelada=True`), nunca DELETE. Monto ≤ saldo al crear.
- Sobrepago permitido; saldo clampa a 0; tolerancia PAGADO: 0.01.
- Dinero: Decimal quantize 0.01 ROUND_HALF_UP.

## Permisos (matriz RolePermission)
- `cxp_captura`: almacenista ver/crear/editar · tesoreria ver · admin todo.
- `cxp_pagos`: tesoreria ver/crear · admin todo (programación es de admin).
- Evidencias (fotos) SOLO vía `EvidenciaView` (FileResponse + permiso).

## Proveedores
- Catálogo sincronizado desde PZ (`apps/etl/extractors/proveedores.py`).
- EspritOS es CANÓNICO para dias_credito/limite_credito/descuento_pronto_pago:
  el ETL solo actualiza identidad en filas existentes.

## Tests
`pytest apps/compras/ apps/etl/tests/test_etl_proveedores_no_clobber.py`
— y la suite completa con `scripts/test_docker.ps1` antes de cerrar.
```

---

## 13. Skills para la fase de build

| Skill | Cuándo | Por qué |
|---|---|---|
| `builder-orquestacion` | arranque | plan de olas sobre este build order |
| `builder-rol-datos` / `builder-rol-backend` / `builder-rol-frontend` / `builder-rol-tests` | pasos 1-3 / 4-9 / 10-11 | roles semánticos de The Builder |
| `builder-rubrica` | apertura y cada milestone | derivar rúbrica de §1 success metrics |
| `webapp-testing` | paso 11 | smoke visual mobile (375px) de captura y tesorería |

NO usar `/frontend-design` ni `/ui-ux-pro-max`: el diseño hereda el sistema de EspritOS (regla §2).

---

## 14. Notas de despliegue

- Sin servicios nuevos: mismas imágenes, mismo compose (`docker-compose.prod.yml`). El volumen `espritos_prod_media` (`docker-compose.prod.yml:263`) ya monta `/app/media` en web y worker — las fotos persisten ahí.
- Post-deploy: `migrate` corre los seeds de permisos; verificar con `prod_readiness_check` (ya valida matriz no vacía).
- Alta de usuarios: asignar grupo `almacenista` a los chicos de almacén y `tesoreria` al usuario de tesorería vía Gestión → Usuarios (UI existente). Montse y Beto ya son admin.
- Primer corrido del ETL de proveedores en prod: revisar en el detalle de 2-3 proveedores que días de crédito/límite quedaron sembrados y editarlos donde PZ esté mal (PZ está en desorden — por eso EspritOS es canónico).
