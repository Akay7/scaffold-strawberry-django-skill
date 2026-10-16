---
name: scaffold-strawberry-django
description: Scaffold Strawberry-Django GraphQL artifacts (Type, Filter, Inputs, Mutations, schema wiring) for a Django model. Use when adding GraphQL read/CRUD support for a model.
license: MIT
user-invocable: true
allowed-tools: Bash(grep *), Bash(find *), Bash(ls *), Bash(head *), Bash(python *), Bash(pytest *), Bash(mypy *), Bash(pyright *), Bash(uv run *), Read(*), Edit(*), Write(*)
---

# Skill: scaffold-strawberry-django

Generate the GraphQL layer for a Django model: a **Type + Filter**, **Inputs + Mutations**, and
wire both into the schema. If the repo has a GraphQL conventions doc (e.g. `docs/graphql.md`),
read it first and let it override anything below.

**Scope**: this scaffolds a plain object/list-field API — list query fields are
`strawberry_django.offset_paginated` (a plain `list[<Type>]` only for small fixed sets), and
objects aren't required to implement Relay's `Node` interface. It does not scaffold Relay-style
**Global Object Identification** (a global `node(id: ID!)` root field resolving any type by opaque ID) or cursor-based **Connections** (`edges`/`node`/
`pageInfo`, `first`/`after` arguments). If the target project's GraphQL layer is Relay-routed, treat
everything below as needing translation to that shape rather than applying it as-is.

Examples use the `Fruit` / `Color` models from the Strawberry-Django documentation as neutral
placeholders: `Fruit` has a `name`, a nullable FK `color` → `Color` (reverse name `fruits`) and an
owner FK `user`; where an example needs more, it adds `sold` / `sold_at` / `status` / `stock` and
an M2M `tags` → `Tag` to `Fruit`. Substitute the real model.
Before writing, find one existing Type and one existing mutation in the repo to copy the exact
local idiom (directory layout — `<app>/types/<stem>.py` vs a flat `<app>/types.py` — naming,
imports, whether each app re-exports through a `schema.py`):

```bash
grep -rl "strawberry_django.type" . | grep -v __pycache__ | head
grep -rl "strawberry_django.mutation\|strawberry_django.input" . | grep -v __pycache__ | head
```

Long reference material lives next to this file; read it only when the step says so:
`references/schema-and-limits.md` (schema object, extensions, limiter settings),
`references/authorization-and-leaks.md` (authorization-sensitive inputs, existence oracles),
`references/concurrency.md` (update writes, and the optional version check for shared rows),
`references/testing.md` (test conventions).

## Shared modules (find or create)

Every resolver below imports from two tiny shared modules. Look for them first
(`grep -rn "def apply_input\|def sync_m2m\|def get_by_id\|def reread\|class IsAuthenticated\|IsStaff()" .`);
create only what is missing, in the project's shared package (e.g. `<pkg>/graphql.py`,
`<pkg>/permissions.py`).

### `graphql.py` — `Info`, `apply_input`, `sync_m2m`, `get_by_id` and `reread`

```python
from collections.abc import Iterable
from enum import Enum
from typing import Protocol

import strawberry
from django.core.exceptions import ValidationError
from django.db import models
from strawberry.django.context import StrawberryDjangoContext
from strawberry.types.base import get_object_definition
from strawberry_django.fields.types import (
    ManyToManyInput,
    ManyToOneInput,
    OneToManyInput,
    OneToOneInput,
)
from strawberry_django.optimizer import optimize

#: Bare `strawberry.Info` defaults both parameters to `Any`, which turns every
#: `info.context.…` into an unchecked expression; annotate resolvers with this.
Info = strawberry.Info[StrawberryDjangoContext, object]


def apply_input(data: object, instance: models.Model) -> list[str]:
    """Assign every *set* field of a Strawberry input onto ``instance``.

    ``strawberry.UNSET`` (an omitted key on a ``partial=True`` input) is skipped, so
    one call serves create and update. A forward relation arrives as ``{ set: <id> }``;
    its id goes to ``<field>_id`` so ``full_clean()`` is what rejects an id naming no
    row. A choice enum is assigned by its value (a ``GENERATE_ENUMS_FROM_CHOICES`` member
    is a plain ``Enum``, which the model field would store as ``"<EnumName>.<member>"``).
    M2M and reverse-FK inputs need a saved instance and are skipped — apply them with
    ``sync_m2m`` after ``save()``.

    Each value goes to the *model* field the input field maps to — its
    ``field_name=`` override (``title: str = strawberry_django.field(field_name="name")``)
    if it has one. An input field that isn't a model field raises ``FieldDoesNotExist``
    instead of being set as a stray attribute: take such a field (a flag, a
    precondition) out of the input before calling this.

    Returns the model field names it set, so an update can ``save(update_fields=...)``
    instead of rewriting the whole row (``references/concurrency.md``).
    """
    applied: list[str] = []
    for field in get_object_definition(data, strict=True).fields:
        value = getattr(data, field.name, strawberry.UNSET)
        if value is strawberry.UNSET or isinstance(value, ManyToManyInput | ManyToOneInput):
            continue
        name = getattr(field, "django_name", None) or field.name
        model_field = instance._meta.get_field(name)
        if isinstance(value, OneToManyInput | OneToOneInput):
            setattr(instance, model_field.attname, value.set)
        else:
            setattr(instance, name, value.value if isinstance(value, Enum) else value)
        applied.append(model_field.name)  # `color`, also when the input maps it as `color_id`
    return applied


class RelatedRows[M: models.Model](Protocol):
    """What ``sync_m2m`` uses of a to-many manager: an M2M one, or a reverse FK's."""

    model: type[M]

    def set(self, objs: Iterable[M]) -> None: ...
    def add(self, *objs: M) -> None: ...
    def remove(self, *objs: M) -> None: ...


def sync_m2m[M: models.Model](
    manager: RelatedRows[M],
    value: object,
    field: str,
    rows: models.QuerySet[M] | None = None,
) -> None:
    """Apply a to-many input (M2M or reverse FK, both ``{ set, add, remove }``) after
    ``save()`` — ``apply_input`` skips these fields because they need a saved instance.

    The input changes the relation in one of two ways:

    - ``set`` — the simple way: the complete list. Whatever isn't in it is unlinked
      (``[]`` unlinks everything).
    - ``add`` / ``remove`` — link or unlink only the rows named; the rest stay linked.

    Both ways in one request is ambiguous and refused. An omitted field is ``UNSET`` —
    on a create input as well as a ``partial=True`` one — and changes nothing.

    Ids are looked up in ``rows`` (default: every row of the related model); pass the
    caller's visible set when linking must be authorized. An id it doesn't hold —
    unknown, someone else's or malformed — is one "Not found.", so they can't be told
    apart. Every id is checked before anything is written. Errors are keyed ``field``.
    """
    if value is strawberry.UNSET:
        return
    assert isinstance(value, ManyToManyInput | ManyToOneInput)
    replace = value.set is not strawberry.UNSET
    if replace and (value.add or value.remove):
        raise ValidationError({field: "Send either `set`, or `add` / `remove` — not both."})
    candidates = manager.model._default_manager.all() if rows is None else rows

    def lookup(ids: list[strawberry.ID] | None) -> list[M]:
        wanted = set(ids or [])
        try:
            found = list(candidates.filter(pk__in=wanted))
        except (ValueError, ValidationError):  # a malformed id: same answer as unknown
            found = []
        if len(found) != len(wanted):
            raise ValidationError({field: "Not found."})
        return found

    to_set = lookup(value.set) if replace else None
    to_add = lookup(value.add) if value.add else []
    to_remove = lookup(value.remove) if value.remove else []
    if to_set is not None:
        manager.set(to_set)
    if to_add:
        manager.add(*to_add)
    if to_remove:
        manager.remove(*to_remove)


def get_by_id[M: models.Model](queryset: models.QuerySet[M], id: object) -> M:
    """``queryset.get(pk=id)``, with a malformed id reported as not-found.

    A malformed id raises ``ValueError`` on an integer pk — which
    ``handle_django_errors`` does not convert, so it would be a top-level error —
    and ``ValidationError`` on a UUID pk. Both are answered by an empty lookup,
    which raises Django's own ``DoesNotExist`` — same message as an unknown id —
    so an unknown, invisible or malformed id is one indistinguishable ``OperationInfo``.
    """
    try:
        return queryset.get(pk=id)
    except (ValueError, ValidationError):
        return queryset.none().get()


def reread[M: models.Model](instance: M, info: Info) -> M:
    """``instance`` as stored, loaded the way a query loads it for this selection.

    A create/update payload is otherwise the in-memory instance, which the optimizer
    never sees: a selected ``annotate=`` field raises, a nested Type's ``get_queryset``
    is skipped (a related row the caller may not see is returned), and a value the
    database shapes on write (a ``DecimalField``'s scale) shows what was sent rather
    than what was stored. One query, shaped by the payload's selection.
    """
    return optimize(type(instance)._default_manager.filter(pk=instance.pk), info).get()
```

