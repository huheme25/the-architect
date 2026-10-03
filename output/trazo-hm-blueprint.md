# Trazo HM — Blueprint

> Generado por The Architect el 02/10/2026
> Archetype: internal-tool (herramienta personal, un solo usuario)
> Idioma del proyecto: español

---

## 1. Project Overview

### Vision

**Trazo HM** es un planificador de espacios 2D/3D de uso personal para Beto (Cremería HM). Permite bocetear la remodelación de un local comercial sin saber CAD: se traza el plano del local (desde cero con medidas, o calcando un DXF exportado de LibreCAD), se colocan muebles paramétricos de un catálogo orientado a cremería/oficina, se comparan varias distribuciones del mismo local, y se exporta cada variante como PNG/PDF con cotas para entregársela al arquitecto, quien la termina a detalle.

No es una herramienta profesional de arquitectura: es una **mesa de bocetaje visual** para comunicar ideas espaciales. Corre 100% local (un comando, se abre en el navegador). Al ser web, un despliegue futuro al VPS es trivial y está fuera del alcance de esta v1.

### Goals

- **G1 — Bocetear rápido**: desde abrir la app hasta tener un plano con muros y mobiliario colocado deben pasar minutos, no horas. Sin curva de aprendizaje CAD.
- **G2 — Calca DXF confiable**: el DXF de LibreCAD se muestra como fondo calibrado a escala real y se puede trazar encima. Nunca se interpreta ni se convierte automáticamente.
- **G3 — Variantes comparables**: varias distribuciones del mismo local (A, B, C…) que se duplican y alternan con un clic.
- **G4 — Entregable para el arquitecto**: export PNG/PDF por variante con cotas, nombre y fecha, listo para enviarse por WhatsApp/correo.
- **G5 — 3D sin esfuerzo**: la vista 3D se genera sola desde el plano 2D (muros extruidos + mobiliario como volúmenes). Cero trabajo extra por diseño.

### Success Metrics

- Beto crea un proyecto, calca su DXF real del local, traza los muros y coloca 10+ muebles en una sola sesión sin consultar documentación.
- Duplicar una variante y modificarla toma < 1 minuto.
- El PDF exportado se lee como plano: cotas legibles, escala indicada, cajetín con nombre/variante/fecha.
- Ctrl+Z funciona siempre y nunca se pierde trabajo (autosave a disco).
- La vista 3D abre en < 2 s y refleja fielmente el plano activo.

---

## 2. Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | **Vite 6 + React 18 (SPA)** | Un editor CAD es 99% cliente. Vite arranca instantáneo y no mete complejidad de servidor que aquí no aporta |
| Language | **TypeScript (strict)** | La geometría (cm, coordenadas, snaps) exige tipos fuertes para no degenerar en bugs de unidades |
| Editor 2D | **react-konva + Konva** | Canvas con capas, drag, transformadores, zoom/pan fluido con cientos de shapes. Hecho exactamente para esto |
| Vista 3D | **three.js + @react-three/fiber + @react-three/drei** | Extrusión de muros y cajas de mobiliario declarativas desde el mismo estado. OrbitControls gratis con drei |
| Calca DXF | **dxf-parser** | Parsea el DXF ASCII de LibreCAD a entidades (LINE, LWPOLYLINE, ARC, CIRCLE) que se dibujan como shapes Konva de fondo |
| Estado | **Zustand** + historial manual | Un solo store, sin boilerplate. Undo/redo con pila de snapshots de la variante activa (sin dependencias extra) |
| Estilos | **Tailwind CSS v4** | UI de herramienta densa y rápida de construir; tokens en CSS variables |
| Persistencia | **Express 4 (mini-API) + JSON en disco** | Cada proyecto es una carpeta en `data/projects/`. Versionable con git, respaldable, inmune a limpiezas del navegador |
| Export | **Konva `toDataURL` + jsPDF** | PNG en alta resolución desde el mismo canvas; jsPDF lo monta en hoja con cajetín |
| Testing | **Vitest** | La lógica de geometría (snap, muros, calibración DXF) es funciones puras perfectas para unit tests |
| Package Manager | **pnpm** | Estándar del ecosistema de Beto |

**Descartado a propósito**: Next.js (no hay SSR ni rutas de servidor que lo justifiquen), react-planner/blueprint3d (proyectos abandonados, React viejo — más caro adaptarlos que construir lo nuestro), conversión DXF→muros (frágil; la calca cubre la necesidad), base de datos real (JSON en disco basta para un usuario).

---

## 3. Directory Structure

