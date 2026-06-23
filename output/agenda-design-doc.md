# Design Doc — `apps/agenda` (EspritOS)

> Doc compacto para que otra sesión de Claude Code construya `apps/agenda`
> con el repo de EspritOS abierto. NO reemplaza `BLUEPRINT.md` ni
> `docs/design-system.md` — los referencia. Léelos primero.
>
> **Fecha:** 03/06/2026 · **Arquitecto:** The Architect · **Origen del brief:**
> `proyectos/CremeriaHM/EspritOS/docs/agenda-calendario-brief.md`
> **Revisión 1 (03/06/2026):** verificación contra repo + Postgres prod, ver §13.

---

## 0. Contexto mínimo

Construye una app nueva `apps/agenda` en EspritOS que permita:

- Programar visitas/llamadas/juntas con detección **atómica** de empalmes.
- Vista calendario personal (mes/semana/día) por vendedora.
- Vista equipo para Beto/gerentes.
- Liga polimórfica a `Lead`, `Cliente`, `Account`, `Prospecto`, `Opportunity`, `Recurso`.
- Cuando un `Evento` se marca COMPLETADO → crear `crm.Actividad` automáticamente.

Lee el brief original (`docs/agenda-calendario-brief.md`) para contexto de negocio.
Lee `BLUEPRINT.md` raíz, `docs/design-system.md` y `apps/core/mixins.py` antes
de tocar código.

---

## 1. Decisiones arquitectónicas (resumen y rationale)

| Decisión | Elegido | Por qué |
|---|---|---|
| App separada o extender `crm.Actividad` | **App nueva `agenda`** | Lifecycle distinto (forward-looking vs bitácora), RLS distinta, constraint de empalme no aplica a bitácora |
| Meterlo en `rutero` | **No** | Rutero es PWA offline single-day. Agenda es multi-día/multi-vendedor. Rutero después *lee* de agenda |
| Liga a entidad de negocio | **`GenericForeignKey`** | ~200 eventos/sem no estresa índices; 5 FK nullable ensucia schema |
| Detección de empalmes | **`ExclusionConstraint` nativo Django + service Python** | Constraint = red atómica; service = feedback rico en UI |
| Recurrencia (RRULE) | **Fase 2**, pero `serie_id` UUID nullable desde día 1 | Evita migración dolorosa de históricos al añadir recurrencia |
| Multi-tenant | **`tenant` FK nullable desde día 1** | Match V5 S8; agregar columna después a millones de filas es caro |
| Privacidad `PERSONAL` | **Banda gris "Ocupado", sin payload** | Implementado en service/serializer, no template |
| Notificaciones | **`apps/notifications`** (signal-driven) | Ya existe; no meter Twilio directo |
| Sync Google/Outlook | **Fase 2** | `CalendarIntegration` ya existe, falta caller — pero Fase 1 ya es 5-8 sesiones |
| UI library | **FullCalendar v6 MIT (vendoreada)** | Sin CDN, sin licencia premium. Ver §6 para limitación vista equipo |
| RLS discriminador | **Sucursal (FK en `Evento`)** | `UserProfile` ya tiene `sucursales_asignadas` + `sucursal_default`. Territorio NO está conectado a Profile en el repo (verificado §13). Sucursal es la realidad operativa de HM |
| Clasificación de la app | **Servicio horizontal** | Importable por otras apps vía `eventos_por_entidad(obj)`. Test de aislamiento se extiende |

---

## 2. ⚠️ Aislamiento de apps — patrón obligatorio

EspritOS tiene test (`apps/core/tests/test_app_isolation.py`) que falla si una
app importa de otra que no sea `core`. Agenda quiere ligarse a 6+ modelos de
otras apps. **Esto se resuelve con strings, ContentType y signals invertidos:**

### Reglas

1. **FK siempre como string:** `FK("crm.Territory", ...)`, `FK("tenants.Tenant", ...)`. Nunca `from apps.crm.models import Territory`.
2. **GFK accede a entidad por ContentType:** nunca importar `Lead`, `Cliente`, etc. en `apps/agenda/*.py`.
3. **Signal "Evento.COMPLETADO → crm.Actividad" vive en `apps/crm/signals.py`,** NO en `apps/agenda`. Crm escucha `post_save` de `agenda.Evento`. Está permitido porque crm depende de agenda, no al revés (agenda permanece "leaf" hacia crm).
4. **Services que necesitan resolver entidad** reciben `entidad_tipo_str + entidad_id`:
   ```python
   def crear_evento(*, owner, inicio, fin, entidad_label=None, entidad_id=None, ...):
       ct = ContentType.objects.get_by_natural_key(*entidad_label.split(".")) if entidad_label else None
       return Evento.objects.create(..., entidad_tipo=ct, entidad_id=entidad_id)
   ```
