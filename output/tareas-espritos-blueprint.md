# Tareas (Task Tracker) — Blueprint para `apps/tareas` dentro de EspritOS

> Generado por The Architect el 28/06/2026
> Arquetipo: Internal Tool / Dashboard — **embebido en el monolito Django 5.1 de EspritOS** (NO greenfield)
> Repo anfitrión: `E:\ClaudeWorks\proyectos\CremeriaHM\EspritOS` · GitHub `huheme25/espritos` · prod https://espritos.app
> Construye: **The Builder**, en rama `feature/tareas`. Cero cambios a apps existentes salvo los puntos de integración de la §10.

---

## 0. Cómo usar este blueprint (para The Builder)

Este documento es **100% autocontenido**. No necesitas leer otra cosa para construir, pero TODO patrón aquí está **calcado de código real que ya pasa en prod** — las rutas `archivo:línea` apuntan al repo EspritOS para que verifiques el original si dudas. Regla de oro: **calca el patrón, no lo reinventes; copia, no importes** (aislamiento de apps, §11.1).

Orden de lectura: §1 (qué es) → §3 (modelo de datos, el corazón) → §4 (servicios/RLS) → §5–7 (vistas/UI/notif) → §9 (matriz permisos) → §10 (puntos de integración) → **§12 (build order, sigue paso por paso)** → §13 (tests) → §14 (reglas) → §15 (CLAUDE.md de la app).

---

## 1. Visión, alcance y métricas

### Visión
EspritOS hoy es el toolkit de la vendedora (CRM, Agenda, Aprendizaje, Rentabilidad) + BI de dirección (Ritmo, Pulso). **Tareas** lo expande a **sistema operativo del equipo entero**: un tracker estilo Notion/AnyType donde se crea trabajo, se asigna a personas reales del equipo y se lleva el avance en tableros kanban configurables. No es funnel de ventas — es el "qué tengo que hacer y quién" de toda la empresa.

### Goals
- Crear **tableros configurables** (el usuario define columnas: nombre, color, orden, terminal) con vista kanban + drag&drop.
- **Asignar tareas a personas reales** del equipo (multi-asignado), con **notificación in-app + correo** al asignar.
- **Track de avance**: historial de movimientos entre columnas + bitácora de comentarios por tarea.
- **RLS estricto**: cada quien ve lo suyo (creado/asignado) + los tableros donde es miembro; gerencia/dirección ve todo.
- Habilitar el módulo **ampliamente** (todo el equipo, no solo ventas), incluyendo un rol nuevo `colaborador` para RH y personas que solo usan el tracker.

### Success metrics
- Un usuario crea tablero → columna → tarea → asigna → el asignado recibe notificación in-app + correo, en < 5 clics y sin recargar página (HTMX).
- Mover una tarjeta entre columnas persiste posición + crea registro de historial, atómico.
- `test_rls.py` verde: persona A nunca ve tareas de B fuera de su alcance.
- Suite completa verde en `scripts/test_docker.ps1` y vistas < 20 queries (sin N+1).

### Anti-alcance (NO se construye — evita el agujero negro de clonar Notion)
- ❌ Bases de datos relacionales arbitrarias por el usuario, relations entre objetos, fórmulas/rollups.
- ❌ Motor de permisos por-tablero con roles/niveles (ACLs granulares). La visibilidad es **membresía binaria** de tablero + creador/asignado + rol gerencial. (Ver nota de diseño en §4.2.)
- ❌ Integración visual tarea↔ficha de Cliente/Lead en el CRM → **diferida a MVP+1**. La GenericFK y el servicio horizontal `tareas_por_entidad` quedan **listos en el modelo**, pero el MVP **no toca `apps/crm`**.
- "Configurable" = crear tableros + definir columnas (nombre, color, orden, es_terminal) + prioridad por tarea. Vistas adicionales (lista, agrupar por prioridad/asignado) son vistas sobre el mismo dato, se agregan incrementalmente post-MVP.

---

## 2. Stack (FIJO — no negociable, es el de EspritOS)

Django 5.1 · Python 3.12 · PostgreSQL 16 · Redis 7 · Celery 5 · **HTMX 2 + Alpine.js 3 + Tailwind v4 (compilado, NO Play CDN) + DaisyUI** · **SortableJS 1.15.2** (drag&drop, ya en el repo). Server-rendered first: HTMX intercambia **HTML**, no JSON. Sin SPA. Clases CSS: convención `hm-*` del repo (`hm-input`, `hm-textarea`, `hm-select`, `hm-badge`), no DaisyUI crudo.

---

## 3. Modelo de datos (el corazón del diseño)

Calca la mecánica del kanban del CRM (`Pipeline/Stage/Opportunity/StageHistory` — `apps/crm/models.py:498–704`) pero con tableros **por usuario/equipo**, no globales, y **persistiendo el orden intra-columna** (mejora sobre el CRM, que ordena por `-created_at` y pierde posición).

### 3.1 Diagrama de entidades

```
Tablero (dueño, miembros M2M, sucursal)
  └── Columna (1..N, ordenadas, es_terminal)
        └── Tarea (columna actual, orden intra-columna, asignados M2M,
                   prioridad P1-P4, vencimiento, GenericFK opcional)
              ├── MovimientoTarea (audit: columna_origen→destino, días)
              └── ComentarioTarea (bitácora cualitativa)
```

### 3.2 `apps/tareas/models.py` (completo — calcable)

