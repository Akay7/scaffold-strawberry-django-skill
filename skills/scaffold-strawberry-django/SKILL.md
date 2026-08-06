---
name: scaffold-strawberry-django
description: Scaffold Strawberry-Django GraphQL artifacts (Type, Filter, Inputs, Mutations, schema wiring) for a Django model. Use when adding GraphQL read/CRUD support for a model.
license: MIT
user-invocable: true
allowed-tools: Bash(grep *), Bash(find *), Bash(python *), Bash(ls *), Read(*), Edit(*), Write(*)
---

# Skill: scaffold-strawberry-django

Generate the GraphQL layer for a Django model: a **Type + Filter**, **Inputs + Mutations**, and
wire both into the schema. If the repo has a GraphQL conventions doc (e.g. `docs/graphql.md`),
read it first and let it override anything below.

**Scope**: this scaffolds a plain object/list-field API — query fields return `list[<Type>]` (or
`strawberry_django.offset_paginated`), and objects aren't required to implement Relay's `Node`
interface. It does not scaffold Relay-style **Global Object Identification** (a global `node(id:
ID!)` root field resolving any type by opaque ID) or cursor-based **Connections** (`edges`/`node`/
`pageInfo`, `first`/`after` arguments). If the target project's GraphQL layer is Relay-routed, treat
everything below as needing translation to that shape rather than applying it as-is.

Examples use the `Fruit` / `Color` models from the Strawberry-Django documentation as neutral
placeholders (`Fruit` has a `name` and a FK `color` → `Color`). Substitute the real model.
Before writing, find one existing Type and one existing mutation in the repo to copy the exact
local idiom (directory layout, naming, imports):

```bash
grep -rl "strawberry_django.type" . | grep -v __pycache__ | head
grep -rl "strawberry_django.mutation\|strawberry_django.input" . | grep -v __pycache__ | head
```

### Reusable helpers

The delete mutation returns the model's own type (`<ModelName>Type`), not a `bool` or a
`DeletionResult`-style wrapper — this matches strawberry_django's own convention for its
generated delete mutations (`delete_x: XType = mutations.delete(...)`, see its
`mutations.resolvers.delete()`; also documented at
https://strawberry.rocks/docs/django/guide/mutations). A `bool` can't participate here: with
`handle_django_errors=True` (the project default — see below), the success type gets unioned
with `OperationInfo`, and GraphQL unions can only contain object types, not scalars (confirmed by
building a scratch schema with a `bool`-returning mutation: `InvalidUnionTypeError: Type 'bool'
cannot be used in a GraphQL Union`).

Since `Model.delete()` clears the in-memory instance's pk (Django sets it to `None` after the row
is removed), save and restore it so the response carries a real id — exactly what
strawberry_django's own resolver does:

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

---

## Steps

### 1. Locate the model and its app

```bash
grep -rln "class <ModelName>(" . | grep -v __pycache__
```

From the path `<app>/models/<stem>.py` capture `<app>` and `<stem>`. Read the model file to learn:

- Fields and their types (scalars, FKs, M2M, OneToOne).
- The primary-key type (`id: auto` exposes whatever the model uses — int or UUID).
- Any `position`/ordering field (→ may need a reorder mutation).
- Which related models already have Types/Filters you can import and reuse.

Derive the names: `<ModelName>Type`, `<ModelName>Filter`, `<ModelName>Input`,
`<ModelName>UpdateInput`, `<ModelName>Mutations`, and the camelCase query/mutation field names
(`<modelName>` / `<model_name_plural>`).

### 2. Generate the Type + Filter (place it next to other types, e.g. `<app>/types/<stem>.py`)

Order within the file: **Filter first, then Type** (the Type references the Filter).

