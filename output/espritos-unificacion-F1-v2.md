# Consolidación HM en EspritOS — Blueprint F1-v2 (Reconciliación + Cutover)

> Generado por The Architect el 24/06/2026. Reemplaza operativamente —no borra— a
> `espritos-unificacion-blueprint.md` (10/06) y `espritos-merge-blueprint.md` (08/06),
> que fueron blueprints de **construcción**. Aquellos cumplieron: el código YA está
> portado. Este blueprint cubre la etapa real de hoy: **reconciliar deltas sin
> commitear, validar paridad y apagar los containers legacy**.
>
> **Audiencia:** una instancia de Claude Code (el Builder) con CERO contexto previo,
> ejecutando dentro del host de producción de Cremería HM (Windows).
> **Stack:** Django 5.1 (sin cambios de framework). **Idioma:** español.

---

## 0. Cómo usar este blueprint

Es autocontenido. Ejecuta el **orden de construcción numerado** (Sección 5) de Paso 1
en adelante. Cada ola cierra con un **criterio de aceptación verificable**; no se abre
una ola sin cerrar la anterior. El antipatrón documentado de HM son las migraciones a
medias (VentasHM, Expenses, POS) — este blueprint existe para no repetirlo.

**Regla maestra:** una fase no inicia sin que la anterior haya terminado en cutover
completo (container viejo apagado, repo archivado, memoria actualizada).

### Estado de verdad por repo
El estado y pendientes de cada repo viven en su `CLAUDE.md` + `NEXT_SESSION.md`. La
consolidación NO crea documentos paralelos de estado: lo que entra a EspritOS pierde su
bullet en `MEMORY.md` y su MOC propio (regla de oro de `E:\ClaudeWorks\CONSOLIDACION.md`).

---

## 1. Contexto estratégico (lo mínimo para operar sin más contexto)

- **EspritOS** es el monolito Django 5.1 (+ Postgres 16/pgvector, Redis, Celery) que es
  el núcleo del cluster Cremería HM. Meta (decidida 19/06/2026): fundir TODA app web
  Django/Python de HM dentro de EspritOS — **un solo deploy, una sola fuente de verdad
  por dato**.
- **datos-hm NO funde.** Es la capa de datos/canon (`analitica_hm`, Postgres host) que
  EspritOS **consume read-only**. Prohibido meter ETL dentro de la app web. Si un dato
  analítico no existe en el canon, se **extiende EN datos-hm**, no se consulta MySQL PZ
  desde la app.
- **Satélites que NO funden:** Termómetro y Prospectos (Next.js/Vercel), sitios
  estáticos (cremeriahm.com, lacret.mx), AnalisisVentas (I+D del canon). **Portal**
  queda como satélite (ver veredicto §6.4).

### Rutas absolutas (Windows, host de producción)

| Repo | Path | Rol en este blueprint |
|---|---|---|
| EspritOS | `E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS` | **Destino.** Todo el código se trabaja aquí |
| pricing-hm | `E:\ClaudeWorks\proyectos\CremeriaHM\pricing-hm` | Origen Ola A. Se archiva al cierre |
| pulso-hm | `E:\ClaudeWorks\proyectos\CremeriaHM\pulso-hm` | Origen Ola B. Se archiva al cierre |
| Ritmo | `E:\ClaudeWorks\proyectos\CremeriaHM\Ritmo` | Ya portado. Solo se apaga/archiva (Ola C) |
| datos-hm | `E:\ClaudeWorks\proyectos\CremeriaHM\datos-hm` | Canon. Solo lectura; NO se toca en este tramo |
| Doc maestro | `E:\ClaudeWorks\CONSOLIDACION.md` | Roadmap general de consolidación |

---

## 2. Estado REAL verificado (24/06/2026) — leer antes de tocar nada

Verificado contra el filesystem y git, **no contra los blueprints previos** (que
describen la fase de construcción, ya superada).

