# Kickoff Prompt v2.0 — Retomar Paso 6 (ETL Punto Zero)

> **Para Beto:** copia todo lo que está debajo de la línea y pégalo en la sesión del builder que quedó pausada ayer en Paso 6. Reemplaza los valores según tu sesión si es necesario.

---

Retomamos el **Paso 6 del Build Order de EspritOS** (ETL desde Punto Zero EVO). Estuviste pausado desde ayer. Mientras tanto, el arquitecto (the-architect) hizo varias correcciones críticas al scaffolding del ETL basadas en la **verificación física contra el servidor real** de Cremería HM. NO empieces a codear antes de leer lo que sigue y aplicar los cambios del scaffolding al repo.

## Estado confirmado antes de retomar

- **Pasos 1-5 completados**: stack corriendo en `localhost:8100`, 70 tests verdes, 10 críticos con `@pytest.mark.critical`. No los toques.
- **Plan C1 validado físicamente**: conexión LAN directa al servidor PZ funciona.
- **Usuario MySQL creado**: `espritos_reader@'192.168.%'` con grants SELECT en `datos1` (Cremería, 4.5 GB, 17.4M filas) y `datos9` (Abarrotera, 93 MB, 277K filas). Ambas son sucursales activas de la misma empresa madre.
- **pymysql 1.4.6 conecta a MySQL 5.1.48 perfecto** (test de `verify_pz_connection.py`).

## Correcciones críticas que el arquitecto hizo al bootstrap del ETL

Las hizo basándose en los scripts de `RevisionServidor/scripts/` (que ya migraron `datos9` exitosamente) — son ground truth del schema real de PZ EVO 5.1.48.

**Léelas ANTES de tocar código** (en este orden):

1. `E:\ClaudeWorks\repos-referencia\the-architect\output\espritos\BLUEPRINT.md` **sección E.6** — la lista completa de correcciones de nombres de columnas y tablas que NO existen. Ejemplos:
   - `productos.Descripcion` NO existe — la columna real es **`Descrip`**
   - `categorias` NO existe como tabla — usa `lineas`
   - `clientes.Direccion` NO existe — son 5 campos: `Domicilio, Colonia, Ciudad, Estado, CP`
   - `clientes.Telefono` → `Telefono1`, `clientes.DiasCredito` → `DCredito`
   - `proveedores.Email` → `EMail` con **E mayúscula** (inconsistencia de PZ, en clientes es minúscula)
   - `tickets` no tiene `Subtotal`, `MetodoPago`, `Cambio`, `Estatus`, `EsCredito`, `Notas`, `Serie` — hay que calcular o inferir
   - `facturas.Id` es **`ID`** con mayúsculas (sí, diferente de `tickets.Id`)
   - `compras.Compra` es el folio (no `Folio`), `comprasdetalle` joinea al padre por `Compra`
   - `cxc/cxp/gastos/cortes` no existen — CxC real es `saldoscli`
2. `E:\ClaudeWorks\repos-referencia\the-architect\output\espritos\BLUEPRINT.md` **Apéndice E.14** — credenciales reales del servidor, riesgo del root compartido, estimaciones de duración del ETL.
3. `context/revision-servidor/scripts/05_migrate_catalogs.py` líneas 137, 180-183, 230-235, 309-316 — los SELECT reales que validó RevisionServidor contra `datos9`.
4. `context/revision-servidor/scripts/06_migrate_sales.py` líneas 98-101, 184-186, 232-234, 319-320 — columnas reales de tickets, ticketsdetalle, facturas, facturasdetalle.
5. `context/revision-servidor/scripts/07_migrate_purchases_finance.py` líneas 79, 122-123 — columnas reales de compras y comprasdetalle.

## Aplica el scaffolding corregido al repo

