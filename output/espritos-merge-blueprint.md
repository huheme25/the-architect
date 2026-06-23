# EspritOS Merge — Blueprint de Consolidación

> Generado por The Architect el 08/06/2026
> Tipo: Consolidación de 3 apps → 1 plataforma CRM
> Repo destino: `E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS`

---

## 1. Visión del Proyecto

### Qué es esto

Consolidar **pricing-hm** y **Prospectos HM** dentro de **EspritOS** para que las vendedoras (Andrea, Valeria, Daniela, Betty) trabajen en una sola app todo su día: captura de prospectos en campo, gestión de clientes/leads, clasificación de rentabilidad, bitácora de interacciones, agenda y capacitación.

### Por qué

- Las vendedoras pidieron explícitamente UNA sola app para todo su trabajo
- EspritOS hace downsizing: de CRM/ERP a CRM enfocado
- POS, remisiones y cotizaciones quedan congelados — el resolver de precios transaccional pierde consumidores
- pricing-hm ya no es solo herramienta de Beto — las vendedoras calificarán a sus clientes ahí

### Métrica de éxito

Las vendedoras abren EspritOS en la mañana y desde ahí: capturan prospectos, revisan su cartera, califican clientes, registran interacciones, ven su agenda, se capacitan. Sin abrir otra app.

---

## 2. Estrategia de Ramas y Despliegue

### REGLA FUNDAMENTAL: Todo se construye en rama, nada toca main hasta validar

```
main (producción — las vendedoras lo usan HOY)
  │
  └── feature/merge-pricing-rentabilidad  ← Fase 1
        │
        └── feature/merge-prospectos       ← Fase 2 (sale de Fase 1 ya mergeada)
```

### Flujo por fase

1. **Crear rama** desde `main` actualizado
2. **Desarrollar** toda la fase en la rama
3. **Correr TODOS los tests** (existentes + nuevos) — 0 fallos
4. **Deploy a ambiente staging** (docker-compose con `.env.staging`) en puerto diferente (8001)
5. **Beto valida** en staging: navega cada vista, prueba cada flujo
6. **Merge a main** solo con aprobación explícita de Beto
7. **Deploy a producción** (`docker compose -f docker-compose.windows.yml --env-file .env.production up -d`)
8. **Smoke test** en producción: login como vendedora, verificar que CRM+Agenda+Aprendizaje siguen funcionando

### Staging local

```bash
# En la rama feature, levantar staging en puerto 8001
docker compose -f docker-compose.windows.yml --env-file .env.staging up -d

# .env.staging es copia de .env.production con:
# DJANGO_PORT=8001
# DEBUG=False
# ALLOWED_HOSTS incluye localhost:8001
```

### Rollback

Si algo falla post-merge:
```bash
git revert --no-commit HEAD
docker compose -f docker-compose.windows.yml --env-file .env.production up -d --build
```

---

## 3. Tech Stack (sin cambios — mismo stack que EspritOS)

| Capa | Tecnología | Nota |
|------|-----------|------|
| Framework | Django 5.1 | Ya en EspritOS |
| Python | 3.12 | Ya en EspritOS |
| Frontend | HTMX 2 + Alpine.js 3 | Ya en EspritOS |
| CSS | Tailwind v4 + DaisyUI | Ya en EspritOS — pricing-hm usa Tailwind CDN, adaptar a build pipeline |
| DB | PostgreSQL 16 | Misma instancia `espritos_prod` |
| DB Canon | PostgreSQL `analitica_hm` (read-only) | EspritOS ya tiene `datos_hm` app con router |
| Cache | Redis 7 | Ya en EspritOS |
| Async | Celery 5 + Beat | Ya en EspritOS |
| Auth | django-allauth + MFA | Ya en EspritOS — reemplaza magic-link de pricing-hm e iron-session de Prospectos |
| Storage (fotos) | Django FileField + `MEDIA_ROOT` | Reemplaza Vercel Blob de Prospectos |
| Deploy | Docker + Cloudflare Tunnel | Ya en EspritOS |

**No se agrega NINGUNA dependencia nueva.** Todo lo que pricing-hm usa (pandas, pymysql, psycopg) ya está en EspritOS.

---

## 4. Estructura de Directorios (cambios)

```
apps/
  ...existentes (crm/, agenda/, aprendizaje/, core/, etc.)...
  │
  ├── rentabilidad/              ← NUEVO (de pricing-hm apps/rentabilidad/)
  │   ├── __init__.py
  │   ├── apps.py
  │   ├── models.py              # ClienteCualitativo, InteraccionCliente, RentParam, RentLista
  │   ├── admin.py
  │   ├── urls.py
  │   ├── views.py               # estado_clientes, ficha_cliente, movil, crear_interaccion, etc.
  │   ├── services/
  │   │   ├── __init__.py
  │   │   ├── rentabilidad.py    # Clasificación seminario (producto + cliente)
  │   │   ├── rolling.py         # Agregados 12M + tendencia 3M
  │   │   ├── canon.py           # Queries a analitica_hm (adaptar a datos_hm router)
  │   │   ├── bitacora.py        # Queries InteraccionCliente
  │   │   ├── sugerencias.py     # Motor de sugerencias cualitativas
  │   │   └── access.py          # RLS: es_dueno_cliente, filtrar_clientes
  │   ├── migrations/
  │   │   └── 0001_initial.py    # Generada por makemigrations
  │   └── tests/
  │       ├── test_models.py
  │       ├── test_rentabilidad.py
  │       ├── test_rolling.py
  │       ├── test_views.py
  │       └── test_access.py
  │
  ├── pricing_calc/              ← NUEVO (de pricing-hm apps/pricing/)
  │   ├── __init__.py
  │   ├── apps.py
  │   ├── models.py              # ProductoScore, ParametroCapa + legacy (MarkupLinea, etc.)
  │   ├── admin.py
  │   ├── urls.py
  │   ├── views.py               # calculadora, parametros (solo Beto)
  │   ├── services/
  │   │   ├── __init__.py
  │   │   └── scoring.py         # Capas multiplicativas, cálculo v2
  │   ├── migrations/
  │   │   └── 0001_initial.py
  │   └── tests/
  │       ├── test_models.py
  │       └── test_scoring.py
  │
  └── crm/                       ← MODIFICAR (agregar captura móvil)
      ├── ...existente...
      ├── views_captura.py       # NUEVO — form mobile-first (GPS + cámara + foto)
      └── templates/
          └── crm/
              └── captura.html   # NUEVO — template mobile-first

templates/
  ...existentes...
  ├── rentabilidad/              ← NUEVO (adaptar de pricing-hm)
  │   ├── estado_clientes.html
  │   ├── estado_productos.html
  │   ├── cliente_ficha.html
  │   ├── movil.html
  │   ├── oportunidades.html
  │   ├── parametros.html
  │   ├── _cmdk_results.html
  │   ├── _quick_nota.html
  │   ├── _score_preview.html
  │   └── sin_canon.html
  └── pricing_calc/              ← NUEVO (adaptar de pricing-hm)
      ├── calculadora_buscar.html
      ├── calculadora.html
      └── parametros.html
```

