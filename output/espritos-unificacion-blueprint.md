# Unificación Pulso + Ritmo → EspritOS — Blueprint

> Generado por The Architect el 10/06/2026
> Arquetipo: Migración / Consolidación de plataformas internas (Django → Django)
> Audiencia: una instancia de Claude Code con CERO contexto previo, ejecutando dentro del repo EspritOS

---

## 0. Cómo usar este blueprint

Este documento es autocontenido. Cada fase es un proyecto cerrado con criterio de salida verificable. **Regla maestra: ninguna fase inicia sin que la anterior haya terminado en cutover completo (containers viejos apagados, repo viejo archivado).** El antipatrón documentado de Cremería HM son las migraciones a medias (VentasHM, Expenses, POS) — este blueprint existe para no repetirlo.

Rutas absolutas de los repos involucrados (Windows, host de producción):

| Repo | Path | Rol en este plan |
|---|---|---|
| EspritOS | `E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS` | **Destino.** Todo el trabajo de código ocurre aquí |
| pulso-hm | `E:\ClaudeWorks\proyectos\CremeriaHM\pulso-hm` | Origen Fase 2. Solo lectura; se archiva al final |
| Ritmo | `E:\ClaudeWorks\proyectos\CremeriaHM\Ritmo` | Origen Fase 1. Solo lectura; se archiva al final |
| datos-hm | `E:\ClaudeWorks\proyectos\CremeriaHM\datos-hm` | Backend canónico de datos. Solo se toca para EXTENDER el canon (prereq Fase 2) |

---

## 1. Visión y metas

### Visión

Cremería HM opera hoy tres aplicaciones Django 5.1 internas con stacks idénticos (Postgres + HTMX/Alpine/Tailwind + Celery donde aplica): **EspritOS** (toolkit de vendedora, 3 apps activas + 18 standby), **Pulso** (BI de dirección, 16 dashboards, push semanal por email) y **Ritmo** (velocidad de venta, inventarios y pedidos, 2 usuarios). Las tres ya leen el mismo canon de datos (`analitica_hm`, Postgres 18 nativo del host). Mantener tres clusters Docker, tres pipelines de auth, tres sistemas de tareas programadas y lógica de negocio duplicada (constantes de calificación ESPEJO con divergencia conocida en Lacret) es costo puro sin beneficio.

Este blueprint consolida Pulso y Ritmo como apps Django dentro de EspritOS, dejando **un solo cluster interno** (`cremeriahm-prod` / `espritos-db`), **un solo login**, **un solo Celery beat**, y **datos-hm intacto como backend canónico**. Lo que se muda es presentación, configuración y lógica de aplicación — nunca el canon.

### Metas

1. Eliminar la duplicación de lógica de calificación (constantes ESPEJO Pulso↔EspritOS) — la calificación Imán/Tesoro/Sólido/Trampa/Herencia vive SOLO en `apps/rentabilidad`.
2. Eliminar la fragilidad operativa de Ritmo (ETL por Task Scheduler de Windows que falló 9 y 11 días) moviéndolo a Celery beat monitoreado.
3. Reducir de 3 stacks de mantenimiento a 1 para apps internas.
4. Reforzar el pivote "toolkit de vendedora": comisiones, metas y cartera visibles donde las vendedoras ya entran a diario.
5. Cero regresión funcional: el push semanal a dirección, los pedidos sugeridos y los dashboards producen resultados idénticos antes y después (verificación lado a lado en cada cutover).

### Métricas de éxito

- Containers `pulso-*` (5) y `ritmo-*` (2) apagados y removidos del host.
- `pulso_reader_espritos` y las constantes ESPEJO eliminadas; divergencia Lacret resuelta en un solo lugar.
- 100% de selectores de Pulso leyendo canon extendido; `pulso.clean_ventas` muerto.
- Suite de tests de EspritOS verde incluyendo ~188 tests portados de Ritmo y 218+ de Pulso.
- Correo semanal a dirección sale de EspritOS con contenido idéntico (dry-run comparado) al de Pulso.