- `@strawberry_django.filter_type(<ModelName>, lookups=True)` — `id: auto`; scalar fields `auto`
  (including choice fields — they become enum lookups; never hand-annotate
  `str | None = strawberry.UNSET`); FK fields as a nested relation filter,
  `color: ColorFilter | None = strawberry.UNSET` (define a minimal filter for the related model if
  none exists — with a UUID pk its `id: auto` is a `UUID` scalar on the wire, not `ID`). Reach for a
  custom `@strawberry_django.filter_field` method **only** when the filter cannot be expressed as a
  narrowing of the base queryset (e.g. selecting a *different*, disjoint scope than the type's
  `get_queryset` default) — a filter method may replace the queryset by returning
  `tuple[QuerySet, Q]` from a `(self, value, queryset, info, prefix)` signature.
- `@strawberry_django.order_type(<ModelName>)` `class <ModelName>Order` — the fields the client may
  sort by, each annotated `Ordering` (`from strawberry_django.ordering import Ordering`).
- `@strawberry_django.type(model=<ModelName>, filters=<ModelName>Filter, ordering=<ModelName>Order)`:
  - Use `auto` for field annotations wherever possible (let Strawberry infer from the model).
  - Expose nested object fields as the related type, e.g. `color: ColorType`. Annotate `RelatedType | None`
    if the relation is nullable. **Never denormalize a related scalar into a resolver field**
    (e.g. an `owner_username` resolver returning `self.user.username`) — expose the relation
    (`user: UserType`) and let clients select `user { username }`; the optimizer handles the join.
  - **Per-row computed values come from queryset annotations, not python resolvers.** A field whose
    value is derivable per row in SQL (an aggregate over related rows, a correlated subquery) is
    declared with an `annotate=` optimizer hint, computed once in the list query instead of N+1
    python-side:
    ```python
    total_spent: Decimal = strawberry_django.field(
        annotate=lambda info: my_subquery_expression()  # callable → rebuilt per request
    )
    ```
    A bare expression works too; use the callable form when the expression depends on request time
    ("today"-relative windows) or the viewer. **Caveat:** the optimizer only annotates querysets it
    resolves — a mutation's returned instance doesn't carry the annotation. For create/update,
    re-read the saved row through the same annotation before returning (the alias is the field's
    python name, so the default resolver finds it):
    ```python
    return cast(FruitType, Fruit.objects.annotate(total=my_expression()).get(pk=obj.pk))
    ```
    Delete payloads can't do this (the row is gone) — they select `id` only.
  - **Don't compute what the client can derive.** A boolean like `is_shared`
    (`self.partnership_id is not None`) is not a field — expose the relation
    (`partnership: PartnershipType | None`) and let the client derive the flag from null-ness.
    Custom `@strawberry.field` resolvers are reserved for values that are genuinely viewer-relative
    or otherwise inexpressible as a model field, relation, or annotation.
  - A related type that must resolve as a nested relation has to be a `strawberry_django.type` over
    the related model — a plain `@strawberry.type` built by hand can't. If a plain one already
    exists (e.g. constructed by a status resolver), convert it: keep model fields `auto`, keep any
    viewer-relative field as a resolver (with `select_related=[...]` hints), and have former
    construction sites return the model instance (`cast(TheType, obj)`) instead of building DTOs.
  - Override `get_queryset` only if you need to scope/annotate the base queryset:
    ```python
    @classmethod
    def get_queryset(cls, qs: QuerySet[<ModelName>], info: Info) -> QuerySet[<ModelName>]:
        return qs.filter(user=get_current_user(info))   # scope; do NOT sort here
    ```
- **Do not** add `select_related`/`prefetch_related` in `get_queryset` — the Django optimizer extension handles it.
- **Scope** with `get_current_user(info)` (`from strawberry_django.auth.utils import get_current_user`) —
  never read `info.context.request.user` directly.
- **Do not sort** in `get_queryset` or resolvers. Ordering is the client's choice: define the
  `<ModelName>Order` above and let callers pass the generated `ordering` argument.