---

## 5. Modelo de Datos

### Modelos NUEVOS (Fase 1 — de pricing-hm)

**ClienteCualitativo** (tabla: `rentabilidad_clientecualitativo`)

| Campo | Tipo | Notas |
|-------|------|-------|
| `cliente_codigo` | CharField(32), PK | Código PZ del cliente — NO es FK a Cliente (puede existir en canon sin estar en EspritOS) |
| `pago` | SmallIntegerField, default=3 | 1-5: puntualidad de pago |
| `procesos` | SmallIntegerField, default=3 | 1-5: respeto a procesos |
| `friccion` | SmallIntegerField, default=3 | 1-5: facilidad de atención |
| `potencial` | SmallIntegerField, default=3 | 1-5: crecimiento/reputación |
| `notas` | TextField, blank | Notas libres |
| `actualizado_en` | DateTimeField, auto_now | Última edición |

Constraints: CheckConstraint en cada dimensión (1 ≤ valor ≤ 5).

**InteraccionCliente** (tabla: `rentabilidad_interaccioncliente`)

| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | AutoField, PK | |
| `cliente_codigo` | CharField(32), indexed | Código PZ del cliente |
| `fecha` | DateField, indexed | Fecha de la interacción |
| `tipo` | CharField(24), choices | charla, queja, pedido_especial, visita, promesa_pago, oportunidad, nota |
| `texto` | TextField, blank | Notas libres |
| `impacto_cualitativo` | JSONField, null/blank | Deltas opcionales: `{"dimension": delta}` |
| `seguimiento_fecha` | DateField, null/blank, indexed | Fecha límite de seguimiento |
| `seguimiento_resuelto` | BooleanField, default=False | Si el seguimiento se cerró |
| `autor` | FK(User), PROTECT | Quién registró |
| `creado_en` | DateTimeField, auto_now_add | |
| `actualizado_en` | DateTimeField, auto_now | |

Indexes: (cliente_codigo, -fecha), (seguimiento_fecha, seguimiento_resuelto).

**RentParam** (tabla: `rentabilidad_rentparam`)

| Campo | Tipo | Notas |
|-------|------|-------|
| `clave` | CharField(64), PK | Ej: `tesoro_min_margen`, `cli_ratio_patrimonio` |
| `valor` | DecimalField(12,4) | Valor numérico del parámetro |
| `descripcion` | CharField(255), blank | Descripción legible |

**RentLista** (tabla: `rentabilidad_rentlista`)

| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | AutoField, PK | |
| `categoria` | CharField(32), choices | `no_comisionable`, `marca_propia` |
| `patron` | CharField(128) | Patrón de texto a matchear |

Unique together: (categoria, patron).

**ProductoScore** (tabla: `pricing_calc_productoscore`)

| Campo | Tipo | Notas |
|-------|------|-------|
| `clave` | CharField(32), PK | Clave PZ del producto |
| `rotacion` | SmallIntegerField, default=2 | 1=alta, 3=baja |
| `competencia` | SmallIntegerField, default=2 | 1=commodity, 3=nicho |
| `costos_indirectos` | SmallIntegerField, default=2 | 1=bajos, 3=altos |
| `refrigeracion` | BooleanField, default=False | Flag capa multiplicativa |
| `congelacion` | BooleanField, default=False | |
| `lacteo` | BooleanField, default=False | |
| `carnico` | BooleanField, default=False | |
| `propio_hm` | BooleanField, default=False | |
| `mano_obra` | BooleanField, default=False | |
| `empaque_especial` | BooleanField, default=False | |
| `merma_idx` | SmallIntegerField, default=1 | 0=baja, 1=media, 2=tolerable, 3=alta |
| `precio_mercado_p5` | DecimalField(12,4), null | Precio mercado referencia |
| `precio_mercado_p3` | DecimalField(12,4), null | |
| `precio_mercado_p1` | DecimalField(12,4), null | |
| `notas` | TextField, blank | |
| `actualizado_en` | DateTimeField, auto_now | |

**ParametroCapa** (tabla: `pricing_calc_parametrocapa`)

| Campo | Tipo | Notas |
|-------|------|-------|
| `clave` | CharField(64), PK | Ej: `capa_refrigeracion`, `m_base_simple` |
| `valor` | DecimalField(10,6) | Decimal (0.02 = 2%) |
| `descripcion` | CharField(255), blank | |

