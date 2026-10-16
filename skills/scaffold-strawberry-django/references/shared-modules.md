# Shared modules: `graphql.py` and `permissions.py`

Every resolver in the scaffold imports from these two small modules. `SKILL.md` lists what
each helper does; copy the code below into the project's shared package (e.g.
`<pkg>/graphql.py`, `<pkg>/permissions.py`) for whatever is missing.

## `graphql.py` — `Info`, `apply_input`, `sync_m2m`, `get_by_id` and `reread`

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

## `permissions.py` — shared permission extensions

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