---

## 2. Estado verificado de los repos (10/06/2026)

Estos hechos fueron verificados contra el filesystem real — NO contra memoria ni CLAUDE.md del hub (ambos desactualizados respecto a Ritmo):

### EspritOS (destino)
- `apps/` contiene 33 apps. Activas: `crm`, `agenda`, `aprendizaje`. Relevantes para este plan: `core` (RBAC vía `URL_TO_MODULO` en `apps/core/middleware.py`), `datos_hm` (modelos unmanaged que leen el canon con fallback graceful), `rentabilidad` (calificación rolling 12M, merged de pricing-hm), y las standby `comisiones`, `catalogo`, `inventario`, `notifications`, `gastos`, `reportes`, `etl`.
- Design system documentado en `docs/design-system.md` (dark mode ámbar). ~2,500 tests con markers `critical`. Celery beat con DatabaseScheduler.
- Regla de arquitectura vigente: **las apps solo importan de `core`**.

### Ritmo (origen Fase 1)
- **Ya es Django 5.1.** Container productivo `ritmo-django` (confirmado en `docker-compose.windows.yml:58`). Streamlit archivado en `_archivado/`.
- `django/apps/` contiene 6 apps: `catalogo`, `core`, `criticos`, `inventario`, `pedidos`, `velocidad`. **Tres colisionan con nombres de apps existentes en EspritOS** (`catalogo`, `core`, `inventario`).
- `compartido/` es la joya: paquetes framework-agnostic `velocidad`, `pedidos`, `catalogo`, `etl`, `orquestacion`, `persistencia`. Motor de velocidad en kg/día hábil con calendario HM, fórmula de pedido `need = vel × horizonte − inv + colchón` con redondeo a cajas, expansión de combos/BOM.
- Soporta canon como fuente (`RITMO_VENTAS_FUENTE=canon` → `mart.venta_detalle`, fallback MySQL PZ). Cache local `ventas_diarias_sku` (28× más rápido).
- ~188 tests. Migración documentada en `docs/BLUEPRINT-MIGRACION-DJANGO.md` del repo Ritmo.

### pulso-hm (origen Fase 2)
- `apps/` contiene: `alerts`, `api`, `config_negocio`, `core`, `cuentas`, `dashboards`, `exports`, `finanzas`, `insights`, `notifications`, `warehouse`. **`core`, `notifications` colisionan con EspritOS.**
- Activos irremplazables: `config_negocio` (motor de comisiones: tiers editables, bajada de metas canal→vendedora con prorrateo estacional), `notifications` (push semanal idempotente y auditado), `alerts` (4 detectores), 30 selectores en `apps/dashboards/selectors/` (~8,129 líneas).
- Acoplamiento existente con EspritOS: `pulso_reader_espritos` (SQL read-only cross-app) + constantes ESPEJO de calificación con divergencia Lacret conocida. Referencias en `apps/dashboards/selectors/calificacion_rolling.py`, `apps/warehouse/routers.py`, `pulso/settings/base.py`, entre otros.
- Warehouse propio (`raw→clean→fact→mart`) semi-deprecado: regla escrita "análisis nuevo lee el CANON", pero los drill-downs (`/cremeria/`, `/abarrotera/`, `/consolidado/`, `/segmentos/`) aún recalculan sobre `clean_ventas` porque el canon no expone grano vendedor/producto fino.
- Compromiso operativo vivo: **primer correo real a dirección el lunes 06/07/2026.** Nada de la Fase 2 inicia antes de esa fecha + 2-3 ciclos estables.

### datos-hm (NO se toca, salvo extensión puntual)
- Canon v9 en `analitica_hm`, Postgres 18 **nativo del host** (fuera de Docker). Fuente única de utilidad/margen validada al <0.02%. Los tres sistemas ya lo leen.

