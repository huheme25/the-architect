# Terminal Financiero Personal — Blueprint

> Generado por The Architect el 04/06/2026
> Arquetipo: Internal Tool / Personal Dashboard
> Audiencia objetivo del blueprint: una instancia fresca de Claude Code (cero contexto previo) que construirá el proyecto de extremo a extremo.

---

## 1. Project Overview

### Visión

Herramienta personal de análisis financiero para un inversionista mexicano de largo plazo que opera vía GBM (BMV + SIC). Consolida datos de mercado globales, indicadores macroeconómicos, fundamentales de emisoras, herramientas de valuación, tracking de portafolio con cálculo fiscal mexicano (ISR), watchlist con alertas, sistema de metas orientado a ingreso pasivo por dividendos, y (V2) un agente IA que genera resumen semanal del portafolio.

El usuario **no hace trading activo**: analiza emisoras, construye portafolio orientado a apreciación de capital y dividendos, y toma decisiones con horizonte de meses a años. No requiere datos en tiempo real — cierre de día o delay de 15-20 min es aceptable.

La app corre **100% local** (Docker Compose en una máquina Windows personal), single-user, sin auth, accesible en `http://localhost:8301`. Sin dependencias de servicios pagados ni datos del usuario fuera de la máquina.

### Goals

- Eliminar el spreadsheet manual de portafolio actual y reemplazarlo con cálculos automáticos basados en precios en vivo.
- Tener una sola ventana para consultar fundamentales globales, indicadores macro MX/US, y posiciones propias.
- Calcular ISR mexicano correctamente (Art. 129 y 152 LISR, ajuste inflacionario INPC, piramidación de dividendos, distribución de FIBRAs).
- Trackear progreso hacia meta de ingreso pasivo por dividendos con proyección y simulación.
- (V2) Recibir un resumen semanal del portafolio escrito por un agente IA: precios, noticias, fundamentales, contexto macro y opinión de rebalanceo.

### Success Metrics

- Sustituir el 100% del spreadsheet de portafolio (cero uso del archivo de Excel actual).
- Cálculo de ISR coincide con la constancia fiscal anual de GBM (verificación al cierre del ejercicio).
- < 2 segundos para abrir la ficha de cualquier emisora (con caché tibio).
- Agente semanal V2: > 80% de las observaciones son accionables o informativas (no genéricas).

---

## 2. Tech Stack

| Capa | Tecnología | Por qué |
|---|---|---|
| Backend framework | **FastAPI 0.115+** | Async nativo (clave para llamar yfinance + Banxico + FRED en paralelo), type hints serios, OpenAPI automático |
| Backend lang | **Python 3.12** | Ecosistema financiero maduro (pandas, numpy, yfinance, fredapi) |
| Backend pkg mgr | **uv** | 10-100x más rápido que pip, lockfile reproducible |
| ORM | **SQLAlchemy 2.0 (async)** | Async-first, mature, queries complejas para series temporales |
| Migraciones | **Alembic** | Standard SQLAlchemy, autogenerate desde modelos |
| DB | **PostgreSQL 16-alpine** | JSONB para filtros/constancias, ventanas para TWR/MWR, partitioning para precios históricos |
| Cache | **Redis 7-alpine** | Cache de respuestas API con TTL por tipo de dato, broker de jobs |
| Jobs periódicos | **APScheduler 3.x (in-process)** | Refresh diario de precios, descarga mensual de INPC, evaluación de alertas. Suficiente para single-user; no requiere Celery |
| HTTP client | **httpx** | Async, type-safe, mejor que requests para FastAPI |
| Frontend framework | **React 19 + Vite 6** | Patrón ya validado en `equity-terminal` y `asistente-personal` del usuario |
| Frontend lang | **TypeScript 5.6+** strict | Type safety end-to-end |
| Frontend pkg mgr | **pnpm** | Velocidad + disk efficiency |
| Routing | **TanStack Router v1** (file-based, type-safe) | Type-safe routes con loaders, mejor DX que React Router para data-heavy apps |
| Styling | **Tailwind CSS v4** | Sin runtime, alineado con stack convenido |
| Componentes UI | **shadcn/ui** | Componentes accesibles, customizables, copy-paste (sin lock-in) |
| Tablas | **TanStack Table v8** | Sort/filter/pagination server-side, columnas configurables (clave para Screener) |
| Charts | **Plotly.js + react-plotly.js** | Convención del stack del usuario (`stack-y-convenciones.md`); superior para series financieras con rangos seleccionables, overlays SMA, comparativas normalizadas |
| State server | **TanStack Query v5** | Cache cliente, refetch automático, invalidación tras mutaciones |
| Forms | **React Hook Form + Zod** | Validación type-safe en transacciones, metas, filtros |
| Iconos | **lucide-react** | Default de shadcn/ui |
| LLM (V2) | **Anthropic Claude API (Sonnet 4.6)** con adapter para **Ollama** | Patrón EspritOS; modelo intercambiable vía env |
| PDF (V2) | **WeasyPrint** (Python, server-side) | Export del resumen semanal a PDF con HTML/CSS — sin dependencias headless browser |
| Empaque | **Docker Compose** (4 servicios: db, redis, backend, frontend-nginx) | Aislamiento, mismo patrón cluster HM pero standalone |
| Hosting | **localhost en Windows** (la PC del usuario) | Single-user, datos privados |
| Auth | **Ninguno** (bind a 127.0.0.1) | Localhost-only; si en el futuro se expone vía Cloudflare Tunnel, agregar basic auth con `.env.production` |

---

## 3. Directory Structure

