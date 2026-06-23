# Ritmo — Blueprint de construcción

> Generado por The Architect el 12 de mayo de 2026
> Arquetipo: Internal Tool / Dashboard
> Lengua: español (mexicano)
> Cliente: Beto (Humberto Heme), Cremería HM
> Stack: Streamlit + Python + PostgreSQL (dedicado Docker) + MySQL (lectura PZ)

---

## 0. Cómo usar este blueprint

Este documento es **autocontenido**. Una instancia de Claude Code sin contexto previo debe poder construir Ritmo de principio a fin siguiendo el **Build Order** (sección 9) sin pedir clarificaciones.

**Si eres el Claude Code que va a construir Ritmo:**
1. Lee este documento de principio a fin antes de tocar código.
2. Trabaja la sección 9 (Build Order) paso a paso. **No avances de fase si la anterior no está verde.**
3. Cuando una decisión no esté en el blueprint, consulta primero los apéndices (sección 17-19), luego el CLAUDE.md generado (sección 15), y solo después pregunta a Beto.
4. Cada fase termina con verificación demostrable (ejecutar query, comparar Excel, abrir URL). Sin verificación no se cierra.
5. Todo el código, comentarios y docs en **español**.

---

## 1. Resumen ejecutivo

**Ritmo** es una aplicación web interna (LAN Cremería HM, `192.168.0.152:8200`) que sustituye los scripts Python ad-hoc actuales de velocidad de venta y pedidos sugeridos por una herramienta autoservicio para el equipo de compras.

### Vision

Hoy cualquier pregunta sobre velocidad de venta o pedidos pasa por Beto, que adapta un script Python y genera un Excel. Cada cambio de scope = nuevo script. El equipo no se autoabastece.

Ritmo desbloquea al equipo: Jamie, Andrea y Valeria consultan velocidades, capturan inventarios y generan pedidos sugeridos **sin tocar código**. Beto deja de ser cuello de botella y se enfoca en decisiones, no en correr scripts.

### Goals

- **G1 — Cero scripts Python ad-hoc nuevos** para preguntas de velocidad/pedido a 30 días post-deploy.
- **G2 — Self-service para Jamie/Andrea/Valeria.** Cualquiera del equipo puede consultar velocidad de cualquier conjunto de SKUs sin permiso de Beto.
- **G3 — Memoria del negocio.** Inventarios capturados y pedidos ejecutados quedan registrados con timestamp, no se pierden al sobrescribir un Excel.
- **G4 — Agnóstica al ERP.** La app es dueña de catálogo y captura física. Lee ventas de PZ hoy, podría leer de EspritOS o cualquier ERP mañana.
- **G5 — Calibración medible.** Al registrar lo que se pidió realmente vs lo que la app sugirió, se puede ajustar parámetros (lead time, colchón) con base en evidencia.

### Success Metrics (30/90 días)

| Métrica | Meta 30d | Meta 90d | Cómo se mide |
|---|---|---|---|
| Scripts ad-hoc nuevos en `cremeria/pedidos/` o `cremeria/reportes/marcas/` | 0 | 0 | `git log` del directorio |
| Inventarios capturados | 8+ | 30+ | `SELECT COUNT(DISTINCT contexto, DATE(timestamp)) FROM inventarios` |
| Pedidos generados desde la app | 4+ | 15+ | `SELECT COUNT(*) FROM pedidos_borrador` |
| Pedidos marcados como ejecutados | 2+ | 10+ | `SELECT COUNT(*) FROM pedidos_ejecutados` |
| Usuarios distintos activos | 2+ | 4+ | `SELECT COUNT(DISTINCT usuario) FROM inventarios WHERE timestamp > NOW() - 30d` |
| Variancia mediana entre pedido_sugerido y pedido_ejecutado | medir baseline | ≤ 15% | variancia_kg_calculada |

---

## 2. Para quién es

### Usuarios primarios

| Persona | Rol | Cómo usa Ritmo |
|---|---|---|
| **Beto** | Dueño operativo, analista de catálogo | Importa SKUs nuevos, define factores pieza-kg, genera pedidos SA/NAYAR/ANDALUCIA quincenal, marca pedidos como ejecutados |
| **Jamie** | Encargado Abarrotera + ruta tienditas | Consulta velocidad de listas arbitrarias de SKUs antes de cargar la ruta del día |
| **Andrea Ortega** (vendedora 08) | Canal AOR (mayoreo) | Consulta velocidad de SKUs clave para anticipar necesidades de clientes |
| **Valeria Navarro** (vendedora 09) | Canal AOR | Igual que Andrea |

### Usuarios secundarios

- Capturista de inventarios físicos semanales (hoy: solo Beto).
- Dirección (Humberto, Laura) para auditoría — no operación diaria.

### Lo que NO es Ritmo

- **No reemplaza PuntoZero.** PZ es el POS, fuente de toda la venta. Ritmo lee de PZ, no escribe.
- **No reemplaza EspritOS / CRM-ERP.** Esos son sistemas corporativos integrales. Ritmo es una herramienta de compras táctica.
- **No es para clientes.** Solo equipo interno HM.
- **No genera reportes ejecutivos.** Para eso está `reporte_v2_canonico.py` en AnalisisVentas.

---

## 3. Contexto mínimo del negocio (5 minutos)

> Esta sección permite a un Claude Code sin contexto previo entender por qué Ritmo existe y cómo opera Cremería HM. **No omitir.**

### 3.1 Qué es Cremería HM

Empresa familiar de distribución de **lácteos, embutidos y abarrotes** en Tonalá, Jalisco. Constituida 50/50 entre los padres de Beto (Humberto y Laura). Dos puntos:

- **Cremería HM** (matriz, mayoreo + mostrador) — ~98% del catálogo se registra primero aquí.
- **Abarrotera HM** (2do punto, abierto 2025) — venta directa al público + surtido amplio de abarrotes.

POS: **PuntoZero (PZ)**, MySQL local en `192.168.0.200`. Dos bases:
- `datos1` — Cremería (matriz, fuente autoritativa de catálogo, precios, costos).
- `datos9` — Abarrotera (catálogo de venta y precios mostrador propios).

~2,800 SKUs activos. 94 líneas de producto. ~10 marcas principales en embutidos.

### 3.2 Canales de venta — críticos para velocidad

| Canal | Tabla PZ | Qué es | % venta Q1 2026 |
|---|---|---|---|
| **Ruta** | `remisiones` | Mayoreo: reparto a domicilio + venta en piso a mayoristas | 87.2% |
| **Mostrador** | `tickets` | Menudeo al público | 10.4% |
| **Factura** | `facturas` (con detalle propio) | Ventas facturadas | 2.4% |

**Regla:** cualquier velocidad debe sumar los **3 canales** o queda mocha. PZ ya hace limpieza: cuando factura, elimina de tickets/remisiones, así que **no hay doble conteo**.

### 3.3 Calendario operativo (crítico)

HM opera **L-S, 7am-4pm**. Descansa **solo 4 días al año:**
- 1 de enero
- Viernes Santo
- Sábado Santo
- 25 de diciembre

**Los festivos oficiales de México son días normales de operación HM** (constitución, Juárez, trabajo, independencia, muertos, revolución, Guadalupe).

Velocidad debe ser **kg / día hábil**, no kg / día calendario. Confundirlos infla la velocidad ~17% (porque 1/7 días no hay venta).

### 3.4 Negocio dual

2026: embutidos 33%, lácteos 32%, huevo 10%, otros 25%. Tres dinámicas de proveedor distintas:

- **Embutidos** — proveedores grandes con catálogo cerrado (SAN ANTONIO, ANDALUCIA, NAYAR, EL MEXICANO, CAPISTRANO). Pedidos semanales/quincenales con horizontes definidos.
- **Lácteos** — proveedores variados, surtido continuo.
- **Abarrotes** — proveedores intercambiables, se compra al de mejor precio cada semana. **Mismo SKU puede venir de proveedores distintos en meses distintos.**

**Implicación:** Ritmo no puede asumir SKU = un proveedor. Debe trabajar con listas arbitrarias de SKUs ("estos 30 códigos, dime sus velocidades") y solo a veces con "marca completa".

### 3.5 Unidades: piezas vs kilogramos

PZ guarda la cantidad vendida en la unidad con que el cajero captura: a veces kg (queso fresco granel), a veces **piezas/paquetes** (jamón empaquetado, salchichas, chistorras).

Para decisiones de pedido, todo va a **kg**. Eso requiere un **factor pieza-kg por SKU** (paquete salchicha SA = 3 kg, queso amarillo SM = 1.82 kg/pieza). Hoy hardcodeado en cada script; en Ritmo vive en `productos.factor_kg`.

---

## 4. Tech Stack

| Capa | Tecnología | Por qué |
|---|---|---|
| UI | **Streamlit 1.31+** | Validado en el proyecto (`compartido/explorador/app.py`). Multipágina nativa cubre 5 pantallas. `st.data_editor` da tablas editables sin JS. Beto no es developer profesional, Streamlit minimiza paint de UI |
| Lenguaje | **Python 3.11+** | Mismo de los scripts actuales. Reutiliza patrones validados |
| Cliente MySQL | **PyMySQL 1.1+** | Mismo de scripts actuales. Lectura de PZ datos1+datos9 |
| Manipulación datos | **pandas 2.1+** | Estándar en scripts existentes. DataFrames para tablas Streamlit |
| Export Excel | **xlsxwriter 3.2+** | Formato vivo (formulas, celdas editables amarillas) replicando `generar_pedido_sa.py` |
| Lectura Excel | **openpyxl 3.1+** | Para imports manuales de factores o catálogos legacy |
| Persistencia | **PostgreSQL 16 dedicado** (Docker container `ritmo-postgres`, puerto 5433, volumen named `ritmo_pgdata`) | Costo de infra ≈ 0 porque Docker ya corre en 192.168.0.152 por EspritOS. **Cluster Postgres totalmente aislado del de EspritOS** (otro container, otro puerto, otro volumen, otro usuario). Concurrencia real multi-writer, JSONB para payloads, TIMESTAMPTZ, futuro-proof si Ritmo crece a 10+ usuarios |
| Driver Postgres | **psycopg[binary] 3.1+** | Sucesor moderno de psycopg2. Pool de conexiones built-in, soporte async (no usado en v1 pero disponible) |
| Cache | **`@st.cache_data(ttl=600)`** | 10 min para catálogo y consultas pesadas; sin cache para velocidad (datos frescos siempre) |
| Tests | **pytest 8+** | Estándar Python. Tests críticos: motor velocidad reproduce Excel NAYAR celda por celda |
| Deploy UI | **NSSM** (Non-Sucking Service Manager) | Servicio Windows autostart en 192.168.0.152 para Streamlit. La UI es proceso Python directo, no Docker |
| Deploy BD | **Docker Compose** (`infra/docker-compose.yml`) | Container Postgres dedicado a Ritmo con `restart: unless-stopped`. Reusa Docker Engine ya instalado por EspritOS |
| Logs | **Python logging** + rotating file handler | `app/logs/ritmo.log` con rotación diaria, retención 14 días |

### Por qué NO otras opciones (registro de decisiones descartadas)

| Alternativa | Por qué se descartó |
|---|---|
| Dash + Plotly | Más boilerplate. Sparklines/multi-callback que no se necesitan en v1. Ya hay un Dash en el repo (dashboard) con scope distinto |
| Django (como módulo de EspritOS) | Beto descartó explícitamente: las existencias de EspritOS pueden no ser correctas, Ritmo debe ser agnóstica del ERP. Decisión documentada el 12/05/2026 |
| Postgres dentro del cluster EspritOS | Acoplamiento operativo a EspritOS aunque no de datos. Si EspritOS se cae o se migra, arrastra a Ritmo. Mejor: Postgres dedicado en su propio container, separado |
| SQLite + WAL mode | Concurrencia escrituras serializada. Backup menos robusto (copiar archivo bajo carga riesgoso). El argumento principal a favor (cero infra) no aplica porque Docker ya corre en la PC por EspritOS — agregar 1 container más cuesta lo mismo que escribir el `.gitignore` |
| FastAPI + frontend React | Sobrediseñado para 3-4 usuarios internos. Quintuplica el trabajo |
| MySQL local | Cero ganancia vs Postgres y rompe con el stack que Beto ya opera (EspritOS Postgres) |

---

## 5. Estructura de directorios