```
trazo-hm/
  package.json                  # scripts: dev (concurrently vite+server), build, start, test
  vite.config.ts                # proxy /api → localhost:8310, alias @/
  tailwind.config / CSS v4      # tokens de diseño como CSS variables
  tsconfig.json                 # strict, paths @/*
  CLAUDE.md                     # guía del proyecto (sección 15)
  data/                         # ⚠ gitignored salvo .gitkeep — proyectos del usuario
    projects/
      <id>/
        project.json            # proyecto completo (meta + calca + variantes)
        calca.dxf               # DXF subido (si hay)
  server/
    index.ts                    # Express: estáticos (prod) + API REST de proyectos
    storage.ts                  # leer/escribir project.json y calca.dxf con validación
  src/
    main.tsx                    # bootstrap React + router
    App.tsx                     # rutas: / (proyectos) y /p/:id (editor)
    pages/
      ProjectsPage.tsx          # lista de proyectos: crear, abrir, renombrar, eliminar
      EditorPage.tsx            # shell del editor: toolbar + canvas + paneles + tabs variantes
    editor/
      store.ts                  # Zustand: proyecto, variante activa, tool, selección, historial, autosave
      history.ts                # pila undo/redo (snapshots JSON de la variante activa, tope 100)
      tools.ts                  # máquina de estados de herramientas (select/wall/door/window/furniture/text/calibrate)
      canvas/
        Canvas2D.tsx            # Stage Konva: capas grid → calca → muros → aberturas → mobiliario → anotaciones → cotas → selección
        GridLayer.tsx           # rejilla 5 cm / 1 m, adaptativa al zoom
        CalcaLayer.tsx          # entidades DXF como líneas/arcos grises con opacidad
        WallsLayer.tsx          # muros como polígonos con grosor; handles en vértices
        OpeningsLayer.tsx       # puertas (arco de abatimiento) y ventanas sobre muros
        FurnitureLayer.tsx      # rects rotables con etiqueta y color por categoría
        DimensionsLayer.tsx     # cotas automáticas de muros + dims del mueble seleccionado
        SelectionLayer.tsx      # marco de selección, transformer, snapping visual
      three/
        Scene3D.tsx             # Canvas R3F: piso, muros extruidos, mobiliario, OrbitControls, luces
        buildGeometry.ts        # variante → segmentos de muro partidos por aberturas + cajas de muebles
      geometry/
        units.ts                # TODO interno en CENTÍMETROS enteros; helpers cm↔m↔px
        snap.ts                 # snap a rejilla, a endpoints de muro, ortogonal con Shift
        walls.ts                # math de muros: dirección, normal, polígono con grosor, partir por aberturas
        dxf.ts                  # parse + normalización de entidades + transformación de calibración
      components/
        Toolbar.tsx             # herramientas + undo/redo + toggle 2D/3D + export
        VariantTabs.tsx         # tabs A/B/C: crear, duplicar, renombrar, eliminar
        CatalogPanel.tsx        # catálogo por categorías, click/drag para colocar
        PropertiesPanel.tsx     # propiedades del seleccionado (medidas, rotación, etiqueta, color)
        CalcaPanel.tsx          # subir DXF, opacidad, visible, calibrar escala
        ExportDialog.tsx        # PNG/PDF, tamaño de hoja, qué variante
      catalog/
        catalog.ts              # catálogo paramétrico (datos abajo, sección 4)
      export/
        exportPng.ts            # stage clonado a resolución de impresión + cajetín
        exportPdf.ts            # jsPDF A4/A3 horizontal con PNG + cajetín + escala
    lib/
      api.ts                    # cliente fetch del API (CRUD proyectos, DXF)
      ids.ts                    # nanoid
    types/
      model.ts                  # Project, Variant, Wall, Opening, FurnitureItem, Annotation, Calca
  tests/
    geometry.test.ts            # snap, walls, units
    dxf.test.ts                 # parse de un DXF fixture de LibreCAD
    storage.test.ts             # roundtrip leer/escribir proyecto
    fixtures/local.dxf          # DXF pequeño real de ejemplo
```

---

## 4. Data Model

Sin base de datos: **el agregado es el proyecto completo**, serializado como un solo `project.json`. Toda medida interna va en **centímetros enteros** (regla no negociable — evita errores de flotantes y de unidades).

### Entities

**Project**
| Field | Type | Notes |
|-------|------|-------|
| id | string | nanoid(10), es el nombre de la carpeta |
| nombre | string | "Local Centro — remodelación" |
| creado / actualizado | string ISO | `actualizado` lo pone el server al guardar |
| calca | Calca \| null | configuración del DXF de fondo |
| variantes | Variant[] | mínimo 1; orden = orden de tabs |
| varianteActivaId | string | última variante abierta |

**Calca**
| Field | Type | Notes |
|-------|------|-------|
| archivo | "calca.dxf" | el DXF vive junto al project.json |
| escala | number | factor unidades-DXF → cm (lo fija la calibración) |
| offset | {x, y} cm | traslación de la calca en el mundo |
| rotacion | number | grados; default 0 |
| opacidad | number | 0–1, default 0.5 |
| visible | boolean | toggle rápido |

**Variant**
| Field | Type | Notes |
|-------|------|-------|
| id | string | nanoid |
| nombre | string | "A — vitrinas en L" |
| muros | Wall[] | |
| aberturas | Opening[] | referencian muro por id |
| mobiliario | FurnitureItem[] | |
| anotaciones | Annotation[] | textos libres sobre el plano |

