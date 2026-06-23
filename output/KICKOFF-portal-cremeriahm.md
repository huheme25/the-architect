# KICKOFF — portal-cremeriahm

> Pega este prompt completo en una nueva sesión de Claude Code abierta en `E:\ClaudeWorks\proyectos\portal-cremeriahm\` (carpeta nueva, vacía).

---

## Prompt para la nueva sesión de Claude Code

```
Vas a construir el proyecto `portal-cremeriahm` siguiendo el blueprint autoritativo en:

E:\ClaudeWorks\repos-referencia\the-architect\output\portal-cremeriahm-blueprint.md

CONTEXTO RÁPIDO
- Soy Beto. Idioma: español. Moneda: MXN. Fechas DD/MM/AAAA.
- Cremería HM es la empresa familiar de mi papá (lácteos/embutidos/carnes en Tonalá).
- El portal es para que ~300 clientes mayoristas vean sus precios personalizados.
- Stack: Django 5.1 + HTMX 2 + Alpine 3 + Tailwind v4. Postgres compartido con EspritOS, role aislado. MySQL PuntoZero readonly. Auth passwordless OTP (Email Resend MVP, WhatsApp Meta Cloud API en sprint 2).
- Acoplado a la infra de EspritOS pero AISLADO a 3 niveles (proceso, datos, red). Si comprometen el portal, NO deben poder leer/modificar nada del ERP.

DECISIONES YA TOMADAS (NO RE-PREGUNTAR)
1. Hostname beta: portal.espritos.app — reutiliza el tunnel existente de EspritOS, no crear uno nuevo.
2. Hostname lanzamiento: portal.cremeriahm.com — yo migro DNS a Cloudflare en paralelo al desarrollo.
3. Auth: passwordless OTP, sin contraseñas. Registro requiere Clave de Cliente (PuntoZero) + Celular obligatorio + Email opcional + Nombre. Validación: clave debe existir en clientes Y celular debe coincidir con alguno de Telefono1..4. Si no match → prospect (solo Precio1).
4. WhatsApp en proceso de verificación con Meta (responde en ~5 días). MVP arranca con email-only.
5. Path local: E:\ClaudeWorks\proyectos\portal-cremeriahm\
6. Path en server WSL2 (deploy): ~/portal-cremeriahm
7. Repo Git: huheme25/portal-cremeriahm (privado).
8. Puerto dev: 8001 (EspritOS usa 8100).

CÓMO PROCEDER
1. Lee el blueprint completo (las 16 secciones). No saltes secciones.
2. Confirma que entendiste la arquitectura (aislamiento triple, multi-DB router, passwordless OTP, Sprint 0).
3. Empieza por Step 1 (Project Scaffolding) del blueprint.
4. Después de cada Step ejecutado, haz commit con mensaje descriptivo y dime el resultado en 3-5 líneas antes de pasar al siguiente.
5. Para Steps que requieran credenciales (Resend API key, Meta tokens, Postgres password de portal_user, MySQL password de portal_reader, Cloudflare tokens), DETENTE y pídeme las credenciales — no inventes valores en .env.
6. Steps 0 (Sprint 0 limpieza de celulares) y 14-16 (deploy) requieren acción mía manual. Cuando llegues, prepárame el script/lista exacta y avísame.

REGLAS NO NEGOCIABLES (Sección 16 del blueprint, no resumir)
- Aislamiento Postgres es sagrado. portal_user solo lee vistas curadas, nunca tablas espritos.* directas.
- OTP en plain solo en RAM. Jamás en logs ni audit_log.
- Rate limiting en TODOS los endpoints públicos.
- Multi-DB router con managed=False para modelos PuntoZero.
- Celulares siempre normalizados a +52XXXXXXXXXX.
- Sin Twilio para WhatsApp — Meta Cloud API directa.
- El script publicar_precios.py de AnalisisVentas NO se toca.

INFRAESTRUCTURA EXISTENTE QUE PUEDES ASUMIR
- Postgres 16 corriendo en 192.168.0.152:5432 (servidor EspritOS WSL2). DB: espritos.
- MySQL 5.1 corriendo en 192.168.0.200:3306. DB: datos1 (PuntoZero Cremería).
- Cloudflare Tunnel ya activo para espritos.app. Reutilizable.
- Docker Compose en ~/espritos en WSL2 con red espritos_default.

EMPIEZA: lee el blueprint y arranca Step 1.
```

---

## Tareas paralelas mías (Beto, manuales)

Mientras Claude Code construye, yo voy avanzando estas:

### 🟢 Esta semana (en paralelo a Steps 1-4)

- [ ] **Crear repo en GitHub:** `huheme25/portal-cremeriahm` (privado)
- [ ] **Cloudflare → Zero Trust → Tunnels → tunnel de EspritOS → Public Hostname:** agregar `portal.espritos.app` apuntando a `http://portal:8000`
- [ ] **Crear cuenta Resend** y verificar dominio `cremeriahm.com` (DNS records SPF/DKIM/DMARC en Hostinger por ahora)
- [ ] **MySQL PuntoZero (192.168.0.200):** crear user `portal_reader` con permisos mínimos (Step 4 del blueprint tiene el SQL exacto)
- [ ] **Postgres EspritOS:** crear role `portal_user` y schema `portal` (Step 3 del blueprint tiene el SQL exacto)
- [ ] **Sprint 0 — limpieza de celulares:** correr el script `pre_launch_audit_celulares.py` en cuanto Claude lo cree (Step 4), revisar el CSV, asignar a vendedores los huecos. Meta: ≥85% cobertura antes del lanzamiento masivo.

### 🟡 Semanas 2-3 (en paralelo a Steps 5-13)

- [ ] **Migrar DNS de cremeriahm.com de Hostinger a Cloudflare** (Step 15 Fase B del blueprint tiene los pasos exactos)
- [ ] **Verificación Meta WhatsApp Business Cloud API** — esperar respuesta. Cuando aprueben:
  - [ ] Crear plantilla `cremeriahm_otp_es` en Meta Business Manager (Authentication category)
  - [ ] Anotar `WHATSAPP_PHONE_NUMBER_ID` y generar `WHATSAPP_ACCESS_TOKEN` permanente
  - [ ] Pasarle a Claude para Step 11
- [ ] **Re-verificar dominio Resend** una vez DNS esté en Cloudflare (SPF/DKIM/DMARC más fáciles desde el panel de Cloudflare)

### 🟣 Semana 4-5 (lanzamiento)

- [ ] **Cloudflare → tunnel EspritOS → Public Hostname:** agregar `portal.cremeriahm.com` apuntando a `http://portal:8000`
- [ ] **Editar HTML de cremeriahm.com** (Hostinger via git push) para agregar botón "Acceso clientes" → `https://portal.cremeriahm.com/`
- [ ] **Beta cerrado:** invitar 5-10 clientes mayoristas conocidos. Recoger feedback 1 semana.
- [ ] **Anuncio masivo** vía WhatsApp Business broadcast a base de mayoristas con celular ≥85% cubierto.

---

## Si algo se atora

- Si el blueprint no responde una pregunta arquitectónica nueva: vuelve aquí (al Architect) con el contexto.
- Si Claude Code se desvía del blueprint: cita la regla violada y pídele que vuelva al plan.
- Si alguna decisión de Steps 1-16 ya no aplica por algo del mundo real: márcalo en el blueprint y vuelve aquí para revisión.

Buena suerte con el build. 🛠️