5. **Queries "todos los eventos del Cliente X" se hacen DESDE el service de `clientes` o de `core`**, no desde agenda:
   ```python
   # apps/clientes/services.py
   def eventos_del_cliente(cliente):
       from django.contrib.contenttypes.models import ContentType
       from apps.agenda.models import Evento  # legal: clientes puede importar de agenda
       ct = ContentType.objects.get_for_model(cliente)
       return Evento.objects.filter(entidad_tipo=ct, entidad_id=cliente.pk)
   ```
   Espera — esto también rompe aislamiento (clientes importa de agenda). **Solución correcta:** este tipo de query vive en `apps/core/services/cross_app_queries.py` o se expone como método de manager de agenda accesible vía signal/API. **Decisión:** método público en `Evento.objects.por_entidad(obj)` que recibe el objeto y resuelve ContentType internamente — el caller hace `from apps.agenda.models import Evento`, y el test de aislamiento se actualiza para permitir importar `agenda` desde apps que la consuman (es una app de servicio, como `notifications`).

   > **DECISIÓN (resuelta 03/06/2026):** agenda se clasifica como **servicio horizontal** (igual
   > que `notifications`, `auditoria`). Justificación: las fichas (cliente, lead, account, recurso)
   > consumen `eventos_por_entidad(obj)` desde sus templates/views. Forzar este flujo via
   > `apps/core/services/cross_app_queries.py` es indirección sin ganancia — `notifications` ya
   > tiene el mismo patrón. El test de aislamiento se actualiza para añadir `agenda` a la lista
   > de servicios horizontales importables.

### Tests de aislamiento

Antes del primer commit que añada modelos, **actualiza** `apps/core/tests/test_app_isolation.py` para cubrir `apps/agenda/`. Si declaras agenda como servicio horizontal, añádela a la lista correspondiente.

---

## 3. Modelo de datos final

