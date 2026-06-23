# EspritOS — Blueprint de Construcción Pre-Piloto

> Generado por The Architect el **2026-04-22**
> **Piloto objetivo:** viernes **2026-05-01** (9 días naturales, 8 días hábiles de construcción + 1 de validación)
> **Cliente:** Beto / Cremería HM (Tonalá, Jalisco)
> **Repos:** `~/espritos/` (WSL2 prod) / `E:\ClaudeWorks\proyectos\EspritOS\` (Windows dev) / `huheme25/espritos` (GitHub)
> **Estado pre-construcción:** V5.0 desplegada en prod, 2,227 tests verde, tag `v5.0.0`, 6 containers healthy en `192.168.0.152`, expuesto en `https://espritos.app` vía Cloudflare Tunnel

---

## 0. Cómo usar este blueprint

Este documento es **autosuficiente**. Una sesión nueva de Claude Code en `~/espritos/`, sin contexto previo, debe poder ejecutar todo el sprint pre-piloto siguiéndolo paso a paso.

**Reglas de oro al construir este sprint:**

1. **No saltes pasos del Build Order.** Las dependencias están explícitas — saltar uno provoca rework.
2. **Cada paso termina con commit verificable.** Tests verde + smoke manual documentado en el body del commit.
3. **Formato de commit:** `[PILOTO][O<ola>.<step>] descripción`. Body incluye tests añadidos, total tests pasando, y verificación manual hecha.
4. **Si un paso falla a la mitad, NO empieces el siguiente.** Pausa y reporta. La integridad del piloto depende del orden.
5. **Cero alcance fuera del sprint.** Si descubres bug ajeno mientras trabajas: documenta en `docs/piloto/findings/` y sigue. Excepción: bug bloqueador del paso actual → escala a Beto.
6. **Tests obligatorios mínimos por step (definidos abajo).** Calidad > cantidad. No infles números con tests basura.
7. **Verifica que tests previos siguen verde** antes de cada commit: `pytest --tb=short -q`. Cualquier regresión se arregla antes de commitear el step.
8. **Stack no se negocia.** Django 5.1 + Postgres 16 + HTMX 2 + Alpine 3 + Tailwind v4 + DaisyUI + Celery + Redis + Claude API. Nada de React/Vue/Next/etc.
9. **Español en comunicación, inglés en código, comentarios español cuando no sea obvio.**
10. **Cero deploy productivo sin Beto presente.** Especialmente la carga histórica de StockLedgerEntry (5.1M filas) y la prueba de hardware POS en tienda.

**Ground truth de patrones existentes:** Antes de crear un template nuevo, lee uno equivalente del módulo más cercano (ej: para CFDI lista, copia el patrón de `templates/remisiones/lista.html`). El estilo visual y la estructura HTMX ya están establecidos — replícalos, no inventes.

---

## 1. Objetivo del sprint

Cerrar todos los gaps que destapó la **auditoría pre-piloto del 2026-04-22** para que **5 usuarios reales** (andrea, betty, carlos, maggi, jamie) más Bernardo arranquen el viernes 1/05 usando EspritOS como sistema primario en lugar de PuntoZero.

**Definición de "listo para piloto":**
- ✅ Todo lo que el equipo ve en el sidebar **funciona**, o está oculto.
- ✅ El cajero puede emitir y consultar facturas (CFDI) desde la UI sin tocar Django admin.
- ✅ El cajero puede recibir mercancía y crear OCs desde la UI.
- ✅ Hardware POS opera sin "Ctrl+P" — impresora ESC/POS, scanner, báscula y cajón responden a comandos del sistema.
- ✅ Los 5 usuarios tienen rol + sucursal + password confirmados, documentados, y entregados con manual.
- ✅ StockLedgerEntry histórico cargado (5.1M filas) → reportes históricos consistentes con snapshot.
- ✅ Lealtad y Promociones tienen UI de uso real (no esqueleto), Rutero oculto.
- ✅ CRM V5 con Kanban drag-drop, vistas de Leads, conversion y forecast.
- ✅ Offline + PWA validados manualmente en prod con Chrome real (persist() concedido).
- ✅ Facturama timbrando sin error 401.
- ✅ UAT contigo (Beto) + ensayo Bernardo completados sin bloqueos críticos abiertos.

**Lo que NO entra en este sprint** (deferido post-piloto, NO tocar):
- 6 decisiones SaaS (Carta Porte 3.1, vertical 2do tenant, goal tenants 12m, pricing, marketplace, hosting)
- Sprint 1 V5 (CFDI MX avanzado) y siguientes — son del Blueprint V5 SaaS
- Paso 13 PC servidor dedicada
- Paso 40 cutover CRM-ERP v4
- E2E Playwright formal, sklearn/Prophet real, SMTP SafeAction real
- Expandir Rutero (se oculta)

---

## 2. Snapshot del estado actual (2026-04-22)

### Lo que SÍ está completo y validado

| Módulo | Estado | Notas |
|---|---|---|
| Dashboard por rol | ✅ | Renderiza bien |
| Clientes (486) | ✅ | Cargados |
| Productos (2,884) | ✅ | Cargados |
| Proveedores (132) | ✅ | Cargados |
| Precios (cascada 4 niveles) | ✅ | |
| Inventario (1,648 bins, 9 templates, kardex/ajustes/lotes/FEFO) | ✅ | Snapshot cubre POS |
| POS | ✅ | 302 al flujo real |
| Remisiones | ✅ | Listas + crear + detalle |
| Cobranza | ✅ | Aging + detalles + pagos |
| Devoluciones | ✅ | |
| Chat IA | ✅ | 114+ tests, validado E2E en prod |
| CRM + Pipeline Kanban | ✅ | seed_crm_v5 corrido (drag-drop pendiente) |
| Reportes | ✅ | Index + PDFs + recientes |
| Gestión usuarios | ✅ | CRUD con wizard + 20 tests |
| Configuración | ✅ | |
| Aprobaciones | ✅ | Inbox + detalle, 7 tests |
| Offline + PWA | ✅ | 113 tests + E2E (pendiente validar persist en prod) |
| Gastos | 🟢 OK básico | 4 models / 9 views / 6 templates |
| Comisiones | 🟢 OK | 6 models / 4 views / 4 templates, nightly S9.2 corrió |
| Cotizaciones (ventas_mayoreo) | 🟡 Aceptable | Probar con vendedor en UAT |

### Lo que falta y este sprint cierra

| Bloque | Item | Prioridad |
|---|---|---|
| **🔴 P0 bloqueadores** | CFDI UI (lista + detalle) | Crítica |
| | Compras UI (lista_ordenes + detalle_orden + _tabla_ordenes + formularios) | Crítica |
| | Setup users piloto (rol + sucursal + puede_ver_ambas confirmados) | Crítica |
| | Deploy a prod de commits offline + reportes pendientes | Crítica |
| | Validar `persist()` en prod con Chrome real + PWA instalada | Crítica |
| **🟡 P1 calidad** | Hardware POS bridge (ESC/POS + scanner + báscula + cajón) | Alta |
| | Carga histórica StockLedgerEntry 5.1M | Alta |
| | Fix Facturama 401 (credenciales sandbox o cuenta nueva) | Alta |
| **🟢 P2 módulos esqueleto** | Lealtad UI completa (saldo monedero + historial + canje en POS) | Media |
| | Promociones UI (CRUD + activar/desactivar) | Media |
| | Kanban drag-drop con Sortable.js | Media |
| | Vistas CRM V5 (Leads list + conversion funnel + forecast dashboard) | Media |
| | Ocultar Rutero del sidebar | Trivial |
| **🔵 P3 data quality** | Decisión código 142F "CARNES FRIAS A GRANEL" con Humberto | Baja |
| | Purga 1.94M detalles huérfanos legacy | Baja |
| **🟣 Validación** | UAT 6 flujos con Beto | Crítica |
| | Ensayo Bernardo en prod (5 ventas + 1 CFDI + corte caja + CxC) | Crítica |
| | Manuales .docx actualizados | Alta |

---

## 3. División de trabajo