> Decisión: **cada variante es una copia completa** (muros incluidos). "Duplicar variante" clona todo. Es más simple que un cascarón compartido y además correcto: en una remodelación, tirar o mover un muro ES parte de la variante.

**Wall** — segmento recto
| Field | Type | Notes |
|-------|------|-------|
| id | string | |
| p1, p2 | {x, y} cm | endpoints en coordenadas de mundo |
| grosor | number cm | default 15 |
| altura | number cm | default 280 (solo afecta al 3D) |

**Opening** — puerta o ventana sobre un muro
| Field | Type | Notes |
|-------|------|-------|
| id | string | |
| muroId | string | FK a Wall; si el muro se borra, sus aberturas también |
| tipo | "puerta" \| "ventana" | |
| offset | number cm | distancia desde p1 del muro al inicio de la abertura |
| ancho | number cm | default puerta 90, ventana 120 |
| alto | number cm | default puerta 210, ventana 120 |
| elevacion | number cm | 0 en puertas; alféizar en ventanas (default 90) |
| abate | "izq" \| "der" \| null | lado del arco de abatimiento (solo puertas) |

**FurnitureItem**
| Field | Type | Notes |
|-------|------|-------|
| id | string | |
| catalogoId | string | referencia al item del catálogo (para icono/categoría) |
| etiqueta | string | editable; default el nombre del catálogo |
| x, y | number cm | centro del mueble |
| rotacion | number | grados, pasos de 15 con snap (libre con Alt) |
| ancho, fondo, alto | number cm | SIEMPRE editables — paramétrico |
| color | string \| null | override opcional del color de categoría |

**Annotation**
| Field | Type | Notes |
|-------|------|-------|
| id, x, y | | posición en mundo |
| texto | string | |
| tamano | number | pt en papel; default 14 |

### Catálogo inicial (datos en `src/editor/catalog/catalog.ts`)

Paramétrico: cada item define medidas default (ancho×fondo×alto en cm) y color de categoría; el usuario las ajusta al colocar.

| Categoría (color) | Items (ancho×fondo×alto default) |
|---|---|
| **Refrigeración** (azul frío) | Vitrina refrigerada 150×80×120 · Vitrina cremería 200×90×130 · Congelador horizontal 150×70×90 · Refrigerador vertical 70×75×200 · Cámara fría 200×200×220 |
| **Venta** (ámbar) | Mostrador 150×60×90 · Caja/checkout 120×60×90 · Báscula sobre mesa 40×40×15 · Exhibidor 100×50×150 |
| **Almacenaje** (gris) | Estantería 100×40×180 · Góndola central 120×60×140 · Rack 240×60×200 · Tarima 120×100×15 |
| **Oficina** (verde) | Escritorio 140×70×75 · Silla 50×50×90 · Archivero 50×60×130 · Mesa de juntas 200×100×75 · Librero 90×30×180 |
| **Generales** (neutro) | Mesa 80×80×75 · Lavabo 50×45×85 · WC 40×65×75 · Puerta de cristal (anot.) · **Bloque genérico** 100×100×100 |

El catálogo es un array TS tipado — agregar el mobiliario exacto de HM después es editar ese archivo, sin tocar código del editor.

### Database Schema

No aplica (archivos JSON). El contrato es el tipo `Project` de `src/types/model.ts`, validado en `server/storage.ts` al leer/escribir (campos requeridos + números finitos; si un project.json está corrupto, el server responde 422 y NO lo sobreescribe).

---

## 5. API Design

Express en `localhost:8310`. En dev, Vite hace proxy de `/api`. Sin auth (local, un usuario).

### Routes Overview
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/projects | Lista (id, nombre, actualizado, nº variantes) | no |
| POST | /api/projects | Crea proyecto `{nombre}` → devuelve Project con 1 variante vacía "A" | no |
| GET | /api/projects/:id | Project completo | no |
| PUT | /api/projects/:id | Guarda Project completo (autosave) | no |
| DELETE | /api/projects/:id | Elimina carpeta (confirmación en UI) | no |
| PUT | /api/projects/:id/dxf | Body = texto DXF (límite 25 MB) → escribe `calca.dxf` | no |
| GET | /api/projects/:id/dxf | Devuelve el texto DXF crudo | no |

### Key Endpoints Detail

**PUT /api/projects/:id** — el caballo de batalla (autosave con debounce de 2 s desde el store).
- Request: `Project` completo. Response: `{ok: true, actualizado}`.
- Validación: estructura mínima + toda medida es número finito ≥ 0. Si falla → 422 con detalle, y el archivo en disco queda intacto.
- Escritura atómica: escribir a `project.json.tmp` y renombrar — nunca un save a medias corrompe el proyecto.

**PUT /api/projects/:id/dxf**
- `Content-Type: text/plain` (el cliente lee el archivo con FileReader y manda el texto).
- El server valida que contenga `SECTION`/`ENTITIES` antes de guardar; si no, 422 "no parece un DXF".
- Al guardar, si `project.calca` es null el cliente la inicializa con defaults y abre el flujo de calibración.

