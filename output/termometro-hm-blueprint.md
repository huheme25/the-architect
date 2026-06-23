# Termómetro HM — Blueprint

> Generado por The Architect el 2026-05-05
> Archetype: Internal Tool
> Cliente: Cremería HM — Proyecto de Mejora Continua
> Proyecto: medición de NPS por cajera en mostrador, vía tablet fija

---

## 1. Project Overview

### Vision
**Termómetro HM** es una app interna de Cremería HM para medir el NPS (Net Promoter Score) de cada cajera. Vive en una tablet fija en el mostrador. La cajera se loguea con PIN al inicio de su turno y, después de cada cliente, le voltea la tablet para que vote (5 caritas) y deje un comentario opcional. El admin (Humberto / Beto) consulta resultados por cajera, lee comentarios, ve tendencia semanal y exporta CSV.

Es deliberadamente pequeña: 6 cajeras totales, una sola tablet, hosting gratuito, costo $0/mes. No es un módulo de EspritOS — vive aparte para no agregar fricción al ERP en construcción.

### Goals
- Capturar feedback granular del cliente sin que la cajera lo manipule (la tablet vive en modo kiosko)
- Permitir comparativa objetiva entre las 6 cajeras (NPS por persona, comentarios, tendencia)
- Cero fricción técnica — funciona desde el día 1 sin entrenamiento más allá de "este es tu PIN"
- Datos exportables a CSV para llevarlos a Excel / cruzar con métricas operativas

### Success Metrics
- ≥ 30 encuestas/día capturadas (rotación normal del mostrador)
- ≥ 90% de encuestas con caritas (comentario es opcional, score no)
- Latencia visible < 200ms al tocar una carita (UX táctil debe sentirse instantánea)
- 0 caídas durante horario de mostrador (L-S 7am-4pm)

---

## 2. Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | **Next.js 15** (App Router) | SSR para login, RSC para admin dashboard, instalable como PWA |
| Language | **TypeScript** strict | Type-safety end-to-end, ya es el estándar de Beto |
| Styling | **Tailwind CSS v4** | Velocidad, consistencia, control total |
| Components | **shadcn/ui** | Copia componentes a tu repo, no es dependencia |
| Database | **PostgreSQL** vía **Neon** (free tier) | Free serverless, suficiente para 6 cajeras × ~50 encuestas/día |
| ORM | **Prisma** | Beto ya lo usa en CRM-ERP, type-safe migrations |
| Auth | **PIN custom** + **iron-session** | 6 usuarios no justifican Clerk/NextAuth. PIN de 4 dígitos hasheado con bcrypt + cookies httpOnly |
| Charts | **Recharts** | NPS por cajera (barras), tendencia semanal (línea) |
| PWA | **next-pwa** | Manifest + service worker mínimo para "Add to Home Screen" en tablet |
| Validación | **Zod** | Schemas de input compartidos entre client y server actions |
| Hosting | **Vercel** (free tier) | Deploy con `git push`, preview deploys automáticos |
| Package Manager | **pnpm** | Más rápido que npm, mismo que el resto del workspace |

**Costo total: $0/mes** (Neon free tier 0.5GB + Vercel hobby 100GB transfer)

---

## 3. Directory Structure

```
termometro-hm/
├── prisma/
│   ├── schema.prisma                   # Modelo de datos (User, Survey)
│   ├── seed.ts                         # Inserta 6 cajeras + 1 admin con PINs hasheados
│   └── migrations/                     # Generadas por prisma migrate
├── public/
│   ├── manifest.json                   # PWA manifest (instalable en tablet)
│   ├── icon-192.png                    # Ícono PWA
│   ├── icon-512.png                    # Ícono PWA
│   └── logo-cremeria.svg               # Logo Cremería HM
├── src/
│   ├── app/
│   │   ├── (kiosko)/                   # Layout kiosko: pantalla completa, sin chrome
│   │   │   ├── login/page.tsx          # Selección de cajera + teclado numérico PIN
│   │   │   ├── encuesta/
│   │   │   │   ├── page.tsx            # 5 caritas grandes (paso 1)
│   │   │   │   ├── comentario/page.tsx # Textarea + skip + enviar (paso 2)
│   │   │   │   └── gracias/page.tsx    # "Que tengas, un cremoso día" (paso 3, auto-redirect)
│   │   │   └── layout.tsx              # Layout kiosko con wake-lock + prevent-zoom
│   │   ├── admin/                      # Layout admin: sidebar + header
│   │   │   ├── login/page.tsx          # Login admin (mismo PIN, role=admin)
│   │   │   ├── dashboard/page.tsx      # NPS por cajera, KPIs, distribución
│   │   │   ├── comentarios/page.tsx    # Lista filtrable
│   │   │   ├── tendencia/page.tsx      # Gráfica de líneas semanal
│   │   │   ├── export/route.ts         # GET → CSV stream
│   │   │   └── layout.tsx              # Sidebar + header
│   │   ├── api/
│   │   │   ├── auth/login/route.ts     # POST PIN → set session cookie
│   │   │   ├── auth/logout/route.ts    # POST → clear session
│   │   │   └── encuesta/route.ts       # POST {score, comment} → guarda Survey
│   │   ├── layout.tsx                  # Root layout (fonts, providers, manifest)
│   │   └── globals.css                 # Tailwind directives + paleta Cremería HM
│   ├── components/
│   │   ├── ui/                         # shadcn primitives (button, input, table, etc.)
│   │   ├── kiosko/
│   │   │   ├── PinKeypad.tsx           # Teclado numérico grande (10 botones)
│   │   │   ├── CajeraPicker.tsx        # Avatar/nombre por cajera, grid de 6
│   │   │   ├── FaceButton.tsx          # Botón gigante con carita SVG + label
│   │   │   ├── FaceRow.tsx             # Las 5 FaceButton en fila
│   │   │   └── CommentBox.tsx          # Textarea + skip + enviar
│   │   ├── admin/
│   │   │   ├── NpsCard.tsx             # Card con NPS grande + delta vs semana anterior
│   │   │   ├── NpsByCashierTable.tsx   # Tabla con NPS por cajera + barras inline
│   │   │   ├── NpsTrendChart.tsx       # Recharts LineChart
│   │   │   ├── ScoreDistribution.tsx   # Histograma de scores 1-5
│   │   │   ├── CommentList.tsx         # Lista con filtros aplicados
│   │   │   ├── DateRangePicker.tsx     # Selector de rango
│   │   │   └── CashierFilter.tsx       # Multi-select de cajeras
│   │   └── shared/
│   │       └── KioskoGuard.tsx         # Previene back/swipe accidental
│   ├── lib/
│   │   ├── db.ts                       # PrismaClient singleton
│   │   ├── session.ts                  # iron-session config + helpers (getSession, requireRole)
│   │   ├── auth.ts                     # bcrypt verify + create user
│   │   ├── nps.ts                      # calcularNPS(scores[]), agruparPorCajera, etc.
│   │   ├── csv.ts                      # buildCsv(surveys[]) → string
│   │   └── utils.ts                    # cn() de shadcn, formatDate, etc.
│   ├── types/
│   │   └── index.ts                    # SurveyScore, NpsData, etc.
│   └── middleware.ts                   # Protege /admin/* y /encuesta/* según rol
├── tests/
│   └── e2e/
│       ├── flujo-cajera.spec.ts        # Login → vota → comenta → gracias → vuelve
│       └── admin-export.spec.ts        # Login admin → export CSV → valida headers
├── .env.example                        # Variables de entorno documentadas
├── .gitignore
├── next.config.ts
├── package.json
├── pnpm-lock.yaml
├── postcss.config.mjs
├── tailwind.config.ts
├── tsconfig.json
└── CLAUDE.md                           # (Ver Sección 15)
```