```
terminal-financiero/
├── docker-compose.yml              # Stack completo: postgres + redis + backend + frontend
├── docker-compose.dev.yml          # Override: monta volúmenes para hot-reload (Vite + uvicorn --reload)
├── .env.example                    # Plantilla de variables (commiteable)
├── .gitignore                      # Incluye .env, .env.local, node_modules, __pycache__, .pytest_cache
├── README.md                       # Setup, comandos, troubleshooting
├── CLAUDE.md                       # Convenciones del proyecto (ver sección 15)
│
├── backend/
│   ├── pyproject.toml              # Dependencias (uv)
│   ├── uv.lock
│   ├── Dockerfile                  # Python 3.12-slim + uv sync
│   ├── .python-version             # 3.12
│   ├── alembic.ini
│   ├── alembic/
│   │   ├── env.py
│   │   ├── script.py.mako
│   │   └── versions/               # Migraciones generadas
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                 # FastAPI app, CORS, routers, startup/shutdown
│   │   ├── config.py               # Pydantic Settings desde .env
│   │   ├── db.py                   # AsyncEngine, AsyncSessionLocal, get_db dependency
│   │   ├── cache.py                # Redis client async
│   │   ├── deps.py                 # Dependencies compartidas (get_db, get_cache)
│   │   │
│   │   ├── models/                 # SQLAlchemy ORM
│   │   │   ├── __init__.py
│   │   │   ├── base.py             # DeclarativeBase + TimestampMixin
│   │   │   ├── emisora.py
│   │   │   ├── precio.py
│   │   │   ├── fundamental.py
│   │   │   ├── estado_financiero.py
│   │   │   ├── transaccion.py
│   │   │   ├── fibra_distribution.py
│   │   │   ├── watchlist.py
│   │   │   ├── alerta.py
│   │   │   ├── macro.py
│   │   │   ├── inpc.py
│   │   │   ├── nota.py
│   │   │   ├── filtro_guardado.py
│   │   │   └── meta.py
│   │   │
│   │   ├── schemas/                # Pydantic v2 (request/response)
│   │   │   ├── emisora.py
│   │   │   ├── transaccion.py
│   │   │   ├── portafolio.py
│   │   │   ├── macro.py
│   │   │   ├── screener.py
│   │   │   ├── valuacion.py
│   │   │   ├── fiscal.py
│   │   │   ├── watchlist.py
│   │   │   └── meta.py
│   │   │
│   │   ├── connectors/             # Acceso a APIs externas
│   │   │   ├── __init__.py
│   │   │   ├── base.py             # CachedClient con TTL + budget tracker
│   │   │   ├── yfinance_client.py
│   │   │   ├── banxico_client.py   # SIE API: CETES, TIIE, USD/MXN, UDIS, PIB
│   │   │   ├── fred_client.py      # Fed funds, CPI, treasury yields, GDP
│   │   │   ├── inegi_client.py     # PIB, INPC
│   │   │   ├── fmp_client.py       # Financial Modeling Prep (fundamentales US, screener)
│   │   │   └── sat_client.py       # Tablas ISR, factor piramidación (dataset estático)
│   │   │
│   │   ├── services/               # Lógica de negocio (orquesta connectors + DB)
│   │   │   ├── __init__.py
│   │   │   ├── portfolio.py        # Posiciones, efectivo, TWR, MWR, distribución
│   │   │   ├── dividendos.py       # Agrupación mensual/trimestral/anual, FIBRA detection
│   │   │   ├── fiscal.py           # ISR ventas (Art. 129), dividendos (CUFIN vs no-CUFIN), anual (Art. 152), ajuste INPC
│   │   │   ├── valuacion.py        # DCF, DDM, comparables
│   │   │   ├── screener.py         # Filtrado de emisoras
│   │   │   ├── comparador.py       # Normalización base 100, métricas relativas
│   │   │   ├── goals.py            # Cálculo de progreso, proyección
│   │   │   ├── alerts.py           # Evaluación de condiciones
│   │   │   ├── macro.py            # Agregación de indicadores
│   │   │   └── agent_weekly.py     # V2 — orquesta LLM
│   │   │
│   │   ├── jobs/                   # APScheduler tasks
│   │   │   ├── __init__.py
│   │   │   ├── scheduler.py        # Setup APScheduler, lifecycle hooks
│   │   │   ├── refresh_prices.py   # Diario 22:00 MX (cierre BMV + cierre US)
│   │   │   ├── refresh_macro.py    # Diario 08:00
│   │   │   ├── refresh_inpc.py     # Día 10 de cada mes
│   │   │   └── evaluate_alerts.py  # Cada 30 min mientras app está abierta
│   │   │
│   │   ├── routers/                # FastAPI endpoints
│   │   │   ├── __init__.py
│   │   │   ├── health.py
│   │   │   ├── emisoras.py
│   │   │   ├── precios.py
│   │   │   ├── macro.py
│   │   │   ├── portafolio.py
│   │   │   ├── transacciones.py
│   │   │   ├── dividendos.py
│   │   │   ├── screener.py
│   │   │   ├── comparador.py
│   │   │   ├── valuacion.py
│   │   │   ├── fiscal.py
│   │   │   ├── watchlist.py
│   │   │   ├── alertas.py
│   │   │   ├── metas.py
│   │   │   ├── notas.py
│   │   │   ├── filtros.py
│   │   │   ├── import_csv.py
│   │   │   └── agent.py            # V2
│   │   │
│   │   ├── llm/                    # Cliente LLM (V2)
│   │   │   ├── __init__.py
│   │   │   ├── base.py             # Interface común
│   │   │   ├── anthropic_client.py
│   │   │   └── ollama_client.py
│   │   │
│   │   └── utils/
│   │       ├── inpc_factor.py      # Factor de actualización (mes compra vs mes anterior venta)
│   │       ├── fibra_detector.py   # Lista hardcoded + lookup
│   │       ├── ticker_resolver.py  # Mapeo BMV (.MX) vs SIC vs sufijos especiales
│   │       └── formatters.py       # MXN, fechas DD/MM/AAAA
│   │
│   └── tests/
│       ├── conftest.py             # pytest-asyncio, DB de prueba (postgres test container)
│       ├── unit/
│       │   ├── test_fiscal.py      # Casos canónicos ISR + piramidación
│       │   ├── test_portfolio.py   # TWR, MWR, costo promedio
│       │   ├── test_valuacion.py   # DCF, DDM con valores conocidos
│       │   └── test_inpc_factor.py
│       └── integration/
│           ├── test_api_portafolio.py
│           ├── test_api_emisoras.py
│           └── test_jobs.py
│
├── frontend/
│   ├── package.json                # pnpm
│   ├── pnpm-lock.yaml
│   ├── vite.config.ts
│   ├── tsconfig.json               # strict: true
│   ├── tailwind.config.ts
│   ├── postcss.config.js
│   ├── components.json             # shadcn/ui config
│   ├── index.html
│   ├── Dockerfile                  # build → nginx:alpine
│   ├── nginx.conf                  # SPA fallback + proxy /api → backend
│   ├── public/
│   │   └── favicon.svg
│   └── src/
│       ├── main.tsx
│       ├── App.tsx                 # RouterProvider de TanStack Router
│       ├── routeTree.gen.ts        # Generado por TanStack Router
│       ├── styles/
│       │   └── globals.css         # Tailwind directives + tokens dark theme
│       ├── lib/
│       │   ├── api.ts              # Fetch wrapper tipado contra OpenAPI
│       │   ├── queryClient.ts      # TanStack Query setup
│       │   ├── formatters.ts       # formatMXN, formatPct, formatDate, formatCompact
│       │   ├── colors.ts           # getReturnColor (verde/rojo), getStrategyColor
│       │   └── utils.ts            # cn() (clsx + tailwind-merge)
│       ├── hooks/
│       │   ├── usePortafolio.ts
│       │   ├── useEmisora.ts
│       │   ├── useMacro.ts
│       │   ├── useTransacciones.ts
│       │   ├── useScreener.ts
│       │   ├── useWatchlist.ts
│       │   └── useMetas.ts
│       ├── components/
│       │   ├── ui/                 # shadcn/ui primitives (button, input, dialog, tabs, ...)
│       │   ├── layout/
│       │   │   ├── AppShell.tsx
│       │   │   ├── Sidebar.tsx     # Navegación entre módulos
│       │   │   └── Header.tsx      # Saldo total + cambio del día
│       │   ├── charts/
│       │   │   ├── PriceChart.tsx          # Línea de precio + volumen
│       │   │   ├── MacroChart.tsx          # Serie temporal de indicador macro
│       │   │   ├── ComparisonChart.tsx     # N emisoras normalizadas base 100
│       │   │   ├── DistributionPie.tsx     # Distribución del portafolio
│       │   │   ├── DividendBars.tsx        # Ingresos por dividendos por mes/quarter
│       │   │   ├── GoalProgress.tsx        # Barra de progreso con marcador
│       │   │   └── plotlyTheme.ts          # Tema dark Plotly consistente
│       │   ├── tables/
│       │   │   ├── DataTable.tsx           # Wrapper genérico TanStack Table
│       │   │   ├── PositionsTable.tsx
│       │   │   ├── TransactionsTable.tsx
│       │   │   ├── DividendsTable.tsx
│       │   │   └── ScreenerResultsTable.tsx
│       │   ├── forms/
│       │   │   ├── TransactionForm.tsx     # Compra, venta, dividendo, depósito, retiro
│       │   │   ├── GoalForm.tsx
│       │   │   ├── AlertForm.tsx
│       │   │   └── ScreenerFilters.tsx
│       │   └── shared/
│       │       ├── KPICard.tsx             # Tarjeta de métrica con cambio (▲▼)
│       │       ├── TickerBadge.tsx         # Con bandera país + tipo instrumento
│       │       ├── PriceCell.tsx           # Formato moneda con color de cambio
│       │       └── EmptyState.tsx
│       └── routes/                 # File-based routing (TanStack Router)
│           ├── __root.tsx          # AppShell wrapper
│           ├── index.tsx           # Dashboard general
│           ├── macro.tsx
│           ├── emisora.$ticker.tsx # Ficha completa
│           ├── screener.tsx
│           ├── comparador.tsx
│           ├── portafolio/
│           │   ├── index.tsx       # Tabs container
│           │   ├── posiciones.tsx
│           │   ├── transacciones.tsx
│           │   ├── dividendos.tsx
│           │   ├── rendimiento.tsx
│           │   └── distribucion.tsx
│           ├── watchlist.tsx
│           ├── valuacion.tsx       # DCF / DDM / Comparables (tabs)
│           ├── fiscal.tsx          # Calculadora ISR (tabs: ventas / dividendos / anual)
│           ├── metas.tsx
│           ├── resumen.tsx         # V2 — agente semanal
│           └── configuracion.tsx   # API keys, comisión default, LLM provider
│
├── data/
│   ├── seed/
│   │   ├── emisoras_bmv.json       # ~145 emisoras de BMV con metadata
│   │   ├── fibras_mx.json          # Listado curado FIBRAs MX
│   │   ├── sp500.json              # 500 emisoras US
│   │   ├── isr_tarifa_2026.json    # Tabla Art. 152 vigente
│   │   └── inpc_historico.json     # INPC 2018+ (seed inicial, después job mensual)
│   └── README.md                   # Notas sobre origen de los datasets
│
└── scripts/
    ├── init_db.sql                 # CREATE EXTENSION pg_trgm; (para búsqueda fuzzy de tickers)
    ├── seed_emisoras.py            # Carga inicial del universo
    ├── import_csv.py               # CLI standalone para importar transacciones
    └── refresh_all.py              # Manual: forzar refresh de toda la data
```

---

## 4. Data Model

### Entidades

**emisoras** — Universo de tickers conocidos
| Campo | Tipo | Notas |
|---|---|---|
| ticker | TEXT | PK — ej. `AAPL`, `FUNO11.MX`, `KOFUBL.MX` |
| nombre | TEXT | NOT NULL |
| bolsa | TEXT | NYSE / NASDAQ / BMV / SIC / LSE / etc. |
| sector | TEXT | |
| industria | TEXT | |
| pais | TEXT | ISO 3166-1 alpha-2 (MX, US, JP, ...) |
| moneda | TEXT | MXN / USD / EUR ... |
| tipo_instrumento | TEXT | enum: `accion`, `fibra`, `etf`, `ckd`, `bono` |
| es_fibra | BOOLEAN | Computed/seeded — facilita queries |
| activa | BOOLEAN | Default true |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**precios_historicos** — OHLCV. Particionada por año
| Campo | Tipo | Notas |
|---|---|---|
| ticker | TEXT | FK emisoras |
| fecha | DATE | |
| open | NUMERIC(20,6) | |
| high | NUMERIC(20,6) | |
| low | NUMERIC(20,6) | |
| close | NUMERIC(20,6) | NOT NULL |
| adj_close | NUMERIC(20,6) | Ajustado por splits/dividendos |
| volume | BIGINT | |
| PK | (ticker, fecha) | |

**fundamentales** — Snapshot de ratios y métricas
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| ticker | TEXT | FK |
| fecha_snapshot | DATE | Cuándo se capturó |
| periodo | TEXT | `TTM`, `2025-Q4`, `2025-FY` |
| metric_name | TEXT | `pe_ratio`, `roe`, `dividend_yield`, ... |
| metric_value | NUMERIC(20,6) | |
| Unique | (ticker, fecha_snapshot, periodo, metric_name) | |

**estados_financieros** — Income Statement, Balance Sheet, Cash Flow
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| ticker | TEXT | FK |
| tipo | TEXT | enum: `income`, `balance`, `cashflow` |
| periodo | TEXT | `2025-Q4`, `2025-FY` |
| fecha | DATE | Fin del período |
| line_item | TEXT | `revenue`, `gross_profit`, `total_assets`, ... |
| valor | NUMERIC(20,2) | En moneda de la emisora |
| Unique | (ticker, tipo, periodo, line_item) | |

**transacciones** — Movimientos del portafolio
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| fecha | DATE | NOT NULL |
| tipo | TEXT | enum: `buy`, `sell`, `dividend`, `deposit`, `withdrawal` |
| ticker | TEXT | NULL para `deposit`/`withdrawal` |
| acciones | NUMERIC(20,6) | Negativo para `sell`; 0 para deposit/withdrawal/dividend; ticker holdings para dividend |
| precio | NUMERIC(20,6) | Por acción para buy/sell; monto por acción para dividend |
| comision | NUMERIC(20,2) | 0 para dividend/deposit/withdrawal |
| total | NUMERIC(20,2) | Generated column: `acciones * precio + comision` para buy; `acciones * precio - comision` para sell; `precio` directo para deposit/withdrawal; `acciones * precio` para dividend |
| moneda | TEXT | MXN / USD |
| estrategia | TEXT | enum: `patrimonial`, `indizado`, `trading`, `n/a` (para cash) |
| notas | TEXT | NULL |
| es_fibra | BOOLEAN | Marca al insertar para tracking ISR |
| created_at | TIMESTAMPTZ | |

**fibra_distributions** — Desglose por componente de distribuciones FIBRA
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| transaccion_id | UUID | FK transacciones (la transacción tipo `dividend` original) |
| componente | TEXT | enum: `resultado_fiscal`, `reembolso_capital`, `utilidad_contable` |
| monto | NUMERIC(20,2) | Por acción |
| isr_retenido | NUMERIC(20,2) | Si aplica |

**watchlists** — Listas de observación
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| nombre | TEXT | UNIQUE |
| created_at | TIMESTAMPTZ | |

**watchlist_items**
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| watchlist_id | UUID | FK |
| ticker | TEXT | FK emisoras |
| nota_rapida | TEXT | NULL |
| fecha_agregado | TIMESTAMPTZ | |
| Unique | (watchlist_id, ticker) | |

