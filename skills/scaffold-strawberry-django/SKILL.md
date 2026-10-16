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

Long reference material lives next to this file; each step says when to read which one:

| File | Read when |
| --- | --- |
| `references/shared-modules.md` | a shared helper (`apply_input`, `sync_m2m`, `get_by_id`, `reread`, the permission extensions) doesn't exist yet — it holds their code |
| `references/types-and-filters.md` | writing a Type or Filter beyond plain scalar fields: relations and their visibility, computed fields, choice fields, lazy imports |
| `references/mutations.md` | writing Inputs and Mutations beyond the template: M2M / reverse-FK inputs, write scope, delete, atomicity, `reread` |
| `references/schema-and-limits.md` | the `schema` object, extensions, limiter settings, error masking and logging |
| `references/authorization-and-leaks.md` | a relation input the caller must be authorized for, existence oracles, retry-safe create, idempotent delete |
| `references/concurrency.md` | update writes, and the optional `updatedAt` check for shared rows |
| `references/testing.md` | writing tests |

## Shared modules (find or create)

Every resolver below imports from two tiny shared modules, in the project's shared package
(e.g. `<pkg>/graphql.py`, `<pkg>/permissions.py`). Look for them first:

```bash
grep -rn "def apply_input\|def sync_m2m\|def get_by_id\|def reread\|class IsAuthenticated\|IsStaff()" .
```

Create only what is missing, from the code in `references/shared-modules.md`. What each does:

- `Info` — `strawberry.Info[StrawberryDjangoContext, object]`. Annotate every resolver with it: a
  bare `strawberry.Info` defaults both parameters to `Any`, so `info.context.…` goes unchecked.
- `apply_input(data, instance) -> list[str]` — assigns every *set* field of an input onto the
  instance (`UNSET` skipped; a `{ set: id }` FK goes to `<field>_id`, so `full_clean()` rejects an
  id naming no row; an enum by `.value`; `field_name=` renames honoured) and returns the model
  field names it set, for `save(update_fields=…)`. M2M and reverse-FK inputs are skipped. One call
  serves create and update — never hand-copy `data.<field>` onto the instance, which yields two
  field lists that drift apart. An input field with no model field behind it raises
  `FieldDoesNotExist`.