```python
# apps/agenda/models.py
import uuid
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.conf import settings
from django.contrib.postgres.constraints import ExclusionConstraint
from django.contrib.postgres.fields import RangeOperators
from django.db import models
from django.db.models import F, Func, Q
from django.core.exceptions import ValidationError

from apps.core.models import AuditedModel  # Verificado: AuditedModel vive en core.models, NO en core.mixins


class TipoEvento(models.TextChoices):
    VISITA_CLIENTE = "VISITA_CLIENTE", "Visita a cliente"
    LLAMADA        = "LLAMADA", "Llamada"
    DEMO           = "DEMO", "Demo / presentación"
    JUNTA_INTERNA  = "JUNTA_INTERNA", "Junta interna"
    CAPACITACION   = "CAPACITACION", "Capacitación"
    BLOQUEO        = "BLOQUEO", "Bloqueo de agenda"
    PERSONAL       = "PERSONAL", "Personal (privado)"


class EstadoEvento(models.TextChoices):
    AGENDADO   = "AGENDADO", "Agendado"
    EN_CURSO   = "EN_CURSO", "En curso"
    COMPLETADO = "COMPLETADO", "Completado"
    CANCELADO  = "CANCELADO", "Cancelado"
    NO_SHOW    = "NO_SHOW", "No-show"


class SyncStatus(models.TextChoices):
    PENDING = "PENDING"
    SYNCED  = "SYNCED"
    FAILED  = "FAILED"
    SKIP    = "SKIP"


class Evento(AuditedModel):
    # Identificadores y tenant
    tenant     = models.ForeignKey("tenants.Tenant", null=True, blank=True,
                                   on_delete=models.PROTECT, db_index=True)
    serie_id   = models.UUIDField(null=True, blank=True, db_index=True,
                                  help_text="RRULE futuro. Eventos de la misma serie comparten UUID.")

    # Core
    tipo       = models.CharField(max_length=20, choices=TipoEvento.choices)
    titulo     = models.CharField(max_length=200)
    descripcion = models.TextField(blank=True)
    inicio     = models.DateTimeField(db_index=True)
    fin        = models.DateTimeField(db_index=True)
    todo_dia   = models.BooleanField(default=False)
    estado     = models.CharField(max_length=15, choices=EstadoEvento.choices,
                                  default=EstadoEvento.AGENDADO, db_index=True)
    lugar      = models.CharField(max_length=255, blank=True)

    # Ownership y sucursal (string refs, sin import)
    owner      = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT,
                                   related_name="eventos_owned", db_index=True)
    sucursal   = models.ForeignKey("core.Sucursal", null=True, blank=True,
                                   on_delete=models.SET_NULL, db_index=True,
                                   help_text="Sucursal donde se ejecuta el evento. "
                                             "Default: sucursal_default del owner. "
                                             "Usado para RLS de gerentes y filtrado en vista equipo.")
    territory  = models.ForeignKey("crm.Territory", null=True, blank=True,
                                   on_delete=models.SET_NULL,
                                   help_text="Opcional. Solo si el owner es dueño de un Territory "
                                             "(Territory.owner == self). Resuelto en save().")

    # Liga polimórfica
    entidad_tipo = models.ForeignKey(ContentType, null=True, blank=True,
                                     on_delete=models.SET_NULL)
    entidad_id   = models.PositiveIntegerField(null=True, blank=True)
    entidad      = GenericForeignKey("entidad_tipo", "entidad_id")

    # Sync externo (campos listos aunque caller sea Fase 2)
    google_event_id  = models.CharField(max_length=255, blank=True, db_index=True)
    outlook_event_id = models.CharField(max_length=255, blank=True, db_index=True)
    sync_status      = models.CharField(max_length=10, choices=SyncStatus.choices,
                                        default=SyncStatus.PENDING)

    class Meta:
        db_table = "agenda_eventos"
        indexes = [
            models.Index(fields=["owner", "inicio"]),
            models.Index(fields=["entidad_tipo", "entidad_id"]),
            models.Index(fields=["inicio", "fin"]),
            models.Index(fields=["tenant", "estado"]),
            models.Index(fields=["sucursal", "inicio"]),
        ]
        constraints = [
            # Django 5.1 NO exporta TstzRange. Se usa Func con función SQL TSTZRANGE.
            # Verificado contra django 5.1.15 corriendo en espritos-web.
            ExclusionConstraint(
                name="agenda_evento_no_overlap_per_owner",
                expressions=[
                    ("owner", RangeOperators.EQUAL),
                    (
                        Func(
                            F("inicio"), F("fin"),
                            function="TSTZRANGE",
                            output_field=models.Field(),
                        ),
                        RangeOperators.OVERLAPS,
                    ),
                ],
                condition=Q(estado__in=["AGENDADO", "EN_CURSO"]) & ~Q(tipo="PERSONAL"),
            ),
            models.CheckConstraint(
                check=Q(fin__gt=F("inicio")),
                name="agenda_evento_fin_gt_inicio",
            ),
        ]

    def __str__(self):
        return f"{self.titulo} ({self.inicio:%Y-%m-%d %H:%M})"

    def clean(self):
        # Validación amistosa antes del IntegrityError del constraint.
        # Inline para evitar import circular models <-> services.
        if self.inicio and self.fin and self.fin <= self.inicio:
            raise ValidationError({"fin": "El fin debe ser posterior al inicio."})
        if not self.owner_id or not self.inicio or not self.fin:
            return
        if self.tipo == TipoEvento.PERSONAL:
            return  # PERSONAL no compite por slot
        conflictos = Evento.objects.filter(
            owner_id=self.owner_id,
            estado__in=[EstadoEvento.AGENDADO, EstadoEvento.EN_CURSO],
            inicio__lt=self.fin,
            fin__gt=self.inicio,
        ).exclude(tipo=TipoEvento.PERSONAL)
        if self.pk:
            conflictos = conflictos.exclude(pk=self.pk)
        if conflictos.exists():
            raise ValidationError({
                "inicio": f"Empalma con: {', '.join(str(c) for c in conflictos[:3])}"
            })

    def save(self, *args, **kwargs):
        # Default sucursal desde el profile del owner (si está vacía)
        if not self.sucursal_id and self.owner_id:
            profile = getattr(self.owner, "profile", None)
            if profile and profile.sucursal_default_id:
                self.sucursal_id = profile.sucursal_default_id
        # Default territory SOLO si owner es dueño de un único Territory
        if not self.territory_id and self.owner_id:
            from django.apps import apps as django_apps
            Territory = django_apps.get_model("crm", "Territory")
            terrs = Territory.objects.filter(owner_id=self.owner_id, is_active=True)
            if terrs.count() == 1:
                self.territory_id = terrs.first().pk
        super().save(*args, **kwargs)


class Asistente(models.Model):
    class Rol(models.TextChoices):
        ORGANIZADOR = "ORGANIZADOR"
        REQUERIDO   = "REQUERIDO"
        OPCIONAL    = "OPCIONAL"

    class Respuesta(models.TextChoices):
        PENDIENTE = "PENDIENTE"
        ACEPTADO  = "ACEPTADO"
        RECHAZADO = "RECHAZADO"

    evento    = models.ForeignKey(Evento, on_delete=models.CASCADE,
                                  related_name="asistentes")
    user      = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    rol       = models.CharField(max_length=15, choices=Rol.choices,
                                 default=Rol.REQUERIDO)
    respuesta = models.CharField(max_length=15, choices=Respuesta.choices,
                                 default=Respuesta.PENDIENTE)

    class Meta:
        db_table = "agenda_asistentes"
        unique_together = [("evento", "user")]


class BloqueoCalendario(models.Model):
    """Vacaciones, día libre, capacitación de día completo (banda gris)."""
    user   = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                               related_name="bloqueos_agenda")
    inicio = models.DateField()
    fin    = models.DateField()
    motivo = models.CharField(max_length=200)

    class Meta:
        db_table = "agenda_bloqueos"
        indexes = [models.Index(fields=["user", "inicio"])]
        constraints = [
            models.CheckConstraint(
                check=Q(fin__gte=models.F("inicio")),
                name="agenda_bloqueo_fin_gte_inicio",
            ),
        ]


# Recordatorio se difiere a Fase 2 (cuando se cablee notifications)
```