**alertas** — Condiciones evaluadas periódicamente
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| ticker | TEXT | FK emisoras |
| campo | TEXT | enum: `precio`, `pe_ratio`, `dividend_yield`, `roe`, ... |
| operador | TEXT | enum: `gt`, `lt`, `eq`, `gte`, `lte` |
| umbral | NUMERIC(20,6) | |
| activa | BOOLEAN | Default true |
| ultima_evaluacion | TIMESTAMPTZ | NULL si nunca evaluada |
| disparada | BOOLEAN | Default false |
| fecha_disparo | TIMESTAMPTZ | NULL hasta que se cumpla |
| created_at | TIMESTAMPTZ | |

**macro_series** — Catálogo de indicadores macro
| Campo | Tipo | Notas |
|---|---|---|
| serie_id | TEXT | PK — ej. `BMX_SF43936` (CETES 28), `FRED_FEDFUNDS` |
| fuente | TEXT | enum: `banxico`, `fred`, `inegi`, `yfinance` |
| nombre | TEXT | Human-readable: "CETES 28 días" |
| descripcion | TEXT | |
| frecuencia | TEXT | enum: `daily`, `weekly`, `monthly`, `quarterly` |
| categoria | TEXT | enum: `tasa`, `inflacion`, `tipo_cambio`, `pib`, `indice`, `otro` |
| pais | TEXT | MX / US / GLOBAL / EU / JP / ... |
| unidad | TEXT | `%`, `MXN`, `index`, etc. |

**macro_valores**
| Campo | Tipo | Notas |
|---|---|---|
| serie_id | TEXT | FK |
| fecha | DATE | |
| valor | NUMERIC(20,6) | |
| PK | (serie_id, fecha) | |

**inpc** — Índice Nacional de Precios al Consumidor (clave para ajuste fiscal)
| Campo | Tipo | Notas |
|---|---|---|
| anio | INT | |
| mes | INT | 1-12 |
| valor | NUMERIC(20,6) | |
| PK | (anio, mes) | |

**notas_usuario** — Notas libres por emisora
| Campo | Tipo | Notas |
|---|---|---|
| ticker | TEXT | PK |
| contenido | TEXT | |
| updated_at | TIMESTAMPTZ | |

**filtros_guardados** — Combinaciones de filtros del Screener
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| nombre | TEXT | UNIQUE |
| criterios | JSONB | Schema validado por Pydantic en `app/schemas/screener.py` |
| created_at | TIMESTAMPTZ | |

**metas** — Sistema de metas
| Campo | Tipo | Notas |
|---|---|---|
| id | UUID | PK |
| tipo | TEXT | enum: `dividendo_mensual`, `dividendo_anual`, `posicion`, `valor_portafolio`, `distribucion_estrategia`, `distribucion_moneda`, `num_emisoras` |
| nombre | TEXT | |
| target_value | NUMERIC(20,2) | NULL para metas tipo "distribución" |
| target_json | JSONB | NULL salvo para distribuciones: `{"patrimonial": 70, "indizado": 25, "trading": 5}` |
| ticker | TEXT | NULL salvo metas tipo `posicion` |
| activa | BOOLEAN | Default true |
| cumplida | BOOLEAN | Default false |
| fecha_cumplida | TIMESTAMPTZ | NULL hasta lograrla |
| created_at | TIMESTAMPTZ | |

### Vistas calculadas (no son tablas)

**v_posiciones** — Posición neta actual por ticker
- SELECT ticker, SUM(CASE WHEN tipo='buy' THEN acciones WHEN tipo='sell' THEN -acciones ELSE 0 END) AS acciones_netas, costo_promedio_ponderado(...) AS precio_promedio_compra, ...

**v_cash_balance** — Saldo de efectivo
- SELECT SUM(CASE WHEN tipo='deposit' THEN total WHEN tipo='withdrawal' THEN -total WHEN tipo='buy' THEN -total WHEN tipo='sell' THEN total WHEN tipo='dividend' THEN total END) AS saldo

**Por qué vistas y no tablas**: Las posiciones son función pura del historial de transacciones. Mantenerlas como tabla introduce un side-effect a sincronizar; como vista, son siempre correctas. Si la performance se degrada (poco probable con < 50k transacciones), se cambia a vista materializada con REFRESH on trigger.

### Relaciones

```
emisoras ─┬─ 1:N ─ precios_historicos
          ├─ 1:N ─ fundamentales
          ├─ 1:N ─ estados_financieros
          ├─ 1:N ─ transacciones
          ├─ 1:N ─ watchlist_items
          ├─ 1:N ─ alertas
          ├─ 1:1 ─ notas_usuario
          └─ 1:N ─ metas (cuando tipo=posicion)

transacciones ── 1:N ─ fibra_distributions (solo cuando tipo=dividend y es_fibra=true)
watchlists ─── 1:N ─ watchlist_items
macro_series ─ 1:N ─ macro_valores
```

### Schema SQL — extracto crítico

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;        -- Búsqueda fuzzy de tickers/nombres
CREATE EXTENSION IF NOT EXISTS pgcrypto;       -- gen_random_uuid()

