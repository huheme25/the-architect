# Agenda "más Google Calendar" — Blueprint de evolución de `apps/agenda` (EspritOS)

> Generado por The Architect el 28/06/2026
> Arquetipo: Internal Tool — **evolución de una app Django ya en producción** dentro del monolito EspritOS (NO greenfield)
> Repo anfitrión: `E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS` · GitHub `huheme25/espritos` · prod https://espritos.app
> Construye: **The Builder**, sobre la rama `feature/agenda-google-feel` (ya trae el color-por-persona) o rama hija. Cero reescritura: se construye ENCIMA.

---

## 0. Cómo usar este blueprint (para The Builder)

Esto **NO es un proyecto nuevo**. `apps/agenda` ya es madura: anti-empalmes atómico (PostgreSQL `ExclusionConstraint`), invitaciones con RSVP, RLS por rol, FullCalendar v6 integrado, free/busy ya calculado (escondido). Este blueprint agrega 4 features grandes **calcando patrones que ya existen** (rutas `archivo:línea` apuntan al repo para que verifiques el original). Regla de oro: **extiende, no reescribas; calca el patrón, no lo reinventes.**

Lo que YA existe y **no se re-especifica** (ver §2): el modelo `Evento`/`Asistente`/`BloqueoCalendario`, los servicios de empalmes/RLS/`sugerir_horarios`, el endpoint `eventos_json`, el init de FullCalendar, el color-por-persona en vista equipo (✅ ya hecho).

Orden de lectura: §2 (estado actual — qué NO tocar) → §3 (decisiones cerradas) → §4–7 (las 4 features F1–F4) → §9 (build order, sigue paso por paso) → §10 (tests) → §11 (reglas).

---

## 1. Visión

La agenda funciona pero "no se siente como Google Calendar". Beto la quiere a ese nivel — **sencilla, amigable, eficiente** — para uso personal y compartido del equipo (Andrea, Valeria, Daniela, Betty, René + dirección). Norte de UX: superponer calendarios de varias personas con toggle y color por persona, eventos recurrentes, recordatorios automáticos, y proponer huecos libres antes de agendar.

---

## 2. Estado ACTUAL de `apps/agenda` (construir ENCIMA — NO rehacer)

> Todo lo de esta sección **ya está en prod**. El blueprint lo da por hecho. Solo se cita lo que las features nuevas tocan.

### 2.1 Modelo (`apps/agenda/models.py`)
- **`Evento`** (`:83-279`, hereda `AuditedModel` → trae `created_at/updated_at/created_by/updated_by`). Campos: `tipo` (7 tipos `TipoEvento`), `titulo`, `descripcion`, `inicio`, `fin`, `todo_dia`, `estado` (5 `EstadoEvento`), `lugar`, `resultado`, `owner` (FK User PROTECT), `sucursal` (FK nullable), `territory` (FK nullable), GenericFK (`entidad_tipo`/`entidad_id`/`entidad`), `google_event_id`/`outlook_event_id`/`sync_status`, y **`serie_id = UUIDField(null=True, blank=True, db_index=True, help_text="RRULE futuro. Eventos de la misma serie comparten UUID.")`** ← reservado, vacío, para F2.
  - **`ExclusionConstraint` GiST** `agenda_evento_no_overlap_per_owner` (`:166-180`): impide empalmes por `owner` + `TSTZRANGE(inicio,fin)` OVERLAPS, **condición** `estado IN (AGENDADO, EN_CURSO) AND tipo != PERSONAL`. + `CheckConstraint fin>inicio`. Requiere extensión `btree_gist` (migración 0001).
  - `clean()` valida empalmes con mensaje amistoso antes del IntegrityError; `save()` deriva sucursal/territory; `color_hex` da color por tipo.
- **`Asistente`** (`:281-320`): M2M Evento↔User, `rol` (ORGANIZADOR/REQUERIDO/OPCIONAL), `respuesta` (PENDIENTE/ACEPTADO/RECHAZADO), `unique_together(evento,user)`.
- **`BloqueoCalendario`** (`:322-354`): vacaciones por usuario (DateField), informativo, NO entra a empalmes.

### 2.2 Servicios (`apps/agenda/services.py`)
- `detectar_empalmes` (`:30-52`), `conflictos_de_invitados` (`:55-89`), **`sugerir_horarios(usuarios, inicio, fin, *, excluir_id=None, max_sugerencias=3)` (`:92-172`)** — ya pre-carga eventos del día en **una query** e itera en memoria (sin N+1); devuelve `list[tuple(inicio, fin)]` tz-aware respetando `apps.core.services.calendario_hm` (L-S 7-16). **Listo para F4.**
- Mutaciones: `crear_evento` (`:180-230`, llama `full_clean()`+`save()`, crea Asistentes; **NO dispara la señal** — eso lo hacen las views), `reagendar` (`:233-240`), `marcar_completado`/`no_show`/`cancelar` (`:243-270`).
- RLS: `eventos_visibles_para` (`:278-298`), `eventos_equipo` (`:301-321`, toda la empresa, decisión 09/06/2026), `_aplicar_rls_eventos` (`:324-358`), `eventos_por_entidad` (`:366-380`, horizontal para fichas), `bloqueos_visibles_para` (`:383-387`).