```python
"""
EspritOS — App Tareas. Tracker kanban de tableros configurables por equipo.

Calca apps/crm/models.py (Pipeline/Stage/Opportunity/StageHistory) y el patrón
GenericForeignKey de apps/agenda/models.py:140-145. La diferencia clave: los
tableros son por usuario/equipo (Tablero.dueno + miembros M2M), no globales, y
el orden intra-columna se PERSISTE (Tarea.orden) — el CRM no lo hace.
"""
from __future__ import annotations

from django.conf import settings
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models


class Prioridad(models.TextChoices):
    P1 = "P1", "P1 — Crítica"
    P2 = "P2", "P2 — Alta"
    P3 = "P3", "P3 — Media"
    P4 = "P4", "P4 — Baja"


class Tablero(models.Model):
    """Tablero kanban configurable. Calca Pipeline pero por usuario/equipo."""

    tenant = models.ForeignKey(
        "tenants.Tenant", on_delete=models.CASCADE,
        null=True, blank=True, related_name="tableros",
    )
    nombre = models.CharField(max_length=120)
    descripcion = models.TextField(blank=True)
    dueno = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT,
        related_name="tableros_propios",
    )
    # 28/06/2026 (decisión Beto): membresía para "ver todo el tablero de su
    # área". Un miembro ve TODAS las tareas del tablero (RLS §4.2). Es
    # membresía binaria — NO un motor de permisos por-tablero (anti-alcance).
    miembros = models.ManyToManyField(
        settings.AUTH_USER_MODEL, blank=True,
        related_name="tableros_miembro",
    )
    # RLS de respaldo + filtro admin_sucursal (RoleFilteredQuerysetMixin).
    sucursal = models.ForeignKey(
        "core.Sucursal", on_delete=models.PROTECT,
        null=True, blank=True, related_name="+",
    )
    orden = models.IntegerField(default=0)
    archivado = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "tareas_tableros"
        verbose_name = "Tablero"
        verbose_name_plural = "Tableros"
        ordering = ["orden", "nombre"]
        indexes = [
            models.Index(fields=["dueno", "archivado"]),
            models.Index(fields=["sucursal", "archivado"]),
        ]

    def __str__(self) -> str:
        return self.nombre


class Columna(models.Model):
    """Columna configurable de un tablero. Calca Stage."""

    tablero = models.ForeignKey(
        Tablero, on_delete=models.CASCADE, related_name="columnas",
    )
    nombre = models.CharField(max_length=80)
    color = models.CharField(max_length=7, default="#64748b")  # hex, como Stage.color
    orden = models.IntegerField()
    # Terminal = "Done"/"Abandoned": una tarea aquí cuenta como completada.
    es_terminal = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "tareas_columnas"
        verbose_name = "Columna"
        verbose_name_plural = "Columnas"
        ordering = ["tablero", "orden"]
        constraints = [
            models.UniqueConstraint(
                fields=["tablero", "orden"],
                name="uq_columna_tablero_orden",
            ),
        ]

    def __str__(self) -> str:
        return f"{self.tablero.nombre}/{self.nombre}"


class Tarea(models.Model):
    """Tarea. Calca Opportunity + GenericFK opcional de agenda."""

    tenant = models.ForeignKey(
        "tenants.Tenant", on_delete=models.CASCADE,
        null=True, blank=True, related_name="tareas",
    )
    # tablero denormalizado (además de columna.tablero) para query directa y
    # para que el RLS filtre por tablero sin un JOIN extra.
    tablero = models.ForeignKey(
        Tablero, on_delete=models.CASCADE, related_name="tareas",
    )
    columna = models.ForeignKey(
        Columna, on_delete=models.PROTECT, related_name="tareas",
    )
    titulo = models.CharField(max_length=200)
    descripcion = models.TextField(blank=True)
    asignados = models.ManyToManyField(
        settings.AUTH_USER_MODEL, blank=True,
        related_name="tareas_asignadas",
    )
    creado_por = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT,
        related_name="tareas_creadas",
    )
    prioridad = models.CharField(
        max_length=2, choices=Prioridad.choices, default=Prioridad.P3,
        db_index=True,
    )
    vencimiento = models.DateField(null=True, blank=True)
    # Orden DENTRO de la columna (persistido en el drop). Mejora sobre el CRM.
    orden = models.IntegerField(default=0)

    # GenericFK OPCIONAL a Cliente/Lead/Account (patrón agenda models.py:140-145).
    # En MVP solo se llena; la lectura desde la ficha del CRM es MVP+1.
    entidad_tipo = models.ForeignKey(
        ContentType, null=True, blank=True,
        on_delete=models.SET_NULL, related_name="+",
    )
    entidad_id = models.PositiveIntegerField(null=True, blank=True)
    entidad = GenericForeignKey("entidad_tipo", "entidad_id")

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "tareas_tareas"
        verbose_name = "Tarea"
        verbose_name_plural = "Tareas"
        ordering = ["columna", "orden", "-created_at"]
        indexes = [
            models.Index(fields=["tablero", "columna"]),
            models.Index(fields=["creado_por"]),
            models.Index(fields=["entidad_tipo", "entidad_id"]),
            models.Index(fields=["vencimiento"]),
        ]

    def __str__(self) -> str:
        return self.titulo

    @property
    def completada(self) -> bool:
        """Derivado: una tarea está completa si su columna es terminal."""
        return self.columna.es_terminal


class MovimientoTarea(models.Model):
    """Audit de cada movimiento entre columnas. Calca StageHistory."""

    tarea = models.ForeignKey(
        Tarea, on_delete=models.CASCADE, related_name="movimientos",
    )
    columna_origen = models.ForeignKey(
        Columna, on_delete=models.PROTECT, null=True, related_name="+",
    )
    columna_destino = models.ForeignKey(
        Columna, on_delete=models.PROTECT, related_name="+",
    )
    movido_por = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="+",
    )
    movido_en = models.DateTimeField(auto_now_add=True)
    dias_en_columna_previa = models.IntegerField(null=True, blank=True)

    class Meta:
        db_table = "tareas_movimientos"
        verbose_name = "Movimiento de tarea"
        verbose_name_plural = "Movimientos de tareas"
        ordering = ["-movido_en"]
        indexes = [
            models.Index(fields=["tarea", "movido_en"]),
        ]


class ComentarioTarea(models.Model):
    """Bitácora cualitativa de avance por tarea."""

    tarea = models.ForeignKey(
        Tarea, on_delete=models.CASCADE, related_name="comentarios",
    )
    autor = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="+",
    )
    texto = models.TextField()
    creado_en = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "tareas_comentarios"
        verbose_name = "Comentario de tarea"
        verbose_name_plural = "Comentarios de tareas"
        ordering = ["creado_en"]
        indexes = [
            models.Index(fields=["tarea", "creado_en"]),
        ]
```

### 3.3 Notas de modelado
- **`Tarea.tablero` denormalizado**: además de `columna.tablero`, para que el RLS filtre por tablero sin JOIN y para `prefetch`. Mantener consistente: al mover de columna NO cambia el tablero (las columnas viven en un tablero).
- **`completada`** es propiedad derivada (no campo) — refleja `columna.es_terminal`. Si en el futuro se necesita filtrar en DB por completada, añadir un campo materializado; para el MVP la propiedad basta.
- **`orden`**: int por columna. El servicio `mover_tarea` lo recalcula en el drop (ver §4.3).
- **`tenant` null/blank**: igual que todo el repo (multi-tenant inactivo). No tocar.

---

## 4. Servicios y RLS — `apps/tareas/services.py`

> Toda la lógica de negocio vive aquí; las vistas solo orquestan (regla del repo). Calca `apps/crm/services/kanban.py` y `apps/agenda/services.py`.

### 4.1 Roles sin restricción (gerencia ve todo)
Calca `_aplicar_rls_eventos` de agenda (`apps/agenda/services.py:324-358`) y el `unrestricted_roles` del mixin core (`apps/core/mixins.py:38-41`).

### 4.2 Regla de visibilidad (decisión de diseño cerrada con Beto, 28/06/2026)

Una persona ve una tarea si **cualquiera** de estas es verdadera:
1. La creó (`creado_por == user`), o
2. Es asignada (`user in asignados`), o
3. Es **dueño** del tablero (`tablero.dueno == user`), o
4. Es **miembro** del tablero (`user in tablero.miembros`).

Más: los **roles gerenciales** (`admin`, `admin_sucursal`, `supervisor_ventas`, `direccion`) y superuser ven **todo**.

> **Por qué membresía y no ACLs** (anti-alcance): Beto pidió que un jefe de área (Yunuen/RH, Carlos/almacén) vea "todo el tablero de su área". La forma mínima de lograrlo SIN un motor de permisos granular es la membresía binaria de tablero: *perteneces al tablero → ves sus tareas*. No hay niveles ni roles por tablero. Cuando Beto crea el tablero "RH" y pone a Yunuen como dueña/miembro, ella coordina todo ese tablero. Carlos (admin_sucursal) ya ve todo por su rol gerencial.

### 4.3 `services.py` (completo — calcable)