| Quién | Tareas |
|---|---|
| **Builder Claude** (sesión en `~/espritos/`) | Todos los pasos de código: templates CFDI, templates Compras, hardware bridge FastAPI, expansión Lealtad/Promociones, Kanban drag-drop, vistas CRM, scripts de setup de users, comando ETL StockLedger, ocultar Rutero, deploy a prod (commits ya en repo). |
| **Beto** | (1) Resetear passwords de los 5 users + decidir rol/sucursal de cada uno. (2) Decisión 142F con Humberto. (3) Conseguir credenciales Facturama válidas (renovar 401 o cuenta nueva). (4) Disparar carga histórica StockLedger en ventana nocturna (acompañado por builder). (5) UAT 3h. (6) Estar presente en prueba de hardware POS en tienda. |
| **Bernardo** | Ensayo de 2h en prod (5 ventas, 1 CFDI, corte caja, CxC). Validación visual de manuales actualizados. |

**Hardware POS confirmado disponible en tienda** (validado por Beto 2026-04-22):
- ✅ PCs de cobro
- ✅ Escáner para códigos
- ✅ Impresoras de tickets ESC/POS
- ✅ Cajones de dinero con apertura automática por comando del sistema

---

## 4. Build Order — 4 Olas / 17 steps

**Calendario base** (días naturales — Cremería HM opera L-S, sin bloqueos por festivos MX en este rango):

| Día | Fecha | Ola | Foco |
|---|---|---|---|
| Mié | 22/04 | — | Auditoría + plan + blueprint (HOY) |
| Jue | 23/04 | **O1** | CFDI templates + deploy offline a prod |
| Vie | 24/04 | **O1** | Compras templates + setup users |
| Sáb | 25/04 | **O2** | Hardware POS bridge día 1 + Facturama fix |
| Dom | 26/04 | **O2** | Hardware POS bridge día 2 + carga StockLedger nocturna |
| Lun | 27/04 | **O3** | Lealtad UI + ocultar Rutero |
| Mar | 28/04 | **O3** | Promociones UI + Kanban drag-drop |
| Mié | 29/04 | **O3** | Vistas CRM V5 + data quality (142F + huérfanos) |
| Jue | 30/04 | **O4** | UAT Beto (mañana) + ensayo Bernardo (tarde) + manuales + buffer |
| Vie | 01/05 | 🚀 | **PILOTO arranca** — guardia primeras 4h |

**Branching:**
- Cada ola es una rama: `pilot/ola-1-bloqueadores`, `pilot/ola-2-hardware-data`, `pilot/ola-3-modulos`, `pilot/ola-4-validacion`
- Mergea cada ola a `main` cuando cierre con tests verde + DoD cumplido
- No mezclar olas en una sola rama

---

### OLA 1 — Bloqueadores P0 (Jue 23 + Vie 24, ~13h efectivas)

#### Step O1.1 — CFDI templates: lista + detalle (3-4h)

**Objetivo:** Eliminar el `TemplateDoesNotExist` al clickear "Facturación" en el sidebar. Permitir consultar y emitir facturas desde la UI.

**Archivos a crear:**

```
apps/cfdi/
├── templates/cfdi/
│   ├── lista.html                ← NUEVO
│   ├── detalle.html              ← NUEVO
│   ├── _tabla_cfdi.html          ← NUEVO (parcial HTMX para paginación)
│   ├── _form_emitir.html         ← NUEVO (parcial para emitir factura individual)
│   └── cancelar_modal.html       ← YA EXISTE, no tocar
└── tests/test_views_ui.py        ← NUEVO (mínimo 6 tests)
```

**Patrón a seguir (autoritativo):** copia estructura de `templates/remisiones/lista.html` y `templates/remisiones/detalle.html`. Mismos componentes DaisyUI, mismos breakpoints, mismo manejo de filtros HTMX.

**Funcionalidad mínima `lista.html`:**
- Tabla de CFDI con columnas: folio, UUID (truncado a 8), serie, cliente, RFC, total, fecha emisión, estado (vigente/cancelado), acciones (ver, descargar XML, descargar PDF, cancelar si vigente)
- Filtros: rango de fechas, cliente (autocomplete), estado, sucursal (si `puede_ver_ambas`)
- Búsqueda por folio o UUID (input con `hx-trigger="keyup changed delay:300ms"`)
- Botón "Emitir factura" → abre `_form_emitir.html` en modal (selecciona remisión a facturar o cliente para factura standalone)
- Paginación HTMX (50 por página)
- Badge "Sandbox" si `FACTURAMA_ENV=sandbox`

**Funcionalidad mínima `detalle.html`:**
- Header: folio fiscal completo, UUID, fecha timbrado, estado, monto total
- Bloques: emisor, receptor, conceptos (tabla), impuestos desglosados, complemento de pago si aplica
- Acciones: descargar XML, descargar PDF, cancelar (si vigente, abre `cancelar_modal.html` existente con motivos 01-04 SAT), reenviar por email
- Histórico de eventos (timbrado, intentos cancelación, descargas)

**Tests mínimos (`test_views_ui.py`):**
1. `test_lista_renderiza_para_admin` — admin ve todas las CFDI de ambas sucursales
2. `test_lista_filtra_por_sucursal_para_cajero` — cajero solo ve CFDI de su sucursal
3. `test_detalle_renderiza_cfdi_vigente` — muestra acciones cancelar/descargar
4. `test_detalle_renderiza_cfdi_cancelado` — oculta acción cancelar
5. `test_emitir_form_valida_remision_existe` — error si remisión no existe o ya facturada
6. `test_lista_busqueda_por_folio_devuelve_html_parcial` — HTMX devuelve `_tabla_cfdi.html` con `hx-target` correcto

**DoD del step:**
- [ ] `pytest apps/cfdi/tests/test_views_ui.py` → 6 verde, 0 rojo
- [ ] Click en "Facturación" del sidebar abre `lista.html` sin excepciones (smoke manual con `runserver`)
- [ ] Filtrar por fecha + cliente devuelve resultado correcto
- [ ] Abrir detalle de CFDI existente muestra todos los bloques
- [ ] Commit `[PILOTO][O1.1] CFDI UI: lista + detalle`

---

#### Step O1.2 — Deploy a prod offline + reportes + smoke `persist()` (1-2h)

**Objetivo:** Que los 3 commits pendientes de offline + reportes (mencionados en auditoría 22/04) lleguen a prod. Validar manualmente que `navigator.storage.persist()` se concede en Chrome real con PWA instalada.

**Pre-condición:** Beto presente para validar smoke en `https://espritos.app` con su laptop/celular.

**Pasos:**

```bash
# En ~/espritos/ (rama main):
git log origin/main..main --oneline           # confirmar 3 commits pendientes
git push origin main                           # dispara CI
# Esperar CI verde

# En el servidor 192.168.0.152 (vía SSH desde la PC de trabajo):
cd ~/espritos
git pull
docker compose -f docker-compose.prod.yml --env-file .env.production pull
docker compose -f docker-compose.prod.yml --env-file .env.production up -d --force-recreate web worker beat
docker compose -f docker-compose.prod.yml --env-file .env.production exec web python manage.py migrate --noinput
docker compose -f docker-compose.prod.yml --env-file .env.production exec web python manage.py collectstatic --noinput

# Verificar containers healthy:
docker compose -f docker-compose.prod.yml ps
```

**Smoke manual (Beto + builder en sesión, ~15 min):**
1. Abrir `https://espritos.app` en Chrome real (no headless)
2. Login → DevTools → Application → Manifest → instalar PWA
3. Una vez instalada: Application → Storage → confirmar "Persistent" = `true`
4. Toggle offline en DevTools → Network → Offline → navegar entre módulos → debe funcionar lectura de cache
5. Crear una venta dummy en POS offline → confirmar que se encola → reconectar → confirmar que se sincroniza
6. Revisar `/reports/` → ver lista de reportes recientes + abrir 1 PDF

**DoD del step:**
- [ ] `git log origin/main..main` vacío
- [ ] CI verde para los 3 commits
- [ ] `docker compose ps` muestra 6 containers healthy en prod
- [ ] PWA instalable + `persist()` = true en Chrome real
- [ ] Offline mode funcional (smoke pasado)
- [ ] Reportes accesibles
- [ ] Documentar resultado del smoke en `docs/piloto/findings/2026-04-23-prod-smoke.md`
- [ ] **Si `persist()` no se concede:** abrir issue, escalar — bloquea piloto

---

#### Step O1.3 — Compras templates: OC + recepciones + CxP (4-6h)

**Objetivo:** Habilitar el módulo Compras en sidebar (quitar el `disabled`). UI funcional para crear órdenes de compra, recibir mercancía, ver estado de CxP.

**Archivos a crear:**