### 2.3 Vistas / endpoints (`apps/agenda/views.py`, `urls.py`)
- `/agenda/` (`calendario`), `/agenda/equipo/` (`calendario_equipo`), `/agenda/api/eventos/` (`eventos_json`, `:118-180`, único JSON FullCalendar, `?modo=personal|equipo`), CRUD HTMX (crear `:279-334`, editar `:374-413`, reagendar drag `:436-458`, completar/cancelar/no-show, RSVP `:502-521`). La señal `invitados_agregados` se dispara en `_notificar_invitados_nuevos` (`:416-433`).
- **Color por persona** (`_PALETA_PERSONA` + `_color_persona`, `:42-56`): en modo equipo el color es por `owner_id`; PERSONAL ajeno se enmascara como "Ocupado" gris. Editable por-evento (solo owner/admin).

### 2.4 Frontend (`templates/agenda/`)
- `calendario.html` — init FullCalendar v6.1.15 (`:339-506`): mes/semana/día/lista, `nowIndicator`, `select`→crear, `eventDrop`/`eventResize`→reagendar, tecla `n`, modal `<dialog>`, **`htmx:beforeSwap` fuerza swap de 422** (`:492`). Móvil: `listWeek`. Leyenda por tipo (`:40-48`).
- `partials/evento_form.html` (Alpine `eventoForm()`, inputs nativos), `partials/evento_detalle.html` (detalle + RSVP).

### 2.5 Tests (`apps/agenda/tests/`)
`test_rls.py`, `test_colaborativa.py`, `test_models.py`. **Cada feature nueva exige tests equivalentes.**

---

## 3. Decisiones de diseño cerradas (con Beto, 28/06/2026)

| Tema | Decisión | Implicación |
|------|----------|-------------|
| **Vista timeline** | **Rejilla custom server-rendered (HTMX)** — NO FullCalendar Premium | Cero licencia. Una fila por persona. Calca la interacción HTMX que el brief original ya imaginó. |
| **Edición de series recurrentes** | **Completa, 3 modos** estilo Google: "solo este" / "este y los siguientes" / "toda la serie" | Requiere split de series + overrides + EXDATE (§5). |
| **Drag en timeline** | **Clic primero, drag después** | MVP timeline: clic-en-slot crea para esa persona + clic-en-evento ve detalle. Drag-reagenda en timeline → MVP+1. (El calendario mes/semana ya tiene drag.) |
| **Recurrencia: almacenamiento** | Modelo `SerieRecurrente` (plantilla) + `Evento` instancias materializadas con `serie_id` | El `ExclusionConstraint` físico exige filas reales → **materializar, no expandir virtual** (§5.1). |
| **Recordatorios** | Modelo `Recordatorio` (ya esbozado en el design-doc 03/06) + tarea Celery Beat | Beat es DB-backed (`django_celery_beat`), registro por migración (§6). |

---

## 4. F1 — Calendarios por persona superponibles + vista timeline  *(prioridad ALTA)*

El gap #1 de "no es Google". Hoy el modo equipo es un volcado plano; no puedes prender/apagar "el calendario de Andrea".

### 4.1 Toggle por persona (panel lateral)
- **Backend — filtro `owners` en `eventos_json`** (`apps/agenda/views.py:118-180`): aceptar `?owners=8,9,10`. Tras obtener el qs vía `eventos_equipo`/`eventos_visibles_para`, si viene `owners`, filtrar `qs = qs.filter(owner_id__in=owners)`. **No reemplaza el RLS** — se aplica DESPUÉS de `eventos_equipo` (que ya enmascara PERSONAL). Es filtro de visualización, no de seguridad.
  ```python
  owners_param = request.GET.get("owners", "").strip()
  if owners_param:
      try:
          owner_ids = [int(x) for x in owners_param.split(",") if x]
          qs = qs.filter(owner_id__in=owner_ids)
      except ValueError:
          pass  # ignora param malformado, no rompe la carga
  ```
- **Context — lista de personas para el panel**: en `calendario_equipo`, agregar al context la lista de personas superponibles con su color. Calca `_color_persona`:
  ```python
  from django.contrib.auth import get_user_model
  personas = [
      {"id": u.pk, "nombre": u.get_full_name() or u.username,
       "color": _color_persona(u.pk)}
      for u in get_user_model().objects.filter(is_active=True, eventos_owned__isnull=False).distinct().order_by("first_name")
  ]
  ctx["personas"] = personas
  ```
  > RLS: el modo equipo ya es visible para toda la empresa (decisión 09/06/2026) y enmascara PERSONAL ajeno. El panel lista al equipo; el filtro `owners` solo recorta la vista. Una vendedora pura en modo **personal** no ve panel (es solo ella).