```python
"""
EspritOS — Servicios de Tareas. Lógica de negocio + RLS.

Calca apps/crm/services/kanban.py (move atómico + board dict) y el RLS de
apps/agenda/services.py:324-380 (helper compartido + contrato horizontal).
"""
from __future__ import annotations

from decimal import Decimal  # noqa: F401  (parидridad con crm; quitar si no se usa)

from django.contrib.contenttypes.models import ContentType
from django.db import transaction
from django.db.models import Q, QuerySet
from django.utils import timezone as dj_tz

from apps.tareas.models import (
    Columna, ComentarioTarea, MovimientoTarea, Tablero, Tarea,
)

# Roles que ven TODO (espejo de apps/core/mixins.py:38-41 + direccion).
ROLES_UNRESTRICTED = {
    "admin", "admin_sucursal", "supervisor_ventas", "direccion",
}


class TareaError(Exception):
    """Error en operación de tareas (transición inválida, etc.)."""


# --------------------------------------------------------------------------
# RLS
# --------------------------------------------------------------------------
def _es_unrestricted(user) -> bool:
    if user.is_superuser:
        return True
    return user.groups.filter(name__in=ROLES_UNRESTRICTED).exists()


def tareas_visibles_para(user) -> QuerySet[Tarea]:
    """RLS principal de tareas. Devuelve las tareas que `user` puede ver.

    Regla (28/06/2026): creador ∪ asignado ∪ dueño-de-tablero ∪
    miembro-de-tablero. Roles gerenciales y superuser ven todo.
    """
    base = Tarea.objects.select_related(
        "tablero", "columna", "creado_por", "entidad_tipo",
    ).prefetch_related("asignados")

    if _es_unrestricted(user):
        return base

    return base.filter(
        Q(creado_por=user)
        | Q(asignados=user)
        | Q(tablero__dueno=user)
        | Q(tablero__miembros=user)
    ).distinct()


def tableros_visibles_para(user) -> QuerySet[Tablero]:
    """Tableros que el usuario puede abrir: propios, donde es miembro, o
    (gerencia) todos. No archivados por default."""
    base = Tablero.objects.filter(archivado=False).select_related("dueno")
    if _es_unrestricted(user):
        return base
    return base.filter(
        Q(dueno=user) | Q(miembros=user)
    ).distinct()


def tareas_por_entidad(obj, *, user) -> QuerySet[Tarea]:
    """Contrato horizontal (MVP+1): tareas ligadas a `obj` (Cliente/Lead/...)
    visibles para `user`. Calca apps/agenda/services.py:366-380.

    NO se consume desde el CRM en el MVP (la GenericFK solo se escribe). Queda
    listo para cuando la ficha del cliente quiera mostrar sus tareas.
    """
    ct = ContentType.objects.get_for_model(obj.__class__)
    base = tareas_visibles_para(user)
    return base.filter(entidad_tipo=ct, entidad_id=obj.pk)


# --------------------------------------------------------------------------
# Mutaciones
# --------------------------------------------------------------------------
@transaction.atomic
def mover_tarea(*, tarea: Tarea, nueva_columna: Columna, movido_por,
                nuevo_orden: int | None = None) -> Tarea:
    """Mueve `tarea` a `nueva_columna`, persiste orden y registra
    MovimientoTarea con días en la columna previa. Calca
    move_opportunity_stage (apps/crm/services/kanban.py:27-100).
    """
    if nueva_columna.tablero_id != tarea.tablero_id:
        raise TareaError("La columna destino es de otro tablero.")

    if nueva_columna.pk == tarea.columna_id and nuevo_orden is None:
        return tarea  # idempotente

    columna_origen = tarea.columna

    # Días en la columna previa (desde el último movimiento o la creación).
    ultimo = (
        MovimientoTarea.objects.filter(tarea=tarea)
        .order_by("-movido_en").first()
    )
    ref = ultimo.movido_en if ultimo else tarea.created_at
    dias_prev = max((dj_tz.now() - ref).days, 0)

    # Historial ANTES de mutar (refleja origen→destino), solo si cambió columna.
    if nueva_columna.pk != tarea.columna_id:
        MovimientoTarea.objects.create(
            tarea=tarea,
            columna_origen=columna_origen,
            columna_destino=nueva_columna,
            movido_por=movido_por,
            dias_en_columna_previa=dias_prev,
        )

    tarea.columna = nueva_columna
    if nuevo_orden is not None:
        tarea.orden = nuevo_orden
    else:
        # Al final de la columna destino.
        ultimo_orden = (
            Tarea.objects.filter(columna=nueva_columna)
            .exclude(pk=tarea.pk)
            .order_by("-orden").values_list("orden", flat=True).first()
        )
        tarea.orden = (ultimo_orden or 0) + 1
    tarea.save(update_fields=["columna", "orden", "updated_at"])
    return tarea


def crear_tarea(*, tablero: Tablero, columna: Columna, titulo: str,
                creado_por, descripcion: str = "", prioridad: str = "P3",
                vencimiento=None, asignados=None, entidad=None) -> Tarea:
    """Crea una tarea. Si trae asignados, dispara la señal de notificación
    (in-app + correo) para los asignados NUEVOS. Ver signals §6."""
    tarea = Tarea(
        tablero=tablero, columna=columna, titulo=titulo,
        creado_por=creado_por, descripcion=descripcion,
        prioridad=prioridad, vencimiento=vencimiento,
    )
    if entidad is not None:
        ct = ContentType.objects.get_for_model(entidad.__class__)
        tarea.entidad_tipo = ct
        tarea.entidad_id = entidad.pk
    ultimo = (
        Tarea.objects.filter(columna=columna)
        .order_by("-orden").values_list("orden", flat=True).first()
    )
    tarea.orden = (ultimo or 0) + 1
    tarea.save()
    if asignados:
        asignar(tarea=tarea, user_ids=[u.pk for u in asignados],
                asignado_por=creado_por)
    return tarea


def asignar(*, tarea: Tarea, user_ids: list[int], asignado_por) -> list[int]:
    """Agrega asignados NUEVOS al M2M y devuelve los ids realmente nuevos
    (para que la VIEW dispare la señal `tarea_asignada`). NO re-notifica a los
    que ya estaban. Igual que agenda: .add()/bulk no disparan signals, la
    notificación se orquesta desde la view (§6)."""
    actuales = set(tarea.asignados.values_list("id", flat=True))
    nuevos = [uid for uid in user_ids if uid not in actuales]
    if nuevos:
        tarea.asignados.add(*nuevos)
    return nuevos


# --------------------------------------------------------------------------
# Render del board
# --------------------------------------------------------------------------
def tablero_render(*, tablero: Tablero, user) -> list[dict]:
    """Devuelve [{columna, tareas[], total}] ordenado por columna.orden, con
    las tareas visibles para `user`. Calca kanban_board
    (apps/crm/services/kanban.py:103-144).
    """
    columnas = list(tablero.columnas.all().order_by("orden"))
    tareas_qs = (
        tareas_visibles_para(user)
        .filter(tablero=tablero)
        .order_by("columna", "orden", "-created_at")
    )
    by_col: dict[int, list] = {c.pk: [] for c in columnas}
    for t in tareas_qs:
        by_col.setdefault(t.columna_id, []).append(t)

    board = []
    for c in columnas:
        tareas = by_col.get(c.pk, [])
        board.append({
            "columna": c,
            "tareas": tareas,
            "total": len(tareas),
        })
    return board
```

> **N+1**: `tareas_visibles_para` ya hace `select_related` (tablero, columna, creado_por, entidad_tipo) + `prefetch_related("asignados")`. El board entero se sirve en pocas queries. El test de §13 lo verifica (<20).

---

## 5. Vistas y URLs

### 5.1 `apps/tareas/views.py` (estructura — calca CRM kanban)

Calca `kanban_view` (`apps/crm/views.py:562-603`) y `opportunity_move_stage_view` (`:641-681`).