---

## 4. Data Model

### Entities

**User** — cajeras y admin

| Field | Type | Notes |
|-------|------|-------|
| id | UUID | PK |
| nombre | String | Display name (ej. "Lupita") |
| rol | Enum | `cashier` \| `admin` |
| pin_hash | String | bcrypt hash del PIN de 4 dígitos |
| activo | Boolean | Soft-deactivation (no aparece en login pero se preservan sus encuestas) |
| created_at | DateTime | Default now() |

**Survey** — encuestas individuales

| Field | Type | Notes |
|-------|------|-------|
| id | UUID | PK |
| cashier_id | UUID | FK → User.id (debe ser rol=cashier) |
| score | Int | 1-5 (1=muy molesto, 5=muy contento) |
| comment | String? | Texto libre opcional, max 500 chars |
| created_at | DateTime | Default now(), indexed para queries por rango |

### Relationships
- `User (1) → (N) Survey`: una cajera tiene muchas encuestas. Survey nunca tiene relación directa con admin (admin solo lee).

### Database Schema (Prisma)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Rol {
  cashier
  admin
}

model User {
  id        String   @id @default(uuid())
  nombre    String
  rol       Rol
  pin_hash  String
  activo    Boolean  @default(true)
  created_at DateTime @default(now())
  surveys   Survey[]

  @@index([rol, activo])
}

model Survey {
  id          String   @id @default(uuid())
  cashier_id  String
  cashier     User     @relation(fields: [cashier_id], references: [id])
  score       Int      // 1-5
  comment     String?  @db.VarChar(500)
  created_at  DateTime @default(now())

  @@index([cashier_id, created_at])
  @@index([created_at])
}
```

### Cálculo de NPS (regla autoritativa — implementar en `src/lib/nps.ts`)

```typescript
// 5 caritas mapeadas a buckets NPS clásicos
type Bucket = 'detractor' | 'passive' | 'promoter';

function bucketFor(score: number): Bucket {
  if (score <= 2) return 'detractor'; // 😡, 😞
  if (score === 3) return 'passive';  // 😐
  return 'promoter';                   // 🙂, 😄
}

function calcularNPS(scores: number[]): number {
  if (scores.length === 0) return 0;
  const total = scores.length;
  const promoters = scores.filter(s => s >= 4).length;
  const detractors = scores.filter(s => s <= 2).length;
  return Math.round(((promoters - detractors) / total) * 100);
}
```

NPS resultante: entero entre -100 y +100.

---

## 5. API Design

### Routes Overview

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/api/auth/login` | Login con PIN. Body: `{userId, pin}`. Set cookie. | público |
| POST | `/api/auth/logout` | Limpia cookie de sesión | sí |
| POST | `/api/encuesta` | Guarda encuesta. Body: `{score, comment?}`. Toma cashier_id de la sesión. | rol=cashier |
| GET | `/admin/export?from=&to=&cashiers=` | Stream CSV de encuestas filtradas | rol=admin |

> El resto de la lectura del admin (dashboard, comentarios, tendencia) usa **React Server Components** con queries directas a Prisma — no requiere endpoints REST. Más simple, menos boilerplate.

### Key Endpoints Detail

#### POST `/api/auth/login`