- **Frontend — panel Alpine** en `calendario.html` (solo cuando `modo == "equipo"`): lista de checkboxes con el color de cada persona. Al togglear, recalcula el set de `owners` activos y llama `calendar.refetchEvents()`. El `fetchEventos` agrega `&owners=${activos.join(",")}` a la URL. **Extiende el init existente, no lo reescribe.**
  ```html
  {% if modo == "equipo" %}
  <aside x-data="panelPersonas()" class="...">
    <p class="text-xs uppercase tracking-wide text-zinc-500 mb-2">Calendarios</p>
    {% for p in personas %}
    <label class="flex items-center gap-2 py-1 cursor-pointer">
      <input type="checkbox" value="{{ p.id }}" checked
             @change="toggle({{ p.id }})" class="hm-checkbox">
      <span class="w-2.5 h-2.5 rounded-sm" style="background:{{ p.color }}"></span>
      <span class="text-sm text-zinc-300">{{ p.nombre }}</span>
    </label>
    {% endfor %}
  </aside>
  {% endif %}
  ```
  ```javascript
  function panelPersonas() {
    return {
      activos: new Set(Array.from(document.querySelectorAll('aside input:checked')).map(i => i.value)),
      toggle(id) {
        const s = String(id);
        if (this.activos.has(s)) this.activos.delete(s); else this.activos.add(s);
        window.__ownersFiltro = Array.from(this.activos).join(",");  // leído por fetchEventos
        window.dispatchEvent(new CustomEvent("agenda-owners-changed"));
      },
    };
  }
  // En fetchEventos (calendario.html), añadir a la URL:
  //   const owners = window.__ownersFiltro;
  //   if (owners !== undefined) url += `&owners=${encodeURIComponent(owners)}`;
  // Y: document.body.addEventListener("agenda-owners-changed", () => calendar.refetchEvents());
  ```

### 4.2 Leyenda por persona (quick win pendiente, cerrar junto)
En modo equipo, la leyenda hoy es "por tipo" (`calendario.html:40-48`). Cuando `modo == "equipo"`, renderizar leyenda **por persona** (reutilizar `personas` con su color). Mantener la leyenda por tipo en modo personal.

### 4.3 Vista timeline "una fila por persona" (server-rendered, custom)
La vista que el brief original definió (§4.4: *"timeline horizontal con una fila por vendedora"*) y nunca se construyó. Para dirección/gerencia: coordinar al equipo de un vistazo.

- **Ruta nueva**: `/agenda/timeline/` (`timeline_view`) + endpoint de datos `/agenda/api/timeline/?fecha=YYYY-MM-DD&owners=...` que devuelve **HTML** (HTMX, no JSON) — una rejilla server-rendered.
- **Estructura**: una columna de horas (7–16, calca `calendario_hm`), una **fila por persona** (de `personas`, filtrable por `owners`). Cada evento se posiciona en su fila con `left`/`width` calculados desde inicio/fin (porcentaje del día hábil). Color por persona. PERSONAL ajeno = "Ocupado" gris (reutilizar la misma lógica de enmascarado de `eventos_json` — **extraer a un helper** `_evento_a_payload(ev, user, modo)` para no duplicar la regla de enmascarado entre `eventos_json` y la timeline).
- **Interacción (MVP, clic-primero)**:
  - Clic en **slot vacío de una fila** → `hx-get` a `/agenda/evento/nuevo/?inicio=...&owner=<persona>` → modal de crear **pre-asignado a esa persona** (gerencia/dirección puede crear para otros; respetar `_check_edicion`).
  - Clic en **evento** → `hx-get` a `/agenda/evento/<pk>/` → modal detalle (el existente).
  - Drag-reagenda en timeline → **MVP+1** (no en esta entrega).
- **N+1**: una sola query de eventos del día para todos los `owners` (calca el patrón de `sugerir_horarios`: una query + iterar en memoria agrupando por `owner_id`). Test de performance obligatorio (<20 queries).
- **RLS**: el endpoint pasa por `eventos_equipo`/`eventos_visibles_para`. Una vendedora pura ve la timeline del equipo igual que el modo equipo (decisión 09/06), con PERSONAL enmascarado.
- **Acceso en sidebar**: la timeline es sobre todo para dirección/gerencia. Exponerla como sub-item o botón "Timeline equipo" dentro de la vista equipo (no requiere módulo nuevo — sigue siendo módulo `agenda`).

> **Refactor previo recomendado (paso 1 de F1)**: extraer la construcción del payload de un evento (color, enmascarado, editable) de `eventos_json` a un helper compartido `_evento_payload(ev, *, user, modo)`. Así la timeline y el JSON usan exactamente la misma regla de privacidad/color. Cero cambio de comportamiento en `eventos_json` (test de regresión).

---

## 5. F2 — Recurrencia (RRULE)  *(prioridad ALTA — la más delicada)*

### 5.1 Decisión: materializar instancias (no expansión virtual)

El `ExclusionConstraint` (`apps/agenda/models.py:166-180`) detecta empalmes sobre **filas físicas** (`TSTZRANGE` por owner). La expansión virtual (`dateutil.rrule` en lectura) **no puede coexistir** con él: las ocurrencias virtuales no existen como filas, así que no competirían por el slot ni respetarían el anti-empalmes, y romperían Asistentes/GenericFK/RSVP por instancia. **Por eso se materializan instancias reales** (cada una es un `Evento` con `serie_id`), dentro de una **ventana de ~1 año**, regenerada por una tarea Beat.

### 5.2 Modelo `SerieRecurrente` (nuevo) + campos en `Evento`