Ritmo **vive dentro del repo existente `E:\ClaudeWorks\proyectos\AnalisisVentas\`**, no es un repo nuevo. Reutiliza la estructura `compartido/` para módulos reutilizables y agrega `app/` como entry Streamlit.

```
AnalisisVentas/
│
├── compartido/                            ← Módulos reutilizables (logic, sin UI)
│   │
│   ├── velocidad/                         ← NUEVO — motor de velocidad
│   │   ├── __init__.py
│   │   ├── core.py                        ← velocidad_skus(claves, ventanas, cierre, canales)
│   │   ├── calendario.py                  ← FESTIVOS_HM + dias_habiles(d1, d2)
│   │   ├── proyeccion.py                  ← proyeccion mes en curso (conservador/realista/optimista)
│   │   ├── yoy.py                         ← comparativo mismo periodo año anterior
│   │   ├── tendencia.py                   ← Acelerando/Subiendo/Estable/Bajando/Cayendo
│   │   ├── fuentes/                       ← Conectores intercambiables a fuentes de venta
│   │   │   ├── __init__.py
│   │   │   ├── base.py                    ← clase abstracta FuenteVentas
│   │   │   ├── pz_mysql.py                ← Conector PZ datos1 + datos9
│   │   │   └── csv_local.py               ← Conector CSV (fallback offline)
│   │   └── cli.py                         ← Entry CLI: python -m compartido.velocidad.cli
│   │
│   ├── pedidos/                           ← NUEVO — recomendador y exportador
│   │   ├── __init__.py
│   │   ├── recomendar.py                  ← vel × horizonte − inv + colchón = pedido
│   │   └── exportar_excel.py              ← xlsxwriter con fórmulas vivas (formato SA)
│   │
│   ├── catalogo/                          ← NUEVO — modelo de productos propio
│   │   ├── __init__.py
│   │   ├── importador.py                  ← Import inicial desde PZ datos1+datos9
│   │   ├── resolver.py                    ← resolver(codigo) → ProductoRitmo | None
│   │   ├── productos.py                   ← CRUD productos + referencias externas
│   │   └── factores.py                    ← lectura/escritura factor_kg
│   │
│   └── persistencia/                      ← NUEVO — Postgres + migraciones
│       ├── __init__.py
│       ├── db.py                          ← Conexión psycopg pool + helpers
│       ├── schema.sql                     ← DDL completo (sección 6 de este blueprint)
│       ├── migraciones/                   ← *.sql versionados (001_, 002_, ...)
│       │   └── 001_inicial.sql            ← Crea las 6 tablas
│       └── seed/                          ← Datos iniciales
│           ├── factores_conocidos.csv    ← SA, NAYAR, ANDALUCIA, EMex, Capistrano
│           └── marcas_alias.csv          ← Aliases comunes (SA → SAN ANTONIO, etc.)
│
├── app/                                   ← NUEVO — entry Streamlit "Ritmo"
│   ├── streamlit_app.py                   ← Main: config + sidebar global + navegación
│   ├── pages/                             ← Streamlit multipágina (orden por prefijo numérico)
│   │   ├── 1_Velocidad_libre.py
│   │   ├── 2_Velocidad_por_marca.py
│   │   ├── 3_Inventario.py
│   │   ├── 4_Pedido_sugerido.py
│   │   └── 5_Criticos.py
│   ├── components/                        ← Helpers UI compartidos
│   │   ├── __init__.py
│   │   ├── header.py                      ← Header con selector de usuario
│   │   ├── tablas.py                      ← Helpers st.data_editor (formato, validación)
│   │   ├── exportar.py                    ← Botones de export Excel/CSV consistentes
│   │   └── filtros.py                     ← Selectores reutilizables (fechas, canales, ventanas)
│   ├── logs/                              ← Logs de la app (GITIGNORED)
│   │   └── ritmo.log
│   ├── config.toml                        ← Config Streamlit (puerto, tema)
│   └── RUNBOOK.md                         ← Troubleshooting para Jamie/sustituto
│
├── infra/                                 ← NUEVO — Infraestructura local de Ritmo (Docker)
│   ├── docker-compose.yml                 ← Servicio `ritmo-postgres` (postgres:16-alpine, puerto 5433, volumen `ritmo_pgdata`)
│   ├── postgres.env.example               ← Plantilla credenciales (commiteable)
│   ├── postgres.env                       ← Credenciales reales (GITIGNORED)
│   └── README.md                          ← Cómo levantar, detener, conectarse, ver logs
│
├── datos/
│   └── backups/                           ← NUEVO — Backups locales (GITIGNORED)
│       └── ritmo/
│           ├── 2026-05-13/
│           │   └── ritmo_db.dump          ← pg_dump custom format diario
│           ├── 2026-05-14/
│           └── ...                        ← Retención 30 días, rotación automática
│
├── docs/
│   ├── plans/
│   │   └── 2026-05-12-app-velocidad-design.md  ← Doc original (referencia histórica)
│   └── recon/                             ← NUEVO — Evidencia de F0.5 (schema PZ confirmado)
│       └── 2026-05-XX-pz-schema-recon.md
│
├── cremeria/                              ← Scripts existentes (CONGELADOS, solo consulta)
│   ├── pedidos/
│   │   ├── velocidad_live_sa.py           ← # DEPRECATED — usar Ritmo
│   │   ├── generar_pedido_sa.py           ← # DEPRECATED
│   │   └── generar_pedido_andalucia.py    ← # DEPRECATED
│   └── reportes/marcas/
│       ├── velocidad_nayar.py             ← # DEPRECATED
│       └── velocidad_san_antonio.py       ← # DEPRECATED
│
├── scripts/
│   ├── ritmo_start.bat                    ← NUEVO — entry dev/prod
│   ├── ritmo_install_service.bat          ← NUEVO — registra NSSM service
│   ├── ritmo_backup.bat                   ← NUEVO — backup diario pg_dump
│   └── ritmo_dev.bat                      ← NUEVO — streamlit run en modo dev
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                        ← Fixtures: Postgres test (rollback) + mocks PZ
│   ├── test_calendario.py                 ← dias_habiles(), FESTIVOS_HM
│   ├── test_velocidad_core.py             ← Reproduce velocidad_nayar_20260508.xlsx ±0
│   ├── test_velocidad_proyeccion.py       ← 3 escenarios cierre de mes
│   ├── test_velocidad_yoy.py              ← Comparativo año anterior
│   ├── test_catalogo_resolver.py          ← Resolver código PZ + barras + interna
│   ├── test_pedidos_recomendar.py         ← Match contra pedido_sa_20260512.xlsx
│   └── test_persistencia.py               ← Migraciones, concurrencia WAL
│
├── requirements.txt                       ← Dependencias (Streamlit, pandas, etc.)
├── requirements-dev.txt                   ← pytest, ruff, etc.
├── .gitignore                             ← Ignora app/logs/, datos/backups/, infra/postgres.env, .env
├── CLAUDE.md                              ← El de AnalisisVentas (no se modifica salvo notas Ritmo)
└── README_RITMO.md                        ← NUEVO — quickstart 5 minutos
```

### Decisiones clave de estructura

- **No es repo nuevo:** Ritmo vive en `AnalisisVentas/` para reutilizar `compartido/`, queries SQL ya validadas y el patrón `datos/outputs/`.
- **`compartido/` separado de `app/`:** la lógica (velocidad, pedidos, catálogo) es reutilizable desde CLI, Jupyter o futura migración a Django. La UI es la única parte específica de Streamlit.
- **Scripts legacy en `cremeria/`:** se congelan con header `# DEPRECATED — usar app Ritmo`. **NO se borran** hasta que Ritmo lleve 30 días estable. Es la red de seguridad de Beto.
- **Infra propia en `infra/`:** Ritmo trae su propio Postgres dedicado en Docker. NO se acopla operativamente al cluster Postgres de EspritOS. Si EspritOS deja de existir, Ritmo no se entera.
- **Backups locales en `datos/backups/`:** decisión explícita de Beto (12/05/2026). No OneDrive. `pg_dump` custom format diario, rotación automática 30 días, validación con `pg_restore --list` después de cada dump.

---

## 6. Modelo de datos

### 6.1 Entidades

#### `productos` — tabla maestra propia de Ritmo

| Campo | Tipo | Notas |
|---|---|---|
| `clave_interna` | INTEGER PRIMARY KEY AUTOINCREMENT | Identificador propio de Ritmo |
| `descripcion` | TEXT NOT NULL | Nombre legible |
| `marca` | TEXT | "SAN ANTONIO", "NAYAR", etc. — texto libre, no normalizado en v1 |
| `linea` | TEXT | "Salchichas", "Jamones", "Queso fresco", etc. |
| `unidad_compra` | TEXT NOT NULL | 'kg' \| 'paq' \| 'pza' \| 'lt' \| 'caja' |
| `factor_kg` | REAL NOT NULL | unidad_compra → kg (paq SA = 3.0) |
| `empaque_caja` | INTEGER | Cuántas unidades trae 1 caja del proveedor (para redondeo) |
| `proveedor_habitual` | TEXT | Texto libre — para abarrotes puede cambiar |
| `activo` | BOOLEAN NOT NULL DEFAULT 1 | Soft delete |
| `notas` | TEXT | Observaciones libres de Beto |
| `creado_at` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | |
| `actualizado_at` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | Trigger AFTER UPDATE |

#### `productos_ref_externas` — múltiples claves externas por producto

| Campo | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | |
| `clave_interna` | INTEGER NOT NULL | FK → productos.clave_interna |
| `sistema` | TEXT NOT NULL | 'PZ_DATOS1' \| 'PZ_DATOS9' \| 'ESPRITOS' \| 'CODIGO_BARRAS' \| otro |
| `clave_externa` | TEXT NOT NULL | El identificador en ese sistema |
| `creado_at` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | |

**Índices:** UNIQUE (`sistema`, `clave_externa`), INDEX (`clave_interna`).

#### `inventarios` — snapshots inmutables append-only

| Campo | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | |
| `timestamp` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | |
| `contexto` | TEXT | "SAN ANTONIO", "ANDALUCIA", "ad-hoc", "ruta dia3", etc. — texto libre |
| `usuario` | TEXT NOT NULL | Beto/Jamie/Andrea/Valeria/otro |
| `clave_interna` | INTEGER NOT NULL | FK → productos.clave_interna |
| `cantidad` | REAL NOT NULL | En la unidad que se capturó |
| `unidad` | TEXT NOT NULL | Snapshot de la unidad usada (puede diferir de unidad_compra) |
| `kg` | REAL NOT NULL | Pre-calculado al guardar (cantidad × factor_kg) |
| `caducidad` | DATE | Opcional |
| `lote` | TEXT | Opcional |
| `notas` | TEXT | Opcional |

**Índice:** (`contexto`, `timestamp` DESC) para "último inventario de SA".

**Política:** **append-only**. Editar un inventario significa capturar uno nuevo con timestamp más reciente. La vista "actual de SA" toma el más reciente por (`contexto`, `clave_interna`).

#### `pedidos_borrador` — sugerencias generadas por la app

| Campo | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | |
| `timestamp` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | |
| `contexto` | TEXT | Mismo contexto del inventario base |
| `usuario` | TEXT NOT NULL | Quién lo generó |
| `inventario_base_max_ts` | TIMESTAMP | Timestamp del inventario que se usó como base |
| `ventana_velocidad` | TEXT NOT NULL | '7d' \| '15d' \| '30d' \| '90d' |
| `modo_horizonte` | TEXT NOT NULL | 'lead_freq_seg' \| 'fecha_objetivo' |
| `lead_dias` | INTEGER | Si modo = lead_freq_seg |
| `frecuencia_dias` | INTEGER | Si modo = lead_freq_seg |
| `seguridad_dias` | INTEGER | Si modo = lead_freq_seg |
| `fecha_objetivo` | DATE | Si modo = fecha_objetivo |
| `horizonte_dh` | INTEGER NOT NULL | Días hábiles calculados |
| `payload` | TEXT NOT NULL | JSON: array de {clave_interna, vel_dh, inv_kg, colchon_kg, pedido_kg, pedido_unid, pedido_cajas} |
| `archivo_excel` | TEXT | Path relativo al Excel exportado, si se exportó |
| `estado` | TEXT NOT NULL DEFAULT 'borrador' | 'borrador' \| 'ejecutado' \| 'descartado' |

#### `pedidos_ejecutados` — qué se pidió realmente y qué llegó (cierra el loop)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | |
| `pedido_borrador_id` | INTEGER NOT NULL | FK → pedidos_borrador.id |
| `fecha_pedido_real` | DATE NOT NULL | Cuándo se envió al proveedor |
| `fecha_entrega_real` | DATE | Cuándo llegó (se llena después) |
| `cantidades_reales` | TEXT NOT NULL | JSON: {clave_interna: cantidad_real_kg} — lo que Beto realmente pidió, puede diferir del sugerido |
| `usuario` | TEXT NOT NULL | Quién marcó como ejecutado |
| `notas` | TEXT | "Pedí menos de SA porque tenía cliente cancelado" |
| `variancia_kg` | REAL | (pedido_real - pedido_sugerido) por producto, agregado o por SKU — JSON o número según se decida |
| `creado_at` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | |

#### `cache_velocidad` — opcional, para acelerar pantalla Críticos

| Campo | Tipo | Notas |
|---|---|---|
| `clave_interna` | INTEGER NOT NULL | |
| `ventana` | TEXT NOT NULL | '7d' \| '30d' \| etc. |
| `cierre` | DATE NOT NULL | |
| `vel_kg_dh` | REAL NOT NULL | |
| `actualizado_at` | TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP | |

**PK:** (`clave_interna`, `ventana`, `cierre`).

**Política:** se llena con job nocturno (Task Scheduler) o on-demand al abrir pantalla 5. TTL 24h.

### 6.2 Relaciones