```typescript
// Body
{ userId: string, pin: string }  // pin = "1234"

// Validación (Zod)
const LoginSchema = z.object({
  userId: z.string().uuid(),
  pin: z.string().regex(/^\d{4}$/),
});

// Flujo
// 1. Busca User por id, where activo=true
// 2. bcrypt.compare(pin, user.pin_hash)
// 3. Si match: setSession({ userId, rol })
// 4. Devuelve { ok, redirectTo: rol === 'admin' ? '/admin/dashboard' : '/encuesta' }

// Errores
// 400 invalid input
// 401 PIN incorrecto / usuario inactivo
// 429 más de 5 intentos en 60s desde mismo IP
```

#### POST `/api/encuesta`

```typescript
// Body
{ score: 1 | 2 | 3 | 4 | 5, comment?: string }

// Validación (Zod)
const EncuestaSchema = z.object({
  score: z.number().int().min(1).max(5),
  comment: z.string().max(500).optional().nullable(),
});

// Flujo
// 1. requireRole('cashier') → obtiene cashier_id de la sesión
// 2. prisma.survey.create({ data: { cashier_id, score, comment } })
// 3. Devuelve { ok: true, surveyId }

// Errores
// 400 invalid input
// 401 sesión inválida
// 403 rol incorrecto
```

#### GET `/admin/export`

```typescript
// Query params
// from=2026-04-01 (ISO date)
// to=2026-05-05
// cashiers=uuid1,uuid2 (opcional, default: todas)

// Respuesta: CSV stream
// Headers: id, fecha, cajera, score, comment
// Content-Type: text/csv; charset=utf-8
// Content-Disposition: attachment; filename="termometro-2026-05-05.csv"

// Stream con prisma cursor para no cargar todo a memoria.
```

---

## 6. Frontend Architecture

### Pages / Routes

| Route | Page | Description |
|-------|------|-------------|
| `/` | redirect | Si hay sesión y rol=cashier → `/encuesta`; si rol=admin → `/admin/dashboard`; si no → `/login` |
| `/login` | LoginKiosko | Grid de 6 cajeras con avatar inicial + teclado numérico PIN |
| `/encuesta` | Caritas | 5 caritas gigantes táctiles. Tap → siguiente paso |
| `/encuesta/comentario` | Comentario | Textarea + 2 botones grandes: "Saltar" y "Enviar" |
| `/encuesta/gracias` | Gracias | "Que tengas, un cremoso día" + auto-redirect 3s a `/encuesta` |
| `/admin/login` | LoginAdmin | Igual que kiosko pero filtra solo admins |
| `/admin/dashboard` | Dashboard | NPS general + por cajera + KPIs |
| `/admin/comentarios` | Comentarios | Lista filtrable |
| `/admin/tendencia` | Tendencia | LineChart NPS por semana |
| `/admin/export` | (route handler) | Descarga CSV |

### Component Hierarchy — Pantalla de cajera

```
EncuestaPage (Server Component)
  └─ KioskoGuard (Client) ─ previene back, mantiene wake-lock
     └─ FaceRow (Client)
        ├─ FaceButton 😡 score=1
        ├─ FaceButton 😞 score=2
        ├─ FaceButton 😐 score=3
        ├─ FaceButton 🙂 score=4
        └─ FaceButton 😄 score=5
     └─ LogoutMicroLink (esquina sup-derecha, opacity baja)
```

Tap en un `FaceButton`:
1. Anima el botón (scale 0.95 por 100ms)
2. Guarda el score en sessionStorage
3. `router.push('/encuesta/comentario')`

### Component Hierarchy — Pantalla admin dashboard

```
DashboardPage (Server Component, hace queries a Prisma)
  ├─ DateRangePicker (Client) — default: últimos 30 días
  ├─ NpsCard (NPS general grande)
  ├─ <Suspense>
  │   └─ NpsByCashierTable — tabla con barras inline
  ├─ ScoreDistribution — histograma 1-5
  └─ Link "Ver tendencia →"
```

### State Management

- **Server Components por default** para todo lo de admin (queries directas a Prisma vía `lib/db.ts`)
- **Client Components mínimos**: solo donde hay interacción (FaceButton, PinKeypad, DateRangePicker)
- **Sin librería de estado global**. La sesión vive en cookie (iron-session). El score parcial entre paso 1 y 2 vive en `sessionStorage` (clave `termometro:pendingScore`)
- **Cache**: revalidar admin pages cada 30s (`export const revalidate = 30`) — no necesita real-time
- **Wake Lock API**: la pantalla del kiosko mantiene `navigator.wakeLock.request('screen')` activo

---

## 7. Design System

### Voz y Personalidad
**Cremería HM** según brandbook (`E:/ClaudeWorks/conocimiento/marcas/cremeria-hm-brand.md`):
> "Si Cremería HM fuera una persona, sería como una mujer madura, cálida y cercana, con un fuerte sentido de tradición y responsabilidad."

Atributos: cálida, confiable, cercana de barrio, tradicional con orgullo, simple y directa.

### Colors

Paleta cálida tradicional, evocando crema, mantequilla y mostrador clásico mexicano.

| Role | Hex | Uso |
|------|-----|-----|
| Primary | `#C9A961` | Botones principales, acentos, dorado mantequilla |
| Primary Dark | `#A88845` | Hover/active de primary |
| Accent | `#8B2D2A` | Granate tradicional — header, branding |
| Background | `#FAF6EE` | Fondo de página, crema clarísima |
| Surface | `#FFFFFF` | Cards, panels |
| Surface Soft | `#F2EBDD` | Cards secundarias, hover |
| Text | `#2A1F18` | Body, marrón oscuro cálido |
| Text Muted | `#8B7E6E` | Secundario, labels |
| Border | `#E5DCC8` | Bordes suaves |
| Destructive | `#A23E3A` | Errores, eliminar |
| Success | `#5A9E5A` | Confirmaciones |