```python
# apps/agenda/models.py — AGREGAR
import uuid
from datetime import datetime, timedelta


class SerieRecurrente(models.Model):
    """Definición (plantilla) de una serie de eventos recurrentes.

    Las ocurrencias son Evento reales (materializados) que comparten
    Evento.serie_id == SerieRecurrente.id. Se materializan ~1 año adelante;
    una tarea Beat regenera la ventana. El ExclusionConstraint físico exige
    filas reales — por eso materializamos en vez de expandir en lectura.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    owner = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT,
        related_name="series_recurrentes",
    )
    # --- Plantilla del evento (se copia a cada instancia) ---
    tipo = models.CharField(max_length=20, choices=TipoEvento.choices)
    titulo = models.CharField(max_length=200)
    descripcion = models.TextField(blank=True)
    lugar = models.CharField(max_length=255, blank=True)
    hora_inicio = models.TimeField(help_text="Hora local de cada ocurrencia.")
    duracion_min = models.PositiveIntegerField(default=60)
    sucursal = models.ForeignKey(
        "core.Sucursal", null=True, blank=True, on_delete=models.SET_NULL, related_name="+",
    )
    # --- Definición de recurrencia (RFC 5545) ---
    rrule_text = models.CharField(
        max_length=255,
        help_text="RRULE sin DTSTART, ej. 'FREQ=WEEKLY;BYDAY=MO,WE'.",
    )
    dtstart = models.DateField(help_text="Fecha de la primera ocurrencia.")
    until = models.DateField(null=True, blank=True, help_text="Fin de la serie (inclusive).")
    count = models.PositiveIntegerField(null=True, blank=True, help_text="Máx. ocurrencias (alternativa a until).")
    exdates = models.JSONField(
        default=list, blank=True,
        help_text="Fechas ISO excluidas (ocurrencias canceladas/movidas).",
    )
    activa = models.BooleanField(default=True)
    tenant = models.ForeignKey(
        "tenants.Tenant", null=True, blank=True, on_delete=models.PROTECT, related_name="+",
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "agenda_series_recurrentes"
        verbose_name = "Serie recurrente"
        verbose_name_plural = "Series recurrentes"
        indexes = [models.Index(fields=["owner", "activa"])]

    def __str__(self) -> str:
        return f"{self.titulo} [{self.rrule_text}]"
```

```python
# apps/agenda/models.py — AGREGAR a Evento (además del serie_id existente):
    # serie_id ya existe (:99-102): UUIDField null/blank, apunta a SerieRecurrente.id.
    es_excepcion = models.BooleanField(
        default=False,
        help_text="True si esta ocurrencia fue editada individualmente ('solo este'). "
                  "La regeneración de la serie la respeta (no la pisa ni borra).",
    )
    ocurrencia_fecha = models.DateField(
        null=True, blank=True, db_index=True,
        help_text="Fecha base de esta ocurrencia en la serie (para idempotencia de la materialización).",
    )
```

> **Migración**: nueva migración para `SerieRecurrente` + `AddField` de `es_excepcion`/`ocurrencia_fecha` en `Evento`. NO editar migraciones aplicadas (regla §8.7). No toca el `ExclusionConstraint` ni `btree_gist`.

### 5.3 Materialización — `services.py` (nuevo)

```python
def materializar_serie(serie: SerieRecurrente, *, hasta=None, creado_por=None) -> dict:
    """Crea las instancias Evento que faltan de `serie` hasta `hasta`
    (default: hoy + 365 días). Idempotente: respeta exdates, instancias
    es_excepcion y las ya existentes (por serie_id + ocurrencia_fecha).

    Cada instancia se inserta en su propio savepoint: si choca con el
    ExclusionConstraint (slot ocupado), se SALTA y se cuenta (no rompe el lote).
    Devuelve {creadas, saltadas_empalme, saltadas_existentes}.
    """
    from datetime import date, timedelta
    from dateutil.rrule import rrulestr
    from django.db import IntegrityError, transaction
    from django.utils import timezone as dj_tz

    if hasta is None:
        hasta = dj_tz.localdate() + timedelta(days=365)

    # Construir el RRULE con DTSTART. dateutil maneja FREQ/BYDAY/UNTIL/COUNT.
    dt0 = datetime.combine(serie.dtstart, serie.hora_inicio)
    rule = rrulestr(serie.rrule_text, dtstart=dt0)

    exdates = set(serie.exdates or [])
    existentes = set(
        Evento.objects.filter(serie_id=serie.id)
        .values_list("ocurrencia_fecha", flat=True)
    )
    creadas = saltadas_empalme = saltadas_existentes = 0
    limite = serie.until or hasta

    for occ in rule:
        f = occ.date()
        if f > limite:
            break
        if serie.count and creadas >= serie.count:
            break
        if f.isoformat() in exdates:
            continue
        if f in existentes:
            saltadas_existentes += 1
            continue
        inicio = dj_tz.make_aware(datetime.combine(f, serie.hora_inicio))
        fin = inicio + timedelta(minutes=serie.duracion_min)
        ev = Evento(
            owner=serie.owner, tipo=serie.tipo, titulo=serie.titulo,
            descripcion=serie.descripcion, lugar=serie.lugar,
            inicio=inicio, fin=fin, sucursal=serie.sucursal, tenant=serie.tenant,
            serie_id=serie.id, ocurrencia_fecha=f,
            created_by=creado_por or serie.owner, updated_by=creado_por or serie.owner,
        )
        try:
            with transaction.atomic():  # savepoint por instancia
                ev.save()
            creadas += 1
        except IntegrityError:
            saltadas_empalme += 1  # slot ocupado → se salta, se reporta
    return {"creadas": creadas, "saltadas_empalme": saltadas_empalme,
            "saltadas_existentes": saltadas_existentes}
```

