# EspritOS — Empieza aquí

> **Si estás leyendo esto, eres el builder** — una sesión nueva de Claude Code que va a construir EspritOS desde cero. O eres Beto y quieres entender qué hay en esta carpeta.

## Qué es EspritOS

Sistema operativo interno de **Cremería HM** (Tonalá, Jalisco). Reemplaza el CRM-ERP v4 actual (Next.js + Prisma, monolito acoplado, deploy por USB) y Punto Zero (POS legacy). El diferenciador es un **reporteador conversacional con IA** para supervisores de ventas, pero el alcance completo es un **ERP retail de 21 módulos** con POS + hardware + CFDI 4.0 + comisiones + lealtad + rutero PWA offline.

Etimología: *esprit* (francés: espíritu, mente, inteligencia) + *OS* = el cerebro operativo del negocio.

## Mapa de esta carpeta

```
output/espritos/
├── START-HERE.md              ← ESTÁS AQUÍ
├── KICKOFF-PROMPT.md          ← Copia-pega este prompt en una sesión nueva de Claude Code
├── BLUEPRINT.md               ← El documento maestro (2,374 líneas). LÉELO COMPLETO.
├── PARITY-CHECKLIST.md        ← Checklist derivado de las 26 fases del CRM-ERP v4
│
├── context/                   ← Material de referencia del CRM-ERP v4 (fuente de verdad)
│   ├── CRM-ERP-v4-CLAUDE.md   ← CLAUDE.md original del sistema viejo
│   ├── schema.prisma          ← 1,542 líneas, 72 modelos — AUTORIDAD sobre data model
│   ├── docs/                  ← 4 docs operativos (necesidades retail, investigación PZ, Facturama, deploy)
│   ├── codebase/              ← 7 análisis arquitectónicos (architecture, stack, structure, conventions, concerns, testing, integrations)
│   └── phases/                ← 27 fases GSD con 200 archivos (RESEARCH, PLAN, SUMMARY, VERIFICATION)
│
└── bootstrap/                 ← Archivos LISTOS PARA USAR en Paso 1 del Build Order
    ├── README.md              ← Instrucciones de uso
    ├── pyproject.toml         ← Dependencias, ruff, pytest, djlint
    ├── docker-compose.yml     ← Stack completo (db+redis+web+worker+beat)
    ├── Dockerfile             ← Python 3.12 + WeasyPrint + uv
    ├── .env.example           ← Variables de entorno
    ├── .gitignore
    ├── manage.py
    ├── conftest.py            ← Fixtures globales de pytest (users por rol)
    ├── scripts/db-init.sql    ← Extensions pgvector/unaccent/pg_trgm + role chat_ia_reader
    ├── config/                ← Proyecto Django listo
    │   ├── settings/base.py   ← Todas las settings configuradas
    │   ├── settings/dev.py    ← Debug toolbar, email consola
    │   ├── settings/prod.py   ← Sentry, SSL, HSTS
    │   ├── settings/test.py   ← Celery eager, password hash rápido
    │   ├── urls.py            ← /health/, /admin/, /accounts/
    │   ├── wsgi.py, asgi.py, celery.py
    └── apps/
        └── core/              ← App inicial funcional
            ├── models.py       ← TimestampedModel, AuditedModel, SoftDeleteModel, DocumentSequence
            ├── mixins.py       ← RoleFilteredQuerysetMixin (la base de RLS)
            ├── permissions.py  ← Helpers es_vendedor/es_admin/etc.
            ├── templatetags/core_tags.py  ← |mxn, |fecha_mx, |tiene_rol
            ├── admin.py
            └── tests/
                ├── test_document_sequence.py  ← Tests de generación atómica
                └── test_app_isolation.py       ← ⭐ Test CRÍTICO que bloquea cross-imports entre apps
```

## Orden de lectura (para el builder)

1. **BLUEPRINT.md** — léelo de cabo a rabo una vez (2,374 líneas). Es denso pero autosuficiente. Presta especial atención a:
   - Sección 5: Reporteador IA completo (pipeline de 11 pasos + validador SQL con 20+ tests de seguridad críticos)
   - Sección 10: Build Order (13 pasos core + olas A-G con 27 pasos de módulos)
   - Sección 17: Reglas no negociables del proyecto
   - Apéndice D: Inventario de 72 modelos Prisma → 21 apps Django + tabla de las 26 fases

2. **context/schema.prisma** — schema real del CRM-ERP v4. Es la **fuente autoritativa de todos los modelos**. Cuando construyas una app Django, los modelos deben reflejar exactamente lo que está aquí (con adaptaciones idiomáticas a Django).