```python
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST

from apps.tareas import services
from apps.tareas.forms import TareaForm, TableroForm, ColumnaForm
from apps.tareas.models import Columna, Tablero, Tarea


@login_required
def lista(request):
    """Índice: tableros visibles para el usuario."""
    tableros = services.tableros_visibles_para(request.user)
    return render(request, "tareas/lista.html", {"tableros": tableros})


@login_required
def tablero_detalle(request, pk):
    """Board kanban de un tablero. HTMX-aware: si es request.htmx devuelve
    solo el partial del board (calca kanban_view)."""
    tablero = get_object_or_404(
        services.tableros_visibles_para(request.user), pk=pk,
    )
    board = services.tablero_render(tablero=tablero, user=request.user)
    ctx = {"tablero": tablero, "board": board,
           "columnas": list(tablero.columnas.order_by("orden"))}
    if getattr(request, "htmx", False):
        return render(request, "tareas/partials/board.html", ctx)
    return render(request, "tareas/tablero.html", ctx)


@login_required
@require_POST
def tarea_mover(request, pk):
    """POST /tareas/tarea/<pk>/mover/ — mueve a columna_id + orden.
    Devuelve el board re-renderizado (HTMX swap). Calca
    opportunity_move_stage_view."""
    tarea = get_object_or_404(
        services.tareas_visibles_para(request.user)
        .select_related("tablero", "columna"),
        pk=pk,
    )
    nueva_columna = get_object_or_404(
        Columna, pk=request.POST.get("columna_id"), tablero=tarea.tablero,
    )
    nuevo_orden = request.POST.get("orden")
    try:
        services.mover_tarea(
            tarea=tarea, nueva_columna=nueva_columna, movido_por=request.user,
            nuevo_orden=int(nuevo_orden) if nuevo_orden else None,
        )
    except services.TareaError as exc:
        from django.http import JsonResponse
        return JsonResponse({"error": str(exc)}, status=400)
    board = services.tablero_render(tablero=tarea.tablero, user=request.user)
    return render(request, "tareas/partials/board.html", {
        "tablero": tarea.tablero, "board": board,
        "columnas": list(tarea.tablero.columnas.order_by("orden")),
    })


# + CRUD vistas: tablero_crear/editar, columna_crear/editar/eliminar,
#   tarea_crear/editar/detalle, comentario_crear. Todas:
#   - filtran por services.tareas_visibles_para / tableros_visibles_para
#   - en tarea_crear/editar: tras guardar, disparan la señal (ver §6)
#   - render parcial si request.htmx
```

**Disparo de la señal en `tarea_crear`/`tarea_editar`** (calca `_notificar_invitados_nuevos`, `apps/agenda/views.py:392-409`):

```python
@login_required
def tarea_crear(request, tablero_pk):
    tablero = get_object_or_404(services.tableros_visibles_para(request.user), pk=tablero_pk)
    if request.method == "POST":
        form = TareaForm(request.POST, tablero=tablero, creado_por=request.user)
        if form.is_valid():
            tarea = form.save()
            _notificar_asignados_nuevos(form, tarea, request.user)  # ← señal
            if getattr(request, "htmx", False):
                # re-render board o cerrar modal + swap
                ...
            return redirect("tareas:tablero_detalle", pk=tablero.pk)
    else:
        form = TareaForm(tablero=tablero, creado_por=request.user)
    return render(request, "tareas/partials/form.html", {"form": form, "tablero": tablero})


def _notificar_asignados_nuevos(form, tarea, asignado_por):
    """form.nuevos_asignados_ids lo deja TareaForm._sync_asignados."""
    nuevos = getattr(form, "nuevos_asignados_ids", None)
    if not nuevos:
        return
    from apps.tareas.signals import tarea_asignada
    tarea_asignada.send(
        sender=Tarea, tarea=tarea, user_ids=nuevos, asignado_por=asignado_por,
    )
```

### 5.2 `apps/tareas/urls.py`

```python
from django.urls import path
from apps.tareas import views

app_name = "tareas"

urlpatterns = [
    path("", views.lista, name="lista"),
    path("tablero/nuevo/", views.tablero_crear, name="tablero_crear"),
    path("tablero/<int:pk>/", views.tablero_detalle, name="tablero_detalle"),
    path("tablero/<int:pk>/editar/", views.tablero_editar, name="tablero_editar"),
    path("tablero/<int:tablero_pk>/columna/nueva/", views.columna_crear, name="columna_crear"),
    path("columna/<int:pk>/editar/", views.columna_editar, name="columna_editar"),
    path("columna/<int:pk>/eliminar/", views.columna_eliminar, name="columna_eliminar"),
    path("tablero/<int:tablero_pk>/tarea/nueva/", views.tarea_crear, name="tarea_crear"),
    path("tarea/<int:pk>/", views.tarea_detalle, name="tarea_detalle"),
    path("tarea/<int:pk>/editar/", views.tarea_editar, name="tarea_editar"),
    path("tarea/<int:pk>/mover/", views.tarea_mover, name="tarea_mover"),
    path("tarea/<int:pk>/comentario/", views.comentario_crear, name="comentario_crear"),
]
```

### 5.3 `apps/tareas/forms.py` (claves — calca agenda forms.py:17-175)

- Inyecta `creado_por`/`tablero` desde la view (no editables). Widgets `hm-*`.
- `vencimiento`: **`<input type="date">` nativo** dentro de modales `<dialog>` — NO Flatpickr (regla §14.7, `memory/flatpickr_dialog_static.md`).
- `asignados`: `ModelMultipleChoiceField` (queryset = usuarios activos). Tras `save`, `_sync_asignados` deja `self.nuevos_asignados_ids` con los M2M nuevos (calca `_sync_asistentes`, `apps/agenda/forms.py:152-175`). **`bulk_create`/`.add()` NO disparan signals** → la view dispara la señal.

```python
class TareaForm(forms.ModelForm):
    asignados = forms.ModelMultipleChoiceField(
        queryset=get_user_model().objects.filter(is_active=True),
        required=False, widget=forms.CheckboxSelectMultiple,
    )

    class Meta:
        model = Tarea
        fields = ["titulo", "descripcion", "columna", "prioridad",
                  "vencimiento", "asignados"]
        widgets = {
            "titulo": forms.TextInput(attrs={"class": "hm-input"}),
            "descripcion": forms.Textarea(attrs={"class": "hm-textarea", "rows": 3}),
            "columna": forms.Select(attrs={"class": "hm-select"}),
            "prioridad": forms.Select(attrs={"class": "hm-select"}),
            # input nativo, NO Flatpickr (regla §14.7)
            "vencimiento": forms.DateInput(attrs={"type": "date", "class": "hm-input"}),
        }

    def __init__(self, *args, tablero=None, creado_por=None, **kwargs):
        super().__init__(*args, **kwargs)
        self._tablero = tablero
        self._creado_por = creado_por
        self.nuevos_asignados_ids: list[int] = []
        if tablero is not None:
            self.fields["columna"].queryset = tablero.columnas.order_by("orden")

    def save(self, commit=True):
        tarea = super().save(commit=False)
        if self._creado_por and not tarea.creado_por_id:
            tarea.creado_por = self._creado_por
        if self._tablero is not None:
            tarea.tablero = self._tablero
        if commit:
            tarea.save()
            self._sync_asignados(tarea)
        return tarea

    def _sync_asignados(self, tarea):
        if "asignados" not in self.cleaned_data:
            return
        seleccionados = {u.pk for u in self.cleaned_data["asignados"]}
        actuales = set(tarea.asignados.values_list("id", flat=True))
        if actuales - seleccionados:
            tarea.asignados.remove(*(actuales - seleccionados))
        nuevos = seleccionados - actuales
        if nuevos:
            tarea.asignados.add(*nuevos)
            self.nuevos_asignados_ids = list(nuevos)
```

---

## 6. Notificación al asignar (in-app + correo)

Calca **exactamente** el patrón agenda→notifications, incluido el **fallback síncrono si Celery cae**. `tareas` NO importa `notifications` (aislamiento); define la señal y `notifications` la escucha.