> **Por qué savepoint por instancia y no `bulk_create(ignore_conflicts=True)`**: Postgres `ON CONFLICT` NO soporta exclusion constraints; `bulk_create(ignore_conflicts=True)` lanzaría `IntegrityError` igual. La inserción una-por-una con `transaction.atomic()` anidado captura el choque y salta solo esa instancia. Para ~52 ocurrencias/año por serie es perfectamente manejable.

> **Dependencia**: `python-dateutil` (para `rrulestr`). Verificar que esté en `requirements`/`pyproject.toml`; si no, agregarlo (es estándar y liviano).

### 5.4 Edición de series — 3 modos (semántica Google)

| Modo | Qué hace | Cómo |
|------|----------|------|
| **Solo este** (editar) | La ocurrencia se vuelve independiente; la serie no la vuelve a tocar | `evento.es_excepcion = True; evento.save()`. La regeneración respeta `es_excepcion`. Si se **mueve de fecha**, agregar la `ocurrencia_fecha` original a `serie.exdates` para que no se re-materialice esa ranura. |
| **Solo este** (cancelar ocurrencia) | Borra/cancela una sola fecha | Borrar el Evento + agregar su `ocurrencia_fecha` a `serie.exdates` (EXDATE). |
| **Este y los siguientes** | Split de la serie en esta fecha | A la serie vieja: `until = ocurrencia_fecha - 1 día`; borrar sus instancias futuras (`inicio >= ocurrencia` y `not es_excepcion`). Crear **nueva** `SerieRecurrente` con los campos editados + `dtstart = ocurrencia_fecha`; `materializar_serie(nueva)`. |
| **Toda la serie** | Edita la definición y propaga | Actualizar campos de `SerieRecurrente`; actualizar instancias **futuras** (`inicio > now`) que **no** sean `es_excepcion` (re-aplicar título/hora/duración/lugar) o borrarlas y `materializar_serie` de nuevo. Las **pasadas** se conservan como histórico. Las `es_excepcion` se respetan. |

- Servicios nuevos: `editar_ocurrencia_solo_esta`, `editar_serie_este_y_siguientes`, `editar_serie_completa`, `cancelar_ocurrencia` — cada uno `@transaction.atomic`. Todos respetan el `ExclusionConstraint` (operan sobre Eventos reales).
- **UI**: al editar/eliminar un evento con `serie_id`, el modal muestra un selector de alcance (radio: "Solo este" / "Este y los siguientes" / "Toda la serie") — calca el modal `<dialog>` existente. El form de creación gana un bloque "Repetir" (frecuencia + intervalo + fin) que arma el `rrule_text` (Alpine, sin librería pesada: select diario/semanal/mensual + BYDAY checkboxes para semanal).

### 5.5 Interacción con anti-empalmes
Cada instancia materializada compite por el slot del owner igual que un evento normal (es una fila real con `estado=AGENDADO`, `tipo != PERSONAL`). El `ExclusionConstraint` la protege. Las instancias que chocarían con un evento existente se **saltan** en la materialización (§5.3) y se reportan al usuario ("3 ocurrencias no se crearon por empalme"). Documentar en la UI.

### 5.6 Tarea Beat de regeneración
`apps/agenda/tasks.py`: `@shared_task regenerar_series_recurrentes()` → para cada `SerieRecurrente.activa`, `materializar_serie(serie)`. Registrada por migración (patrón §6.3) con `CrontabSchedule` nocturno (ej. 03:00 MX, `enabled=True`).

---

## 6. F3 — Recordatorios automáticos  *(prioridad MEDIA — infra lista)*

Aviso configurable (default 15 min y 1 h antes) → notificación in-app + correo. Reusa Celery + Beat + `apps/notifications` (patrón de señales ya existente).

### 6.1 Modelo `Recordatorio` (calca el esbozo del design-doc 03/06)

```python
# apps/agenda/models.py — AGREGAR
class Recordatorio(models.Model):
    class Canal(models.TextChoices):
        IN_APP = "IN_APP", "Notificación in-app"
        EMAIL = "EMAIL", "Correo"

    evento = models.ForeignKey(
        Evento, on_delete=models.CASCADE, related_name="recordatorios",
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="recordatorios_agenda",
    )
    minutos_antes = models.PositiveIntegerField(default=15)
    canal = models.CharField(max_length=10, choices=Canal.choices, default=Canal.IN_APP)
    enviado_at = models.DateTimeField(null=True, blank=True, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "agenda_recordatorios"
        verbose_name = "Recordatorio"
        verbose_name_plural = "Recordatorios"
        indexes = [
            models.Index(fields=["enviado_at", "evento"]),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=["evento", "user", "minutos_antes", "canal"],
                name="uq_recordatorio_unico",
            ),
        ]
```

- **Defaults**: al crear un evento (con owner + asistentes), auto-crear `Recordatorio(IN_APP, 15)` para el owner. La UI permite agregar/quitar offsets (15m, 1h, 1día) y canal. Para los asistentes que aceptan, opcional crear recordatorio in-app (MVP: solo owner; asistentes en MVP+1).