```
productos (1) ────< (N) productos_ref_externas       [un producto, N claves externas]
productos (1) ────< (N) inventarios                  [un producto, N capturas]
productos (1) ────< (N) cache_velocidad              [un producto, N ventanas/cierres]

pedidos_borrador (1) ──── (0..1) pedidos_ejecutados  [un borrador → 0 o 1 ejecución]
```

### 6.3 Schema SQL completo (PostgreSQL 16, en `compartido/persistencia/migraciones/001_inicial.sql`)

```sql
-- Migración 001: schema inicial Ritmo (PostgreSQL 16)
-- Aplicar contra ritmo_db, usuario ritmo_user, container ritmo-postgres:5433

-- ─── Trigger genérico para actualizado_at ────────────────────────────────
CREATE OR REPLACE FUNCTION fn_set_actualizado_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.actualizado_at := NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ─── productos ───────────────────────────────────────────────────────────
CREATE TABLE productos (
  clave_interna       SERIAL PRIMARY KEY,
  descripcion         TEXT          NOT NULL,
  marca               TEXT,
  linea               TEXT,
  unidad_compra       TEXT          NOT NULL CHECK (unidad_compra IN ('kg','paq','pza','lt','caja','ml','gr')),
  factor_kg           NUMERIC(10,4) NOT NULL CHECK (factor_kg > 0),
  empaque_caja        INTEGER       CHECK (empaque_caja IS NULL OR empaque_caja > 0),
  proveedor_habitual  TEXT,
  activo              BOOLEAN       NOT NULL DEFAULT TRUE,
  notas               TEXT,
  creado_at           TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  actualizado_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_productos_marca   ON productos(marca);
CREATE INDEX idx_productos_linea   ON productos(linea);
CREATE INDEX idx_productos_activo  ON productos(activo) WHERE activo = TRUE;

CREATE TRIGGER trg_productos_actualizado
BEFORE UPDATE ON productos
FOR EACH ROW EXECUTE FUNCTION fn_set_actualizado_at();

-- ─── productos_ref_externas ──────────────────────────────────────────────
CREATE TABLE productos_ref_externas (
  id              SERIAL PRIMARY KEY,
  clave_interna   INTEGER     NOT NULL REFERENCES productos(clave_interna) ON DELETE CASCADE,
  sistema         TEXT        NOT NULL CHECK (sistema IN ('PZ_DATOS1','PZ_DATOS9','ESPRITOS','CODIGO_BARRAS','OTRO')),
  clave_externa   TEXT        NOT NULL,
  creado_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (sistema, clave_externa)
);

CREATE INDEX idx_ref_externas_clave_interna ON productos_ref_externas(clave_interna);

-- ─── inventarios ─────────────────────────────────────────────────────────
CREATE TABLE inventarios (
  id              BIGSERIAL PRIMARY KEY,
  timestamp       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  contexto        TEXT,
  usuario         TEXT          NOT NULL,
  clave_interna   INTEGER       NOT NULL REFERENCES productos(clave_interna),
  cantidad        NUMERIC(14,4) NOT NULL CHECK (cantidad >= 0),
  unidad          TEXT          NOT NULL,
  kg              NUMERIC(14,4) NOT NULL CHECK (kg >= 0),
  caducidad       DATE,
  lote            TEXT,
  notas           TEXT
);

CREATE INDEX idx_inventarios_contexto_ts ON inventarios(contexto, timestamp DESC);
CREATE INDEX idx_inventarios_clave       ON inventarios(clave_interna);
CREATE INDEX idx_inventarios_ts          ON inventarios(timestamp DESC);

-- ─── pedidos_borrador ────────────────────────────────────────────────────
CREATE TABLE pedidos_borrador (
  id                      BIGSERIAL PRIMARY KEY,
  timestamp               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  contexto                TEXT,
  usuario                 TEXT        NOT NULL,
  inventario_base_max_ts  TIMESTAMPTZ,
  ventana_velocidad       TEXT        NOT NULL CHECK (ventana_velocidad IN ('7d','15d','30d','90d')),
  modo_horizonte          TEXT        NOT NULL CHECK (modo_horizonte IN ('lead_freq_seg','fecha_objetivo')),
  lead_dias               INTEGER,
  frecuencia_dias         INTEGER,
  seguridad_dias          INTEGER,
  fecha_objetivo          DATE,
  horizonte_dh            INTEGER     NOT NULL CHECK (horizonte_dh > 0),
  payload                 JSONB       NOT NULL,
  archivo_excel           TEXT,
  estado                  TEXT        NOT NULL DEFAULT 'borrador' CHECK (estado IN ('borrador','ejecutado','descartado'))
);

CREATE INDEX idx_borrador_ts        ON pedidos_borrador(timestamp DESC);
CREATE INDEX idx_borrador_contexto  ON pedidos_borrador(contexto);
CREATE INDEX idx_borrador_estado    ON pedidos_borrador(estado);
CREATE INDEX idx_borrador_payload   ON pedidos_borrador USING GIN (payload);

-- ─── pedidos_ejecutados ──────────────────────────────────────────────────
CREATE TABLE pedidos_ejecutados (
  id                    BIGSERIAL PRIMARY KEY,
  pedido_borrador_id    BIGINT      NOT NULL UNIQUE REFERENCES pedidos_borrador(id),
  fecha_pedido_real     DATE        NOT NULL,
  fecha_entrega_real    DATE,
  cantidades_reales     JSONB       NOT NULL,
  usuario               TEXT        NOT NULL,
  notas                 TEXT,
  variancia_kg          NUMERIC(14,4),
  creado_at             TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ejecutados_fecha ON pedidos_ejecutados(fecha_pedido_real DESC);

-- ─── cache_velocidad ─────────────────────────────────────────────────────
CREATE TABLE cache_velocidad (
  clave_interna   INTEGER       NOT NULL REFERENCES productos(clave_interna),
  ventana         TEXT          NOT NULL,
  cierre          DATE          NOT NULL,
  vel_kg_dh       NUMERIC(14,4) NOT NULL,
  actualizado_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  PRIMARY KEY (clave_interna, ventana, cierre)
);

-- ─── Verificación post-migración ─────────────────────────────────────────
-- Después de aplicar, verificar:
--   SELECT tablename FROM pg_tables WHERE schemaname='public' ORDER BY tablename;
--   → debe devolver 6 tablas: productos, productos_ref_externas, inventarios,
--     pedidos_borrador, pedidos_ejecutados, cache_velocidad
```

---

## 7. Conectores a fuentes externas

### 7.1 Abstracción `FuenteVentas` (`compartido/velocidad/fuentes/base.py`)

```python
from abc import ABC, abstractmethod
from datetime import date
import pandas as pd

class FuenteVentas(ABC):
    """Contrato: cualquier fuente que pueda dar ventas históricas por SKU."""

    nombre: str  # 'PZ_DATOS1', 'PZ_DATOS9', 'CSV_LOCAL', etc.

    @abstractmethod
    def ventas_por_sku(
        self,
        claves_externas: list[str],
        desde: date,
        hasta: date,
        canales: set[str] | None = None,  # None = todos
    ) -> pd.DataFrame:
        """
        Devuelve DataFrame con columnas:
          clave_externa | fecha | canal | cantidad | unidad | kg
        """
        ...

    @abstractmethod
    def ping(self) -> bool:
        """Verifica que la fuente esté viva."""
        ...
```

### 7.2 Implementación PZ MySQL (`fuentes/pz_mysql.py`)

Consulta UNION ALL sobre las 3 tablas de detalle (`remisionesdet`, `tickets`, `facturasdet`), suma cantidades y agrupa por (clave_pz, fecha, canal).

**Queries canónicas (extraídas de scripts existentes):**

Ver Apéndice A para los SELECTs completos validados contra `reporte_v2_canonico.py`.

### 7.3 Implementación CSV (`fuentes/csv_local.py`)

Lee de `E:\Datos-PuntoZero\` (histórico descargado). Útil como **fallback offline** si MySQL PZ no responde, y para análisis de periodos pre-2025 cuando no había acceso live.

### 7.4 Política de degradación

Si `pz_mysql.ping()` falla:
1. Banner amarillo en la UI: "PZ no responde. Mostrando datos cacheados al DD/MM HH:MM."
2. Velocidad se calcula con `csv_local.py` apuntando al último snapshot disponible.
3. Inventario y pedidos **no** se bloquean (son operaciones locales contra `ritmo-postgres`).

Si `ritmo-postgres` (BD propia) falla:
1. Banner rojo en la UI: "BD local no responde. Captura de inventario y pedidos deshabilitada."
2. Velocidad puede seguir funcionando (solo lee de PZ).
3. RUNBOOK indica: `docker compose -f infra/docker-compose.yml restart`.

---

## 8. Pantallas y flujos UX

### 8.1 Layout general

```
┌─────────────────────────────────────────────────────────────────────────┐
│  RITMO                                              Usuario: [Beto ▼]   │
│  Velocidad de venta y pedidos sugeridos                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  [Sidebar Streamlit]            [Contenido principal]                    │
│                                                                          │
│  • Velocidad libre              ...                                      │
│  • Velocidad por marca                                                   │
│  • Inventario                                                            │
│  • Pedido sugerido                                                       │
│  • Críticos                                                              │
│                                                                          │
│  ─────────────────                                                       │
│  Fuente: PZ datos1+datos9                                                │
│  Último sync: 12/05 14:30                                                │
│  [⚠ banner si fuente caída]                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Pantalla 1 — Velocidad libre (CASO PRIMARIO)

**Estado inicial:**

```
┌─ Velocidad por código ───────────────────────────────────────────────┐
│  Ingresa códigos (clave interna, PZ, barras, descripción exacta).    │
│  Máx 50. Uno por línea.                                              │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ SA                                                            │    │
│  │ 7501030487012                                                 │    │
│  │ QASM                                                          │    │
│  │ ...                                                           │    │
│  └──────────────────────────────────────────────────────────────┘    │
│   12 / 50 códigos                                                    │
│                                                                       │
│  Cierre: [11/05/2026 ▼]                                              │
│  Canales: [✓Ruta] [✓Mostrador] [✓Factura]                            │
│  Ventanas: [✓7d] [✓15d] [✓30d] [✓90d]                                │
│  [✓ Incluir comparativo YoY]                                         │
│                                                                       │
│  [Calcular velocidad y proyección]                                   │
└──────────────────────────────────────────────────────────────────────┘
```

**Estado con resultados:**

```
┌─ Resultados ─────────────────────────────────────────────────────────┐
│  Clave | Descripción | v7d | v15d | v30d | v90d | v30d_2025 | Tend  │
│  SA    | Salch paq SA | 461 | 458  | 455  | 432  | 410       | ↑    │
│  ...                                                                  │
│                                                                       │
│  Proyección a fin de mes (escenarios):                               │
│    Conservador: 8,420 kg                                             │
│    Realista:    9,180 kg                                             │
│    Optimista:  10,050 kg                                             │
│                                                                       │
│  [Exportar Excel]  [Usar estos códigos para pedido →]                │
│                                                                       │
│  ⚠ 2 códigos no resueltos: "XYZ", "12345"                            │
│  [Crear producto en Ritmo para XYZ]                                  │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.3 Pantalla 2 — Velocidad por marca/línea

```
┌─ Velocidad por marca o línea ────────────────────────────────────────┐
│  Selector:                                                           │
│    Marca:  [SAN ANTONIO ▼]   (NAYAR, EL MEXICANO, ANDALUCIA, etc.)   │
│    o Línea: [───────────  ▼]  (Salchichas, Jamones, Quesos, etc.)    │
│                                                                       │
│  Resultado: 35 claves cargadas → [Ir a velocidad libre con esta lista]│
└──────────────────────────────────────────────────────────────────────┘
```

Al confirmar, redirige a pantalla 1 con los códigos pre-cargados y dispara cálculo automático.

### 8.4 Pantalla 3 — Inventario

```
┌─ Captura de inventario ──────────────────────────────────────────────┐
│  Contexto: [SAN ANTONIO ▼ o escribir nuevo]                          │
│  Fecha: 12/05/2026 11:30     Usuario: Beto                           │
│  Último inv de SAN ANTONIO: 09/05/2026 (Beto) — 35 SKUs              │
│                                                                       │
│  [Cargar SKUs del último inv]  [Cargar todos de una marca]           │
│                                                                       │
│  Clave | Descripción       | Unidad | Cant   | → kg   | Caducidad   │
│  SA    | Salchicha paq SA  | paq    | [824]  | 2,472  | [27/05/2026]│
│  SSM   | Salchicha paq SM  | paq    | [480]  | 1,440  | [22/05/2026]│
│  QASM  | Queso Amarillo SM | kg     | [ 20]  |    20  | [18/05/2026]│
│  ...                                                                  │
│  [+ Agregar SKU]                                                     │
│                                                                       │
│  Total: 4,532 kg en 35 SKUs                                          │
│  [Guardar inventario]  [Ver pedido sugerido →]                       │
└──────────────────────────────────────────────────────────────────────┘
```

**Validaciones:**
- Cantidad debe ser ≥ 0.
- Si SKU tiene `factor_kg = 1.0` y `unidad_compra ≠ 'kg'`, mostrar warning amarillo: "Factor pieza-kg no configurado, revisar antes de guardar."
- Caducidad opcional pero si se llena, debe ser ≥ hoy.

### 8.5 Pantalla 4 — Pedido sugerido

```
┌─ Pedido sugerido ────────────────────────────────────────────────────┐
│  Contexto: [SAN ANTONIO ▼]                                           │
│  Inventario base: 12/05/2026 11:30 (Beto) — 35 SKUs                  │
│  Velocidad base: [30d ▼]                                             │
│                                                                       │
│  Modo de horizonte:                                                  │
│    ( ) Lead + Frecuencia + Seguridad                                 │
│        Lead: [3] dh   Frec: [14] dh   Seg: [3] dh                    │
│    (•) Fecha objetivo                                                │
│        Cubrir hasta: [31/05/2026]                                    │
│                                                                       │
│  Horizonte calculado: 17 dh                                          │
│                                                                       │
│  [Calcular pedido]                                                   │
│                                                                       │
│  ─── Resultado editable ─────────────────────────────────────────    │
│  Clave | Vel kg/dh | Inv kg | Need | Colchón | Pedido kg | Cajas    │
│  SA    | 455       | 2,472  | 7,735| [0]     | 5,263     | 219       │
│  SSM   | 187       | 1,440  | 3,179| [200]   | 1,939     | ...       │
│  ...                                                                  │
│  (Columna Colchón es editable, recalcula al cambiar)                 │
│                                                                       │
│  [Guardar borrador]  [Exportar Excel formulado]  [Marcar como ejecut.]│
└──────────────────────────────────────────────────────────────────────┘
```

**Al "Marcar como ejecutado":**
- Modal: "¿Qué cantidades pediste realmente al proveedor?"
- Pre-rellena con las sugeridas, Beto edita.
- Captura fecha de pedido real (default: hoy).
- Guarda en `pedidos_ejecutados`.
- Después, cuando llegue el pedido, otra acción "Registrar entrega" llena `fecha_entrega_real`.

### 8.6 Pantalla 5 — Críticos

```
┌─ Productos críticos (se acaban pronto) ──────────────────────────────┐
│  Mostrar SKUs con < [5] dh de cobertura (vel 30d sobre último inv)   │
│  Filtrar por contexto: [Todos ▼]                                     │
│                                                                       │
│  Clave | Desc          | Marca | Inv kg | v30d | Cobertura dh | Stat │
│  SA    | Salch paq SA  | SA    | 120    | 455  | 0.3 dh       | ⚠⚠⚠ │
│  RCH   | Chistorra RCH | NAYAR | 25     | 8    | 3.1 dh       | ⚠   │
│  ...                                                                  │
│                                                                       │
│  [Exportar lista]  [Pre-cargar a pedido sugerido]                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.7 Header global (todas las pantallas)