### Notas sobre el modelo

- **`sucursal` es el discriminador RLS principal en Fase 1.** La realidad operativa de
  HM es por sucursal (Cremería/Abarrotera), no por territorio. `UserProfile.sucursales_asignadas`
  y `UserProfile.sucursal_default` YA existen (`apps/core/models.py:368-403`); territorio NO
  está conectado a Profile. Default en `save()`: si `sucursal` viene vacía, se llena con
  `owner.profile.sucursal_default` (que puede ser null para directivos con `puede_ver_ambas`).
- **`territory` se mantiene como FK opcional** para futuro. Se llena en `save()` SOLO si
  `Territory.objects.filter(owner=self.owner).count() == 1` (vendedora dueña de un único
  territorio). En cualquier otro caso queda null. Reescribir cuando RH conecte territorios a
  vendedoras formalmente.
- **`clean()` lanza ValidationError amistoso**; el `ExclusionConstraint` queda como red de seguridad para race conditions y bulk operations.
- **`tenant` FK nullable** — match V5 S8. Cuando V6 enforce tenants, migración trivial.
- **`AuditedModel` se importa de `apps.core.models`** (no de `core.mixins` — verificado contra el repo, los abstracts viven en `models.py`).

---

## 4. Migración con extensión Postgres

**✅ Verificado en prod (03/06/2026):**
- `btree_gist` NO está instalado (`pg_extension` reporta: pg_trgm, pgcrypto, plpgsql, unaccent, vector).
- Usuario `espritos` ES `rolsuper=t` → `CREATE EXTENSION` correrá sin intervención manual.

```python
# apps/agenda/migrations/0001_initial.py
from django.db import migrations

class Migration(migrations.Migration):
    initial = True
    dependencies = [
        ("core", "0XXX_latest"),       # AuditedModel + Sucursal + UserProfile
        ("crm", "0XXX_latest"),        # Territory
        ("tenants", "0XXX_latest"),
        ("contenttypes", "0002_remove_content_type_name"),
    ]
    operations = [
        migrations.RunSQL(
            sql="CREATE EXTENSION IF NOT EXISTS btree_gist;",
            reverse_sql="-- btree_gist no se desinstala automáticamente",
        ),
    ]
```

Migración 0002 con los modelos (auto-generada por `makemigrations` con el modelo de §3).

**Si en el futuro un entorno NO tiene espritos como superuser** (caso multi-tenant aislado),
ejecutar manualmente antes de migrar:
```sql
\c espritos
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

---

## 5. Services (lógica de negocio)

```python
# apps/agenda/services.py
from typing import Optional
from django.conf import settings
from django.contrib.contenttypes.models import ContentType
from django.db.models import Q, QuerySet