**Caritas** (gradiente semafórico suavizado):
| Score | Hex | Emoji |
|-------|-----|-------|
| 1 — Muy molesto | `#D14D5C` | 😡 |
| 2 — Molesto | `#E08856` | 😞 |
| 3 — Neutral | `#C9B26E` | 😐 |
| 4 — Contento | `#88B370` | 🙂 |
| 5 — Muy contento | `#5A9E5A` | 😄 |

### Typography

| Role | Font | Size | Weight |
|------|------|------|--------|
| Headings (H1-H3) | **Lora** (serif) | 32-48px en kiosko / 24-32px en admin | 600-700 |
| Body | **Inter** (sans) | 16-18px | 400-500 |
| Pantalla "gracias" | **Lora Italic** | 56px (kiosko) | 500 |
| Caritas labels | **Inter** | 18px | 600 |
| Code | **JetBrains Mono** | 14px | 400 |

Fonts cargados vía `next/font/google` (cero CLS).

### Spacing & Layout

- Spacing scale Tailwind default: 4px base
- Border radius: `12px` cards, `16px` botones grandes táctiles, `9999px` (full) para avatares
- Shadow: `shadow-md` para cards admin, `shadow-xl` para FaceButton
- Max content width admin: `1280px` centrado
- **Kiosko**: viewport completo, sin max-width, sin scroll horizontal
- Breakpoints: solo `md` (768px) para distinguir tablet portrait/landscape

### Component Style

- **Bordes redondeados generosos** (12-16px) — sensación cálida
- **Sombras suaves**, no duras
- **Sin animaciones excesivas** — la marca es tradicional, no efectista
- **Botones gigantes** en kiosko (mínimo 120px alto) — manos sucias, mostrador
- **Espacios amplios** entre caritas (gap-6 mínimo) — evitar mistaps
- **Tipografía serif para títulos** — refuerza tradición

### Frase final (literal, NO modificar)

> **"Que tengas, un cremoso día"**

Renderizado: Lora Italic 56px, color `#8B2D2A` (granate), centrado, fade-in 400ms al cargar `/encuesta/gracias`.

---

## 8. Authentication & Authorization

### Auth Flow

**Cajera (turno completo, ~8h):**
1. Tablet abre en `/` → redirect a `/login`
2. Cajera toca su avatar (uno de 6) → muestra teclado numérico
3. Teclea PIN 4 dígitos → POST `/api/auth/login`
4. Servidor valida con bcrypt → setea cookie httpOnly (TTL 12h)
5. Redirect a `/encuesta`
6. Tablet queda en kiosko todo el turno
7. Logout manual: tap micro-link en esquina sup-der → confirmación → POST `/api/auth/logout`

**Admin:**
1. Va a `/admin/login` desde su laptop/tablet
2. Login con PIN (mismo mecanismo, role check distinto)
3. Cookie con TTL 1h (más corto por seguridad)

### Protected Routes

- **Públicas**: `/login`, `/admin/login`, `/api/auth/login`
- **Cajera o admin**: `/encuesta/*`, `/api/auth/logout`
- **Solo cajera**: `/api/encuesta`
- **Solo admin**: `/admin/*`, `/admin/export`

Implementado en `src/middleware.ts` revisando cookie y rol antes de servir.

### Roles & Permissions

| Role | Can Do |
|------|--------|
| `cashier` | Ver `/encuesta/*`, enviar encuestas a su nombre, hacer logout |
| `admin` | Ver `/admin/*` (dashboard, comentarios, tendencia), exportar CSV |

**Sin role mixto**: un usuario es cashier O admin, no ambos. Si Beto necesita ambos roles, tiene 2 usuarios distintos.

### Session Management

- **iron-session** con cookies firmadas (`SESSION_PASSWORD` en `.env`, mín 32 chars)
- Cookie name: `termometro_session`
- httpOnly, secure (en prod), sameSite=lax
- Payload: `{ userId: string, rol: 'cashier' | 'admin' }`
- TTL: 12h cashier, 1h admin
- No JWT, no refresh tokens — overkill para este caso

### Rate Limiting (anti brute-force PIN)

- 5 intentos fallidos por IP por 60s → 429
- Implementado con un Map en memoria (suficiente para 1 instancia Vercel) o `@vercel/kv` si se quisiera persistente

---

## 9. Build Order

Cada paso es atómico — termina con un commit funcional. **No saltarse pasos.**

### Step 1: Project Scaffolding
```bash
pnpm create next-app@latest termometro-hm --typescript --tailwind --app --src-dir --import-alias "@/*" --use-pnpm
cd termometro-hm
pnpm add prisma @prisma/client bcryptjs iron-session zod recharts next-pwa
pnpm add -D @types/bcryptjs tsx playwright @playwright/test
npx prisma init --datasource-provider postgresql
```
- Configurar `tailwind.config.ts` con paleta Cremería HM (Sección 7)
- Configurar `tsconfig.json` strict mode
- Inicializar git, primer commit