### Lo que YA está hecho (no rehacer)
- **`apps/ritmo`** → portado 1:1 (modelos + ~37 archivos de servicios con la lógica de
  velocidad/pedidos en SQL, ~35 archivos de tests). `/ritmo/` registrado en RBAC.
- **`apps/pulso`** → **portado** (NO es andamiaje): **29 selectors** (vs 26 en pulso-hm),
  16 módulos de vistas, templates completos. `/pulso/` redirige internamente a `master`.
  **No existe redirect a `:8500`** (el container `pulso-web` sigue arriba como fallback,
  pero EspritOS ya no lo consume).
  - Deudas que un análisis ingenuo marcaría, **ya resueltas en EspritOS**:
    `insights.engine` (Ollama) → reemplazado por `selectors/resumen_canon.py`
    (determinista, lee canon); tabla de metas → `apps/comisiones` (`MetaMensual`,
    `MetaProducto`) + `selectors/metas.py`.
  - EspritOS va **adelante** de pulso-hm: tiene `kpis.py`, `pnl.py`, `exclusiones.py`,
    `resumen_canon.py` que pulso-hm no tiene.
- **`apps/rentabilidad`**, **`apps/pricing_calc`**, **`apps/precios`** → funcionales
  (vienen de pricing-hm). **`apps/ventas_mayoreo`**, **`apps/cfdi`** → funcionales.
- **RBAC ya registrado** en `apps/core/middleware.py` (`URL_TO_MODULO`): `/ritmo/`,
  `/pulso/`, `/precios/`, `/pricing/`, `/rentabilidad/`, `/cotizaciones/`, `/cfdi/`.
- **Conexión al canon ya configurada:** alias Django `datos_hm` + `DatosHmRouter`
  (`apps/datos_hm/router.py`), modelos unmanaged `DimCliente`, `VentaDetalle`,
  `ClienteMensual` (`apps/datos_hm/models.py`), env `DATOS_HM_URL`. Lectura vía
  `connections["datos_hm"]` o el ORM unmanaged con `.using("datos_hm")`.

### Lo que falta (este blueprint)
- **Reconciliar pricing-hm:** 22 archivos sin commitear en la rama
  `feature/calificacion-visita` (riesgo de pérdida de trabajo). Parte ya está en
  EspritOS; hay delta genuino sin portar.
- **Reconciliar Pulso:** 5 selectors con delta sin commitear en pulso-hm + 1 template.
- **Apagar legacy:** containers `ritmo-django` y `pulso-web` (`:8500`) siguen arriba.

### Fases futuras (fuera del primer tramo — §6)
- **RH-v2 (Yunuen)** → `apps/documentos` **no existe**; es build real.
- **Impuestos (Montse, FastAPI)** → no portado; deuda de seguridad (creds MySQL
  hardcodeadas en `services/pipeline.py`).
- **Portal (clientes mayoristas, OTP Twilio)** → veredicto: **satélite, no funde** (§6.4).

### Containers legacy aún arriba (apagado por fase, fallback intencional)
`ritmo-django`, `pulso-web` (LAN :8500), `pricing-web` (LAN :8300), `portal-web`
(portal.cremeriahm.com), `rh-v2-web` (LAN :8000), `impuestos-web` (LAN :8400).

---

## 3. Reglas no negociables para el Builder

1. **Repo-first, commits atómicos, impacto mínimo, simplicidad primero.** Mover ≠
   mejorar. Cero refactor "de paso".
2. **Reusar todo lo ya escrito y probado.** No reescribir lo que ya funciona en EspritOS.
   El trabajo es *diff y reconcilia*, no *reconstruir*.
3. **Mismo stack que EspritOS (Django 5.1). Cero migraciones de framework.**
4. **Frontera datos-hm:** EspritOS LEE el canon vía `datos_hm`. Prohibido meter ETL en
   la app web, leer `pulso.clean_ventas`, crear warehouses locales, o importar un driver
   MySQL en una app nueva (hay test `critical` `apps/core/tests/test_ingesta_canon.py`
   que rompe CI si se viola). Si falta un dato, se extiende EN datos-hm.