---

## 3. Decisiones a ratificar (GATE de Fase 0)

El builder NO procede más allá de la Fase 0 sin que Beto ratifique estas 5 decisiones y queden registradas en `memory/decisiones.md`:

| # | Decisión previa | Qué hace este plan | Ratificación requerida |
|---|---|---|---|
| 1 | "Una app, un cluster; compartir por API, no SQL" (16/05) | La **revierte para apps internas**. Nueva regla: EspritOS = única app interna. Portal/Termómetro/Prospectos (audiencia externa) siguen separados | SÍ — cambio de regla arquitectónica |
| 2 | Pivote toolkit 04/06: "nada en apps standby sin autorización" | Este plan ES la autorización explícita (revive `comisiones`, `notifications`, `gastos`) | SÍ — autorización formal |
| 3 | "Pulso = frontend único del canon" (27/05) | Transfiere el título a EspritOS | SÍ — relevo de rol |
| 4 | Canon como fuente única (25/05) | La respeta y profundiza: el warehouse de Pulso muere, lo que falte se agrega AL canon | No (compatible) — registrar |
| 5 | Antipatrón de migraciones a medias | Mitigación: cada fase termina en cutover con containers apagados | No (mitigación) — registrar |

**Riesgo estructural aceptado conscientemente:** todo lo interno queda en `espritos-db`. Si EspritOS cae, caen BI, pedidos e inventarios a la vez. Atenuantes: el negocio opera transaccionalmente en Punto Zero (MySQL), los backups nocturnos al NAS existen, y el Paso 13 pendiente (servidor dedicado + UPS) se recomienda como acompañante de la Fase 2.

---

## 4. Arquitectura destino

### Mapa de apps (con resolución de colisiones de nombres)

Las colisiones son reales y obligan a un renombrado explícito en el trasplante:

**Ritmo → EspritOS (Fase 1):** una sola app `apps/ritmo/`. Con 2 usuarios y 6 pantallas, 6 apps separadas serían sobre-ingeniería y multiplicarían las colisiones.

| Origen (Ritmo) | Destino (EspritOS) | Notas |
|---|---|---|
| `compartido/` (completo) | `apps/ritmo/services/` | Trasplante literal; la lógica NO cambia. Es Python puro, se importa tal cual |
| `django/apps/velocidad` | `apps/ritmo/views/velocidad.py` + templates | Re-skin al design system ámbar |
| `django/apps/pedidos` | `apps/ritmo/views/pedidos.py` + templates | Ídem |
| `django/apps/inventario` | `apps/ritmo/views/inventario.py` + templates | NO confundir con `apps/inventario` standby de EspritOS — conviven hasta Fase 3 |
| `django/apps/catalogo` | `apps/ritmo/models.py` (maestro Ritmo) | NO fusionar con `apps/catalogo` standby todavía — Fase 3 |
| `django/apps/criticos` | `apps/ritmo/views/criticos.py` + detector como tarea beat | En Fase 3 se conecta a "Mi Día" |
| `django/apps/core` | Se disuelve | Lo que sea genérico ya existe en `apps/core` de EspritOS |

**Pulso → EspritOS (Fase 2):**

