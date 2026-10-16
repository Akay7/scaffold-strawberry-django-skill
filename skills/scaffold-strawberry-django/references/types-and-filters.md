# Types and Filters: relations, computed values, choice fields

Detail for step 2 of `SKILL.md`. Read it when the model has a relation, a computed value or a
choice field.

## Filters

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

## Relations and visibility

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
  their row through the shared `reread` (`SKILL.md` step 3). Scoping the related Type also
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
- A related type that must resolve as a nested relation has to be a `strawberry_django.type` over
  the related model — a plain `@strawberry.type` built by hand can't. If a plain one already
  exists (e.g. constructed by a status resolver), convert it: keep model fields `auto`, keep any
  viewer-relative field as a resolver (with `select_related=[...]` hints), and have former
  construction sites return the model instance (`cast(TheType, obj)`) instead of building DTOs.

## Computed values

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
  their row through the shared `reread(obj, info)` (`SKILL.md` step 3), which loads it through the
  optimizer, so the annotation is there with no second copy of its expression. Delete payloads
  can't do this (the row is gone) — they select `id` only. Every other resolver or non-model type
  that hands out instances of the Type needs the same, or selecting the field raises. So before adding an
  `annotate=` field, check "Don't compute what the client can derive" below — if the client can derive the value, don't add it.
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

## One Type per model, and results that aren't models

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

## Choice fields

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
    mode as "Union name collisions" in `mutations.md`, and same fix: give each its own distinct class name
    (`FruitStatus`, `ColorStatus`, ...). Check for an existing choices class with the same name
    before adding a new one. (The `GENERATE_ENUMS_FROM_CHOICES` names are per field and don't collide.)
  - Once introspection is disabled outside dev (see `references/schema-and-limits.md`), a client
    can no longer discover an enum's members from the schema — if a product feature needs the
    caller to know the valid values (e.g. a picker), expose them through an explicit query field.

## Importing related types

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