### Modelos que NO se migran (se quedan en pricing-hm hasta retiro)

- `ProductoPZ`, `LineaPZ`, `SyncLog` — EspritOS ya tiene `catalogo.Producto` + ETL
- `Usuario`, `MagicLinkToken` — EspritOS usa django-allauth
- `Auditoria` (core pricing-hm) — EspritOS tiene `auditoria` propia
- Legacy v1: `MarkupLinea`, `Parametro`, `PalabraGranel`, `TipoDefault`, `ProductoOverride` — opcionales, migrar solo si la calculadora los necesita para tests de regresión v1 vs v2

---

## 6. Adaptaciones Técnicas Clave

### 6.1 Canon: pricing-hm queries → datos_hm router

**pricing-hm** usa `canon_connector.py` con queries SQL raw contra `analitica_hm`.
**EspritOS** ya tiene `apps/datos_hm/` con modelos unmanaged (`DimCliente`, `VentaDetalle`) y un router que desvía queries a la DB `datos_hm`.

**Decisión: mantener queries SQL raw** en `rentabilidad/services/canon.py` pero ejecutarlas a través del router de EspritOS:

```python
from django.db import connections

def _canon_cursor():
    """Cursor read-only al canon analitica_hm via datos_hm router."""
    return connections["datos_hm"].cursor()
```

Esto es más flexible que adaptar todo a Django ORM (las agregaciones con GROUP BY, benchmarks por canal, series mensuales son SQL puro que no vale la pena ORMizar).

**Fallback**: Si `DATOS_HM_URL` no está configurada, mostrar template `sin_canon.html` (ya existe en pricing-hm).

### 6.2 Access control: vendedor_canon

pricing-hm mapea usuarios a vendedores via `user.vendedor_canon` (campo en el modelo User de pricing-hm).

**Adaptación**: En EspritOS, el campo equivalente es el `vendedor_asignado` de `Cliente` + el sistema RLS de `core`. Para `es_dueno_cliente()`:

```python
def es_dueno_cliente(user, codigo_cliente):
    """True si el usuario es el vendedor asignado del cliente, o es staff."""
    if user.is_staff:
        return True
    # Buscar en dim_cliente del canon si el vendedor coincide
    from apps.datos_hm.models import DimCliente
    try:
        dim = DimCliente.objects.using("datos_hm").get(clave=codigo_cliente)
        return dim.vendedor == getattr(user, "codigo_vendedor", None)
    except DimCliente.DoesNotExist:
        return False
```

**Pendiente**: Confirmar con Beto cómo mapea `user` → `codigo_vendedor` en EspritOS. Si no existe el campo, agregarlo al modelo User o al Profile.

### 6.3 Templates: Tailwind CDN → Build pipeline

pricing-hm usa `<script src="https://cdn.tailwindcss.com">`.
EspritOS usa Tailwind v4 compilado + DaisyUI.

**Adaptación**: Reescribir templates usando clases DaisyUI donde aplique:
- `btn btn-primary` en lugar de clases Tailwind raw para botones
- `card` para contenedores
- `badge badge-success` para status badges
- `table` para tablas
- Mantener clases Tailwind raw para layout (grid, flex, spacing)

Esto es trabajo de template por template — no hay shortcut.

### 6.4 Permisos: staff_required para vistas de Beto

pricing-hm tiene un gap de permisos: las vistas de parámetros (`/parametros/guardar/`) no verifican permisos.

**Corrección en EspritOS**: Todas las vistas de tuning de parámetros y calculadora llevan `@staff_member_required` o se registran en `URL_TO_MODULO` con módulo `pricing_calc` y permisos `modificar` restringidos a admin.

### 6.5 Fotos de Prospectos: Vercel Blob → Django media

```python
# En settings/base.py (ya debería existir o agregar):
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"

# El form de captura guarda fotos en media/leads/
# Se sirven via whitenoise o nginx en producción
```

**Migración de fotos existentes**: Script one-shot que descarga desde Vercel Blob (usando las URLs absolutas que ya existen en los leads de EspritOS via `extra_fields.foto_url`) y las guarda localmente.

---

## 7. Build Order

### FASE 1: pricing-hm → EspritOS (rama `feature/merge-pricing-rentabilidad`)

**Step 1: Crear rama y scaffold de apps**

```bash
cd E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS
git checkout main && git pull
git checkout -b feature/merge-pricing-rentabilidad
```

Crear estructura de directorios:
- `apps/rentabilidad/` con `__init__.py`, `apps.py`, `admin.py`, `urls.py`, `views.py`, `models.py`
- `apps/rentabilidad/services/` con `__init__.py`
- `apps/rentabilidad/tests/` con `__init__.py`
- `apps/pricing_calc/` con misma estructura
- `templates/rentabilidad/`
- `templates/pricing_calc/`

Registrar en `config/settings/base.py` → `INSTALLED_APPS`:
```python
"apps.rentabilidad",
"apps.pricing_calc",
```

Registrar en `config/urls.py`:
```python
path("rentabilidad/", include("apps.rentabilidad.urls")),
path("pricing/", include("apps.pricing_calc.urls")),  # Reusa /pricing/ en lugar de /precios/ para evitar conflicto
```

Registrar en `apps/core/middleware.py` → `URL_TO_MODULO`:
```python
("/rentabilidad/", "rentabilidad"),
("/pricing/", "pricing_calc"),
```

**Entregable**: Apps registradas, EspritOS arranca sin errores, tests existentes pasan.

---

**Step 2: Migrar modelos de rentabilidad**

Copiar modelos de `pricing-hm/apps/rentabilidad/models.py` a `apps/rentabilidad/models.py`.