3. **context/phases/** — las 27 fases GSD del CRM-ERP v4. **ANTES de construir cualquier módulo, lee el RESEARCH.md y SUMMARY.md de la fase correspondiente.** Es el spec de negocio validado. Ejemplos críticos:
   - `02-precios-volumen-descuentos-pos/` antes del Paso 19 (POS)
   - `03-hardware-pos/` antes del Paso 21 (Hardware)
   - `04-cfdi-40-base/` antes del Paso 23 (CFDI)
   - `14-comisiones-de-ventas/` antes del Paso 29 (Comisiones)

4. **context/docs/** — 4 docs operativos:
   - `2026-02-10 Necesidades y Soluciones ERP en Retail.md` (requerimientos originales — MUY valioso)
   - `2026-02-26-investigacion-puntozero.md` (qué mapear de PuntoZero y cómo)
   - `facturama-api-reference.md` (API de Facturama completa — lo usas en Ola C)
   - `deploy-checklist.md` (aprende qué evitar del deploy por USB)

5. **context/codebase/** — 7 análisis arquitectónicos del v4:
   - `ARCHITECTURE.md` — patrones del v4 (muchos los vamos a reusar, otros cambiar)
   - `STACK.md` — stack del v4 (útil para el mapeo Prisma → Django)
   - `STRUCTURE.md` — estructura de carpetas del v4
   - `CONVENTIONS.md` — convenciones de código que funcionaron
   - `CONCERNS.md` — **preocupaciones conocidas** (lo que falló — crítico para no repetirlas)
   - `TESTING.md` — estrategia (o falta) de testing del v4
   - `INTEGRATIONS.md` — integraciones externas

6. **bootstrap/README.md** — cuando estés listo para el Paso 1, este README te dice exactamente cómo copiar los archivos al repositorio nuevo.

7. **PARITY-CHECKLIST.md** — ANTES de apagar el CRM-ERP v4, verifica que EspritOS implementa cada checkbox.

## Reglas sagradas (del Blueprint sección 17)

1. **Validador SQL del chat IA** (`apps/chat_ia/pipeline/sql_validator.py`) es código de seguridad crítica. Sus tests son blockers de deploy.
2. **Apps Django aisladas**. Ningún cross-import. `test_app_isolation.py` (ya incluido en bootstrap) enforce esto automáticamente.
3. **Cero deploys con tests fallando**. CI bloquea el merge.
4. **RLS obligatorio en toda vista que muestre datos**. Vendedor no ve clientes de otro vendedor.
5. **Cero deploys por USB**. Todo via GitHub → Coolify.
6. **Multi-sucursal desde el día 1**. Toda tabla con FK a sucursal.
7. **Circuit breaker IA**. Si gasto diario supera tope, se corta.
8. **Auditoría append-only**. `AuditLog` nunca se borra.

## Cómo arrancar HOY MISMO

```bash
# 1. Crear repo nuevo en GitHub (manual en el navegador o con gh):
gh repo create huheme25/espritos --private --description "EspritOS — Sistema operativo interno Cremería HM"

# 2. Clonar localmente
cd E:/ClaudeWorks/proyectos/
git clone https://github.com/huheme25/espritos.git
cd espritos

# 3. Copiar el bootstrap completo
cp -r /e/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/* .
cp /e/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/.env.example .
cp /e/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/.gitignore .

# 4. Abrir una nueva sesión de Claude Code en el directorio
#    y pegarle el contenido de KICKOFF-PROMPT.md

# 5. La sesión nueva seguirá el Build Order del BLUEPRINT.md
```

## Quick Start del ETL (Paso 6 del Build Order)

> **Pre-requisito:** Pasos 1-5 del Build Order completados. Postgres + Redis + Django corriendo en Docker. App `apps.core` migrada con `User`, `Sucursal`, `groups` creados.

**Estado del entorno servidor PuntoZero (verificado 2026-04-09):**
- ✅ Servidor `192.168.0.200` accesible desde la PC dev `192.168.0.152`
- ✅ MySQL 5.1.48 con bases `datos1` (Cremería, 4.5 GB) y `datos9` (Abarrotera, 93 MB)
- ✅ Usuario `espritos_reader` creado con grants SELECT en ambas bases
- ✅ Ventana 9pm-7am libre para correr ETL

**Pasos para activar el ETL:**

```bash
# 1. Configurar .env (la password está en el Blueprint Apéndice E.14.7)
nano .env
# Agregar:
#   PUNTOZERO_HOST=192.168.0.200
#   PUNTOZERO_USER=espritos_reader
#   PUNTOZERO_PASSWORD=<de Blueprint E.14.7>
#   PUNTOZERO_CHARSET=latin1
#   ETL_SOURCE=puntozero_mysql_direct

# 2. Migrar las tablas del ETL
docker compose run --rm web python manage.py migrate apps.etl

# 3. Importar los mappings históricos de RevisionServidor
docker compose run --rm web python manage.py etl_import_mappings

# 4. Pre-flight check (validar conexión + tablas + mappings)
docker compose run --rm web python manage.py etl_preflight
# Resultado esperado: ✅ READY

# 5. Carga histórica completa (tarda ~30-60 min para datos1, ~5 min para datos9)
docker compose run --rm web python manage.py etl_initial_load --confirm
# Carga primero datos1 (Cremería, sucursal_id=1), luego datos9 (Abarrotera, sucursal_id=2)

# 6. Verificar carga
docker compose run --rm web python manage.py etl_status
docker compose run --rm web python manage.py etl_verify
# Diferencia < 0.1% por tabla = OK

# 7. Activar el incremental nocturno (Celery Beat)
# Ya está programado en config/celery.py para correr a las 21:00 todos los días
# Verificar que celery-beat container está corriendo:
docker compose ps beat
docker compose logs beat --tail 50

# 8. Listo. Mañana a las 21:00 corre automáticamente el incremental.
```

**Si algo falla en el `etl_preflight`:**
- Error de conexión → verificar que `192.168.0.200` es alcanzable, password correcta
- Error de tabla faltante → verificar `SHOW TABLES;` en `datos1` y `datos9` desde mysql CLI
- Error de migraciones → correr `python manage.py migrate apps.etl` primero

**Documentación completa:** Apéndice E del BLUEPRINT.md (44 tablas mapeadas, transformers, watermarks, plan de rollback, programación Celery, comando para Bernardo, riesgo del root compartido).

## Contacto / dudas

- El blueprint debe responder el 95% de las preguntas. Si no lo hace, **pregúntale a Beto antes de improvisar**.
- Las 27 fases del CRM-ERP v4 son tu biblioteca de decisiones validadas. Úsala.
- Si encuentras una contradicción entre el blueprint y el schema.prisma real, **gana el schema.prisma** — es el ground truth de la operación actual.