`app/components/header.py`:

```python
def render_header():
    col1, col2 = st.columns([4, 1])
    with col1:
        st.markdown("### 🍃 RITMO")
        st.caption("Velocidad de venta · Inventarios · Pedidos sugeridos · Cremería HM")
    with col2:
        usuario = st.selectbox("Usuario", ["Beto", "Jamie", "Andrea", "Valeria", "Otro"],
                                key="ritmo_usuario")
        st.session_state["usuario"] = usuario
```

---

## 9. Build Order

**Reglas para el constructor:**

1. **Fases secuenciales.** No empezar la siguiente sin verde en la anterior.
2. **Cada fase tiene verificación demostrable.** Sin verificación, no se cierra.
3. **Commits atómicos por fase.** Mensaje: `feat(ritmo): F1 motor velocidad reutilizable`.
4. **Tests viven con el código.** No "los hago al final."
5. **Toda la sintaxis en español** (variables, comentarios, mensajes de log, UI).

---

### Fase 0 — Preparación del repo (30 min)

**Objetivo:** dejar AnalisisVentas listo para recibir Ritmo sin romper nada existente.

**Tareas:**
1. Crear branch `feat/ritmo`.
2. Crear directorios vacíos:
   - `compartido/velocidad/`, `compartido/velocidad/fuentes/`
   - `compartido/pedidos/`
   - `compartido/catalogo/`
   - `compartido/persistencia/`, `compartido/persistencia/migraciones/`, `compartido/persistencia/seed/`
   - `app/`, `app/pages/`, `app/components/`, `app/logs/`
   - `infra/`
   - `datos/backups/ritmo/`
   - `scripts/`
   - `tests/`
3. Agregar al `.gitignore`:
   ```
   app/logs/
   datos/backups/
   infra/postgres.env
   .env
   .venv/
   __pycache__/
   *.pyc
   .pytest_cache/
   ```
4. Crear `requirements.txt` con dependencias (sección 11).
5. Crear `tests/conftest.py` con fixture `db_postgres_test` que conecta a BD `ritmo_db_test` y aplica rollback al final de cada test.

**Verificación:** `pytest tests/` corre sin tests pero sin errores. `git status` no muestra archivos sensibles para commit.

**Commit:** `chore(ritmo): F0 estructura de carpetas y dependencias`

---

### Fase 0.5 — Reconocimiento de schema PZ (30 min)

**Objetivo:** confirmar nombres exactos de columnas en PZ antes de escribir queries en F1. Resuelve los riesgos abiertos del blueprint con queries directas usando `espritos_reader`.

**Tareas:**
1. Conectar con `espritos_reader` a `192.168.0.200:3306/datos1` (cliente: DBeaver, HeidiSQL, mysql CLI, o `python -c` con PyMySQL).
2. Correr las queries de reconocimiento (**Apéndice F**).
3. Crear `docs/recon/2026-05-XX-pz-schema-recon.md` con:
   - Output completo de los `DESCRIBE`/`INFORMATION_SCHEMA`.
   - **Resolución explícita de las 4 incógnitas:**
     - **a.** `Cancelado` o `Cancelada`? (¿en cuáles tablas? ¿hay variaciones por tabla?)
     - **b.** ¿Cómo se llama el campo de código de barras en `productos`? (`CodigoBarras`, `UPC`, `EAN`, `Barras`, otro)
     - **c.** ¿Cómo se marca un SKU descontinuado? (`Activo`, `Estatus`, `Vigente`, `Baja`, otro)
     - **d.** ¿`Fecha`, `FechaDoc` o `FechaCaptura`? (¿igual en remisiones, tickets y facturas?)
   - Sample de 3 filas por tabla (sin datos sensibles).
4. Si hay sorpresas (ej. no existe campo de código de barras en `productos`), **detener** y resolver con Beto antes de F1. Las queries de F1 dependen de estos nombres.

**Verificación:** archivo `docs/recon/...md` commiteado, con las 4 respuestas explícitas + samples. El archivo es la fuente de verdad para construir queries en F1, F3 y F4.

**Commit:** `docs(ritmo): F0.5 reconocimiento schema PZ con nombres canónicos confirmados`

---

### Fase 1 — Motor de velocidad reutilizable (4-6 h)

**Objetivo:** función pura que dada una lista de claves PZ devuelve velocidades multi-ventana validadas contra el Excel NAYAR.

**Archivos a crear:**
- `compartido/velocidad/calendario.py` — `FESTIVOS_HM` y `dias_habiles(d1, d2)`. Extraer de `cremeria/reportes/marcas/velocidad_nayar.py`.
- `compartido/velocidad/fuentes/base.py` — clase abstracta `FuenteVentas`.
- `compartido/velocidad/fuentes/pz_mysql.py` — conector PZ con queries UNION ALL (ver Apéndice A).
- `compartido/velocidad/tendencia.py` — clasificador 'Acelerando' | 'Subiendo' | 'Estable' | 'Bajando' | 'Cayendo' basado en ratios de ventanas.
- `compartido/velocidad/core.py` — `velocidad_skus(claves, ventanas, cierre, canales, fuente)`.
- `compartido/velocidad/cli.py` — entry CLI: `python -m compartido.velocidad.cli --claves SA,RCH --ventanas 7,30,90`.
- `tests/test_calendario.py` — verifica `dias_habiles(2026-05-01, 2026-05-31) == 26`.
- `tests/test_velocidad_core.py` — reproduce velocidad NAYAR 08/05/2026 celda por celda.

**Verificación:**
```bash
python -m compartido.velocidad.cli --claves "210,313,RCH,178" --ventanas 7,15,30,90 --cierre 2026-05-08
```
Salida = mismas velocidades que el Excel `velocidad_nayar_20260508.xlsx`. Diferencia tolerada: ±0.01 kg/dh por redondeo.

**Commit:** `feat(ritmo): F1 motor velocidad reutilizable con tests vs NAYAR`

---

### Fase 2 — Postgres dedicado + schema + seed (2-3 h)

**Objetivo:** Postgres corriendo en container Docker dedicado a Ritmo, con las 6 tablas creadas y seed inicial poblado.

**Archivos a crear:**
- `infra/docker-compose.yml` — servicio `ritmo-postgres` con:
  ```yaml
  services:
    ritmo-postgres:
      image: postgres:16-alpine
      container_name: ritmo-postgres
      restart: unless-stopped
      ports:
        - "127.0.0.1:5433:5432"     # solo localhost, no expuesto a LAN
      env_file:
        - postgres.env
      volumes:
        - ritmo_pgdata:/var/lib/postgresql/data
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U ritmo_user -d ritmo_db"]
        interval: 10s
        timeout: 5s
        retries: 5
  volumes:
    ritmo_pgdata:
      name: ritmo_pgdata
  ```
- `infra/postgres.env.example` — plantilla:
  ```
  POSTGRES_DB=ritmo_db
  POSTGRES_USER=ritmo_user
  POSTGRES_PASSWORD=cambiar_este_password_aqui
  ```
- `infra/postgres.env` — copia editable con password real (gitignored).
- `infra/README.md` — quickstart (levantar, detener, conectar, logs, backup manual).
- `compartido/persistencia/schema.sql` — DDL Postgres (copiar de sección 6.3).
- `compartido/persistencia/migraciones/001_inicial.sql` — mismo contenido.
- `compartido/persistencia/db.py` — `get_conn()` con psycopg pool, `aplicar_migraciones()`, `transaccion()` context manager.
- `compartido/persistencia/seed/__init__.py` y `poblar_factores()`.
- `compartido/persistencia/seed/factores_conocidos.csv` — factores conocidos hoy (SAN ANTONIO, NAYAR, ANDALUCIA, EL MEXICANO, CAPISTRANO).
- `tests/test_persistencia.py` — verifica que las 6 tablas existen, FKs respetados, trigger `actualizado_at` dispara, JSONB acepta dict.

**Tareas operativas:**
1. Generar password fuerte: `python -c "import secrets; print(secrets.token_urlsafe(32))"`.
2. Crear `infra/postgres.env` desde el example y pegar el password.
3. Levantar container:
   ```bash
   cd infra
   docker compose up -d
   docker compose ps     # debe mostrar "healthy" en < 30 s
   cd ..
   ```
4. Aplicar migraciones:
   ```bash
   python -c "from compartido.persistencia.db import aplicar_migraciones; aplicar_migraciones()"
   ```
5. Crear BD test para tests (una vez):
   ```bash
   docker exec ritmo-postgres psql -U ritmo_user -d postgres -c "CREATE DATABASE ritmo_db_test OWNER ritmo_user;"
   ```
6. Poblar seed:
   ```bash
   python -c "from compartido.persistencia.seed import poblar_factores; poblar_factores()"
   ```

**Verificación:**
- `docker compose -f infra/docker-compose.yml ps` muestra `ritmo-postgres` con estado `healthy`.
- `psql "host=localhost port=5433 dbname=ritmo_db user=ritmo_user" -c "\dt"` lista las 6 tablas.
- `SELECT clave_interna, descripcion, factor_kg FROM productos WHERE marca='SAN ANTONIO' LIMIT 5;` devuelve 5 SKUs SA con factor_kg correcto.
- Reiniciar Docker daemon y verificar que el container vuelve solo (`restart: unless-stopped`).

**Commit:** `feat(ritmo): F2 Postgres dedicado en Docker + schema + seed factores conocidos`

---

### Fase 3 — Catálogo: importador y resolver (3-4 h)

**Objetivo:** poder importar desde PZ y resolver cualquier código a un producto.

**Archivos a crear:**
- `compartido/catalogo/importador.py` — `importar_desde_pz(sistema='PZ_DATOS1' o 'PZ_DATOS9')`. Lee productos.*, crea entradas en `productos` y `productos_ref_externas`. Idempotente: re-importar no duplica.
- `compartido/catalogo/resolver.py` — `resolver(codigo) → Producto | None`. Busca en este orden:
  1. `clave_interna` (numérico)
  2. `productos_ref_externas.clave_externa` (PZ y barras)
  3. `productos.descripcion` exact match (case-insensitive)
- `compartido/catalogo/productos.py` — CRUD básico.
- `compartido/catalogo/factores.py` — `factor_kg(clave_interna)` y `set_factor_kg(...)`.
- `tests/test_catalogo_resolver.py` — verifica los 3 caminos de resolución.