- `sync_m2m(manager, value, field, rows=None)` — applies an M2M / reverse-FK input (`set`, or
  `add` / `remove`) after `save()`. It looks every id up first: one naming no row is "Not found."
  keyed `field`, and nothing is written. Pass `rows` (the caller's visible set) when linking must
  be authorized.
- `get_by_id(queryset, id)` — `queryset.get(pk=id)`, with a malformed id reported as not-found
  instead of a top-level `ValueError`. Update and delete look their object up with it.
- `reread(instance, info)` — the saved row as stored, loaded through the optimizer for the
  payload's selection. Create and update return it.
- `authenticated_required` / `staff_required` — the shared permission extensions. Import them
  instead of constructing `IsAuthenticated()` / `IsStaff()` per field, so the denial message is
  configured once. Every not-signed-in refusal carries the code `UNAUTHENTICATED`: clients match
  `errors[].extensions.code` / `messages[].code`, never the message text. Leave `fail_silently` at
  its default `True`.

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

Order within the file: **Filter first, then Type** (the Type references the Filter). Read
`references/types-and-filters.md` once the model has a relation, a computed value or a choice
field — nearly every model does.

- `@strawberry_django.filter_type(<ModelName>, lookups=True)` — `id: auto`; scalar fields `auto`
  (choice fields too — they become enum lookups; never hand-annotate
  `str | None = strawberry.UNSET`); a FK as a nested relation filter,
  `color: ColorFilter | None = strawberry.UNSET` (define a minimal filter for the related model if
  none exists), and a reverse FK the same way on the *parent's* filter
  (`fruits: FruitFilter | None = strawberry.UNSET`). That covers "any of several ids" and "has a
  child matching…", so there is no top-level `colors: [ID]` argument and no custom `filter_field`
  / hand-built `Exists`. Same-row semantics, `DISTINCT` and the rare `filter_field` are in the
  reference.
- `@strawberry_django.order_type(<ModelName>)` `class <ModelName>Order` — the fields the client may
  sort by, each annotated `Ordering` (`from strawberry_django.ordering import Ordering`).
- `@strawberry_django.type(model=<ModelName>, filters=<ModelName>Filter, ordering=<ModelName>Order)`:
  - Use `auto` for field annotations wherever possible (let Strawberry infer from the model).
  - A forward relation is the related type: `color: ColorType | None`. Annotate `| None` if the
    relation is nullable **or the related Type has a `get_queryset`** — even over a non-null FK. A
    related row the viewer may not see comes back `null`, and on a non-null field that `null` nulls
    the parent and, through a non-null list, the whole response. Plain `RelatedType` only when the
    target Type has no scope. A reverse FK is `fruits: list[FruitType]`, or
    `fruits: OffsetPaginated[FruitType] = strawberry_django.offset_paginated()` when a parent can
    have many children (a plain list has no row limit — step 4).
  - **A Type that other rows point at filters its own rows**: give the related Type its own
    `get_queryset` with *its* visibility rule; don't rely on the parent's scope, don't filter
    relations in a resolver. The schema needs `DjangoOptimizerExtension` for a forward FK's scope
    to apply.
  - **Never write a resolver over a relation**, and never denormalize a related scalar into one
    (`owner_username`) — it is N+1 and breaks `annotate=` fields beneath it. Declare the plain
    relation and let clients select `user { username }`.
  - **Per-row computed values** come from `strawberry_django.field(annotate=lambda info: …)`, or
    from named `QuerySet` methods composed in `get_queryset` when the same annotation is needed
    off the Type — not from python resolvers. Add one only for a value the client can't derive
    from fields it already selects (`has_color` is not a field: the client reads null-ness).
  - **One Type per model; audience-varying rows go in `get_queryset`** — a second Type is for
    different *fields*, not different rows. Handle the anonymous caller first:
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
  - A non-model result type (a checkout summary) is a plain `@strawberry.type` over rows someone
    already read; the optimizer and nested scopes don't run on that path.
- **Do not** add `select_related`/`prefetch_related` in `get_queryset` — the Django optimizer extension handles it.
- **Scope** with `get_current_user(info)` (`from strawberry_django.auth.utils import get_current_user`) —
  never read `info.context.request.user` directly.
- **Do not sort** in `get_queryset` or resolvers. Ordering is the client's choice: define the
  `<ModelName>Order` above and let callers pass the generated `ordering` argument.
- **Choice fields**: annotate `auto` (Types, Inputs and Filters alike) — never a hand-written enum
  mirroring the choices. What it produces depends on the model, so check before promising clients
  an enum: `django-choices-field` gives a GraphQL enum named after the choices class (serialized
  by member *name*); a plain `CharField(choices=…)` gives a `String` unless
  `STRAWBERRY_DJANGO = {"GENERATE_ENUMS_FROM_CHOICES": True}` (default off — don't turn it on
  while scaffolding one model, it changes every existing choice field's wire type). Give each
  `django-choices-field` choices class a project-unique name (`FruitStatus`, not `Status`): equal
  names collide on one enum. `apply_input` writes enums back by `.value`; any other code copying
  an input enum onto a model needs the same.
- Import a related type from another module **directly** (`from colors.types import ColorType`);
  use `Annotated["ColorType", strawberry.lazy("colors.types")]` (absolute module path) only for a
  real import cycle — check with `grep -n "^from\|^import" <other module>` first.

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
- **M2M and reverse-FK inputs**: `auto` generates `ManyToManyInput` / `ManyToOneInput`
  (`{ set, add, remove }`) in both inputs. `apply_input` skips them; after `save()` apply them with
  `sync_m2m(fruit.tags, data.tags, "tags")` (the last argument is the field its errors are keyed
  under). `set` is the complete list; `add` / `remove` change only the rows named; both ways in one
  request is refused. Never `fruit.tags.set(data.tags.set or [])`. A reverse-FK input re-parents
  rows, so pass `rows=` — what the caller may change (`references/mutations.md`).
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
    # `create` / `update` — see references/mutations.md. Exposed as `createFruit` etc.
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
        # see rows they may not change, a narrower write scope
        # (references/mutations.md). Not-found / not-yours / malformed id are one
        # indistinguishable OperationInfo.
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

- **Look the object up in the rows the caller may *change*.** Usually those are the rows they may
  see: `get_by_id(<Type>.get_queryset(Model.objects.all(), info), id)` (a Type with no override:
  `get_by_id(Model.objects.all(), id)`) — never a second hand-rolled ownership check. When callers
  can see rows they may not change (teammates' rows, published rows), define one write scope as a
  narrowing of the read scope and use it for update, delete and re-parenting inputs
  (`references/mutations.md`).
- **Delete returns `<ModelName>Type`**, not a `bool` (a `bool` can't join the `OperationInfo`
  union); restore the pk after `delete()` as in the template. If any FK to the model is
  `PROTECT`/`RESTRICT`, catch `ProtectedError` around `delete()` as shown — it isn't converted
  automatically.
- **Method names are `create_<model_name>` / `update_<model_name>` / `delete_<model_name>`**, never
  bare `create` / `update`: with `handle_django_errors` the error union is named from the method
  name alone, and a second `create` collides on it (`references/mutations.md`).
- The input parameter is named `data` — Strawberry-Django's default argument name.
- **A mutation that writes multiple rows is atomic** (the `save()` plus its M2M sync, or any
  multi-model change): `@transaction.atomic` *under* the mutation decorator
  (`references/mutations.md` has the example).
- **Create and update return their row through `reread(obj, info)`**, never the in-memory
  instance: the optimizer never sees that instance, so a selected `annotate=` field raises, a
  nested Type's `get_queryset` is skipped (a hidden `user` is returned), and a value the database
  shapes on write shows what was sent. Delete can't re-read (the row is gone): its payload selects
  `id` only.
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

The checklist to verify a finished scaffold against; each rule is explained in the step or
reference named.

- **`auto` annotations** wherever possible, in Types, Filters and Inputs alike: scalars, choice
  fields (enum with `django-choices-field` or `GENERATE_ENUMS_FROM_CHOICES`, else `String`) and
  FKs (`{ set: id }` relation input). The only exception is preserving an existing plain-string
  wire format on an already-deployed field.
- **Filters**: a nested relation filter (`user: UserFilter | None`, `fruits: FruitFilter`), not a
  custom `filter_field` or a hand-built `Exists`, for anything a declared field can express.
- **Types**: no resolvers over relations, no denormalized related scalars (`owner_username`), no
  derived booleans (`has_color`, "has an unsold child") — expose the relation and let clients
  select or derive. Per-row computed values come from `annotate=` or named `QuerySet` methods,
  never a python resolver looping per row, and only when the client can't derive them.
- **Relations**: a nullable relation, and any relation to a Type with a `get_queryset` (a non-null
  FK included), is `RelatedType | None`; the related Type filters its own rows.
- **One Type per model.** Different audiences see different rows through `get_queryset`, not
  through a second Type or query field.
- **Lists are paginated.** A list query field is `OffsetPaginated`; a plain `list[...]` —
  top-level or a nested to-many relation — returns every row unless the client sends
  `pagination`, so keep it for sets that are small by construction.
- **Ordering is the client's**: an `order_type` + the `ordering` argument; never sort in a
  resolver or `get_queryset`. No `select_related` / `prefetch_related` in `get_queryset`.
- **Writes**: create and update apply the input with `apply_input`, to-many inputs with `sync_m2m`
  after the save, and return the row through `reread(obj, info)` (not `refresh_from_db()`, not the
  in-memory instance; delete payloads select `id` only). Updates write only the fields sent —
  `save(update_fields=[*fields, "updated_at"])` — plus the optional `updatedAt` precondition when
  a record has several writers (`references/concurrency.md`).
- **One read rule, one write rule**: update and delete look the object up with `get_by_id` over
  the rows the caller may change — the Type's `get_queryset`, or one write scope narrowing it —
  not a re-implemented ownership check and not a bare `.get(pk=id)`, which turns a malformed
  integer id into a top-level error.
- No private lookup helpers (`_resolve_color`, `_color_exists`): `full_clean()` already validates
  that a FK names a real row. The exception is an authorization check on a relation
  (`references/authorization-and-leaks.md`).
- **No existence oracles.** For every failure branch, ask whether telling the cases apart leaks
  something the caller couldn't otherwise learn; if so, one identical message, pinned by a
  parametrized identical-literal test.
- Get the user via `get_current_user(info)` (`strawberry_django.auth.utils`), never
  `info.context.request.user`. Gate with the shared `staff_required` / `authenticated_required`
  on each mutation, not just the namespace, so denial is a typed `OperationInfo`; every
  not-signed-in refusal carries the code `UNAUTHENTICATED`.
- **Errors**: check whether the project sets `MUTATIONS_DEFAULT_HANDLE_ERRORS`. On: mutations
  return `<ModelName>Type | OperationInfo`, Django exceptions convert automatically (a malformed
  integer id only via `get_by_id`; `ProtectedError` / `RestrictedError` by hand), and a delete
  can't opt out because a `bool` can't union with `OperationInfo` (opt another mutation out with
  `handle_django_errors=False` only for a specific reason). Off: return `<ModelName>Type`
  and let exceptions surface. **Queries never convert Django exceptions**: a deliberate, readable
  failure there is a `GraphQLError`; a `ValidationError` is masked or leaks `"['…']"`.
- Validation belongs on the **model**; the mutation only calls `obj.full_clean()`. A nullable
  string field needs `blank=True` as well as `null=True`.
- Prune comments that restate the line below them; keep ones carrying a non-obvious *why*.
- `id: auto` exposes whatever PK the model uses (a UUID pk is a `UUID` scalar on the wire, not
  `ID`).
- This skill generates GraphQL artifacts only — it does not edit models or create migrations.