### Step 2: Setup shadcn/ui
- Ejecutar skill `/shadcn-ui` o `pnpm dlx shadcn@latest init`
- Instalar componentes base: `button`, `input`, `card`, `table`, `dialog`, `select`, `popover`, `calendar`, `dropdown-menu`, `toast`
- Personalizar `globals.css` con CSS variables = paleta Cremería HM
- Cargar fonts Lora + Inter vía `next/font/google` en `app/layout.tsx`

### Step 3: Database + Schema + Seed
- Crear cuenta Neon, copiar `DATABASE_URL` a `.env`
- Pegar schema completo de Sección 4 en `prisma/schema.prisma`
- `pnpm prisma migrate dev --name init`
- Crear `prisma/seed.ts`:
  - 6 cajeras con PINs hasheados (PINs reales los pone Beto al final, en seed temporal: 1111-6666)
  - 1 admin con PIN provisional `9999`
- `pnpm prisma db seed`
- Crear `src/lib/db.ts` con PrismaClient singleton (patrón Next.js)

### Step 4: Sesión + Auth
- Crear `src/lib/session.ts` con iron-session config
- Crear `src/lib/auth.ts` con `verifyPin(userId, pin)`, `hashPin(pin)`
- `src/app/api/auth/login/route.ts`: POST → valida → setSession → devuelve `{redirectTo}`
- `src/app/api/auth/logout/route.ts`: POST → destroySession
- `src/middleware.ts`: protege rutas según pathname y rol en cookie
- Test manual con `curl` o Postman antes de seguir

### Step 5: Login Kiosko (cajera)
- `src/app/(kiosko)/layout.tsx`: layout pantalla completa, fonts, body class para deshabilitar zoom
- `src/app/(kiosko)/login/page.tsx`: server component que carga lista de cajeras activas
- `src/components/kiosko/CajeraPicker.tsx`: grid 3×2 con avatar (inicial sobre círculo de color) + nombre
- `src/components/kiosko/PinKeypad.tsx`: 10 botones grandes (0-9) + ⌫ + ✓
  - Display de 4 puntos arriba que se llenan al teclear
  - Al llegar a 4 dígitos, hace fetch a `/api/auth/login`
  - En error: shake animation + limpia PIN
- Probar el flujo completo de login en una tablet real (o devtools con touch sim)

### Step 6: Pantalla Caritas
- `src/app/(kiosko)/encuesta/page.tsx` (server)
- `src/components/kiosko/FaceRow.tsx` + `FaceButton.tsx`
  - SVG inline para cada carita (NO emojis del sistema, varían entre tablets) — usar `lucide-react` o SVGs custom
  - Tamaño mínimo 160×160px por botón
  - Color de fondo según paleta de caritas (Sección 7)
  - Label debajo: "Muy molesto", "Molesto", "Neutral", "Contento", "Muy contento"
  - Tap → guarda score en sessionStorage → router.push a `/encuesta/comentario`
- `KioskoGuard.tsx`:
  - `usePreventBackNavigation()` con `popstate` handler
  - `useWakeLock()` con `navigator.wakeLock.request('screen')`
  - `meta viewport` con `user-scalable=no`

### Step 7: Pantalla Comentario
- `src/app/(kiosko)/encuesta/comentario/page.tsx`
- `src/components/kiosko/CommentBox.tsx`:
  - Textarea grande (min 10 líneas), font-size 20px
  - Counter de 0/500
  - 2 botones gigantes lado a lado: **"Saltar"** (outline) y **"Enviar"** (primary)
  - Al enviar: lee score de sessionStorage, POST `/api/encuesta`, limpia sessionStorage, router.push a `/encuesta/gracias`
  - Al saltar: igual pero con comment=null
  - Si no hay score en sessionStorage: redirect a `/encuesta` (caso edge)
- API `/api/encuesta`: validar Zod, requireRole('cashier'), prisma.survey.create

### Step 8: Pantalla Gracias
- `src/app/(kiosko)/encuesta/gracias/page.tsx` (client component)
- Render: fondo crema, frase **"Que tengas, un cremoso día"** en Lora Italic 56px granate, fade-in 400ms
- `useEffect` con `setTimeout(() => router.replace('/encuesta'), 3000)`
- Sin botones — la espera de 3s es deliberada para que el cliente lea la frase

### Step 9: Layout Admin
- `src/app/admin/layout.tsx`: sidebar izquierda + header
- Sidebar items: Dashboard, Comentarios, Tendencia, Export, Logout
- Header: nombre del admin logueado + fecha
- Logo Cremería HM + título "Termómetro" en sidebar top
- Login admin (`src/app/admin/login/page.tsx`): igual que kiosko pero filtra `rol=admin` y pos-login redirige a `/admin/dashboard`

### Step 10: Admin Dashboard
- `src/app/admin/dashboard/page.tsx` (server)
- Queries (en `src/lib/nps.ts`):
  - `getNpsGeneral(from, to)` → number
  - `getNpsPorCajera(from, to)` → `{ cashierId, nombre, nps, total }[]`
  - `getDistribucion(from, to)` → `[{ score, count }]`
  - `getKpis(from, to)` → `{ totalEncuestas, conComentario, npsActual, npsSemanaAnterior }`
- Componentes:
  - `NpsCard` (grande, NPS general + delta semana anterior)
  - `NpsByCashierTable` (tabla con barras inline coloreadas según el bucket — verde/amarillo/rojo)
  - `ScoreDistribution` (histograma vertical 1-5 usando Recharts BarChart)
- DateRangePicker default: últimos 30 días, persistir en query string