**Verificación:**
```bash
python -c "from compartido.catalogo.importador import importar_desde_pz; importar_desde_pz('PZ_DATOS1')"
```
Importa ~2,800 SKUs. Re-correr no genera duplicados.

```bash
python -c "from compartido.catalogo.resolver import resolver; p = resolver('SA'); print(p.descripcion, p.factor_kg)"
```
Devuelve "Salchicha paq SA" y 3.0.

**Commit:** `feat(ritmo): F3 catálogo agnóstico con importador PZ y resolver multi-formato`

---

### Fase 4 — App Streamlit base + Pantalla 1 Velocidad libre (5-6 h)

**Objetivo:** primer pantallazo en LAN. Caso primario funcionando.

**Archivos a crear:**
- `app/streamlit_app.py` — main con config y sidebar.
- `app/config.toml` — puerto 8200, tema (sección 12).
- `app/components/header.py` — header con selector de usuario.
- `app/components/tablas.py` — helpers st.data_editor.
- `app/components/exportar.py` — botón export Excel reutilizable.
- `app/pages/1_Velocidad_libre.py` — pantalla 1 completa.
- `scripts/ritmo_dev.bat` — `streamlit run app/streamlit_app.py --server.port 8200`.

**Verificación:**
```bash
.\scripts\ritmo_dev.bat
```
- Abre `http://localhost:8200`, ve el header con "🍃 RITMO".
- Pega 30 códigos de la ruta dia3 → calcula < 3 s, muestra tabla con vel 7/15/30/90.
- Botón "Exportar Excel" descarga `.xlsx` con formato.

**Commit:** `feat(ritmo): F4 app Streamlit base + pantalla velocidad libre`

---

### Fase 5 — Pantalla 2 Velocidad por marca/línea (1-2 h)

**Objetivo:** carga rápida de SKUs por marca/línea.

**Archivos a crear:**
- `app/pages/2_Velocidad_por_marca.py`

**Verificación:** Seleccionar marca=NAYAR carga 27 claves correctas y permite "Ir a velocidad libre" pre-cargado.

**Commit:** `feat(ritmo): F5 pantalla velocidad por marca y línea`

---

### Fase 6 — Comparativo YoY (1-2 h)

**Objetivo:** columna `vel_30d_2025` (mismo periodo año anterior) al lado de `vel_30d`.

**Archivos a crear:**
- `compartido/velocidad/yoy.py` — `velocidad_yoy(claves, ventana, cierre)` devuelve velocidad de [cierre - ventana - 1año, cierre - 1año].
- Modificar `core.py` para incluir columnas YoY cuando `incluir_yoy=True`.
- Modificar `app/pages/1_Velocidad_libre.py` para mostrar columnas YoY si el checkbox está activo.

**Verificación:** Para SKUs con historia, columna `vel_30d_2025` se llena. Para SKUs creados este año, muestra "—".

**Commit:** `feat(ritmo): F6 comparativo YoY en velocidad`

---

### Fase 7 — Pantalla 3 Inventario (4-5 h)

**Objetivo:** captura tabular, conversión kg, persistencia inmutable.

**Archivos a crear:**
- `app/pages/3_Inventario.py`
- `compartido/catalogo/factores.py` ya tiene el factor; aquí solo se usa.

**Verificación:**
- Capturar inventario SAN ANTONIO con 35 SKUs.
- Cerrar la app, reabrir.
- Pantalla 3 muestra "Último inv de SAN ANTONIO: hoy (Beto) — 35 SKUs".
- Capturar el mismo contexto con cantidades distintas crea NUEVO registro (no edita el anterior).

**Commit:** `feat(ritmo): F7 captura de inventario inmutable`

---

### Fase 8 — Recomendador de pedidos + export Excel (3-4 h)

**Objetivo:** lógica de pedido que iguala (±1%) `pedido_sa_20260512.xlsx`.

**Archivos a crear:**
- `compartido/pedidos/recomendar.py` — `recomendar_pedido(claves, vel_df, inv_df, lead, freq, seg | fecha_objetivo, colchones)`.
- `compartido/pedidos/exportar_excel.py` — replica formato de `generar_pedido_sa.py` con `xlsxwriter`: columnas, fórmulas vivas, celda amarilla editable de "cajas finales", calculo automático.
- `tests/test_pedidos_recomendar.py` — match contra Excel real de mayo.

**Verificación:**
```python
recomendar_pedido(claves=SA_35, vel_df=vel_30d, inv_df=inv_12mayo, fecha_objetivo='2026-05-31')
```
Output igual a `pedido_sa_20260512.xlsx` ± 1% por SKU.

**Commit:** `feat(ritmo): F8 recomendador de pedidos con Excel formulado`

---

### Fase 9 — Pantalla 4 Pedido sugerido + pedidos_ejecutados (4-5 h)

**Objetivo:** UI para generar pedido, guardar como borrador, marcar como ejecutado.

**Archivos a crear:**
- `app/pages/4_Pedido_sugerido.py`
- Lógica `marcar_como_ejecutado(borrador_id, cantidades_reales, fecha_pedido, fecha_entrega, notas)` en `compartido/pedidos/recomendar.py`.

**Verificación:**
- Generar pedido SA desde la app → match con Excel de Fase 8.
- Marcar como ejecutado con cantidades editadas → registro en `pedidos_ejecutados` con `variancia_kg` calculada.
- Vista "Pedidos recientes" muestra borrador y ejecutado.

**Commit:** `feat(ritmo): F9 pantalla pedido sugerido + registro de ejecución`

---

### Fase 10 — Pantalla 5 Críticos (2-3 h)

**Objetivo:** lista de SKUs con stock para < N dh.

**Archivos a crear:**
- `app/pages/5_Criticos.py`
- Lógica `productos_criticos(umbral_dh, contexto=None)` en `compartido/velocidad/core.py`.

**Verificación:** Después de capturar inventarios SA + NAYAR, pantalla 5 muestra top 20 SKUs ordenados por cobertura ascendente con `< 5 dh`.

**Commit:** `feat(ritmo): F10 pantalla productos críticos`

---

### Fase 11 — Deploy LAN + backups + runbook (3-4 h)

**Objetivo:** Ritmo corre como servicio 24/7 en 192.168.0.152:8200, Jamie puede entrar desde su PC.

**Tareas:**
1. **Servicio Windows con NSSM:**
   - Descargar NSSM si no está.
   - `scripts/ritmo_install_service.bat`: `nssm install RitmoApp "C:\Python311\python.exe" "-m" "streamlit" "run" "app/streamlit_app.py" "--server.port" "8200" "--server.address" "0.0.0.0"`.
   - Configurar working directory, autostart, restart on failure.
2. **Backup diario con pg_dump:**
   - `scripts/ritmo_backup.bat`:
     ```bat
     @echo off
     for /f "tokens=2 delims==" %%a in ('"wmic os get localdatetime /value"') do set DT=%%a
     set FECHA=%DT:~0,4%-%DT:~4,2%-%DT:~6,2%
     set DIR=E:\ClaudeWorks\proyectos\AnalisisVentas\datos\backups\ritmo\%FECHA%
     if not exist "%DIR%" mkdir "%DIR%"
     docker exec ritmo-postgres pg_dump -U ritmo_user -F c -d ritmo_db -f /tmp/ritmo_db.dump
     docker cp ritmo-postgres:/tmp/ritmo_db.dump "%DIR%\ritmo_db.dump"
     docker exec ritmo-postgres rm /tmp/ritmo_db.dump
     REM Validar dump
     docker exec ritmo-postgres pg_restore --list /dev/null < "%DIR%\ritmo_db.dump" > "%DIR%\ritmo_db.toc.txt" 2>&1
     REM Rotación 30 días
     forfiles /p "E:\ClaudeWorks\proyectos\AnalisisVentas\datos\backups\ritmo" /d -30 /c "cmd /c if @isdir==TRUE rmdir /s /q @path" 2>nul
     ```
   - Registrar en Task Scheduler: diario 3:00 am, ejecutar como usuario con permisos a Docker.
3. **Firewall:** abrir puerto 8200 TCP solo en red privada/LAN. Puerto 5433 **NO** se expone a LAN (solo loopback por config `127.0.0.1:5433` en compose).
4. **Verificar IP estática 192.168.0.152.**
5. **RUNBOOK** (`app/RUNBOOK.md`):
   - Cómo reiniciar la UI: `nssm restart RitmoApp`.
   - Cómo reiniciar la BD: `docker compose -f infra/docker-compose.yml restart`.
   - Cómo ver logs UI: `Get-Content app/logs/ritmo.log -Tail 100 -Wait`.
   - Cómo ver logs BD: `docker compose -f infra/docker-compose.yml logs --tail 100 -f`.
   - Cómo verificar MySQL PZ: `python -c "from compartido.velocidad.fuentes.pz_mysql import ping; print(ping())"`.
   - Cómo verificar Postgres Ritmo: `docker exec ritmo-postgres pg_isready -U ritmo_user -d ritmo_db`.
   - Cómo restaurar backup:
     ```bash
     docker cp datos/backups/ritmo/YYYY-MM-DD/ritmo_db.dump ritmo-postgres:/tmp/
     docker exec ritmo-postgres pg_restore -U ritmo_user -d ritmo_db --clean --if-exists /tmp/ritmo_db.dump
     ```
   - Quién más tiene acceso a Claude Code para resucitar la app si Beto está fuera.

**Verificación:**
- Reiniciar la PC 192.168.0.152. Ritmo está vivo al boot.
- Jamie entra desde su PC a `http://192.168.0.152:8200` y consulta velocidad de 30 códigos.
- Verificar al día siguiente que `datos/backups/ritmo/YYYY-MM-DD/` tiene una copia nueva.

**Commit:** `feat(ritmo): F11 deploy LAN + backups locales + runbook`

---

### Fase 12 — Cleanup scripts legacy (1 h)

**Objetivo:** marcar scripts viejos como deprecated, conservarlos como red de seguridad.

**Tareas:**
1. Header en cada script viejo:
   ```python
   # DEPRECATED 2026-05-XX — usar la app Ritmo (http://192.168.0.152:8200).
   # Este script se mantiene como referencia hasta 2026-06-15.
   ```
2. Actualizar `cremeria/CLAUDE.md` (si existe) o `AnalisisVentas/CLAUDE.md` con nota de Ritmo.
3. Crear `README_RITMO.md` en la raíz del repo con quickstart.

**Verificación:** Ningún script viejo es invocado en los siguientes 14 días (verificar con `git log` y `Get-EventLog`).

**Commit:** `chore(ritmo): F12 deprecate scripts legacy + README quickstart`

---

### Resumen de fases

| # | Fase | Tiempo | Bloqueante para |
|---|---|---|---|
| 0 | Preparación repo | 30 min | Todas |
| 0.5 | Reconocimiento schema PZ | 30 min | F1, F3, F4 |
| 1 | Motor velocidad | 4-6 h | F4, F6, F8, F10 |
| 2 | Postgres dedicado + schema | 2-3 h | F3, F7, F9 |
| 3 | Catálogo + importador | 3-4 h | F4, F7 |
| 4 | App + Pantalla 1 | 5-6 h | F5, F11 |
| 5 | Pantalla 2 | 1-2 h | — |
| 6 | YoY | 1-2 h | — |
| 7 | Pantalla 3 Inventario | 4-5 h | F9, F10 |
| 8 | Recomendador pedidos | 3-4 h | F9 |
| 9 | Pantalla 4 Pedido | 4-5 h | F11 |
| 10 | Pantalla 5 Críticos | 2-3 h | — |
| 11 | Deploy LAN + Docker + backups | 3-4 h | F12 |
| 12 | Cleanup legacy | 1 h | — |

**Total: 35-47 horas de trabajo neto.** A tiempo parcial (1-2 h/día): 3-4 semanas calendario.

---

## 10. Environment Setup

### Prerequisitos

- **Windows 10/11 64-bit** (la PC 192.168.0.152).
- **Python 3.11+** instalado en `C:\Python311\` o vía pyenv.
- **Docker Engine** ya instalado (lo trae EspritOS en la misma PC).
- **Acceso de red a 192.168.0.200:3306** (MySQL PZ).
- **Usuario MySQL `espritos_reader`** con permisos SELECT en `datos1` y `datos9`. Credenciales en `.env` (no commitear).
- **NSSM** (descargar de nssm.cc, copiar `nssm.exe` a `C:\nssm\`).
- **Puertos libres:** 8200 (Streamlit), 5433 (Postgres Ritmo). Verificar con `netstat -ano | findstr ":8200 :5433"`.

### Variables de entorno

**Raíz del repo: `.env` (gitignored)** — credenciales de fuentes externas y config Streamlit:

| Variable | Descripción | Ejemplo |
|---|---|---|
| `PZ_MYSQL_HOST` | Host MySQL PZ | `192.168.0.200` |
| `PZ_MYSQL_PORT` | Puerto | `3306` |
| `PZ_MYSQL_USER` | Usuario read-only | `espritos_reader` |
| `PZ_MYSQL_PASSWORD` | Password | (de Beto) |
| `PZ_DB_DATOS1` | Nombre BD Cremería | `datos1` |
| `PZ_DB_DATOS9` | Nombre BD Abarrotera | `datos9` |
| `RITMO_PG_HOST` | Host Postgres Ritmo | `localhost` |
| `RITMO_PG_PORT` | Puerto Postgres Ritmo | `5433` |
| `RITMO_PG_DB` | Nombre BD | `ritmo_db` |
| `RITMO_PG_USER` | Usuario | `ritmo_user` |
| `RITMO_PG_PASSWORD` | Password | (mismo que en `infra/postgres.env`) |
| `RITMO_LOG_LEVEL` | INFO en prod, DEBUG en dev | `INFO` |
| `RITMO_PORT` | Puerto Streamlit | `8200` |

**`infra/postgres.env` (gitignored)** — credenciales del container Postgres (solo lo usa Docker):

| Variable | Ejemplo |
|---|---|
| `POSTGRES_DB` | `ritmo_db` |
| `POSTGRES_USER` | `ritmo_user` |
| `POSTGRES_PASSWORD` | (mismo password que `RITMO_PG_PASSWORD` del `.env`) |

### Setup inicial paso a paso

```powershell
# 1. Ir al repo
cd E:\ClaudeWorks\proyectos\AnalisisVentas