### 6.1 `apps/tareas/signals.py`

```python
"""Señal propia de tareas. La escucha apps.notifications (servicio
cross-cutting). tareas NO importa notifications — regla de aislamiento
(test_no_cross_app_imports). Calca apps/agenda/signals.py."""
from __future__ import annotations
import django.dispatch

# kwargs del send(): tarea (Tarea), user_ids (list[int]), asignado_por (User)
tarea_asignada = django.dispatch.Signal()
```

### 6.2 `apps/tareas/apps.py`

```python
from django.apps import AppConfig


class TareasConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.tareas"
    verbose_name = "Tareas / Tracker"

    def ready(self) -> None:
        from apps.tareas import signals  # noqa: F401
```

### 6.3 Receiver en `apps/notifications/signals.py` (AGREGAR, no reescribir)

```python
from apps.tareas.signals import tarea_asignada  # nuevo import

@receiver(tarea_asignada, dispatch_uid="notif_tarea_asignada")
def on_tarea_asignada(sender, tarea, user_ids, asignado_por, **kwargs):
    """Asignados nuevos en una Tarea → notificación in-app + correo."""
    from apps.notifications import services
    services.notificar_asignados_tarea(tarea, user_ids, asignado_por)
```

### 6.4 `notificar_asignados_tarea` en `apps/notifications/services.py` (AGREGAR)

Calca `notificar_invitados_evento` (`apps/notifications/services.py:61-116`): in-app síncrono + correo `.delay()` con **fallback síncrono**. Usa el `notify()` existente (`:26-47`).

```python
def notificar_asignados_tarea(tarea, user_ids, asignado_por) -> None:
    if not user_ids:
        return
    User = get_user_model()
    asignados = list(User.objects.filter(pk__in=user_ids, is_active=True))
    if not asignados:
        return
    quien = asignado_por.get_full_name() or asignado_por.username
    titulo = f"{quien} te asignó una tarea"
    cuerpo = f"{tarea.titulo} · {tarea.tablero.nombre}"
    if tarea.vencimiento:
        cuerpo += f" · vence {tarea.vencimiento:%d/%m/%Y}"
    tenant = getattr(tarea, "tenant", None)
    for u in asignados:
        notify(
            u, title=titulo, body=cuerpo,
            kind=NotificationKind.GENERAL, severity=NotificationSeverity.INFO,
            link=f"/tareas/tarea/{tarea.pk}/", tenant=tenant,
            metadata={"tarea_id": tarea.pk, "tipo": "asignacion_tarea"},
        )
    from apps.notifications.tasks import enviar_email_asignacion_tarea
    for u in asignados:
        if not u.email:
            continue
        try:
            enviar_email_asignacion_tarea.delay(u.pk, tarea.pk, asignado_por.pk)
        except Exception:
            logger.warning("No se pudo encolar email asignación (tarea=%s, user=%s); "
                           "envío sincrónico.", tarea.pk, u.pk, exc_info=True)
            try:
                enviar_email_asignacion_tarea(u.pk, tarea.pk, asignado_por.pk)
            except Exception:
                logger.exception("Falló email asignación (tarea=%s, user=%s).",
                                 tarea.pk, u.pk)
```

### 6.5 Celery task en `apps/notifications/tasks.py` (AGREGAR)

Calca `enviar_email_invitacion_evento` (`apps/notifications/tasks.py:33-103`). Importa `apps.tareas.models` (permitido: notifications es app consumidora cross-cutting). **Remitente = `settings.DEFAULT_FROM_EMAIL`** (NO un correo hardcodeado — el repo lo resuelve de env var; ver §10 nota). Plantillas nuevas: `templates/notifications/email/asignacion_tarea.{html,txt}`.

```python
@shared_task(bind=True, name="apps.notifications.tasks.enviar_email_asignacion_tarea",
             max_retries=3, default_retry_delay=60)
def enviar_email_asignacion_tarea(self, user_id, tarea_id, asignado_por_id):
    from django.conf import settings
    from django.contrib.auth import get_user_model
    from django.core.mail import EmailMultiAlternatives
    from django.template.loader import render_to_string
    from django.utils.html import strip_tags
    from apps.tareas.models import Tarea
    User = get_user_model()
    user = User.objects.filter(pk=user_id).first()
    tarea = Tarea.objects.select_related("tablero").filter(pk=tarea_id).first()
    asignado_por = User.objects.filter(pk=asignado_por_id).first()
    if not (user and user.email and tarea and asignado_por):
        return "skip: faltan datos"
    ctx = {
        "destinatario": user.get_full_name() or user.username,
        "quien": asignado_por.get_full_name() or asignado_por.username,
        "tarea": tarea,
        "url": f"{settings.SITE_BASE_URL}/tareas/tarea/{tarea.pk}/",
    }
    html = render_to_string("notifications/email/asignacion_tarea.html", ctx)
    texto = strip_tags(render_to_string("notifications/email/asignacion_tarea.txt", ctx))
    msg = EmailMultiAlternatives(
        subject=f"Tarea asignada: {tarea.titulo}",
        body=texto, from_email=settings.DEFAULT_FROM_EMAIL, to=[user.email],
    )
    msg.attach_alternative(html, "text/html")
    try:
        msg.send(fail_silently=False)
    except Exception as exc:
        raise self.retry(exc=exc)
    return f"sent asignacion tarea={tarea_id} user={user_id}"
```

> **Solo asignados nuevos**: la cadena nace de `nuevos_asignados_ids` (el form) → la view dispara la señal solo con los ids nuevos. Editar una tarea sin cambiar asignados NO re-notifica.

---

## 7. UI / Templates (kanban + drag&drop)

Calca `templates/crm/kanban.html` + `templates/crm/partials/kanban_board.html`.

### 7.1 Archivos
- `templates/tareas/lista.html` — grid de tableros visibles + botón "Nuevo tablero".
- `templates/tareas/tablero.html` — layout del board + el `<script>` `tareasDnd()` (SortableJS). Carga el partial.
- `templates/tareas/partials/board.html` — board re-renderizable (`id="tareas-board"`). Itera columnas (`data-columna-id`), contenedor `.tareas-cards` por columna, tarjeta `.tarea-card` con `data-tarea-id`. Header con nombre/color/contador; tarjeta con prioridad (badge `hm-badge`), título, asignados (avatares/iniciales), vencimiento.
- `templates/tareas/partials/form.html` — form de tarea en `<dialog>` (modal). `vencimiento` = input date nativo.
- `templates/tareas/partials/detalle.html` — detalle de tarea: descripción, asignados, historial (`movimientos`), bitácora (`comentarios`) + form de comentario `hx-post`.

### 7.2 SortableJS — `tareasDnd()` (calca kanban.html:290-376)

Diferencia clave vs CRM: **persistimos `orden`**. En `onEnd`, además de `columna_id`, mandamos el **índice destino** como `orden`:

```javascript
function tareasDnd() {
  return {
    instances: [],
    init() {
      this.bind();
      document.body.addEventListener("htmx:afterSwap", (e) => {
        if (e.detail.target && e.detail.target.id === "tareas-board") this.bind();
      });
    },
    bind() {
      this.instances.forEach(s => s.destroy());
      this.instances = [];
      document.querySelectorAll(".tareas-cards").forEach(col => {
        this.instances.push(Sortable.create(col, {
          group: "tareas-board", animation: 150, ghostClass: "opacity-40",
          onEnd: (evt) => this.onDrop(evt),
        }));
      });
    },
    async onDrop(evt) {
      const toCol = evt.to.dataset.columnaId;
      const tareaId = evt.item.dataset.tareaId;
      const nuevoOrden = evt.newIndex;  // ← posición destino persistida
      evt.item.style.opacity = "0.5";
      try {
        const csrf = document.cookie.split("; ")
          .find(c => c.startsWith("csrftoken="))?.split("=")[1] || "";
        const form = new FormData();
        form.append("columna_id", toCol);
        form.append("orden", nuevoOrden);
        const resp = await fetch(`/tareas/tarea/${tareaId}/mover/`, {
          method: "POST",
          headers: { "X-CSRFToken": csrf, "HX-Request": "true" },
          body: form,
        });
        const text = await resp.text();
        if (!resp.ok) {
          window.dispatchEvent(new CustomEvent("tareas-error",
            { detail: { msg: "Error al mover" } }));
          evt.from.appendChild(evt.item); evt.item.style.opacity = "1"; return;
        }
        document.getElementById("tareas-board").innerHTML = text;  // swap completo
      } catch (err) {
        evt.from.appendChild(evt.item); evt.item.style.opacity = "1";
      }
    },
  };
}
```

> **CSRF** (regla §14.3): el `fetch` manda `X-CSRFToken` desde la cookie. Cualquier form HTMX (`hx-post`) lleva `{% csrf_token %}`. El test client NO valida CSRF — los 403 solo salen en prod.

---

## 8. Design System (reusa el de EspritOS)

No se diseña un sistema nuevo: Tareas **hereda el design system del monolito** (dark zinc/amber, clases `hm-*`, layout del sidebar). Brand origin: `existing` (EspritOS ya tiene identidad). Register: `product`. El builder calca el look del kanban del CRM (`templates/crm/partials/kanban_board.html`): columnas `rounded-xl border border-zinc-800 bg-zinc-900/30`, tarjetas `bg-zinc-900 border-zinc-800`, acentos por prioridad. Punto de color por prioridad (P1 rojo, P2 ámbar, P3 zinc, P4 zinc-600) y por `columna.color` (hex configurable). Banned: glassmorphism por defecto, gradientes de texto, eyebrows uppercase por sección. Motion: bajo (transiciones `hm-transition`, sin bounce). `prefers-reduced-motion` respetado.

---

## 9. Autorización — matriz de permisos por rol

### 9.1 Matriz `tareas` (módulo nuevo)

| Rol | V | C | E | D | Cómo se aplica |
|-----|---|---|---|---|----------------|
| **admin** | ✅ | ✅ | ✅ | ✅ | Automático vía `_FULL` al añadir `TAREAS` al enum. No tocar. |
| **admin_sucursal** (Carlos) | ✅ | ✅ | ✅ | ✅ | Automático vía `ALL_MODULES` en su matriz. Unrestricted → ve todo. No tocar. |
| **supervisor_ventas** (Andrea) | ✅ | ✅ | ✅ | ✅ | Agregar fila. Unrestricted → ve todo. |
| **direccion** (Humberto, Laura) | ✅ | ✅ | ✅ | ✅ | Agregar fila. Unrestricted → ve todo. |
| **vendedor** (Valeria, Daniela…) | ✅ | ✅ | ✅ | — | Agregar fila. Ve lo suyo + sus tableros. |
| **cobranza** (Montserrat) | ✅ | ✅ | ✅ | — | Agregar fila. Ve lo suyo + sus tableros. |
| **almacenista** (René) | ✅ | ✅ | ✅ | — | Agregar fila. Ve lo suyo + sus tableros. |
| **colaborador** (Yunuen/RH) ⭐NUEVO | ✅ | ✅ | ✅ | — | Grupo nuevo. ÚNICO módulo: tareas. Ve lo suyo + sus tableros. |
| **cajero** | — | — | — | — | **Omitido** (decisión Beto 28/06). Automático: su matriz da False a todo `ALL_MODULES` salvo POS. No tocar. |

⭐ **Rol `colaborador`**: grupo ligero para RH y cualquier persona del equipo que solo usa el tracker (no ventas/CRM). Su único módulo con permiso es `tareas`; Agenda y Aprendizaje ya están abiertos a todo autenticado. Yunuen coordina su área porque es **dueña/miembro del tablero "RH"** (RLS §4.2), no por su rol.

### 9.2 Dónde se siembra (CRÍTICO — dos lugares)

> Hallazgo de la auditoría 23/06/2026 (migración `0026_seed_base_rolepermission_matrix.py`): **prod siembra permisos por MIGRACIÓN, no por el comando** `seed_permissions` (el deploy corre `migrate`, no el comando). Por eso hay que tocar **ambos**:

1. **`apps/core/management/commands/seed_permissions.py`** (`DEFAULTS`) — fuente de verdad para dev/test. Agregar la fila `"tareas": (...)` a cada rol relevante + el rol `"colaborador"` completo.
2. **Migración de datos nueva** en `apps/core/migrations/00XX_seed_tareas_permissions.py` — siembra en prod. `get_or_create` (NO `update_or_create`) de cada grant de tareas + `Group.objects.get_or_create(name="colaborador")`. `reverse_code` = noop. Patrón literal de `0026` (no importar de `seed_permissions` — las migraciones son inmutables).

---

## 10. Sidebar

Nueva sección visual **"Equipo"**:

1. `apps/core/sidebar.py` — agregar al dict `SIDEBAR_SECCIONES` (después de `"inicio"`):
   ```python
   "inicio": "Inicio",
   "equipo": "Equipo",   # ← NUEVO
   "vender": "Vender",
   ...
   ```
2. En `SIDEBAR_TREE`, insertar **contiguo tras "Mi Día"** (el template usa `{% regroup arbol by seccion %}` → los nodos de una sección deben ir juntos y el orden visual lo da la posición en la lista):
   ```python
   SidebarItem(label="Tareas", icon="kanban", url_name="tareas:lista",
               modulo="tareas", seccion="equipo"),
   ```
   `es_visible()` ya valida `check_permission(user, "tareas", "ver")` (`apps/core/sidebar.py:70-71`). El icono `kanban` ya existe (lo usa el Pipeline del CRM).

---

## 11. Puntos de integración obligatorios (checklist — el orden importa)

> Reglas no-negociables de EspritOS: una app nueva que no toque estos puntos queda rota o insegura. Cada uno es un cambio **mínimo y aditivo** a `apps/core`/`config` (permitido; no es violar el aislamiento).

| # | Archivo | Cambio | Ref. original |
|---|---------|--------|---------------|
| 1 | `config/settings/base.py` (LOCAL_APPS, ~línea 109) | Agregar `"apps.tareas",` al final de la lista | `:68-109` |
| 2 | `apps/core/models.py` (enum `Modulo`, ~256-289) | Agregar `TAREAS = "tareas", "Tareas"`. Genera migración `AlterField` en core (choices cambian) | `:256-289` |
| 3 | `apps/core/middleware.py` (`URL_TO_MODULO`, 128-161) | Agregar `("/tareas/", "tareas"),` | `:128-161` |
| 4 | `apps/core/management/commands/seed_permissions.py` (`DEFAULTS`, 27-210) | Filas `"tareas"` por rol + rol `"colaborador"` (§9.1) | `:27-210` |
| 5 | **Migración nueva** `apps/core/migrations/00XX_seed_tareas_permissions.py` | `get_or_create` grants tareas + Group `colaborador` (§9.2) | patrón `0026` |
| 6 | `apps/core/sidebar.py` (`SIDEBAR_SECCIONES` + `SIDEBAR_TREE`) | Sección "Equipo" + SidebarItem Tareas (§10) | `:89-96`, `:100-233` |
| 7 | `config/urls.py` (~54) | `path("tareas/", include("apps.tareas.urls", namespace="tareas")),` | `:16-59` |
| 8 | `apps/notifications/signals.py` | Receiver de `tarea_asignada` (§6.3) | `:17-20` |
| 9 | `apps/notifications/services.py` + `tasks.py` | `notificar_asignados_tarea` + task email + plantillas (§6.4-6.5) | `:61-116`, `tasks.py:33-103` |
| 10 | `apps/core/management/commands/setup_pilot_users.py` (`ROLES_VALIDOS`, ~44-47) | Agregar `"colaborador"` al set (para crear a Yunuen: `--rol colaborador`) | `:44-47` |
| 11 | `apps/tareas/` estructura completa (molde `apps/agenda`) | `__init__, apps, admin, forms, models, services, signals, urls, views, migrations/, tests/` | — |
| 12 | `npm run build:css` | Compilar clases Tailwind nuevas (§14.6) | — |

