# Pulso HM — Kickoff para el builder

Eres Claude Code arrancando un proyecto nuevo: **Pulso HM**, plataforma interna de BI para Cremería HM y Abarrotera HM. Warehouse local + app Django con dashboards por rol.

## Lo primero que haces

1. **Lee el blueprint completo** en `E:\ClaudeWorks\repos-referencia\the-architect\output\pulso-hm-blueprint.md`. No lo escanees — léelo de cabo a rabo. Especial atención a Sección 8 (Build Order), Sección 14 (Reglas no negociables) y Sección 13 (CLAUDE.md para el target).
2. **Lee el design doc** en `E:\ClaudeWorks\proyectos\AnalisisVentas\docs\plans\2026-05-16-pulso-hm-design.md` para entender el "por qué".
3. **Hojea los blueprints hermanos** para captar el formato y patrones:
   - `E:\ClaudeWorks\repos-referencia\the-architect\output\portal-cremeriahm-blueprint.md` (patrón principal a replicar)
   - `E:\ClaudeWorks\repos-referencia\the-architect\output\ritmo-blueprint.md` (patrón ETL nightly)
4. **Revisa los repos vivos** para los patrones de Django + schema dedicado + Cloudflare Tunnel:
   - `E:\ClaudeWorks\proyectos\portal-cremeriahm\` (estructura, settings, db_routers, deploy)
   - `E:\ClaudeWorks\proyectos\Ritmo\` (factores pieza-kg, pre-agregación, ETL)
   - `E:\ClaudeWorks\proyectos\AnalisisVentas\cremeria\reportes\mensual\reporte_v2_canonico.py` (cifras de referencia para cuadre obligatorio Fase 4)
   - `E:\ClaudeWorks\proyectos\AnalisisVentas\abarrotera\reportes\utilidad\abarrotera_utilidad.py` (margen 12.19%, validación Fase 10)
   - `E:\ClaudeWorks\proyectos\AnalisisVentas\cremeria\comisiones\comisiones.py` (validación `fact_comision_mensual`)

## Dónde clonar el repo

```
E:\ClaudeWorks\proyectos\pulso-hm\
```

GitHub: `huheme25/pulso-hm` (privado, crear si no existe).

## Cuál es la Fase 0

**Bootstrap del repo.** Ver Sección 8 → Fase 0 del blueprint. Output esperado: container Docker que levanta Django en `localhost:8400` y se conecta al Postgres prod del stack EspritOS. Estimado 2-3 h.

**No avanzar a Fase 1 hasta que Fase 0 esté verde** (criterio: `python manage.py check` retorna 0 + container puede `SELECT 1` contra Postgres prod).

## Reglas no negociables que aplican desde Fase 0

1. **Postgres host SIEMPRE `127.0.0.1`, NUNCA `localhost`.** Settings y `.env.example`.
2. **Dockerfile usa `FROM python:3.11-slim@sha256:<digest>` pinned.**
3. **Schema `pulso` aislado con rol `pulso_user`.**
4. **Idioma español** en UI, comentarios y commits.
5. **Branding HM:** paleta `#6B4E32` + Playfair Display + Inter.

Las 20 reglas completas están en Sección 14 del blueprint.

## Cómo reportar al cierre de cada fase

En `NEXT_SESSION.md` del repo `pulso-hm` (lo creas en Fase 0), al final de cada fase escribe:

```markdown
## Fase X — <nombre> — COMPLETADA <fecha>

### Verificación ejecutada
- Comando: `<comando exacto>`
- Resultado: <output relevante>
- Criterio verde cumplido: si/no — <por qué>

### Archivos creados/modificados
- <lista>

### Decisiones tomadas durante el build (si difieren del blueprint)
- <decisión> — <por qué>

### Pendientes para próxima fase
- <bullets>

### Heads-up para Beto
- <cualquier cosa que Beto deba saber antes de la siguiente sesión>
```

## Si te atoras

- **Si un test de cuadre falla**: NO marques la fase como completa. Investiga la diferencia. Las cifras de Pulso DEBEN cuadrar con los reportes manuales ya generados (margen Aba 12.19%, comisiones Andrea/Valeria ±$50, mart_kpi_master vs `reporte_v2_canonico.py`).
- **Si el blueprint y el design doc se contradicen**: el blueprint es la fuente de verdad para implementación.
- **Si el design doc y los scripts de AnalisisVentas se contradicen**: los scripts son la fuente de verdad para el cálculo, el design/blueprint para la estructura.
- **Si encuentras una regla que no se puede cumplir**: detente, escribe en `NEXT_SESSION.md` el conflicto, y consulta con Beto antes de cambiar enfoque.

## Lo que NO debes hacer

- Saltarte fases.
- Introducir tecnologías fuera del stack (React, Vue, dbt, Airflow, etc.).
- Tocar EspritOS, Portal o Ritmo. Pulso vive aislado.
- Capturar datos. Pulso solo lee.
- Usar `localhost` para Postgres.
- Hacer commits jumbo. Una fase = uno o pocos commits coherentes.
- Avanzar si la verificación de la fase actual no quedó verde.

## Arranca

Cuando hayas leído el blueprint y los repos de referencia, anuncia que estás listo para Fase 0 y comienza. No pidas confirmación para cada paso — el blueprint es el contrato.

---

*Beto está esperando ver Pulso en `https://pulso.espritos.app` en 4-5 semanas. Build Order de 15 fases. Cada fase verde antes de avanzar. Ánimo.*