-- emisoras
CREATE TABLE emisoras (
  ticker             TEXT PRIMARY KEY,
  nombre             TEXT NOT NULL,
  bolsa              TEXT NOT NULL,
  sector             TEXT,
  industria          TEXT,
  pais               TEXT NOT NULL,
  moneda             TEXT NOT NULL CHECK (moneda IN ('MXN','USD','EUR','GBP','JPY','CAD')),
  tipo_instrumento   TEXT NOT NULL CHECK (tipo_instrumento IN ('accion','fibra','etf','ckd','bono')),
  es_fibra           BOOLEAN GENERATED ALWAYS AS (tipo_instrumento = 'fibra') STORED,
  activa             BOOLEAN NOT NULL DEFAULT true,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_emisoras_bolsa ON emisoras(bolsa);
CREATE INDEX idx_emisoras_sector ON emisoras(sector);
CREATE INDEX idx_emisoras_nombre_trgm ON emisoras USING gin (nombre gin_trgm_ops);

-- precios_historicos (particionada por año)
CREATE TABLE precios_historicos (
  ticker      TEXT NOT NULL REFERENCES emisoras(ticker) ON DELETE CASCADE,
  fecha       DATE NOT NULL,
  open        NUMERIC(20,6),
  high        NUMERIC(20,6),
  low         NUMERIC(20,6),
  close       NUMERIC(20,6) NOT NULL,
  adj_close   NUMERIC(20,6),
  volume      BIGINT,
  PRIMARY KEY (ticker, fecha)
) PARTITION BY RANGE (fecha);

-- Particiones por año (crear 2018-2030 al init; agregar nuevas con job)
CREATE TABLE precios_historicos_2026 PARTITION OF precios_historicos
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
-- repetir para cada año

-- transacciones
CREATE TABLE transacciones (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fecha        DATE NOT NULL,
  tipo         TEXT NOT NULL CHECK (tipo IN ('buy','sell','dividend','deposit','withdrawal')),
  ticker       TEXT REFERENCES emisoras(ticker),
  acciones     NUMERIC(20,6) NOT NULL DEFAULT 0,
  precio       NUMERIC(20,6) NOT NULL DEFAULT 0,
  comision     NUMERIC(20,2) NOT NULL DEFAULT 0,
  total        NUMERIC(20,2) NOT NULL,
  moneda       TEXT NOT NULL DEFAULT 'MXN',
  estrategia   TEXT NOT NULL DEFAULT 'patrimonial'
                 CHECK (estrategia IN ('patrimonial','indizado','trading','n/a')),
  notas        TEXT,
  es_fibra     BOOLEAN NOT NULL DEFAULT false,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT chk_ticker_required CHECK (
    (tipo IN ('buy','sell','dividend') AND ticker IS NOT NULL) OR
    (tipo IN ('deposit','withdrawal'))
  )
);
CREATE INDEX idx_transacciones_ticker_fecha ON transacciones(ticker, fecha DESC);
CREATE INDEX idx_transacciones_tipo_fecha ON transacciones(tipo, fecha DESC);

-- v_posiciones (vista)
CREATE OR REPLACE VIEW v_posiciones AS
SELECT
  t.ticker,
  SUM(CASE WHEN t.tipo='buy' THEN t.acciones
           WHEN t.tipo='sell' THEN -t.acciones ELSE 0 END) AS acciones_netas,
  -- Costo promedio ponderado solo sobre compras vigentes (no consumidas por sells)
  -- Se calcula en el servicio en Python por simplicidad y precisión FIFO opcional
  SUM(CASE WHEN t.tipo='buy' THEN t.total ELSE 0 END)
    - SUM(CASE WHEN t.tipo='sell' THEN t.total ELSE 0 END) AS costo_base_simple,
  MAX(t.moneda) AS moneda,
  MAX(t.estrategia) AS estrategia
FROM transacciones t
WHERE t.ticker IS NOT NULL
GROUP BY t.ticker
HAVING SUM(CASE WHEN t.tipo='buy' THEN t.acciones
                WHEN t.tipo='sell' THEN -t.acciones ELSE 0 END) <> 0;

-- v_cash_balance
CREATE OR REPLACE VIEW v_cash_balance AS
SELECT
  moneda,
  SUM(CASE
    WHEN tipo='deposit' THEN total
    WHEN tipo='withdrawal' THEN -total
    WHEN tipo='buy' THEN -total
    WHEN tipo='sell' THEN total
    WHEN tipo='dividend' THEN total
    ELSE 0
  END) AS saldo
FROM transacciones
GROUP BY moneda;
```

> **Nota crítica para el builder**: El cálculo de **costo promedio ponderado** y **ganancia realizada vs no realizada** se hace en `app/services/portfolio.py` con pandas (no en SQL). Razón: lotes FIFO o promedio ponderado con ajuste por splits son lógica con muchos casos edge — mejor en código testeable que en SQL.

---

## 5. API Design

### Estilo

- **REST + JSON**, OpenAPI generado por FastAPI en `/docs`.
- Prefijo `/api/v1/` para todos los endpoints.
- Respuestas envueltas: `{"data": ..., "meta": {...}}` para listados, objeto plano para single.
- Errores: `{"error": {"code": "...", "message": "...", "detail": ...}}` con HTTP status apropiado.
- Sin auth (localhost), pero CORS solo permite origen `http://localhost:5173` (dev) y `http://localhost:8301` (prod).

### Routes Overview

| Method | Path | Descripción |
|---|---|---|
| GET | `/api/v1/health` | Healthcheck (db + redis + connectors) |
| **Emisoras** | | |
| GET | `/api/v1/emisoras/search?q=AAPL` | Búsqueda fuzzy por ticker o nombre |
| GET | `/api/v1/emisoras/{ticker}` | Ficha completa (metadata + último precio + fundamentales TTM + dividendos) |
| GET | `/api/v1/emisoras/{ticker}/precios?range=1Y` | Serie histórica OHLCV |
| GET | `/api/v1/emisoras/{ticker}/fundamentales` | Todos los ratios disponibles |
| GET | `/api/v1/emisoras/{ticker}/estados-financieros?tipo=income&period=quarterly` | |
| GET | `/api/v1/emisoras/{ticker}/dividendos` | Historial completo |
| POST | `/api/v1/emisoras` | Agregar emisora manual al universo |
| **Notas** | | |
| GET | `/api/v1/notas/{ticker}` | |
| PUT | `/api/v1/notas/{ticker}` | Upsert |
| **Macro** | | |
| GET | `/api/v1/macro/series?pais=MX&categoria=tasa` | Catálogo |
| GET | `/api/v1/macro/series/{serie_id}/valores?range=5Y` | Serie temporal |
| GET | `/api/v1/macro/dashboard?pais=MX` | Snapshot de KPIs para tarjetas |
| **Portafolio** | | |
| GET | `/api/v1/portafolio/dashboard` | Valor total, cash, P&L, métricas resumen |
| GET | `/api/v1/portafolio/posiciones` | Lista con precios en vivo y P&L unrealized |
| GET | `/api/v1/portafolio/distribucion?by=sector` | Por sector / estrategia / país / moneda / tipo |
| GET | `/api/v1/portafolio/rendimiento?from=2024-01-01&benchmark=^MXX` | TWR/MWR + benchmark |
| **Transacciones** | | |
| GET | `/api/v1/transacciones?ticker=...&tipo=...&from=...&to=...` | Listado paginado |
| POST | `/api/v1/transacciones` | Crear |
| PATCH | `/api/v1/transacciones/{id}` | Editar |
| DELETE | `/api/v1/transacciones/{id}` | Borrar |
| POST | `/api/v1/transacciones/import-csv` | Multipart upload |
| **Dividendos** | | |
| GET | `/api/v1/dividendos/recibidos?from=...&to=...` | |
| GET | `/api/v1/dividendos/calendario` | Ex-dates y fechas de pago próximas de holdings |
| GET | `/api/v1/dividendos/ingresos-agrupados?by=month` | Para gráficas |
| **Screener** | | |
| POST | `/api/v1/screener/run` | Body: filtros JSON → resultados paginados |
| GET | `/api/v1/screener/filtros-guardados` | |
| POST | `/api/v1/screener/filtros-guardados` | |
| DELETE | `/api/v1/screener/filtros-guardados/{id}` | |
| **Comparador** | | |
| POST | `/api/v1/comparador` | Body: `{"tickers":[...], "from":"..."}` → tabla + serie normalizada |
| **Valuación** | | |
| POST | `/api/v1/valuacion/dcf` | Body: inputs DCF → valor intrínseco + tabla sensibilidad |
| POST | `/api/v1/valuacion/ddm` | |
| POST | `/api/v1/valuacion/comparables` | Body: `{"ticker":..., "peers":[...]}` |
| **Fiscal** | | |
| POST | `/api/v1/fiscal/isr-venta` | Calcula ISR Art. 129 con ajuste INPC |
| POST | `/api/v1/fiscal/isr-dividendo` | Maneja CUFIN vs no-CUFIN vs FIBRA |
| POST | `/api/v1/fiscal/isr-anual` | Estimación con tarifa progresiva Art. 152 |
| GET | `/api/v1/fiscal/inpc` | Tabla INPC completa |
| **Watchlist** | | |
| GET | `/api/v1/watchlists` | |
| POST | `/api/v1/watchlists` | |
| GET | `/api/v1/watchlists/{id}/items` | Con precios y métricas en vivo |
| POST | `/api/v1/watchlists/{id}/items` | |
| DELETE | `/api/v1/watchlists/{id}/items/{ticker}` | |
| **Alertas** | | |
| GET | `/api/v1/alertas?activa=true` | |
| GET | `/api/v1/alertas/disparadas` | Las que cumplen condición hoy |
| POST | `/api/v1/alertas` | |
| DELETE | `/api/v1/alertas/{id}` | |
| **Metas** | | |
| GET | `/api/v1/metas?activa=true` | Con progreso calculado |
| POST | `/api/v1/metas` | |
| PATCH | `/api/v1/metas/{id}` | |
| DELETE | `/api/v1/metas/{id}` | |
| POST | `/api/v1/metas/{id}/simular` | Body: `{"ticker":..., "acciones":N}` → cómo cambia el progreso |
| **Agent (V2)** | | |
| POST | `/api/v1/agent/resumen-semanal` | Trigger manual, retorna markdown |
| GET | `/api/v1/agent/resumenes?limit=10` | Historial |
| GET | `/api/v1/agent/resumenes/{id}/pdf` | Export |

### Detalle de endpoints críticos

#### POST `/api/v1/fiscal/isr-venta`

**Request:**
```json
{
  "ticker": "KOFUBL.MX",
  "lotes_compra": [
    {"fecha": "2023-03-15", "acciones": 100, "precio_unitario": 145.50, "comision": 50.00}
  ],
  "venta": {"fecha": "2026-05-20", "acciones": 100, "precio_unitario": 178.00, "comision": 55.00}
}
```

**Response:**
```json
{
  "ticker": "KOFUBL.MX",
  "costo_base_total": 14600.00,
  "costo_promedio_por_accion": 146.00,
  "inpc_mes_compra_anterior": 128.547,
  "inpc_mes_venta_anterior": 134.892,
  "factor_actualizacion": 1.0494,
  "costo_actualizado_por_accion": 153.21,
  "costo_actualizado_total": 15321.00,
  "precio_venta_neto": 17745.00,
  "ganancia_fiscal": 2424.00,
  "isr": 242.40,
  "ganancia_neta": 2181.60,
  "warnings": []
}
```

**Reglas de validación:**
- Si `inpc` del mes requerido no existe en DB → 422 con código `INPC_NOT_AVAILABLE` indicando qué meses faltan.
- Si suma de `acciones` en lotes != `venta.acciones` → 422 (no soporta venta parcial en este endpoint; usar `services/portfolio.py` para FIFO de portafolio real).

#### POST `/api/v1/transacciones/import-csv`

**Request:** `multipart/form-data` con archivo CSV.

**Esquema CSV esperado (compatible con spreadsheet actual):**
```
fecha,tipo,ticker,acciones,precio,comision,moneda,estrategia,notas
2025-01-15,buy,FUNO11.MX,100,28.50,8.30,MXN,patrimonial,Compra inicial
2025-02-28,dividend,FUNO11.MX,100,0.62,0,MXN,patrimonial,Distribución 1Q
2025-03-10,deposit,,0,5000.00,0,MXN,n/a,
```

**Response:**
```json
{
  "importadas": 47,
  "saltadas": 2,
  "errores": [
    {"linea": 12, "error": "Ticker no encontrado: XYZQ.MX"},
    {"linea": 23, "error": "Fecha inválida: 32/01/2025"}
  ]
}
```

#### POST `/api/v1/agent/resumen-semanal` (V2)

**Request:** `{}` (sin body — usa portafolio actual)

**Response (streaming):** Server-Sent Events con chunks del markdown a medida que el LLM responde. Cliente acumula y renderiza.

**Orquestación interna en `app/services/agent_weekly.py`:**
1. Captura snapshot del portafolio (posiciones, cash, P&L semana).
2. Para cada ticker en portafolio: pull noticias 7 días vía yfinance + cambios fundamentales.
3. Pull macro de la semana (Banxico decision, Fed, inflación, FX).
4. Pull próximos ex-dividend dates.
5. Evalúa alertas y metas.
6. Construye prompt con TODO el contexto (system prompt detallado en `app/llm/prompts/weekly_summary.md`).
7. Llama Claude API con tool use opcional para consultas adicionales.
8. Guarda resultado en tabla `agent_summaries` con `created_at`, `markdown`, `prompt_tokens`, `completion_tokens`, `provider`.

---

## 6. Frontend Architecture

### Pages / Routes

| Route | Componente | Descripción |
|---|---|---|
| `/` | `Dashboard` | KPIs portafolio + alertas disparadas + macro snapshot |
| `/macro` | `Macro` | Tarjetas MX/US/Global + selector de serie con gráfica |
| `/emisora/$ticker` | `EmisoraDetail` | Ficha completa con tabs: overview / fundamentales / financieros / dividendos / notas |
| `/screener` | `Screener` | Sidebar de filtros + tabla de resultados |
| `/comparador` | `Comparador` | Selector multi-ticker + tabla + gráfica normalizada |
| `/portafolio` | `PortafolioLayout` | Tabs: posiciones / transacciones / dividendos / rendimiento / distribución |
| `/watchlist` | `Watchlist` | Tabs por lista + tabla resumen |
| `/valuacion` | `Valuacion` | Tabs: DCF / DDM / Comparables |
| `/fiscal` | `Fiscal` | Tabs: ISR ventas / ISR dividendos / ISR anual |
| `/metas` | `Metas` | Cards de metas activas + creación + simulador |
| `/resumen` | `ResumenSemanal` | V2 — botón "generar" + render markdown + historial |
| `/configuracion` | `Configuracion` | API keys, comisión default GBM, LLM provider, refresh manual |

### Component Hierarchy — Dashboard

```
<AppShell>
  <Sidebar />
  <main>
    <Header /> <!-- valor total + cambio del día siempre visible -->
    <Dashboard>
      <Grid cols={4}>
        <KPICard title="Valor total" value={...} change={...} />
        <KPICard title="Cambio del día" value={...} pct={...} />
        <KPICard title="P&L no realizada" value={...} pct={...} />
        <KPICard title="Dividendos YTD" value={...} />
      </Grid>
      <Grid cols={2}>
        <Card title="Distribución por estrategia">
          <DistributionPie data={...} />
        </Card>
        <Card title="Top 5 posiciones">
          <PositionsTable limit={5} compact />
        </Card>
      </Grid>
      <Card title="Alertas disparadas">
        <AlertList items={...} />
      </Card>
      <Card title="Snapshot macro">
        <MacroSnapshotGrid pais="MX" />
      </Card>
    </Dashboard>
  </main>
</AppShell>
```

### State Management

- **Server state**: TanStack Query con `staleTime` configurado por tipo:
  - Precios en vivo: 30s
  - Fundamentales: 1h
  - Macro: 1h
  - Portafolio (depende de transacciones): infinito hasta invalidación tras mutation
- **Mutations invalidan queries relacionadas**: crear transacción → invalida `['portafolio']`, `['posiciones']`, `['cash']`, `['dividendos']`.
- **UI state local**: useState/useReducer para forms, modals, filtros aún no aplicados.
- **No Redux/Zustand global** — TanStack Query cubre 95% del state; UI state es siempre local.
- **Router state**: TanStack Router type-safe con loaders que prefechan data antes de render.

### Patrones críticos

- **Optimistic updates en transacciones**: al crear una transacción, actualizar cache de `posiciones` y `cash` antes de la confirmación del server.
- **Streaming SSE para agente V2**: hook `useAgentSummary()` lee `EventSource` y acumula markdown chunks.
- **Tablas con virtualización** para listados grandes (`@tanstack/react-virtual`) — relevante en Screener con 500+ resultados.

---

## 7. Design System

### Tema base — Dark first

Aesthetic: **terminal financiera profesional**. Densidad alta, color funcional (verde/rojo de mercados), tipografía monoespaciada para números, animaciones mínimas. **No** glassmorphism, **no** sombras flotantes, **no** gradientes decorativos. La pantalla es para mirar datos durante horas.

### Colors

| Rol | Hex | Uso |
|---|---|---|
| **Background base** | `#0a0e14` | Fondo principal (casi negro, con tinte azulado) |
| **Surface** | `#11161d` | Cards, panels, sidebar |
| **Surface elevated** | `#1a2029` | Modales, dropdowns, hover de filas |
| **Border** | `#1f2933` | Bordes sutiles entre secciones |
| **Border strong** | `#2d3742` | Bordes de inputs, separadores enfáticos |
| **Text primary** | `#e6edf3` | Texto principal |
| **Text secondary** | `#8b949e` | Labels, metadata |
| **Text muted** | `#6e7681` | Hints, placeholders |
| **Accent (primary)** | `#3b82f6` | Botones primarios, links, foco |
| **Success / Up** | `#10b981` | Cambios positivos, ▲ |
| **Danger / Down** | `#ef4444` | Cambios negativos, ▼ |
| **Warning** | `#f59e0b` | Alertas activas, valores en zona gris |
| **Info** | `#06b6d4` | Tips, notas neutrales |
| **Strategy: Patrimonial** | `#8b5cf6` | Etiquetas/distribuciones |
| **Strategy: Indizado** | `#14b8a6` | |
| **Strategy: Trading** | `#f97316` | |

Tema **light** disponible vía toggle (espejo de los colores, mismos tokens semánticos). Defaults a dark.

### Tipografía

| Rol | Fuente | Tamaño | Peso |
|---|---|---|---|
| Headings (h1) | Inter | 24px | 600 |
| Headings (h2) | Inter | 18px | 600 |
| Headings (h3) | Inter | 16px | 600 |
| Body | Inter | 14px | 400 |
| Body small | Inter | 13px | 400 |
| Caption | Inter | 12px | 500 (uppercase, tracking 0.5px) para labels |
| **Números / precios** | **JetBrains Mono** | 14px | 500 |
| **Números grandes (KPI)** | JetBrains Mono | 28px | 600 |
| Tickers | JetBrains Mono | 14px | 600 |
| Código (rara vez) | JetBrains Mono | 13px | 400 |

> **Razón JetBrains Mono para números**: alineación perfecta de columnas en tablas, distinción visual instantánea entre digit 0 y letra O.

### Spacing & Layout

- Spacing scale: 4px base — `4, 8, 12, 16, 20, 24, 32, 40, 48, 64`.
- Border radius: 6px default (más cuadrado que el típico shadcn — feel terminal), 8px cards, 4px badges, `9999px` solo para avatares.
- Max content width: sin límite (full viewport). Sidebar 240px fija, header 56px.
- Densidad de tabla: row height 36px (compacta), padding celda 8px 12px.
- Breakpoints: `sm 640`, `md 768`, `lg 1024`, `xl 1280`, `2xl 1536`. La app está optimizada para `xl+` (laptop/desktop). Mobile no es prioridad pero no debe romperse.

### Componente — Estilo

- **Botones**: planos, sin sombra, hover oscurece 8%, focus ring de 2px en accent.
- **Inputs**: fondo igual a surface, border 1px, focus border-color accent.
- **Cards**: surface + border 1px, sin sombra.
- **Tabs**: underline minimalista (no pills).
- **Modales**: backdrop blur opcional 4px + surface-elevated.
- **Toasts**: arriba a la derecha, surface-elevated, border accent según tipo.

### Charts (Plotly theme)

`plotlyTheme.ts` exporta layout base reutilizable:
- `paper_bgcolor`: transparente
- `plot_bgcolor`: `#0a0e14`
- `font`: `{family: 'Inter', size: 12, color: '#e6edf3'}`
- Grid: `#1f2933`, axis line `#2d3742`
- Default colorway: `['#3b82f6', '#10b981', '#f59e0b', '#8b5cf6', '#06b6d4', '#ef4444', '#14b8a6', '#f97316']`
- Hover: `{bgcolor: '#1a2029', bordercolor: '#3b82f6'}`
- Sin barras de herramientas Plotly (toolbar oculto), salvo download que sí se conserva.

### Convenciones de formato

- **MXN**: `$1,234,567.89 MXN` (símbolo `$`, separador miles `,`, decimal `.`, sufijo moneda).
- **USD**: `US$1,234.56`.
- **Porcentajes**: `+2.45%` o `-1.32%` con signo explícito y color.
- **Fechas**: `12/05/2026` (DD/MM/AAAA, formato MX). Para timestamps: `12/05/2026 14:32`.
- **Cambios**: `▲ $12.45 (+0.85%)` o `▼ -$8.10 (-0.62%)` — flecha unicode + monto + pct + color.
- **Compactación**: `$1.2M`, `$45.3K` para KPIs cuando aplique.

---

## 8. Authentication & Authorization

**No aplica.** App single-user, bind a `127.0.0.1` únicamente. Docker compose expone puerto solo a `127.0.0.1:8301`.

**Futuro:** Si el usuario decide exponer la app vía Cloudflare Tunnel (acceso desde laptop fuera de casa):
1. Agregar `BASIC_AUTH_USER` + `BASIC_AUTH_PASSWORD` a `.env.production`.
2. Middleware FastAPI `HTTPBasic` en `app/middleware/auth.py`.
3. Frontend muestra prompt de browser nativo.
4. **No** implementar antes de que sea necesario.

---

## 9. Build Order

Construcción incremental en 7 fases del PRD, cada una con pasos numerados ejecutables. **Cada paso debe terminar con un commit verificable.**

### Fase 1 — Cimientos (3-5 días)

**Step 1: Scaffolding del repo**
- Crear `terminal-financiero/` en `E:\ClaudeWorks\proyectos\PersonalBeto\Portafolio\`.
- `git init`, agregar remoto `huheme25/terminal-financiero` (crear repo en GitHub).
- Crear `.gitignore` (Python + Node + Docker + .env).
- Escribir `README.md` skeleton y `CLAUDE.md` (copiar de sección 15 de este blueprint).
- Crear `docker-compose.yml`, `docker-compose.dev.yml`, `.env.example`.
- Commit: `chore: scaffolding inicial`.

**Step 2: Backend skeleton**
- `cd backend && uv init`, agregar deps base: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `httpx`, `pydantic-settings`, `redis[hiredis]`, `apscheduler`, `python-multipart`.
- Crear `app/main.py` con healthcheck `/api/v1/health`.
- `app/config.py` con Pydantic Settings (lee `.env`).
- `app/db.py` con AsyncEngine.
- `Dockerfile` (python:3.12-slim + uv sync).
- Verificar: `docker compose up backend` → `curl http://localhost:8000/api/v1/health` retorna `{"status":"ok"}`.
- Commit: `feat(backend): skeleton FastAPI con healthcheck`.

**Step 3: Frontend skeleton**
- `cd frontend && pnpm create vite . --template react-ts`.
- Agregar Tailwind v4 (`@tailwindcss/vite` plugin), shadcn/ui (`pnpm dlx shadcn@latest init`), TanStack Router, TanStack Query, react-plotly.js, lucide-react.
- `tsconfig.json` con `strict: true`, path alias `@/*` → `src/*`.
- Crear `App.tsx` con RouterProvider, layout básico `AppShell` con sidebar placeholder.
- `Dockerfile` multi-stage: build con node → serve con nginx.
- `nginx.conf` con SPA fallback + proxy `/api → http://backend:8000`.
- Verificar: `docker compose up frontend` → `http://localhost:8301` muestra "Terminal Financiero" en dark mode.
- Commit: `feat(frontend): skeleton React + Tailwind + shadcn + routing`.

**Step 4: Modelo de datos base**
- `app/models/base.py` con `DeclarativeBase` y `TimestampMixin`.
- Modelar TODAS las entidades de la sección 4 en `app/models/*.py`.
- `alembic init alembic`, configurar `env.py` para usar AsyncEngine + import de modelos.
- `alembic revision --autogenerate -m "initial schema"`.
- Revisar la migración generada: agregar **manualmente** particiones de `precios_historicos`, índices `gin_trgm_ops` y `CHECK` constraints complejos (Alembic no los autogenera bien).
- `alembic upgrade head`.
- Commit: `feat(db): schema inicial con particiones y vistas`.

**Step 5: Conectores externos con cache**
- `app/connectors/base.py` con `CachedClient` clase abstracta: TTL configurable, dedupe de requests concurrentes, budget tracker para FMP.
- Implementar `yfinance_client.py`: wrappers async sobre `yfinance` (que es sync — usar `asyncio.to_thread`).
- Implementar `banxico_client.py` y `fred_client.py`: httpx + parseo + cache Redis.
- Implementar `inegi_client.py` y `fmp_client.py`.
- `app/connectors/sat_client.py`: lookup local en `data/seed/isr_tarifa_2026.json` (no es API).
- Tests unitarios mockeando httpx con `pytest-httpx`.
- Commit: `feat(connectors): yfinance, banxico, fred, inegi, fmp con cache Redis`.

**Step 6: Seed inicial del universo de emisoras**
- Escribir `scripts/seed_emisoras.py` que carga `data/seed/emisoras_bmv.json` + `fibras_mx.json` + `sp500.json` + `inpc_historico.json` (datos hasta abril 2026 mínimo).
- Comando: `docker compose exec backend python scripts/seed_emisoras.py`.
- Verificar: `SELECT count(*) FROM emisoras WHERE bolsa='BMV'` >= 100.
- Commit: `feat(data): seed inicial de universo y INPC histórico`.

**Step 7: Jobs periódicos básicos**
- `app/jobs/scheduler.py` arranca APScheduler con jobstore en memoria.
- Job `refresh_macro.py`: corre diario 08:00 MX, descarga macro de Banxico/FRED, upsert en `macro_valores`.
- Job `refresh_inpc.py`: corre día 10 de cada mes, descarga INPC último mes publicado por INEGI.
- Lifecycle: arrancar/parar scheduler en `app.on_event("startup")/("shutdown")`.
- Verificar: invocar job manualmente vía endpoint debug → datos en DB.
- Commit: `feat(jobs): refresh diario de macro y mensual de INPC`.

### Fase 2 — Portafolio core (5-7 días)

**Step 8: Endpoints CRUD de transacciones**
- `app/routers/transacciones.py` con GET (listado paginado), POST, PATCH, DELETE.
- Validación Pydantic estricta: total se calcula en server, no se confía en el cliente.
- Marcar `es_fibra=true` automáticamente al insertar dividend si `emisoras.tipo_instrumento='fibra'`.
- Tests integración con DB de prueba.
- Commit: `feat(transacciones): CRUD completo con validación server-side`.

**Step 9: Importador CSV**
- `scripts/import_csv.py` (CLI standalone).
- `POST /api/v1/transacciones/import-csv` con upload multipart.
- Parsea con pandas, valida fila por fila, retorna reporte de errores.
- Soporta formato del spreadsheet actual (columnas en español).
- Commit: `feat(transacciones): importador CSV con reporte de errores`.

**Step 10: Servicio de portafolio (posiciones, cash, P&L)**
- `app/services/portfolio.py`:
  - `get_posiciones()` — query `v_posiciones` + JOIN último precio + calcula P&L unrealized.
  - `get_cash_balance(moneda)`.
  - `get_dashboard_summary()` — agrega todo para vista de overview.
  - `calcular_costo_promedio_ponderado(ticker)` — pandas con lotes vigentes.
  - `calcular_pnl_realizado()` — recorre sells, matchea con compras (default promedio ponderado; opción FIFO en config).
- Endpoints `/api/v1/portafolio/*` que consumen el servicio.
- Tests con casos canónicos.
- Commit: `feat(portfolio): cálculo de posiciones, cash, P&L realizado y unrealized`.

**Step 11: UI — Página de portafolio (Tab Posiciones + Transacciones)**
- Skill: `/shadcn-ui` para agregar `table`, `dialog`, `form`, `select`, `input`, `tabs`, `card`.
- Página `routes/portafolio/index.tsx` con tabs.
- `PositionsTable.tsx` con TanStack Table: columnas Ticker, Acciones, Precio Prom, Precio Actual, Costo Base, Valor Actual, P&L $, P&L %, Estrategia. Click en ticker → navega a `/emisora/$ticker`.
- `TransactionsTable.tsx` con TanStack Table + filtros (tipo, ticker, rango fechas).
- `TransactionForm.tsx` (Dialog con React Hook Form + Zod) para crear/editar.
- Botón "Importar CSV" → file picker → POST → toast con resultado.
- Verificar: importar CSV real del usuario, ver posiciones correctas.
- Commit: `feat(ui): página portafolio con posiciones y transacciones`.

**Step 12: Dashboard del portafolio (Tab Distribución + Rendimiento + KPIs)**
- Skill: `/frontend-design` para layout dashboard.
- `routes/index.tsx` (Dashboard general) con KPICards + DistributionPie + AlertList placeholder.
- `routes/portafolio/distribucion.tsx` con pies (estrategia, sector, país, moneda, tipo).
- `routes/portafolio/rendimiento.tsx` con line chart TWR + comparativa benchmark (^MXX, ^GSPC, NAFTRAC).
- Servicio `portfolio.calcular_twr_mwr(from, to)` — pandas con cash flow weighting.
- Commit: `feat(ui): dashboard de portafolio con distribución y rendimiento`.

### Fase 3 — Datos y exploración (4-5 días)

**Step 13: Macro Dashboard**
- `app/routers/macro.py` con catálogo + valores + dashboard snapshot.
- `routes/macro.tsx` con grid de tarjetas (CETES 28, TIIE 28, USD/MXN, Inflación, etc.) clickeables → modal con gráfica histórica.
- `MacroChart.tsx` (Plotly line) con selector de rango (`1M, 3M, 6M, 1A, 3A, 5A, MAX`).
- Vista comparativa: selector multi-serie en la misma gráfica (ej. CETES 28 vs Inflación YoY).
- Commit: `feat(macro): dashboard con tarjetas y gráficas comparativas`.

**Step 14: Explorador de Emisoras — Overview + Precios**
- `app/routers/emisoras.py` con search, detail, precios.
- `routes/emisora.$ticker.tsx` con tabs (Overview / Fundamentales / Financieros / Dividendos / Notas).
- Tab Overview: KPIs (precio, market cap, sector, P/E, dividend yield) + `PriceChart.tsx` (Plotly candlestick o line + volumen).
- Botón "Agregar a watchlist" + "Agregar transacción".
- Skill: `/frontend-design` para ficha visual atractiva.
- Commit: `feat(emisoras): ficha con overview y gráfica de precios`.

**Step 15: Explorador — Fundamentales + Estados Financieros + Dividendos**
- Tab Fundamentales: grid con todos los ratios (valuación / rentabilidad / dividendos / deuda / crecimiento).
- Tab Financieros: tablas income / balance / cashflow con toggle anual/trimestral, columnas son períodos, filas son line items, badges de variación %.
- Tab Dividendos: tabla histórica + bar chart anual + indicador FIBRA si aplica.
- Tab Notas: textarea persistente (autosave debounced).
- Commit: `feat(emisoras): fundamentales, estados financieros, dividendos, notas`.

### Fase 4 — Análisis (3-4 días)

**Step 16: Screener**
- `app/services/screener.py` que aplica filtros JSON sobre emisoras + último snapshot de fundamentales.
- `routes/screener.tsx` con sidebar de filtros (form con todos los criterios del PRD) + tabla resultados.
- Columnas configurables (dropdown toggle).
- Filtros guardados: CRUD + cargar rápido.
- Skill: `/shadcn-ui` para command palette de "cargar filtro guardado".
- Commit: `feat(screener): filtros + tabla resultados + filtros guardados`.

**Step 17: Comparador**
- `routes/comparador.tsx` con selector multi-ticker (combobox con search).
- Tabla comparativa con highlight mejor/peor por fila (verde mejor, rojo peor).
- `ComparisonChart.tsx`: serie de precios normalizada a 100 desde fecha seleccionada.
- Tab adicional: comparativa dividend yield histórico + crecimiento DPA.
- Commit: `feat(comparador): tabla y gráfica normalizada de hasta 5 emisoras`.

### Fase 5 — Dividendos y Metas (4-5 días)

**Step 18: Tab Dividendos del portafolio**
- `app/services/dividendos.py`: agrupación por mes/quarter/año, separación por FIBRA, por estrategia.
- `routes/portafolio/dividendos.tsx`:
  - Tabla de dividendos recibidos.
  - Bar chart mensual de ingresos.
  - Pie por estrategia.
  - Calendario próximas fechas ex-dividend y de pago (de holdings actuales).
- Commit: `feat(dividendos): historial agrupado + calendario próximos`.

**Step 19: Sistema de metas**
- `app/services/goals.py`: calcula progreso por tipo de meta.
- `app/routers/metas.py` CRUD + endpoint `/simular` (POST con ticker+acciones → delta de progreso).
- `routes/metas.tsx` con cards de metas activas + form de creación + simulador inline.
- `GoalProgress.tsx` componente con barra + marcador de proyección.
- Logros: badge "✓ Cumplida" + fecha.
- Commit: `feat(metas): sistema de metas con simulador y proyección`.

### Fase 6 — Valuación, Fiscal, Watchlist (5-6 días)

**Step 20: Calculadora ISR**
- `app/services/fiscal.py` — implementar:
  - `calcular_isr_venta(lotes, venta)` con factor INPC (mes compra anterior vs mes venta anterior).
  - `calcular_isr_dividendo_no_cufin(monto)` con piramidación 1.4286 + ISR corporativo 30% + ISR adicional 10%.
  - `calcular_isr_dividendo_cufin(monto)` = monto * 0.10.
  - `calcular_isr_anual(ingresos_acumulables)` con tarifa progresiva Art. 152.
- Tests con casos canónicos publicados por contadores (validar con ejemplos del SAT).
- `routes/fiscal.tsx` con tabs y forms.
- Skill: `/frontend-design` para presentación clara de cálculo paso a paso.
- Commit: `feat(fiscal): calculadora ISR ventas, dividendos y anual con ajuste INPC`.

**Step 21: Watchlist + Alertas**
- `app/routers/watchlist.py` y `alertas.py` CRUD.
- `app/services/alerts.py` evalúa todas las alertas activas.
- Job `evaluate_alerts.py` cada 30 min cuando la app está corriendo (configurable).
- `routes/watchlist.tsx` con tabs por lista + tabla con métricas en vivo + nota rápida.
- `AlertForm.tsx` para definir condiciones.
- Dashboard general: card `AlertList` mostrando alertas disparadas.
- Commit: `feat(watchlist): listas + alertas evaluadas periódicamente`.

**Step 22: Herramientas de Valuación**
- `app/services/valuacion.py`:
  - `dcf(fcf, growth_rates, terminal_growth, wacc, shares)` → intrinsic value + tabla sensibilidad.
  - `ddm(dividend, growth, required_return)` → intrinsic value.
  - `comparables(ticker, peers)` → tabla múltiplos + mediana + precio implícito.
- `routes/valuacion.tsx` con tabs.
- Botón "Cargar desde emisora" pre-llena inputs con datos de fundamentales.
- Commit: `feat(valuacion): DCF con sensibilidad, DDM, comparables`.

### Fase 7 — V2 — Agente IA (5-7 días)

**Step 23: Cliente LLM con adapter**
- `app/llm/base.py` interface `LLMClient` con `complete(messages, tools=None) -> stream`.
- `app/llm/anthropic_client.py` (Claude API).
- `app/llm/ollama_client.py` (HTTP local).
- Config en `.env`: `LLM_PROVIDER=anthropic|ollama`, `LLM_MODEL=claude-sonnet-4-6`, `ANTHROPIC_API_KEY=...`.
- Commit: `feat(llm): cliente con adapter Anthropic + Ollama`.

**Step 24: Agente de resumen semanal**
- Prompt en `app/llm/prompts/weekly_summary.md` (system prompt detallado: rol de analista financiero senior, contexto del usuario, formato esperado).
- `app/services/agent_weekly.py` orquesta: snapshot portafolio → contexto market → contexto macro → próximos dividendos → alertas/metas → prompt → LLM call (streaming).
- Tool calls opcionales: `lookup_emisora`, `lookup_macro` para que el agente profundice si necesita.
- Persistir resultado en tabla `agent_summaries` (nueva: agregar en migración).
- Endpoint `POST /api/v1/agent/resumen-semanal` retorna SSE.
- Endpoint `GET /api/v1/agent/resumenes` listado paginado.
- Commit: `feat(agent): resumen semanal con orquestación de contexto`.

**Step 25: UI del resumen semanal + export PDF**
- `routes/resumen.tsx`: botón "Generar resumen ahora" + hook `useAgentSummary()` con `EventSource`.
- Render del markdown con `react-markdown` + Tailwind typography.
- Historial paginado de resúmenes anteriores.
- Botón "Exportar a PDF" → `GET /api/v1/agent/resumenes/{id}/pdf` (WeasyPrint server-side).
- Skill: `/pdf-design` para template HTML/CSS del PDF.
- Commit: `feat(ui): UI agente semanal con streaming + export PDF`.

### Cierre

**Step 26: Página de configuración**
- `routes/configuracion.tsx`: forms para API keys (FMP, Anthropic), comisión default GBM, LLM provider/model, refresh manual (botones que invocan jobs).
- Persistencia en tabla `app_config` (key-value) o solo `.env` (más simple — usuario edita y reinicia).
- Commit: `feat(config): página de configuración`.

**Step 27: Polish + verificación**
- Loading states, empty states, error boundaries.
- Skill: `/ui-ux-pro-max` revisión final de visual consistency.
- Verificar: importar CSV real, navegar todas las páginas, comparar ISR calculado contra constancia GBM real, generar resumen semanal V2.
- README con setup completo desde cero.
- Commit: `chore: polish y documentación final`.

---

## 10. Environment Setup

### Prerequisites

- **Docker Desktop for Windows** (con WSL2 backend), versión 4.30+.
- **Git** instalado.
- **Node.js 24** local (opcional, solo para correr frontend sin Docker en dev).
- **Python 3.12** + **uv** local (opcional, igual razón).
- API keys (todas gratuitas):
  - **Banxico SIE**: https://www.banxico.org.mx/SieAPIRest/service/v1/token
  - **FRED**: https://fred.stlouisfed.org/docs/api/api_key.html
  - **INEGI BIE**: https://www.inegi.org.mx/servicios/api_indicadores.html
  - **Financial Modeling Prep**: https://site.financialmodelingprep.com/developer/docs (free tier 250/día)
  - **Anthropic** (V2): https://console.anthropic.com/ (Sonnet 4.6)

### Environment Variables

| Variable | Descripción | Default | Dónde obtener |
|---|---|---|---|
| `POSTGRES_PASSWORD` | Password Postgres | (generar) | `python -c "import secrets;print(secrets.token_hex(24))"` |
| `BANXICO_TOKEN` | Token Banxico SIE | — | URL Banxico arriba |
| `FRED_API_KEY` | API key FRED | — | URL FRED arriba |
| `INEGI_TOKEN` | Token INEGI BIE | — | URL INEGI arriba |
| `FMP_API_KEY` | API key FMP | — | URL FMP arriba |
| `COMISION_DEFAULT_PCT` | Comisión GBM default | `0.0029` | — |
| `INCLUIR_IVA_COMISION` | Incluir IVA 16% sobre comisión | `false` | — |
| `LLM_PROVIDER` | `anthropic` o `ollama` | `anthropic` | — |
| `LLM_MODEL` | Modelo | `claude-sonnet-4-6` | — |
| `ANTHROPIC_API_KEY` | API key Claude | — | https://console.anthropic.com |
| `OLLAMA_BASE_URL` | URL Ollama local | `http://host.docker.internal:11434` | — |
| `LOG_LEVEL` | Nivel logging | `INFO` | — |
| `TZ` | Timezone | `America/Mexico_City` | — |

### Initial Setup Commands

```powershell
# 1. Clonar el repo
git clone https://github.com/huheme25/terminal-financiero.git E:\ClaudeWorks\proyectos\PersonalBeto\Portafolio\terminal-financiero
cd E:\ClaudeWorks\proyectos\PersonalBeto\Portafolio\terminal-financiero

# 2. Generar .env desde .env.example
Copy-Item .env.example .env
# Editar .env y rellenar tokens

# 3. Levantar todo
docker compose up -d --build

# 4. Correr migraciones
docker compose exec backend alembic upgrade head

# 5. Seed inicial de universo
docker compose exec backend python scripts/seed_emisoras.py

# 6. (Opcional) Importar transacciones desde CSV existente
docker compose exec backend python scripts/import_csv.py /data/mi-portafolio.csv

# 7. Abrir
Start-Process http://localhost:8301
```

### Modo desarrollo (hot-reload)

```powershell
# Backend con --reload
docker compose -f docker-compose.yml -f docker-compose.dev.yml up backend

# Frontend con Vite dev server (fuera de Docker, mejor DX)
cd frontend
pnpm install
pnpm dev    # http://localhost:5173, proxy a backend:8000
```

---

## 11. Dependencies

### Backend — Core

| Package | Purpose |
|---|---|
| `fastapi[standard]` | Framework web async |
| `uvicorn[standard]` | ASGI server |
| `sqlalchemy[asyncio]` | ORM async |
| `asyncpg` | Postgres driver async |
| `alembic` | Migraciones |
| `pydantic-settings` | Config desde env |
| `httpx` | HTTP client async |
| `redis[hiredis]` | Cliente Redis |
| `apscheduler` | Jobs periódicos |
| `yfinance` | Datos mercado |
| `fredapi` | FRED |
| `pandas` | Análisis (TWR, MWR, agregaciones) |
| `numpy` | Matemáticas |
| `python-multipart` | Upload de archivos |
| `anthropic` | Cliente Claude (V2) |
| `weasyprint` | Export PDF (V2) |
| `markdown` | Render markdown a HTML para PDF (V2) |
| `python-dateutil` | Parsing fechas robusto |

### Backend — Dev

| Package | Purpose |
|---|---|
| `pytest` + `pytest-asyncio` | Testing async |
| `pytest-httpx` | Mock httpx |
| `testcontainers[postgres]` | Postgres real en tests integración |
| `ruff` | Linter + formatter |
| `mypy` | Type checking |

### Frontend — Core

| Package | Purpose |
|---|---|
| `react`, `react-dom` | v19 |
| `@tanstack/react-router` | Routing type-safe |
| `@tanstack/react-query` | State server |
| `@tanstack/react-table` | Tablas |
| `@tanstack/react-virtual` | Virtualización |
| `react-hook-form` + `@hookform/resolvers` | Forms |
| `zod` | Validación |
| `plotly.js-dist-min` + `react-plotly.js` | Charts |
| `lucide-react` | Iconos |
| `clsx` + `tailwind-merge` | utilidad `cn()` |
| `date-fns` | Manejo fechas |
| `react-markdown` + `remark-gfm` | Render markdown V2 |
| `sonner` | Toasts |
| shadcn/ui primitives | (copy-paste, no son deps) |

### Frontend — Dev

| Package | Purpose |
|---|---|
| `vite` | Bundler |
| `@vitejs/plugin-react` | |
| `@tailwindcss/vite` | Tailwind v4 plugin |
| `typescript` | |
| `@types/react`, `@types/react-dom`, `@types/plotly.js` | |
| `eslint` + plugins | |
| `vitest` + `@testing-library/react` | Tests |

---

## 12. Deployment Strategy

### Hosting

**localhost en la PC del usuario.** Docker Compose levanta 4 servicios en la red interna `terminal-net`:

- `db` (postgres:16-alpine) — solo accesible dentro de la red Docker.
- `redis` (redis:7-alpine) — solo accesible dentro de la red Docker.
- `backend` (custom image) — expone `:8000` en la red interna.
- `frontend` (nginx con build de React) — expone `:80` en la red interna y publica en `127.0.0.1:8301:80` en el host.

Solo el frontend publica al host, y bind explícito a `127.0.0.1` (no `0.0.0.0`) para evitar exposición LAN.

### CI/CD

- **Sin CI** (proyecto personal, single-user). Cambios → `pnpm/uv test` local → commit → `docker compose up -d --build`.
- **Opcional futuro**: GitHub Actions con tests de PR (lint + unit), pero no deploy automatizado.

### Domain & DNS

- N/A. Acceso por `http://localhost:8301`.
- **Si en el futuro el usuario quiere acceder desde la laptop fuera de casa**: agregar `cloudflared` service al compose con tunnel a `terminal.beto.dev` o similar (decisión del usuario), + basic auth.

### Environments

- **dev**: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up` — backend con `--reload`, frontend con Vite dev server (fuera de Docker).
- **prod**: `docker compose up -d` — frontend buildeado servido por nginx, backend sin reload.
- **No** hay staging.

### Backups

- Postgres: script PowerShell `scripts/backup_db.ps1` con `pg_dump` → `F:\Respaldos\terminal-financiero\YYYY-MM-DD.sql.gz`.
- Programar en Windows Task Scheduler diario 23:00.
- Retención: últimos 30 días.

---

## 13. Testing Strategy

### Unit Tests (backend)

**Cobertura crítica:**
- `app/services/fiscal.py` — ISR con casos canónicos publicados (SAT, contadores). Mínimo 15 casos por escenario (CUFIN/no-CUFIN/FIBRA/venta con factor INPC).
- `app/services/portfolio.py` — TWR/MWR con cash flows conocidos, costo promedio ponderado con lotes múltiples, ganancia realizada con sells parciales.
- `app/services/valuacion.py` — DCF/DDM con valores y resultados verificados a mano.
- `app/utils/inpc_factor.py` — cálculo factor de actualización.

**Framework**: pytest + pytest-asyncio. Sin DB para unit tests (servicios reciben datos parametrizados).

### Integration Tests (backend)

- `tests/integration/test_api_*.py` con `testcontainers[postgres]`.
- Cubrir flujo: crear transacciones → consultar posiciones → verificar cálculos.
- Cubrir importación CSV con archivos de prueba.
- Mock de connectors externos (yfinance, banxico) — no llamar APIs reales en CI.

### E2E (futuro / opcional)

- Si la complejidad crece, agregar Playwright con flujos clave:
  1. Crear transacción → ver en posiciones.
  2. Aplicar filtro screener → verificar resultados.
  3. Calcular ISR → ver resultado correcto.
- **No en V1.** El proyecto es personal y los tests integración + uso manual son suficientes.

### Manual verification al final de cada fase

Antes de cerrar una fase, el builder debe demostrar que funciona end-to-end con datos reales:
- Fase 2: importar CSV del usuario, ver posiciones correctas que cuadren con su spreadsheet.
- Fase 6: calcular ISR de una venta real del año pasado, validar que coincida con su declaración.
- Fase 7: generar un resumen semanal y revisar coherencia.

---

## 14. Skills to Use During Build

| Skill | When to Use | Why |
|---|---|---|
| `/shadcn-ui` | Steps 3, 11, 16 (cada vez que se agregan componentes UI) | Setup inicial + comandos `pnpm dlx shadcn add ...` para primitives |
| `/frontend-design` | Steps 12, 14, 20 (dashboard, ficha emisora, calculadora fiscal) | UI distintiva y densidad correcta para terminal financiera |
| `/ui-ux-pro-max` | Step 27 (polish final) | Revisión de consistencia visual y micro-interacciones |
| `/pdf-design` | Step 25 (export PDF V2) | Template HTML/CSS profesional para WeasyPrint |
| `/deep-research` | Cuando dudes sobre regla fiscal MX o API de un connector | Investigación cuidadosa antes de implementar lógica fiscal sensible |
| `/playwright-cli` | Opcional, step E2E si se agrega | Browser automation para tests |

**Skills que NO se usan en este blueprint:**
- `/seo-audit` — app local sin SEO.
- `/humanizer` — no hay copy marketing.
- `/chrome-bridge-automation` — no se analizan reference sites.

---

## 15. CLAUDE.md for Target Project

```markdown
# Terminal Financiero Personal

App local de análisis financiero para inversionista MX que opera vía GBM. Backend FastAPI + Postgres + Redis. Frontend React + Vite + Tailwind + shadcn/ui. Charts Plotly. Docker Compose. Single-user, localhost-only, sin auth.

## Comandos

- `docker compose up -d` — Levantar producción local
- `docker compose -f docker-compose.yml -f docker-compose.dev.yml up backend` — Backend dev con reload
- `cd frontend && pnpm dev` — Frontend dev server (Vite, http://localhost:5173 con proxy a backend)
- `docker compose exec backend alembic upgrade head` — Aplicar migraciones
- `docker compose exec backend alembic revision --autogenerate -m "..."` — Generar migración
- `docker compose exec backend pytest` — Tests backend
- `cd frontend && pnpm test` — Tests frontend
- `cd frontend && pnpm lint` — Linter frontend
- `docker compose exec backend ruff check .` — Linter backend
- `docker compose exec backend python scripts/seed_emisoras.py` — Re-seed universo
- `docker compose exec backend python scripts/import_csv.py /data/archivo.csv` — Importar transacciones

## Tech Stack

FastAPI + Python 3.12 (uv) + SQLAlchemy 2.0 async + PostgreSQL 16 + Redis 7 + React 19 + Vite 6 + TypeScript + Tailwind v4 + shadcn/ui + TanStack Router/Query/Table + Plotly.js + Docker Compose

## Architecture

### Directory Structure
- `backend/app/connectors/` — Wrappers async sobre APIs externas (yfinance, Banxico, FRED, INEGI, FMP). Toda llamada externa va por acá.
- `backend/app/services/` — Lógica de negocio. Nunca llamar connectors desde routers; siempre vía service.
- `backend/app/models/` — SQLAlchemy ORM. Una clase por tabla.
- `backend/app/schemas/` — Pydantic v2 request/response. Validación estricta.
- `backend/app/routers/` — Endpoints FastAPI. Thin layer: valida, llama service, retorna schema.
- `backend/app/jobs/` — Tasks APScheduler. Refresh periódico de datos externos.
- `backend/app/llm/` — Cliente LLM con adapter (Anthropic/Ollama).
- `frontend/src/routes/` — File-based routing TanStack Router. Una página por archivo.
- `frontend/src/components/` — UI dividido en `ui/` (shadcn primitives), `layout/`, `charts/`, `tables/`, `forms/`, `shared/`.
- `frontend/src/hooks/` — useQuery wrappers tipados por dominio.
- `frontend/src/lib/` — api client, formatters MXN, utils.

### Data Flow
Frontend (React Query) → `/api/v1/*` (FastAPI router) → Service (lógica) → [Connector con cache Redis | SQLAlchemy session]. Mutations en frontend invalidan queries relacionadas. Para series temporales largas, el cliente pasa `range` y el server decide TTL.

### Key Patterns
- **Connectors siempre cachean**. Nunca llamar APIs externas sin pasar por `CachedClient`.
- **Total de transacciones se calcula server-side**. No confiar en el cliente.
- **Posiciones y cash son vistas calculadas**. No tablas. Refrescar es siempre correcto.
- **Cálculos fiscales en pandas, no SQL**. Casos edge dominan; mejor en código testeable.
- **Auto-marcar `es_fibra=true`** al insertar dividend si la emisora es FIBRA.
- **Server Components**: N/A (Vite SPA, no Next.js).
- **Tipos compartidos**: el frontend genera tipos desde el OpenAPI del backend con `openapi-typescript` (script `pnpm types:generate`).

## Code Organization Rules

1. **Una clase/función exportada por archivo** en módulos sensibles (services, models, components principales). Máx 300 líneas por archivo.
2. **Path alias**: backend usa imports absolutos `from app.services.portfolio import ...`; frontend usa `@/` para `src/`.
3. **No barrel exports**: importar directo del archivo (mejor para tree-shaking y navegación).
4. **Idioma**: comentarios y variables descriptivas en español cuando ameriten. Identificadores técnicos pueden quedar en inglés (`get_posiciones`, `calcular_isr_venta`). Mensajes de error al usuario siempre en español.
5. **Validación**: Zod en frontend forms, Pydantic en backend endpoints. Misma estructura espejada.
6. **Cero `any` en TypeScript**, cero `# type: ignore` en Python sin justificación en comentario.
7. **Tests son obligatorios para `services/fiscal.py`, `services/portfolio.py`, `services/valuacion.py`, `utils/inpc_factor.py`**.
8. **Commits atómicos**: un commit = un cambio coherente con mensaje en español siguiendo convención `tipo(scope): descripción`.

## Design System

### Colors (dark-first)
- Background: `#0a0e14` / Surface: `#11161d` / Surface elevated: `#1a2029`
- Border: `#1f2933` / Border strong: `#2d3742`
- Text: `#e6edf3` / Text secondary: `#8b949e` / Muted: `#6e7681`
- Accent: `#3b82f6` (blue) / Success/Up: `#10b981` / Danger/Down: `#ef4444` / Warning: `#f59e0b` / Info: `#06b6d4`
- Strategies: Patrimonial `#8b5cf6` / Indizado `#14b8a6` / Trading `#f97316`

### Typography
- Inter para UI (headings 600, body 400, caption 500 uppercase tracking 0.5px)
- **JetBrains Mono para TODOS los números, precios y tickers** (alineación de columnas + distinción 0/O)

### Style
- Border radius: 6px default, 8px cards, 4px badges
- Density: row height tabla 36px, padding celda 8px 12px
- Spacing scale: 4, 8, 12, 16, 20, 24, 32, 40, 48, 64
- Plano: sin sombras decorativas, sin gradientes, sin glassmorphism
- Charts Plotly con tema dark centralizado en `frontend/src/components/charts/plotlyTheme.ts`

### Formato
- MXN: `$1,234,567.89 MXN`. USD: `US$1,234.56`.
- Fechas: `12/05/2026` (DD/MM/AAAA).
- Cambios: `▲ $12.45 (+0.85%)` verde / `▼ -$8.10 (-0.62%)` rojo.

## Environment Variables

| Variable | Descripción |
|---|---|
| `POSTGRES_PASSWORD` | Password Postgres |
| `BANXICO_TOKEN` | Token Banxico SIE |
| `FRED_API_KEY` | API key FRED |
| `INEGI_TOKEN` | Token INEGI BIE |
| `FMP_API_KEY` | API key Financial Modeling Prep |
| `COMISION_DEFAULT_PCT` | Comisión GBM default (0.0029) |
| `INCLUIR_IVA_COMISION` | Bool, default false |
| `LLM_PROVIDER` | `anthropic` o `ollama` |
| `LLM_MODEL` | `claude-sonnet-4-6` por default |
| `ANTHROPIC_API_KEY` | Claude API |
| `OLLAMA_BASE_URL` | `http://host.docker.internal:11434` |
| `TZ` | `America/Mexico_City` |
| `LOG_LEVEL` | `INFO` |

## Reglas No Negociables

1. **Localhost-only**. El frontend publica solo en `127.0.0.1:8301`. Nunca cambiar a `0.0.0.0` sin agregar auth primero.
2. **Postgres en Windows = 127.0.0.1** si alguna vez se conecta fuera del compose. Reglar global del usuario.
3. **Encoding ASCII en archivos .ps1** si los invoca powershell.exe.
4. **Nunca commitear `.env`, `.env.local`, ni archivos con tokens**.
5. **Toda llamada a API externa pasa por `app/connectors/` con cache Redis**. Cero fetches ad hoc desde routers o services.
6. **Cálculos fiscales (`services/fiscal.py`) requieren test unitario antes de mergear**. No es código que se "verifica en QA".
7. **Comentarios y variables descriptivas en español cuando ameriten** (no sobre-documentar; código legible primero).
8. **Sin features de "trading real"** (órdenes a GBM). Esto es una herramienta de análisis, no un broker. Si surge la idea, rechazar.
9. **Sin recolección de telemetría / analytics**. App personal local.
10. **Cambios al schema requieren migración Alembic**, jamás `metadata.create_all()` en código de aplicación.
```

---

## 16. Reglas No Negociables

1. **No publicar puertos al host fuera de `127.0.0.1`**. La app es localhost-only por diseño. Cambiar esto requiere agregar auth primero.
2. **TypeScript en strict mode, sin `any`**. Cero excepciones.
3. **Cálculos financieros en pandas, no SQL**. Lotes, FIFO, factor INPC son lógica con muchos casos edge — debe ser testeable en aislamiento.
4. **Toda llamada externa pasa por `connectors/` con cache Redis**. Nunca `httpx.get(...)` directo desde router o service. Esto previene rate limiting accidental y permite trabajar offline con datos cacheados.
5. **Posiciones y cash son vistas calculadas, no tablas**. Es la única forma de garantizar que siempre estén correctas tras edits/deletes de transacciones.
6. **Tests obligatorios** para `services/fiscal.py`, `services/portfolio.py`, `services/valuacion.py`, `utils/inpc_factor.py`. PR sin tests para estos módulos no se mergea.
7. **Migraciones Alembic siempre**. Prohibido `Base.metadata.create_all()` fuera de tests.
8. **Comisión y total de transacciones se calculan server-side** desde inputs primitivos. Nunca confiar en `total` enviado por el cliente.
9. **Auto-marcar `es_fibra=true`** al insertar transacción tipo `dividend` si la emisora es FIBRA. El usuario no debe acordarse.
10. **Formato MX consistente**: MXN con `$` y sufijo, fechas DD/MM/AAAA, separador miles `,`, decimal `.`. Implementado en `frontend/src/lib/formatters.ts` — todo render numérico pasa por ahí.
11. **Sin features de ejecución de órdenes**. Esto es herramienta de análisis. Si alguien pide "enviar orden a GBM", se rechaza.
12. **Encoding ASCII en archivos .ps1**. Regla global del usuario para evitar problemas de PowerShell en Windows.
13. **Nunca commitear `.env*` con secretos reales**. `.env.example` con placeholders sí.
14. **Idioma español** en UI, mensajes de error al usuario, y comentarios cuando ameriten. Identificadores técnicos OK en inglés.
15. **Backups de Postgres a `F:\Respaldos\`** programados desde Windows Task Scheduler. No commitear el script si tiene paths personales.