**Errores**: shape única `{ok: false, error: string}` con status 4xx/5xx. El cliente muestra toast y, en fallo de autosave, un indicador persistente "⚠ sin guardar" + reintento.

---

## 6. Frontend Architecture

### Pages / Routes
| Route | Page | Description |
|-------|------|-------------|
| / | ProjectsPage | Tarjetas de proyectos: crear, abrir, renombrar, eliminar |
| /p/:id | EditorPage | El editor completo (2D/3D, variantes, catálogo, export) |

### Component Hierarchy (EditorPage)
```
EditorPage
├── Toolbar                    # izquierda→derecha: select, muro, puerta, ventana, texto | undo/redo | 2D/3D | export
├── VariantTabs                # A · B · C · [+ duplicar]
├── <main>
│   ├── CatalogPanel           # izquierda, colapsable; categorías → items → click coloca en centro de vista
│   ├── Canvas2D | Scene3D     # centro; se montan en exclusiva según el toggle
│   └── PropertiesPanel        # derecha; contextual al seleccionado (muro/abertura/mueble/texto) + CalcaPanel
└── StatusBar                  # zoom, coords del cursor (m), estado de guardado, hint de la herramienta activa
```

### State Management

- **Un store Zustand** (`editor/store.ts`): `project`, `varianteActivaId`, `tool`, `seleccion` (ids), `camara2D` (pan/zoom), `historial`.
- **Mutaciones solo vía acciones del store** que (1) aplican el cambio inmutable, (2) empujan snapshot al historial, (3) agendan autosave (debounce 2 s).
- **Historial**: pila de snapshots JSON de la **variante activa** (tope 100). Cambiar de variante limpia el historial — simple y predecible.
- **Derivados, nunca estado**: cotas, polígonos de muro con grosor, geometría 3D — todo se calcula de `muros/aberturas/mobiliario` en render (memoizado). Una sola fuente de verdad.
- La calca DXF parseada se cachea en memoria (`useRef`) fuera del store — es grande e inmutable; en el store solo vive su configuración (`Calca`).

### Interacciones clave del canvas (contrato de UX)

- **Zoom** con rueda (al cursor), **pan** con botón medio o Space+drag.
- **Snap**: rejilla 5 cm siempre; endpoints de muros (imán 10 px de pantalla); Shift fuerza ortogonal al dibujar muros. Indicador visual del punto de snap.
- **Muro**: click-click-click polilínea, Esc/Enter termina, doble click cierra contra el primer punto.
- **Puerta/ventana**: con la herramienta activa, hover sobre un muro muestra fantasma; click inserta; drag desliza sobre el muro.
- **Mueble**: click en catálogo → aparece en el centro de la vista pegado al cursor → click suelta. R rota 15°; drag con snap de bordes contra muros y otros muebles (tolerancia 5 cm).
- **Supr** borra selección; **Ctrl+D** duplica; **Ctrl+Z/Ctrl+Shift+Z** undo/redo; **Esc** vuelve a select.
- **Cotas automáticas**: longitud sobre cada muro (en m, 2 decimales); el mueble seleccionado muestra ancho×fondo.

### Calibración de calca (flujo exacto)

1. Subes el DXF → se dibuja con escala tentativa 1 unidad = 1 cm.
2. Botón "Calibrar": clickeas dos puntos sobre la calca (ej. extremos de un muro que sabes que mide 8 m), escribes la distancia real.
3. `escala = distanciaReal / distanciaDibujada`; se recalcula y la calca queda a escala del mundo. Offset se ajusta arrastrando la calca con la herramienta de calca activa.

---

## 7. Design System

> Autosuficiente: los diseñadores del builder consumen esto directo.

### Brand Origin & Register
- **Brand origin:** from-scratch — dirección confirmada con el cliente: herramienta de precisión personal, SIN marca Cremería HM en la interfaz (es su mesa de trabajo, no un producto de la empresa).
- **Register:** product — el diseño sirve al plano. La vara: que un usuario fluido de Figma/LibreCAD se sienta en casa en 5 minutos.
- **Direction:** *Mesa de dibujo técnico, no dashboard.* El lienzo es un papel técnico claro y frío donde el plano es el protagonista absoluto; el cromo (toolbar, paneles) es grafito oscuro que se retira a los bordes, como los márgenes de una mesa de luz. Un solo acento azul de selección — el resto del color del canvas lo ponen las categorías del mobiliario, que funcionan como código de plano, no como decoración. No colapsa en el cliché porque rechaza las dos salidas fáciles de la categoría: ni el dark-mode total de "app CAD futurista", ni el SaaS crema con cards — es papel frío + grafito, la paleta de una mesa de dibujo real.
- **Dials:** VARIANCE low · MOTION_INTENSITY low · DENSITY high