| Origen (Pulso) | Destino (EspritOS) | Notas |
|---|---|---|
| `apps/config_negocio` | `apps/comisiones` (standby, se revive) | GATE: auditar primero el contenido del standby. Si sus modelos chocan, archivar lo standby con migración de squash y portar limpio. Un solo hogar para comisiones |
| `apps/dashboards` (16 dashboards + 30 selectores) | `apps/pulso/` | Conserva identidad de marca. Módulo RBAC `pulso`, grupo `direccion` (Humberto/Laura) |
| `apps/notifications` (push semanal) | `apps/notifications` de EspritOS (standby, se revive) | Portar tareas, templates (`direccion_semana.html/.txt`), idempotencia y auditoría |
| `apps/alerts` (4 detectores) | Tareas beat en `apps/pulso/tasks.py`, emiten vía `apps/notifications` | |
| `apps/finanzas` (captura gastos) | `apps/gastos` (standby, se revive) | |
| `apps/exports` | `apps/pulso/exports.py` | Va con los dashboards |
| `apps/insights` (Ollama) | **FUERA del primer corte** | Evaluar después de Fase 3, si acaso |
| `apps/cuentas` | **Muere** | Auth de EspritOS lo reemplaza |
| `apps/api` | **Auditar y matar** | Si algún consumidor externo lo usa, portar ese endpoint a `apps/pulso`; si no, muere |
| `apps/warehouse` (raw/clean/fact/mart) | **Muere — NO se migra** | Excepción: `FactComisionMensual` (resultados congelados de comisiones) SÍ se porta a `apps/comisiones` porque es operativa, no analítica |
| `apps/core` | Se disuelve | |

### Diagrama de flujo de datos (destino final)

```
MySQL Punto Zero (transaccional, intocable)
        │ ETL nocturno (datos-hm, host)
        ▼
analitica_hm / canon v9 + EXTENSIÓN grano vendedor-producto   ← Postgres 18 host, FUERA de Docker
        │ lectura unmanaged (apps/datos_hm, patrón existente)
        ▼
EspritOS (espritos-db, cluster cremeriahm-prod)
  ├─ apps activas: crm, agenda, aprendizaje
  ├─ apps/ritmo        ← velocidad, pedidos, inventarios, críticos (cache local ventas_diarias_sku)
  ├─ apps/pulso        ← 16 dashboards, 30 selectores, exports (lee SOLO canon)
  ├─ apps/comisiones   ← tiers, metas, overrides, FactComisionMensual
  ├─ apps/notifications← push semanal dirección + alertas
  ├─ apps/gastos       ← captura de gastos
  └─ apps/rentabilidad ← ÚNICA fuente de calificación Imán/Tesoro/Sólido/Trampa/Herencia
```

### Modelo de datos — qué migra, qué muere, qué se extiende

| Datos | Acción | Mecánica |
|---|---|---|
| Ritmo: `productos`, refs externas, `inventarios` (append-only), `pedidos_borrador`, `pedidos_ejecutados`, `ventas_diarias_sku` | **Migran a `espritos-db`** como modelos Django managed | `pg_dump` de `ritmo-db` → restore en schema propio → comando Django de verificación de counts por tabla |
| Pulso: `config_negocio` (tiers, metas, overrides) | **Migra a `espritos-db`** | dumpdata/loaddata o dump SQL + verificación |
| Pulso: `FactComisionMensual` | **Migra** (histórico operativo congelado) | dump + restore + verificación de sumas por mes |
| Pulso: auditoría de notifications | **Migra** (trazabilidad del push) | dump + restore |
| Pulso: `raw`, `clean_ventas`, `dim_*`, `fact_*`, `mart_*` | **Mueren** | El grano faltante se agrega AL CANON (prereq Fase 2), no se muda el warehouse |
| Canon (`analitica_hm`) | **Se extiende**: mart con grano vendedor × producto × día (lo que los drill-downs de Pulso necesitan) | 1-2 sesiones EN datos-hm, validación contra cifras actuales de Pulso |

---

## 5. Orden de construcción (LA SECCIÓN CRÍTICA)

### FASE 0 — Decisión y preparación (1 sesión — puede ser hoy)

**Paso 1. Ratificar y registrar.** Presentar a Beto las 5 decisiones de la Sección 3. Registrar las ratificadas en `memory/decisiones.md` con fecha.

**Paso 2. Corregir memoria desactualizada.** En `E:\ClaudeWorks\CLAUDE.md` (hub), `MEMORY.md` y `memory/decisiones.md`: Ritmo ya es Django 5.1 (migrado ~16/05, Streamlit en `_archivado/`, container `ritmo-django`). La decisión "NO migrar a Django" del 16/05 fue revertida.