Los archivos del arquitecto viven en `E:\ClaudeWorks\repos-referencia\the-architect\output\espritos\bootstrap\apps\etl\`. Cópialos al repo de EspritOS sobrescribiendo lo que tengas:

```bash
# Desde el raíz del repo espritos/
cp -r E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/apps/etl/. apps/etl/
```

Lo que vas a obtener corregido:
- `extractors/productos.py` — columnas reales (Descrip, Linea, Precio1-5, etc.)
- `extractors/clientes.py` — dirección en 5 campos, Telefono1, DCredito
- `extractors/proveedores.py` — EMail con E mayúscula
- `extractors/vendedores.py` — solo 5 columnas reales (Id, Clave, Nombre, Comision, ComisionPrecio1)
- `extractors/ventas_pos.py` — infiere método de pago de Efectivo+TC, combina Fecha+Hora
- `extractors/facturas.py` — ID mayúsculas, sin UUID/xml directo
- `extractors/compras.py` — Compra=folio, Proveedor=clave
- `extractors/inventario.py` — **schema NO verificado** (ver abajo)
- `loaders/postgres_upsert.py` — `bulk_upsert_copy()` implementado con psycopg2 COPY + TEMP staging + ON CONFLICT. 10-50x más rápido que `update_or_create` para batches grandes. Lo usa `inventario.py` para las 17M filas de `historicoalmacen`.
- `management/commands/etl_preflight.py` — tabla de tablas requeridas actualizada (sin `categorias`, `cxc`, `cxp`, `cortes`, `gastos`)
- `sources/punto_zero_mysql.py` — config para MySQL 5.1 legacy con `READ UNCOMMITTED`, `SET NAMES latin1`, `init_command` específico

También hay cambios en settings y env que ya debes tener aplicados — si no, compara tu `.env` contra `bootstrap/.env.example`.

## Configuración de `.env` (credenciales reales)

```env
ETL_SOURCE=puntozero_mysql_direct
PUNTOZERO_HOST=192.168.0.200
PUNTOZERO_PORT=3306
PUNTOZERO_USER=espritos_reader
PUNTOZERO_PASSWORD=EspritReader2025!
PUNTOZERO_DB_CREMERIA=datos1
PUNTOZERO_DB_ABARROTERA=datos9
PUNTOZERO_CHARSET=latin1
PUNTOZERO_CONNECT_TIMEOUT=10
PUNTOZERO_READ_TIMEOUT=180
```

⚠️ **CRÍTICO**: `PUNTOZERO_CHARSET=latin1`. Si pones `utf8` obtienes mojibake (textos corruptos). PZ EVO guarda en latin1 y el transformer `fix_latin1()` lo convierte después.

## ⚠️ Un gap crítico que tienes que resolver antes del `etl_initial_load`

**El schema de `historicoalmacen` NO está verificado.** RevisionServidor nunca la leyó directamente, solo migró `cambiosdeprecios`, `traspasos` y `ajustes`. Los nombres de columna en `inventario.py` son educated guesses.

Antes de activar ese extractor, ejecuta desde tu máquina (no desde el contenedor):

```bash
mysql -h 192.168.0.200 -u espritos_reader -p'EspritReader2025!' datos1
> SHOW COLUMNS FROM historicoalmacen;
> SELECT * FROM historicoalmacen LIMIT 3;
> EXIT;
```

Pega el output en el chat y ajusta `apps/etl/extractors/inventario.py` con los nombres reales de:
- Columna de producto (guess: `Producto`)
- Columna de almacén (guess: `Almacen`)
- Columna de cantidad (guess: `Cantidad`)
- Columna de costo unitario (guess: `Costo`)
- Columna de saldo/existencia (guess: `Saldo`)
- Columna de tipo de movimiento (guess: `Tipo`)
- Columna de documento (guess: `Documento`)
- Columna de número de documento (guess: `NumeroDocumento`)

## Orden de ejecución del Paso 6

```bash
# 1. Aplicar migraciones del ETL
docker compose run --rm web python manage.py migrate apps.etl

# 2. Importar mappings históricos de RevisionServidor
docker compose run --rm web python manage.py etl_import_mappings

# 3. Preflight check — ANTES de activar extractors que escriben datos
docker compose run --rm web python manage.py etl_preflight
# Esperado: "✅ READY"

# 4. Verificar schema de historicoalmacen (ver sección anterior) y ajustar inventario.py

# 5. Dry run con una sola tabla primero para validar el patrón
docker compose run --rm web python manage.py etl_dry_run --table productos

# 6. Carga inicial completa (30-60 min para datos1, <5 min para datos9)
docker compose run --rm web python manage.py etl_initial_load --confirm

# 7. Verificación cruzada PZ vs Postgres
docker compose run --rm web python manage.py etl_verify
# Diferencia < 0.1% por tabla = OK
```

## Reglas no negociables para este paso

1. **NO activar `destination_model` en los extractors** hasta que `apps.catalogo.Producto`, `apps.clientes.Cliente`, `apps.proveedores.Supplier`, etc. existan. Esos apps se crean en la **Ola A** del Build Order (Pasos 14-17). En Paso 6 solo debes tener los extractors escritos y los tests unitarios de transformers verdes.
2. **NO toques los tests críticos de los Pasos 1-5** (los 10 con `@pytest.mark.critical`). Si uno se rompe, investiga root cause antes de commit.
3. **NO hardcodees credenciales** en el código — van en `.env`. Si por accidente commiteas el `.env`, rota la password inmediatamente.
4. **NO corras `etl_initial_load` sin haber pasado `etl_preflight`**. El preflight valida que las 19 tablas críticas existan antes de gastar 30-60 min en un load que pueda fallar a mitad.
5. **Agrega un test específico para `fix_latin1()`** con una fila real del productos de `datos1` que tenga acentos. Corre el test dentro del container después del preflight. Si el test pasa, el encoding va a funcionar para todo el ETL.
6. **Cada extractor nuevo que actives debe tener su test unitario de `transform()`** con un fixture de fila PZ. No hay excusa para activar un extractor sin test.
7. **Commit atomico por extractor activado**. Mensaje tipo `paso6: activa extractor productos con transform tests`.

## Cuando termines Paso 6 (antes de cerrar sesión)

Actualiza `PARITY-CHECKLIST.md` marcando los items de "Infraestructura del ETL" que hayas completado, y los items de extractors que hayas activado en la sección "Extractors" dentro de la Fase 01.

Avísale a Beto con un resumen de:
- Tablas cargadas con counts (PZ vs EspritOS)
- Diff del `etl_verify`
- Cualquier anomalía en la carga (errores de encoding, filas rechazadas, timeouts)
- Tiempo total de carga

Si el schema de `historicoalmacen` resultó muy diferente a los guesses, documenta los nombres reales en un comentario arriba del extractor corregido para la próxima iteración.

---

Arranca leyendo el Blueprint sección E.6 completo. Luego aplica el scaffolding corregido. Luego preflight. Luego historicoalmacen schema discovery. Luego load. En ese orden, sin brincar pasos.