> **CRÍTICO middleware (#3)**: la mecánica evolucionó a **fail-closed** (auditoría 23/06/2026, `apps/core/middleware.py:200-292`). Si la URL `/tareas/` **no** está en `URL_TO_MODULO`, cae en la rama "ruta sin módulo": exige solo `login_required` (cualquier autenticado entra, **sin** check de permiso de módulo). Es decir, **sin el punto #3 el módulo queda abierto a todo autenticado** — saltándose la matriz de permisos. Por eso es obligatorio.

> **CRÍTICO seed (#4+#5)**: si una migración cambia `RolePermission`, actualizar el seed en el **MISMO commit** — si no, re-correr el comando revierte la migración. Y sin la migración (#5), prod queda sin permisos de tareas (deploy no corre el comando).

---

## 12. Build Order (numerado — sigue paso por paso)

> Rama: `feature/tareas`. Tests aislados (`scripts/test_docker.ps1`, proyecto `espritos-test`, NO toca prod). Commits atómicos. Al final: merge a `master` → `scripts/deploy_prod.ps1` (gate de suite adentro).

**Paso 1 — Scaffolding de la app.**
Crear `apps/tareas/` con el molde de `apps/agenda`: `__init__.py`, `apps.py` (§6.2), `admin.py`, `forms.py`, `models.py`, `services.py`, `signals.py`, `urls.py`, `views.py`, `migrations/__init__.py`, `tests/__init__.py`. Registrar en `LOCAL_APPS` (punto #1).

**Paso 2 — Modelos + migración inicial.**
Escribir `models.py` (§3.2) completo. `python manage.py makemigrations tareas`. Verificar índices y el `UniqueConstraint(tablero, orden)` en la migración. NO editar la migración una vez aplicada (regla §14.8).

**Paso 3 — Enum Modulo + permisos (mismo commit).**
Punto #2 (enum `TAREAS`) → `makemigrations core` (AlterField). Punto #4 (`DEFAULTS` + rol `colaborador`). Punto #5 (migración de datos de grants + Group `colaborador`). Punto #10 (`ROLES_VALIDOS`). Todo en un commit para no dejar la matriz inconsistente.

**Paso 4 — Servicios + RLS.**
`services.py` (§4.3): `_es_unrestricted`, `tareas_visibles_para`, `tableros_visibles_para`, `tareas_por_entidad`, `mover_tarea`, `crear_tarea`, `asignar`, `tablero_render`.

**Paso 5 — Señal de asignación.**
`signals.py` (§6.1). `apps.py.ready()` la carga (§6.2).

**Paso 6 — Integración notifications.**
Puntos #8 y #9: receiver + `notificar_asignados_tarea` + Celery task + plantillas `asignacion_tarea.{html,txt}`. Reusa `notify()` existente.

**Paso 7 — Forms.**
`forms.py` (§5.3): `TableroForm`, `ColumnaForm`, `TareaForm` (con `_sync_asignados` + `nuevos_asignados_ids`). `vencimiento` = input date nativo.

**Paso 8 — Vistas + URLs.**
`views.py` (§5.1) + `urls.py` (§5.2). Punto #7 (montar en `config/urls.py`). Punto #3 (`URL_TO_MODULO`). HTMX-aware en board y move.

**Paso 9 — Templates + DnD.**
`lista.html`, `tablero.html` (con `tareasDnd()`, §7.2), `partials/{board,form,detalle}.html`. Calcar look del kanban CRM. `{% csrf_token %}` en cada form HTMX.

**Paso 10 — Sidebar.**
Punto #6: sección "Equipo" + SidebarItem.

**Paso 11 — Admin.**
`admin.py`: `TableroAdmin`, `ColumnaAdmin` (inline en Tablero opcional), `TareaAdmin` (list_display titulo/tablero/columna/prioridad/creado_por; raw_id_fields; readonly entidad_tipo/entidad_id), `ComentarioTarea`/`MovimientoTarea` read-only.

**Paso 12 — Compilar CSS.**
`npm run build:css` (punto #12). Verificar que las clases nuevas existen en el bundle (no Play CDN).

**Paso 13 — Tests (§13).**
`tests/{conftest,test_models,test_rls,test_services,test_views,test_signal_asignacion}.py`. RLS y N+1 obligatorios.

**Paso 14 — Suite aislada verde.**
`.\scripts\test_docker.ps1 apps/tareas/tests --create-db`. Correr también `apps/core/tests/test_app_isolation.py` (aislamiento) y los de notifications.

**Paso 15 — Seed + smoke local.**
`python manage.py seed_permissions` (dev). Crear a Yunuen: `python manage.py setup_pilot_users --user yunuen --rol colaborador --sucursal CREM`. Smoke (§16).

**Paso 16 — Merge + deploy.**
Merge `feature/tareas` → `master` con todo verde. `.\scripts\deploy_prod.ps1` (gate de suite + migrate, que aplica la migración de permisos en prod).

---

## 13. Testing Strategy

Pytest + `scripts/test_docker.ps1` (proyecto `espritos-test`). Fixtures calcan `apps/agenda/tests/conftest.py` (usuarios con rol/grupo + sucursal).

**`test_rls.py` (OBLIGATORIO — regla #4 de EspritOS).** Calca `apps/agenda/tests/test_rls.py`:
- `test_persona_no_ve_tareas_ajenas`: A crea tarea en su tablero; B (sin relación) no la ve vía `tareas_visibles_para(B)`.
- `test_asignado_ve_su_tarea`: A asigna a B → B la ve.
- `test_miembro_ve_todo_el_tablero`: B es miembro del tablero de A → B ve TODAS las tareas del tablero (cubre "todo el tablero de su área").
- `test_dueno_tablero_ve_todo_su_tablero`: el dueño ve todas, aunque no sea asignado.
- `test_gerencia_ve_todo`: admin/supervisor_ventas/admin_sucursal/direccion ven tareas de cualquiera.
- `test_colaborador_solo_ve_lo_suyo_y_sus_tableros`: `colaborador` sin membresía no ve tareas ajenas.

**`test_services.py`**: `mover_tarea` crea `MovimientoTarea` con `dias_en_columna_previa` correcto, es atómico, rechaza columna de otro tablero (`TareaError`), persiste `orden`. `asignar` devuelve solo ids nuevos (idempotente).

**`test_signal_asignacion.py`**: asignar dispara `tarea_asignada` → crea `Notification` in-app para el asignado nuevo; NO re-notifica al editar sin cambiar asignados.

**Performance / N+1** (regla #6, calca `apps/crm/tests/test_views.py:275-294`): `CaptureQueriesContext` sobre `tablero_detalle` y `lista` → `< 20` queries con varias tareas/asignados.

**CSRF** (regla §14.3): test de que `tarea_mover` requiere POST + el template incluye `{% csrf_token %}`/`X-CSRFToken`. (Recordar: el test client NO valida CSRF; el riesgo real es prod.)

**Aislamiento** (`apps/core/tests/test_app_isolation.py`): correr la suite completa — `tareas` solo importa de sí misma + `core` + `IMPORTABLE_BY_ALL` (`core, auditoria, datos_hm, agenda`). NO importa `crm`/`notifications`/etc.

---

## 14. Reglas No Negociables (el blueprint las impone al Builder)

1. **Aislamiento de apps** (`apps/core/tests/test_app_isolation.py`): `tareas` NO importa `crm`/`pulso`/`notifications`/etc. Para el kanban se **copia el patrón del CRM, no se importa**. Cross-módulo vía señales o servicios en core. (`tareas` NO entra a `IMPORTABLE_BY_ALL` en el MVP.)
2. **RLS obligatorio con test**: toda vista que muestra tareas filtra por `services.tareas_visibles_para` / `tableros_visibles_para`. `test_rls.py` prueba que A no ve lo de B.
3. **CSRF en cada `hx-post`** (`memory/htmx_csrf_sin_config_global.md`): `{% csrf_token %}` en el form o `X-CSRFToken`. El test client NO valida CSRF — botón HTMX "muerto" en prod + tests verdes = CSRF.
4. **HTMX no hace swap de 4xx** (`memory/htmx_swap_422_validacion.md`): si se usan respuestas 4xx para validación, forzar swap con `htmx:beforeSwap`, o quedan invisibles.
5. **Comentarios Django multilínea** (`memory/django_comments_multilinea.md`): `{# #}` SOLO en una línea. Multilínea = `{% comment %}...{% endcomment %}`. Validar cada `.html` antes de commit.
6. **Tailwind compilado, no Play CDN** (`memory/tailwind_build_real.md`): clases nuevas requieren `npm run build:css`. Cache-busting vía `STORAGES` (`memory/static_cachebust_django51.md`).
7. **Date picker en modales = input nativo** (`memory/flatpickr_dialog_static.md`): `vencimiento` en `<dialog>` usa `<input type="date">` nativo, NO Flatpickr.
8. **Migraciones aplicadas NO se editan** (regla #5 EspritOS). La migración de permisos usa `get_or_create` y es inmutable.
9. **Sin N+1** (regla #6): `select_related`/`prefetch_related` (asignados M2M, columna, tablero). Test de performance falla si una vista hace >20 queries.
10. **Tests aislados** con `scripts/test_docker.ps1` (proyecto `espritos-test`). **Deploy** solo desde `master`, vía `scripts/deploy_prod.ps1`.
11. **Remitente de correo** = `settings.DEFAULT_FROM_EMAIL` (env var), NUNCA hardcodear un correo.
12. **El módulo NO queda fuera de `URL_TO_MODULO`** (punto #3): si falta, el middleware deja `/tareas/` abierto a todo autenticado.

---

## 15. CLAUDE.md para `apps/tareas` (pegar en el repo EspritOS)

> Esta app vive dentro del monolito. Este CLAUDE.md complementa el `CLAUDE.md` raíz de EspritOS (comandos del cluster, reglas globales). Colócalo en `apps/tareas/CLAUDE.md`.

```markdown
# apps/tareas — Task Tracker

Tracker kanban de tableros configurables para todo el equipo de HM. App Django
dentro del monolito EspritOS. Server-rendered (HTMX + Alpine + SortableJS), sin SPA.

## Comandos (desde la raíz del repo EspritOS)
- `.\scripts\test_docker.ps1 apps/tareas/tests --create-db` — suite de la app (NO toca prod)
- `python manage.py makemigrations tareas` — migraciones del modelo
- `python manage.py seed_permissions` — re-siembra matriz de permisos (dev)
- `npm run build:css` — compilar Tailwind tras agregar clases nuevas
- `python manage.py setup_pilot_users --user <u> --rol colaborador --sucursal CREM` — alta de colaborador

## Arquitectura
- `models.py` — Tablero · Columna · Tarea · MovimientoTarea · ComentarioTarea.
  Calca apps/crm (Pipeline/Stage/Opportunity/StageHistory) + GenericFK de agenda.
- `services.py` — TODA la lógica: RLS (`tareas_visibles_para`, `tableros_visibles_para`),
  `mover_tarea` (atómico + historial), `crear_tarea`, `asignar`, `tablero_render`.
  Las vistas SOLO orquestan.
- `signals.py` — `tarea_asignada` (la escucha apps.notifications; tareas NO importa
  notifications — aislamiento).
- `views.py` — HTMX-aware: si `request.htmx`, devuelve partials. Board re-renderiza
  el partial completo tras mover.

## Data flow
Drag&drop (SortableJS) → fetch POST `/tareas/tarea/<pk>/mover/` con X-CSRFToken →
`services.mover_tarea` (atómico, crea MovimientoTarea, persiste orden) → swap del
`#tareas-board`. Asignar → form deja `nuevos_asignados_ids` → la VIEW dispara
`tarea_asignada.send()` → notifications crea in-app + encola correo (fallback síncrono).

## RLS (memorízalo)
Ves una tarea si: la creaste ∪ te asignaron ∪ eres dueño/miembro de su tablero.
Roles gerenciales (admin, admin_sucursal, supervisor_ventas, direccion) + superuser
ven todo. Implementado en `services.tareas_visibles_para`. Toda vista lo usa.

## Reglas No Negociables
1. NO importar otras apps de negocio (crm, pulso, notifications). Copia patrones,
   no imports. Cross-módulo via señal/servicio core. (test_app_isolation)
2. RLS en cada vista vía services. test_rls.py obligatorio (A no ve lo de B).
3. `{% csrf_token %}` / X-CSRFToken en cada hx-post (el test client no valida CSRF).
4. `{# #}` solo una línea; multilínea = {% comment %}. Valida cada .html.
5. vencimiento en <dialog> = <input type="date"> nativo, NO Flatpickr.
6. Tras clases Tailwind nuevas: `npm run build:css` (no Play CDN).
7. No editar migraciones aplicadas. Migración de permisos = get_or_create, inmutable.
8. select_related/prefetch_related — vistas < 20 queries (test de performance).
9. Remitente de correo = settings.DEFAULT_FROM_EMAIL (env), nunca hardcodeado.
```

---

## 16. Verificación de cierre (smoke manual)

Tras suite verde, en local/preview:
1. Crear tablero "RH" (dueño Beto), agregar columnas To Do · En curso · Hecho (es_terminal).
2. Agregar a Yunuen como **miembro** del tablero.
3. Crear tarea "Actualizar contratos", prioridad P2, asignar a Yunuen → confirmar: (a) notificación in-app a Yunuen, (b) correo a su email desde `DEFAULT_FROM_EMAIL`.
4. Mover la tarjeta To Do → En curso (drag) → confirmar que persiste posición y se creó `MovimientoTarea` (revisar admin o detalle).
5. Agregar un comentario en la tarea → aparece en la bitácora.
6. **RLS**: loguear como Valeria (vendedora, sin relación con el tablero RH) → NO ve la tarea de Yunuen. Loguear como Yunuen → ve TODAS las tareas del tablero RH (membresía). Loguear como Beto/admin → ve todo.

---

> **Siguiente paso:** The Builder construye desde este blueprint en `feature/tareas`. Cero cambios a apps existentes salvo los 12 puntos de integración (§11). Cierre = suite verde + smoke (§16) + deploy desde `master`.