```
apps/compras/
├── templates/compras/                       ← CARPETA NUEVA
│   ├── lista_ordenes.html                   ← NUEVO
│   ├── detalle_orden.html                   ← NUEVO
│   ├── _tabla_ordenes.html                  ← NUEVO (parcial HTMX)
│   ├── crear_orden.html                     ← NUEVO (form completo)
│   ├── recibir_mercancia.html               ← NUEVO (recepción contra OC)
│   ├── _line_item.html                      ← NUEVO (parcial: línea de OC editable)
│   └── cxp_lista.html                       ← NUEVO (cuentas por pagar)
└── tests/test_views_ui.py                   ← NUEVO (mínimo 8 tests)
```

**Patrón a seguir:**
- `lista_ordenes.html` → patrón de `templates/remisiones/lista.html`
- `detalle_orden.html` → patrón de `templates/remisiones/detalle.html`
- `crear_orden.html` → patrón de `templates/pos/crear_remision.html` (o equivalente que permita agregar líneas dinámicas con HTMX)
- `recibir_mercancia.html` → form que toma `OC.id`, lista líneas pendientes, cantidad recibida por línea, número de lote (si aplica), expira (si perecedero)

**Sidebar:**
- Editar `templates/_includes/sidebar.html` (o donde viva la nav)
- Quitar `disabled` y tooltip honesto en el item "Compras"
- Verificar que el ícono apunta a `{% url 'compras:lista_ordenes' %}`

**Tests mínimos:**
1. `test_lista_ordenes_renderiza` — admin ve todas las OC abiertas, cerradas y canceladas
2. `test_lista_filtra_por_proveedor`
3. `test_crear_orden_valida_proveedor_y_lineas` — form rechaza OC sin líneas o sin proveedor
4. `test_crear_orden_persiste_y_genera_folio` — crea OC con folio incremental único
5. `test_detalle_muestra_estado_recepcion` — calcula % recibido vs OC
6. `test_recibir_mercancia_actualiza_inventario` — ingresa lotes a `InventoryBin` correcto
7. `test_recibir_mercancia_genera_movimiento_kardex` — crea entrada `StockLedgerEntry`
8. `test_cxp_lista_calcula_aging` — agrupa CxP por días vencidos (0-30, 31-60, 61-90, 90+)

**DoD del step:**
- [ ] `pytest apps/compras/tests/test_views_ui.py` → 8 verde
- [ ] Sidebar muestra "Compras" sin disabled, click navega a `lista_ordenes.html`
- [ ] Crear OC nueva con 3 líneas → guarda con folio
- [ ] Recibir parcialmente → estado = "Parcialmente recibida", inventario actualizado
- [ ] Recibir completo → estado = "Cerrada", CxP creada
- [ ] Commit `[PILOTO][O1.3] Compras UI: OC + recepciones + CxP`

---

#### Step O1.4 — Setup de los 5 users del piloto (1h builder + 30min Beto)

**Objetivo:** Los 5 users (andrea, betty, carlos, maggi, jamie) tienen rol + sucursal + `puede_ver_ambas` correctos, password generada, y credenciales documentadas en lugar seguro.

**Beto define ANTES (input para el builder):**

| User | Rol (vendedor / cajero / supervisor / cobranza / almacenista / admin) | Sucursal (CREM / ABAR) | Puede ver ambas (sí/no) |
|---|---|---|---|
| andrea | ? | ? | ? |
| betty | ? | ? | ? |
| carlos | ? | ? | ? |
| maggi | ? | ? | ? |
| jamie | ? | ? | ? |

**Builder ejecuta:** crear management command `apps/core/management/commands/setup_pilot_users.py` parametrizado:

```python
python manage.py setup_pilot_users \
  --user andrea --rol cajero --sucursal CREM --puede-ver-ambas no \
  --user betty --rol vendedor --sucursal CREM --puede-ver-ambas no \
  ...
```

El comando:
1. Crea/actualiza el `User` (no recrea si existe — idempotente)
2. Asigna al grupo correcto (`cajeros`, `vendedores`, etc.)
3. Setea `Profile.sucursal_principal_id` y `Profile.puede_ver_ambas_sucursales`
4. Genera password segura (16 chars, alfanumérica + símbolos seguros) si no se pasa `--password`
5. Imprime tabla final con `username | rol | sucursal | password_temporal` para que Beto la copie a 1Password / KeePass

**Tests mínimos:**
1. `test_setup_pilot_users_es_idempotente` — correr 2 veces no duplica grupos ni sobrescribe password si ya existe (a menos que pases `--reset-password`)
2. `test_setup_pilot_users_asigna_grupo_correcto`
3. `test_setup_pilot_users_setea_sucursal_y_flag`

**Verificación post-ejecución (Beto en Django admin o script):**

```bash
python manage.py shell -c "
from django.contrib.auth import get_user_model
U = get_user_model()
for u in U.objects.filter(username__in=['andrea','betty','carlos','maggi','jamie']):
    print(u.username, list(u.groups.values_list('name', flat=True)),
          u.profile.sucursal_principal.codigo, u.profile.puede_ver_ambas_sucursales)
"
```

**DoD del step:**
- [ ] 5 users existen con rol+sucursal+flag confirmados
- [ ] Passwords documentadas en lugar seguro (responsabilidad de Beto)
- [ ] Tests verde
- [ ] Commit `[PILOTO][O1.4] Setup users piloto`

---

### 🎯 Hito de cierre Ola 1

Antes de mergear `pilot/ola-1-bloqueadores` → `main`:
- [ ] Los 4 steps con DoD ✅
- [ ] `pytest -q` total verde (sin regresiones a 2,227 baseline)
- [ ] Click en cualquier item del sidebar ya no tira `TemplateDoesNotExist`
- [ ] Prod tiene los 3 commits offline + reportes + `persist()` validado
- [ ] 5 users listos
- [ ] Smoke manual de los 4 steps documentado en `docs/piloto/findings/2026-04-24-ola-1-cierre.md`

---

### OLA 2 — Hardware POS + Data histórica (Sáb 25 + Dom 26, ~16h efectivas)

#### Step O2.1 — Hardware POS bridge: FastAPI local (1.5 días, ~12h)

**Objetivo:** Servicio local en cada PC de cobro que recibe comandos HTTP del POS (Django) y los traduce a operaciones reales de hardware (impresora ESC/POS, scanner USB HID, báscula serial, cajón vía pin RJ11 de la impresora).

**Arquitectura:**
- Servicio FastAPI corriendo en `localhost:9100` en cada PC de cobro (Windows)
- Empaquetado como `.exe` con PyInstaller para arranque automático en boot
- POS de Django hace `fetch('http://localhost:9100/print', ...)` desde el navegador (CORS habilitado solo para `https://espritos.app` y `http://localhost:8100`)
- Hardware esperado (confirmado disponible 22/04):
  - Impresora térmica ESC/POS 80mm (USB o ethernet)
  - Scanner código de barras USB HID (no requiere driver — funciona como teclado)
  - Báscula con interfaz serial RS-232 / USB-serial (Kretz NX o equivalente)
  - Cajón monedero conectado al pin RJ11 de la impresora (apertura por comando ESC/POS `0x1B 0x70`)

**Estructura del proyecto bridge:**

```
hardware-bridge/                              ← REPO SEPARADO o sub-folder en ~/espritos/extras/hardware-bridge/
├── pyproject.toml
├── README.md
├── bridge/
│   ├── __init__.py
│   ├── main.py                              ← FastAPI app
│   ├── printer.py                           ← python-escpos
│   ├── scanner.py                           ← logueo + endpoint /scanner/last
│   ├── scale.py                             ← pyserial — leer peso estable
│   ├── drawer.py                            ← apertura cajón
│   └── config.py                            ← lee config.toml local
├── config.example.toml
├── tests/
│   ├── test_printer_mock.py
│   ├── test_scale_mock.py
│   └── test_endpoints.py
└── build.bat                                ← PyInstaller spec
```

**Endpoints HTTP que expone:**

| Método | Path | Body | Respuesta | Para qué |
|---|---|---|---|---|
| GET | `/health` | — | `{ok: true, version, hardware: {...}}` | Healthcheck desde POS |
| POST | `/print/ticket` | `{ticket_html, copies}` | `{ok, ms}` | Imprimir ticket de venta |
| POST | `/print/raw` | `{escpos_b64}` | `{ok, ms}` | Comando ESC/POS arbitrario |
| POST | `/drawer/open` | — | `{ok, ms}` | Abrir cajón |
| GET | `/scale/weight` | — | `{ok, weight_kg, stable}` | Leer peso báscula |
| GET | `/scanner/listen` | — | SSE stream | Stream de scans (futuro) |