Adaptaciones:
- `autor` FK: apuntar a `settings.AUTH_USER_MODEL` (ya lo hace pricing-hm)
- NO agregar FKs a `clientes.Cliente` en `ClienteCualitativo` — usar `cliente_codigo` como CharField (pricing-hm lo diseñó así porque el cliente puede existir en canon sin estar en EspritOS)

```bash
python manage.py makemigrations rentabilidad
python manage.py migrate
```

**Entregable**: 4 tablas creadas en PostgreSQL, admin registrado para cada modelo.

---

**Step 3: Migrar servicios de rentabilidad**

Copiar de `pricing-hm/apps/rentabilidad/services/` a `apps/rentabilidad/services/`:

| Archivo origen | Archivo destino | Adaptaciones |
|----------------|-----------------|-------------|
| `rentabilidad.py` | `rentabilidad.py` | Cambiar imports de modelos a `apps.rentabilidad.models` |
| `rolling.py` | `rolling.py` | Sin cambios (lógica pura, no toca DB directamente) |
| `canon.py` | `canon.py` | Cambiar conexión: usar `connections["datos_hm"].cursor()` en lugar de conexión propia |
| `bitacora.py` | `bitacora.py` | Cambiar imports de modelos |
| `sugerencias.py` | `sugerencias.py` | Cambiar imports de modelos |
| `access.py` | `access.py` | **Reescribir**: adaptar `es_dueno_cliente()` y `filtrar_clientes()` al sistema RLS de EspritOS (ver sección 6.2) |

**Entregable**: Servicios importables, tests unitarios de lógica pura pasan (rolling, rentabilidad, sugerencias).

---

**Step 4: Migrar vistas de rentabilidad (vendedoras)**

Copiar vistas de `pricing-hm/apps/rentabilidad/views.py`. Migrar en este orden (por dependencias):

1. `estado_clientes` — Vista principal rolling 12M (default para vendedoras)
2. `estado_productos` — Productos rolling 12M
3. `ficha_cliente` — Detalle 3 columnas
4. `movil` — Vista mobile-first de cartera propia
5. `crear_interaccion` — POST para bitácora
6. `quick_nota_form` — Fragment HTMX
7. `resolver_seguimiento` — POST para cerrar seguimiento
8. `aplicar_sugerencia` — POST para aceptar sugerencia cualitativa
9. `preview_cualitativo` — Fragment HTMX de preview score
10. `guardar_cualitativo` — POST para guardar score
11. `cmdk_buscar` — Command palette HTMX
12. `oportunidades` — Vista legacy de oportunidades

Adaptaciones por vista:
- `@login_required` (ya existe en pricing-hm, mantener)
- Imports de servicios apuntan a `apps.rentabilidad.services.*`
- `request.user` funciona igual en ambos stacks
- Cache keys: prefijo `rent_` para evitar colisiones con cache de EspritOS

Copiar URLs de `pricing-hm/apps/rentabilidad/urls.py` a `apps/rentabilidad/urls.py`.

**Entregable**: Todas las URLs de rentabilidad responden sin error 500.

---

**Step 5: Adaptar templates de rentabilidad a UI EspritOS**

Para CADA template en `pricing-hm/templates/rentabilidad/`:

1. Cambiar `{% extends "base.html" %}` → `{% extends "base.html" %}` de EspritOS (verificar que el nombre coincida)
2. Cambiar `{% block content %}` al block correcto de EspritOS
3. Reemplazar Tailwind CDN clases → DaisyUI components donde aplique:
   - Botones: `class="bg-teal-600 text-white px-4 py-2 rounded"` → `class="btn btn-primary"`
   - Badges: aplicar `badge badge-{color}` DaisyUI
   - Tablas: `table table-zebra` DaisyUI
   - Cards: `card bg-base-100 shadow` DaisyUI
4. Verificar que `{% load static %}`, `{% load humanize %}` funcionan
5. Verificar que los `hx-*` attributes de HTMX apuntan a URLs correctas (namespace `rentabilidad:`)
6. Verificar que Alpine.js `x-data`, `x-show`, etc. funcionan (EspritOS ya carga Alpine 3)

Templates prioritarios (hacer primero):
- `estado_clientes.html` — es la vista principal
- `cliente_ficha.html` — es la más compleja (3 columnas)
- `movil.html` — es la que usan en campo

**Entregable**: Cada template renderiza correctamente en el navegador con la UI de EspritOS.

---

**Step 6: Migrar modelos y servicios de pricing_calc (Beto)**

Copiar modelos de `pricing-hm/apps/pricing/models.py` a `apps/pricing_calc/models.py`.

Copiar `pricing-hm/apps/pricing/services/scoring.py` a `apps/pricing_calc/services/scoring.py`.

Adaptaciones:
- Imports de modelos: `apps.pricing_calc.models`
- El scoring NO depende de `ProductoPZ` — usa `clave` como string
- Para la vista calculadora, necesita leer `costo_reemplazo` y `precio_1..5` del producto. Dos opciones:
  - **Opción A**: Leer de `catalogo.Producto` de EspritOS (si el ETL ya sincroniza esos campos)
  - **Opción B**: Leer de canon `analitica_hm` via query SQL
  - **Recomendación**: Opción A — `catalogo.Producto` ya tiene `costo_reemplazo`, `precio_1..5`

Copiar vistas de `pricing-hm/apps/pricing/views.py` a `apps/pricing_calc/views.py`.

Agregar `@staff_member_required` a TODAS las vistas de pricing_calc.

```bash
python manage.py makemigrations pricing_calc
python manage.py migrate
```

**Entregable**: Calculadora funcional en `/pricing/calculadora/`, solo accesible para staff.

---

