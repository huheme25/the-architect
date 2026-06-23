# EspritOS — Kickoff Prompt

> **Para Beto:** Copia todo el contenido debajo de la línea y pégalo en una nueva sesión de Claude Code abierta en el directorio `espritos/` recién clonado. Esto arranca al builder con todo el contexto que necesita.

---

## 🎯 Prompt para pegar

Hola. Vas a construir **EspritOS** — un sistema operativo interno de 21 módulos Django para Cremería HM (distribuidora de lácteos/embutidos/carnes en Tonalá, Jalisco, 25 empleados, 2 sucursales). Reemplaza un CRM-ERP v4 actual (Next.js + Prisma, monolito acoplado) y el POS legacy Punto Zero.

**Antes de escribir una sola línea de código, haz lo siguiente:**

### 1. Lee el blueprint completo

```
E:\ClaudeWorks\repos-referencia\the-architect\output\espritos\BLUEPRINT.md
```

Son ~2,374 líneas. Léelas todas. No saltes secciones. Presta atención especial a:
- Sección 0: Reglas de oro para construir
- Sección 3: Directory Structure (21 apps Django)
- Sección 5: Reporteador IA (la pieza diferenciadora — 11 pasos del pipeline + validador SQL con 20+ tests críticos de seguridad)
- Sección 10: Build Order (13 pasos core + 7 olas A-G)
- Sección 17: Reglas no negociables
- Apéndice D: Inventario de 72 modelos Prisma mapeados a 21 apps Django + tabla de 27 fases GSD

### 2. Explora el contexto del CRM-ERP v4

La carpeta `E:\ClaudeWorks\repos-referencia\the-architect\output\espritos\context\` contiene la fuente autoritativa de todo el dominio de negocio:

- `schema.prisma` (1,542 líneas, 72 modelos) — es **tu referencia canónica** para la data model de cada módulo. No inventes modelos: copia los que están aquí con adaptaciones idiomáticas a Django.
- `docs/2026-02-10 Necesidades y Soluciones ERP en Retail.md` — requerimientos operativos reales
- `docs/facturama-api-reference.md` — integración CFDI 4.0
- `codebase/CONCERNS.md` — **lo que falló** en el v4 (evita repetir)
- `phases/` — 27 fases del v4 con RESEARCH/PLAN/SUMMARY/VERIFICATION. **ANTES de construir cada módulo de EspritOS, lee la fase correspondiente del v4**. Por ejemplo:
  - Fase 02 (`phases/02-precios-volumen-descuentos-pos/`) antes del Paso 17 (motor de precios) y Paso 19 (POS)
  - Fase 04 (`phases/04-cfdi-40-base/`) antes del Paso 23 (CFDI)
  - Fase 14 (`phases/14-comisiones-de-ventas/`) antes del Paso 29 (comisiones avanzadas)

### 3. Usa el bootstrap para arrancar

La carpeta `E:\ClaudeWorks\repos-referencia\the-architect\output\espritos\bootstrap\` tiene archivos **listos para copiar** al repositorio: `pyproject.toml`, `docker-compose.yml`, `Dockerfile`, `.env.example`, `config/settings/*.py`, `apps/core/` con modelos abstractos + mixins + test crítico de aislamiento. Esto te ahorra horas del Paso 1.

**Cómo usarlo:**
```bash
cp -r E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/* .
cp E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/.env.example .
cp E:/ClaudeWorks/repos-referencia/the-architect/output/espritos/bootstrap/.gitignore .
```

Luego sigue los pasos en `bootstrap/README.md` (generar SECRET_KEY, crear `.env`, `docker compose up`, `migrate`, `createsuperuser`, correr tests).

### 4. Ejecuta el Build Order paso a paso

Sigue los 13 pasos core del Build Order (sección 10 del blueprint). Reglas de ejecución:

- **No saltes pasos.** Cada paso depende del anterior.
- **Cada paso termina con un commit verificable**: tests pasando + smoke test manual.
- **Si un test de `test_app_isolation.py` falla, NO lo deshabilites** — arregla el cross-import que rompió la regla.
- **Antes de cada paso, crea un plan corto** (con TaskCreate si hay ≥3 sub-steps) y actualízalo.
- **Cuando entres a las olas A-G** (pasos 14-40), primero lee la fase del CRM-ERP v4 correspondiente en `context/phases/`, luego implementa.

### 5. Contexto operativo de Beto (importante)

- **Hispanohablante nativo**. Todas las conversaciones, commits y docs en español. El código en inglés (variables, funciones), comentarios en español cuando aporten contexto.
- **Moneda MXN**, **fechas DD/MM/AAAA**, **timezone America/Mexico_City** — ya configurado en `config/settings/base.py`.
- **Presupuesto total del proyecto: $100-150 USD/mes** — casi todo se va al API de Claude. Hosting on-prem en PC servidor de oficina (costo $0 mensual). Si propones servicios externos, valida que quepan en el presupuesto.
- **Primera vez con Django** — el blueprint está escrito para que te guíes tú, pero cuando expliques algo a Beto, hazlo con analogías a Next.js (su experiencia previa).
- **Manejo de procesos**: NUNCA hagas `taskkill /IM node.exe` ni similares — Beto trabaja con múltiples terminales. Mata solo PIDs específicos que tú iniciaste.

### 6. Cuando tengas dudas

- **Blueprint primero** (responde el 95%)
- **Schema.prisma del v4 segundo** (ground truth de modelos)
- **Fase correspondiente del v4 tercero** (decisiones validadas)
- **Pregúntale a Beto cuarto** (pero ponle las 2-3 opciones con tu recomendación, no preguntas abiertas)

### 7. Empieza ahora

Tu primer mensaje debe ser:

1. Un resumen de 5-10 bullets de lo que entendiste del proyecto (para que Beto valide que leíste el blueprint)
2. El plan específico del **Paso 1** con TaskCreate (sub-tareas atómicas)
3. Una pregunta si hay algo ambiguo (solo 1, la más crítica), o si no hay ambigüedad, proceder directo

No empieces a escribir código hasta que Beto confirme el plan del Paso 1.

---

**Recuerda:** Este proyecto es el **reemplazo del sistema operativo completo** de Cremería HM. Si sale bien, el negocio corre sobre EspritOS. Si sale mal, perdemos meses y tu credibilidad como constructor. La ganancia principal del nuevo stack (Django + HTMX + apps aisladas + tests obligatorios) es **cero regresiones al hacer cambios**. Esa promesa se cumple solo si respetas las reglas sagradas del blueprint.

Arranca cuando estés listo.