**Config TOML por PC de cobro:**

```toml
[printer]
type = "usb"             # "usb" | "ethernet"
vendor_id = 0x04b8
product_id = 0x0202
# o si ethernet: host = "192.168.0.50", port = 9100

[scale]
port = "COM3"
baudrate = 9600
protocol = "kretz_nx"    # "kretz_nx" | "ohaus" | "generic_iso"

[drawer]
mode = "via_printer"     # apertura por pin RJ11 de la impresora

[allowed_origins]
origins = ["https://espritos.app", "http://localhost:8100"]
```

**Cambios en EspritOS** (apps.pos):
- `templates/pos/cobrar.html` — botón "Imprimir ticket" hace `fetch('http://localhost:9100/print/ticket', ...)` con timeout 5s, fallback a `window.print()` si no responde
- `templates/pos/cobrar.html` — botón "Abrir cajón" tras cobro en efectivo
- `templates/pos/agregar_producto.html` — campo de peso con `hx-get` periódico a `/api/pos/peso/` que internamente llama al bridge
- Nueva view `apps/pos/views.py::leer_peso_bascula` que envuelve la llamada al bridge (timeout, error handling, log)
- Setting `HARDWARE_BRIDGE_URL = "http://localhost:9100"` en `config/settings/base.py` (override por env var)

**Tests del bridge (mínimo 8):**
1. `test_print_ticket_con_impresora_mock`
2. `test_print_raw_envia_bytes_correctos`
3. `test_drawer_open_envia_secuencia_escpos`
4. `test_scale_lectura_estable_retorna_peso`
5. `test_scale_lectura_inestable_reintenta_3_veces`
6. `test_health_devuelve_estado_hardware_detectado`
7. `test_endpoints_rechaza_origen_no_permitido` (CORS)
8. `test_config_loader_valida_archivo_toml`

**Tests EspritOS (en apps.pos, mínimo 4):**
1. `test_pos_imprimir_ticket_llama_a_bridge_url`
2. `test_pos_imprimir_ticket_fallback_a_window_print_si_bridge_no_responde`
3. `test_pos_leer_peso_devuelve_kg_estable`
4. `test_pos_abrir_cajon_solo_tras_cobro_en_efectivo`

**Despliegue (Beto presente):**
1. Builder genera `.exe` con `pyinstaller --onefile bridge/main.py`
2. Beto + builder llevan `.exe` + `config.toml` a las PCs de cobro de la tienda
3. Configurar arranque automático (Task Scheduler de Windows: trigger "At log on" → Action: ejecutar el `.exe`)
4. Probar cada hardware con 1 venta real (sin cobrar): escaneo + peso + impresión + cajón
5. Documentar IDs USB / puertos COM en `docs/piloto/hardware-tienda.md`

**DoD del step:**
- [ ] Tests del bridge verde
- [ ] Tests EspritOS verde
- [ ] `.exe` corriendo en al menos 1 PC de cobro de la tienda
- [ ] Test E2E manual: escanear → agregar producto → leer peso → cobrar → imprimir → abrir cajón → todo en < 15 segundos
- [ ] Documentado en `docs/piloto/hardware-tienda.md` (IDs USB, puertos COM, marcas/modelos exactos)
- [ ] Commit `[PILOTO][O2.1] Hardware POS bridge`

---

#### Step O2.2 — Carga histórica StockLedgerEntry 5.1M (1h ejecución + verificación, ventana nocturna)

**Objetivo:** Cargar las 5.1M entradas históricas de movimientos de inventario desde Punto Zero. El POS ya funciona con el snapshot — esto es para que reportes históricos y kardex retroactivo funcionen correctamente.

**Pre-condiciones:**
- ✅ Backup de prod tomado antes de la ventana (`pg_dump` completo, validable con restore)
- ✅ Ventana nocturna confirmada (sáb 25 → dom 26, 22:00-06:00, sistema no operativo)
- ✅ Espacio en disco suficiente: 5.1M filas × ~200 bytes ≈ 1 GB + índices = reservar 3 GB libres en `/var/lib/postgresql`
- ✅ `etl_load_stock_ledger_historical` ya existe y se probó en dev con 100 filas

**Comando:**

```bash
# En el servidor 192.168.0.152, dentro del container web:
docker compose -f docker-compose.prod.yml --env-file .env.production exec web bash

# Backup pre-load (obligatorio):
python manage.py dbbackup --output-filename=pre_stockledger_load_$(date +%Y%m%d_%H%M).psql.gz

# Pre-flight:
python manage.py etl_preflight --module stock_ledger_historical
# Espera: ✅ READY, total filas en origen: 5,XXX,XXX, tiempo estimado: ~45 min

# Ejecutar (con --confirm requerido):
python manage.py etl_load_stock_ledger_historical --confirm --batch-size=10000 \
  | tee logs/etl_stockledger_$(date +%Y%m%d_%H%M).log

# Verificar:
python manage.py etl_status --module stock_ledger_historical
python manage.py etl_verify --module stock_ledger_historical --tolerance=0.001
# Diferencia < 0.1% por sucursal y por mes = OK
```

**Verificación de integridad post-load:**
1. `SELECT count(*) FROM apps_inventario_stockledgerentry;` → debe ser ≈ 5.1M
2. `SELECT sucursal_id, count(*), min(fecha), max(fecha) FROM ... GROUP BY 1;` → ambas sucursales presentes, rango de fechas coherente
3. Spot-check: 10 productos random — comparar saldo histórico contra Punto Zero

**Plan de rollback** (si verify falla):

```bash
# 1. Vaciar la tabla:
python manage.py shell -c "from apps.inventario.models import StockLedgerEntry; StockLedgerEntry.objects.filter(origen='etl_historico').delete()"

# 2. O si la corrupción es mayor, restore del backup:
python manage.py dbrestore --input-filename=pre_stockledger_load_<TIMESTAMP>.psql.gz
```

**DoD del step:**
- [ ] Backup pre-load existe y es restaurable
- [ ] Comando completó sin errores en log
- [ ] `etl_verify` < 0.1% diferencia
- [ ] Spot-check 10 productos consistente
- [ ] Documentar resultado en `docs/piloto/findings/2026-04-26-stockledger-load.md`
- [ ] Commit (solo doc) `[PILOTO][O2.2] Carga histórica StockLedger ejecutada`

---

#### Step O2.3 — Fix Facturama 401 (1-2h)

**Objetivo:** Resolver el error 401 que actualmente bloquea las pruebas de timbrado CFDI sandbox.

**Decisión de Beto previa al step:**

| Opción | Esfuerzo | Cuándo elegirla |
|---|---|---|
| A. Renovar credenciales sandbox actuales | 30 min | Si la cuenta sigue activa pero token expiró |
| B. Crear cuenta nueva en Facturama sandbox | 1h | Si la cuenta original se perdió |
| C. Subir directo a Facturama producción con CSD real | 2h | Si hay urgencia y CSD `EKU9003173C9` está listo |

**Mi voto:** **A**. Renovar primero. Si la cuenta murió → B. Producción la dejas para post-piloto (a menos que el piloto SÍ va a timbrar facturas reales — confirma con Beto).

**Pasos comunes a A/B:**
1. Loguearse en `https://api.facturama.mx` (Beto)
2. Generar nuevo token de API en sandbox
3. Actualizar `.env.production` en el servidor:
   ```
   FACTURAMA_USERNAME=<nuevo>
   FACTURAMA_PASSWORD=<nuevo>
   FACTURAMA_ENV=sandbox
   ```
4. Restart container web: `docker compose ... restart web`
5. Probar timbrado: en `lista.html` (recién creado en O1.1) → emitir factura test contra cliente Cremería HM con datos sandbox conocidos
6. Verificar que XML se genera, UUID se devuelve, PDF se descarga

**Tests:**
- Smoke manual cuenta como tests aquí. Si quieres test automatizado: `tests/test_facturama_integration.py::test_timbrar_factura_sandbox_devuelve_uuid` con `@pytest.mark.integration` (skippable en CI).

**DoD del step:**
- [ ] Timbrado test exitoso (UUID válido recibido)
- [ ] Nuevas credenciales documentadas en `.env.production` y backup secreto de Beto
- [ ] Commit (solo doc/env): `[PILOTO][O2.3] Facturama 401 resuelto`

---

### 🎯 Hito de cierre Ola 2

- [ ] Bridge `.exe` corriendo en al menos una PC de tienda con hardware funcional
- [ ] StockLedger 5.1M cargado y verificado (< 0.1% diferencia)
- [ ] Facturama timbrando sin 401
- [ ] Tests totales verde, sin regresiones
- [ ] Mergear `pilot/ola-2-hardware-data` → `main`