**Step 7: Migrar datos de pricing-hm a EspritOS**

Script de migración one-shot (`scripts/migrate_pricing_data.py`):

```python
"""
Migra datos de pricing-hm PostgreSQL → EspritOS PostgreSQL.
Ejecutar UNA VEZ después de crear las migraciones.

Requiere acceso a la DB de pricing-hm:
  PRICING_HM_DB_URL=postgres://...@localhost:5433/pricing
"""
import os
import psycopg
from django.core.management.base import BaseCommand

TABLAS_RENTABILIDAD = [
    ("rentabilidad_rentparam", "rentabilidad_rentparam"),
    ("rentabilidad_rentlista", "rentabilidad_rentlista"),
    ("rentabilidad_clientecualitativo", "rentabilidad_clientecualitativo"),
    ("rentabilidad_interaccioncliente", "rentabilidad_interaccioncliente"),
]

TABLAS_PRICING = [
    ("pricing_productoscore", "pricing_calc_productoscore"),
    ("pricing_parametrocapa", "pricing_calc_parametrocapa"),
]
```

Pasos:
1. Conectar a DB pricing-hm (puerto 5433 del container `pricing-db`)
2. Para cada tabla: `SELECT *` de origen → `INSERT` en destino
3. Respetar PKs (cliente_codigo, clave)
4. Para `InteraccionCliente`: mapear `autor_id` al User equivalente de EspritOS (por email)
5. Log de filas migradas por tabla
6. **Idempotente**: usar `ON CONFLICT DO NOTHING` para re-ejecución segura

**Entregable**: Datos de pricing-hm disponibles en EspritOS. Verificar con queries manuales.

---

**Step 8: Actualizar navegación de EspritOS**

En el template base (sidebar/navbar):
- Agregar enlace "Rentabilidad" → `/rentabilidad/estado/` (visible para todos)
- Agregar enlace "Calculadora" → `/pricing/calculadora/` (visible solo para staff)
- Verificar que CRM, Agenda, Aprendizaje siguen en su lugar

En `apps/core/middleware.py` → registrar módulos en `URL_TO_MODULO`:
- `("/rentabilidad/", "rentabilidad")` — todos pueden ver, ownership gates en edición
- `("/pricing/", "pricing_calc")` — solo staff

Crear permisos en `apps/gestion/` o vía migration:
- `rentabilidad.ver` — todos los usuarios autenticados
- `rentabilidad.modificar` — vendedoras (editar cualitativo de su cartera)
- `pricing_calc.ver` — solo staff
- `pricing_calc.modificar` — solo staff

**Entregable**: Sidebar actualizado, permisos configurados.

---

**Step 9: Tests de Fase 1**

Migrar tests relevantes de pricing-hm:
- `tests/test_rentabilidad.py` → `apps/rentabilidad/tests/test_rentabilidad.py`
- `tests/test_rolling.py` → `apps/rentabilidad/tests/test_rolling.py`
- `tests/test_scoring.py` → `apps/pricing_calc/tests/test_scoring.py`

Adaptar imports a la nueva ubicación.

Tests NUEVOS a escribir:
- `test_access.py`: verificar que `es_dueno_cliente()` funciona con el sistema RLS de EspritOS
- `test_views.py`: smoke test de cada URL (200 OK para usuario autenticado, 302 para anónimo)
- `test_permisos.py`: verificar que vistas de pricing_calc devuelven 403 para no-staff
- `test_canon_integration.py`: verificar que queries al canon funcionan via `datos_hm` router

Correr suite completa:
```bash
python -m pytest --tb=short -q
# TODOS los tests de EspritOS (2700+) + los nuevos deben pasar
```

**Entregable**: 0 tests fallidos. Coverage de las apps nuevas ≥ 80%.

---

**Step 10: Validación Fase 1 — Checklist de Beto**

Antes de mergear a main, Beto verifica en staging (puerto 8001):

- [ ] Login como vendedora (Andrea) → sidebar muestra Rentabilidad
- [ ] `/rentabilidad/estado/` → tabla de clientes rolling 12M carga (o muestra "canon no disponible" si no hay DB)
- [ ] Filtrar por vendedor → solo muestra clientes de Andrea
- [ ] Click en un cliente → ficha 3 columnas (cuantitativo + cualitativo + bitácora)
- [ ] Editar cualitativo (pago=4, procesos=3) → guarda sin error
- [ ] Intentar editar cualitativo de cliente de OTRA vendedora → no permite
- [ ] Crear interacción (tipo: visita, texto: "Revisión de pedido") → aparece en bitácora
- [ ] `/rentabilidad/movil/` → solo cartera de Andrea, seguimientos pendientes
- [ ] Login como Beto → sidebar muestra Calculadora
- [ ] `/pricing/calculadora/` → búsqueda de producto funciona
- [ ] `/pricing/calculadora/<clave>/` → scoring + cálculo + comparación
- [ ] Guardar score → persiste
- [ ] Login como vendedora → `/pricing/calculadora/` → 403 Forbidden
- [ ] CRM, Agenda, Aprendizaje → siguen funcionando exactamente igual
- [ ] Crear un lead en CRM → funciona
- [ ] Ver agenda → funciona

**Solo con TODOS los checks ✓ → merge a main.**

---

### FASE 2: Prospectos → EspritOS CRM (rama `feature/merge-prospectos`)

**Step 11: Crear rama desde main (post-merge Fase 1)**

```bash
git checkout main && git pull
git checkout -b feature/merge-prospectos
```

---

**Step 12: Crear vista de captura móvil en CRM**

Nuevo archivo: `apps/crm/views_captura.py`

