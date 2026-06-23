# EspritOS V5 — Kickoff prompt para builder

> Este archivo se usa al inicio de cada sesión Claude Code que ejecute trabajo de V5.
> Ajusta la sección [SCOPE] para indicar qué sprint/steps tocan en esta sesión.

---

## Cómo usar este archivo

1. Abrir Claude Code en `~/espritos/` (o `E:\ClaudeWorks\proyectos\EspritOS\` en Windows)
2. Verificar que estás en branch `v5/sprint-<n>` correcto (`git branch --show-current`)
3. Pegar el bloque "Prompt Maestro" abajo, ajustando el `[SCOPE]`
4. Dejar a Claude trabajar — interrumpir solo si hace algo fuera de scope
5. Al final de la sesión: revisar commits, mergear si OK

---

## Prompt Maestro (copiar desde aquí)

```
Estás trabajando en EspritOS V5, una transformación de 9 sprints (~32 semanas, ~894h) que convierte el ERP single-tenant actual en SaaS multi-tenant configurable con vertical de perecederos opt-in y modo offline-first.

CONTEXTO OBLIGATORIO A LEER ANTES DE ESCRIBIR UNA SOLA LÍNEA DE CÓDIGO:

1. `E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/BLUEPRINT-V5.md` — el blueprint maestro (4,445 líneas). Lee toda la sección 9 del sprint que toca esta sesión + las secciones 16 (Reglas No Negociables V5), 17 (Migration Strategy si toca S2.7) y 18 (Decisions Log).

2. `~/espritos/CLAUDE.md` — instrucciones del proyecto vigentes (V1-V4 + addendum V5 que se agrega en S2).

3. `E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/BLUEPRINT.md` — blueprint original V1-V4 (referencia, NO ejecutar). Solo úsalo para entender modelos existentes que vas a extender.

4. `git log --oneline -30` — últimos commits (entender desde dónde retomas).

[SCOPE DE ESTA SESIÓN]
- Sprint: S<n>
- Steps a ejecutar: S<n>.<a> hasta S<n>.<b> (incluido)
- Tiempo estimado: <X>h (suma de estimaciones de los steps)
- Branch git: v5/sprint-<n>-<nombre>
- Bloqueos previos verificados: <lista>

REGLAS DE EJECUCIÓN ESTRICTAS:

A. UN STEP A LA VEZ, EN ORDEN.
   No saltes steps. Cada step termina con commit + tests verde + verificación manual documentada en commit body. Si un step falla a la mitad, NO empieces el siguiente — pausa y reporta.

B. CERO ALCANCE FUERA DEL SCOPE.
   Si descubres bug en V4 mientras trabajas: documenta en `docs/v5/findings/` y sigue. No "aprovechar para arreglarlo". Excepción: si el bug bloquea el step actual, escala al usuario antes de tocarlo.

C. CADA COMMIT FORMATO `[V5][S<n>.<n>] descripción`.
   Body del commit incluye:
   - Tests añadidos / total tests pasando
   - Verificación manual hecha (lista checklist marcada)
   - Cualquier deviation del blueprint con justificación

D. TESTS OBLIGATORIOS MÍNIMOS DEFINIDOS POR EL BLUEPRINT.
   El blueprint lista por step "mínimo X tests". Cumple ese mínimo. Si no se te ocurren más, está bien — calidad > cantidad. NO escribas tests basura para inflar número.

E. VERIFICA QUE TESTS PREVIOS SIGUEN VERDE.
   Antes de cada commit: `pytest --tb=short -q`. Si rompiste tests V1-V4 o de sprints anteriores, ARREGLA antes de commitear el step.

F. SI EL BLUEPRINT ES AMBIGUO O CONTRADICTORIO, PREGUNTA AL USUARIO.
   No asumas. No improvises. El blueprint es la fuente de verdad — si está mal, se actualiza el blueprint.

G. NO HAGAS DEPLOYS NI MIGRACIONES PRODUCTIVAS sin confirmación explícita del usuario, incluso si el step lo pide. Especialmente S2.7 (migración Cremería HM) — eso lo dispara Beto manualmente con presencia.

H. ESPAÑOL EN COMUNICACIÓN, INGLÉS EN CÓDIGO, COMENTARIOS ESPAÑOL CUANDO NO SEA OBVIO.
   El usuario es Beto, hijo del dueño de Cremería HM, finanzas/estrategia/tecnología.

I. STACK NO SE NEGOCIA.
   Django 5.1 + Postgres 16 + HTMX + Alpine + Tailwind v4 + Celery + Redis. Adiciones V5 ya listadas en blueprint sección 2. NO sugerir migrar a React, Vue, otra ORM, otro framework. Si necesitas librería nueva no listada: pregunta primero.

J. CUANDO TERMINES EL SCOPE DE LA SESIÓN, REPORTA:
   - Steps completados: lista
   - Tests añadidos en sesión: número + nombres clave
   - Tests totales pasando ahora vs antes de la sesión
   - Próximo step a ejecutar (siguiente sesión)
   - Bloqueos / dudas para el usuario

K. SI TE QUEDAS SIN CONTEXTO ANTES DE COMPLETAR EL SCOPE:
   Para. Reporta dónde quedaste con suficiente detalle para que la próxima sesión retome (qué files tocados, qué falta, qué decisiones tomadas). NO intentes apretar más en menos calidad.

EMPIEZA AHORA. Primer paso: leer los 4 archivos de contexto obligatorio. Después, ejecutar Step S<n>.<a>.
```

---

## Variantes del prompt según fase

### Para el primer sprint (S1) — sin V5 antes

Reemplaza `[SCOPE DE ESTA SESIÓN]` con:

```
- Sprint: S1 — CFDI MX avanzado
- Steps a ejecutar: S1.1 (catálogos SAT versionados)
- Tiempo estimado: 8h
- Branch git: v5/sprint-1-cfdi (créalo si no existe desde v5/develop)
- Bloqueos previos verificados:
  * S0.1 backup tomado y restaurable: SÍ/NO
  * S0.3 BASELINE-V4.md commiteado: SÍ/NO
  * Branch v5/develop creado: SÍ/NO
- Pre-flight especial: validar que `python manage.py test` pasa los 1,041 tests baseline ANTES de tocar nada.
```

### Para sprints 2+ — verificando cierre del anterior

Antes de iniciar Sprint N, agrega al inicio del prompt:

```
PRE-FLIGHT SPRINT N:
- Sprint N-1 mergeado a v5/develop: SÍ/NO
- docs/v5/sprint-(N-1)-summary.md existe y firmado: SÍ/NO
- Hito de cierre Sprint N-1 todos los criterios ✅: SÍ/NO
- Tests totales V5 hasta ahora: <número> verde

Si alguno es NO: pausar, reportar al usuario, NO empezar Sprint N.
```

### Para el step crítico S2.7 (migración Cremería HM)

USA SESIÓN DEDICADA SOLO A ESTE STEP. El prompt:

```
SESIÓN CRÍTICA — S2.7 MIGRACIÓN CREMERÍA HM A MULTI-TENANT.

Esta sesión NO ejecuta la migración productiva. Esta sesión:
1. Implementa el comando `migrate_cremeria_to_v5` (idempotente)
2. Lo prueba LOCAL contra una restauración del backup productivo
3. Documenta runbook en `docs/v5/migration-v5-cremeriahm.md`
4. Crea rollback runbook en `docs/v5/rollback-s2.7.md`
5. Reporta listo para que Beto agende ventana sábado y dispare manualmente

NO ejecutar migración productiva en esta sesión bajo ningún concepto.
NO conectarse al servidor de prod (192.168.0.152) para "probar".
TODO en local con backup restaurado.

Lee blueprint sección 9 step S2.7 y sección 17 (Migration Strategy) completas antes de empezar.
```

### Para sesión de "verificación de hito de cierre"

Cuando un sprint dice estar terminado pero quieres validación independiente:

```
SESIÓN DE VERIFICACIÓN — HITO DE CIERRE SPRINT <n>.

NO modificas código. Solo validas.

Para cada criterio del "Hito de cierre Sprint <n>" del blueprint:
1. Comando exacto que lo verifica
2. Resultado real
3. ✅ / ❌ / ⚠️ con explicación

Reporta al final: ¿se puede mergear a v5/develop? Sí/No + razones.

Si encuentras criterio incumplido: NO arregles. Reporta.
```

---

## Checklist mental antes de pegar el prompt

- [ ] ¿Estoy en el repo correcto? (`pwd` debe mostrar EspritOS)
- [ ] ¿Branch correcto? (`git branch --show-current`)
- [ ] ¿Trabajo previo commiteado o stasheado? (`git status` limpio)
- [ ] ¿Sé qué steps quiero hacer en esta sesión? (no más de 3-5 a la vez)
- [ ] ¿El blueprint dice cuánto cuestan esos steps en horas? (revisar estimación)
- [ ] ¿Tengo tiempo para esa carga? (Claude no se cansa, tú sí — no abrir 8h de scope si vas a estar 2h disponible)

---

## Antipatrones a evitar

1. **"Hazme todo el sprint S1 en una sesión"** → context overflow, baja calidad. Divide en bloques de 3-5 steps.
2. **"Trabaja en S2 y S3 al mismo tiempo"** → S3 depende de S2 cerrado y mergeado. Cero paralelismo entre sprints.
3. **"Salta S1 porque CFDI es aburrido"** → S1 es legal urgency. Saltarlo expone a multas SAT.
4. **"Usa otro modelo más rápido para los steps simples"** → consistencia stack. Sonnet o lo que uses, no mezcles.
5. **"Pega el blueprint completo al inicio"** → 198KB en context. Mejor: que Claude lea el archivo cuando lo necesite.
6. **"Aprovechemos para refactorizar X mientras estamos aquí"** → cero alcance fuera de scope. V5 ya tiene 894h.
7. **"Salta los tests, ya los hago al final"** → blueprint requiere tests por step. Sin tests no hay step completo.
8. **"Mergea S1 a main directamente"** → flujo es `v5/sprint-1-cfdi` → `v5/develop` → `main` solo en V5.0.0 release.

---

## Flujo recomendado por semana

| Día | Actividad |
|---|---|
| Lunes | Sesión Claude 2-3h: revisar último step del viernes, ejecutar próximos 2-3 steps |
| Martes | Sesión Claude 2-3h: continuar steps del sprint actual |
| Miércoles | Validación manual de lo construido + smoke test producción Cremería |
| Jueves | Sesión Claude 2-3h: completar steps faltantes del sprint actual |
| Viernes | Sesión de verificación + commit doc summary + plan próxima semana |

A este ritmo: ~10h Claude/semana × 32 semanas = ~320h. Para 894h totales necesitas ~28 semanas si das 30h/sem o ~56 semanas si das 16h/sem. Calibra realista.

---

## Cuando llamar al usuario (Beto)

- Decisión arquitectónica no especificada en blueprint
- Conflicto entre 2 secciones del blueprint
- Bloqueo externo (PAC no responde, OAuth no autoriza, dependencia rota)
- Step pide tocar producción
- Test falla y no es obvio si es bug en código nuevo o regresión real
- Más de 30 minutos atorado en mismo problema

NO llamar a Beto por:
- Decidir nombre de variable
- Elegir entre 2 implementaciones equivalentes (decide y avisa)
- Ajuste menor a un test
- Comentario español vs inglés (decidir caso por caso)

---

**FIN DEL KICKOFF V5**