### Step 11: Página Comentarios
- `src/app/admin/comentarios/page.tsx` (server)
- Filtros (vía query string): `?from=&to=&cashiers=&scores=`
- Render: lista de cards con:
  - Carita del score
  - Nombre cajera
  - Fecha (formato `DD/MM/AAAA HH:mm`)
  - Texto del comentario (o "(sin comentario)" en muted)
- Paginación: 50 por página, server-side cursor con Prisma
- `CashierFilter` (multi-select shadcn) + `DateRangePicker` + `ScoreFilter` (chips multi-select 1-5)

### Step 12: Página Tendencia
- `src/app/admin/tendencia/page.tsx` (server)
- Query: NPS por semana (agrupar por `date_trunc('week', created_at)`) y por cajera
- Recharts `LineChart`: eje X = semana, eje Y = NPS, una línea por cajera + una línea negra "general" gruesa
- Toggle: ver todas las cajeras vs solo las seleccionadas

### Step 13: Export CSV
- `src/app/admin/export/route.ts` (route handler GET)
- Query params: `from`, `to`, `cashiers`
- Stream CSV con headers: `id,fecha,cajera,score,comment`
- Comment: escapar con CSV-quoting estándar (doble comilla)
- Filename: `termometro-{from}-a-{to}.csv`
- En `/admin/export/page.tsx`: form simple con inputs de fecha + button que dispara descarga

### Step 14: PWA Setup
- `public/manifest.json`:
  ```json
  {
    "name": "Termómetro HM",
    "short_name": "Termómetro",
    "start_url": "/",
    "display": "fullscreen",
    "orientation": "landscape",
    "background_color": "#FAF6EE",
    "theme_color": "#8B2D2A",
    "icons": [
      { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
      { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
    ]
  }
  ```
- Configurar `next-pwa` en `next.config.ts` con service worker mínimo (cache de assets estáticos solo)
- Crear iconos 192/512 con logo Cremería HM (Beto los provee, o se generan con un placeholder)
- En la tablet: abrir Chrome → menú → "Add to Home screen" → app instalada en pantalla completa

### Step 15: Polish & Edge Cases
- Loading states (Suspense + skeletons en admin)
- Empty states ("Aún no hay encuestas para este filtro")
- Error boundary global con mensaje en voz Cremería HM ("Algo se nos atravesó. Intenta de nuevo en un momento.")
- Toast con sonner para confirmaciones admin (export, etc.)
- Logout confirmation dialog en kiosko
- Verificar que Lighthouse PWA score = 100

### Step 16: Testing E2E
- `tests/e2e/flujo-cajera.spec.ts`:
  - Login con PIN
  - Tap carita 4
  - Escribir comentario
  - Enviar
  - Verificar redirect a `/encuesta/gracias` y luego de vuelta a `/encuesta`
  - Verificar que la encuesta quedó en DB
- `tests/e2e/admin-export.spec.ts`:
  - Login admin
  - Ir a export
  - Disparar download
  - Validar headers CSV
- Correr con `pnpm test`

### Step 17: Deploy a Vercel
- `git remote add origin git@github.com:huheme25/termometro-hm.git`
- `git push -u origin main`
- En Vercel: importar repo, agregar env vars (`DATABASE_URL`, `SESSION_PASSWORD`)
- Deploy automático en cada push
- Conectar dominio (opcional): `termometro.cremeriahm.com` o subpath en cremeriahm.com

### Step 18: Onboarding Cajeras
- Beto define los 6 PINs reales (uno por cajera) y se los pasa en una hoja
- Beto crea su PIN admin
- Update de seed o ejecutar migración de datos: actualizar `pin_hash` por usuario
- Tablet del mostrador: abrir URL → "Add to Home Screen" → ícono fullscreen
- Capacitación: 30 segundos por cajera ("aquí está tu carita, tecleas tus 4 números, listo")

---

## 10. Environment Setup

### Prerequisites
- Node.js 20+ (LTS)
- pnpm 9+
- Cuenta Neon (free tier)
- Cuenta Vercel (free tier)
- Cuenta GitHub (huheme25, ya existe)

### Environment Variables

| Variable | Description | Where to Get |
|----------|-------------|--------------|
| `DATABASE_URL` | Postgres connection string | Neon dashboard → Connection details |
| `SESSION_PASSWORD` | Secret para iron-session (mín 32 chars) | `openssl rand -base64 48` |
| `NODE_ENV` | `development` / `production` | Auto en Vercel |

### Initial Setup Commands

```bash
# Clone & install
git clone git@github.com:huheme25/termometro-hm.git
cd termometro-hm
pnpm install

# Env
cp .env.example .env
# Editar .env con valores reales de Neon

# DB
pnpm prisma migrate dev
pnpm prisma db seed

# Run dev
pnpm dev
# → http://localhost:3000
```

---

## 11. Dependencies

### Core
| Package | Purpose |
|---------|---------|
| `next` | Framework |
| `react`, `react-dom` | UI |
| `@prisma/client` | DB client |
| `bcryptjs` | Hash de PINs |
| `iron-session` | Cookies de sesión firmadas |
| `zod` | Validación |
| `recharts` | Gráficas |
| `next-pwa` | PWA setup |
| `lucide-react` | Iconos |
| `sonner` | Toasts |
| `date-fns` | Format/parse fechas |
| `clsx`, `tailwind-merge` | Utility para cn() |

### Dev
| Package | Purpose |
|---------|---------|
| `prisma` | CLI de migrations |
| `typescript` | Type-checking |
| `@types/*` | Type defs |
| `eslint`, `eslint-config-next` | Linting |
| `tailwindcss`, `postcss`, `autoprefixer` | CSS |
| `tsx` | Ejecutar seed.ts |
| `playwright`, `@playwright/test` | E2E tests |