```python
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect
from django.contrib import messages

@login_required
def captura_prospecto(request):
    """Form mobile-first para captura de leads en campo."""
    if request.method == "POST":
        # Validar campos requeridos
        # Crear Lead directamente (sin API externa)
        # Guardar foto si existe
        # Redirect con mensaje de éxito
        ...
    return render(request, "crm/captura.html")
```

Nuevo URL en `apps/crm/urls.py`:
```python
path("captura/", views_captura.captura_prospecto, name="captura"),
```

**Campos del form** (mapeados al modelo Lead existente):

| Campo Prospectos | Campo Lead EspritOS | Notas |
|-----------------|-------------------|-------|
| nombreTienda | company | |
| nombreContacto | first_name + last_name | Separar por primer espacio |
| telefono | phone | |
| email | email | |
| direccion | extra_fields.direccion_capturada | |
| latitud/longitud | extra_fields.latitud/longitud | |
| fotoUrl | extra_fields.foto_url (ahora path local) | |
| productosInteres | extra_fields.productos_interes | |
| comentarios | extra_fields.comentarios_captura | |
| — | source = "CAPTURA_MOVIL" | Nuevo valor de source |
| — | owner = request.user o default | Asignar al vendedor que captura |
| — | status = "NEW" | |

---

**Step 13: Template de captura mobile-first**

Archivo: `templates/crm/captura.html`

Funcionalidades clave (client-side, no requieren React):

**GPS** (Alpine.js):
```html
<div x-data="{ lat: null, lng: null, capturing: false }">
  <button @click="capturing=true; navigator.geolocation.getCurrentPosition(
    pos => { lat=pos.coords.latitude; lng=pos.coords.longitude; capturing=false },
    err => { capturing=false },
    { enableHighAccuracy: true, timeout: 10000, maximumAge: 0 }
  )" class="btn btn-outline">
    <span x-show="!lat">Capturar ubicacion</span>
    <span x-show="lat" x-cloak>Ubicacion capturada</span>
  </button>
  <input type="hidden" name="latitud" :value="lat">
  <input type="hidden" name="longitud" :value="lng">
</div>
```

**Camara** (HTML nativo):
```html
<input type="file" name="foto" accept="image/jpeg,image/png,image/webp"
       capture="environment" class="file-input file-input-bordered w-full">
```

**Preview de foto** (Alpine.js):
```html
<div x-data="{ preview: null }">
  <input type="file" name="foto" accept="image/*" capture="environment"
         @change="preview = URL.createObjectURL($event.target.files[0])">
  <img x-show="preview" :src="preview" class="mt-2 rounded max-h-48" x-cloak>
</div>
```

**Layout mobile-first**:
- `max-w-md mx-auto` para centrar en desktop
- `px-4` padding lateral
- Inputs `h-12` para touch targets ≥ 48px
- Submit button sticky al fondo: `fixed bottom-0 inset-x-0 p-4 bg-base-100 shadow-up`
- Form reset al guardar exitosamente (redirect + message)

---

**Step 14: Storage de fotos**

En `config/settings/base.py`, verificar:
```python
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

En `config/urls.py` (solo development):
```python
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

En producción: whitenoise sirve `MEDIA_ROOT` o configurar nginx via Cloudflare Tunnel.

La vista guarda la foto:
```python
if foto := request.FILES.get("foto"):
    from django.core.files.storage import default_storage
    path = default_storage.save(f"leads/{foto.name}", foto)
    extra_fields["foto_url"] = default_storage.url(path)
```

---

**Step 15: Migrar datos de Prospectos (Neon) → EspritOS**

Script one-shot (`scripts/migrate_prospectos_data.py`):

1. Conectar a Neon Postgres (obtener `DATABASE_URL` de Vercel dashboard)
2. `SELECT * FROM leads` (tabla Prisma)
3. Para cada lead:
   - Buscar si ya existe en EspritOS via `extra_fields__prospectos_id` (idempotencia)
   - Si no existe: crear Lead con campos mapeados (Step 12)
   - Si existe: skip (ya fue sincronizado via ingest API)
4. Descargar fotos de Vercel Blob → `media/leads/` (via las URLs absolutas)
5. Log de leads migrados / skipped / errores

**Entregable**: Todos los leads de Prospectos disponibles en EspritOS CRM.

---

**Step 16: Simplificar endpoint de ingest**

El endpoint `/api/crm/leads/ingest/` ya no recibirá tráfico de Prospectos (la captura es interna).

Opciones:
- **Opción A**: Dejarlo como está (no rompe nada, puede servir para integraciones futuras)
- **Opción B**: Marcarlo como deprecated en docstring

**Recomendación**: Opción A — dejarlo. Zero costo de mantenimiento, potencial valor futuro.

---

**Step 17: Tests de Fase 2**

Tests nuevos:
- `apps/crm/tests/test_captura.py`:
  - GET `/crm/captura/` → 200 para autenticado, 302 para anónimo
  - POST con datos válidos → crea Lead, redirect con message
  - POST con datos inválidos → re-render con errores
  - POST con foto → foto guardada en media/leads/
  - Lead creado tiene `source="CAPTURA_MOVIL"` y `owner=request.user`
- Verificar que tests existentes de CRM siguen pasando

```bash
python -m pytest --tb=short -q
# 0 fallos
```

**Entregable**: Tests pasan, coverage de captura ≥ 90%.

---

**Step 18: Validación Fase 2 — Checklist de Beto**

En staging:

- [ ] Login como vendedora → sidebar muestra "Captura" (o accesible desde CRM)
- [ ] `/crm/captura/` → form mobile-first
- [ ] Llenar form completo (nombre tienda, contacto, dirección, productos)
- [ ] Capturar GPS → muestra "Ubicacion capturada"
- [ ] Tomar foto con cámara → preview aparece
- [ ] Guardar → mensaje de éxito, form se limpia
- [ ] Ir a CRM → el nuevo lead aparece en la lista con source="CAPTURA_MOVIL"
- [ ] El lead tiene foto, GPS, productos en sus campos
- [ ] Abrir desde celular (Chrome mobile) → form es usable con una mano
- [ ] Toda la funcionalidad existente de CRM sigue igual
- [ ] Rentabilidad (Fase 1) sigue funcionando
- [ ] Agenda y Aprendizaje siguen funcionando

**Solo con TODOS los checks ✓ → merge a main.**

---

### FASE 3: Limpieza (rama `feature/cleanup-downsizing`)

**Step 19: Congelar apps sin consumidores activos**

En `config/settings/base.py`, agregar comentario de freeze:

```python
# --- APPS CONGELADAS (downsizing 06/2026) ---
# Las siguientes apps permanecen instaladas para preservar migraciones
# y datos históricos. NO agregar features, NO refactorizar.
# Reactivar solo con autorización explícita de Beto.
"apps.pos",           # Congelada: sin POS por el momento
"apps.remisiones",    # Congelada: depende de POS
"apps.ventas_mayoreo", # Congelada: depende de POS
"apps.cobranza",      # Congelada: sin flujo activo
# ... (las 18 standby ya listadas)
```

**NO desinstalar apps** — las migraciones deben existir para la integridad de la DB.

---

**Step 20: Actualizar sidebar/navegación definitiva**

Sidebar de EspritOS post-merge:

```
┌─────────────────────────┐
│  EspritOS               │
├─────────────────────────┤
│  CRM                    │  ← /crm/
│    └─ Captura           │  ← /crm/captura/     (mobile icon)
│  Rentabilidad           │  ← /rentabilidad/estado/
│    └─ Móvil             │  ← /rentabilidad/movil/ (mobile icon)
│  Agenda                 │  ← /agenda/
│  Aprendizaje            │  ← /aprendizaje/
├─────────────────────────┤
│  Solo admin:            │
│  Calculadora            │  ← /pricing/calculadora/
│  Configuración          │  ← /configuracion/    (si existe)
├─────────────────────────┤
│  Perfil / Logout        │
└─────────────────────────┘
```

---

**Step 21: Actualizar CLAUDE.md de EspritOS**

Reflejar la nueva realidad: 5 apps activas (CRM, Rentabilidad, Agenda, Aprendizaje, pricing_calc), captura móvil integrada, pricing-hm y Prospectos absorbidos.

---

**Step 22: Retirar apps externas**

Después de validar que TODO funciona en producción durante 1 semana:

1. **pricing-hm**: Detener container `pricing-{web,db,redis}`. NO borrar — dejar imagen Docker como respaldo.
2. **Prospectos**: Quitar deploy automático de Vercel. Mantener repo como archivo.
3. **DNS**: Redirigir `pricing.cremeriahm.com` → `espritos.app/pricing/calculadora/` (o remover)
4. **Prospectos URL**: Redirigir `prospectos-hm.vercel.app` → `espritos.app/crm/captura/` (o remover)

---

## 8. Autenticación y Permisos Post-Merge

### Roles

| Rol | Acceso |
|-----|--------|
| **Vendedora** (Andrea, Valeria, Daniela, Betty) | CRM (todo), Captura, Rentabilidad (ver todo, editar cualitativo de SU cartera, crear interacciones), Agenda, Aprendizaje |
| **Admin/Supervisor** (Beto, Jamie) | Todo lo anterior + Calculadora + Parámetros rentabilidad + Parámetros pricing + Admin Django |

### Mapeo usuario → vendedor_canon

Para que `es_dueno_cliente()` funcione, cada User de EspritOS necesita un campo o atributo que lo vincule a su código de vendedor en el canon.

**Verificar**: ¿Existe `User.codigo_vendedor` o similar en EspritOS? Si no:
- Agregar campo `codigo_vendedor` al modelo User (o al Profile si existe)
- Poblar para cada vendedora: Andrea="08", Valeria="09", Daniela="10", Betty="ABAR" (o el código que corresponda)
- Este campo lo usa `access.py` para filtrar cartera

---

## 9. Variables de Entorno (cambios)

| Variable | Descripción | Dónde |
|----------|-------------|-------|
| `DATOS_HM_URL` | Ya existente — conexión a analitica_hm canon | `.env.production` |
| `MEDIA_ROOT` | Path para fotos de leads (si no existe) | `.env.production` |

**No se necesitan variables nuevas.** pricing-hm's canon queries reusarán la conexión `datos_hm` ya configurada.

---

## 10. Testing Strategy

### Tests migrados de pricing-hm

| Suite | Archivos | Qué valida |
|-------|----------|-----------|
| Rentabilidad pura | `test_rentabilidad.py` | Clasificación Tesoro/Imán/Trampa + Patrimonio/Vitrina/Aspiradora/Fantasma |
| Rolling | `test_rolling.py` | Agregados 12M, tendencias 3M, pesos ponderados |
| Scoring v2 | `test_scoring.py` | Capas multiplicativas, cascada p1-p5, 3D score |
| Sugerencias | `test_sugerencias.py` | Reglas: quejas→fricción↓, visitas→procesos↑ |
| Bitácora | `test_bitacora.py` | Queries de interacciones, conteo por tipo, seguimientos pendientes |

### Tests nuevos para el merge

| Suite | Archivos | Qué valida |
|-------|----------|-----------|
| Access RLS | `test_access.py` | Ownership de clientes, filtrado de cartera, staff bypass |
| Views smoke | `test_views.py` | Cada URL responde 200/302/403 según rol |
| Captura | `test_captura.py` | Flujo completo: form → Lead → foto → GPS |
| Canon integration | `test_canon.py` | Queries al canon via datos_hm router (puede necesitar mock si canon no disponible en CI) |