from apps.agenda.models import Evento, EstadoEvento, TipoEvento


def detectar_empalmes(
    owner,
    inicio,
    fin,
    excluir_id: Optional[int] = None,
) -> QuerySet[Evento]:
    qs = Evento.objects.filter(
        owner=owner,
        estado__in=[EstadoEvento.AGENDADO, EstadoEvento.EN_CURSO],
        inicio__lt=fin,
        fin__gt=inicio,
    ).exclude(tipo=TipoEvento.PERSONAL)
    if excluir_id:
        qs = qs.exclude(pk=excluir_id)
    return qs


def crear_evento(
    *,
    owner,
    tipo: str,
    titulo: str,
    inicio,
    fin,
    creado_por,
    entidad_label: Optional[str] = None,  # "crm.Lead", "clientes.Cliente", ...
    entidad_id: Optional[int] = None,
    descripcion: str = "",
    lugar: str = "",
    tenant=None,
    sucursal=None,
) -> Evento:
    """Punto de entrada único. Levanta ValidationError si empalma."""
    ct = None
    if entidad_label:
        app_label, model = entidad_label.split(".")
        ct = ContentType.objects.get_by_natural_key(app_label, model.lower())

    evento = Evento(
        owner=owner, tipo=tipo, titulo=titulo, descripcion=descripcion,
        inicio=inicio, fin=fin, lugar=lugar, tenant=tenant, sucursal=sucursal,
        entidad_tipo=ct, entidad_id=entidad_id,
        created_by=creado_por, updated_by=creado_por,
    )
    evento.full_clean()  # corre clean() → detecta empalmes amistoso
    evento.save()        # save() resuelve sucursal/territory defaults
    return evento


def reagendar(evento: Evento, *, nuevo_inicio, nuevo_fin, por) -> Evento:
    evento.inicio = nuevo_inicio
    evento.fin = nuevo_fin
    evento.updated_by = por
    evento.full_clean()
    evento.save(update_fields=["inicio", "fin", "updated_by", "updated_at"])
    return evento


def marcar_completado(evento: Evento, *, por, resultado: str = "") -> Evento:
    evento.estado = EstadoEvento.COMPLETADO
    evento.updated_by = por
    evento.save(update_fields=["estado", "updated_by", "updated_at"])
    # signal post_save en apps/crm/signals.py crea la Actividad
    return evento


# ---------------------------------------------------------------------------
# RLS — Row Level Security
# ---------------------------------------------------------------------------
#
# REALIDAD verificada del repo (03/06/2026):
#   - NO existe `user.profile.territory` ni `user.profile.territorios_supervisados`.
#   - UserProfile (apps/core/models.py:368) tiene: `sucursales_asignadas` (M2M
#     a Sucursal), `sucursal_default`, `puede_ver_ambas`.
#   - Territory (apps/crm/models.py:151) tiene `owner` FK pero NO M2M de
#     supervisión. Quien es "gerente" de un Territory no está modelado.
#
# DECISIÓN Fase 1: RLS por sucursal (que SÍ existe) + flags simples.
# Cuando RH formalice jerarquía de supervisión, se extiende sin migración de datos.


def eventos_visibles_para(user, desde, hasta) -> QuerySet[Evento]:
    """RLS principal.
    - Superuser / grupo "admin": todo.
    - Grupo "gerente" o profile.puede_ver_ambas=True: eventos cuyo
      `sucursal` esté en `user.profile.sucursales_asignadas` (o todos si
      puede_ver_ambas y no hay restricción).
    - Vendedora / default: `owner=self` ∪ donde es asistente.
    """
    base = (
        Evento.objects
        .filter(inicio__lt=hasta, fin__gt=desde)
        .select_related("owner", "sucursal", "territory", "tenant",
                        "entidad_tipo")
    )
    if user.is_superuser or user.groups.filter(name="admin").exists():
        return base

    profile = getattr(user, "profile", None)
    es_gerente = user.groups.filter(name="gerente").exists() or (
        profile is not None and profile.puede_ver_ambas
    )
    if es_gerente and profile is not None:
        suc_ids = list(
            profile.sucursales_asignadas.values_list("id", flat=True)
        )
        if profile.puede_ver_ambas and not suc_ids:
            # Directivo sin restricción de sucursal → ve todo
            return base
        return base.filter(
            Q(owner=user)
            | Q(sucursal_id__in=suc_ids)
            | Q(asistentes__user=user)
        ).distinct()

    return base.filter(
        Q(owner=user) | Q(asistentes__user=user)
    ).distinct()