### Colors (OKLCH)
| Role | OKLCH | Usage |
|------|-------|-------|
| Canvas (papel) | `oklch(0.975 0.004 240)` | fondo del lienzo 2D — blanco técnico frío, jamás crema |
| Grid menor/mayor | `oklch(0.93 0.005 240)` / `oklch(0.86 0.008 240)` | rejilla 5 cm / 1 m |
| Trazo de plano | `oklch(0.28 0.012 250)` | muros, cotas, texto sobre el canvas |
| Calca DXF | `oklch(0.68 0.01 250)` @ opacidad 0.5 | líneas del DXF de fondo |
| Chrome bg | `oklch(0.22 0.014 250)` | toolbar, paneles, tabs, status bar |
| Surface | `oklch(0.27 0.014 250)` | inputs, hover, cards dentro de paneles |
| Ink (sobre chrome) | `oklch(0.93 0.008 250)` | texto principal del cromo (contraste ≥ 7:1) |
| Muted | `oklch(0.68 0.015 250)` | labels secundarios (≥ 4.5:1 sobre chrome) |
| Accent | `oklch(0.62 0.17 255)` | selección, herramienta activa, botón primario, handles |
| Destructive | `oklch(0.58 0.21 25)` | eliminar variante/proyecto |
| Success | `oklch(0.64 0.14 150)` | "guardado", confirmaciones |
| Cat. Refrigeración | `oklch(0.75 0.09 230)` | relleno de muebles @ 0.85 + borde del trazo |
| Cat. Venta | `oklch(0.80 0.11 75)` | ídem |
| Cat. Almacenaje | `oklch(0.75 0.02 250)` | ídem |
| Cat. Oficina | `oklch(0.76 0.10 155)` | ídem |
| Cat. Generales | `oklch(0.82 0.01 250)` | ídem |

- **Color strategy:** restrained — un acento; el color "de verdad" es semántico (categorías sobre el plano).

### Typography
| Role | Font | Size | Weight |
|------|------|------|--------|
| UI (cromo) | **Inter** (variable) | 13px base; headings 14–16px | 400/500/600 |
| Cotas, coords, medidas | **JetBrains Mono** | 11–12px en pantalla; 9pt en papel | 400/500 |
| Cajetín de export | Inter 600 + JetBrains Mono | título 14pt / datos 9pt | — |

- Pareja en eje de contraste humanista-sans + mono técnica: toda cifra de medida va SIEMPRE en mono — es el sabor "instrumento de precisión" de la app.

### Spacing & Layout
- Base 4px: 4, 8, 12, 16, 24, 32. Paneles laterales 280px (catálogo) / 300px (propiedades), colapsables.
- Radius: 6px controles, 8px diálogos, 0 en el canvas y sus overlays.
- Z-index semántico: canvas-overlays → dropdown → modal-backdrop → modal → toast → tooltip.
- Breakpoints: desktop-first (es una herramienta de escritorio); mínimo soportado 1280px; en menos, los paneles colapsan a iconos.

### Motion
MOTION_INTENSITY low — sin motion de autor. Solo transiciones funcionales de 100–150 ms ease-out en hover/colapso de paneles. `prefers-reduced-motion` las apaga. Nada se anima en el canvas (el feedback ahí es inmediato, frame a frame).

### Component Style & Banned Anti-Patterns
- **Aesthetic:** plano y denso; bordes 1px `oklch(0.34 0.014 250)` sobre chrome; sin sombras salvo diálogos (una sola, suave); iconografía de 16px tipo Lucide, trazo 1.5.
- **Banned (AI slop):** side-stripe borders, gradient text, glassmorphism, hero-metric template, grids de cards idénticas, eyebrows uppercase por sección, marcadores 01/02/03, headings desbordados. Además, prohibido aquí: skeuomorfismo de "plano arquitectónico vintage" (texturas de papel, sellos) y cualquier branding HM en el cromo.

---

## 8. Authentication & Authorization

**No hay.** App local de un solo usuario; el server escucha en `localhost` únicamente (bind a `127.0.0.1` explícito — regla no negociable). Si algún día se despliega al VPS, se le antepone Cloudflare Access como al resto del ecosistema HM — fuera de alcance de v1.

---

## 9. Build Order

> Cada paso termina con la app corriendo y lo nuevo demostrable. **Checkpoint clave: al cerrar el paso 7 Beto ya puede bocetear de verdad** (muros + calca + mobiliario + guardado); del 8 en adelante es completar la experiencia.

**Step 1 — Scaffolding**
`pnpm create vite trazo-hm --template react-ts`; instalar Tailwind v4, react-router-dom, zustand, nanoid; estructura de carpetas de la sección 3; tokens CSS del design system en `:root`; Express mínimo en `server/` con `GET /api/health`; scripts: `dev` (concurrently: vite + tsx watch server), `build`, `start` (server sirve `dist/` + API en 8310), `test` (vitest). Verificar: `pnpm dev` levanta ambos y `/api/health` responde vía proxy.