# 2. Crear y activar venv
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 3. Instalar deps
pip install -r requirements.txt
pip install -r requirements-dev.txt

# 4. Crear .env (en raíz, copiar de .env.example y rellenar)
Copy-Item .env.example .env
# editar .env con credenciales reales (PZ + Postgres Ritmo)

# 5. Levantar Postgres dedicado en Docker
cd infra
Copy-Item postgres.env.example postgres.env
# editar postgres.env con password generado: python -c "import secrets; print(secrets.token_urlsafe(32))"
docker compose up -d
docker compose ps   # esperar a "healthy"
cd ..

# 6. Aplicar migraciones (crea las 6 tablas)
python -c "from compartido.persistencia.db import aplicar_migraciones; aplicar_migraciones()"

# 7. Crear BD test (una sola vez, para correr pytest)
docker exec ritmo-postgres psql -U ritmo_user -d postgres -c "CREATE DATABASE ritmo_db_test OWNER ritmo_user;"

# 8. Importar catálogo PZ (primera vez)
python -c "from compartido.catalogo.importador import importar_desde_pz; importar_desde_pz('PZ_DATOS1')"
python -c "from compartido.catalogo.importador import importar_desde_pz; importar_desde_pz('PZ_DATOS9')"

# 9. Poblar factores conocidos
python -c "from compartido.persistencia.seed import poblar_factores; poblar_factores()"

# 10. Correr en modo dev
.\scripts\ritmo_dev.bat
# o
streamlit run app/streamlit_app.py --server.port 8200
```

---

## 11. Dependencias

### `requirements.txt`

```
streamlit>=1.31,<2.0
pandas>=2.1,<3.0
numpy>=1.26,<2.0
PyMySQL>=1.1,<2.0
psycopg[binary,pool]>=3.1,<4.0
xlsxwriter>=3.2,<4.0
openpyxl>=3.1,<4.0
python-dotenv>=1.0,<2.0
```

### `requirements-dev.txt`

```
pytest>=8.0,<9.0
pytest-mock>=3.12,<4.0
ruff>=0.4,<1.0
```

**Pinning:** versiones mayores fijas, menores libres. Re-pin cada 6 meses.

---

## 12. Deployment Strategy

### Hosting

- **PC de trabajo:** Windows 10/11, IP estática **192.168.0.152**.
- **UI Streamlit:** servicio Windows con NSSM, autostart al boot. Puerto **8200** expuesto a LAN.
- **BD Postgres:** container Docker `ritmo-postgres` (postgres:16-alpine, `restart: unless-stopped`). Puerto **5433** solo expuesto en `127.0.0.1` (no en LAN). Volumen `ritmo_pgdata` para persistencia.
- **URL interna:** `http://192.168.0.152:8200`.
- **DNS local opcional:** agregar `ritmo.lan → 192.168.0.152` en el router HM.

### Aislamiento de EspritOS

| Recurso | EspritOS | Ritmo |
|---|---|---|
| Container Postgres | `espritos-prod-postgres` | `ritmo-postgres` |
| Puerto Postgres | 5432 | 5433 |
| Volumen Docker | `espritos_pgdata` | `ritmo_pgdata` |
| Usuario BD | `espritos_user` | `ritmo_user` |
| Red Docker | `espritos_prod` | default bridge |
| Backup | (su propio job) | `datos/backups/ritmo/` |

**Garantía:** Ritmo nunca abre conexión a `espritos-prod-postgres:5432`. Si lo hace, falla el test `test_aislamiento_espritos`.

### `app/config.toml`

```toml
[server]
port = 8200
address = "0.0.0.0"
headless = true
runOnSave = false
maxUploadSize = 50

[browser]
gatherUsageStats = false

[theme]
base = "light"
primaryColor = "#16a34a"        # verde Cremería HM
backgroundColor = "#ffffff"
secondaryBackgroundColor = "#f5f5f4"
textColor = "#1c1917"
font = "sans serif"
```

### CI/CD

**No aplica formalmente.** El "CI" es:
1. `pytest tests/` debe pasar local antes de merge a `main`.
2. Push a `main` no dispara nada automático — Beto hace `git pull` en la PC 192.168.0.152 y `nssm restart RitmoApp`.
3. Para mayor disciplina futura: GitHub Actions con `pytest` en cada PR (opcional, v1.2).

### Ambientes

| Ambiente | UI | BD | Datos |
|---|---|---|---|
| **Dev (Beto laptop)** | `streamlit run` directo | `docker compose up -d` con `ritmo_db_dev` en puerto 5433 (laptop) | BD dev separada, datos de prueba |
| **Test (CI/local)** | n/a | BD `ritmo_db_test` en mismo container, transacciones rollback | Datos sintéticos por test |
| **Prod (192.168.0.152)** | NSSM service `RitmoApp` | Container `ritmo-postgres` con `ritmo_db` | BD productiva |

---

## 13. Testing Strategy

### Unit tests

- **`tests/test_calendario.py`** — `dias_habiles(d1, d2)`, FESTIVOS_HM. Casos:
  - Lunes a lunes (siguiente) = 6.
  - 1 enero a 31 diciembre 2026 = 308 dh (52×6 - 4 festivos HM).
  - Semana santa: viernes a martes incluye 2 festivos.
- **`tests/test_velocidad_core.py`** — reproduce NAYAR 08/05/2026 celda por celda. Tolerancia ±0.01 kg/dh.
- **`tests/test_velocidad_yoy.py`** — comparativo año anterior. Caso: SKU con historia → valor; SKU nuevo → None.
- **`tests/test_pedidos_recomendar.py`** — match contra `pedido_sa_20260512.xlsx`. Tolerancia 1%.

### Integration tests

- **`tests/test_persistencia.py`** — migraciones idempotentes, FKs respetados, trigger `actualizado_at` dispara, JSONB acepta dict y devuelve dict, 5 conexiones concurrentes escribiendo no se bloquean.
- **`tests/test_catalogo_resolver.py`** — los 3 caminos: clave_interna, ref_externa, descripción.
- **`tests/test_aislamiento_espritos.py`** — verifica que la config de Ritmo nunca apunta a `espritos-prod-postgres` ni a `localhost:5432`. Si alguien por error pone esos hosts, el test rompe la build.

### Smoke tests (manuales, post-deploy)

Checklist en `app/RUNBOOK.md`:
- [ ] `http://192.168.0.152:8200` carga.
- [ ] Pantalla 1: pegar 5 códigos SA, obtener velocidad < 3 s.
- [ ] Pantalla 3: capturar 3 SKUs, guardar, reabrir, ver registro.
- [ ] Pantalla 4: generar pedido de 5 SKUs, descargar Excel, abrir en Excel.
- [ ] Pantalla 5: ver lista de críticos.

### Mocks y fixtures

`tests/conftest.py` provee:
- **`db_postgres_test`** — conexión a `ritmo_db_test` (BD dedicada de tests, misma container). Cada test corre dentro de una transacción que se hace **rollback** al final → aislamiento total entre tests sin truncar tablas. Crear la BD una vez con `docker exec ritmo-postgres psql -U ritmo_user -d postgres -c "CREATE DATABASE ritmo_db_test OWNER ritmo_user;"`.
- **`fuente_mock`** — implementa `FuenteVentas` con datos sintéticos. Permite tests del motor de velocidad sin depender de MySQL PZ vivo.
- **`pz_real_conn`** — conexión real a PZ con `espritos_reader`. Marcado con `@pytest.mark.requires_pz` y skippeado si no hay red a 192.168.0.200.

---

## 14. Skills to Use During Build

| Skill | Cuándo usar | Por qué |
|---|---|---|
| `/test-driven-development` | F1, F8 | Validar motor velocidad y recomendador contra Excels reales antes de escribir UI |
| `/systematic-debugging` | Cuando F1 o F8 no reproduzcan Excel | Comparar fila por fila, identificar dónde diverge (canal mocho, calendario, factor) |
| `/verification-before-completion` | Cierre de cada fase | Antes de mover a la siguiente, demostrar que la actual entrega valor |
| `/webapp-testing` | F11 (smoke test post-deploy) | Verificar pantallazos desde "PC de Jamie" simulando navegador externo |
| `/davila7-xlsx` | F8 (replicar Excel formulado) | Si surgen problemas con fórmulas vivas en xlsxwriter |
| `/spec-create`, `/spec-execute` | Opcional, fases grandes (F4, F7, F9) | Disciplina spec-driven dentro de la fase |

**Skills NO usar:**
- `/frontend-design`, `/ui-ux-pro-max` — Streamlit no tiene customización profunda de UI, sería trabajo inútil.
- `/shadcn-ui` — irrelevante, no es React.
- `/seo-audit` — app interna LAN, sin SEO.

---

## 15. CLAUDE.md para el proyecto target

> Pegar este contenido en `E:\ClaudeWorks\proyectos\AnalisisVentas\CLAUDE_RITMO.md` (o como una nueva sección al final del `CLAUDE.md` existente del repo). Da contexto suficiente a otro Claude Code para trabajar sobre Ritmo.

```markdown
# Ritmo (módulo dentro de AnalisisVentas)

App Streamlit interna LAN para velocidad de venta, captura de inventarios y pedidos sugeridos. Reemplaza scripts Python ad-hoc de `cremeria/pedidos/` y `cremeria/reportes/marcas/`.

## Comandos

- `docker compose -f infra/docker-compose.yml up -d` — Levantar Postgres dedicado
- `docker compose -f infra/docker-compose.yml ps` — Verificar healthy
- `docker compose -f infra/docker-compose.yml logs --tail 50 -f` — Ver logs Postgres
- `.\scripts\ritmo_dev.bat` — Levantar Streamlit en modo desarrollo (puerto 8200)
- `pytest tests/` — Correr toda la suite
- `pytest tests/test_velocidad_core.py -v` — Solo motor de velocidad
- `python -m compartido.velocidad.cli --claves SA,RCH --ventanas 7,30` — CLI standalone
- `python -c "from compartido.persistencia.db import aplicar_migraciones; aplicar_migraciones()"` — Aplicar migraciones Postgres
- `scripts\ritmo_backup.bat` — Backup manual pg_dump

## Tech Stack

Streamlit 1.31 + Python 3.11 + pandas + PyMySQL (PZ) + psycopg 3 + PostgreSQL 16 (Docker dedicado) + xlsxwriter + NSSM (Windows service para UI)

## Arquitectura

### Estructura

- `compartido/velocidad/` — Motor reutilizable (logic). Independiente de Streamlit.
- `compartido/pedidos/` — Recomendador y exportador Excel.
- `compartido/catalogo/` — Productos maestros + importador PZ + resolver códigos.
- `compartido/persistencia/` — Postgres connection pool, migraciones, schema.
- `app/` — UI Streamlit (pages, components). **No mete lógica de negocio aquí.**
- `infra/docker-compose.yml` — Container `ritmo-postgres:5433` con volumen `ritmo_pgdata`.
- `infra/postgres.env` — Credenciales container (gitignored).
- `datos/backups/ritmo/` — Backups `pg_dump` diarios con retención 30 días (gitignored).
- `cremeria/` — Scripts legacy congelados con header `# DEPRECATED`.

### Flujo de datos

```
Streamlit page ──► compartido.*.{función} ──► PyMySQL (PZ datos1+datos9, read-only)
                                          ──► psycopg (Postgres Ritmo localhost:5433)
                                          ──► pandas DataFrame
                                          ──► st.dataframe / xlsxwriter export
```

### Patrones clave

