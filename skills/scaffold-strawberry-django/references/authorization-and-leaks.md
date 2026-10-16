# Authorization-sensitive inputs, existence oracles, idempotent delete

Read this when a mutation takes a relation the caller must be *authorized* for, when a failure
branch could tell a caller something they couldn't otherwise learn, or when a delete must never
report "not found" as a failure.

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
