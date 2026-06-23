# rh v2 — Motor de Documentos de RH · Blueprint

> Generado por The Architect el 27/05/2026
> Arquetipo: Internal Tool / Dashboard (LAN, multiusuario)
> Empresa: Cremería Herrera Méndez (Cremería HM) · Tonalá, Jalisco
> Repo: `huheme25/cremeria-hm-rh` (branch `v2`) · Container destino: `rh-web`

---

## 0. Cómo usar este blueprint (léeme primero)

Eres una instancia de Claude Code que va a **construir rh v2 desde cero**. Este documento es tu única fuente de verdad: contiene el stack, el modelo de datos, los schemas exactos, el sistema de diseño, el orden de build numerado y el `CLAUDE.md` que debes colocar en el proyecto.

**Contexto del rewrite:** existe `cremeria-hm-rh` v1 (FastAPI + SQLite) que genera SOLO perfiles de puesto y corre en producción (LAN puerto 8000, container `rh-web`). v1 es legacy. v2 lo reemplaza, estilo "EspritOS reemplazó al CRM-ERP": se construye nuevo, se migran los datos, se hace cutover. **v1 NO se toca mientras Yunuen lo use.**

**Reglas de oro mientras construyes:**
1. v2 vive en branch `v2` del repo existente, en carpeta local nueva `cremeria-hm-rh-v2/` (NO en la carpeta de v1).
2. Sigue el orden de build de la Sección 9. No saltes pasos.
3. El `contenido` de cada documento se valida con el MISMO JSON Schema que se pasa a Claude API (single source of truth).
4. Las fuentes (Playfair Display, Inter) se auto-hospedan en `static/`. La caja es LAN/sin internet garantizado — NUNCA dependas del CDN de Google Fonts.
5. Idioma de TODO (UI, código comentado, documentos generados): **español**. Moneda MXN, fechas DD/MM/AAAA.

---

## 1. Project Overview

### Visión
rh v2 es una **plataforma de generación de documentos de RH on-brand** para Cremería HM, asistida por Claude API. No es un generador de perfiles: es un **motor multi-documento con tipos enchufables**. La v1 de la plataforma soporta dos tipos —Perfil de Puesto y Plan de Trabajo 30/60/90 días— pero el modelo está diseñado para que agregar un tipo nuevo (carta oferta, evaluación de desempeño, inducción) sea enchufar **un schema + un template + un prompt**, sin tocar el core.