**Paso 3. Snapshot de seguridad.** `pg_dump` completos de `ritmo-db` y `pulso-db` al NAS (`F:\Respaldos\` o ruta de backups vigente). Verificar que el dump restaura en un Postgres efímero antes de declararlo válido.

**Paso 4. Registrar el mapa destino** (Sección 4 de este blueprint) como decisión en `memory/decisiones.md`.

**Criterio de salida F0:** decisiones registradas, memoria corregida, dumps verificados.

---

### FASE 1 — Ritmo → EspritOS (2-3 sesiones — junio)

*Por qué primero: mismo stack, 2 usuarios, cero compromisos de calendario, elimina la fragilidad de Task Scheduler.*

**Paso 5. Trasplantar `compartido/` como `apps/ritmo/services/`.**
- Copiar los paquetes `velocidad`, `pedidos`, `catalogo`, `etl`, `orquestacion`, `persistencia` de `Ritmo\compartido\` a `EspritOS\apps\ritmo\services\`.
- Ajustar imports internos (`compartido.velocidad` → `apps.ritmo.services.velocidad`). NO cambiar lógica.
- Portar primero los tests que reproducen NAYAR celda por celda y el pedido SA del 12/05 — son la red de seguridad. Deben pasar ANTES de seguir.

**Paso 6. Modelos y migración de datos.**
- Convertir el `schema.sql` de Ritmo a modelos Django managed en `apps/ritmo/models.py` (productos con sus DOS factores kg, refs externas, inventarios append-only, pedidos_borrador/ejecutados, ventas_diarias_sku).
- `makemigrations` + dump/restore de datos desde `ritmo-db`.
- Comando de management `python manage.py ritmo_verificar_migracion` que compara counts y checksums por tabla origen vs destino. Falla ruidosamente si difieren.

**Paso 7. ETL a Celery beat.**
- Tarea beat nocturna que reemplaza el Task Scheduler, reutilizando el patrón retry/circuit-breaker del ETL existente de EspritOS (`apps/etl`).
- Fuente preferente: canon (`mart.venta_detalle`); fallback MySQL PZ (mismo orden que `RITMO_VENTAS_FUENTE=canon` hoy).
- Mantener el cache local `ventas_diarias_sku` (28× más rápido) — se alimenta de la tarea beat.
- Alerta si el ETL no corre: el modo de falla histórico de Ritmo fue SILENCIOSO (9 días de cache clavado). La tarea registra heartbeat y hay un check que avisa si el último run > 26 horas.

**Paso 8. UI y RBAC.**
- Re-skin de las 6 pantallas al design system de EspritOS (`docs/design-system.md`, dark ámbar). Son templates HTMX/Alpine — costo bajo, no reescribir lógica de vistas.
- Registrar prefijo `/ritmo/` en `URL_TO_MODULO` (`apps/core/middleware.py`), módulo RBAC `ritmo`. Beto admin; Carlos según uso real.

**Paso 9. Tests.**
- Portar los ~188 tests a la suite de EspritOS bajo `apps/ritmo/tests/`.
- `test_aislamiento_espritos.py` se invierte: ya no aplica el aislamiento; reemplazar por tests de integración (servicios de ritmo leyendo canon vía `datos_hm`).
- Marcar `critical` los tests de fórmula de pedido y velocidad.

**Paso 10. Cutover Ritmo.**
1. Una semana de verificación lado a lado: ambos sistemas corriendo, comparar pedido sugerido diario EspritOS vs Ritmo (deben ser idénticos).
2. Apagar `ritmo-django` y `ritmo-db`; remover del compose del host.
3. Tag `archivado-YYYYMMDD` en el repo Ritmo; mover a `proyectos/CremeriaHM/ARCHIVO/`.
4. Actualizar `DEPLOY-WINDOWS.md`, CLAUDE.md hub, `MEMORY.md`, memoria del proyecto.

**Criterio de salida F1:** containers ritmo apagados, repo archivado, suite EspritOS verde con tests de ritmo incluidos, una semana de pedidos idénticos documentada.

---

### PRERREQUISITO FASE 2 — Extensión del canon (1-2 sesiones, EN datos-hm — junio/julio)

**Paso 11. Extender el canon con grano vendedor × producto.**
- En datos-hm: agregar mart(s) con el grano que los drill-downs de Pulso (`/cremeria/`, `/abarrotera/`, `/consolidado/`, `/segmentos/`) hoy recalculan sobre `clean_ventas`.
- Inventario previo: leer los 30 selectores de `pulso-hm\apps\dashboards\selectors\` y listar exactamente qué columnas/granos consumen de `clean_ventas`. Ese listado define el contrato del mart nuevo.
- Validación: cifras del mart nuevo vs cifras actuales de Pulso para 3 semanas de referencia. Tolerancia: igual al estándar del canon (<0.02%).
- **Cuidado con los dos bugs históricos del warehouse de Pulso** (IEPS doble-descontado, `t.Facturada` como booleano): si el mart nuevo difiere de Pulso, la discrepancia puede ser un bug DE PULSO — validar contra el canon, no contra Pulso ciegamente.

**Criterio de salida:** mart publicado en `analitica_hm`, validado, documentado en datos-hm.

---

### FASE 2 — Pulso → EspritOS (5-7 sesiones — agosto, DESPUÉS del 06/07 + 2-3 ciclos de correo estables)

**Paso 12. Auditar y revivir `apps/comisiones`.**
- Leer el contenido actual del standby `apps/comisiones` de EspritOS. Si sus modelos son compatibles/vacíos → portar `config_negocio` dentro. Si chocan → archivar lo standby (squash de migraciones) y portar limpio.
- Portar: tiers editables en /admin, bajada de metas canal→vendedora con prorrateo estacional, overrides, comando `pulso_bajar_metas`, y `FactComisionMensual` (con verificación de sumas por mes contra Pulso).
- Migrar datos de `config_negocio` desde `pulso-db`.

**Paso 13. Portar dashboards y selectores → `apps/pulso/`.**
- 30 selectores apuntando 100% al canon extendido (Paso 11). CERO lecturas de `pulso.clean_ventas` — grep de verificación al final.
- Las constantes de calificación NO se portan: los selectores que califican importan de `apps/rentabilidad` (vía servicio, respetando "solo importar de core" — si hace falta, exponer la calificación como servicio en core o como contrato documentado entre apps; decidir en sesión con la regla de elegancia).
- Re-skin al design system. `apps/exports` se porta como `apps/pulso/exports.py`.
- RBAC: prefijo `/pulso/` en `URL_TO_MODULO`, módulo `pulso`, grupo `direccion` (Humberto/Laura) ya existente.

**Paso 14. Portar el push semanal → `apps/notifications` de EspritOS.**
- Portar tareas, templates `direccion_semana.html/.txt`, idempotencia y tabla de auditoría.
- **El cutover más delicado del plan:** dry-run en EspritOS comparado byte-a-byte (o cifra-a-cifra) contra el correo real de Pulso durante 2 lunes. Tercer lunes: el correo sale de EspritOS, Pulso en standby listo para reactivarse. Cuarto lunes: Pulso ya no manda.

**Paso 15. Portar alertas y finanzas.**
- 4 detectores de `apps/alerts` como tareas beat en `apps/pulso/tasks.py`, emitiendo vía `apps/notifications`.
- `apps/finanzas` (captura de gastos) → revivir `apps/gastos` de EspritOS.
- `apps/insights` (Ollama): NO se porta en este corte.

**Paso 16. Matar la deuda de acoplamiento.**
- Eliminar `pulso_reader_espritos` (settings, routers, permisos SQL).
- Eliminar constantes ESPEJO; resolver la divergencia Lacret en `apps/rentabilidad` (decidir el valor correcto UNA vez, documentarlo).
- Auditar `apps/api` de Pulso: si nadie lo consume, muere; si algo lo consume, portar solo ese endpoint.

**Paso 17. Cutover Pulso.**
1. Ingress: `pulso.espritos.app` → redirect a `espritos.app` (editar `infra/cloudflared/`).
2. Apagar los 5 containers `pulso-*`; remover del compose.
3. Tag `archivado-YYYYMMDD`; mover repo a `ARCHIVO/`.
4. Actualizar `DEPLOY-WINDOWS.md`, CLAUDE.md hub, memoria.

**Criterio de salida F2:** correo semanal saliendo de EspritOS 2+ lunes consecutivos sin incidente, containers pulso apagados, repo archivado, grep confirma cero referencias a `clean_ventas` y `pulso_reader_espritos`.

---

### FASE 3 — Consolidación (2-3 sesiones — septiembre, sin prisa)

**Paso 18. Unificar catálogo maestro.** Fusionar el maestro de productos de `apps/ritmo` con `apps/catalogo` (standby): un solo maestro con los dos factores kg como campos. Migración de datos con verificación.

**Paso 19. Conectar lo que la separación impedía.**
- Críticos de Ritmo como alertas en "Mi Día".
- Velocidad de venta visible en la ficha de producto del CRM.
- Comisión de la vendedora junto a su cartera.

**Paso 20. Limpieza de infraestructura.**
- Retirar users MySQL `pulso_reader` / `ritmo_reader` (queda `espritos_reader`).
- Borrar volúmenes Docker huérfanos.
- Verificación final de `DEPLOY-WINDOWS.md` contra la realidad del host.

---

## 6. Estrategia de pruebas

| Capa | Qué | Cómo |
|---|---|---|
| Lógica trasplantada | `compartido/` de Ritmo, selectores de Pulso | Tests existentes portados SIN modificar expectativas (NAYAR celda por celda, pedido SA 12/05, 218+ de Pulso). Si un test falla tras el trasplante, el trasplante está mal — no el test |
| Migración de datos | Cada dump/restore | Comando de verificación counts + checksums; sumas por mes para `FactComisionMensual` |
| Equivalencia funcional | Pedido sugerido (F1), correo semanal (F2), drill-downs (F2) | Lado a lado: mismo input, output idéntico, documentado antes de cada cutover |
| Canon extendido | Mart nuevo | Validación <0.02% contra canon; discrepancias vs Pulso se investigan (pueden ser bugs de Pulso) |
| Regresión EspritOS | Las 3 apps activas no se rompen | Suite completa + markers `critical` en CI local antes de cada merge |

## 7. Rollback por fase

- **F1:** mientras `ritmo-db` no se borre (solo se apaga), reactivar el compose viejo restaura Ritmo en minutos. No borrar volúmenes hasta cerrar F3.
- **F2 (correo):** Pulso queda en standby reactivable durante 2 lunes post-cutover. El dump de F0 + volúmenes intactos permiten volver atrás.
- **Regla:** los volúmenes de `ritmo-db` y `pulso-db` se borran SOLO en el Paso 20, nunca antes.

## 8. Qué NO se toca (anti-scope-creep)

1. **datos-hm**: sigue en el host, con sus marts y ETL nocturno. Solo se EXTIENDE (Paso 11). Es más valioso separado.
2. **Portal, Termómetro, Prospectos**: audiencia externa, aislados. La regla "una app, un cluster" sigue viva para ellos.
3. **MySQL Punto Zero**: transaccional de facto. Este plan no acerca ni aleja su cutover.
4. **`apps/insights` (Ollama)**: fuera del corte.
5. **Las 18 apps standby de EspritOS no mencionadas**: siguen en standby. Este plan solo autoriza revivir `comisiones`, `notifications`, `gastos` (+ `catalogo`/`inventario` en F3).

## 9. Skills a usar durante el build

| Skill | Cuándo | Para qué |
|---|---|---|
| `/retoma` | Inicio de cada sesión | Cargar contexto del proyecto |
| `/gsd:plan-phase` + `/gsd:execute-phase` | Cada fase | EspritOS ya usa GSD (`.planning/`); cada fase de este blueprint = milestone/fases GSD |
| `/systematic-debugging` | Si la verificación lado a lado difiere | Encontrar causa raíz antes de cutover |
| `/code-review` | Antes de cada cutover | Revisión del diff de la fase |
| `/resumen-jornada` | Fin de cada sesión | Memoria actualizada (mitiga el riesgo de migración a medias entre sesiones) |

## 10. Adiciones a CLAUDE.md (no reemplazos — EspritOS ya tiene CLAUDE.md)

**Al CLAUDE.md de EspritOS, agregar al aprobar Fase 0:**

```markdown
## Unificación en curso (blueprint: the-architect/output/espritos-unificacion-blueprint.md)

- EspritOS es la ÚNICA app interna de HM. Pulso y Ritmo se integran como apps (F1: ritmo, F2: pulso/comisiones/notifications/gastos).
- La calificación Imán/Tesoro/Sólido/Trampa/Herencia vive SOLO en `apps/rentabilidad`. Prohibido duplicar constantes.
- Todo análisis lee el canon (`analitica_hm` vía `apps/datos_hm`). Prohibido leer `pulso.clean_ventas` o crear warehouses locales.
- Regla de fases: no se inicia una fase sin cutover completo de la anterior (containers apagados, repo archivado).
- Volúmenes de `ritmo-db`/`pulso-db` NO se borran hasta cerrar Fase 3.
```

**Al CLAUDE.md del hub (`E:\ClaudeWorks\CLAUDE.md`), corregir:**
- Ritmo: ~~"Streamlit (blueprint Django pendiente)"~~ → "Django 5.1 (migrado 16/05/2026) — EN INTEGRACIÓN a EspritOS"
- Al cierre de cada fase: actualizar tabla de proyectos productivos (quitar fila de Ritmo en F1, de Pulso en F2).

## 11. Reglas no negociables para el builder

1. **Cero cambios de lógica durante el trasplante.** Mover ≠ mejorar. Las mejoras van en sesiones posteriores al cutover.
2. **Los tests portados son el contrato.** Si fallan tras el trasplante, el error está en el trasplante.
3. **Ningún cutover sin verificación lado a lado documentada.**
4. **Una fase a la vez.** No se abre F2 con F1 a medias. El patrón VentasHM/Expenses/POS es el riesgo #1 documentado.
5. **datos-hm es solo-extensión.** Nada de este plan modifica marts existentes del canon.
6. **Apps solo importan de `core`** (regla vigente de EspritOS). Si la calificación de `rentabilidad` se necesita cross-app, se expone como servicio explícito, no como import directo ni constante copiada.
7. **Volúmenes de BD origen se conservan hasta Fase 3, Paso 20.**
8. **Memoria al día en cada cierre de sesión** (`/resumen-jornada`): el riesgo de este proyecto es perder el hilo entre sesiones.

## 12. Calendario

| Fase | Sesiones | Cuándo | Bloqueador |
|---|---|---|---|
| 0 — Decisión y prep | 1 | Ya (junio) | Ratificación de Beto |
| 1 — Ritmo | 2-3 | Junio | F0 cerrada |
| Extensión canon | 1-2 | Junio-julio (en datos-hm) | Puede correr en paralelo a F1 |
| 2 — Pulso | 5-7 | Agosto | 06/07 + 2-3 correos estables + canon extendido + F1 cerrada |
| 3 — Consolidación | 2-3 | Septiembre | F2 cerrada |

**Total estimado: 11-16 sesiones** repartidas en ~4 meses, con los cutovers como compuertas.