### 6.2 Señal + tarea Beat
- `apps/agenda/signals.py`: agregar `recordatorio_disparado = Signal()` (kwargs `recordatorio, evento, user`). Calca `invitados_agregados` (`:22`).
- `apps/notifications/signals.py`: receiver `on_recordatorio_disparado` → `services.notificar_recordatorio(...)` (in-app vía `notify()` + correo `.delay()` con fallback síncrono — mismo patrón que invitaciones).
- `apps/agenda/tasks.py`: tarea que corre **cada 5 min**:
  ```python
  @shared_task(name="apps.agenda.tasks.check_recordatorios")
  def check_recordatorios():
      """Dispara recordatorios cuya hora llegó. Idempotente vía enviado_at."""
      from django.utils import timezone as dj_tz
      from datetime import timedelta
      from apps.agenda.models import Recordatorio, EstadoEvento
      from apps.agenda.signals import recordatorio_disparado
      ahora = dj_tz.now()
      pendientes = (
          Recordatorio.objects.filter(enviado_at__isnull=True)
          .select_related("evento", "user")
          .filter(evento__estado__in=[EstadoEvento.AGENDADO, EstadoEvento.EN_CURSO])
      )
      disparados = 0
      for r in pendientes:
          umbral = r.evento.inicio - timedelta(minutes=r.minutos_antes)
          if umbral <= ahora < r.evento.inicio:  # llegó la hora y el evento no ha pasado
              recordatorio_disparado.send(
                  sender=Recordatorio, recordatorio=r, evento=r.evento, user=r.user,
              )
              r.enviado_at = ahora
              r.save(update_fields=["enviado_at"])
              disparados += 1
      return f"recordatorios disparados={disparados}"
  ```
- **Registro Beat** por migración (`apps/agenda/migrations/00XX_beat_recordatorios.py`), patrón verbatim de `apps/notifications/migrations/0003_beat_push_semanal.py`:
  ```python
  cron_5min, _ = CrontabSchedule.objects.get_or_create(
      minute="*/5", hour="*", day_of_week="*", day_of_month="*",
      month_of_year="*", timezone="America/Mexico_City",
  )
  PeriodicTask.objects.update_or_create(
      name="agenda-check-recordatorios",
      defaults={"task": "apps.agenda.tasks.check_recordatorios",
                "crontab": cron_5min, "enabled": True,
                "description": "Dispara recordatorios de agenda próximos cada 5 min."},
  )
  # Y la de regeneración de series (§5.6) con cron nocturno.
  ```
- **Idempotencia**: `enviado_at` impide re-disparo. La ventana `umbral <= ahora < inicio` evita disparar recordatorios de eventos ya pasados (si Beat estuvo caído). PERSONAL/BloqueoCalendario no spamea a otros: el recordatorio es del `user` dueño del recordatorio, no de terceros.

---

## 7. F4 — Free/busy proactivo  *(prioridad MEDIA — reusa lo existente)*

`sugerir_horarios()` ya hace el cálculo (§2.2). Solo falta exponerlo.

- **Endpoint** `apps/agenda/views.py`: `/agenda/api/horarios-libres/` (`horarios_libres_json`, `@require_GET`): lee `?participantes=8,9,10&inicio=...&fin=...`, llama `services.sugerir_horarios(usuarios, inicio, fin)`, devuelve los huecos (HTML de chips para HTMX, o JSON). Pasa por RLS (solo usuarios que el requester puede ver como participantes).
- **UI**: botón **"Buscar hora libre"** en `evento_form.html`. El organizador elige participantes (los Asistentes ya seleccionados + él) → `hx-get` al endpoint → render de chips "10:00–11:00 · 11:30–12:30 · 14:00–15:00" → clic en un chip rellena `inicio`/`fin` del form (Alpine). Calca el mecanismo de sugerencias que hoy aparece en el warning de conflicto (`_warnings_de_invitados`).
- **Opcional (MVP+1)**: mini-vista free/busy (barras de ocupado por participante) al elegir invitados.

---

## 8. Quick wins de UX (pulido barato — intercalar)

| Quick win | Estado | Dónde |
|-----------|--------|-------|
| Color por persona en vista equipo | ✅ **YA HECHO** | `eventos_json` `_color_persona` |
| Leyenda por persona en modo equipo | ⏳ cerrar con F1 | `calendario.html:40-48` |
| "Fin" siempre visible + auto-relleno +1h | ⏳ pulido | `evento_form.html` |
| Botón "Buscar hora libre" | → es F4 | — |
| Búsqueda de eventos / salto a fecha | ⏳ baja prioridad | nueva |
| Mobile font / más atajos teclado | ⏳ baja prioridad | `calendario.html` |

---

## 9. Build Order (priorizado — sigue paso por paso)

> Rama: `feature/agenda-google-feel` (o hija). Tests aislados (`scripts/test_docker.ps1`, proyecto `espritos-test`). Commits atómicos. Cierre: merge a `master` → `scripts/deploy_prod.ps1`.