Create and update both go through `apply_input` — never hand-copy `data.<field>` onto the
instance, which yields two field lists that drift apart. It writes each input field to the model
field it maps to (a `field_name=` rename included) and returns model field names, so
`save(update_fields=…)` never meets an input-only name; an input field with no model field
behind it is refused (`FieldDoesNotExist`) rather than silently dropped on create. Their M2M
and reverse-FK inputs go through `sync_m2m` after the save. Update and delete look their object
up with `get_by_id`; create and update return it through `reread`.

### `permissions.py` — shared permission extensions

```python
import strawberry
from django.core.exceptions import PermissionDenied
from graphql import GraphQLError
from strawberry_django.fields.types import OperationInfo
from strawberry_django.permissions import IsAuthenticated as _IsAuthenticated
from strawberry_django.permissions import IsStaff

#: What a client matches to tell "the session is gone" from any other failure. The message
#: stays display text — reworded or translated freely — so it is never the contract.
UNAUTHENTICATED = "UNAUTHENTICATED"


class IsAuthenticated(_IsAuthenticated):
    """strawberry_django's check, with ``UNAUTHENTICATED`` on every refusal it reports: as
    ``extensions.code`` on a top-level error, and as ``code`` on an ``OperationInfo``
    message. A silent refusal (an empty list or page, ``null``) is not an error and has none."""

    def handle_no_permission(self, exception: BaseException, *, info: strawberry.Info) -> object:
        try:
            result = super().handle_no_permission(exception, info=info)
        except PermissionDenied as exc:
            raise GraphQLError(self.message, extensions={"code": UNAUTHENTICATED}) from exc
        if isinstance(result, OperationInfo):
            for message in result.messages:
                message.code = UNAUTHENTICATED
        return result


staff_required = IsStaff()
authenticated_required = IsAuthenticated()
```

Import these instead of constructing `IsAuthenticated()` / `IsStaff()` per field, so the denial
message and options (`message=`) are configured in one place and every gated field reads the same.
(Inline instances are harmless on current strawberry-graphql-django — all instances of a class
share one `@isAuthenticated` directive; before 0.89 two of them broke the schema build.)

**Clients detect a lost session by the code, never the message.** A client that must stop on "not
signed in" — sign the user out, halt a batch upload instead of failing every remaining item —
matches `UNAUTHENTICATED`: `errors[].extensions.code` on a query field or a namespace-gated
mutation, `messages[].code` on a per-mutation gate's `OperationInfo`. The text "User is not
authenticated." is display text: once it is translated or reworded, a client matching it breaks
silently. The code is set by the check itself, not inferred from who is asking, so an anonymous
caller refused for any other reason never looks like a lost session. `IsStaff` and other checks
get no code — a signed-in user lacking a right has not lost the session. Leave `fail_silently` at
its default `True`: `False` raises before the union check, so a per-mutation gate's typed
`OperationInfo` becomes a top-level error and silent list/`null` refusals become errors.

### Delete returns the model's own type