---

### OLA 3 — Completar módulos esqueleto + Data quality (Lun 27 + Mar 28 + Mié 29, ~24h efectivas)

#### Step O3.1 — Lealtad: UI completa con monedero activo (1 día, ~8h)

**Objetivo:** Cliente con tarjeta de lealtad acumula saldo en monedero por compra, ve su saldo, y puede canjearlo en POS.

**Backend ya existe:** 3 modelos (`TarjetaLealtad`, `MovimientoMonedero`, probablemente `ConfiguracionLealtad`). Confirmar leyendo `apps/lealtad/models.py`.

**Cambios:**

```
apps/lealtad/
├── templates/lealtad/
│   ├── lista.html                    ← YA EXISTE básica, expandir
│   ├── detalle_tarjeta.html          ← NUEVO
│   ├── _movimientos.html             ← NUEVO (parcial historial)
│   └── canje_modal.html              ← NUEVO (usado desde POS)
├── views.py                          ← agregar: detalle_tarjeta, canjear_monedero
├── services/
│   └── monedero.py                   ← NUEVO: acumular(), canjear(), saldo_actual()
└── tests/test_monedero_service.py    ← NUEVO (mínimo 6 tests)

apps/pos/
└── templates/pos/cobrar.html         ← agregar botón "Aplicar monedero" si cliente tiene tarjeta
```

**Reglas de negocio (validar con Beto si dudas):**
- % de acumulación configurable global (ej: 1% del subtotal post-IVA)
- Saldo se acumula al momento del cobro exitoso (no en remisión a crédito)
- Saldo se puede canjear desde 1 peso, máximo 100% del subtotal
- Saldo no caduca (a menos que `ConfiguracionLealtad.dias_caducidad > 0`)
- Movimientos auditados (acumular, canjear, ajuste manual admin)

**Tests mínimos del service:**
1. `test_acumular_suma_porcentaje_correcto`
2. `test_acumular_solo_en_cobro_efectivo_o_tarjeta`
3. `test_canjear_descuenta_del_saldo`
4. `test_canjear_falla_si_excede_saldo`
5. `test_canjear_falla_si_excede_subtotal`
6. `test_movimientos_son_auditados_con_user_y_fecha`

**Tests UI mínimos (2):**
1. `test_detalle_tarjeta_muestra_saldo_e_historial`
2. `test_pos_aplica_monedero_y_descuenta_subtotal`

**DoD:**
- [ ] Tests verde
- [ ] Cliente puede ver saldo en `/lealtad/<tarjeta>/`
- [ ] POS muestra botón "Aplicar monedero" si cliente tiene tarjeta
- [ ] Cobro exitoso acumula saldo automáticamente
- [ ] Commit `[PILOTO][O3.1] Lealtad UI + monedero activo`

---

#### Step O3.2 — Promociones: CRUD UI mínimo (1 día, ~6h)

**Objetivo:** Admin/supervisor puede crear, editar, activar/desactivar promociones desde la UI sin tocar Django admin.

**Backend ya existe:** 3 modelos (`Promocion`, `RegistroAplicacion`, posiblemente `ProductoEnPromocion`). Confirmar.

**Cambios:**

```
apps/promociones/
├── templates/promociones/
│   ├── lista.html                    ← YA EXISTE mínimo, expandir
│   ├── form_crear_editar.html        ← NUEVO
│   ├── detalle.html                  ← NUEVO
│   └── _toggle_activa.html           ← NUEVO (parcial HTMX para activar/desactivar)
├── views.py                          ← agregar: crear, editar, toggle, detalle
├── forms.py                          ← NUEVO si no existe
└── tests/test_views_ui.py            ← NUEVO (mínimo 6 tests)
```

**Tipos de promoción a soportar (mínimo viable para piloto):**
- **% descuento** sobre productos seleccionados
- **NxM** (lleva N, paga M)
- **Precio fijo** para combo de productos
- (Futuro post-piloto: 2x1, regalo con compra, escalonadas)

**Form de crear/editar:**
- Nombre, descripción, tipo (dropdown), parámetros del tipo (% / N / M / precio fijo), productos aplicables (multi-select con búsqueda), sucursales aplicables (multi), fecha inicio/fin, activa (switch), prioridad (entero, para resolución de conflictos)

**Tests mínimos:**
1. `test_lista_renderiza_con_filtros_activa_inactiva_vencida`
2. `test_crear_promocion_porcentaje_persiste`
3. `test_crear_promocion_nxm_valida_n_mayor_que_m`
4. `test_editar_promocion_no_pierde_aplicaciones_pasadas`
5. `test_toggle_activa_cambia_estado_y_audita`
6. `test_detalle_muestra_aplicaciones_recientes_y_descuento_total`

**DoD:**
- [ ] Tests verde
- [ ] Crear promoción "20% galletas Marinela del 28/04 al 5/05" desde UI funciona end-to-end
- [ ] Aplicar en POS al escanear producto en promo aplica el descuento
- [ ] Commit `[PILOTO][O3.2] Promociones UI`

---

#### Step O3.3 — Kanban CRM con drag-drop Sortable.js (2-3h)

**Objetivo:** Mover tarjetas de oportunidad entre etapas del pipeline arrastrando en lugar de usar el dropdown actual.

**Cambios:**
- Agregar Sortable.js (CDN o npm si ya hay build): `https://cdn.jsdelivr.net/npm/sortablejs@1.15.2/Sortable.min.js`
- En `templates/crm/pipeline_kanban.html`:
  - Inicializar Sortable en cada columna con `Alpine.js x-init`
  - Al `onEnd`, `fetch POST` a `/crm/oportunidad/<id>/mover-etapa/` con `{ etapa_destino_id, posicion }`
  - View existente `mover_etapa` (o crearla) actualiza modelo + devuelve `_card.html` parcial
- HTMX swap: el card se reemplaza in-place con el HTML actualizado
- Loading state (opacity reducido mientras se persiste)
- Toast de error si falla (con rollback visual)

**Tests:**
1. `test_mover_etapa_actualiza_modelo` (existente o nuevo)
2. `test_mover_etapa_audita_cambio_de_etapa`
3. `test_mover_etapa_falla_si_no_hay_permiso_en_oportunidad`

**DoD:**
- [ ] Drag de tarjeta entre 2 columnas persiste (verifica con `Oportunidad.objects.get(id=X).etapa`)
- [ ] Tests verde
- [ ] Commit `[PILOTO][O3.3] Kanban drag-drop`

---

#### Step O3.4 — Vistas CRM V5: Leads + Conversion + Forecast (1-2 días, ~10h)

**Objetivo:** Habilitar las vistas CRM V5 que ya tienen modelos pero no están linkeadas al sidebar.

**Cambios:**

```
apps/crm/
├── templates/crm/
│   ├── leads_lista.html              ← NUEVO (lista de Leads no convertidos)
│   ├── lead_detalle.html             ← NUEVO
│   ├── lead_convertir.html           ← NUEVO (Lead → Oportunidad o Cliente)
│   ├── conversion_funnel.html        ← NUEVO (visual del embudo)
│   └── forecast_dashboard.html       ← NUEVO (pipeline ponderado, esperado vs real)
├── views.py                          ← agregar: leads_lista, lead_detalle, convertir, funnel, forecast
└── tests/test_v5_views.py            ← NUEVO (mínimo 8 tests)

templates/_includes/sidebar.html      ← agregar items "Leads", "Embudo", "Forecast" bajo CRM
```

**Funcionalidad mínima:**

- **Leads lista:** tabla de Leads (nombre, fuente, status, score, owner, última actividad, acciones convertir/descartar)
- **Lead detalle:** info contacto, notas, actividades, score breakdown, botón "Convertir a Oportunidad" / "Convertir a Cliente"
- **Conversion funnel:** chart Chart.js (ya en stack si no, agregar) — Leads → Calificados → Oportunidades → Ganadas, con tasas %
- **Forecast dashboard:** suma del valor esperado del pipeline ponderado por probabilidad de cada etapa, agrupado por mes, comparado con cuota