5. **Apps solo importan de `core`** (enforced `apps/core/tests/test_app_isolation.py`).
   Cross-app vía services en core o signals. La calificación Imán/Tesoro/Sólido/Trampa/
   Herencia vive SOLO en `apps/rentabilidad`; prohibido duplicar constantes.
6. **Cero cambios de lógica durante el trasplante.** Los tests portados son el contrato:
   si fallan tras el trasplante, el error está en el trasplante, no en el test.
7. **Ningún cutover sin paridad lado a lado documentada** (mismo input → output idéntico).
8. **Una fase a la vez.** No se abre la siguiente ola con la anterior a medias.
9. **Volúmenes de BD origen (`ritmo-db`, `pulso-db`, `pricing-db`) NO se borran** en este
   tramo; solo se apagan los containers. Borrado diferido (rollback vivo).
10. **Deploy SOLO desde `master` con todo mergeado.** La imagen se hornea del working
    tree (`COPY . .`), no de git: deployar una rama atrasada revierte prod (pasó 18/06).
    Vía bendecida: `scripts\deploy_prod.ps1` (mete el gate adentro: exige master limpio,
    corre suite completa, build + up + verifica `/health/`).
11. **Tests SIEMPRE aislados:** `scripts\test_docker.ps1` (proyecto `espritos-test`).
    Un `docker compose up` a secas recrea `espritos-db`/`espritos-redis` de PROD y tumba
    espritos.app (pasó 10/06).
12. **Comentarios Django multilínea:** usar `{% comment %}…{% endcomment %}`. `{# … #}`
    SOLO en una línea (un `{# #}` partido en varias líneas rompe la página entera —
    pasó 5+ veces). Validar antes de cada edit de `.html` con comentarios.
13. **Memoria al día al cerrar cada sesión** (`/resumen-jornada`). El riesgo #1 de este
    proyecto es perder el hilo entre sesiones.

### Comandos clave (verificados)
```powershell
# Tests aislados (NO tocan dev/prod)
.\scripts\test_docker.ps1                      # suite completa
.\scripts\test_docker.ps1 -m critical          # solo critical
.\scripts\test_docker.ps1 apps\pulso\tests     # path concreto
.\scripts\test_docker.ps1 --create-db          # BD limpia (tras cambios de migración)

# Deploy a prod (gate + build + deploy + health). Exige master limpio y suite verde.
.\scripts\deploy_prod.ps1
.\scripts\deploy_prod.ps1 -DryRun              # corre el gate y PARA antes de build/deploy
```

---

## 4. Cierre de cada fold (ritual obligatorio)