---

## 12. Deployment Strategy

### Hosting

**Vercel (free tier hobby)**
- Despliegue por push a `main`
- Preview deploys automáticos por PR
- Edge runtime no necesario (todas las routes son Node.js)
- Build command: `prisma generate && next build`
- Output: standalone

### CI/CD

- **Vercel maneja todo** — no hace falta GitHub Actions para v1
- Pre-deploy hook: `prisma migrate deploy` (configurado en `package.json` script `vercel-build`)
- Si en algún momento las migrations dejan de aplicarse limpio, mover a CI explícito

### Domain & DNS

Opciones:
1. **Subdominio en cremeriahm.com**: `termometro.cremeriahm.com` (recomendado)
2. **Path en cremeriahm.com**: `cremeriahm.com/termometro` (más complejo, requiere proxy)
3. **Vercel default**: `termometro-hm.vercel.app` (perfectamente válido para v1)

### Environments

- **Dev**: localhost + Neon dev branch
- **Production**: Vercel + Neon main branch
- **Sin staging** — el equipo es muy chico, los preview deploys de Vercel cumplen ese rol

### Backup

- Neon hace backups automáticos en el plan free (point-in-time recovery 24h)
- Para extra seguridad: cron en Vercel → ejecuta dump diario → sube a Cloudflare R2 (opcional, V2)

---

## 13. Testing Strategy

### Unit Tests (mínimo necesario)
- `src/lib/nps.ts`: testear `calcularNPS()` con casos típicos (todos promotores → 100, todos detractores → -100, mezcla, vacío)
- `src/lib/csv.ts`: testear escape de comillas/saltos en `comment`
- Framework: **Vitest** (más rápido que Jest, integración nativa con Vite/Next)

### Integration Tests
- No críticos para v1. Saltarse.

### E2E Tests (críticos)
- **Playwright** con dos specs (Step 16):
  - `flujo-cajera.spec.ts`: end-to-end del flujo completo de votación
  - `admin-export.spec.ts`: login admin + export CSV
- Correr en CI antes de cada deploy a producción
- En local: `pnpm test:e2e`

### Manual Smoke Test pre-deploy
1. Login cajera con PIN real
2. Vota carita 5
3. Escribe comentario "Test"
4. Verifica que aparece en `/admin/comentarios`
5. Logout
6. Cierra y reabre tablet → debe seguir requiriendo login

---

## 14. Skills to Use During Build

| Skill | When to Use | Why |
|-------|-------------|-----|
| `/shadcn-ui` | Step 2 (setup) | Instalar y customizar componentes base |
| `/frontend-design` | Steps 5, 6, 7, 8, 10 | Pantallas táctiles grandes con personalidad Cremería HM, no genéricas |
| `/ui-ux-pro-max` | Step 2 (refinar paleta) | Si se quiere validar/expandir la paleta cálida propuesta |
| `/playwright-cli` | Step 16 (testing) | Ejecutar E2E tests, debug de selectors |
| `/anthropics-brand-voice` | Steps 5-15 | Validar todo texto visible (frases, errores, empty states) contra la voz Cremería HM (`E:/ClaudeWorks/conocimiento/marcas/cremeria-hm-brand.md`) |

---

## 15. CLAUDE.md for Target Project

```markdown
# Termómetro HM

App interna de Cremería HM para medir NPS por cajera vía tablet de mostrador. 6 cajeras, 1 admin, hosting Vercel free.

## Commands

- `pnpm dev` — Dev server en http://localhost:3000
- `pnpm build` — Build de producción
- `pnpm lint` — ESLint
- `pnpm test` — Vitest (unit)
- `pnpm test:e2e` — Playwright (e2e)
- `pnpm prisma migrate dev` — Crear/aplicar migración
- `pnpm prisma db seed` — Cargar 6 cajeras + 1 admin
- `pnpm prisma studio` — UI para inspeccionar DB

## Tech Stack

Next.js 15 (App Router) + TypeScript strict + Tailwind v4 + shadcn/ui + Prisma + Postgres (Neon) + iron-session + Recharts + next-pwa. Hosting Vercel.

## Architecture

### Directory Structure
- `src/app/(kiosko)/` — Layout kiosko (login, encuesta, comentario, gracias). Pantalla completa, sin chrome.
- `src/app/admin/` — Layout admin con sidebar. Solo accesible con rol=admin.
- `src/app/api/` — Route handlers para auth y submit de encuesta. El resto del admin lee con RSC.
- `src/components/ui/` — Primitivas shadcn
- `src/components/kiosko/` — Componentes específicos del flujo táctil (FaceButton, PinKeypad, etc.)
- `src/components/admin/` — Componentes del dashboard (NpsCard, charts, filtros)
- `src/lib/db.ts` — PrismaClient singleton
- `src/lib/session.ts` — iron-session config
- `src/lib/nps.ts` — Cálculo de NPS y agrupaciones
- `src/middleware.ts` — Auth + role guard

### Data Flow
- **Cajera vota**: Client tap → POST `/api/encuesta` → Prisma create → success → redirect cliente
- **Admin lee**: Server Component → Prisma query → render → revalidate cada 30s
- **Sesión**: cookie firmada (iron-session) con `{userId, rol}`. Middleware revisa antes de servir rutas protegidas

### Key Patterns
- **Server Components por default**. Solo `"use client"` en componentes con interacción (botones, inputs, charts interactivos).
- **Todas las queries pasan por `lib/db.ts`** (singleton de Prisma).
- **Validación con Zod** en cada route handler antes de tocar la DB.
- **Sin estado global**: sesión vive en cookie, score parcial en `sessionStorage`.
- **NPS calculation autoritativa en `lib/nps.ts`** — nunca recalcular ad-hoc en otros archivos.

## Code Organization Rules

1. **Una pantalla = un page.tsx + componentes colocados al lado** si son específicos. Componentes reutilizables van a `src/components/{kiosko|admin|ui}/`.
2. **Path alias `@/`** apunta a `src/`. Siempre usar `@/lib/...`, no rutas relativas largas.
3. **Sin barrel files (`index.ts`)**. Importar directo del archivo.
4. **Server Components por default.** `"use client"` solo cuando se necesita estado, eventos o browser APIs.
5. **Componentes ≤ 300 líneas**. Si crecen, extraer subcomponentes.
6. **Validación Zod en cada API route** antes de tocar la DB. Sin excepciones.
7. **Nunca hardcodear PINs**. Siempre vienen de la DB hasheados.

## Design System

### Colors (CSS variables en globals.css)
- `--primary: #C9A961` (dorado mantequilla)
- `--accent: #8B2D2A` (granate)
- `--background: #FAF6EE` (crema)
- `--surface: #FFFFFF`
- `--text: #2A1F18`
- `--muted: #8B7E6E`
- `--border: #E5DCC8`
- `--destructive: #A23E3A`
- `--success: #5A9E5A`