def eventos_por_entidad(obj) -> QuerySet[Evento]:
    """Helper para consumir desde fichas (cliente, lead, account, recurso).

    Se llama así desde otras apps SIN romper aislamiento:
        from apps.agenda.services import eventos_por_entidad
        eventos = eventos_por_entidad(cliente)

    Agenda queda clasificada como **servicio horizontal** (ver §2 decisión).
    """
    ct = ContentType.objects.get_for_model(obj.__class__)
    return Evento.objects.filter(entidad_tipo=ct, entidad_id=obj.pk)
```

---

## 6. UI / Frontend

### Stack

- **FullCalendar v6** vendoreado en `apps/agenda/static/agenda/fullcalendar/`. NO CDN.
- Vistas plugins necesarias: `dayGrid`, `timeGrid`, `interaction` (drag). Todas MIT.
- **Vista equipo:** `resourceTimeline` es **premium (no MIT)**. Alternativa MIT: usar `timeGridWeek` con filtro por owner y código de color por vendedora, o renderizar grid HTML propio con CSS Grid (una fila por owner × columnas por día). **Decisión:** vista equipo en Fase 1.5 con grid HTML propio, NO usar FullCalendar para esa vista.

### Patrones HTMX

- Click slot vacío → `hx-get="/agenda/evento/nuevo?inicio={iso}&fin={iso}"` → modal Alpine.
- Submit form → `hx-post="/agenda/evento/"`:
  - 201 → cierra modal, dispara `hx-trigger="agendaRecargar from:body"` que refresca calendar.
  - 422 con `conflicto_warning.html` → reemplaza form, vendedora ve qué empalma.
- Drag-to-reschedule → `hx-post="/agenda/evento/{id}/reagendar/"`:
  - **Crítico:** en `eventDrop` callback de FullCalendar, si response no es 2xx → `info.revert()`. Sin esto la UI miente.

### Reutiliza

- `docs/design-system.md` — paleta HM, componentes DaisyUI, patrones modal.
- `apps/core/templates/core/base.html` — layout maestro.
- No introducir librería UI nueva.

---

## 7. Calendario operativo HM — vive en `core`, no en agenda

Crear/extender `apps/core/services/calendario_hm.py`:

```python
from datetime import date, datetime, time, timedelta

# Solo 4 días de cierre/año
DIAS_CIERRE = {
    # 1 enero, Viernes Santo, Sábado Santo, 25 diciembre
    # Viernes/Sábado Santo se calculan vía dateutil.easter — no hardcode
}

HORARIO_APERTURA = time(7, 0)
HORARIO_CIERRE   = time(16, 0)
# L-S abierto, domingo cerrado