- **Choice fields** (`models.TextChoices` / `django-choices-field`): annotate `auto` — they become a
  GraphQL **enum**, serialized by member *name* (`Direction.OUT = "out"` → `OUT` on the wire). This
  is the default for Inputs and Filters. Annotate `str` only to preserve an *existing* wire format
  that already ships plain values (e.g. a deployed output field clients read as lowercase strings) —
  never introduce a new `str`-annotated choice field.
  - **Enum name collisions**: the generated GraphQL enum takes its name from the Python
    `TextChoices`/`IntegerChoices` class itself, not from the model or field that uses it. GraphQL
    type names are global, so two unrelated models each declaring their own inner class with the same
    name (e.g. two different models both defining a `class Status(models.TextChoices)`) collide on
    one GraphQL enum type — same failure mode as the mutation "Union name collisions" case below, and
    same fix: give each its own distinct class name (`TransactionStatus`, `InviteStatus`, ...) rather
    than a generic `Status` repeated per model. Check for an existing choices class with the same name
    elsewhere in the codebase before adding a new one.
  - Once introspection is disabled outside dev (see `DisableIntrospection` under "Defining the schema
    object" below), a client can no longer discover an enum's members by querying the schema itself —
    if a product feature needs the caller to know the valid values (e.g. to populate a picker), expose
    them through an explicit query field, not through introspection.
- For cross-app/cross-module related types, reference them **lazily** to avoid circular imports. Wrap the
  type in `typing.Annotated` with a *stringified* type name and `strawberry.lazy(<module path>)`:
  ```python
  from typing import Annotated
  import strawberry

  category: Annotated["CategoryType", strawberry.lazy("categories.types")]
  # nullable relation:
  color: Annotated["ColorType", strawberry.lazy("fruits.types.color")] | None
  ```
  The string `"CategoryType"` is the type's class name; `strawberry.lazy("categories.types")` is the
  **absolute** import path of the module that defines it (Strawberry imports it only at schema-build
  time). Do **not** import the related type at the top of the file for these fields — that's the cycle
  you're avoiding. Types in the *same* module can be referenced directly without `lazy`.

### 3. Generate Inputs + Mutations (place next to other mutations in the repo)

- `@strawberry_django.input(model=<ModelName>)` `class <ModelName>Input` — create fields (use `auto`).
- `@strawberry_django.input(model=<ModelName>, partial=True)` `class <ModelName>UpdateInput` — same
  fields; `partial=True` makes them all optional so an edit sends only what changed.
- **FK inputs**: `auto` on a FK produces a relation-input (`{ set: id }`). If you'd rather accept a
  plain id, declare `<fk>_id: strawberry.ID | None` explicitly (override `auto`) — keep the Type and
  Input consistent. Same trick for choice fields you want as `str` instead of an enum.
- **M2M inputs**: `auto` on a ManyToManyField generates `ManyToManyInput` (`{ set, add, remove }`).
  Use `auto` in both the create and update inputs. In the mutation, access `.set or []` for
  replace-all semantics. With `partial=True`, the whole field is `UNSET` when omitted.
- **`handle_django_errors` defaults to `True` project-wide** (`STRAWBERRY_DJANGO =
  {"MUTATIONS_DEFAULT_HANDLE_ERRORS": True}` in settings — check it's actually set before assuming
  this; if the repo hasn't adopted it, follow the repo's existing pattern instead). With it on,
  `ValidationError`/`PermissionDenied`/`ObjectDoesNotExist` raised inside create/update resolvers
  are automatically converted to a typed `<ModelName>Type | OperationInfo` union return instead of
  a raised top-level GraphQL error — **you don't write any error-catching code for this**, it's
  automatic. Only `full_clean()`/`update_model_instance_based_on_input`'s `field_errors` +
  `raise ValidationError(...)` need to be present, same as before.
  ```python
  from collections import defaultdict
  from django.core.exceptions import ValidationError
  from strawberry import UNSET
  from config.graphql import update_model_instance_based_on_input

  @strawberry.type
  class <ModelName>Mutations:
      # Named `create_<model_name>`/`update_<model_name>` (NOT bare `create`/`update`) — see
      # "Union name collisions" below for why this matters when handle_django_errors is on.
      # `name="create"` is optional: only add it if this project's convention exposes the
      # field as `create`/`update` (check CLAUDE.md / an existing mutation first); if a more
      # specific exposed name (e.g. `createFruit`) is fine or preferred here, drop `name=`
      # entirely and let it auto-camelCase from the method name.
      @strawberry_django.mutation(name="create")
      def create_<model_name>(self, info: Info, data: <ModelName>CreateInput) -> <ModelName>Type:
          user = get_current_user(info)
          # Don't hand-copy `data.name, data.color, ...` field by field — apply the
          # input with the same shared applier update uses. Construct with only what
          # the input can't carry (the owner), then let the applier do the rest
          # (scalars + `{ set: id }` FK/OneToOne inputs; M2M is skipped, sync it
          # separately after save).
          obj = <ModelName>(user=user)
          field_errors: defaultdict[str, list[str]] = defaultdict(list)
          update_model_instance_based_on_input(data, obj, user, field_errors)
          if field_errors:
              raise ValidationError(field_errors)
          obj.full_clean()
          obj.save()
          return cast(<ModelName>Type, obj)

      @strawberry_django.mutation(name="update")
      def update_<model_name>(self, info: Info, id: ID, data: <ModelName>UpdateInput) -> <ModelName>Type:
          user = get_current_user(info)
          # The single-object lookup is a `.get(id=...)` on the SAME visibility
          # queryset that backs the type's `get_queryset` (extract it into a
          # `resolve_visible_<x>(user)` helper if it's inline) — never a second
          # hand-rolled ownership/membership check, which would drift from the
          # read path. The scoped `.get()` also keeps the not-found-or-not-yours
          # blend: both cases raise DoesNotExist identically.
          obj = resolve_visible_<model_name_plural>(user).get(id=id)  # DoesNotExist -> OperationInfo, not raised
          field_errors: defaultdict[str, list[str]] = defaultdict(list)
          update_model_instance_based_on_input(data, obj, user, field_errors)
          if field_errors:
              raise ValidationError(field_errors)  # type: ignore[arg-type]
          obj.full_clean()
          obj.save()
          return cast(<ModelName>Type, obj)

      # No opt-out here: delete uses the same handle_django_errors=True default
      # as create/update, for a uniform error contract. Returns the model's
      # own type, not a wrapper — see "Reusable helpers" above for why, and
      # for the pk-restore dance `.delete()` requires.
      @strawberry_django.mutation
      def delete_<model_name>(self, info: Info, id: strawberry.ID) -> <ModelName>Type:
          user = get_current_user(info)
          obj = resolve_visible_<model_name_plural>(user).get(id=id)  # DoesNotExist -> OperationInfo, not raised
          pk = obj.pk
          obj.delete()
          obj.pk = pk  # Django nils it post-delete; restore for the response
          return cast(<ModelName>Type, obj)
  ```
  If this project's delete mutation must stay idempotent/never-raise (e.g. because a client
  offline-sync outbox retries it and a "not found" response must not be treated as failure — see
  `lib/transactions.ts`'s `deleteTransaction()` in this repo, if present, for the pattern), that
  idempotency belongs at the **frontend boundary**, not the resolver: keep `.get()` raising
  normally so the schema/error-handling stays uniform, and have the frontend caller treat an
  `OperationInfo` response from *that specific mutation* as a successful no-op (safe only because
  `.get()` raising `DoesNotExist` is the *only* exception a plain delete resolver can produce —
  confirm that's still true before relying on it).
- **Union name collisions**: `handle_django_errors=True` auto-generates an error-union type named
  from the **Python method name** alone — `create` → `"CreatePayload"` — not from the mutation's
  namespace/class. GraphQL type names are global, so a second, unrelated mutation method also
  literally named `create` (e.g. in another model's `<OtherModel>Mutations`) collides on that same
  union name; Strawberry silently keeps only the first one registered, and the second's fragments
  fail schema validation ("Fragment cannot be spread here..."). Avoid this by never naming the
  Python method bare `create`/`update` — always `create_<model_name>`/`update_<model_name>`, as
  shown above. This mirrors the existing `delete_<model_name>` convention. Add `name="create"`/
  `name="update"` only if this project's exposed field name needs to stay `create`/`update`;
  otherwise the auto-camelCased method name (e.g. `createFruit`) is a fine exposed name too.
- Mutation input parameter is named `data` — Strawberry-Django's default argument name.
- **Create and update both apply the input via the shared applier**
  (`update_model_instance_based_on_input`) — never hand-copy `data.<field>` onto the instance
  field by field. Create constructs the instance with only what the input can't carry (the owner),
  then runs the applier; M2M inputs are synced separately after save.
- **A mutation that writes multiple rows is atomic** — e.g. the instance `save()` plus its M2M
  sync, or any multi-model change — so the writes land together or not at all. For readability,
  decorate the whole resolver rather than wrapping an inner block, placing `@transaction.atomic`
  *under* the mutation decorator:
  ```python
  @strawberry_django.mutation
  @transaction.atomic
  def create_fruit(self, info: Info, data: FruitCreateInput) -> FruitType:
  ```
  A raised `ValidationError` still rolls back first and then becomes `OperationInfo`. A single-row
  save needs no explicit transaction, and neither does `delete()` (Django's collector already runs
  deletes atomically).
- **Authorization-sensitive relation inputs are checked on the instance, after the applier.** The
  applier resolves *any* existing pk for a `{ set: id }` input, so a relation the caller must be
  authorized for (e.g. "must be *your* partnership/team") gets an explicit check on what actually
  landed on the instance. Guard against the existence leak: the applier's "Not found." (nonexistent
  id) and your "not yours" (foreign id) must surface as ONE blended message — *assign* (don't
  append) the field's error list so the applier's entry is replaced:
  ```python
  update_model_instance_based_on_input(data, obj, user, field_errors)
  if obj.partnership is not None or "partnership" in field_errors:
      active = get_active_partnership(user)
      if active is None or obj.partnership != active:
          field_errors["partnership"] = [NOT_YOURS_MSG]  # replaces "Not found."
  ```
  Pin the no-leak guarantee with a **parametrized test** rather than a comment: parametrize over
  `id_exists` (a real foreign row vs `uuid4()`) — plus any other axis, like "caller has one at
  all" — and have every case assert the *same literal* response dict (`__typename`, `messages`
  with kind/field/message/code). Identical literals across params is what proves the responses
  can't be told apart.
- Validation lives **on the model** (field `validators=`, a DB `CheckConstraint`, `Model.clean()`);
  the mutation just calls `obj.full_clean()` and lets any `ValidationError` become a typed
  `OperationInfo` (or, if `handle_django_errors` is off/not adopted in this repo, a top-level
  GraphQL error — check first).
- Input class naming: `<ModelName>CreateInput` (not `<ModelName>Input`) to distinguish from the
  update input and avoid name collisions when multiple mutations share method names like `create`.

### 4. Wire into the schema (`schema.py` / wherever `Query` and `Mutation` are defined)

- **Imports** (follow the existing grouping/order):
  - `from <app>.types.<stem> import <ModelName>Type`
  - `from <...>.mutations.<stem> import <ModelName>Mutations`
- **Query** — gate auth with the `IsAuthenticated` extension. For a *list* field an unauthorized
  caller gets an empty list (not an error), which is usually what you want:
  ```python
  from strawberry_django.permissions import IsAuthenticated

  <model_name_plural>: list[<ModelName>Type] = strawberry_django.field(
      extensions=[IsAuthenticated()],
  )
  # or OffsetPaginated[<ModelName>Type] = strawberry_django.offset_paginated(extensions=[...])
  ```
- **Mutation** (nested one level) — gate the whole namespace once on the wrapper field:
  ```python
  @strawberry.field(extensions=[IsAuthenticated()])
  def <model_name>(self) -> <ModelName>Mutations:
      return <ModelName>Mutations()
  ```

#### Defining the `schema` object

There is a single module-level variable named **`schema`** — an instance of `strawberry.Schema`
built from the `Query` and `Mutation` classes. It must already exist (you're adding fields to its
`Query`/`Mutation`, not creating it). If the repo has no schema yet, define it at the bottom of the
same module, after the `Query` and `Mutation` classes:

```python
import strawberry
from django.conf import settings
from strawberry.extensions import (
    DisableIntrospection,
    MaxAliasesLimiter,
    MaxTokensLimiter,
    QueryDepthLimiter,
)
from strawberry_django.optimizer import DjangoOptimizerExtension

# Pass the class / factory callables (NOT instances) — recent Strawberry deprecates
# passing a built extension instance (it must build a fresh one per request).
extensions = [
    DjangoOptimizerExtension,     # generates select_related/prefetch_related automatically
    lambda: MaxTokensLimiter(max_token_count=settings.STRAWBERRY_MAX_TOKENS),
    lambda: MaxAliasesLimiter(max_alias_count=settings.STRAWBERRY_MAX_ALIASES),
    lambda: QueryDepthLimiter(max_depth=settings.STRAWBERRY_MAX_QUERY_DEPTH),
    # Don't publish the schema outside development: introspection is a
    # reconnaissance aid against a production API, and nothing shipped should
    # need clients to read the schema itself (they query declared fields).
    # Mirrors DEBUG-gating other dev-only surfaces (e.g. a debug toolbar);
    # dev keeps GraphiQL/introspection working.
    *([] if settings.DEBUG else [DisableIntrospection]),
]

schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    # Field names default to camelCase (don't disable auto_camel_case); datetimes
    # are ISO 8601 via the built-in DateTime scalar.
    extensions=extensions,
)
```

The variable **must be called `schema`** — Django/Strawberry config points at it by dotted path
(e.g. `STRAWBERRY_DJANGO` / the `GraphQLView`'s `schema=`, typically `"<module>.schema:schema"`).

**The four base extensions are required, plus a fifth in production:**

- `DjangoOptimizerExtension()` — makes the "no manual `select_related`/`prefetch_related`" rule work.
- `MaxTokensLimiter`, `MaxAliasesLimiter`, `QueryDepthLimiter` — abuse/DoS guards against oversized,
  alias-bombed, or deeply-nested queries. Never drop them.
- `DisableIntrospection`, added only when `not settings.DEBUG` — introspection lets a caller read the
  entire schema (every type, field, and enum value) without any query of their own; disabling it in
  production removes that reconnaissance surface while leaving GraphiQL/introspection-based tooling
  working in dev. Any product feature that needs client-visible reference data (e.g. an enum's valid
  values) must ship as its own explicit query field rather than relying on introspection to expose it.
  Gate it the same way the repo gates other dev-only surfaces (check `settings.py` for an existing
  `DEBUG`-gated toggle, e.g. a debug toolbar, and mirror that condition rather than inventing a new one).

#### Limiter values via environment variables

The three limits are read from Django settings, which in turn read environment variables (so they can
differ per environment without code changes). Add to your settings module — using `django-environ`:

```python
# settings.py
import environ
env = environ.Env()

STRAWBERRY_MAX_TOKENS = env.int("STRAWBERRY_MAX_TOKENS", default=1000)
STRAWBERRY_MAX_ALIASES = env.int("STRAWBERRY_MAX_ALIASES", default=10)
STRAWBERRY_MAX_QUERY_DEPTH = env.int("STRAWBERRY_MAX_QUERY_DEPTH", default=10)
```

or with plain `os.environ`:

```python
import os
STRAWBERRY_MAX_TOKENS = int(os.environ.get("STRAWBERRY_MAX_TOKENS", 1000))
STRAWBERRY_MAX_ALIASES = int(os.environ.get("STRAWBERRY_MAX_ALIASES", 10))
STRAWBERRY_MAX_QUERY_DEPTH = int(os.environ.get("STRAWBERRY_MAX_QUERY_DEPTH", 10))
```

Then set the env vars (e.g. in `.env`, Docker, or CI) to override the defaults:

```bash
STRAWBERRY_MAX_TOKENS=1000      # max total tokens in a query document
STRAWBERRY_MAX_ALIASES=10       # max field aliases per query
STRAWBERRY_MAX_QUERY_DEPTH=10   # max nesting depth
```

### 5. Verify

```bash
python manage.py check
```

Confirm the schema builds (no `UnresolvedFieldTypeError` from lazy paths, no duplicate type names).

### 6. Offer next steps

Suggest adding tests for the new queries/mutations, and `make makemigrations` only if the model
itself changed (this skill does not change models).

**Test-writing conventions:**
- **Layout**: tests live in a per-app `tests/` package next to the model/type/mutation code being
  tested (`<app>/tests/test_<stem>.py`), not in one repo-wide top-level test directory. Confirm this
  against the project's actual test runner invocation (e.g. its CLAUDE.md/README "run a single test"
  example) before assuming the path — the package name and how deep it nests can differ per repo.
- One `@pytest.mark.django_db()` function per case — **no test classes**, even though pytest and
  pytest-django both support (and their own docs often default to) grouping cases in a
  `class Test<Something>:` container. Call this out explicitly because that default is easy to
  reach for out of habit or from generic examples; this project's convention is flat, independent
  functions per file, grouped only by filename — don't reintroduce classes "for organization."
- Keep all GraphQL document strings as **constants at the top of the file** (queries and
  mutations), then helpers, then the test functions.
- Always assign the query result to a variable (e.g. `res = gql_client.query(...)`) and inspect
  it — never discard it, and never rely on a client-level "assert no errors" convenience flag for
  a case where you expect a failure; inspect the response body directly instead.
- Use named local variables for values that appear in both setup and assertions (e.g.
  `initial_first_name = "Kept"`). If the mutation changes a value, name that too (e.g.
  `new_city = "Berlin"`) and assert it in both the response and the DB.
- For `OperationInfo` message assertions: assert `len(messages) == N`, then index into each
  message and check `kind`/`field`/`message`/`code` explicitly. Never `any()`, a `for` loop, or
  other iteration to search for a matching message — an exact count plus exact fields is what
  proves the response can't be hiding an extra or different error (this is the same "identical
  literals across params" principle as the no-leak guarantee test above).
- Permissions: grant object-level access in a test with `assign_perm(PERM, user, obj)`
  (django-guardian). If this project scopes access some other way (a role/group field, a
  membership table), match that project's existing test setup instead — check an existing
  permission-gated test for the pattern before writing a new one.

Test pattern depends on whether `handle_django_errors` is on for this mutation:
- **With `handle_django_errors` on** (the project default, if adopted — this includes delete,
  which uses the same default as create/update, see "Reusable helpers"): the mutation field
  returns a union, so every create/update/delete selection needs `__typename` plus both
  `... on <ModelName>Type { ... }` and `... on OperationInfo { messages { field message code } }`
  inline fragments. Success → `"errors" not in body` and
  `body["data"]["<model>"]["create"]["__typename"] == "<ModelName>Type"`. Validation failure →
  `"errors" not in body` still, but `result["__typename"] == "OperationInfo"`,
  `len(result["messages"]) == 1`, and `result["messages"][0]["field"] == "<field>"` (check
  `kind`/`message`/`code` too if the test cares about them — no `any()`/iteration, see
  "Test-writing conventions" above). Auth failure → still `assert body.get("errors")` (the
  IsAuthenticated extension raises before the resolver runs, so this is unaffected either way).
  For delete specifically, "not found for this user" is *also* an `OperationInfo` (not a raised
  error) — same reasoning as update's `.get()`.
- **Without it** (a mutation that explicitly opts out with `handle_django_errors=False` for its
  own reason, e.g. a genuinely idempotent/never-raising resolver): success → check fields
  directly on `body["data"]["<model>"]["create"]`; not-found/validation failure →
  `assert body.get("errors")`; auth failure → `assert body.get("errors")` (the namespace field is
  null).

---

## Critical rules

- `auto` annotations wherever possible — let Strawberry infer field types from the model. This
  includes choice fields (→ enum) in filters and inputs. The only exceptions: preserving an
  existing plain-string wire format on an already-deployed field, and FK inputs you want as
  `<fk>_id: ID` (not a relation-input) — annotate those explicitly.
- Filters: no custom `filter_field` methods for what a declared field can express — a relation
  scope filter is `user: UserFilter | None = strawberry.UNSET`, not a resolver. Custom methods are
  reserved for filters that can't be written as a narrowing of the base queryset.
- Types: no denormalized related-scalar resolvers (`owner_username`-style) — nest the related type
  and let clients select the field they need. Same for derived booleans (`is_shared`-style): expose
  the relation, the client derives the flag.
- Per-row computed values: `strawberry_django.field(annotate=...)` (callable for request-time
  expressions), never a python resolver looping per row. Create/update resolvers re-read the saved
  row through the same annotation before returning so their payloads can select the field; delete
  payloads can't (row gone).
- Don't call `refresh_from_db()` after save "just in case" — it doesn't refresh M2M (those query
  the DB on every access) and no column changes after `save()`; an annotated re-read (above) or
  nothing at all is correct.
- M2M fields: use `auto` in both create and update inputs → generates `ManyToManyInput`. In mutation
  code access `.set or []` for replace-all; with `partial=True`, the whole field is `UNSET` when
  omitted. Clients send `{ set: [...] }`.
- Nullable relations are annotated `RelatedType | None`.
- Mutation input parameter is named `data` (Strawberry-Django's default argument name); update inputs
  are `partial=True`. Create input class is named `<ModelName>CreateInput`.
- Get the user via `get_current_user(info)` (`strawberry_django.auth.utils`), never
  `info.context.request.user`.
- **One visibility rule, used everywhere**: update/delete look the object up via
  `resolve_visible_<x>(user).get(id=id)` — the same queryset backing the type's `get_queryset` —
  not a re-implemented ownership/membership check in the mutation. Call it inline; don't wrap the
  one-liner in a `_get_accessible_<x>` helper.
- Check whether this project sets `STRAWBERRY_DJANGO = {"MUTATIONS_DEFAULT_HANDLE_ERRORS": True}`
  in settings before assuming `handle_django_errors` behavior either way — grep for it first.
  - **If it's on** (or you're passing `handle_django_errors=True` explicitly): create/update/delete
    all return `<ModelName>Type | OperationInfo`; name the Python methods `create_<model_name>`/
    `update_<model_name>`/`delete_<model_name>` (see "Union name collisions"); `full_clean()`/
    `ValidationError`/`ObjectDoesNotExist` are converted automatically, no manual catching needed.
    Delete gets no special opt-out — see "Reusable helpers" for why (a `bool` return can't union
    with `OperationInfo` at all, and the model type is strawberry_django's own convention). A
    mutation opts out with `handle_django_errors=False` only when it has its own specific reason
    to keep raise-on-failure semantics (e.g. a deliberately idempotent/never-raising resolver).
  - **If it's off/not adopted**: return `<ModelName>Type` directly. Strawberry surfaces
    `DoesNotExist` and other Django exceptions as GraphQL errors automatically — no need to catch
    and re-raise. Only use `GraphQLError` explicitly when the failure condition isn't a model
    exception (e.g. `if not deleted` after a filter-based delete).
- Validation belongs on the **model** (field validators, DB `CheckConstraint`, `clean()`); the
  mutation only calls `obj.full_clean()` and lets any `ValidationError` surface as either an
  `OperationInfo` or a top-level GraphQL error, per the above.
- Ordering is client-requested via an `order_type` + the `ordering` argument — never sort in a resolver
  or `get_queryset`.
- Auth: gate query fields and the mutation namespace with the `IsAuthenticated()` field extension.
- Schema extensions are passed as the class or a factory callable, never as a built instance.
- Cross-module related types use lazy refs: `Annotated["TypeName", strawberry.lazy("app.types.module")]`
  (append `| None` for nullable). `strawberry.lazy()` paths are always absolute module paths.
- `id: auto` exposes whatever PK the model uses.
- This skill generates GraphQL artifacts only — it does not edit models or create migrations.