**Step 2 — Modelo + persistencia + pantalla de proyectos**
Tipos de `model.ts`; `server/storage.ts` (escritura atómica, validación 422); endpoints CRUD de proyectos; `lib/api.ts`; ProjectsPage completa (crear/abrir/renombrar/eliminar con confirmación). Tests: roundtrip de storage. Verificar: crear proyecto desde la UI y ver su carpeta en `data/projects/`.

**Step 3 — Lienzo 2D: mundo, grid, cámara**
Canvas2D con Konva: sistema de coordenadas (1 unidad = 1 cm; eje Y hacia abajo está bien, documentarlo), zoom a cursor con límites (0.05x–20x), pan con botón medio/Space, GridLayer adaptativa (5 cm desaparece al alejarse, 1 m persiste), StatusBar con coords en metros y zoom. `geometry/units.ts` + `snap.ts` con tests. Verificar: navegar un mundo vacío se siente fluido.

**Step 4 — Muros + cotas + undo/redo**
Herramienta muro (polilínea click-click, Esc/Enter/doble-click, Shift ortogonal, snap a grid y endpoints); `geometry/walls.ts` (polígono con grosor) con tests; selección y edición (mover muro, arrastrar vértices, grosor/altura en PropertiesPanel); DimensionsLayer con cota por muro; `history.ts` + Ctrl+Z/Ctrl+Shift+Z; Supr borra. Autosave activado desde aquí (debounce 2 s + indicador en StatusBar). Verificar: trazar el contorno de un local de 12×8 m en < 1 min, undo/redo coherente tras 20 operaciones mezcladas.

**Step 5 — Calca DXF**
`geometry/dxf.ts`: parse con dxf-parser, normalizar LINE/LWPOLYLINE/POLYLINE/ARC/CIRCLE a primitivas propias (ignorar lo demás sin romper, reportando "N entidades omitidas"); endpoints de DXF; CalcaPanel (subir, opacidad, visible, arrastrar para offset); flujo de calibración de 2 puntos (sección 6). Test con `fixtures/local.dxf` real de LibreCAD. Verificar: subir el DXF del local de Beto, calibrarlo con una medida conocida y trazar un muro encima que coincida.

**Step 6 — Puertas y ventanas**
Opening sobre muros: herramienta con fantasma en hover, insertar, deslizar sobre el muro, editar medidas en propiedades; render 2D (puerta: hueco en el muro + arco de abatimiento con lado izq/der; ventana: hueco con doble línea); borrar muro borra sus aberturas. `walls.ts` gana `splitWallByOpenings()` (lo reutiliza el 3D) con tests. Verificar: puerta de 90 que se desliza sin salirse del muro y cambia de abatimiento.

**Step 7 — Catálogo y mobiliario** ✦ checkpoint "ya se puede bocetear"
`catalog.ts` con las 5 categorías; CatalogPanel; colocar (click catálogo → fantasma al cursor → click suelta), mover con snap de bordes (muros y muebles, 5 cm), rotar con R (15°, libre con Alt) y con transformer, duplicar Ctrl+D, redimensionar ancho/fondo/alto + etiqueta + color en propiedades; dims del seleccionado en el canvas. Verificar: amueblar un piso de venta (8+ piezas) en una sesión fluida; recargar la página y que todo esté ahí.

**Step 8 — Variantes**
VariantTabs: crear vacía, **duplicar** (clon profundo con ids nuevos), renombrar inline, eliminar con confirmación (nunca la última), cambiar con un clic (< 100 ms, historial se resetea). Verificar: A con vitrinas en L, duplicar a B, mover todo a isla central, alternar A/B sin fugas de estado.

**Step 9 — Export PNG/PDF**
`exportPng.ts`: clonar el stage fuera de pantalla a pixelRatio de impresión, encuadre automático al contenido + margen, fondo blanco, calca excluible (toggle), cajetín (proyecto, variante, fecha, escala gráfica de 1 m). `exportPdf.ts`: jsPDF A4/A3 horizontal con el PNG ajustado + cajetín. ExportDialog con preview. Verificar: PDF A3 de una variante legible impreso — cotas y etiquetas nítidas.

**Step 10 — Vista 3D**
`buildGeometry.ts` (muros partidos por aberturas → cajas; dintel sobre puertas, antepecho+dintel en ventanas; muebles → cajas con color de categoría y arista marcada); Scene3D (piso del bounding de muros, luz ambiente + direccional suave, OrbitControls con target al centro del plano); toggle 2D/3D instantáneo (el 3D se regenera al entrar). Sin texturas ni sombras caras en v1. Verificar: local con 2 puertas, 1 ventana y 10 muebles orbitando a 60 fps.

**Step 11 — Pulido**
Atajos documentados en un panel "?" ; estados vacíos (proyecto nuevo → hint "traza tu primer muro o sube tu DXF"); manejo de errores de API con toasts + reintento de autosave; confirmaciones destructivas consistentes; revisión del design system contra la sección 7 (colores, mono en cifras, densidad); barrido de performance (memoización de capas, `listening=false` en capas estáticas). Verificar: sesión completa de 30 min sin fricción ni consola con errores.