**Bloque F1 — Calendarios superponibles + timeline**
1. **Refactor sin cambio de comportamiento**: extraer `_evento_payload(ev, *, user, modo)` de `eventos_json` (regla color/enmascarado/editable). Test de regresión de `eventos_json`.
2. Filtro `?owners=` en `eventos_json` + `personas` en context de `calendario_equipo`. Test RLS (vendedora no ve PERSONAL ajeno aunque filtre).
3. Panel lateral Alpine + leyenda por persona en `calendario.html` (modo equipo). `npm run build:css`.
4. Timeline: ruta `/agenda/timeline/` + endpoint `/agenda/api/timeline/` (HTML, una query, agrupa por owner en memoria) + template rejilla. Clic-crea (pre-asignado a persona) y clic-detalle. Test de performance (<20 queries) + RLS.

**Bloque F2 — Recurrencia**
5. Modelo `SerieRecurrente` + campos `es_excepcion`/`ocurrencia_fecha` en `Evento`. Migración. Verificar `python-dateutil` en deps.
6. `materializar_serie` (savepoint por instancia) + tests (idempotencia, exdates, salto por empalme).
7. Servicios de edición 3 modos + `cancelar_ocurrencia`. Tests de cada modo (split, override, propagación; respeto de `es_excepcion`).
8. UI: bloque "Repetir" en el form + selector de alcance en editar/eliminar. `npm run build:css`.
9. Tarea Beat `regenerar_series_recurrentes` + registro por migración.

**Bloque F3 — Recordatorios**
10. Modelo `Recordatorio` + migración. Auto-crear default al crear evento.
11. Señal `recordatorio_disparado` + receiver en notifications + `notificar_recordatorio` (in-app + correo, fallback síncrono).
12. Tarea Beat `check_recordatorios` (cada 5 min) + registro por migración. Test de idempotencia (no re-dispara) + ventana (no dispara pasados).
13. UI mínima: agregar/quitar offsets de recordatorio en el form.

**Bloque F4 — Free/busy**
14. Endpoint `/agenda/api/horarios-libres/` (reusa `sugerir_horarios`) + botón "Buscar hora libre" + chips clicables en el form. Test RLS.

**Pulido + cierre**
15. Quick wins restantes (fin visible, búsqueda, atajos) según tiempo.
16. Suite completa verde (`test_docker.ps1`) + `test_app_isolation`. Merge a `master` → `deploy_prod.ps1` (migrate aplica modelos + registra tareas Beat; habilitar `PeriodicTask` si quedaron `enabled=False`).

---

## 10. Testing Strategy

Pytest + `scripts/test_docker.ps1`. Fixtures calcan `apps/agenda/tests/conftest.py`.

- **F1**: `test_owners_filter` (filtra por persona), `test_timeline_rls` (vendedora no ve PERSONAL ajeno en timeline), `test_timeline_no_n_plus_1` (`CaptureQueriesContext` < 20).
- **F2** (la crítica): `test_materializar_idempotente` (correr 2x no duplica), `test_exdate_respetada`, `test_empalme_salta_instancia`, `test_editar_solo_este` (es_excepcion + exdate), `test_split_este_y_siguientes` (until viejo + serie nueva), `test_editar_toda_la_serie` (propaga a futuras, respeta excepciones, no toca pasadas), `test_instancia_compite_por_slot` (ExclusionConstraint).
- **F3**: `test_recordatorio_dispara_en_ventana`, `test_recordatorio_idempotente` (no re-dispara), `test_no_dispara_evento_pasado`, `test_recordatorio_in_app_y_correo`.
- **F4**: `test_horarios_libres_endpoint`, `test_horarios_libres_rls`.
- **Regresión**: `eventos_json` sin cambio de comportamiento tras el refactor del paso 1.
- **Aislamiento** (`apps/core/tests/test_app_isolation.py`): `agenda` sigue sin importar apps no permitidas; las features nuevas usan señales (recordatorios) y `core.services.calendario_hm`. `notifications` importa de `agenda` (permitido: agenda en `IMPORTABLE_BY_ALL`).

---

## 11. Reglas No Negociables (el blueprint las impone al Builder)