### Correr tests

```bash
# Tests de las apps nuevas
python -m pytest apps/rentabilidad/ apps/pricing_calc/ -v

# Suite completa (verificar 0 regresiones)
python -m pytest --tb=short -q

# Con coverage
python -m pytest --cov=apps/rentabilidad --cov=apps/pricing_calc --cov-report=term-missing
```

---

## 11. Skills para la Fase de Build

| Skill | Cuándo usar | Para qué |
|-------|-------------|----------|
| `/code-review` | Después de cada Step, antes de commit | Revisar calidad del código migrado |
| `/test-driven-development` | Steps 9, 17 | Escribir tests antes de las adaptaciones complejas |
| `/verification-before-completion` | Steps 10, 18 | Checklist formal de validación |
| `/systematic-debugging` | Si algo falla post-migración | Debug metódico de errores de integración |
| `/frontend-design` | Step 5 (templates) | Adaptar templates de pricing-hm a UI EspritOS con calidad |

---

## 12. CLAUDE.md Actualizado para EspritOS (Sección relevante a agregar)

```markdown
## Apps Activas (post-merge 06/2026)

| App | Usuarios | Función |
|-----|----------|---------|
| `crm` | Vendedoras + Admins | Gestión de leads, clientes, pipeline, captura móvil |
| `rentabilidad` | Vendedoras + Admins | Clasificación clientes (Patrimonio/Vitrina/Aspiradora/Fantasma), cualitativo, bitácora, oportunidades, vista móvil |
| `pricing_calc` | Solo Beto/Admin | Calculadora de precios (scoring 3D + capas multiplicativas) |
| `agenda` | Vendedoras | Calendario visitas/llamadas |
| `aprendizaje` | Vendedoras | Capacitación interna |
| `core` | Infraestructura | RLS, middleware, auth, shared services |
| `datos_hm` | Infraestructura | Read-only bridge a analitica_hm canon |
| `etl` | Infraestructura | Sync de Punto Zero MySQL |
| `auditoria` | Infraestructura | Audit log append-only |
| `gestion` | Admin | Usuarios, roles, permisos |

## Apps Congeladas (downsizing 06/2026)

`pos`, `remisiones`, `ventas_mayoreo`, `precios` (resolver transaccional), `inventario`, `compras`, `cobranza`, `devoluciones`, `cfdi`, `promociones`, `gastos`, `comisiones`, `lealtad`, `rutero`, `reportes`, `catalogo` (standby — datos se usan via ETL), `clientes` (standby — datos vienen de canon), `proveedores`, `chat_ia` (frozen desde 14/05), `tenants`, `platform_config`, `offline`, `approvals`, `notifications`

**Regla**: CERO features nuevas en apps congeladas. Solo fixes de migración si `core` los requiere.

## Apps Absorbidas

- **pricing-hm** → `apps/rentabilidad/` + `apps/pricing_calc/` (merge 06/2026)
- **Prospectos HM** → `apps/crm/views_captura.py` + template captura.html (merge 06/2026)

## Filtro de Decisión

Antes de cualquier trabajo, preguntar:
"¿Esto aumenta la adopción del toolkit (CRM + Rentabilidad + Agenda + Aprendizaje) por Andrea, Valeria, Daniela o Betty?"

- SÍ → Proceder
- NO, pero es infra/seguridad crítica → Proceder
- NO → STOP, preguntar a Beto
```

---

## 13. Reglas No Negociables

1. **NUNCA mergear a main sin validación de Beto en staging.** Cada fase se valida por separado.
2. **NUNCA modificar apps congeladas** como efecto colateral del merge. Si una migración requiere cambios en `pos` o `remisiones`, hacer el fix MÍNIMO de compatibilidad.
3. **NUNCA importar entre apps** excepto `core` — la regla de aislamiento de EspritOS se extiende a las apps nuevas. `rentabilidad` NO importa de `crm`. `pricing_calc` NO importa de `rentabilidad`. Ambas leen del canon via `datos_hm`.
4. **NUNCA exponer datos sin filtro RLS.** Las vistas de rentabilidad DEBEN respetar ownership (vendedora solo edita su cartera).
5. **NUNCA romper tests existentes.** Los 2700+ tests de EspritOS deben pasar después de cada Step. Si un test falla, arreglar antes de continuar.
6. **Fotos se guardan localmente** en `MEDIA_ROOT`, no en servicios externos. Sin dependencias nuevas.
7. **Templates adaptados a DaisyUI** — no mezclar Tailwind CDN con el build pipeline de EspritOS.
8. **El merge de datos es idempotente** — los scripts de migración usan `ON CONFLICT DO NOTHING` o verificación de existencia.
9. **pricing-hm y Prospectos NO se apagan hasta 1 semana después** de validar en producción. Periodo de gracia.
10. **Commits atómicos por Step** — cada Step es un commit con mensaje descriptivo. Si algo falla, revert limpio.

---

## 14. Estimación de Esfuerzo

| Fase | Steps | Esfuerzo estimado | Riesgo |
|------|-------|-------------------|--------|
| Fase 1 (pricing-hm → rentabilidad + pricing_calc) | Steps 1-10 | 2-3 sesiones de trabajo | Medio — templates requieren adaptación manual |
| Fase 2 (Prospectos → captura móvil) | Steps 11-18 | 1-2 sesiones | Bajo — funcionalidad acotada, Lead model ya existe |
| Fase 3 (limpieza) | Steps 19-22 | 1 sesión | Bajo — cambios cosméticos |

**Total**: ~4-6 sesiones de Claude Code, con validación de Beto entre cada fase.