**Step 12 — Cierre**
README (cómo arrancar, dónde viven los datos, cómo respaldar `data/`); suite de tests verde; `pnpm build && pnpm start` sirve todo en 8310; commit y tag `v1.0`. **Fuera de alcance explícito**: deploy VPS, export DXF, conversión DXF→muros, renders fotorrealistas, multiusuario.

---

## 10. Environment Setup

### Prerequisites
- Node.js ≥ 20 LTS
- pnpm ≥ 9
- Navegador Chromium/Firefox reciente (WebGL2 para el 3D)

### Environment Variables
| Variable | Description | Where to Get |
|----------|-------------|--------------|
| `PORT` | Puerto del server Express | opcional; default **8310** |
| `DATA_DIR` | Carpeta de proyectos | opcional; default `./data` |

Sin secretos, sin servicios externos, sin `.env` obligatorio.

### Initial Setup Commands
```bash
pnpm create vite trazo-hm --template react-ts
cd trazo-hm
pnpm add react-router-dom zustand nanoid konva react-konva three @react-three/fiber @react-three/drei dxf-parser jspdf express
pnpm add -D tailwindcss @tailwindcss/vite tsx concurrently vitest @types/express @types/three
pnpm dev   # levanta Vite (5173, proxy /api) + Express (8310)
```

---

## 11. Dependencies

### Core
| Package | Purpose |
|---------|---------|
| react / react-dom | UI |
| react-router-dom | 2 rutas (proyectos, editor) |
| konva + react-konva | editor 2D en canvas |
| three + @react-three/fiber + @react-three/drei | vista 3D autogenerada |
| zustand | estado del editor |
| dxf-parser | parseo del DXF de LibreCAD |
| jspdf | export PDF |
| express | API local + servir build |
| nanoid | ids |

### Dev
| Package | Purpose |
|---------|---------|
| vite + @vitejs/plugin-react | build/dev del frontend |
| typescript | strict en todo (frontend y server) |
| tailwindcss + @tailwindcss/vite | estilos v4 |
| tsx | correr/watch el server TS sin compilar |
| concurrently | `pnpm dev` levanta vite + server |
| vitest | tests de geometría, DXF y storage |

---

## 12. Deployment Strategy

### Hosting
**v1: ninguno.** App local en la PC de Beto. `pnpm dev` para desarrollo; para uso diario, `pnpm build && pnpm start` deja todo servido por Express en `http://localhost:8310` (un solo proceso). El server hace bind a `127.0.0.1`.

### CI/CD
No aplica en v1. Repo git local con remote privado en GitHub (`huheme25/trazo-hm`) desde el día 1 — los **datos** (`data/`) van gitignored; si Beto quiere versionar sus diseños, `data/` puede ser su propio repo aparte (mismo patrón que su memoria).

### Ruta futura (documentada, no construida)
Dockerfile de dos etapas (build Vite → Node runtime) + compose en el patrón `templates/app-stack-windows/` del ecosistema HM, detrás del tunnel CF con Cloudflare Access. El diseño actual (SPA + API + archivos en un volumen) lo permite sin reescritura.

### Environments
Uno solo (local). `DATA_DIR` permite apuntar a otra carpeta para experimentar sin tocar los proyectos reales.

---

## 13. Testing Strategy

### Unit Tests (Vitest — el grueso)
- `geometry/units.ts`: conversiones cm↔m↔px, redondeos.
- `geometry/snap.ts`: snap a grid, a endpoints, ortogonal.
- `geometry/walls.ts`: polígono con grosor, `splitWallByOpenings` (casos: abertura al borde, dos aberturas, abertura que no cabe).
- `geometry/dxf.ts`: fixture real de LibreCAD → nº de entidades esperado, entidades no soportadas se omiten sin lanzar, transformación de calibración.
- `history.ts`: push/undo/redo/tope.

### Integration Tests
- `server/storage.ts`: roundtrip crear→guardar→leer, escritura atómica (el .tmp nunca queda), JSON corrupto → 422 sin sobreescribir.

### E2E Tests
- Un smoke con la skill `webapp-testing` (Playwright) al cerrar el paso 11: crear proyecto → trazar 4 muros → colocar 2 muebles → duplicar variante → export PNG descarga. No más — es una herramienta personal; el E2E pesado no paga renta aquí.

---

## 14. Skills to Use During Build

| Skill | When to Use | Why |
|-------|-------------|-----|
| `/frontend-design` | Steps 2, 11 (shell de la UI, pulido visual) | Cromo de herramienta distintivo, no genérico |
| `/ui-ux-pro-max` | Step 11 si hay dudas de contraste/jerarquía | Validar la ejecución del design system |
| `webapp-testing` | Steps 7, 9, 11 (checkpoint, export, smoke E2E) | Verificar flujos reales en navegador con Playwright |
| `/shadcn-ui` | NO usar | Los paneles son custom-densos; shadcn metería estética SaaS que la sección 7 prohíbe |

