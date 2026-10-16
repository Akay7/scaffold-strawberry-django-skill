# Authorization-sensitive inputs, existence oracles, idempotent create and delete

Read this when a mutation takes a relation the caller must be *authorized* for, when a failure
branch could tell a caller something they couldn't otherwise learn, when a create must be safe to
retry (a client that queues writes while offline), or when a delete must never report "not found"
as a failure.

## The general rule: no existence oracles

For every failure branch you write, ask: **does telling these cases apart leak something the caller
could not otherwise learn?** If so, give every case one identical message.

- `addToBasket`-style "add a fruit by id": already sold, held in someone else's basket, unknown
  id, and a string that isn't a UUID all get the **same** message. Separate "already sold" /
  "no such id" answers let any caller enumerate which ids exist.
- Login: unknown email, wrong password and inactive account get the same message.
- Not every branch leaks. A refusal that is a statement about the caller's *own* data ("Your basket
  is empty") leaks nothing and can be specific.

Pin the guarantee with a **parametrized test** rather than a comment (see `testing.md`): every case
asserts the *same literal* response dict. Identical literals across params is what proves the
responses can't be told apart.

## Authorization-sensitive relation inputs

`apply_input` (see SKILL.md) assigns *any* pk from a `{ set: id }` input, and `full_clean()` only
proves the row exists. A relation the caller must be authorized for (e.g. "must be *your*
team") therefore needs an explicit check on what landed on the instance, **before**
`full_clean()` so the two failure modes can't produce different messages:

```python
apply_input(data, obj)
if obj.team_id is not None:
    try:
        allowed = visible_teams(user).filter(pk=obj.team_id).exists()
    except (ValidationError, ValueError):  # malformed id: same answer as a foreign one
        allowed = False
    if not allowed:
        # A nonexistent id and a foreign id are both "not in your visible set",
        # so one message covers both — full_clean() would otherwise say
        # "does not exist" for the first and stay silent for the second.
        raise ValidationError({"team": [NOT_YOURS_MSG]})
obj.full_clean()
```

`visible_teams(user)` should be the queryset that already backs the read side, not a second
hand-rolled ownership rule — or the write scope that narrows it, when callers can see teams they
may not use (SKILL.md step 3). Parametrize the test over `id_exists` (a real foreign row vs `uuid4()`)
plus any other axis, like "caller has one at all".

This is the one place a lookup is warranted. Where Django already checks something
(`full_clean()` validating that a FK names a real row), don't write a private lookup helper.

A **to-many** input (M2M, reverse FK) needs no hand-written check: give the shared `sync_m2m`
(SKILL.md) the same visible set as `rows` —
`sync_m2m(color.fruits, data.fruits, "fruits", rows=visible_fruits)`. It looks every id up in
`rows` before writing, so a foreign, unknown or malformed id is one "Not found." on the field and
nothing is linked. Pin it the same way: parametrize over a real foreign row vs `uuid4()`.

## Delete: idempotent-by-design projects

If a project's delete mutation must stay idempotent/never-raise (e.g. a client that queues writes
while offline retries it, and "not found" must not count as failure), that idempotency belongs at
the **frontend boundary**, not the resolver: keep `.get()` raising normally so the
schema/error-handling stays uniform, and have the frontend treat an `OperationInfo` response from
*that specific mutation* as a successful no-op.

That is only safe if "not found" is the *only* failure a delete can produce. It is not once any FK
to the model uses `on_delete=PROTECT`/`RESTRICT` — the resolver then also produces the
`ProtectedError` → `ValidationError` case from SKILL.md, and the frontend can no longer treat every
`OperationInfo` as success. Confirm the model has no protected reverse relations before relying on
it, or distinguish by `kind`/`code`.

## Create: idempotent on a client-generated id

A client that queues writes while offline creates a record before the server has seen it, then
retries the request until one attempt is acknowledged. The record needs an id from the start —
the client's own UUID, so nothing has to be remapped after sync — and a retry of a create that
already landed must not create a second row or overwrite the first. The id is the idempotency
key: the model's pk is a `UUIDField(primary_key=True, default=uuid.uuid4)`, and the create input
carries it as an optional `id: auto` (omitted → the model default, as before).

```python
@strawberry_django.mutation(extensions=[authenticated_required])
@transaction.atomic
def create_fruit(self, info: Info, data: FruitCreateInput) -> FruitType:
    # A retry of a create that already landed: answer with that row, untouched.
    if data.id and (existing := writable_fruits(info).filter(pk=data.id).first()):
        return cast(FruitType, reread(existing, info))
    fruit = Fruit(user=get_current_user(info))
    apply_input(data, fruit)
    fruit.full_clean()  # an id another row holds: "Fruit with this Id already exists."
    fruit.save()
    sync_m2m(fruit.tags, data.tags, "tags")
    return cast(FruitType, reread(fruit, info))
```

- **A retry returns the stored row unchanged**, even if its other fields differ from the first
  attempt — the first one to land wins. A client that edits a queued record before it syncs sends
  the edit as an update, not as a changed create.
- **The existing row must be the caller's to change** — look it up in the write scope
  (`writable_fruits`, SKILL.md step 3), or a caller could fetch another user's row by sending its
  id. An id held by a row outside that scope is a collision: one `OperationInfo` on `field: "id"`,
  "Fruit with this Id already exists.", and the other row untouched.
- **Accepted exposure.** That collision answer differs from a free id's (which is simply
  created), so a caller who already holds another user's id — from a shared view, a link, an
  ex-teammate's data — can learn whether that row still exists, and can create a row under a
  deleted record's id. Random UUIDs can't be guessed, so only ids the caller has already seen are
  exposed. The design accepts this to keep the client's id as the record's id. If it isn't
  acceptable, key retries per caller instead (a `client_id` unique together with `user`, or a pk
  derived from the caller and the client's id), so another user's id can never collide.
- **The pk must have a default** (`default=uuid.uuid4`) — and under multi-table inheritance, so
  must every parent's pk. Then `save()` on a new instance is a plain `INSERT` even with the pk
  already set, and a held id fails with `IntegrityError` without touching the row that holds it.
  Without a default, `save()` tries an `UPDATE` first and **overwrites** whatever row holds that
  id — another user's included (checked: bob's row took alice's values). `full_clean()` catches
  the collision first; the `INSERT` is what holds when two requests race past that check.
- **Two retries racing** both pass the lookup; one `INSERT` wins and the other fails with an
  `IntegrityError`, which the client sees as an unexpected error. It retries as it does after any
  failure, and that retry finds the row. Nothing is written twice, so no special handling is
  needed.
- **`id: null` is refused** ("This field cannot be null." on `id`), not treated as omitted: a
  client without an id leaves the key out.
- **Tests**, one per behaviour: a create with an id uses it; a retry with the same id returns the
  row and leaves one row (even with a different `name`); another user's id is the `id`
  `OperationInfo` and their row is unchanged; and an omitted id gets a server one.