Cada documento se genera con Claude (structured outputs + streaming sección por sección), se edita y regenera por sección, se versiona, y se exporta como **HTML "estilo de casa" listo para imprimir a PDF**. El render on-brand (Playfair Display + Inter, acento café #6B4E32, secciones numeradas, bloque de firmas) es requisito central, no un extra.

### Usuarios
- **Yunuen López** — Gerente de RH. Usuaria principal: genera, edita y exporta documentos.
- **Beto Herrera** — Admin: edita el contexto de empresa y gestiona usuarios.

### Goals
- Reemplazar v1 sin pérdida de datos (migrar perfiles existentes de SQLite a Postgres).
- Generar Perfiles de Puesto (paridad funcional con v1) + Planes 30/60/90.
- Render on-brand idéntico a la plantilla autoritativa, exportable a PDF de alta fidelidad.
- Arquitectura de tipos enchufables verificada (agregar el 2º tipo no toca el core).

### Success Metrics
- 100% de los perfiles de v1 migrados y renderizables en v2.
- Generar un Perfil completo en streaming < 60s; un Plan 30/60/90 < 90s.
- Agregar el tipo "Plan" requiere 0 cambios en `documentos/` core (solo `tipos/`).
- PDF on-brand fiel al preview del navegador (mismo HTML).

---

## 2. Tech Stack

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| Framework | Django 5.1 (vistas async) | Convención del cluster CremeriaHM (EspritOS/Portal/Pulso). Async necesario para streaming SSE de Claude |
| Lenguaje | Python 3.12 | Estándar del cluster |
| Frontend interactivo | HTMX 2 + Alpine 3 | Wizard, edición inline y regeneración por sección sin SPA. Convención del cluster |
| Estilos (app) | Tailwind CSS v4 | Convención del cluster para la UI de la app |
| Estilos (documentos) | CSS "estilo de casa" dedicado | El render del documento NO usa Tailwind; usa el CSS autoritativo (Sección 7) |
| Base de datos | Postgres 16 (`rh-db`, dedicado) | Convención del cluster. JSONB para `contenido` |
| Cache / sesiones | Redis 7 (`rh-redis`) | Sesiones, cache. Convención del cluster |
| IA | Anthropic Python SDK (`AsyncAnthropic`) | Claude API directo, sin proxy. Structured outputs + streaming |
| PDF | Playwright (Chromium headless, API async) | Fidelidad pixel-perfect del HTML on-brand (Grid+flex+@media print) |
| Auth | `django.contrib.auth` (sesiones) | Built-in, 2 roles, sin dependencias cloud. Air-gapped LAN |
| Servidor | Uvicorn (ASGI) | Vistas async + streaming |
| Contenedor | Docker + docker-compose | Cluster `cremeriahm-prod`, red `cremeriahm-prod` |
| Gestor de paquetes | pip + requirements.txt | Convención del cluster |

**Modelo de Claude:** `claude-sonnet-4-5-20250929` (el que ya usa v1; mantener salvo instrucción contraria). Reintentos con backoff en 429/529, escalado de `max_tokens` 4096→8192 si `stop_reason == "max_tokens"`.

---

## 3. Directory Structure

```
cremeria-hm-rh-v2/                 # branch v2, carpeta local separada de v1
├── manage.py
├── requirements.txt
├── Dockerfile                     # base con Chromium para Playwright
├── docker-compose.yml             # rh-web + rh-db + rh-redis en red cremeriahm-prod
├── .env.example                   # ANTHROPIC_API_KEY, DB, SECRET_KEY (real .env gitignored)
├── .dockerignore
├── CLAUDE.md                      # (ver Sección 15 — pegar tal cual)
├── tailwind.config.js
├── static/
│   ├── css/app.css                # Tailwind (UI de la app)
│   ├── fonts/                     # Playfair Display + Inter AUTO-HOSPEDADAS (woff2)
│   └── doc/casa.css               # CSS "estilo de casa" para documentos (Sección 7)
├── templates/
│   ├── base.html                  # shell de la app (sidebar + header, Tailwind)
│   ├── login.html
│   ├── biblioteca.html            # listado de documentos (filtro por tipo)
│   ├── wizard.html                # wizard de creación (selecciona tipo → form → genera)
│   ├── editor.html                # editor con regeneración por sección (HTMX)
│   ├── configuracion.html         # edición del contexto de empresa (con backup .bak)
│   └── doc/                       # templates de RENDER on-brand, uno por tipo
│       ├── _layout_casa.html      # estructura base on-brand (header/firma/footer/print)
│       ├── perfil.html            # render del Perfil de Puesto
│       └── plan_30_60_90.html     # render del Plan 30/60/90
├── config/                        # proyecto Django (settings)
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py                    # ASGI (Uvicorn) — streaming
│   └── wsgi.py
├── core/                          # app: auth, layout, contexto de empresa
│   ├── models.py                  # ContextoEmpresa
│   ├── views.py                   # login, dashboard, configuracion
│   ├── context_loader.py          # carga los 3 JSON → system prompt (porta v1)
│   └── management/commands/
│       └── migrar_v1.py           # SQLite v1 → Postgres v2 (idempotente)
├── documentos/                    # app: CORE agnóstico de tipo
│   ├── models.py                  # Documento, DocumentoVersion
│   ├── views.py                   # CRUD, generar(stream), regenerar_seccion, render, pdf
│   ├── pdf.py                      # Playwright: HTML on-brand → PDF
│   ├── render.py                  # resuelve template del tipo + inyecta casa.css
│   └── urls.py
├── tipos/                         # app: REGISTRY + descriptores enchufables
│   ├── registry.py                # DocumentType, REGISTRY, register()
│   ├── base.py                    # protocolo/dataclass DocumentType
│   ├── perfil/
│   │   ├── schema.py              # PERFIL_SCHEMA (Sección 4.3 — verbatim de v1)
│   │   └── prompt.py              # system + user prompt del perfil
│   └── plan_30_60_90/
│       ├── schema.py              # PLAN_SCHEMA (Sección 4.4 — nuevo)
│       └── prompt.py              # system + user prompt del plan (modos aterrizaje/estructuración)
├── ia/                            # app: cliente Anthropic + generación
│   ├── client.py                  # singleton AsyncAnthropic
│   ├── generator.py               # generar(stream), regenerar_seccion (agnóstico via registry)
│   └── sse.py                     # helpers SSE + heurística detección de secciones (porta v1)
└── db/
    └── puestos_v1.bak.db          # copia READ-ONLY del SQLite de v1 para migrar (gitignored)
```

---

## 4. Data Model

### 4.1 Entidades

**`Documento`** (`documentos.models.Documento`) — el modelo único, agnóstico de tipo.
| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | BigAutoField | PK |
| `tipo` | CharField(40) | slug del tipo (`perfil`, `plan_30_60_90`). Debe existir en REGISTRY |
| `titulo` | CharField(200) | Nombre legible (p.ej. "Cajera — Piso de Ventas") |
| `contenido` | JSONField (JSONB) | El documento estructurado. Validado contra `REGISTRY[tipo].json_schema` |
| `version` | PositiveIntegerField | Empieza en 1, +1 por cada actualización |
| `documento_contexto` | FK self → Documento (null) | Link OPCIONAL. Un plan referencia un perfil; su `contenido` se inyecta como contexto al generar |
| `creado_por` | FK → User (null) | Atribución (auth) |
| `creado_en` | DateTimeField(auto_now_add) | |
| `actualizado_en` | DateTimeField(auto_now) | |

**`DocumentoVersion`** (`documentos.models.DocumentoVersion`) — snapshot histórico.
| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | BigAutoField | PK |
| `documento` | FK → Documento (CASCADE) | |
| `version` | PositiveIntegerField | Número de versión del snapshot |
| `contenido` | JSONField (JSONB) | Snapshot del `contenido` ANTES de la actualización |
| `creado_en` | DateTimeField(auto_now_add) | |

**`ContextoEmpresa`** (`core.models.ContextoEmpresa`) — el contexto editable que dirige la generación.
| Campo | Tipo | Notas |
|-------|------|-------|
| `id` | BigAutoField | PK (singleton: usar `pk=1`) |
| `empresa` | JSONField | Equivale a `empresa.json` de v1 |
| `politicas` | JSONField | Equivale a `politicas.json` de v1 |
| `catalogo_puestos` | JSONField | Equivale a `catalogo_puestos.json` de v1 |
| `actualizado_en` | DateTimeField(auto_now) | |

> El backup `.bak`: antes de guardar cambios desde `configuracion.html`, serializar el `ContextoEmpresa` actual a `db/contexto_empresa.<timestamp>.bak.json` (porta el comportamiento de v1).

### 4.2 Relationships
- `Documento 1—N DocumentoVersion` (historial de versiones).
- `Documento N—1 Documento` (self-FK opcional `documento_contexto`: plan → perfil).
- `Documento N—1 User` (`creado_por`).
- `ContextoEmpresa` es singleton (una fila).

### 4.3 Schema del tipo PERFIL (verbatim de v1 — `tipos/perfil/schema.py`)

> Portar EXACTAMENTE este schema desde `cremeria-hm-rh/services/ai_generator.py`. Es el contrato probado de 11 secciones. NO modificar nombres de campos.

```python
PERFIL_SCHEMA = {
    "type": "object",
    "properties": {
        "encabezado": {
            "type": "object",
            "properties": {
                "nombre_puesto": {"type": "string"},
                "departamento": {"type": "string"},
                "codigo": {"type": "string"},
                "version": {"type": "string"},
            },
            "required": ["nombre_puesto", "departamento", "codigo", "version"],
            "additionalProperties": False,
        },
        "objetivo_general": {"type": "string"},
        "perfil": {
            "type": "object",
            "properties": {
                "escolaridad": {"type": "string"},
                "experiencia_laboral": {"type": "string"},
                "conocimientos_tecnicos": {"type": "array", "items": {"type": "string"}},
                "habilidades_blandas": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["escolaridad", "experiencia_laboral", "conocimientos_tecnicos", "habilidades_blandas"],
            "additionalProperties": False,
        },
        "cualidades_especificas": {
            "type": "object",
            "properties": {
                "edad_preferida": {"type": "string"},
                "genero": {"type": "string"},
                "estado_civil": {"type": "string"},
                "disponibilidad_viajar": {"type": "string"},
                "cambio_residencia": {"type": "string"},
            },
            "required": ["edad_preferida", "genero", "estado_civil", "disponibilidad_viajar", "cambio_residencia"],
            "additionalProperties": False,
        },
        "condiciones": {
            "type": "object",
            "properties": {
                "contrato_prueba": {"type": "string"},
                "contrato_definitivo": {"type": "string"},
                "salario_referencia": {"type": "string"},
                "prestaciones": {"type": "string"},
                "frecuencia_pago": {"type": "string"},
                "horario": {"type": "string"},
            },
            "required": ["contrato_prueba", "contrato_definitivo", "salario_referencia", "prestaciones", "frecuencia_pago", "horario"],
            "additionalProperties": False,
        },
        "funciones": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "categoria": {"type": "string"},
                    "actividades": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "descripcion": {"type": "string"},
                                "frecuencia": {"type": "string"},
                                "herramienta": {"type": "string"},
                            },
                            "required": ["descripcion", "frecuencia", "herramienta"],
                            "additionalProperties": False,
                        },
                    },
                },
                "required": ["categoria", "actividades"],
                "additionalProperties": False,
            },
        },
        "kpis": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "indicador": {"type": "string"},
                    "medicion": {"type": "string"},
                    "meta": {"type": "string"},
                },
                "required": ["indicador", "medicion", "meta"],
                "additionalProperties": False,
            },
        },
        "resultados_esperados": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "alcance": {"type": "string"},
                    "responsabilidad": {"type": "string"},
                },
                "required": ["alcance", "responsabilidad"],
                "additionalProperties": False,
            },
        },
        "relaciones_interpersonales": {
            "type": "object",
            "properties": {
                "internas": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {"puesto": {"type": "string"}, "motivo": {"type": "string"}},
                        "required": ["puesto", "motivo"],
                        "additionalProperties": False,
                    },
                },
                "externas": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {"puesto": {"type": "string"}, "motivo": {"type": "string"}},
                        "required": ["puesto", "motivo"],
                        "additionalProperties": False,
                    },
                },
            },
            "required": ["internas", "externas"],
            "additionalProperties": False,
        },
        "documentos_clave": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "documento": {"type": "string"},
                    "mantenimiento": {"type": "string"},
                    "formato": {"type": "string"},
                },
                "required": ["documento", "mantenimiento", "formato"],
                "additionalProperties": False,
            },
        },
        "verificacion_antecedentes": {
            "type": "object",
            "properties": {
                "referencias": {"type": "string"},
                "psicometricos": {"type": "string"},
                "buro_credito": {"type": "string"},
                "historial_legal": {"type": "string"},
            },
            "required": ["referencias", "psicometricos", "buro_credito", "historial_legal"],
            "additionalProperties": False,
        },
    },
    "required": [
        "encabezado", "objetivo_general", "perfil", "cualidades_especificas",
        "condiciones", "funciones", "kpis", "resultados_esperados",
        "relaciones_interpersonales", "documentos_clave", "verificacion_antecedentes",
    ],
    "additionalProperties": False,
}

SECCIONES_ORDENADAS = list(PERFIL_SCHEMA["required"])

SECCIONES_LABELS = {
    "encabezado": "Encabezado",
    "objetivo_general": "Objetivo General",
    "perfil": "Perfil del Candidato",
    "cualidades_especificas": "Cualidades Específicas",
    "condiciones": "Condiciones Laborales",
    "funciones": "Funciones y Actividades",
    "kpis": "KPIs",
    "resultados_esperados": "Resultados Esperados",
    "relaciones_interpersonales": "Relaciones Interpersonales",
    "documentos_clave": "Documentos Clave",
    "verificacion_antecedentes": "Verificación de Antecedentes",
}
```

### 4.4 Schema del tipo PLAN 30/60/90 (nuevo — `tipos/plan_30_60_90/schema.py`)

> Diseño nuevo. Metadatos + 3 bloques (30/60/90) con el patrón de **delegación gradual** (30d acompaña → 60d prepara → 90d delega). El `modo` cambia el encuadre.

```python
_BLOQUE = {
    "type": "object",
    "properties": {
        "titulo": {"type": "string"},                 # p.ej. "Mes 1 — Inmersión y toma de cartera"
        "enfoque": {"type": "string"},                 # encuadre del bloque según el modo y la etapa de delegación
        "objetivos": {"type": "array", "items": {"type": "string"}},
        "semanas": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "titulo": {"type": "string"},      # "Semana 1 — Bienvenida, producto y sistemas"
                    "rango_dias": {"type": "string"},  # "Días 1-6"
                    "actividades": {"type": "array", "items": {"type": "string"}},
                },
                "required": ["titulo", "rango_dias", "actividades"],
                "additionalProperties": False,
            },
        },
        "entregables": {"type": "array", "items": {"type": "string"}},
        "metricas_avance": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "indicador": {"type": "string"},
                    "meta": {"type": "string"},
                },
                "required": ["indicador", "meta"],
                "additionalProperties": False,
            },
        },
        "apoyos_recursos": {"type": "array", "items": {"type": "string"}},
    },
    "required": ["titulo", "enfoque", "objetivos", "semanas", "entregables", "metricas_avance", "apoyos_recursos"],
    "additionalProperties": False,
}

PLAN_SCHEMA = {
    "type": "object",
    "properties": {
        "metadatos": {
            "type": "object",
            "properties": {
                "puesto": {"type": "string"},
                "titular": {"type": "string"},          # persona (nueva o existente)
                "modo": {"type": "string", "enum": ["aterrizaje", "estructuracion"]},
                "fecha_inicio": {"type": "string"},      # DD/MM/AAAA
                "jefe_directo": {"type": "string"},
                "linea_reporte": {"type": "string"},
            },
            "required": ["puesto", "titular", "modo", "fecha_inicio", "jefe_directo", "linea_reporte"],
            "additionalProperties": False,
        },
        "resumen_ejecutivo": {"type": "string"},
        "reto_en_una_linea": {"type": "string"},
        "bloque_30": _BLOQUE,
        "bloque_60": _BLOQUE,
        "bloque_90": _BLOQUE,
        "delegacion_gradual": {
            "type": "object",
            "properties": {
                "dia_30": {"type": "string"},   # "acompaña / observa"
                "dia_60": {"type": "string"},   # "prepara / toma con sombra"
                "dia_90": {"type": "string"},   # "delega / autónomo"
            },
            "required": ["dia_30", "dia_60", "dia_90"],
            "additionalProperties": False,
        },
    },
    "required": ["metadatos", "resumen_ejecutivo", "reto_en_una_linea", "bloque_30", "bloque_60", "bloque_90", "delegacion_gradual"],
    "additionalProperties": False,
}

SECCIONES_ORDENADAS = list(PLAN_SCHEMA["required"])

SECCIONES_LABELS = {
    "metadatos": "Metadatos",
    "resumen_ejecutivo": "Resumen Ejecutivo",
    "reto_en_una_linea": "El Reto en una Línea",
    "bloque_30": "Primeros 30 días",
    "bloque_60": "Días 31-60",
    "bloque_90": "Días 61-90",
    "delegacion_gradual": "Delegación Gradual",
}
```

**Modos (cambian el encuadre, NO el schema):**
- `aterrizaje` — persona NUEVA. 30d = inmersión/aprendizaje, 60d = toma de funciones con sombra, 90d = operación autónoma.
- `estructuracion` — titular EXISTENTE que ordena/mejora su área. 30d = diagnóstico del estado actual, 60d = rediseño de procesos, 90d = implementación y métricas.

El `modo` se inyecta al `system_prompt_builder` del tipo plan para reencuadrar todo. Si `documento_contexto` apunta a un perfil, su `contenido` (funciones/KPIs reales) se pasa como contexto para alinear el plan.

### 4.5 Postgres / migraciones
Usar migraciones de Django (`makemigrations`/`migrate`). `contenido` es `JSONField` (mapea a JSONB en Postgres 16). Índices: `Documento(tipo)`, `Documento(actualizado_en)`.

---

## 5. Registry — el corazón de "enchufable"

`tipos/base.py`:
```python
from dataclasses import dataclass
from typing import Callable

@dataclass(frozen=True)
class DocumentType:
    slug: str                          # "perfil", "plan_30_60_90"
    label: str                         # "Perfil de Puesto"
    json_schema: dict                  # el schema validado por Claude + por el modelo
    secciones_orden: list[str]
    secciones_labels: dict[str, str]
    system_prompt_builder: Callable[..., str]   # (contexto_empresa, **kwargs) -> str
    user_prompt_builder: Callable[..., str]     # (form_data, documento_contexto) -> str
    template: str                      # "doc/perfil.html"
    max_tokens: int = 8192
    soporta_contexto: bool = False     # True para plan (puede linkear perfil)
```

`tipos/registry.py`:
```python
REGISTRY: dict[str, DocumentType] = {}

def register(dt: DocumentType) -> None:
    REGISTRY[dt.slug] = dt

def get(slug: str) -> DocumentType:
    if slug not in REGISTRY:
        raise KeyError(f"Tipo de documento '{slug}' no registrado")
    return REGISTRY[slug]

# Registro de los tipos incluidos (import explícito en apps.py ready())
def cargar_tipos():
    from tipos.perfil.schema import PERFIL_SCHEMA, SECCIONES_ORDENADAS as P_ORD, SECCIONES_LABELS as P_LBL
    from tipos.perfil.prompt import system_prompt as perfil_system, user_prompt as perfil_user
    from tipos.plan_30_60_90.schema import PLAN_SCHEMA, SECCIONES_ORDENADAS as PL_ORD, SECCIONES_LABELS as PL_LBL
    from tipos.plan_30_60_90.prompt import system_prompt as plan_system, user_prompt as plan_user

    register(DocumentType("perfil", "Perfil de Puesto", PERFIL_SCHEMA, P_ORD, P_LBL,
                          perfil_system, perfil_user, "doc/perfil.html", soporta_contexto=False))
    register(DocumentType("plan_30_60_90", "Plan de Trabajo 30/60/90", PLAN_SCHEMA, PL_ORD, PL_LBL,
                          plan_system, plan_user, "doc/plan_30_60_90.html", soporta_contexto=True))
```

**Regla de oro de la arquitectura:** `documentos/` (CRUD, versionado, streaming, render, PDF) e `ia/` (generación) **NUNCA importan un tipo concreto**. Solo consultan `registry.get(documento.tipo)`. Agregar un tipo = nuevo paquete en `tipos/` + `register(...)`. Cero cambios al core. El Paso 9 del build verifica esto.

---

## 6. API / Rutas (Django URLs)

| Método | Path | Descripción | Auth |
|--------|------|-------------|------|
| GET/POST | `/login/` | Login (Django auth) | público |
| POST | `/logout/` | Logout | sí |
| GET | `/` | Dashboard / biblioteca (listado de documentos, filtro por tipo) | sí |
| GET | `/documentos/nuevo/?tipo=<slug>` | Wizard de creación (form dinámico por tipo) | sí |
| POST | `/documentos/generar/` | **SSE stream**: genera el documento sección por sección | sí |
| GET | `/documentos/<id>/` | Editor del documento (HTMX) | sí |
| POST | `/documentos/<id>/seccion/<key>/regenerar/` | Regenera UNA sección, bump de versión | sí |
| POST | `/documentos/<id>/seccion/<key>/guardar/` | Guarda edición manual de una sección | sí |
| GET | `/documentos/<id>/render/` | Render on-brand (HTML imprimible) | sí |
| GET | `/documentos/<id>/pdf/` | PDF vía Playwright (descarga) | sí |
| POST | `/documentos/<id>/duplicar/` | Duplica documento | sí |
| POST | `/documentos/<id>/eliminar/` | Elimina (con confirmación) | sí |
| GET | `/documentos/<id>/versiones/` | Historial de versiones | sí |
| GET/POST | `/configuracion/` | Edita contexto de empresa (backup .bak) | superuser |

### Endpoint crítico: `POST /documentos/generar/` (streaming)
- **Request:** `tipo` (slug), `form_data` (campos del wizard según tipo), `documento_contexto_id` (opcional, solo si `soporta_contexto`).
- **Respuesta:** `StreamingHttpResponse` con `content_type="text/event-stream"`. Eventos SSE:
  - `section_complete` → `{seccion, index, label}` (mientras Claude streamea).
  - `complete` → `{documento_id}` (tras parsear el JSON completo y persistir `Documento`).
  - `error` → `{error}`.
- **Implementación:** vista async → `ia.generator.generar_stream(dt, form_data, contexto)` (async generator) → reusar la heurística de detección de secciones de v1 (`ia/sse.py`): la sección `i` se marca completa cuando aparece la key de la sección `i+1` en el texto acumulado.

### Endpoint crítico: `POST /documentos/<id>/seccion/<key>/regenerar/`
- Carga `Documento`, llama `ia.generator.regenerar_seccion(dt, contenido, key)` (wrapper schema de una sola sección, igual que v1), guarda snapshot en `DocumentoVersion`, hace `contenido[key] = nuevo`, bump `version`, devuelve el partial HTMX de esa sección renderizada.

---

## 7. Design System (estilo de casa — autoritativo)

> Fuente de verdad: `cremeria-hm-rh/documentos/_plantilla-referencia/plan_60_dias_ejecutiva_ventas.html`. Estos tokens y patrones se portan a `static/doc/casa.css`. **NO** es el azul corporativo de v1 (#1F4E79 queda DEPRECATED para documentos).

### Tokens (CSS variables)
```css
:root {
  --accent: #6B4E32;      /* café — acento principal (títulos h3, th, bordes, pills) */
  --accent-soft: #B89472;
  --accent-bg: #F8F1E7;   /* fondo de cajas .nota y filas total */
  --ink: #1f1a14;         /* texto principal */
  --muted: #6b6557;       /* texto secundario */
  --line: #e6dfd2;        /* bordes y separadores */
  --ok: #356E3F;          /* pill verde */
  --warn: #B47600;        /* pill ámbar */
  --bad: #9A2A2A;         /* pill rojo */
  --bg: #FAF7F1;          /* fondo de página (pantalla) */
}
```

### Tipografía (AUTO-HOSPEDADA en `static/fonts/`)
| Rol | Fuente | Tamaño / peso |
|-----|--------|---------------|
| H1 (título doc) | Playfair Display | 38px / 700 |
| H2 (sección numerada) | Playfair Display | 26px / 600, `border-bottom` |
| H3 (subsección) | Inter | 17px / 600, UPPERCASE, color acento |
| Cuerpo | Inter | 15px / 400, line-height 1.55 |

> Descargar los woff2 de Playfair Display (400,600,700) e Inter (300,400,500,600,700) a `static/fonts/` y declararlos con `@font-face` en `casa.css`. La app NO puede depender del CDN de Google (LAN offline).

### Componentes on-brand (portar del CSS de referencia)
- `.wrap` — contenedor max 1100px, padding 48px.
- `header.h1` con `.eyebrow` (kicker uppercase), `.lede`, `.meta` (jefe directo, línea de reporte, fecha inicio).
- `.kpis` / `.kpi` — grid de 4, número en Playfair color acento.
- `table` — header café, `td.num` tabular, `tr.total` fondo `--accent-bg`.
- `.semana` — caja con borde-izquierdo acento, `.head` (titulo + rango días). **Núcleo del render de Planes.**
- `.nota` — caja destacada fondo `--accent-bg`.
- `.pill` (+ `.ok/.warn/.bad`) — etiquetas de estado.
- `.check` — checklist con `☐`.
- `.firma` — grid 2 col, líneas de firma + nombre + rol.
- `footer.foot`.

### CSS de impresión (obligatorio)
```css
@media print {
  body { background: white; font-size: 12px; }
  .wrap { padding: 20px; max-width: 100%; }
  h2 { page-break-before: always; page-break-after: avoid; }
  h2:first-of-type { page-break-before: avoid; }
  h3, h4 { page-break-after: avoid; }
  .semana, .nota, .kpis, .firma { page-break-inside: avoid; }
  table { page-break-inside: auto; }
  thead { display: table-header-group; }
  tr { page-break-inside: avoid; }
  a { color: inherit; text-decoration: none; }
}
```

### UI de la app (Tailwind) — distinta del documento
La UI (sidebar, biblioteca, wizard, editor) usa Tailwind v4 con la misma paleta de marca (acento café) para coherencia, pero es funcional y densa (es herramienta interna: velocidad > animaciones). El documento renderizado usa exclusivamente `casa.css`.

---

## 8. Auth & Authorization

### Flujo
`django.contrib.auth` con sesiones (cookie, almacenadas en Redis). Sin registro público. Usuarios creados por `createsuperuser` / admin de Django.

### Rutas protegidas
Todo excepto `/login/` requiere `@login_required` (o `LoginRequiredMiddleware` en Django 5.1). `/configuracion/` requiere `is_superuser`.

### Roles
| Rol | Quién | Puede |
|-----|-------|-------|
| Superuser | Beto | Todo: CRUD documentos + editar contexto de empresa + gestionar usuarios |
| Staff | Yunuen | CRUD documentos (generar/editar/regenerar/exportar). NO edita contexto ni usuarios |

Distinción simple por `is_superuser` (sin Groups — overkill para 2-3 usuarios). `creado_por` se llena con `request.user` al generar.

### Sesiones
Sessions backend en Redis (`django.contrib.sessions.backends.cache` o `cached_db`). `SESSION_COOKIE_AGE` largo (LAN, sin riesgo de exposición). CSRF activo en todos los POST (HTMX incluye el token).

---

## 9. Build Order (LA SECCIÓN CRÍTICA)

> Cada paso es entregable y verificable. No avances sin que el anterior funcione.

**Paso 1 — Scaffold + cluster Docker**
- Crear proyecto Django 5.1 en `cremeria-hm-rh-v2/` (branch `v2`). Apps: `core`, `documentos`, `tipos`, `ia`.
- `requirements.txt`: django==5.1.*, uvicorn, psycopg[binary], redis, django-redis, anthropic, playwright, python-dotenv, whitenoise.
- `docker-compose.yml`: servicios `rh-web` (build local), `rh-db` (postgres:16), `rh-redis` (redis:7), red externa `cremeriahm-prod`, naming `{app}-{role}`. Volúmenes para Postgres y para `static/`.
- `Dockerfile`: base Python 3.12; instalar Playwright Chromium (`playwright install --with-deps chromium`); recopilar estáticos; arrancar Uvicorn ASGI.
- `.env.example` con `ANTHROPIC_API_KEY`, `SECRET_KEY`, `DATABASE_URL`, `REDIS_URL`, `DJANGO_DEBUG`.
- **Verificar:** `docker compose up` levanta los 3 containers healthy; `/` responde (aún vacío).

**Paso 2 — Auth + layout base**
- Configurar `django.contrib.auth`, sesiones en Redis, `login.html`, `LoginRequiredMiddleware`.
- `base.html`: shell con sidebar (Biblioteca, Nuevo documento, Configuración [solo superuser]) + header con usuario/logout (Tailwind v4).
- `createsuperuser` para Beto; crear usuario staff Yunuen.
- **Verificar:** login funciona; rutas protegidas redirigen a `/login/`; `/configuracion/` bloquea a no-superuser.

**Paso 3 — Modelo Documento + registry + migración v1**
- `documentos/models.py`: `Documento`, `DocumentoVersion`. `core/models.py`: `ContextoEmpresa`.
- `tipos/base.py` + `tipos/registry.py` (vacío de tipos aún, pero funcional). Cargar en `AppConfig.ready()`.
- `core/management/commands/migrar_v1.py`: lee `db/puestos_v1.bak.db` (copia read-only del SQLite de v1), crea un `Documento(tipo='perfil', titulo=nombre, contenido=perfil_json, version, creado_en/actualizado_en preservados)` por fila; mapea `historial_versiones` → `DocumentoVersion`; valida cada `perfil_json` contra `PERFIL_SCHEMA` (loguea mismatches, no aborta); idempotente (re-ejecutable: detecta ya-importados por titulo+tipo o flag `--reset`).
- `core/context_loader.py`: carga `ContextoEmpresa` (o seed inicial desde los 3 JSON de v1) → construye system prompt (porta `cargar_contexto_empresa()` de v1, leyendo de DB en vez de archivos).
- Seed inicial de `ContextoEmpresa` con los JSON de v1 (`empresa.json`, `politicas.json`, `catalogo_puestos.json`).
- **Verificar:** `migrate` OK; `migrar_v1` importa todos los perfiles de v1 sin pérdida; conteo coincide; `DocumentoVersion` poblado.

**Paso 4 — Tipo Perfil (registrar el primer tipo)**
- `tipos/perfil/schema.py` (Sección 4.3 verbatim). `tipos/perfil/prompt.py` (porta system/user prompt de v1, leyendo contexto de DB).
- `register(...)` del tipo perfil en el registry.
- **Verificar:** `registry.get('perfil')` devuelve el descriptor; schema válido.

**Paso 5 — Motor IA streaming (agnóstico de tipo)**
- `ia/client.py`: singleton `AsyncAnthropic` (porta de v1).
- `ia/generator.py`: `generar_stream(dt, form_data, contexto)` async generator usando `client.messages.stream()` con `output_config={"format": {"type": "json_schema", "schema": dt.json_schema}}`; reintentos 429/529; escalado 4096→8192. `regenerar_seccion(dt, contenido, key)` (wrapper de una sección). **Todo vía `dt` del registry — sin tipos hardcodeados.**
- `ia/sse.py`: heurística de detección de secciones (porta `_detectar_seccion_completada` de v1, usando `dt.secciones_orden`).
- `documentos/views.py`: vista async `generar` → `StreamingHttpResponse` SSE.
- `wizard.html`: selección de tipo → form dinámico (campos del perfil) → EventSource consume SSE, marca secciones completadas (Alpine).
- **Verificar:** generar un Perfil nuevo en streaming muestra progreso sección-por-sección y persiste `Documento`.

**Paso 6 — Editor + regeneración por sección**
- `editor.html`: muestra el `contenido` por secciones (según `dt.secciones_orden`/`labels`); cada sección con botón "Regenerar" (HTMX POST) y edición manual.
- Endpoints `regenerar` y `guardar` por sección → bump de versión + snapshot.
- `/documentos/<id>/versiones/` lista el historial.
- **Verificar:** regenerar una sección actualiza solo esa sección y crea una `DocumentoVersion`; edición manual se guarda.

**Paso 7 — Render on-brand + fuentes auto-hospedadas**
- Descargar Playfair Display + Inter (woff2) a `static/fonts/`. `static/doc/casa.css` con tokens + componentes + print CSS (Sección 7).
- `templates/doc/_layout_casa.html` (header/firma/footer/print) + `doc/perfil.html` (render del perfil con secciones numeradas, tablas café, etc.).
- `documentos/render.py` + endpoint `/documentos/<id>/render/`.
- **Verificar:** el render del perfil se ve on-brand (Playfair/Inter, café), las fuentes cargan SIN internet, `Ctrl+P` muestra saltos de página correctos.

**Paso 8 — PDF vía Playwright**
- `documentos/pdf.py`: lanza Chromium headless (instancia reusada), abre el HTML de `/render/`, `page.pdf(print_background=True, format="Letter", margin=...)`. Vista async `/documentos/<id>/pdf/` devuelve el PDF.
- **Verificar:** el PDF descargado es fiel al preview del navegador (colores, fuentes, saltos de página). Fallback documentado: imprimir a PDF desde el navegador.

**Paso 9 — Tipo Plan 30/60/90 (PRUEBA de enchufabilidad)**
- `tipos/plan_30_60_90/schema.py` (Sección 4.4) + `prompt.py` (modos `aterrizaje`/`estructuracion`; si hay `documento_contexto`, inyecta el perfil). `register(...)`.
- `doc/plan_30_60_90.html` (usa `.semana`, `.kpis`, `.nota`, `.firma`; bloques 30/60/90; sección de delegación gradual).
- Wizard: form del plan (puesto, titular, modo, fecha_inicio, link opcional a perfil).
- **VERIFICAR LO CRÍTICO:** agregar este tipo NO requirió tocar `documentos/` ni `ia/`. Si tuviste que tocar el core, la abstracción del registry está mal — corrígela.
- **Verificar:** generar un Plan en ambos modos; con y sin perfil ligado; render on-brand; PDF.

**Paso 10 — Contexto de empresa editable**
- `configuracion.html`: edita `ContextoEmpresa` (empresa/politicas/catalogo). Antes de guardar: backup `.bak` timestamped. Solo superuser.
- **Verificar:** editar el contexto cambia la generación; backup se crea.

**Paso 11 — Polish + cutover**
- Estados de carga/vacío/error; biblioteca con filtro por tipo y búsqueda; confirmaciones de borrado.
- Desplegar v2 en **puerto paralelo** (p.ej. 8001/8100) en el cluster. UAT con Yunuen mientras v1 sigue en 8000.
- Re-ejecutar `migrar_v1` con un dump fresco del SQLite de v1 justo antes del cutover (captura perfiles creados en el ínterin).
- Cutover: v2 toma el puerto/booking LAN, detener container v1, tag `v1-archivado-YYYYMMDD`, backup del SQLite a `F:\Respaldos`. `v2` → `main`.
- **Verificar:** Yunuen valida; todos los perfiles presentes; PDF on-brand correcto.

---

## 10. Environment Setup

### Prerequisitos
- Docker Desktop (Windows) con la red `cremeriahm-prod` existente.
- Python 3.12 (para desarrollo local fuera de Docker, opcional).
- Acceso al repo `huheme25/cremeria-hm-rh`.

### Variables de entorno (`.env`, gitignored)
| Variable | Descripción | Dónde obtener |
|----------|-------------|---------------|
| `ANTHROPIC_API_KEY` | Clave de Claude API | Consola Anthropic (reusar la de v1) |
| `SECRET_KEY` | Django secret | Generar (`django.core.management.utils.get_random_secret_key`) |
| `DATABASE_URL` | Postgres `rh-db` | `postgres://rh:<pwd>@rh-db:5432/rh` |
| `REDIS_URL` | Redis `rh-redis` | `redis://rh-redis:6379/0` |
| `DJANGO_DEBUG` | `0` en prod | — |
| `DJANGO_ALLOWED_HOSTS` | hosts LAN | IP/hostname del servidor |

### Comandos iniciales
```bash
git checkout -b v2                      # en clon/worktree separado → carpeta cremeria-hm-rh-v2/
cp .env.example .env                    # llenar valores
docker compose build
docker compose up -d
docker compose exec rh-web python manage.py migrate
docker compose exec rh-web python manage.py createsuperuser   # Beto
# Copiar el SQLite de v1 a db/puestos_v1.bak.db (READ-ONLY), luego:
docker compose exec rh-web python manage.py migrar_v1
docker compose exec rh-web python manage.py collectstatic --noinput
```

---

## 11. Dependencies

### Core
| Paquete | Propósito |
|---------|-----------|
| django==5.1.* | Framework |
| uvicorn[standard] | Servidor ASGI (async + streaming) |
| psycopg[binary] | Driver Postgres 16 |
| redis + django-redis | Cache y sesiones |
| anthropic | Claude API (AsyncAnthropic, structured outputs, streaming) |
| playwright | Chromium headless → PDF |
| python-dotenv | Carga `.env` |
| whitenoise | Servir estáticos en prod |

### Dev
| Paquete | Propósito |
|---------|-----------|
| pytest + pytest-django + pytest-asyncio | Tests (unit + async) |
| ruff | Lint + format |
| django-debug-toolbar | Debug local (opcional) |

---

## 12. Deployment Strategy

- **Hosting:** Docker en el servidor del cluster `cremeriahm-prod` (Windows + Docker Desktop). Container `rh-web` + `rh-db` + `rh-redis`. Red externa `cremeriahm-prod`. Sin exposición a internet — solo LAN.
- **Puerto:** v2 corre en puerto paralelo durante UAT; toma el puerto LAN de v1 (8000) en el cutover.
- **CI/CD:** push a branch `v2`; build manual en el servidor (igual que el resto del cluster — sin pipeline cloud). Ver `E:\ClaudeWorks\DEPLOY-WINDOWS.md`.
- **Entornos:** dev local (DEBUG=1, Postgres local o el del compose) y prod (DEBUG=0, en el cluster). No hay staging formal: el "puerto paralelo" hace de staging.
- **Backups:** Postgres `rh-db` entra al esquema de respaldo del cluster. El SQLite de v1 se preserva en `F:\Respaldos` tras el cutover.

---

## 13. Testing Strategy

- **Unit:** validación de `contenido` contra el schema del tipo (perfil y plan); `context_loader` arma el system prompt correcto; registry resuelve tipos.
- **Integración:** `migrar_v1` importa un SQLite de muestra sin pérdida (conteo y contenido); CRUD de `Documento` + versionado (snapshot antes de update, bump de versión); endpoints HTMX devuelven partials correctos.
- **Async:** `generar_stream` emite eventos SSE en orden; `regenerar_seccion` devuelve solo la sección pedida.
- **E2E (Playwright):** flujo completo generar Perfil → editar → render → PDF; generar Plan en ambos modos. Verificar que el PDF se produce y las fuentes cargan offline.
- **Cuándo correr:** unit/integración en cada commit; E2E antes del cutover.

---

## 14. Skills to Use During Build

| Skill | Cuándo | Por qué |
|-------|--------|---------|
| `/frontend-design` | Pasos 2, 6 (layout app, editor) | UI interna limpia y densa con HTMX/Alpine/Tailwind |
| `/ui-ux-pro-max` | Paso 7 (render on-brand) | Afinar el sistema visual del documento partiendo de la plantilla autoritativa |
| `/playwright-cli` | Paso 8 y E2E | Generación de PDF y pruebas de flujo en navegador |
| `/claude-api` | Paso 5 (motor IA) | Patrón correcto de structured outputs + streaming + prompt caching del contexto de empresa |

> **Nota sobre prompt caching:** el system prompt (contexto de empresa) es grande y estable entre generaciones. Cachearlo (cache_control en el bloque system) reduce costo/latencia notablemente. Implementar en el Paso 5.

---

## 15. CLAUDE.md para el proyecto destino

```markdown
# rh v2 — Motor de Documentos de RH (Cremería HM)

Plataforma Django que genera, edita y exporta documentos de RH on-brand (Perfiles de Puesto y Planes 30/60/90) con Claude API. Motor multi-documento con tipos enchufables. Reemplaza a v1 (FastAPI/SQLite). Uso interno LAN, sin internet.

## Commands

- `docker compose up -d` — Levanta rh-web + rh-db + rh-redis
- `docker compose exec rh-web python manage.py migrate` — Aplica migraciones
- `docker compose exec rh-web python manage.py migrar_v1` — Migra perfiles de v1 (SQLite→Postgres, idempotente)
- `docker compose exec rh-web python manage.py createsuperuser` — Crea admin
- `docker compose exec rh-web python manage.py collectstatic --noinput` — Estáticos
- `docker compose exec rh-web pytest` — Tests
- `docker compose exec rh-web ruff check .` — Lint

## Tech Stack

Django 5.1 (async) + HTMX 2 + Alpine 3 + Tailwind v4 + Postgres 16 (rh-db) + Redis 7 (rh-redis) + Anthropic SDK + Playwright. Uvicorn ASGI. Cluster Docker `cremeriahm-prod`, container `rh-web`, naming {app}-{role}.

## Architecture

### Apps
- `core/` — Auth (django.contrib.auth, sesiones en Redis), layout, ContextoEmpresa editable, comando migrar_v1.
- `documentos/` — CORE agnóstico de tipo: modelo Documento + DocumentoVersion, CRUD, versionado, render, pdf.
- `tipos/` — Registry + descriptores enchufables (perfil/, plan_30_60_90/). Cada tipo = schema + prompt + template.
- `ia/` — Cliente AsyncAnthropic, generación streaming SSE, regeneración por sección.

### Data Flow
Wizard (HTMX) → vista async `generar` → `ia.generator.generar_stream(dt, ...)` con output_config json_schema → StreamingHttpResponse SSE sección-por-sección → persiste Documento(contenido=JSONB). Editor → regenerar/guardar por sección → snapshot en DocumentoVersion + bump version. Render → template del tipo + casa.css → PDF vía Playwright sobre el MISMO HTML.

### Key Patterns
- **Registry, no herencia de modelos:** un solo modelo Documento (tipo + JSONField contenido). El core NUNCA importa un tipo concreto; siempre `registry.get(documento.tipo)`. Agregar un tipo = paquete nuevo en tipos/ + register(). Cero cambios al core.
- **Schema = single source of truth:** el mismo JSON Schema valida `contenido` y se pasa a Claude (structured outputs).
- **Streaming async:** vistas async + StreamingHttpResponse. Heurística: sección i completa cuando aparece la key de i+1.
- **Contexto de empresa cacheado:** system prompt grande y estable → usar prompt caching (cache_control).

## Code Organization Rules

1. `documentos/` e `ia/` NO importan tipos concretos. Solo `registry.get(slug)`. Si necesitas un `if tipo == ...` en el core, la abstracción está mal.
2. Un schema por archivo en `tipos/<tipo>/schema.py`. Nombres de campos del perfil son INTOCABLES (contrato con v1).
3. El render del documento usa SOLO `static/doc/casa.css`. La UI de la app usa Tailwind. No mezclar.
4. Fuentes auto-hospedadas en `static/fonts/`. NUNCA CDN de Google (LAN offline).
5. Vistas de generación/PDF son `async def`. Acceso a DB con `sync_to_async` o el ORM async de Django.

## Design System (documento — estilo de casa)

Tokens: --accent #6B4E32, --accent-soft #B89472, --accent-bg #F8F1E7, --ink #1f1a14, --muted #6b6557, --line #e6dfd2, --ok #356E3F, --warn #B47600, --bad #9A2A2A, --bg #FAF7F1.
Tipografía: Playfair Display (H1 38px/700, H2 26px/600) + Inter (H3 17px/600 uppercase acento, cuerpo 15px/400 lh 1.55).
Componentes: .wrap (max 1100px), header.h1 (.eyebrow/.lede/.meta), .kpis/.kpi, table (header café, td.num, tr.total), .semana (núcleo de Planes), .nota, .pill (.ok/.warn/.bad), .check, .firma, footer.foot.
Print: h2 { page-break-before: always } salvo el primero; .semana/.nota/.kpis/.firma { page-break-inside: avoid }; thead { display: table-header-group }.
El azul corporativo de v1 (#1F4E79) está DEPRECATED para documentos.

## Environment Variables

| Variable | Descripción |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Clave Claude API (gitignored) |
| `SECRET_KEY` | Django secret |
| `DATABASE_URL` | Postgres rh-db |
| `REDIS_URL` | Redis rh-redis |
| `DJANGO_DEBUG` | 0 en prod |
| `DJANGO_ALLOWED_HOSTS` | hosts LAN |

## Reglas No Negociables

1. Idioma español en todo (UI, comentarios que ameriten, documentos). Moneda MXN, fechas DD/MM/AAAA.
2. NO tocar el repo/carpeta de v1 mientras esté en producción. v2 vive en branch v2, carpeta separada.
3. Modelo de Claude: claude-sonnet-4-5-20250929. Structured outputs (output_config json_schema) siempre. Reintentos 429/529, escalado max_tokens 4096→8192.
4. El core (documentos/, ia/) es agnóstico de tipo. Toda lógica específica de tipo vive en tipos/.
5. Render on-brand con casa.css y fuentes auto-hospedadas. Nunca depender de internet.
6. Nunca commitear .env ni el SQLite con datos reales.
7. Versionar TODO documento en cada actualización (snapshot en DocumentoVersion antes de modificar).
```

---

## 16. Reglas No Negociables (para el builder)

1. **v1 es intocable** mientras corra en producción. Construir v2 en branch `v2`, carpeta local separada (`cremeria-hm-rh-v2/`).
2. **Core agnóstico de tipo.** `documentos/` e `ia/` jamás importan un tipo concreto. El Paso 9 (agregar Plan) es la prueba: si tocas el core, refactoriza el registry.
3. **Schema = single source of truth.** El mismo JSON Schema valida `contenido` y va a Claude. Los nombres de campos del Perfil se portan VERBATIM de v1.
4. **Render on-brand offline.** Fuentes auto-hospedadas, `casa.css` autoritativo, print CSS con saltos de página. El azul de v1 queda deprecated para documentos.
5. **Streaming async.** Generación vía `StreamingHttpResponse` SSE sección-por-sección, reusando la heurística de v1.
6. **Versionado universal.** Toda actualización guarda snapshot en `DocumentoVersion` y hace bump de `version`, para TODOS los tipos.
7. **Migración sin pérdida.** `migrar_v1` es idempotente, valida contra schema y preserva timestamps e historial. Verificar conteo antes del cutover.
8. **Secretos fuera de git.** `ANTHROPIC_API_KEY` y datos reales nunca se commitean.
9. **Idioma español, MXN, DD/MM/AAAA** en todo.
```