---

## 15. CLAUDE.md for Target Project

```markdown
# Trazo HM

Planificador 2D/3D local para bocetear la remodelación de locales de Cremería HM.
Un usuario (Beto), sin auth, datos como JSON en disco. Idioma del proyecto: español.

## Commands

- `pnpm dev` — Vite (5173, proxy /api) + Express (8310) con hot reload
- `pnpm build` — build de producción del frontend
- `pnpm start` — Express sirve dist/ + API en http://localhost:8310 (uso diario)
- `pnpm test` — Vitest (geometría, DXF, storage)

## Tech Stack

Vite + React 18 + TypeScript strict + Tailwind v4 + react-konva (2D) +
react-three-fiber (3D) + Zustand + dxf-parser + jsPDF + Express (JSON en disco).

## Architecture

- `server/` — Express: CRUD de proyectos sobre `data/projects/<id>/project.json` + `calca.dxf`.
  Escritura atómica (tmp+rename), validación al leer/escribir, bind SOLO a 127.0.0.1.
- `src/editor/store.ts` — ÚNICO store Zustand. Toda mutación pasa por acciones del store:
  cambio inmutable → snapshot al historial → autosave (debounce 2 s).
- `src/editor/geometry/` — funciones PURAS (units, snap, walls, dxf). Aquí vive la lógica
  testeable; los componentes Konva/R3F solo renderizan estado derivado.
- `src/editor/canvas/` — capas Konva en orden: grid → calca → muros → aberturas →
  mobiliario → anotaciones → cotas → selección.
- `src/editor/three/buildGeometry.ts` — variante → geometría 3D. El 3D SIEMPRE se deriva
  del 2D; jamás tiene estado propio.
- Datos: el agregado es `Project` (src/types/model.ts). Variantes son copias completas
  (muros incluidos). El catálogo es un array TS en `src/editor/catalog/catalog.ts`.

## Code Organization Rules

1. **Toda medida interna en CENTÍMETROS enteros.** m/px solo en bordes de UI vía `units.ts`.
2. Un componente por archivo, máx 300 líneas. Alias `@/` para `src/`.
3. Geometría = funciones puras con tests en `tests/`. Nada de math inline en componentes.
4. Sin barrel exports. Sin `any` (strict). Comentarios en español solo donde ameriten.
5. Capas Konva estáticas con `listening={false}`; memoizar derivados caros.

## Design System (resumen — la fuente es la sección 7 del blueprint)

- Canvas papel frío `oklch(0.975 0.004 240)`; cromo grafito `oklch(0.22 0.014 250)`;
  acento único `oklch(0.62 0.17 255)`; categorías de mobiliario con color semántico.
- Inter 13px para UI; **JetBrains Mono para TODA cifra de medida** (cotas, coords, dims).
- Denso, plano, bordes 1px, radius 6px, motion mínimo. Prohibido: branding HM en el cromo,
  estética SaaS-crema, glassmorphism, texturas de papel vintage.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `PORT` | Puerto Express (default 8310) |
| `DATA_DIR` | Carpeta de datos (default ./data) |

## Reglas No Negociables

1. `data/` va gitignored — son los diseños del usuario, no código.
2. El server NUNCA sobreescribe un project.json que no valida (422 y fuera).
3. Ctrl+Z siempre operativo: ninguna acción del editor muta sin pasar por el historial.
4. El 3D se deriva del 2D — cero estado propio en la escena.
5. Express bind a 127.0.0.1 exclusivamente (app local).
6. La calca DXF es referencia visual: NUNCA convertirla en muros automáticamente.
```

---

## 16. Reglas No Negociables

1. **Centímetros enteros en todo el modelo.** Conversión a m/px solo en la frontera de UI (`units.ts`). Un bug de unidades aquí invalida la app entera.
2. **TypeScript strict, cero `any`.** La geometría tipada es la red de seguridad.
3. **Ninguna mutación fuera del store**, y toda acción del editor empuja al historial antes del autosave. Si Ctrl+Z se rompe, el paso no cierra.
4. **Escritura atómica en disco** (tmp + rename) y validación antes de sobreescribir. Los diseños de Beto no se corrompen jamás.
5. **La calca DXF nunca se interpreta** — es fondo visual calibrable. Resistir la tentación de "detectar muros".
6. **El 3D es un derivado puro** de la variante activa. Nada se edita en 3D en v1.
7. **`data/` gitignored**; el repo versiona código, no diseños.
8. **Server local-only** (bind 127.0.0.1). El deploy futuro es otra fase con Access delante.
9. **Fidelidad a la sección 7**: cifras en mono, canvas papel frío, cromo grafito, un acento. Sin branding HM, sin estética SaaS.
10. **Cada paso del build order termina demostrable** en el navegador; el paso 7 es el checkpoint de "ya se puede bocetear" y se verifica con una sesión real de bocetaje.