**Tests mínimos:**
1. `test_leads_lista_filtra_por_owner_para_vendedor`
2. `test_leads_lista_admin_ve_todos`
3. `test_convertir_lead_a_oportunidad_crea_oportunidad_y_marca_lead_convertido`
4. `test_convertir_lead_a_cliente_crea_cliente_y_marca_lead_convertido`
5. `test_funnel_calcula_tasas_correctamente`
6. `test_forecast_pondera_por_probabilidad_de_etapa`
7. `test_forecast_filtra_por_owner_segun_rol`
8. `test_sidebar_muestra_links_v5_solo_si_user_tiene_permiso_crm`

**DoD:**
- [ ] Tests verde
- [ ] Sidebar muestra Leads / Embudo / Forecast
- [ ] Convertir un Lead end-to-end funciona
- [ ] Funnel y forecast renderizan con datos seed reales
- [ ] Commit `[PILOTO][O3.4] CRM V5 vistas`

---

#### Step O3.5 — Ocultar Rutero del sidebar (15 min)

**Objetivo:** Que el equipo del piloto no vea Rutero (esqueleto, no para uso real).

**Cambios:**
- En `templates/_includes/sidebar.html`, envolver el item Rutero en:
  ```django
  {% if request.user.is_superuser or 'rutero' in request.user.groups.values_list('name', flat=True) %}
  ```
- O más simple: agregar setting `FEATURES = {"rutero": False}` y `{% if FEATURES.rutero %}`

**Tests:** 1 — `test_sidebar_no_muestra_rutero_para_user_normal`

**DoD:**
- [ ] Login como cajero/vendedor → no ve Rutero
- [ ] Login como admin → sigue viéndolo (para futuro)
- [ ] Commit `[PILOTO][O3.5] Ocultar Rutero del sidebar piloto`

---

#### Step O3.6 — Data quality: 142F + huérfanos (2-4h)

**Objetivo:** Decidir qué hacer con 2 anomalías de data legacy que distorsionan reportes.

**(a) Código 142F "CARNES FRIAS A GRANEL"** — Beto consulta con Humberto:
- ¿Es producto real (granel sin SKU, distorsiona top productos pero es legítimo) o comodín de captura (debe excluirse de reportes)?

**Acción según respuesta:**
- **Real:** crear excepción documentada, dejarlo, pero etiquetarlo como `categoria_reporte='granel'` para excluir de "top SKUs"
- **Comodín:** marcar `producto.activo=False` y excluir de reportes históricos vía flag

**(b) 1.94M detalles huérfanos de `ticketsdetalle`** — purga:

```bash
# En el servidor (con backup ya tomado en O2.2):
python manage.py shell -c "
from apps.etl.models import LegacyTicketDetalle  # ajustar al nombre real
huerfanos = LegacyTicketDetalle.objects.filter(ticket__isnull=True)
print(f'A purgar: {huerfanos.count():,}')
"

# Si el conteo coincide con 1.94M ± 5%, ejecutar purga en batches:
python manage.py purge_huerfanos_ticketsdetalle --confirm --batch=50000
```

**Test:** `test_purge_huerfanos_no_borra_ticketsdetalle_con_ticket_valido`

**DoD:**
- [ ] Decisión 142F documentada en `docs/piloto/findings/2026-04-29-data-quality.md`
- [ ] Acción correspondiente ejecutada
- [ ] Huérfanos purgados o documentado por qué se posponen
- [ ] Commit `[PILOTO][O3.6] Data quality: 142F + huérfanos`

---

### 🎯 Hito de cierre Ola 3

- [ ] Lealtad operativa con monedero
- [ ] Promociones CRUD funcional
- [ ] Kanban drag-drop
- [ ] Vistas CRM V5 linkeadas
- [ ] Rutero oculto
- [ ] Data quality limpio
- [ ] Tests totales verde
- [ ] Mergear `pilot/ola-3-modulos` → `main`

---

### OLA 4 — Validación pre-piloto (Jue 30, ~8h)

#### Step O4.1 — UAT con Beto: 6 flujos clave (3h)

**Objetivo:** Beto + builder recorren 6 flujos completos en `https://espritos.app` con cuenta admin. Documentar cada fricción, bug o ambigüedad.

**Flujos a recorrer (en este orden):**

1. **Login + dashboard** → cada rol ve lo correcto (probar admin, cajero CREM, vendedor)
2. **POS venta completa** → buscar producto → escanear (con scanner real) → leer peso (báscula real) → cobrar mixto efectivo+tarjeta → imprimir ticket (impresora real) → abrir cajón → aplicar monedero si cliente tiene tarjeta
3. **Remisión a crédito** → crear remisión → enviar → recibir pago parcial → ver aging actualizado en cobranza
4. **Compras** → crear OC → recibir parcialmente → recibir resto → CxP creada → marcar pagada
5. **CFDI** → emitir factura desde remisión → descargar XML + PDF → cancelar con motivo 02 → confirmar status cancelado
6. **Reportes + Chat IA** → generar reporte de ventas semanales → preguntar al chat IA "cuánto le vendí a FDH la última semana"

**Por cada flujo, capturar:**
- ✅ / ⚠️ / ❌
- Tiempo de ejecución (debe ser < 60s para venta, < 30s para reportes)
- Fricciones encontradas
- Bugs (issue al instante con prioridad B/C/D)

**Output:** `docs/piloto/uat-2026-04-30-beto.md` con tabla de resultado.

**DoD:**
- [ ] 6 flujos recorridos
- [ ] Documento UAT creado
- [ ] Bugs B (bloqueador) → fix inmediato en O4.3
- [ ] Bugs C/D → backlog post-piloto

---

#### Step O4.2 — Ensayo Bernardo en prod (2h)

**Objetivo:** Bernardo (admin onboardeado) hace ejercicio realista solo (Beto observa pero no interviene).

**Script para Bernardo:**
1. Login con su usuario
2. Hacer 5 ventas POS de productos distintos (algunas en efectivo, algunas tarjeta, una mixta)
3. Emitir CFDI para el cliente "TEST MOSTRADOR" en una de esas 5 ventas
4. Hacer 1 corte de caja al final
5. Revisar CxC del día en cobranza
6. Generar 1 reporte de ventas del día

**Métricas a capturar:**
- Tiempo total del ejercicio
- ¿Hubo que llamar a Beto para algo? Si sí, qué
- Bernardo's qualitative feedback ("¿esto es más fácil o más difícil que PuntoZero?")

**Output:** `docs/piloto/ensayo-bernardo-2026-04-30.md`

**DoD:**
- [ ] Ejercicio completado
- [ ] Documento generado
- [ ] Issues abiertas para fricciones identificadas

---

#### Step O4.3 — Fixes + manuales actualizados (variable, ~3h buffer)

**Objetivo:** Resolver bugs B (bloqueadores) que salieron en UAT/ensayo. Actualizar los manuales `.docx` con capturas de pantalla reales.

**Manuales a actualizar:**
1. **Manual Cajero** — flujos POS + cobro + cierre
2. **Manual Vendedor** — remisiones + CRM + cotizaciones
3. **Manual Admin** — todo + reportes + configuración

**Capturas:** sacar screenshots reales de la prod actualizada (no de mockups).

**Imprimir:** 1 copia por user del piloto, encuadernada (Beto se encarga del print físico).

**DoD:**
- [ ] Bugs B resueltos y verificados con re-test
- [ ] Manuales `.docx` actualizados con capturas reales
- [ ] PDFs generados desde los `.docx`
- [ ] Copias físicas listas para entregar el viernes 1/05 a primera hora

---

### 🎯 Hito de cierre Ola 4

- [ ] Bugs B = 0
- [ ] Manuales actualizados y impresos
- [ ] UAT + ensayo documentados

---

## 5. Patrones técnicos clave

### 5.1 Templates HTMX (CFDI, Compras, Lealtad, Promociones)

Todo template `lista.html` sigue el mismo patrón:

```django
{% extends "base.html" %}
{% load core_tags %}

{% block content %}
<div class="container mx-auto p-4">
  <div class="flex justify-between items-center mb-4">
    <h1 class="text-2xl font-bold">{{ titulo }}</h1>
    <div class="flex gap-2">
      <a href="{% url 'modulo:crear' %}" class="btn btn-primary">+ Nuevo</a>
    </div>
  </div>

  <form hx-get="{% url 'modulo:tabla' %}"
        hx-target="#tabla-container"
        hx-trigger="submit, keyup changed delay:300ms from:input"
        class="bg-base-200 p-4 rounded mb-4 flex gap-2">
    {# filtros aquí #}
  </form>

  <div id="tabla-container">
    {% include "modulo/_tabla.html" %}
  </div>
</div>
{% endblock %}
```

El parcial `_tabla.html` debe ser autocontenido (renderizable por sí mismo) para que HTMX lo intercambie sin romper estilos.