Cada app consolidada cierra IGUAL:
1. **Paridad documentada** lado a lado (legacy vs EspritOS) — output idéntico.
2. **Tests verdes** en EspritOS incluyendo los portados (`scripts\test_docker.ps1`).
3. **RBAC** del prefijo registrado en `URL_TO_MODULO` (ya hecho para ritmo/pulso/pricing).
4. **Apagar** el container legacy (NO borrar volumen) y removerlo del compose del host.
5. **Tag** `archivado-YYYYMMDD` en el repo origen + mover a
   `proyectos\CremeriaHM\ARCHIVO\`.
6. **Memoria:** actualizar `E:\ClaudeWorks\CLAUDE.md` (hub), `MEMORY.md` (quitar bullet),
   `DEPLOY-WINDOWS.md`, y el MOC en la bóveda. El estado pasa a vivir SOLO en EspritOS.

---

## 5. Orden de construcción (LA SECCIÓN CRÍTICA)

Tres olas, máximo valor / mínimo riesgo. Pasos numerados de corrido.

### OLA A — Reconciliación pricing-hm (1-2 sesiones)

*Objetivo: que el trabajo sin commitear deje de estar en peligro y que el delta genuino
quede en EspritOS, sin reescribir lo que ya está portado.*

**Paso 1 — Asegurar el trabajo en peligro (pricing-hm).**
En `pricing-hm`, rama `feature/calificacion-visita`, commitear el working tree (22
archivos) ANTES de tocar nada. Es captura de seguridad, no merge.
```powershell
cd E:\ClaudeWorks\proyectos\CremeriaHM\pricing-hm
git add -A
git commit -m "wip(rentabilidad): snapshot calificacion-visita antes de consolidar a EspritOS"
```
*Criterio:* `git status` limpio en pricing-hm; el trabajo queda en historia git (reversible).

**Paso 2 — Diff dirigido repo↔EspritOS.**
Para cada archivo modificado/nuevo de `feature/calificacion-visita`, comparar contra su
equivalente en `EspritOS\apps\rentabilidad\` y clasificar: **YA en EspritOS** (no portar) /
**DELTA** (portar). Tabla de reconciliación de referencia (validar con diff real):

| Archivo pricing-hm | ¿En EspritOS? | Acción |
|---|---|---|
| `services/canon.py` (`buscar_clientes` JOIN dim_cliente) | Sí (equivalente) | Verificar paridad; no portar si idéntico |
| `services/rolling.py` (`agregar_clientes(dim=...)`) | Sí (equivalente) | Verificar paridad; no portar si idéntico |
| `services/rentabilidad.py` (totales `n_bajo_benchmark`, `potencial_total`…) | Sí (equivalente) | Verificar paridad; no portar si idéntico |
| `services/access.py` | Sí — EspritOS tiene fallback `profile.vendedor_canon` (más robusto) | **Adoptar versión EspritOS**; no sobreescribir |
| `services/dim_cliente.py` | Sí pero distinto (EspritOS usa `connections["datos_hm"]`) | **Adoptar versión EspritOS** (psycopg2 directo NO entra) |
| `management/commands/cargar_cualitativo_xlsx.py` | **No** | **Copiar** a `apps/rentabilidad/management/commands/` |
| `tests/test_dim_cliente.py` | **No** | **Copiar y adaptar** a `datos_hm` |
| `tests/test_access.py` | Sí (subset) | Consolidar tests faltantes |
| `apps/cuenta/migrations/0003_usuario_vendedor_canon.py` + `cuenta/models.py` `vendedor_canon` | Patrón distinto | **Ver Paso 3** (decisión de modelo) |
| `config/settings/base.py` (`VENDEDOR_MAP`, `CANON_PG`, `ALLOWED_EMAILS`) | Parcial | **Portar `VENDEDOR_MAP` + hidratación** (Paso 3) |
| `templates/rentabilidad/*.html` (7), `_tabla_clientes_op.html` | Parcial (tier1 sí, tier2 parcial) | Portar delta de UX que falte; tier2 (voz/sticky) opcional |

*Criterio:* documento corto `docs/unificacion/reconciliacion-pricing.md` en EspritOS con
la tabla anterior resuelta contra el diff real (archivo → YA/DELTA → acción).

**Paso 3 — Portar el delta genuino a EspritOS (rama `feature/reconcilia-pricing`).**
- `vendedor_canon`: **adoptar el patrón EspritOS = `UserProfile`** (decisión ratificada
  por Beto 24/06). NO copiar el campo a `Usuario`. Si `UserProfile.vendedor_canon` no
  existe, crearlo con migración; poblar para cada vendedora.
- `VENDEDOR_MAP` (env, vía `django-environ` en `config/settings/`, NUNCA `os.environ`
  directo) + hidratación de `vendedor_canon` post-login (señal/auth pipeline).
- Copiar `cargar_cualitativo_xlsx` (management command) y `test_dim_cliente.py` (adaptado
  a `connections["datos_hm"]`).
- Consolidar tests de access faltantes.
*Criterio:* `scripts\test_docker.ps1 apps\rentabilidad` verde, incluyendo los tests
copiados; `apps/rentabilidad` y `apps/pricing_calc` arrancan sin error.

**Paso 4 — Paridad de cálculo rentabilidad.**
Comparar la calificación de cartera (clientes y productos) y los totales de oportunidades
entre `pricing-web` (:8300) y `/rentabilidad/` de EspritOS, para 1 vendedora real y el
consolidado. Deben coincidir.
*Criterio:* paridad documentada (capturas/cifras) en `docs/unificacion/reconciliacion-pricing.md`.

---

### OLA B — Pulso: reconciliar + paridad (1-2 sesiones)

*Objetivo: incorporar el delta de los 5 selectors sin commitear de pulso-hm sin pisar lo
que EspritOS ya tiene (que en varios casos va adelante), y demostrar paridad.*

**Paso 5 — Asegurar el trabajo en peligro (pulso-hm).**
En `pulso-hm` (rama `master`), commitear los 5 selectors con delta + el template:
`apps/dashboards/selectors/{canon,clientes,desglose,master,utilidad}.py` y
`templates/dashboards/clientes/index.html`. Los **22 scripts ad-hoc** en `scripts/`
(proyecciones/comisiones de Andrea/Daniela/Valeria) se commitean aparte o se dejan; **no
son infraestructura** y no se portan a EspritOS.
```powershell
cd E:\ClaudeWorks\proyectos\CremeriaHM\pulso-hm
git add apps/dashboards/selectors/ templates/dashboards/clientes/index.html
git commit -m "wip(dashboards): snapshot de selectors con delta antes de reconciliar a EspritOS"
```
*Criterio:* los 5 selectors quedan en historia git.

**Paso 6 — Diff selector-por-selector contra EspritOS.**
Comparar cada uno de los 5 selectors de pulso-hm contra
`EspritOS\apps\pulso\selectors\` y clasificar el delta. Lógica conocida a preservar:
descuento de sobreprecio (11/06), fuga RFM (umbral $15k / caída 50% vs promedio 3m),
cascada de fallback de snapshot en `master`. **Solo portar lo que no esté ya en
EspritOS.** Notas de inventario:
- Selectors **solo en pulso-hm:** `segmentos.py` → **POSPUESTO** (requiere extensión del
  canon; no entra en este tramo).
- Selectors **solo en EspritOS** (no tocar, no traer de vuelta): `exclusiones.py`,
  `kpis.py`, `pnl.py`, `resumen_canon.py`.
*Criterio:* `docs/unificacion/reconciliacion-pulso.md` con la tabla
selector → delta → portado/ya-presente.

**Paso 7 — Portar el delta a EspritOS (rama `feature/reconcilia-pulso`).**
Aplicar solo el delta genuino a `apps/pulso/selectors/`. Cero lecturas de
`clean_ventas`/warehouse local: todo lee canon vía `datos_hm` (regla #4).
*Criterio:* `scripts\test_docker.ps1 apps\pulso\tests` verde.

**Paso 8 — Paridad de dashboards.**
Lado a lado `pulso-web` (:8500) vs `/pulso/` de EspritOS para 3 cortes de referencia en
los drill-downs clave (`/master/`, `/clientes/`, `/utilidad/`, `/cremeria/`,
`/abarrotera/`, `/consolidado/`). Cuidado con los bugs históricos del warehouse de Pulso
(IEPS doble-descontado, `t.Facturada` como booleano): si EspritOS difiere de Pulso, la
discrepancia puede ser un **bug de Pulso** — validar contra el canon, no contra Pulso
ciegamente.
*Criterio:* paridad documentada; discrepancias explicadas (o confirmadas como bug legacy).

---

### OLA C — Apagado de legacy (Ritmo + Pulso) (1 sesión)

*Objetivo: cobrar el premio de la consolidación — un solo cluster — sin perder red de
seguridad.*

**Paso 9 — Merge a master y deploy.**
Mergear `feature/reconcilia-pricing` y `feature/reconcilia-pulso` a `master` (regla #10),
suite completa verde, y `scripts\deploy_prod.ps1`.
*Criterio:* `/health/` OK; `/ritmo/`, `/pulso/`, `/rentabilidad/`, `/pricing/` responden
en prod; smoke test de las 3 apps activas (crm/agenda/aprendizaje) sin regresión.

**Paso 10 — Cutover Ritmo.**
Validación: 1 semana de pedido sugerido idéntico EspritOS vs Ritmo ya cubierta por
paridad; si no está documentada, documentarla antes de apagar. Luego: apagar
`ritmo-django` (+ `ritmo-db`), remover del compose del host. Ritual de cierre §4 (tag
`archivado-YYYYMMDD`, mover a `ARCHIVO/`, memoria).
*Criterio:* `ritmo-django` apagado, repo Ritmo archivado, sin dependencia rota.

**Paso 11 — Cutover Pulso.**
Tras la paridad del Paso 8 + (si aplica) 1 ciclo del correo semanal a dirección estable
desde EspritOS: apagar los containers `pulso-*` (:8500), remover del compose. Ritual de
cierre §4. Si existe ingress `pulso.espritos.app`, redirigir a `espritos.app/pulso/` en
`infra/cloudflared/`.
*Criterio:* containers `pulso-*` apagados, repo pulso-hm archivado, grep confirma cero
referencias a `clean_ventas` en `apps/pulso`.

**Paso 12 — Cutover pricing-hm.**
Tras 1 semana de gracia post-deploy de la Ola A: apagar `pricing-web` (:8300). Ritual de
cierre §4. Redirigir `pricing.cremeriahm.com` → `espritos.app/rentabilidad/` (o remover).
*Criterio:* `pricing-web` apagado, repo pricing-hm archivado.

**Criterio de salida del primer tramo:** `ritmo-django`, `pulso-web`, `pricing-web`
apagados y archivados; suite EspritOS verde; memoria y `DEPLOY-WINDOWS.md` al día;
volúmenes de BD origen conservados (rollback vivo).

---

## 6. Fases futuras — esquema + decisión pendiente

### 6.1 Pricing fold completo
Mayormente cubierto por Ola A + Paso 12. **Decisión ya tomada:** `vendedor_canon` en
`UserProfile`. Pendiente operativo: poblar `VENDEDOR_MAP` con todas las vendedoras y
verificar hidratación multi-usuario en login.

### 6.2 RH-v2 (Yunuen) → `apps/documentos`
**Build real, no reconciliación** — `apps/documentos` NO existe. Usuario activo (Yunuen),
container `rh-v2-web` (:8000). Esquema: nueva app `apps/documentos` (motor de documentos),
RBAC `/documentos/`, RLS por rol, migración de datos desde `rh-v2-db`, tests, paridad,
cutover. **Decisión pendiente:** alcance del motor de documentos a portar (¿1:1 o
recorte?). Requiere su propio blueprint de build.

### 6.3 Impuestos (Montse, FastAPI) → `apps/impuestos`
**Build real + reescritura de framework** (FastAPI → Django) — la única excepción a "cero
migraciones de framework", por consolidación de stack. Usuario activo (Montse), container
`impuestos-web` (:8400). **Bloqueante de seguridad a resolver ANTES de portar:** creds
MySQL hardcodeadas en `services/pipeline.py` (sacar a env/`django-environ`). El pipeline
IVA/IEPS debe respetar la frontera datos-hm (leer canon, no ETL en la app). Requiere su
propio blueprint.

### 6.4 Portal (clientes mayoristas, OTP Twilio) — **VEREDICTO: satélite, NO funde**
El doc maestro lo ponía en F4 ("fundir al final, el más delicado"). **Recomendación
revisada: reclasificar a satélite permanente** (como Termómetro/Prospectos), no fundir.

| Criterio | Portal | EspritOS |
|---|---|---|
| Audiencia | **Cliente** (externo) | **Staff** (interno) |
| Auth | OTP Twilio | django-allauth + MFA |
| BD | Aislada (superficie pública) | `espritos-db` (interno) |
| Dominio | Propio (portal.cremeriahm.com) | espritos.app |

Fundirlo metería superficie de ataque pública dentro del monolito interno **sin premio
operativo** (no comparte usuarios ni auth ni datos transaccionales con el staff).
Reconsiderar SOLO si el costo de mantener su stack se vuelve carga. Acción: dejarlo como
está; documentar la reclasificación en `CONSOLIDACION.md` (mover de "Fase 4 — fundir" a
"Satélites — no funden").

---

## 7. Rollback por fase

- **Ola A/Pulso/Ritmo:** mientras los volúmenes `ritmo-db`/`pulso-db`/`pricing-db` no se
  borren (solo se apagan), reactivar el compose viejo restaura el sistema en minutos.
- **Deploy:** `scripts\deploy_prod.ps1` exige master limpio + suite verde; si algo falla
  en prod, `git revert` + redeploy (la imagen se hornea del working tree).
- **Regla:** los volúmenes de BD origen se borran SOLO en una fase de limpieza posterior
  (fuera de este tramo), nunca antes.

## 8. Qué NO se toca (anti-scope-creep)
1. **datos-hm:** solo lectura. Nada en este tramo modifica el canon.
2. **`segmentos` de Pulso:** pospuesto (requiere extensión del canon).
3. **Portal, Termómetro, Prospectos, sitios estáticos, AnalisisVentas:** satélites.
4. **Las apps standby de EspritOS** no mencionadas: siguen en standby.
5. **`apps/insights` (Ollama):** no se porta; `resumen_canon.py` ya lo reemplaza.

## 9. Skills a usar durante el build
| Skill | Cuándo | Para qué |
|---|---|---|
| `/retoma` | Inicio de cada sesión | Cargar contexto del proyecto |
| `/systematic-debugging` | Si la paridad difiere | Causa raíz antes de cutover |
| `/code-review` | Antes de cada merge a master | Revisión del diff de la ola |
| `/resumen-jornada` | Fin de cada sesión | Memoria al día (mitiga migración a medias) |

## 10. Adiciones a memoria al cerrar el tramo
**A `E:\ClaudeWorks\CLAUDE.md` (hub):** quitar de la tabla de productivos las filas de
Ritmo, Pulso y pricing-hm conforme se archivan; marcar Portal como satélite.
**A `MEMORY.md`:** quitar bullets de los repos archivados (regla de oro).
**A `E:\ClaudeWorks\CONSOLIDACION.md`:** mover Portal de "Fase 4 — fundir" a "Satélites";
marcar Ola A/B/C cerradas.
**A `memory/decisiones.md` (global):** registrar (1) `vendedor_canon` en `UserProfile`;
(2) Portal = satélite permanente; (3) cierre del primer tramo de consolidación.

## 11. Resumen ejecutivo
| Ola | Pasos | Esfuerzo | Riesgo | Bloqueador |
|---|---|---|---|---|
| A — Reconciliación pricing-hm | 1-4 | 1-2 sesiones | Medio (pérdida de WIP) | Ninguno — empezar ya por Paso 1 |
| B — Pulso reconcile + paridad | 5-8 | 1-2 sesiones | Bajo-Medio | Ola A no bloquea (paralelizable tras Paso 1 y 5) |
| C — Apagado legacy | 9-12 | 1 sesión | Bajo | Olas A y B cerradas + paridad |

**Total estimado del primer tramo: 3-5 sesiones.** Las fases futuras (RH-v2, impuestos)
requieren cada una su propio blueprint de build.