- **BD propia totalmente aislada de EspritOS.** Container `ritmo-postgres` en puerto 5433. Nunca tocar `espritos-prod-postgres:5432`.
- **Catálogo agnóstico:** `productos` es maestra propia. Refs externas en `productos_ref_externas`. Un producto puede existir sin equivalente en ningún ERP.
- **Velocidad consolidada datos1+datos9:** v1 mezcla matriz + abarrotera. Si surge necesidad de segmentar, v1.1.
- **Inventarios append-only:** editar = capturar nuevo con timestamp más reciente. La vista "actual" toma el más reciente por (contexto, clave_interna).
- **Pedidos en 2 fases:** `borrador` (lo que sugirió la app) → `ejecutado` (lo que Beto realmente pidió, con variancia).
- **Fuente externa con fallback:** si MySQL PZ cae, lee de CSV local con banner de aviso. Si Postgres Ritmo cae, captura y pedidos deshabilitados (velocidad sigue funcionando).
- **Conexión Postgres vía pool psycopg:** una sola configuración en `compartido/persistencia/db.py`, todo el código pide `get_conn()`.

## Reglas de organización de código

1. **Lógica en `compartido/`, UI en `app/`.** Si una función puede correrse desde CLI sin Streamlit, vive en `compartido/`.
2. **Cada `app/pages/N_Nombre.py` máx 250 líneas.** Si crece, extraer a `app/components/`.
3. **Imports absolutos** (`from compartido.velocidad.core import ...`), nunca relativos.
4. **Sin lógica de negocio en `st.session_state`.** Solo inputs temporales y selección de usuario.
5. **Variables, comentarios, mensajes de log, UI: todo en español.**

## Sistema de diseño

- **Color primario:** #16a34a (verde Cremería HM)
- **Fondo:** #ffffff
- **Fondo secundario:** #f5f5f4
- **Texto:** #1c1917
- **Tipografía:** sans-serif (default Streamlit)
- **Estética:** funcional, denso, sin animaciones. Es una herramienta de trabajo, no un dashboard ejecutivo.

## Variables de entorno

| Variable | Descripción |
|---|---|
| `PZ_MYSQL_HOST` | IP MySQL PZ (192.168.0.200) |
| `PZ_MYSQL_USER` | Usuario read-only (espritos_reader) |
| `PZ_MYSQL_PASSWORD` | Password (de Beto) |
| `PZ_DB_DATOS1` | datos1 |
| `PZ_DB_DATOS9` | datos9 |
| `RITMO_PG_HOST` | Host Postgres Ritmo (localhost) |
| `RITMO_PG_PORT` | Puerto (5433) |
| `RITMO_PG_DB` | Nombre BD (ritmo_db) |
| `RITMO_PG_USER` | Usuario (ritmo_user) |
| `RITMO_PG_PASSWORD` | Password (mismo que infra/postgres.env) |
| `RITMO_PORT` | Puerto Streamlit (8200) |

## Reglas no negociables

1. **Velocidad SIEMPRE en kg/día hábil.** Días hábiles HM = L-S menos 4 días/año (1 ene, V santo, S santo, 25 dic). Los festivos oficiales MX son días normales HM.
2. **Velocidad SIEMPRE suma los 3 canales (ruta + mostrador + factura).** PZ ya hace limpieza, no hay doble conteo.
3. **Inventarios son append-only.** Nunca UPDATE ni DELETE. Editar = nuevo registro.
4. **Cierre por defecto = ayer, no hoy.** Hoy todavía está vendiendo, no compara contra ventana completa.
5. **Productos pueden existir sin clave_externa.** La app es dueña del catálogo, no PZ.
6. **MySQL PZ solo se LEE.** Nunca INSERT/UPDATE/DELETE en datos1 ni datos9.
7. **Conversión a kg al guardar inventario.** No se calcula al consultar (puede haber cambiado el factor).
8. **Pedidos sugeridos NO son pedidos ejecutados.** Solo se considera "real" lo que está en `pedidos_ejecutados`.
9. **Backups diarios obligatorios.** `pg_dump` validado con `pg_restore --list`. Si falla 2 días seguidos, alerta en runbook.
10. **Scripts legacy en `cremeria/` no se ejecutan más, pero NO se borran hasta 2026-06-15.**
11. **Postgres dedicado de Ritmo NUNCA toca BDs de EspritOS.** Container, puerto, volumen, usuario y red separados. Si un día Ritmo necesitara datos de EspritOS, será vía API o export, jamás cross-database query.
```

---

## 16. Reglas no negociables

(Estas son globales del proyecto, las del CLAUDE.md son las que Claude Code lee en cada interacción.)

1. **Cero scripts Python ad-hoc nuevos** en `cremeria/pedidos/` o `cremeria/reportes/marcas/` durante construcción de Ritmo y los 30 días siguientes. Si surge una pregunta nueva del equipo, **se resuelve en Ritmo o se prioriza una fase v1.1**, no se hace un script más.
2. **No tocar EspritOS, CRM-ERP, ni ninguna otra app.** Ritmo es agnóstica por decisión explícita.
3. **Docker solo para Postgres dedicado de Ritmo.** Streamlit corre como proceso Windows directo (NSSM). No microservicios, no Kubernetes, no Compose para la UI. La UI es 1 proceso Python.
4. **No agregar autenticación con password** en v1. Selector de usuario en header. Si el equipo crece, v2.
5. **No introducir frameworks JS** (React, Vue, etc.). Streamlit puro.
6. **No commitear** `.env`, `infra/postgres.env`, `app/logs/`, `datos/backups/`.
7. **Toda decisión arquitectónica nueva** se registra en `docs/decisions/YYYY-MM-DD-titulo.md` (formato ADR ligero) antes de implementarse.
8. **Tests obligatorios** para `compartido/velocidad/core.py`, `compartido/pedidos/recomendar.py`, `compartido/catalogo/resolver.py`, `compartido/persistencia/db.py`. Las pantallas Streamlit pueden no tener tests automatizados (sí smoke manuales).
9. **Manejo de procesos en deploy:** **nunca** `taskkill /IM python.exe` ni `streamlit` global, **nunca** `docker stop $(docker ps -aq)`. Solo PID específico del servicio NSSM o `docker compose stop ritmo-postgres` por nombre exacto del container.
10. **Calendario operativo HM es ley.** Si algún cálculo accidentalmente usa calendario MX oficial o calendario calendario, falla el test.
11. **Postgres dedicado NUNCA toca BDs de EspritOS.** Container, puerto, volumen, usuario y red separados. Test automatizado `test_aislamiento_espritos` verifica que la config no apunta a `localhost:5432` ni a `espritos-prod-postgres`.

---

## 17. Riesgos y mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| **MySQL PZ no responde** (red caída, BD apagada) | Media | Alto | Fallback a CSV local con banner amarillo "datos al DD/MM HH:MM" |
| **Factor pieza-kg incorrecto** (default 1.0 sin ajustar) | Media | Alto | UI marca SKUs con factor=1.0 y unidad≠kg en amarillo. Pantalla 3 bloquea guardar inventario con factor sin ajustar (opt-in para guardar de todas formas) |
| **Código de barras no en PZ** | Alta | Bajo | Resolver devuelve None, UI marca en rojo y permite crear producto en Ritmo |
| **Streamlit reinicia y pierde estado** | Media | Bajo | Todo el estado vive en Postgres. `session_state` solo para inputs temporales |
| **Beto fuera, app rota, nadie sabe arreglar** | Baja | Alto | RUNBOOK detallado + Jamie tiene acceso a Claude Code en su PC |
| **EspritOS evoluciona y absorbe parte de Ritmo** | Alta | Medio | Documentar en blueprint la posible migración futura. v1 NO se acopla a EspritOS |
| **Docker daemon se cae en 192.168.0.152** | Baja | Alto | Container con `restart: unless-stopped`. Si Docker muere completo, Streamlit muestra banner rojo. RUNBOOK con `docker compose restart` |
| **Backup `pg_dump` corrupto o silencioso** | Baja | Alto | `ritmo_backup.bat` valida con `pg_restore --list` después de cada dump. Si falla, WARNING en log + revisar `datos/backups/ritmo/YYYY-MM-DD/ritmo_db.toc.txt` |
| **Volumen `ritmo_pgdata` se llena** | Muy Baja | Alto | Tamaño esperado < 100 MB/año. Monitor manual mensual con `docker system df` |
| **Alguien apunta accidentalmente a `espritos-prod-postgres`** | Baja | Crítico | Test `test_aislamiento_espritos` falla la build. Container Ritmo en red bridge separada |
| **Velocidad consolidada datos1+datos9 confunde a Jamie** | Media | Medio | Riesgo aceptado v1. v1.1 agrega segmentación si surge la necesidad real |
| **Catálogo PZ cambia y rompe el importador** | Baja | Alto | Test `tests/test_catalogo_importador.py` valida shape esperada del schema PZ y falla loud si cambia |

---

## 18. Apéndices

### Apéndice A — Queries SQL canónicas de PZ

> Estas queries son la **fuente de verdad** validada contra `reporte_v2_canonico.py`. Extraerlas de los scripts existentes en `cremeria/reportes/marcas/velocidad_nayar.py` y adaptarlas a parámetros.

**Ventas Ruta (remisiones):**
```sql
SELECT
  r.Fecha AS fecha,
  rd.Clave AS clave_pz,
  rd.Cantidad AS cantidad,
  rd.Unidad AS unidad,
  'ruta' AS canal
FROM remisionesdet rd
JOIN remisiones r ON r.IdRemision = rd.IdRemision
WHERE rd.Clave IN (%s)
  AND r.Fecha BETWEEN %s AND %s
  AND r.Cancelada = 0
```

**Ventas Mostrador (tickets):**
```sql
SELECT
  t.Fecha AS fecha,
  td.Clave AS clave_pz,
  td.Cantidad AS cantidad,
  td.Unidad AS unidad,
  'mostrador' AS canal
FROM ticketsdet td
JOIN tickets t ON t.IdTicket = td.IdTicket
WHERE td.Clave IN (%s)
  AND t.Fecha BETWEEN %s AND %s
  AND t.Cancelado = 0
```

**Ventas Factura (facturasdet):**
```sql
SELECT
  f.Fecha AS fecha,
  fd.Clave AS clave_pz,
  fd.Cantidad AS cantidad,
  fd.Unidad AS unidad,
  'factura' AS canal
FROM facturasdet fd
JOIN facturas f ON f.IdFactura = fd.IdFactura
WHERE fd.Clave IN (%s)
  AND f.Fecha BETWEEN %s AND %s
  AND f.Cancelada = 0
```

**Consolidado (UNION ALL):**
```sql
SELECT * FROM (
  -- ruta
  UNION ALL
  -- mostrador
  UNION ALL
  -- factura
) v
ORDER BY clave_pz, fecha
```

**⚠ Verificar nombres exactos** de columnas en el schema PZ antes de F1. Hoy hay incertidumbre en `Cancelada` vs `Cancelado` y en algunas tablas el campo se llama `FechaDoc` o `FechaCaptura`. Esto se resuelve mirando los scripts existentes y/o haciendo `DESCRIBE` en MySQL.

### Apéndice B — Factores pieza-kg conocidos (seed inicial)

Extraer del código fuente de los scripts existentes:
- `cremeria/reportes/marcas/velocidad_nayar.py` — diccionario `PESO_POR_PIEZA` para 27 SKUs NAYAR.
- `cremeria/reportes/marcas/velocidad_san_antonio.py` — 35 SKUs SA+SM.
- `cremeria/pedidos/velocidad_live_sa.py` — variantes adicionales SA.
- `cremeria/pedidos/generar_pedido_andalucia.py` — SKUs ANDALUCIA.

**Formato del seed (`compartido/persistencia/seed/factores_conocidos.csv`):**

```csv
clave_pz,descripcion,marca,linea,unidad_compra,factor_kg,empaque_caja,proveedor_habitual
SA,Salchicha paquete SA,SAN ANTONIO,Salchichas,paq,3.0,24,SAN ANTONIO
SSM,Salchicha paquete SM,SAN ANTONIO,Salchichas,paq,3.0,24,SAN ANTONIO
QASM,Queso Amarillo SM,SAN ANTONIO,Quesos,pza,1.82,12,SAN ANTONIO
RCH,Chistorra granel,NAYAR,Chistorras,kg,1.0,,NAYAR
210,...,NAYAR,...,...,...,...,NAYAR
...
```

Después del seed, Beto puede ajustar factores desde la pantalla 3 (al capturar inventario) o desde una pantalla de admin (v1.1 si surge necesidad).

### Apéndice C — Cálculo de pedido sugerido (fórmula canónica)

```python
def recomendar_pedido_sku(
    vel_kg_dh: float,           # velocidad del SKU en la ventana elegida
    inv_kg: float,              # inventario actual en kg
    horizonte_dh: int,          # días hábiles a cubrir
    colchon_kg: float = 0.0,    # extra manual por SKU crítico
    factor_kg: float = 1.0,     # para convertir a unidades
    empaque_caja: int = None,   # para redondeo a cajas
) -> dict:
    need_kg = vel_kg_dh * horizonte_dh
    pedido_kg = max(0, need_kg - inv_kg + colchon_kg)
    pedido_unid = pedido_kg / factor_kg

    if empaque_caja:
        # redondear hacia arriba a cajas completas
        cajas = math.ceil(pedido_unid / empaque_caja)
        pedido_unid_final = cajas * empaque_caja
        pedido_kg_final = pedido_unid_final * factor_kg
    else:
        cajas = None
        pedido_unid_final = pedido_unid
        pedido_kg_final = pedido_kg

    return {
        "vel_kg_dh": vel_kg_dh,
        "inv_kg": inv_kg,
        "horizonte_dh": horizonte_dh,
        "need_kg": need_kg,
        "colchon_kg": colchon_kg,
        "pedido_kg_calculado": pedido_kg,
        "pedido_unid_calculado": pedido_unid,
        "cajas": cajas,
        "pedido_unid_final": pedido_unid_final,
        "pedido_kg_final": pedido_kg_final,
    }