### 5.2 Permisos en views (ya establecido en EspritOS)

```python
from apps.core.mixins import RoleFilteredQuerysetMixin

class CFDIListView(LoginRequiredMixin, RoleFilteredQuerysetMixin, ListView):
    model = CFDI
    template_name = "cfdi/lista.html"
    paginate_by = 50
    role_filter_field = "sucursal"  # filtra automáticamente si user.profile.puede_ver_ambas == False
```

### 5.3 Tests UI (patrón establecido)

```python
import pytest
from django.urls import reverse
from apps.core.tests.fixtures import user_admin, user_cajero_crem  # ya en conftest.py

@pytest.mark.django_db
def test_cfdi_lista_renderiza_para_admin(client, user_admin):
    client.force_login(user_admin)
    resp = client.get(reverse("cfdi:lista"))
    assert resp.status_code == 200
    assert "lista de facturas" in resp.content.decode().lower()
```

---

## 6. Setup operativo del piloto

### 6.1 Credenciales y secretos

| Item | Dónde vive | Quién | Cuándo |
|---|---|---|---|
| Passwords 5 users | 1Password / KeePass de Beto | Beto | O1.4 |
| Token Facturama renovado | `.env.production` + backup secreto Beto | Beto | O2.3 |
| CSD `EKU9003173C9` (si va a prod) | Carpeta cifrada | Beto | Post-piloto |
| Config TOML hardware bridge | Cada PC de cobro local + repo backup | Builder + Beto | O2.1 |

### 6.2 Hardware en tienda

Confirmado disponible 22/04: PCs de cobro, scanner, impresoras tickets, cajones automáticos. Documentar marcas/modelos exactos en `docs/piloto/hardware-tienda.md` durante O2.1.

### 6.3 Plan de comunicación al equipo

**Lunes 27/04** (3 días antes del piloto):
- Reunión 30 min con los 5 users + Bernardo
- Mostrar EspritOS en pantalla, recorrer flujos básicos
- Entregar manual impreso
- Resolver dudas

**Jueves 30/04** (víspera):
- WhatsApp grupal: "Mañana arrancamos. Login en `https://espritos.app`. Beto y Bernardo en guardia primeras 4h. Si algo se rompe: reportar en este grupo, NO improvisar."

**Viernes 01/05** (día 1):
- 7am: Beto + Bernardo presentes en tienda
- 7:30am: Login con cada user, validar acceso
- 8am: Operación normal arranca con EspritOS

---

## 7. Go / No-Go gates

### Gate 1 — Cierre Ola 2 (Domingo 26/04 noche)

| Criterio | OK / Bloqueado |
|---|---|
| CFDI UI funcional, sin TemplateDoesNotExist | |
| Compras UI funcional | |
| 5 users configurados | |
| Bridge hardware corriendo en al menos 1 PC tienda | |
| StockLedger 5.1M cargado y verificado | |
| Facturama timbra sin 401 | |

**Si 2+ son ❌:** considerar diferir piloto a martes 5/05 (extender 4 días).

### Gate 2 — Cierre Ola 4 (Jueves 30/04 noche)

| Criterio | OK / Bloqueado |
|---|---|
| UAT con Beto = 6 flujos sin bugs B abiertos | |
| Ensayo Bernardo OK + feedback positivo o neutro | |
| Manuales impresos | |
| Lealtad + Promociones + Kanban + CRM V5 funcionando | |
| `pytest -q` total verde | |

**Si Gate 2 OK:** confirmar piloto 1/05 7am.
**Si Gate 2 con dudas:** llamada Beto + builder a las 8pm del jueves para decidir.

### Gate 3 — Pre-flight día del piloto (Viernes 01/05 7am)

```bash
# Smoke automatizado en prod:
docker compose -f docker-compose.prod.yml ps                # 6 healthy
curl -s https://espritos.app/health/ | jq                   # ok: true
docker compose ... exec web python manage.py check --deploy # 0 issues
docker compose ... exec web python manage.py showmigrations | grep -E "\[ \]"  # vacío
```

Y manual:
- [ ] Cada uno de los 5 users hace login exitoso
- [ ] Bridge hardware responde en cada PC: `curl localhost:9100/health`
- [ ] Impresora imprime ticket de prueba
- [ ] Cajón abre

**Si todo OK:** 🟢 piloto arranca.
**Si algo falla:** 🟡 30 min de fix → reintento. Si después de 1 hora no se resuelve, se evalúa diferir hasta el lunes 4/05.

---

## 8. Plan de contingencia (durante el piloto)

| Severidad | Definición | Acción |
|---|---|---|
| **A — Sistema caído** | EspritOS no responde, no se puede vender | Bernardo activa "modo PuntoZero" en cajas (llave física + caja vieja como respaldo). Beto avisa al equipo. Builder en sesión inmediata para diagnosticar. |
| **B — Módulo crítico roto** | POS funciona pero CFDI/Cobranza/Compras roto | Mitigación temporal: usar Django admin (`/admin/`) para esa función. Builder fix en < 4h. |
| **C — Bug menor** | UI fea, dato mal calculado, performance lenta | Issue en `docs/piloto/findings/`, fix en próxima sesión. No interrumpe operación. |
| **D — Fricción UX** | "no sé dónde está X" | Beto/Bernardo coachean en vivo. Anotar para mejora post-piloto. |

**Modo PuntoZero como red de seguridad:**
- PuntoZero NO se desconecta durante la semana del piloto
- Cajeros tienen instrucciones explícitas: "si EspritOS se cae, abren caja vieja en PuntoZero, anotan ventas en hoja física, al final del día Beto las recaptura en EspritOS"
- Doble captura solo si severity A. Severity B con mitigación admin no requiere doble captura.

**Métricas a capturar durante la semana del piloto** (instrumentar antes del 1/05):
- # ventas registradas en EspritOS por día / por user
- # incidentes severity A / B / C / D
- Tiempo promedio de venta POS (logueado en `apps.pos.metrics`)
- # CFDI emitidos
- # llamadas de soporte a Beto / Bernardo
- Encuesta cualitativa al cierre del día 7 (jueves 7/05): "¿seguimos con EspritOS o regresamos a PuntoZero?" (escala 1-5 + comentario)

---

## 9. Kickoff prompt para el builder

Pegar este bloque en una sesión nueva de Claude Code abierta en `~/espritos/`:

```
Estás trabajando en el sprint pre-piloto de EspritOS (~9 días) que cierra todos los gaps detectados en la auditoría del 2026-04-22 para arrancar el piloto el viernes 2026-05-01 con 5 usuarios reales + Bernardo.

CONTEXTO OBLIGATORIO A LEER ANTES DE ESCRIBIR UNA SOLA LÍNEA DE CÓDIGO:

1. `E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/PILOTO-CONSTRUCCION-BLUEPRINT.md` — el blueprint maestro de este sprint. Lee TODO. Es la fuente de verdad.

2. `~/espritos/CLAUDE.md` — instrucciones del proyecto vigentes.

3. `git log --oneline -30` — últimos commits para entender desde dónde retomas.

4. Para cada step antes de empezarlo:
   - Lee el patrón a seguir mencionado (ej: `templates/remisiones/lista.html`)
   - Lee los modelos involucrados (`apps/<modulo>/models.py`)
   - Lee los tests existentes (`apps/<modulo>/tests/`) para entender convenciones

[SCOPE DE ESTA SESIÓN]
- Ola: O<n>
- Steps a ejecutar: O<n>.<a> hasta O<n>.<b>
- Tiempo estimado: <X>h (suma de estimaciones)
- Branch git: pilot/ola-<n>-<nombre> (créalo si no existe desde main)
- Pre-flight verificado: <lista de DoD del step anterior>

REGLAS ESTRICTAS:

A. UN STEP A LA VEZ, EN ORDEN. Cada step termina con commit + tests verde + smoke manual documentado en commit body.

B. CERO ALCANCE FUERA DEL SCOPE. Si descubres bug ajeno: documenta en `docs/piloto/findings/` y sigue. Excepción: bug bloqueador del step actual → escala a Beto.

C. FORMATO DE COMMIT: `[PILOTO][O<n>.<n>] descripción`. Body incluye tests añadidos, total tests pasando, verificación manual.

D. TESTS OBLIGATORIOS MÍNIMOS POR STEP. Definidos en el blueprint. Cumple el mínimo. Calidad > cantidad.

E. VERIFICA TESTS PREVIOS VERDE. Antes de cada commit: `pytest --tb=short -q`. Sin regresiones contra baseline 2,227.

F. SI EL BLUEPRINT ES AMBIGUO, PREGUNTA A BETO. No improvises.

G. NO DEPLOYS PRODUCTIVOS NI CARGAS HISTÓRICAS SIN BETO PRESENTE.

H. ESPAÑOL EN COMUNICACIÓN, INGLÉS EN CÓDIGO, COMENTARIOS ESPAÑOL CUANDO NO SEA OBVIO.

I. STACK NO SE NEGOCIA. Django 5.1 + Postgres 16 + HTMX 2 + Alpine 3 + Tailwind v4 + DaisyUI + Celery + Redis + Claude API.

J. AL FINAL DE LA SESIÓN, REPORTA:
   - Steps completados
   - Tests añadidos / total tests
   - Próximo step
   - Bloqueos / dudas para Beto

K. SI TE QUEDAS SIN CONTEXTO: para, reporta dónde quedaste con suficiente detalle para retomar.

EMPIEZA AHORA. Primer paso: leer el blueprint completo + git log -30. Después, ejecutar el primer step del scope.
```

