# Inputs and Mutations: detail

Detail for step 3 of `SKILL.md`; the mutation template there is the shape every section
below assumes.

## M2M and reverse-FK inputs

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
  `rows` (their visible set, or the narrower write scope in "The rows update and delete may touch"
  below) — `sync_m2m(color.fruits, data.fruits, "fruits", rows=visible_fruits)` with
  `visible_fruits = FruitType.get_queryset(Fruit.objects.all(), info)` — and a row they can't
  see gets the same "Not found." as one that doesn't exist
  (`references/authorization-and-leaks.md`). `remove`, and a `set` that drops rows, need the FK
  to be nullable. Pass `rows` for an M2M too when its related rows aren't all the caller's to
  link (per-user tags).

## The rows update and delete may touch

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

## Delete returns the model's own type

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

- **`ProtectedError` / `RestrictedError` are not converted automatically.** If any FK to the model
  uses `on_delete=PROTECT`/`RESTRICT`, catch it around `delete()` and raise `ValidationError` as
  above; otherwise an in-use delete surfaces as a server error. Plain delete needs no explicit
  transaction (Django's collector already runs it atomically).

## Union name collisions

- **Union name collisions**: `handle_django_errors=True` auto-generates an error-union type named
  from the **Python method name** alone — `create` → `"CreatePayload"` — not from the mutation's
  namespace/class. GraphQL type names are global, so a second, unrelated mutation method also
  named `create` (in another model's `<OtherModel>Mutations`) collides on that union name;
  Strawberry silently keeps only the first one registered, and the second's fragments fail schema
  validation ("Fragment cannot be spread here..."). Hence `create_<model_name>` /
  `update_<model_name>` / `delete_<model_name>`. Add `name="create"` only if this project's exposed
  field must stay `create`/`update`.

## Atomic writes

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

## Create and update return their row through `reread`

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
