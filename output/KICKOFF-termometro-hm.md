# Kickoff — Termómetro HM

> Pega esto literal en una sesión nueva de Claude Code, ya posicionada en el directorio donde quieras crear el proyecto (recomendado: `E:\ClaudeWorks\proyectos\termometro-hm\`).

---

## Prompt

Vas a construir **Termómetro HM**, una app interna para Cremería HM que mide NPS por cajera vía tablet de mostrador.

### Blueprint autoritativo

El diseño completo vive aquí:
**`E:\ClaudeWorks\repos-referencia\the-architect\output\termometro-hm-blueprint.md`**

Léelo completo antes de tocar una sola línea de código. Es la única fuente de verdad: stack, schema, paleta, voz de marca, build order de 18 pasos, CLAUDE.md objetivo y reglas no negociables.

### Cómo trabajas

1. **Sigue el Build Order al pie de la letra.** Pasos 1 → 18, en orden. Cada paso termina con un commit atómico funcional.
2. **No improvises stack.** Si el blueprint dice Prisma, no metas Drizzle. Si dice iron-session, no metas Clerk.
3. **Voz Cremería HM en toda copy visible.** Brandbook en `E:\ClaudeWorks\conocimiento\marcas\cremeria-hm-brand.md`. Nada corporativo, nada de !!!. Cálido, simple, directo.
4. **La frase final "Que tengas, un cremoso día" es literal.** No la reescribas, no la traduzcas, no le cambies la coma.
5. **Idioma de UI y commits: español.** Código y nombres de variables: inglés (estándar).
6. **TypeScript strict, validación Zod en toda API route, PIN siempre hasheado.** Las 12 reglas no negociables están al final del blueprint — léelas.
7. **Si te bloqueas con algo que el blueprint no resuelve**, pausa y pregúntame. No inventes decisiones de arquitectura por tu cuenta.

### Skills recomendadas durante el build

| Skill | Cuándo |
|-------|--------|
| `/shadcn-ui` | Step 2 (instalar componentes base) |
| `/frontend-design` | Steps 5-10 (pantallas táctiles del kiosko) |
| `/anthropics-brand-voice` | Cualquier paso con copy visible |
| `/playwright-cli` | Step 16 (E2E tests) |

### Arranque

1. Lee el blueprint completo (~25 minutos)
2. Crea el directorio del proyecto si no existe
3. Inicia con **Step 1: Project Scaffolding**
4. Ejecuta los comandos exactos del blueprint
5. Avísame al terminar cada step para que valide antes de avanzar al siguiente — al menos durante los pasos 1-5; del 6 en adelante puedes hacer 2-3 pasos seguidos antes de chequeo.

### Contexto del cliente (Beto)

- 6 cajeras totales, una sola tablet en mostrador, hosting Vercel + Neon free
- Beto ya conoce Next.js + Prisma del CRM-ERP — no le expliques lo básico
- Costo objetivo: $0/mes
- Timeline: 2-3 días de trabajo concentrado

Arranca.