---

## 10. Apéndices

### A. Mapa de archivos nuevos esperados

```
~/espritos/
├── apps/cfdi/templates/cfdi/{lista,detalle,_tabla_cfdi,_form_emitir}.html  [O1.1]
├── apps/cfdi/tests/test_views_ui.py                                         [O1.1]
├── apps/compras/templates/compras/{lista_ordenes,detalle_orden,_tabla_ordenes,crear_orden,recibir_mercancia,_line_item,cxp_lista}.html  [O1.3]
├── apps/compras/tests/test_views_ui.py                                      [O1.3]
├── apps/core/management/commands/setup_pilot_users.py                       [O1.4]
├── apps/lealtad/templates/lealtad/{detalle_tarjeta,_movimientos,canje_modal}.html  [O3.1]
├── apps/lealtad/services/monedero.py                                        [O3.1]
├── apps/lealtad/tests/test_monedero_service.py                              [O3.1]
├── apps/promociones/templates/promociones/{form_crear_editar,detalle,_toggle_activa}.html  [O3.2]
├── apps/promociones/tests/test_views_ui.py                                  [O3.2]
├── apps/crm/templates/crm/{leads_lista,lead_detalle,lead_convertir,conversion_funnel,forecast_dashboard}.html  [O3.4]
├── apps/crm/tests/test_v5_views.py                                          [O3.4]
├── docs/piloto/
│   ├── hardware-tienda.md                                                   [O2.1]
│   ├── findings/2026-04-23-prod-smoke.md                                    [O1.2]
│   ├── findings/2026-04-24-ola-1-cierre.md                                  [Hito O1]
│   ├── findings/2026-04-26-stockledger-load.md                              [O2.2]
│   ├── findings/2026-04-29-data-quality.md                                  [O3.6]
│   ├── uat-2026-04-30-beto.md                                               [O4.1]
│   └── ensayo-bernardo-2026-04-30.md                                        [O4.2]
└── extras/hardware-bridge/                                                   [O2.1]
    └── (estructura completa en sección 4 / Step O2.1)
```

### B. Comandos de verificación rápida

```bash
# Tests totales:
pytest -q

# Solo tests del piloto:
pytest apps/cfdi/tests/test_views_ui.py apps/compras/tests/test_views_ui.py \
       apps/lealtad/tests/test_monedero_service.py apps/promociones/tests/test_views_ui.py \
       apps/crm/tests/test_v5_views.py

# Health prod:
curl -sS https://espritos.app/health/ | jq

# Containers prod:
docker compose -f docker-compose.prod.yml ps

# Migraciones pendientes:
docker compose ... exec web python manage.py showmigrations | grep "\[ \]"

# Smoke bridge hardware (en cada PC de cobro):
curl http://localhost:9100/health
```

### C. Referencias cruzadas a otros blueprints

| Para | Ver |
|---|---|
| Arquitectura completa de EspritOS V1-V4 | `BLUEPRINT.md` (sec. 5 chat IA, sec. 10 Build Order olas A-G) |
| Roadmap SaaS post-piloto (V5) | `BLUEPRINT-V5.md` (9 sprints / 77 steps / ~894h) |
| Cambios al Blueprint del 2026-04-11 | `CAMBIOS-BLUEPRINT-20260411.md` |
| Checklist de paridad CRM-ERP v4 | `PARITY-CHECKLIST.md` |
| Cómo retomar desde cero | `START-HERE.md` |
| Kickoff de V5 | `KICKOFF-V5.md` |
| Kickoff de este sprint piloto | sección 9 de este blueprint |

### D. Decisiones tomadas en este sprint (registro)

| Fecha | Decisión | Razón |
|---|---|---|
| 2026-04-22 | Diferir piloto del 24/04 al 1/05 | Costo de 9 días << costo de mala primera impresión con 5 users |
| 2026-04-22 | Lealtad + Promociones se construyen, Rutero se oculta | Lealtad/Promo son visibles al cliente, Rutero no aplica al piloto sin vendedor en ruta |
| 2026-04-22 | Hardware POS bridge entra en el sprint | Operacionalmente "Ctrl+P" se siente a demo, daña percepción día 1 |
| 2026-04-22 | PuntoZero queda activo como red de seguridad durante semana del piloto | Zero downside de tenerlo prendido, gran upside si EspritOS cae |
| 2026-04-22 | 6 decisiones SaaS quedan deferidas hasta cierre del piloto (jueves 7/05) | Datos del piloto informan mejor las decisiones que decidir a ciegas ahora |
| 2026-04-22 | Bugs B post-UAT se arreglan en O4.3, bugs C/D van a backlog post-piloto | Mantener foco en arrancar |

### E. Métricas de éxito del piloto (para evaluar el 7/05)

| Métrica | Umbral verde | Umbral amarillo | Umbral rojo |
|---|---|---|---|
| % ventas registradas en EspritOS vs total | ≥ 90% | 70-90% | < 70% |
| Incidentes severity A | 0 | 1-2 | ≥ 3 |
| Incidentes severity B | ≤ 5 | 6-15 | > 15 |
| Tiempo promedio venta POS | ≤ 30s | 31-60s | > 60s |
| CFDI emitidos exitosamente | ≥ 95% intentos | 80-95% | < 80% |
| NPS interno (encuesta día 7) | ≥ 7/10 | 5-7 | < 5 |
| ¿Equipo dice "sigamos con EspritOS"? | Sí unánime | Mayoría | No / dividido |

**Decisión post-piloto** (tarde del jueves 7/05 con Beto + Bernardo + los 5 users):
- 🟢 verde en todas → cutover gradual + arrancar V5 sprint 1
- 🟡 amarillo en 2+ → extender piloto 1 semana + fixes
- 🔴 rojo en cualquier → roll-back a PuntoZero + post-mortem antes de reintentar

---

## 11. Reglas no negociables del sprint

1. **Cero deploys productivos sin Beto presente.** Aplica a: O1.2 (deploy offline+reportes), O2.2 (StockLedger 5.1M), O2.3 (Facturama prod si aplica), O4.x (cualquier deploy de fixes).
2. **Cero merge a `main` con tests rojos.** Si la rama de la ola tiene rojo, se arregla antes de mergear.
3. **Cero alcance fuera del sprint.** Lo que no esté en este blueprint, no se construye. Si surge necesidad → escalar a Beto, no improvisar.
4. **Cada step termina con commit verificable.** Sin excepciones.
5. **Patrón existente > patrón nuevo.** Antes de inventar estructura, copia del módulo existente más cercano.
6. **Manuales se actualizan con capturas REALES de la prod actualizada**, no con mockups ni de versiones anteriores.
7. **Hardware POS se prueba en tienda, no en laboratorio.** O2.1 requiere viaje físico a tienda con Beto.
8. **PuntoZero no se apaga durante la semana del piloto.** Es la red de seguridad.
9. **Si un user del piloto dice "esto no funciona"**, se investiga en < 30 minutos antes de descartarlo. Las observaciones del equipo de la tienda son ground truth, no opiniones.
10. **El piloto NO es una prueba académica.** Es operación real con clientes reales. Cualquier decisión que aumente el riesgo a la operación se escala a Beto antes de tomarla.

---

**FIN DEL BLUEPRINT PRE-PILOTO**

*Este documento es vivo durante el sprint — actualízalo si encuentras desviaciones materiales (no para registrar progreso, eso va en commits y `docs/piloto/findings/`).*