The delete mutation returns `<ModelName>Type`, not a `bool` or a `DeletionResult`-style wrapper —
strawberry_django's own convention for its generated delete mutations
(https://strawberry.rocks/docs/django/guide/mutations). A `bool` can't work: with
`handle_django_errors=True` the success type is unioned with `OperationInfo`, and GraphQL unions
can only contain object types (`InvalidUnionTypeError: Type 'bool' cannot be used in a GraphQL
Union`). `Model.delete()` sets the instance's pk to `None`, so save and restore it so the response
carries a real id:

```python
pk = instance.pk
instance.delete()
instance.pk = pk  # Django nils it post-delete; restore for the response
```

## Invocation

```
/scaffold-strawberry-django <ModelName|model_file_stem> [--type-only | --mutations-only] [--no-update]
```

- `<ModelName>` — the Django model (e.g. `Fruit`) or its file stem (`fruit`).
- `--type-only` — generate only the Type + Filter (query/read side).
- `--mutations-only` — generate only Inputs + Mutations (assumes the Type already exists).
- `--no-update` — generate create + delete only, skip the update mutation/`UpdateInput`.

Default (no flag) generates the Type + Filter **and** create/update/delete mutations, then wires both into the schema.
To add a field to a model that is already scaffolded, see "Extending an already-scaffolded model".

---

## Steps

### 1. Locate the model and its app

```bash
grep -rln "class <ModelName>(" . | grep -v __pycache__
```

From the path capture `<app>` and `<stem>`. Read the model file to learn:

- Fields and their types (scalars, FKs, M2M, OneToOne).
- The primary-key type (`id: auto` exposes whatever the model uses — int or UUID).
- Any `position`/ordering field (→ may need a reorder mutation).
- Which related models already have Types/Filters you can import and reuse.
- Every FK *to* this model and its `on_delete` (`PROTECT`/`RESTRICT` changes delete, step 3).

Derive the names: `<ModelName>Type`, `<ModelName>Filter`, `<ModelName>CreateInput`,
`<ModelName>UpdateInput`, `<ModelName>Mutations`, and the camelCase query/mutation field names
(`<modelName>` / `<model_name_plural>`).

### 2. Generate the Type + Filter (place it next to other types)

Order within the file: **Filter first, then Type** (the Type references the Filter).

- `@strawberry_django.filter_type(<ModelName>, lookups=True)` — `id: auto`; scalar fields `auto`
  (including choice fields — they become enum lookups; never hand-annotate
  `str | None = strawberry.UNSET`); FK fields as a nested relation filter,
  `color: ColorFilter | None = strawberry.UNSET` (define a minimal filter for the related model if
  none exists — with a UUID pk the lookup takes ids as *strings*, and a malformed one is an error,
  not an empty page). The nested filter also covers "any of several ids"
  (`{ color: { id: { inList: [...] } } }`), so no top-level `colors: [ID]` argument is needed.
  **Reverse relations filter the same way**: on the *parent's* filter, a reverse FK is
  `fruits: FruitFilter | None = strawberry.UNSET`, so "colors that have an unsold fruit" is
  `{ fruits: { sold: { exact: false }, soldAt: { isNull: true } } }` — no `filter_field` with a
  hand-built `Exists(...)`. All lookups inside one nested filter apply to the **same** related row
  (one `INNER JOIN`, both conditions in its `WHERE`). The related filter needs `lookups=True` for
  `isNull`/`exact`, and it may expose a field the related Type doesn't (filterable is not readable).
  The join returns a parent once per matching child; when more than one child can match, the
  client passes the `DISTINCT: true` argument every filter type gets. Reach for a custom `@strawberry_django.filter_field` method
  **only** when the filter cannot be expressed as a narrowing of the base queryset (e.g. selecting a
  *different*, disjoint scope than the type's `get_queryset` default) — it may replace the queryset
  by returning `tuple[QuerySet, Q]` from a `(self, value, queryset, info, prefix)` signature.
- `@strawberry_django.order_type(<ModelName>)` `class <ModelName>Order` — the fields the client may
  sort by, each annotated `Ordering` (`from strawberry_django.ordering import Ordering`).
- `@strawberry_django.type(model=<ModelName>, filters=<ModelName>Filter, ordering=<ModelName>Order)`:
  - Use `auto` for field annotations wherever possible (let Strawberry infer from the model).
  - Expose nested object fields as the related type, e.g. `color: ColorType`. Annotate
    `RelatedType | None` if the relation is nullable **or the related Type has a `get_queryset`**
    — even over a non-null FK. That scope applies to the relation (next bullets), so a related row
    the viewer may not see comes back `null`. On a non-null field that `null` is an error
    ("Cannot return null for non-nullable field FruitType.user") which nulls the parent and, through
    a non-null list, the whole response: one fruit with a hidden owner gives `data: null`. Plain
    `RelatedType` only when the target Type has no scope. **Never denormalize a related scalar into a
    resolver field** (e.g. an `owner_username` resolver returning `self.user.username`) — expose the
    relation (`user: UserType | None`) and let clients select `user { username }`; the optimizer
    handles the join.
  - A reverse FK is `fruits: list[FruitType]` (the `related_name`); the optimizer prefetches it for
    the whole list in one extra query. That list has no row limit — every child of every parent on
    the page — so when a parent can have many children, make it
    `fruits: OffsetPaginated[FruitType] = strawberry_django.offset_paginated()` instead (see "A plain
    list has no row limit" in step 4).
  - **A Type that other rows point at filters its own rows.** A row the viewer may see can point at
    rows they may not — a visible fruit's `user` is someone else's account. Give the related Type
    its own `get_queryset` with *its* visibility rule; don't rely on the parent's scope to cover
    its relations, and don't filter them in a resolver. Excluded rows are then simply invisible:
    missing from a list (reverse FK, M2M) and `null` on a forward FK or one-to-one (hence `| None`
    above).
    ```python
    @strawberry_django.type(User)
    class UserType:
        username: auto

        @classmethod
        def get_queryset(cls, queryset: QuerySet[User], info: Info) -> QuerySet[User]:
            user = get_current_user(info)  # the viewer only; widen to whoever they may see
            return queryset.filter(pk=user.pk) if user.is_authenticated else queryset.none()
    ```
    Lists are filtered by the library itself; a forward FK only by the **optimizer**, so the schema
    needs `DjangoOptimizerExtension`, and the parent must come from a queryset the library resolves.
    An instance your own code hands back — a resolver's `.first()`, a mutation payload, a plain-type
    DTO — skips the scope, and its `user` is the hidden account; that is why create/update return
    their row through the shared `reread` (step 3). Scoping the related Type also
    turns that relation's join into one prefetch query. It hides rows, not fields: every field of a
    row that passes is readable, so keep owner-only fields off a Type other callers can reach.
  - **Never write a resolver over a relation.** `def color(self) -> ColorType | None: return
    Color.objects.get(pk=self.color_id)`, or a filtered pick from a reverse relation
    (`self.fruits.filter(sold=False).first()`), is opaque to the optimizer: it runs one query per
    parent row (N+1), and an `annotate=` field selected beneath it raises because nothing annotated
    that instance. Declare the plain relation field instead. If clients need one particular child,
    expose the list and let them pick (the relation's own fields say which). Only when that really
    can't work, keep the resolver but give it
    `prefetch_related=lambda info: Prefetch("fruits", queryset=..., to_attr="_unsold")` and read
    `self._unsold`, so the filtered set is still fetched once for the page.
  - **Per-row computed values come from queryset annotations, not python resolvers.** A field whose
    value is derivable per row in SQL (an aggregate over related rows, a correlated subquery) is
    declared with an `annotate=` optimizer hint, computed once in the list query instead of N+1
    python-side:
    ```python
    # on ColorType: how many of the color's fruits were sold today
    sold_today: int = strawberry_django.field(
        annotate=lambda info: sold_today_count()  # callable → rebuilt per request
    )
    ```
    A bare expression works too; use the callable form when the expression depends on request time
    ("today"-relative windows) or the viewer. **Caveat:** the optimizer only annotates querysets it
    resolves — a mutation's returned instance doesn't carry the annotation. Create/update return
    their row through the shared `reread(obj, info)` (step 3), which loads it through the
    optimizer, so the annotation is there with no second copy of its expression. Delete payloads
    can't do this (the row is gone) — they select `id` only. Every other resolver or non-model type
    that hands out instances of the Type needs the same, or selecting the field raises. So before adding an
    `annotate=` field, check the next bullet — if the client can derive the value, don't add it.
  - **When the same filter and annotation are also needed off the Type** (a basket or checkout
    summary that lists these rows, a mutation's re-read), put them on the model's `QuerySet` as
    named methods (`available()`, `with_availability()`) and have `get_queryset` compose them,
    rather than duplicating an `annotate=` lambda. Keep filtering and annotating as separate methods, and
    apply the annotation for *every* caller or the field falls back to a query per row.
  - **Don't compute what the client can derive.** A boolean like `has_color`
    (`self.color_id is not None`) is not a field — expose the relation
    (`color: ColorType | None`) and let the client derive the flag from null-ness.
    The same goes for a flag over a *reverse* relation ("has an unsold fruit" on a color): if the
    child's own fields decide it (`sold` false, `soldAt` null), expose
    `fruits: list[FruitType]` with those fields and put the rule in one client helper —
    not an `annotate=` `Exists` field. Keep the rule server-side only as a **filter** (nested, as
    in the Filter bullet above), for queries over rows the client hasn't loaded (e.g. counting
    such colors across every page). When the client derives state from a field, make sure the
    field can't be set in any other state, and say so in the related Type's docstring and the spec.
    Custom `@strawberry.field` resolvers are reserved for values that are genuinely viewer-relative
    or otherwise inexpressible as a model field, relation, or annotation.
  - A related type that must resolve as a nested relation has to be a `strawberry_django.type` over
    the related model — a plain `@strawberry.type` built by hand can't. If a plain one already
    exists (e.g. constructed by a status resolver), convert it: keep model fields `auto`, keep any
    viewer-relative field as a resolver (with `select_related=[...]` hints), and have former
    construction sites return the model instance (`cast(TheType, obj)`) instead of building DTOs.
  - **One Type per model; audience-varying rows go in `get_queryset`.** When callers should see
    different *rows* (staff see sold fruit, others don't), branch on the caller inside one
    Type's `get_queryset` — do not add a second Type and a second query field. A second Type is for
    genuinely different *fields*, not different rows. State in the docs that a shared field's
    results vary by caller.
    ```python
    @classmethod
    def get_queryset(cls, queryset: QuerySet[<ModelName>], info: Info) -> QuerySet[<ModelName>]:
        user = get_current_user(info)
        if not user.is_authenticated:
            return queryset.none()     # or the public rows, if anonymous callers may see some
        if getattr(user, "is_staff", False):
            return queryset            # staff manage everything
        return queryset.filter(user=user)  # the scope; do NOT sort here
    ```
    Handle the anonymous caller **first**. `get_queryset` runs wherever the Type is reached, not
    only behind a gate — a nested relation under an ungated field, a public query — and
    `filter(user=AnonymousUser())` raises `TypeError` ("Field 'id' expected a number but got
    <AnonymousUser …>"), which the client sees as an unexpected error instead of nothing.
  - **Non-model result types** (a checkout summary, a "basket" that is a view over a user's rows,
    no table of its own) are legitimately a plain `@strawberry.type` over rows someone already
    read. Carry context in `strawberry.Private[...]` fields and cache the read in a private field so selecting
    two computed fields reads once. Neither the optimizer nor the nested Type's `get_queryset` /
    annotations run on that path, so annotate the rows yourself and hand the value across to the
    nested instance, or give the nested Type's annotated field a resolver that prefers the
    annotation and falls back to one lookup for that row. **If the grouping has an identity of its
    own** — a client-supplied id, a created-at, a lifecycle — give it a table and a real Type with
    `annotate=` counts instead: a hand-built aggregate DTO can't be filtered, ordered or paginated by
    the standard arguments, and the client ends up depending on a shape you invented.
- **Do not** add `select_related`/`prefetch_related` in `get_queryset` — the Django optimizer extension handles it.
- **Scope** with `get_current_user(info)` (`from strawberry_django.auth.utils import get_current_user`) —
  never read `info.context.request.user` directly.
- **Do not sort** in `get_queryset` or resolvers. Ordering is the client's choice: define the
  `<ModelName>Order` above and let callers pass the generated `ordering` argument.
- **Choice fields**: annotate `auto` (Types, Inputs and Filters alike). What that produces depends
  on how the model declares the field — check before promising clients an enum:
  - **`django-choices-field`** (`TextChoicesField(choices_enum=Status)`): always a GraphQL **enum**
    named after the Python choices class (`Status`), serialized by member *name*
    (`Status.RIPE = "ripe"` → `RIPE` on the wire). `status: auto` is all it takes — never a
    hand-written `@strawberry.enum` that mirrors the choices plus a `def status(self)` resolver
    converting to it. That's a second list of members to keep in step, for the same wire shape.
  - **Plain `CharField(choices=Status.choices)`**: a `String`, unless settings have
    `STRAWBERRY_DJANGO = {"GENERATE_ENUMS_FROM_CHOICES": True}` (default `False`). With it, an enum
    named from app + model + field (`FruitsFruitStatusEnum`), serialized by *value* (`ripe`). Don't
    turn the setting on while scaffolding one model — it changes the wire type of every existing
    choice field.
  - **Writing a choice enum back** (inputs). The two paths hand the resolver different objects.
    `django-choices-field` passes a member of your own `TextChoices` class — a `str` subclass, so
    the model field takes it as is. `GENERATE_ENUMS_FROM_CHOICES` passes a member of a generated
    **plain `Enum`**, which is not a `str`: assigned to the model unchanged, `CharField.to_python()`
    turns it into `str(member)`, and `full_clean()` rejects every value, valid ones included:
    ```
    mutation { createFruit(data: { name: "Apple", status: ripe }) { … } }
    → OperationInfo { kind: VALIDATION, field: "status",
                      message: "Value 'FruitsFruitStatusEnum.ripe' is not a valid choice." }
    ```
    (Without `full_clean()` the string `FruitsFruitStatusEnum.ripe` would be stored.) So
    `apply_input` assigns `value.value` for every enum — right for both paths, and what the
    library's own create/update resolvers do. Any other code that copies an input enum onto a
    model needs the same `.value`.

  Annotate `str` only to preserve an *existing* wire format that already ships plain values — never
  introduce a new `str`-annotated choice field on an enum-producing path.
  - **Enum name collisions** (`django-choices-field` only): since the enum is named after the choices
    class, not the model or field, and GraphQL type names are global, two unrelated models each
    declaring a `class Status(models.TextChoices)` collide on one GraphQL enum type — same failure
    mode as "Union name collisions" in step 3, and same fix: give each its own distinct class name
    (`FruitStatus`, `ColorStatus`, ...). Check for an existing choices class with the same name
    before adding a new one. (The `GENERATE_ENUMS_FROM_CHOICES` names are per field and don't collide.)
  - Once introspection is disabled outside dev (see `references/schema-and-limits.md`), a client
    can no longer discover an enum's members from the schema — if a product feature needs the
    caller to know the valid values (e.g. a picker), expose them through an explicit query field.
- Import a related type from another module **directly** (`from colors.types import
  ColorType`) unless that import would cycle — i.e. the other module (or anything it imports)
  imports this one back. Check before choosing: `grep -n "^from\|^import" <other module>`. Only
  for a real cycle, reference it **lazily**: wrap the
  type in `typing.Annotated` with a *stringified* type name and `strawberry.lazy(<module path>)`:
  ```python
  from typing import Annotated
  import strawberry

  color: Annotated["ColorType", strawberry.lazy("colors.types")]
  # nullable relation:
  color: Annotated["ColorType", strawberry.lazy("colors.types")] | None
  ```
  The string `"ColorType"` is the type's class name; `strawberry.lazy("colors.types")` is the
  **absolute** import path of the module that defines it (Strawberry imports it only at schema-build
  time). Do **not** also import the related type at the top of the file for these fields — that's
  the cycle you're avoiding. A lazy ref where no cycle exists only hides the dependency. Types in
  the *same* module are referenced directly.

### 3. Generate Inputs + Mutations (place next to other mutations in the repo)

- `@strawberry_django.input(model=<ModelName>)` `class <ModelName>CreateInput` — create fields, all
  `auto`. (Named `…CreateInput`, not `…Input`, to distinguish it from the update input.)
- `@strawberry_django.input(model=<ModelName>, partial=True)` `class <ModelName>UpdateInput` — the
  same fields, `partial=True` so an edit sends only what changed. A **create-only** field (e.g. a
  `stock` fixed at creation) is on the create input and simply absent from the update input.
- **Every input field is `auto` — no wire-shape overrides.** A FK is a relation input,
  `color: { set: "<uuid>" }`; do **not** replace it with `color_id: ID`. `full_clean()` rejects an
  id that names no row (and a string that isn't a UUID) and reports it under the *relation* name
  (`color`), so no private `_resolve_color()` / `_color_exists()` lookup helper is needed. Choice
  fields stay enums.
- **M2M inputs**: `auto` on a ManyToManyField generates `ManyToManyInput` (`{ set, add, remove }`)
  in both inputs. It gives the client two ways to change the relation:
  - **The simple way — `set`**: `{ set: [id, …] }` is the complete list; whatever isn't in it is
    unlinked, and `{ set: [] }` unlinks everything. Enough for a form that edits the whole list.
  - **Two extra options — `add` / `remove`**: `{ add: [id, …] }` and `{ remove: [id, …] }` (alone
    or together) link or unlink only the rows they name and leave the rest, so a client toggling
    one tag doesn't have to send — or race another writer over — the whole list.

  One request uses one way: `set` together with `add`/`remove` is refused. An omitted field is
  `UNSET` on the create input as well as a `partial=True` update, and changes nothing.
  `apply_input` skips the field; after `save()`, apply it with the shared `sync_m2m`, which
  honors both ways: `sync_m2m(fruit.tags, data.tags, "tags")` — the last argument is the field
  its errors are keyed under (`field: "tags"` in the `OperationInfo`). `full_clean()` never sees
  M2M ids, so `sync_m2m` looks every id up first and answers one naming no row with "Not found."
  on that field, writing nothing. Never `fruit.tags.set(data.tags.set or [])` — it turns an
  `{ add: … }` request into "unlink everything", raises `AttributeError` when the field was
  omitted, and lets an unknown id surface as an unconverted `IntegrityError`.
- **Reverse-FK inputs**: a reverse FK on an input (`fruits: auto` on `Color`) generates
  `ManyToOneInput` of the same shape and goes through `sync_m2m` the same way. It re-parents the
  rows the caller names, so it is authorization-sensitive: pass the rows the caller may change as
  `rows` (their visible set, or the narrower write scope — step 3) — `sync_m2m(color.fruits, data.fruits, "fruits", rows=visible_fruits)` with
  `visible_fruits = FruitType.get_queryset(Fruit.objects.all(), info)` — and a row they can't
  see gets the same "Not found." as one that doesn't exist
  (`references/authorization-and-leaks.md`). `remove`, and a `set` that drops rows, need the FK
  to be nullable. Pass `rows` for an M2M too when its related rows aren't all the caller's to
  link (per-user tags).
- **`handle_django_errors`** — check `STRAWBERRY_DJANGO = {"MUTATIONS_DEFAULT_HANDLE_ERRORS":
  True}` is set in settings before assuming either behavior (see `references/schema-and-limits.md`).
  With it on, `ValidationError` / `PermissionDenied` / `ObjectDoesNotExist` raised inside a resolver
  become a typed `<ModelName>Type | OperationInfo` union instead of a top-level error — you write no
  catching code. A malformed id is the exception: on an integer pk `.get(pk="junk")` raises
  `ValueError`, which is *not* converted — that is why lookups go through `get_by_id`. If the repo hasn't adopted it, return `<ModelName>Type` directly
  and let Django exceptions surface as GraphQL errors (use `GraphQLError` only for failures that
  aren't model exceptions).
- **Gate each mutation** with the shared extension from `permissions.py`, on the mutation itself:
  a denied caller then gets a typed `OperationInfo { kind: PERMISSION, code: "UNAUTHENTICATED" }`
  and the resolver never runs, so authorization precedes validation and existence checks. (Gating only the namespace
  wrapper field yields a top-level error and a null namespace instead — use that only if the repo
  already does.)

```python
from typing import cast

import strawberry
import strawberry_django
from django.core.exceptions import ValidationError
from django.db.models import ProtectedError

from <pkg>.graphql import Info, apply_input, get_by_id, reread
from <pkg>.permissions import authenticated_required  # or staff_required


@strawberry.type
class FruitMutations:
    # Python method names are `create_fruit` / `update_fruit` / `delete_fruit`, never bare
    # `create` / `update` — see "Union name collisions". Exposed as `createFruit` etc.
    @strawberry_django.mutation(extensions=[authenticated_required])
    def create_fruit(self, info: Info, data: FruitCreateInput) -> FruitType:
        # Construct with only what the input can't carry, e.g. the owner:
        # Fruit(user=get_current_user(info)).
        fruit = Fruit()
        apply_input(data, fruit)
        fruit.full_clean()
        fruit.save()
        return cast(FruitType, reread(fruit, info))  # as stored, through the optimizer

    @strawberry_django.mutation(extensions=[authenticated_required])
    def update_fruit(self, info: Info, id: strawberry.ID, data: FruitUpdateInput) -> FruitType:
        # The rows the caller may change — here the same as the rows they see; when they can
        # see rows they may not change, the write scope below. Not-found / not-yours /
        # malformed id are one indistinguishable OperationInfo.
        fruit = get_by_id(FruitType.get_queryset(Fruit.objects.all(), info), id)
        fields = apply_input(data, fruit)
        fruit.full_clean()
        # Only what was sent, plus the auto_now version (drop it if the model has none) — a
        # whole-row save() would revert a column someone else changed after the read
        # (references/concurrency.md).
        fruit.save(update_fields=[*fields, "updated_at"])
        return cast(FruitType, reread(fruit, info))

    @strawberry_django.mutation(extensions=[authenticated_required])
    def delete_fruit(self, info: Info, id: strawberry.ID) -> FruitType:
        fruit = get_by_id(FruitType.get_queryset(Fruit.objects.all(), info), id)
        pk = fruit.pk
        try:
            fruit.delete()
        except ProtectedError as exc:  # only when a FK to this model is PROTECT/RESTRICT
            raise ValidationError("This fruit is in use and cannot be deleted.") from exc
        fruit.pk = pk  # Django nils it post-delete; restore for the response
        return cast(FruitType, fruit)
```

- **Look the object up in the rows the caller may *change*.** Usually those are exactly the rows
  they may see: reuse the Type's `get_queryset`
  (`get_by_id(<Type>.get_queryset(Model.objects.all(), info), id)`; a Type with no override:
  `get_by_id(Model.objects.all(), id)`) — never a second hand-rolled ownership check that drifts
  from the read path. **When callers can see rows they may not change** (teammates' fruits on a
  shared board, published rows everyone reads), the read scope would let them edit or delete
  anything they can merely see. Define the write scope once, as a narrowing of the read scope, and
  use it for update, delete, and a reverse-FK input that re-parents rows (`sync_m2m(…, rows=…)`):
  ```python
  def writable_fruits(info: Info) -> QuerySet[Fruit]:
      """Visible fruits the caller may change: their own. Teammates' are read-only."""
      return FruitType.get_queryset(Fruit.objects.all(), info).filter(user=get_current_user(info))
  ```
  Narrowing keeps "can't change what you can't see" true by construction. A visible but read-only
  row then answers like a missing one; a specific "You can't edit this fruit." (`PermissionDenied`)
  would leak nothing, since the caller can see the row, so use it if the UI needs it. Test the
  refusal with a caller who **can see** the row: a stranger is refused by both scopes, so only a
  viewer catches a write that looks the row up in the read scope (`references/testing.md`).
- **`ProtectedError` / `RestrictedError` are not converted automatically.** If any FK to the model
  uses `on_delete=PROTECT`/`RESTRICT`, catch it around `delete()` and raise `ValidationError` as
  above; otherwise an in-use delete surfaces as a server error. Plain delete needs no explicit
  transaction (Django's collector already runs it atomically).
- **Union name collisions**: `handle_django_errors=True` auto-generates an error-union type named
  from the **Python method name** alone — `create` → `"CreatePayload"` — not from the mutation's
  namespace/class. GraphQL type names are global, so a second, unrelated mutation method also
  named `create` (in another model's `<OtherModel>Mutations`) collides on that union name;
  Strawberry silently keeps only the first one registered, and the second's fragments fail schema
  validation ("Fragment cannot be spread here..."). Hence `create_<model_name>` /
  `update_<model_name>` / `delete_<model_name>`. Add `name="create"` only if this project's exposed
  field must stay `create`/`update`.
- The input parameter is named `data` — Strawberry-Django's default argument name.
- **A mutation that writes multiple rows is atomic** — the instance `save()` plus its M2M sync, or
  any multi-model change — so the writes land together or not at all. Decorate the whole resolver,
  placing `@transaction.atomic` *under* the mutation decorator:
  ```python
  @strawberry_django.mutation(extensions=[authenticated_required])
  @transaction.atomic
  def create_fruit(self, info: Info, data: FruitCreateInput) -> FruitType:
      fruit = Fruit()
      apply_input(data, fruit)
      fruit.full_clean()
      fruit.save()
      sync_m2m(fruit.tags, data.tags, "tags")  # after save(): linking needs the row's pk
      return cast(FruitType, reread(fruit, info))
  ```
  A raised `ValidationError` still rolls back first and then becomes `OperationInfo`.
- **Create and update return their row through `reread(obj, info)`**, never the in-memory instance.
  The instance is what the resolver built, and the optimizer never sees it, so three things go
  wrong in the payload:
  - a selected `annotate=` field raises (`'Fruit' object has no attribute 'tag_count'`) and the
    payload is lost;
  - a nested Type's `get_queryset` is skipped — `user { username }` returns an account the caller
    may not see, where a query would answer `null`;
  - a value the database shapes on write shows what was sent: `amount: "5000"` is stored as
    `5000.00`, the payload says `"5000"`, and a client comparing it with a later read sees a
    change that never happened (the same for a column a trigger sets).

  `reread` loads the saved row through the optimizer for the payload's own selection — one query,
  plus a prefetch per selected to-many relation. Django already fetches `db_default` and
  `GeneratedField` values back on save on a backend with `RETURNING` (checked on Django 6 with
  SQLite); `reread` covers the others too. Delete can't re-read (the row is gone), so its payload
  selects `id` only.
- **Update writes only the fields the input set** (`save(update_fields=...)`, template above) — never
  the whole row. For a record with several writers (users sharing it, a background job, offline
  clients) add the optional `updatedAt` precondition so a stale edit is refused instead of
  overwriting; both are in `references/concurrency.md`. A delete has no `data`, so it carries no version.
- **Authorization-sensitive relation inputs and existence oracles** — a relation the caller must be
  *authorized* for (e.g. "must be *your* team"), and any failure branch whose message could reveal
  what exists — are covered in `references/authorization-and-leaks.md`. Read it when either applies.
- **A create that must be safe to retry** (a client queueing writes offline) takes the client's
  own UUID as an optional `id` and returns the existing row on a retry — the resolver, the race
  between two retries, the pk default it relies on and what it exposes are in
  `references/authorization-and-leaks.md`.
- Validation lives **on the model** (`validators=`, a DB `CheckConstraint`, `Model.clean()`); the
  mutation just calls `obj.full_clean()`.

### 4. Wire into the schema (`schema.py` / wherever `Query` and `Mutation` are defined)

- **Imports** — follow the existing grouping and the repo's import site (some repos re-export each
  app's Type and Mutations through `<app>/schema.py` so the root imports once per app).
- **Query** — declare fields flat on the root `Query`; a list of model rows is `OffsetPaginated`.
  Gate with the shared extension; a denied caller gets an empty page (`totalCount: 0`,
  `results: []`) from a paginated field, an empty list from a plain list, `null` from an optional
  field, and a top-level error from anything else:
  ```python
  from strawberry_django.pagination import OffsetPaginated

  from <pkg>.permissions import authenticated_required

  <model_name_plural>: OffsetPaginated[<ModelName>Type] = strawberry_django.offset_paginated(
      extensions=[authenticated_required],
  )
  ```
  Clients select `{ totalCount results { … } }` and pass `pagination: { offset, limit }` alongside
  the usual `ordering` / `filters`. `OffsetPaginated` always applies the library's page limits
  (`STRAWBERRY_DJANGO` settings: `PAGINATION_DEFAULT_LIMIT` when the client sends no `pagination`
  or `null`, `PAGINATION_MAX_LIMIT` as the ceiling for any `limit`; 100 each by default) — change
  them there. Don't cap or sort inside `get_queryset` to imitate a page size.
- **A plain list has no row limit.** `list[<ModelName>Type] = strawberry_django.field(...)` returns
  the whole queryset when the client omits `pagination` or sends `null` — even with
  `pagination=True`, only a request that sends a `pagination` input is capped — and the
  token/alias/depth limiters count the query, not the rows. Use one only for a set that is small
  by construction (a fixed reference table), and say so in the field's description. A nested
  to-many relation is the same: `fruits: list[FruitType]` on `ColorType` returns every child of
  every color on the page. Make it `fruits: OffsetPaginated[FruitType] =
  strawberry_django.offset_paginated()` (each parent's page is capped like a top-level one), or
  leave it off and let clients page the top-level list filtered by the parent.
- **Errors in a query** — `handle_django_errors` covers mutations only. A Django `ValidationError`
  raised in a query resolver is not converted: unmasked, a list- or dict-form one reaches the client
  as `str(exc)` (`"['…']"`, `"{'field': […]}"`); if the schema masks unexpected errors (`MaskErrors`, `references/schema-and-limits.md`) it becomes
  "Unexpected error." (the masker lets only `GraphQLError`, `PermissionDenied` and
  `ObjectDoesNotExist` through — a scoped lookup's not-found reads the same for a missing row and
  another user's). Raise `GraphQLError("…")` for a message the client should read, and return
  `None` / `[]` where the design says "nothing" (the single-object query below).
- **Single-object query** — a list field isn't enough for `<model_name>(id:)`. Return a
  **QuerySet**, not an instance: the library runs the field's own pipeline on it — the Type's
  `get_queryset`, the optimizer (joins, prefetches, `annotate=` fields, nested Types' scopes) and
  `.first()`. An unknown, hidden *or malformed* id answers `null` (`.filter(pk="junk")` raises
  `ValueError` on an integer pk and `ValidationError` on a UUID pk; neither is converted in a query):
  ```python
  def resolve_<model_name>(id: strawberry.ID) -> <ModelName>Type | None:
      try:
          found = <ModelName>.objects.filter(pk=id)
      except (ValueError, ValidationError):  # malformed id -> no such row
          found = <ModelName>.objects.none()
      return cast("<ModelName>Type | None", found)  # a QuerySet: the library takes .first()

  <model_name>: <ModelName>Type | None = strawberry_django.field(resolver=resolve_<model_name>)
  ```
  Don't call `.first()` yourself: the bare instance it returns skips the optimizer and every
  nested scope. Each relation becomes its own query, a selected `annotate=` field raises
  (`'Fruit' object has no attribute 'tag_count'`) and nulls the object, and a nested Type's
  `get_queryset` is skipped — a hidden owner's `user` is returned.
- **Mutation** (nested one level) — the namespace field is a plain wrapper; the gate is on each
  mutation (step 3):
  ```python
  @strawberry.field
  def <model_name>(self) -> <ModelName>Mutations:
      return <ModelName>Mutations()
  ```
- There is a single module-level `schema` built from `Query` and `Mutation` with the optimizer, the
  three limiters, and introspection disabled whenever `DEBUG` is off (decided per request). It
  must already exist; if the repo has none, or its extensions are missing a guard, read
  `references/schema-and-limits.md`. Extensions are passed as classes / factory callables, never
  built instances.

### 5. Verify

```bash
python manage.py check
```

Confirm the schema builds (no `UnresolvedFieldTypeError` from lazy paths, no duplicate type or
directive names). Then run the repo's type checker and match its idiom rather than assuming one:

- The templates return `cast(<ModelName>Type, obj)` because the model instance isn't the Type to
  the checker; use `cast(User, get_current_user(info))` where the concrete user model is needed —
  `get_current_user` is typed `AbstractBaseUser | AnonymousUser`. Keep a cast only where the
  checker needs it.
- **mypy**: `strawberry_django.mutation` is an untyped decorator → `# type: ignore[untyped-decorator]`
  under strict settings.
- **pyright**: `strawberry_django.type(..., filters=, ordering=)` needs
  `# pyright: ignore[reportArgumentType]` (upstream: strawberry_django's `type()` shadows the
  `type` builtin it annotates those with), and `logout: bool = auth.logout()` needs
  `# pyright: ignore[reportAssignmentType]`.
- Add an ignore only for an error the checker actually reports, with its specific rule code.

### 6. Offer next steps

Suggest adding tests for the new queries/mutations (conventions and response shapes:
`references/testing.md` — defer to a repo-local GraphQL testing skill or `docs/graphql.md` if one
exists), and `makemigrations` only if the model itself changed (this skill does not change models).
If the repo documents its API (`docs/graphql.md`, an OpenSpec/spec tree), carry any behavior you
introduced into it in the same pass.

---

## Extending an already-scaffolded model

Adding a field to a model that already has a Type and Inputs touches every layer that mentions the
model's fields — a change made in only one of them ships a field clients can read but not write, or
the reverse. Checklist (edit, don't regenerate):

1. **Type** — add `<field>: strawberry.auto` (or the related type for a relation).
2. **Filter / Order** — add it if clients should filter or sort by it; say so if you leave it out.
3. **`<ModelName>CreateInput`** and **`<ModelName>UpdateInput`** — add it to both; a create-only
   field goes on the create input alone.
4. **Model validity** — this skill doesn't edit the model, so check the field is consistent with
   what the inputs advertise. `null=True` alone is **not** enough on a string-like field
   (`URLField`, `CharField`, `EmailField`): without `blank=True`, `full_clean()` rejects `None` with
   "This field cannot be blank.", so the input looks optional but the mutation demands it. Tell the
   user the model needs `null=True, blank=True` (plus a migration) rather than editing it yourself.
5. Docs/spec and tests that enumerate the model's fields.

---

## Critical rules

- `auto` annotations wherever possible — let Strawberry infer field types from the model. This
  includes choice fields (→ enum with `django-choices-field` or `GENERATE_ENUMS_FROM_CHOICES`, else
  `String`) and FKs (→ `{ set: id }` relation input) in filters and inputs.
  The only exception is preserving an existing plain-string wire format on an already-deployed field.
- Filters: no custom `filter_field` methods for what a declared field can express — a relation
  scope filter is `user: UserFilter | None = strawberry.UNSET`, not a resolver, and a condition
  on children is a nested reverse-relation filter (`fruits: FruitFilter`), not a hand-built
  `Exists`.
- Types: no denormalized related-scalar resolvers (`owner_username`-style) and no derived booleans
  (`has_color`-style, or "has an unsold child" over a reverse relation) — expose the relation and
  let clients select or derive.
- **No resolvers over relations.** Forward FK → `color: ColorType | None`; reverse FK →
  `fruits: list[FruitType]` (`OffsetPaginated[FruitType]` when a parent can have many). A resolver that re-reads or filters a relation is N+1 and breaks
  annotated fields beneath it.
- **Lists are paginated.** A list query field is `OffsetPaginated`; a plain `list[...]` — top-level
  or a nested to-many relation — returns every row unless the client sends `pagination`, so keep
  it for sets that are small by construction.
- **One Type per model.** Different audiences see different rows through `get_queryset`, not
  through a second Type or query field.
- Per-row computed values: `strawberry_django.field(annotate=...)` (callable for request-time
  expressions), or named `QuerySet` methods composed in `get_queryset` when the same annotation is
  reused off the Type — never a python resolver looping per row. Add one only for a value the
  client can't derive from fields it already selects.
- **Create/update payloads go through `reread(obj, info)`** — the saved row as stored, loaded by
  the optimizer for the payload's selection (annotations, nested scopes, stored decimals). Not
  `refresh_from_db()` (it skips M2M and the optimizer) and not the in-memory instance. Delete
  payloads select `id` only.
- Nullable relations, and any relation to a Type with a `get_queryset` (a non-null FK included),
  are annotated `RelatedType | None`; the related Type filters its own rows. M2M and reverse FKs: `auto` in both
  inputs, applied after the save with `sync_m2m` — `set` replaces the list, `add`/`remove` change
  only the rows named, an unknown id is "Not found." on the field; pass the caller's visible set
  as `rows` when linking must be authorized. Never `.set(data.<field>.set or [])`.
- Create and update apply the input through the shared `apply_input` — never hand-copy fields.
- **Updates write only the fields sent** (`save(update_fields=[*fields, "updated_at"])`), never the
  whole row; add the optional `updatedAt` precondition when a record has several writers
  (`references/concurrency.md`).
- Don't write private lookup/validation helpers (`_resolve_color`, `_color_exists`) — `full_clean()`
  already validates that a FK names a real row. The exception is an authorization check on a
  relation (`references/authorization-and-leaks.md`).
- **No existence oracles.** For every failure branch, ask whether telling the cases apart leaks
  something the caller couldn't otherwise learn; if so, one identical message, pinned by a
  parametrized identical-literal test.
- Get the user via `get_current_user(info)` (`strawberry_django.auth.utils`), never
  `info.context.request.user`.
- **Shared permission extensions**: import `staff_required` / `authenticated_required` from one
  module rather than configuring `IsAuthenticated()` / `IsStaff()` per field. Gate each mutation,
  not just the namespace, so denial is a typed `OperationInfo`. Every not-signed-in refusal carries
  the code `UNAUTHENTICATED`; clients match the code, never the message.
- **One read rule, one write rule**: update/delete look the object up with `get_by_id` over the
  rows the caller may change — the Type's `get_queryset` (when it has one) if that is everything
  they see, else one write scope that narrows it — not a re-implemented ownership check, not a
  per-model helper. A bare `.get(pk=id)` turns a malformed integer id into a top-level error.
- **Queries don't convert Django exceptions** — a deliberate, readable failure in a query is a
  `GraphQLError`; a `ValidationError` there is masked or can leak `"['…']"`. Under the masking
  recipe `PermissionDenied` and a user-scoped `ObjectDoesNotExist` stay readable. Mutations rely on
  `handle_django_errors` instead.
- Check whether the project sets `MUTATIONS_DEFAULT_HANDLE_ERRORS` before assuming either behavior.
  - **On**: create/update/delete return `<ModelName>Type | OperationInfo`; name the methods
    `create_/update_/delete_<model_name>`; `full_clean()`/`ValidationError`/`ObjectDoesNotExist`
    are converted automatically (a malformed integer id only via `get_by_id`). Delete gets no opt-out — a `bool`
    can't union with `OperationInfo` — but `ProtectedError`/`RestrictedError` must be mapped to
    `ValidationError` by hand. Opt a mutation out with `handle_django_errors=False` only for a
    specific reason (e.g. a deliberately never-raising resolver).
  - **Off**: return `<ModelName>Type` directly; Django exceptions surface as GraphQL errors.
- Validation belongs on the **model**; the mutation only calls `obj.full_clean()`. A nullable string
  field needs `blank=True` as well as `null=True`.
- Ordering is client-requested via an `order_type` + the `ordering` argument — never sort in a
  resolver or `get_queryset`.
- Cross-module related types are imported directly; only an import that would cycle uses a lazy
  ref, `Annotated["TypeName", strawberry.lazy("app.types.module")]` (append `| None` for
  nullable). `strawberry.lazy()` paths are always absolute module paths.
- Prune comments that restate the line below them; keep ones carrying a non-obvious *why*.
- `id: auto` exposes whatever PK the model uses (a UUID pk is a `UUID` scalar on the wire, not `ID`).
- This skill generates GraphQL artifacts only — it does not edit models or create migrations.