```

### Apéndice D — Horizonte: lead_freq_seg vs fecha_objetivo

**Modo `lead_freq_seg`:**

```python
horizonte_dh = lead_dias + frecuencia_dias + seguridad_dias
```

Lógica: si pido hoy, llega en `lead_dias` dh. Hasta el próximo pedido pasan `frecuencia_dias` dh. Más `seguridad_dias` de colchón. **No** se convierten a días hábiles porque ya están en dh.

**Modo `fecha_objetivo`:**

```python
horizonte_dh = dias_habiles(hoy, fecha_objetivo)
```

Lógica: cubrir hasta exactamente esa fecha. Útil para "cubrir hasta cierre de mes" o "cubrir hasta vacaciones de proveedor".

### Apéndice E — Tendencia (clasificador)

```python
def clasificar_tendencia(v7d: float, v30d: float, v90d: float) -> str:
    """
    Compara ventanas para detectar aceleración/desaceleración.
    
    Reglas:
      - v7d > v30d * 1.15  Y  v30d > v90d * 1.10  →  'Acelerando'
      - v7d > v30d * 1.05                          →  'Subiendo'
      - 0.95 ≤ v7d/v30d ≤ 1.05                    →  'Estable'
      - v7d < v30d * 0.95                          →  'Bajando'
      - v7d < v30d * 0.80                          →  'Cayendo'
    """
    if v30d == 0:
        return "Sin datos"
    ratio_corto = v7d / v30d
    ratio_largo = v30d / v90d if v90d > 0 else 1.0
    
    if ratio_corto > 1.15 and ratio_largo > 1.10:
        return "Acelerando"
    elif ratio_corto > 1.05:
        return "Subiendo"
    elif ratio_corto < 0.80:
        return "Cayendo"
    elif ratio_corto < 0.95:
        return "Bajando"
    else:
        return "Estable"
```

### Apéndice F — Queries de reconocimiento PZ (para F0.5)

> Ejecutar todas con `espritos_reader` contra `192.168.0.200:3306/datos1`. Pegar outputs en `docs/recon/2026-05-XX-pz-schema-recon.md`.

**F.1 — Shape de las 9 tablas de venta y catálogo:**

```sql
DESCRIBE datos1.remisiones;
DESCRIBE datos1.remisionesdet;
DESCRIBE datos1.tickets;
DESCRIBE datos1.ticketsdet;
DESCRIBE datos1.facturas;
DESCRIBE datos1.facturasdet;
DESCRIBE datos1.productos;
DESCRIBE datos1.marcas;
DESCRIBE datos1.lineas;
```

**F.2 — ¿Cancelado o Cancelada? (¿en cuáles tablas?):**

```sql
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE, COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA='datos1'
  AND COLUMN_NAME REGEXP '^Cancel'
ORDER BY TABLE_NAME;
```

**F.3 — ¿Cómo se llama el campo de código de barras en `productos`?:**

```sql
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA='datos1' AND TABLE_NAME='productos'
  AND (COLUMN_NAME LIKE '%Bar%'
       OR COLUMN_NAME LIKE '%Codigo%'
       OR COLUMN_NAME LIKE '%UPC%'
       OR COLUMN_NAME LIKE '%EAN%');
```

**F.4 — ¿Cómo se marca un SKU descontinuado?:**

```sql
SELECT COLUMN_NAME, DATA_TYPE, COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA='datos1' AND TABLE_NAME='productos'
  AND (COLUMN_NAME LIKE '%Activo%'
       OR COLUMN_NAME LIKE '%Estatus%'
       OR COLUMN_NAME LIKE '%Status%'
       OR COLUMN_NAME LIKE '%Vigente%'
       OR COLUMN_NAME LIKE '%Descon%'
       OR COLUMN_NAME LIKE '%Baja%');
```

**F.5 — ¿`Fecha`, `FechaDoc`, `FechaCaptura`? (¿igual en las 3 tablas de cabecera?):**

```sql
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA='datos1'
  AND COLUMN_NAME LIKE '%Fecha%'
  AND TABLE_NAME IN ('remisiones','tickets','facturas')
ORDER BY TABLE_NAME, COLUMN_NAME;
```

**F.6 — Sample de 3 filas por tabla para inspección visual:**

```sql
SELECT * FROM datos1.remisiones    LIMIT 3;
SELECT * FROM datos1.remisionesdet LIMIT 3;
SELECT * FROM datos1.tickets       LIMIT 3;
SELECT * FROM datos1.ticketsdet    LIMIT 3;
SELECT * FROM datos1.facturas      LIMIT 3;
SELECT * FROM datos1.facturasdet   LIMIT 3;
SELECT * FROM datos1.productos     LIMIT 3;
SELECT * FROM datos1.marcas        LIMIT 5;
SELECT * FROM datos1.lineas        LIMIT 5;
```

**F.7 — Conteos para sanity check:**

```sql
SELECT COUNT(*) AS total_productos FROM datos1.productos;
-- Esperado: ~2,800

-- Después de F.4, sustituir <campo_estatus> por el campo real encontrado
-- SELECT <campo_estatus>, COUNT(*) FROM datos1.productos GROUP BY 1;
```

**F.8 — Verificar también en datos9 (Abarrotera):**

```sql
DESCRIBE datos9.productos;
SELECT COUNT(*) FROM datos9.productos;
-- ¿El schema es idéntico a datos1.productos? ¿Hay columnas extra?
```

**Formato del output `docs/recon/2026-05-XX-pz-schema-recon.md`:**

```markdown
# Reconocimiento schema PZ — DD/MM/AAAA

Ejecutado por: <nombre>
Cliente: <DBeaver|mysql CLI|PyMySQL>
Hora: HH:MM

## Resolución de incógnitas

| Pregunta | Respuesta canónica |
|---|---|
| Campo de cancelación en `remisiones` | `Cancelado` (TINYINT, default 0) |
| Campo de cancelación en `tickets` | ... |
| Campo de cancelación en `facturas` | ... |
| Campo código de barras en `productos` | `CodigoBarras` (VARCHAR(50)) |
| Campo descontinuado en `productos` | `Activo` (TINYINT, default 1) |
| Campo fecha en `remisiones` | `Fecha` (DATETIME) |
| Campo fecha en `tickets` | ... |
| Campo fecha en `facturas` | ... |

## Outputs completos

(pegar los DESCRIBE y SELECT aquí, dentro de bloques de código)
```

Este archivo es la **fuente de verdad** para las queries SQL del Apéndice A. Si después se descubre que un nombre canónico cambió (PZ actualizó el schema), se vuelve a correr F0.5 y se actualiza el archivo con fecha nueva.

---

## 19. Glosario para externos

| Término | Significado |
|---|---|
| **Ritmo** | Esta app. Velocidad de venta + inventarios + pedidos sugeridos. |
| **PZ** | PuntoZero, el POS MySQL de Cremería. Fuente de toda la venta. |
| **datos1** | Base MySQL de Cremería (matriz). Fuente autoritativa de catálogo. |
| **datos9** | Base MySQL de Abarrotera HM. |
| **EspritOS** | Plataforma corporativa Django en construcción. Reemplazará CRM-ERP. Ritmo NO depende de ella. |
| **CRM-ERP** | ERP Next.js que está siendo reemplazado por EspritOS. |
| **dh** | Día hábil de Cremería HM (L-S menos 4 días/año). |
| **kg/dh** | Velocidad de venta: kilogramos vendidos por día hábil. |
| **Factor pieza-kg** | Cuántos kg pesa una unidad cuando el SKU se vende empaquetado (paq SA = 3 kg). |
| **Ventana** | Periodo de N dh terminando en `cierre` para calcular velocidad. |
| **Cierre** | Última fecha incluida en el cálculo. Default = ayer. |
| **Tendencia** | Clasificación cualitativa comparando ventanas (Acelerando/Subiendo/Estable/Bajando/Cayendo). |
| **YoY** | Year over Year. Comparativo vs mismo periodo año anterior. |
| **Lead time** | dh entre que se pide al proveedor y llega. |
| **Frecuencia** | dh entre pedidos al mismo proveedor. |
| **Stock de seguridad** | dh extra de inventario como colchón. |
| **Horizonte** | dh totales a cubrir con el pedido = lead + freq + seg, o calculado por fecha objetivo. |
| **Contexto** | Etiqueta libre del inventario/pedido ("SAN ANTONIO", "ANDALUCIA", "ruta dia3", "ad-hoc"). No es lo mismo que marca. |
| **AOR** | Canal de venta "Atención Operativa Ruta" — clientes mayoreo de Andrea/Valeria. |

---

## 20. Decisiones tomadas en sesión de diseño (12/05/2026)

| # | Decisión | Alternativa descartada | Razón |
|---|---|---|---|
| 1 | Stack = Streamlit standalone | Módulo en EspritOS / Híbrido con Postgres EspritOS | Beto: la app debe ser agnóstica de cualquier ERP, ningún acoplamiento de datos |
| 2 | Persistencia = Postgres 16 dedicado en Docker (puerto 5433) | SQLite local / Postgres cluster EspritOS | Docker ya corre en 192.168.0.152 por EspritOS → costo de infra ≈ 0. Concurrencia real multi-writer, JSONB, futuro-proof. Container/puerto/volumen/usuario aislados de EspritOS (decisión refinada el 12/05/2026 tras cuestionar SQLite) |
| 3 | Catálogo = maestro propio + refs externas | Solo reflejo de PZ | Permite productos sin equivalente en ningún ERP |
| 4 | Import inicial PZ + sync manual | Sync automático nocturno / Just-in-time | Cero magia, control total. Beto importa cuando lo necesita |
| 5 | Captura inventario = formulario web tabular | Escaneo móvil / OCR foto / Híbrido | Lo más simple. Volumen bajo (35 SKUs/captura) no justifica complejidad móvil en v1 |
| 6 | Alcance v1 = 5 pantallas juntas (~3-4 semanas) | MVP delgado 2 pantallas / Ultra-delgado 1 | Beto eligió big-bang. Riesgo de retrabajo aceptado |
| 7 | Velocidad consolidada datos1+datos9 | Solo Cremería / Segmentación obligatoria | v1 simple. Segmentación = v1.1 si Jamie la pide |
| 8 | Mejoras en v1: pedidos_ejecutados + YoY + Críticos | Diferir todas a v1.1 | Las 3 son baratas y pagan caro |
| 9 | Backup = local en `datos/backups/ritmo/` | OneDrive | Beto: prefiere local en el repo. Rotación 30 días |
| 10 | Nombre = "Ritmo" | "VeloHM" / "Pulso HM" / "Cadencia" | Más español, evocativo |
| 11 | Scripts legacy = congelados hasta 2026-06-15 | Borrar al deploy | Red de seguridad mientras Ritmo estabiliza |
| 12 | Sin auth dura | Login con password | 4 usuarios LAN, selector de usuario en header es suficiente |
| 13 | Riesgos abiertos de schema PZ → F0.5 de reconocimiento con `espritos_reader` | Dejarlos como riesgos abiertos | Beto observó (12/05/2026) que las queries `DESCRIBE` y `INFORMATION_SCHEMA` resuelven en 10 min lo que el doc llamaba "riesgo." Evidencia commiteada en `docs/recon/` |

---

## 21. Pre-requisitos antes de F1

Los antiguos "riesgos abiertos" de schema PZ se resuelven en **F0.5 — Reconocimiento de schema PZ** con `espritos_reader`. El builder no avanza a F1 sin la evidencia en `docs/recon/`.

**Lista de checks antes de empezar el build:**

1. **Acceso a `espritos_reader` confirmado.** Credenciales disponibles en password manager de Beto o en `.env.dev`. Probar: `python -c "import pymysql; print(pymysql.connect(host='192.168.0.200', user='espritos_reader', password='...', database='datos1').open)"`.
2. **Docker Engine instalado y corriendo** en la PC donde se construye. Probar: `docker version` y `docker compose version`.
3. **Puertos libres:** 8200 (Streamlit Ritmo), 5433 (Postgres Ritmo). Verificar con `netstat -ano | findstr ":8200 :5433"` — deben aparecer vacíos o solo en estado LISTENING de Ritmo después del setup.
4. **Puerto 5432 ya ocupado por EspritOS** (esto es esperado, NO se toca).
5. **Espacio en `E:\` ≥ 5 GB** para volumen Docker (`ritmo_pgdata`) + backups + logs.
6. **Acceso de Jamie a Claude Code** configurado en su PC para soporte de emergencia.
7. **Validación con datos reales antes de F12:** no deprecar scripts viejos hasta que F11 lleve 14 días sin errores.

---

*Fin del blueprint. Para preguntas durante construcción, consultar primero apéndices, luego CLAUDE.md de la sección 15.*