1. **RLS obligatorio + test** (regla #4 EspritOS): todo endpoint nuevo (owners, timeline, horarios-libres) filtra por usuario/rol vía los servicios existentes. Test que A no ve lo que no debe.
2. **CSRF en cada `hx-post`** (`memory/htmx_csrf_sin_config_global`): `{% csrf_token %}` / `X-CSRFToken`. El test client no valida CSRF — el 403 solo sale en prod.
3. **HTMX descarta 4xx** (`memory/htmx_swap_422_validacion`): reusar el `htmx:beforeSwap` existente (`calendario.html:492`) para avisos 422.
4. **Comentarios Django multilínea** (`memory/django_comments_multilinea`): `{# #}` una línea; multilínea = `{% comment %}`. Validar cada `.html`.
5. **Tailwind compilado** (`memory/tailwind_build_real`): `npm run build:css` al cambiar clases. Cache-busting vía `STORAGES`.
6. **Date/time picker nativo en modales** (`memory/flatpickr_dialog_static`): `<input type="datetime-local">`/`date`, NO Flatpickr.
7. **NO editar migraciones aplicadas** (regla #5): nuevas migraciones para `SerieRecurrente`/`Recordatorio`/campos. Cuidado con el `ExclusionConstraint` y `btree_gist` — NO recrearlos.
8. **No queries N+1** (regla #6): `select_related`/`prefetch_related` (owner, asistentes, serie, recordatorios). La timeline y la materialización siguen el patrón "una query, iterar en memoria" de `sugerir_horarios`. Test de perf.
9. **Aislamiento de apps** (regla #2): `agenda` no importa apps no permitidas. Recordatorios → señal hacia `notifications` (no import). Mantener `agenda` en `IMPORTABLE_BY_ALL` y su interfaz horizontal (`eventos_por_entidad`, etc.).
10. **No romper el anti-empalmes**: cada Evento (incl. instancias recurrentes) pasa por `full_clean()`/`ExclusionConstraint`. La materialización salta empalmes con savepoint por instancia (NO `bulk_create(ignore_conflicts)` — no aplica a exclusion constraints).
11. **Tests aislados** (`scripts/test_docker.ps1`, proyecto `espritos-test`). **Deploy** solo desde `master`, vía `scripts/deploy_prod.ps1`. Cero cambio de lógica sin test.

---

## 12. Notas para el CLAUDE.md de la app (delta — agregar a `apps/agenda/`)

> La app ya existe; esto se **suma** a su documentación. Bloque para `apps/agenda/CLAUDE.md` (o la sección de agenda en el CLAUDE del repo).

```markdown
## apps/agenda — features Google-feel (F1–F4)

### Recurrencia (F2)
- `SerieRecurrente` = plantilla (rrule_text RFC-5545 sin DTSTART, dtstart, until/count, exdates).
- Los Evento son instancias REALES materializadas con `serie_id` + `ocurrencia_fecha`.
  Se materializa ~1 año adelante (`services.materializar_serie`, idempotente). NO hay
  expansión virtual — el ExclusionConstraint físico exige filas reales.
- `Evento.es_excepcion=True` → ocurrencia editada individualmente; la regeneración la respeta.
- Editar serie: "solo este" (es_excepcion + exdate) / "este y siguientes" (split: until viejo
  + serie nueva) / "toda la serie" (propaga a futuras no-excepción). Todo en services, atómico.
- Materializar salta empalmes con savepoint por instancia (NO bulk_create ignore_conflicts:
  Postgres ON CONFLICT no soporta exclusion constraints).
- Dep: python-dateutil (rrulestr). Tarea Beat `regenerar_series_recurrentes` (nocturna).

### Recordatorios (F3)
- `Recordatorio(evento, user, minutos_antes, canal, enviado_at)`. Default 15min in-app al owner.
- Tarea Beat `check_recordatorios` cada 5 min → señal `recordatorio_disparado` → notifications
  (in-app + correo, fallback síncrono). Idempotente vía enviado_at; ventana evita disparar pasados.

### Calendarios superponibles + timeline (F1)
- `eventos_json?owners=8,9,10` filtra por persona (DESPUÉS del RLS, no lo reemplaza).
- `_evento_payload(ev, *, user, modo)` = regla única de color/enmascarado (usada por JSON y timeline).
- Timeline `/agenda/timeline/`: rejilla server-rendered (HTMX), una fila por persona, una query +
  agrupar en memoria. Clic-crea (pre-asignado) / clic-detalle. Drag = MVP+1.

### Free/busy (F4)
- `/agenda/api/horarios-libres/?participantes=...` reusa `services.sugerir_horarios` (ya hecho).
  Botón "Buscar hora libre" en el form → chips clicables.

### Reglas (heredadas + nuevas)
- No expansión virtual de recurrencia. No bulk_create con ignore_conflicts en eventos.
- Recordatorios y cross-módulo SIEMPRE vía señal (agenda no importa notifications).
- Cada endpoint nuevo pasa por RLS (eventos_visibles_para/eventos_equipo). Test obligatorio.
```

---

## 13. Verificación de cierre (smoke manual)

Tras suite verde, en local/preview:
1. **F1**: en vista equipo, apagar el calendario de una persona en el panel → sus eventos desaparecen; los demás conservan su color. Abrir `/agenda/timeline/` → una fila por persona; clic en slot vacío de Andrea → modal de crear pre-asignado a Andrea.
2. **F2**: crear evento "Junta semanal" que repite cada lunes → se materializan las ocurrencias. Editar una y elegir "Solo este" → solo esa cambia. Editar otra con "Este y los siguientes" → split correcto. "Toda la serie" → propaga a futuras, respeta la excepción.
3. **F3**: crear evento a 20 min → recibir recordatorio in-app + correo ~15 min antes (o ajustar offset para probar rápido). Confirmar que no se re-dispara.
4. **F4**: en el form, elegir 2 invitados → "Buscar hora libre" → aparecen 3 chips de huecos comunes; clic rellena inicio/fin.
5. **RLS**: como vendedora pura, la timeline/equipo enmascara PERSONAL ajeno como "Ocupado"; no ve detalle privado.

---

> **Siguiente paso:** The Builder construye F1→F2→F3→F4 desde este blueprint sobre `feature/agenda-google-feel`. Construye ENCIMA de lo existente (cero reescritura de la agenda madura). Cierre por bloque: tests verdes → al final, merge a `master` + `deploy_prod.ps1` (migrate + habilitar tareas Beat).