Caritas:
- score 1: `#D14D5C`
- score 2: `#E08856`
- score 3: `#C9B26E`
- score 4: `#88B370`
- score 5: `#5A9E5A`

### Typography
- Headings: **Lora** (serif) 600-700, 32-48px en kiosko / 24-32px en admin
- Body: **Inter** (sans) 400-500, 16-18px
- Frase final "Que tengas, un cremoso día": Lora Italic 56px granate

### Style
- Border radius: 12px cards, 16px botones grandes, 9999px avatares
- Shadows suaves (`shadow-md` admin, `shadow-xl` FaceButton)
- Spacing base 4px (Tailwind default)
- Aesthetic: cálido, tradicional con orgullo, espacios amplios, botones gigantes en kiosko

## Voz y Branding

Toda copy visible (incluyendo errores y empty states) debe respetar la voz Cremería HM:
- Cálida, cercana, simple y directa
- NUNCA: "ALL CAPS", "!!!", lenguaje corporativo, tecnicismos
- Frases tipo: "Algo se nos atravesó. Intenta de nuevo en un momento." en lugar de "Error 500: Internal Server Error"
- Frase final del flujo de cliente, **literal e inalterable**: **"Que tengas, un cremoso día"**
- Brandbook completo en `E:/ClaudeWorks/conocimiento/marcas/cremeria-hm-brand.md`

## Environment Variables

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | Postgres URL (Neon) |
| `SESSION_PASSWORD` | Secret iron-session (mín 32 chars) |

## Reglas No Negociables

1. **TypeScript strict mode**, sin `any`. Si necesitas tipo flexible, usa `unknown` y narrowing.
2. **Validación Zod en TODA API route** antes de hablar con la DB.
3. **PIN nunca en logs ni en respuestas HTTP**. Solo el hash, solo en DB.
4. **Server Components por default** — no abusar de "use client".
5. **Frase "Que tengas, un cremoso día" no se cambia** — es marca registrada del flujo, literal.
6. **El kiosko nunca debe permitir navegación accidental** — wake-lock activo, prevenir back, prevenir zoom, prevenir context menu.
7. **Toda copy visible respeta la voz Cremería HM** — no genérica, no corporativa.
8. **Sin dependencias innecesarias**. Antes de agregar un paquete, preguntarse si Next/React/Tailwind ya lo resuelven.
9. **Comentarios en español** cuando aporten — sin sobre-documentar lo obvio.
10. **Commits atómicos**: un paso del Build Order = un commit funcional.
```

---

## 16. Reglas No Negociables

1. **TypeScript strict mode** en todo el proyecto. Sin `any` salvo casos justificados con `// eslint-disable-next-line @typescript-eslint/no-explicit-any` y comentario del por qué.
2. **Validación Zod en cada API route** antes de tocar la DB. No confiar en input nunca.
3. **PIN siempre hasheado con bcrypt (10 rounds mínimo)**. Nunca log, nunca respuesta HTTP.
4. **Cookies httpOnly + secure en prod**. iron-session con secret de al menos 32 chars.
5. **Frase "Que tengas, un cremoso día"** es literal. No reescribir. No traducir. No cambiar puntuación.
6. **Voz Cremería HM en toda copy visible** — incluyendo errores y empty states. Cero lenguaje corporativo.
7. **Mobile-first / tablet-first**: el flujo de cajera DEBE funcionar perfectamente en una tablet de 10". El admin puede ser desktop-first.
8. **Wake-lock + prevent-zoom + prevent-back** activos en el kiosko. Si la tablet se duerme o el cliente hace pinch-zoom accidental, mata la UX.
9. **NPS calculation centralizada en `lib/nps.ts`**. Toda página o endpoint que muestre NPS la importa de ahí.
10. **Build Order es serial**. Cada paso termina con un commit funcional. Saltar pasos rompe el orden de dependencias.
11. **Commits atómicos en español**. Mensaje claro, presente, qué se logró ("agregar pantalla de gracias con auto-redirect").
12. **No agregar features fuera de scope**. Alertas push, integración EspritOS, dashboards extra → V2.