def es_dia_habil(d: date) -> bool: ...
def siguiente_dia_habil(d: date) -> date: ...
def en_horario_operacion(dt: datetime) -> bool: ...
def slots_del_dia(d: date, duracion_min: int = 30) -> list[datetime]: ...
```

Agenda lo consume vía `from apps.core.services.calendario_hm import ...`.
Esto está permitido por el aislamiento (`core` es libre import).

**Cobranza, rutero y otras apps lo reutilizarán** — por eso vive en `core`,
no en agenda.

---

## 8. Build order (ejecutar en este orden)

> Cada paso debe terminar con tests en verde y commit atómico.

1. **`apps/core/services/calendario_hm.py` + tests.** Standalone. Devuelve `es_dia_habil`, `slots_del_dia`. Test: 1 enero 2027 NO hábil; 16 septiembre 2026 SÍ hábil; viernes santo 2027 NO hábil.
2. **`apps/agenda/` scaffold:** `apps.py`, `__init__.py`, `urls.py` vacío, registrar en `INSTALLED_APPS` y en `URL_TO_MODULO` de `apps/core/middleware.py`.
3. **Actualizar `apps/core/tests/test_app_isolation.py`** para cubrir `agenda` (decidir si es leaf o servicio horizontal — recomiendo leaf inicialmente, promover si surge necesidad).
4. **Migración 0001 con `CREATE EXTENSION btree_gist`** (RunSQL solo, vacía de modelos).
5. **Modelos `Evento`, `Asistente`, `BloqueoCalendario`** + migración 0002. Constraint de exclusión incluido. Test: empalme insertado por bulk_create levanta IntegrityError; empalme PERSONAL pasa; empalme CANCELADO pasa.
6. **`services.py`:** `detectar_empalmes`, `crear_evento`, `reagendar`, `marcar_completado`, `eventos_visibles_para`. Test cada uno aislado.
7. **`forms.py`** con `EventoForm` (ModelForm) y `validate()` que llama `detectar_empalmes`.
8. **Views CRUD HTMX-first:** `calendario_mes`, `calendario_semana`, `evento_form` (modal), `evento_create`, `evento_update`, `evento_detalle`, `evento_reagendar`, `evento_marcar_completado`. Aplicar `RoleFilteredQuerysetMixin`. Test RLS por vista.
9. **Templates:** `calendario_mes.html`, `calendario_semana.html`, `partials/evento_form.html`, `partials/conflicto_warning.html`. FullCalendar vendoreado.
10. **JS de FullCalendar:** init, eventSources apuntando a `/agenda/api/eventos/?desde=&hasta=` (endpoint JSON pequeño, único caso de JSON en la app — necesario porque FullCalendar consume JSON). `eventDrop` con revert en error.
11. **Signal `Evento.COMPLETADO → crm.Actividad`** vive en `apps/crm/signals.py`. Test: marcar completado crea Actividad con `tipo=VISITA`.
12. **`admin.py`:** registrar los 3 modelos con `list_display`, `list_filter`, `search_fields`.
13. **Smoke test E2E con `pytest-django` + `playwright`** opcional: crear evento, drag a otro slot, verificar UI.

---

## 9. Tests obligatorios (definition of done)

- `test_models.py`
  - `Evento.clean()` levanta ValidationError en empalme.
  - Constraint Postgres levanta IntegrityError en bulk_create con empalme.
  - PERSONAL no bloquea.
  - CANCELADO no bloquea.
- `test_services.py`
  - `crear_evento` con entidad_label="crm.Lead", entidad_id=X funciona.
  - `crear_evento` sin entidad funciona.
  - `reagendar` a slot ocupado levanta error.
  - `marcar_completado` cambia estado y dispara signal.
- `test_views.py`
  - GET calendario_mes 200 con HTML.
  - POST evento_create 201 cierra modal.
  - POST evento_create con empalme 422 con partial conflicto.
- `test_rls.py`
  - Vendedora A NO ve eventos de vendedora B (misma sucursal).
  - Vendedora A SÍ ve evento donde es asistente.
  - Gerente VE eventos de su(s) sucursal(es) `sucursales_asignadas`.
  - Gerente NO ve eventos de sucursales fuera de las asignadas.
  - Profile `puede_ver_ambas=True` sin restricción de sucursal → ve todo.
  - Admin VE todo.
  - `eventos_por_entidad(cliente)` solo devuelve eventos ligados a ese cliente.
- `test_isolation.py` (extender el existente)
  - `apps/agenda/` no importa de apps que no sean `core`.
- `test_calendario_operativo.py`
  - 4 días de cierre correctos por año (2026, 2027, 2028).
  - 16 septiembre es hábil.
  - Slots del día respetan apertura/cierre.

---

## 10. Fuera de Fase 1 (NO construir ahora)

- Sync bidireccional Google/Outlook (caller de `CalendarIntegration`).
- Recurrencia RRULE (`serie_id` ya está en el modelo para cuando llegue).
- Recordatorios automáticos vía `apps/notifications`.
- Dashboard KPIs (no-show rate por vendedora, ocupación).
- Vista pública estilo Calendly.
- Optimizador de ruteo (combinar visitas cercanas — va con `rutero`).
- Vista equipo con `resourceTimeline` premium (Fase 1.5 con grid HTML propio).

---

## 11. Estimación

- **Fase 1 sin vista equipo:** 5-7 sesiones (~1 sprint corto).
- **Vista equipo (Fase 1.5):** +1-2 sesiones.
- **Riesgo:** instalación de `btree_gist` en prod requiere permisos. Verificar con Beto antes de Step 4.

---

## 12. Handoff al builder

Antes de empezar, lee en este orden:
1. Este doc completo.
2. `proyectos/CremeriaHM/EspritOS/docs/agenda-calendario-brief.md` (brief original — contexto de negocio).
3. `proyectos/CremeriaHM/EspritOS/BLUEPRINT.md` raíz.
4. `proyectos/CremeriaHM/EspritOS/docs/design-system.md`.
5. `apps/core/mixins.py`, `apps/core/middleware.py`, `apps/core/tests/test_app_isolation.py`.
6. `apps/crm/models.py` líneas 99 (Actividad), 151 (Territory), 928 (CalendarIntegration) para entender qué ya existe.

**Preguntas abiertas del arquitecto — RESUELTAS (03/06/2026):**

1. ✅ **Agenda = servicio horizontal.** Importable por apps que consuman vía
   `eventos_por_entidad(obj)`. Ver §2 y §5.
2. ✅ **`btree_gist`: usuario `espritos` ES `rolsuper=t`.** La migración
   `CREATE EXTENSION IF NOT EXISTS btree_gist` corre limpia. Ver §4.

---

## 13. Log de verificación contra repo y prod (03/06/2026)

Antes de aplicar este doc se verificaron 5 supuestos del draft original:

| # | Supuesto del draft original | Verificación | Resultado |
|---|---|---|---|
| 1 | `btree_gist` instalado o user puede crearlo | `docker exec espritos-db psql -U espritos -d espritos -c "SELECT extname FROM pg_extension; SELECT rolsuper FROM pg_roles WHERE rolname='espritos';"` | btree_gist **NO** instalado · `espritos.rolsuper = t` → migración con `CREATE EXTENSION` funciona ✅ |
| 2 | `from django.db.models.functions import TstzRange` (Django 5.x) | `docker exec espritos-web python -c "from django.db.models import functions; print([n for n in dir(functions) if 'tstz' in n.lower()])"` contra django 5.1.15 | `TstzRange` **NO existe**. Reemplazado por `Func(F("inicio"), F("fin"), function="TSTZRANGE")` ✅ |
| 3 | `"auth.User"` como string-ref | `grep AUTH_USER_MODEL config/settings/` | No hay override (usan `auth.User` default). Por consistencia con `crm.CalendarIntegration` y `crm.Territory.owner`, todos los FK al user usan `settings.AUTH_USER_MODEL` ✅ |
| 4 | `from apps.core.mixins import AuditedModel` | `grep "class AuditedModel" apps/core/` | `AuditedModel` vive en `apps/core/models.py:34`, **NO** en `mixins.py`. Import corregido ✅ |
| 5 | `user.profile.territory` y `user.profile.territorios_supervisados` | Leído `apps/core/models.py:368-426` (`UserProfile`) | **NO existen.** Profile tiene `sucursales_asignadas`, `sucursal_default`, `puede_ver_ambas`, `locale`, `timezone`, `currency_display`. Territorio (en `apps/crm/models.py:151`) NO está conectado a Profile. **Cambio mayor:** RLS de Fase 1 usa Sucursal como discriminador; territory queda como cache opcional ✅ |

### Cambios mayores aplicados al doc

1. **§3 modelo:** agregado `sucursal` FK a `core.Sucursal`. `territory` se llena en `save()` solo si el owner es dueño de exactamente UN `Territory.is_active=True`.
2. **§3 `clean()`:** ya no importa `services` (evita import circular). Inline la query de empalmes + validación `fin > inicio`.
3. **§3 `save()`:** resuelve defaults de `sucursal` (desde `profile.sucursal_default`) y `territory`.
4. **§4:** confirma que `espritos.rolsuper=t` → la extensión se crea en la migración sin manual ops.
5. **§5 `eventos_visibles_para`:** reescrito por completo. Discrimina por sucursal + grupos. Maneja `puede_ver_ambas` para directivos.
6. **§5 `eventos_por_entidad(obj)`:** nuevo helper público — es el contrato del servicio horizontal.
7. **§9 tests RLS:** alineados con la nueva semántica por sucursal.
8. **§1 tabla decisiones:** añadidas dos filas (RLS por sucursal, app = servicio horizontal).
9. **§2 decisión leaf vs servicio horizontal:** resuelta a **servicio horizontal**.
10. **§12 preguntas abiertas:** ambas marcadas como ✅ resueltas.

### Inventario de archivos del repo consultados

- `apps/core/models.py:34` — `AuditedModel`
- `apps/core/models.py:368-426` — `UserProfile`
- `apps/crm/models.py:99` — `Actividad`
- `apps/crm/models.py:151-189` — `Territory`
- `apps/crm/models.py:928-963` — `CalendarIntegration`
- `config/settings/` — sin `AUTH_USER_MODEL` override
- Postgres prod `espritos@db`: `pg_extension` + `pg_roles`
- Django runtime: 5.1.15
