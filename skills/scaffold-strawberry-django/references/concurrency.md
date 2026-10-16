# Update writes: only what was sent, and an optional version check

Read this when generating an update mutation, and always when the record has more than one writer
(several users sharing it, a background job that also writes it, an offline client that queues
edits).

## The problem

`apply_input(data, obj)` → `full_clean()` → `obj.save()` writes **every column** from a copy read
earlier in the request. Anything committed in between — another request, a background task,
another user — is silently reverted, including columns the client never sent. It is a real, quiet
data-loss bug: a background job applies a result, then a user's stale save writes the old values
back over it.

## 1. Always: write only the fields that were sent

Have `apply_input` return the names of the fields it set, and pass them to `save`:

```python
fields = apply_input(data, fruit)
fruit.full_clean()
fruit.save(update_fields=[*fields, "updated_at"])
```

- Django skips the save entirely for an **empty** `update_fields`, so always include the model's
  `auto_now` version field (here `updated_at`) — otherwise an edit that changes nothing doesn't
  advance the version. Drop it only if the model has none.
- `update_fields` takes field names (`color`, not `color_id`). M2M is synced after the save, as before.

This alone stops an edit from reverting columns it didn't send.

## 2. When the record has several writers: an optional version precondition

The client sends back the version it read (`updatedAt`) as a field **of the update input**. It is a
precondition, never a value to write. `updated_at` is `auto_now` (not editable), so `auto` can't
generate it — declare it by hand on the input:

```python
@strawberry_django.input(model=Fruit, partial=True)
class FruitUpdateInput:
    name: auto
    ...
    updated_at: datetime | None  # the version the edit was based on
```

The resolver takes it out of the input **before** `apply_input`, so it is never written:

```python
base_version = None if data.updated_at is strawberry.UNSET else data.updated_at
data.updated_at = strawberry.UNSET
fields = apply_input(data, fruit)
fruit.full_clean()
save_input_fields(fruit, fields, base_version)
```

The write is one conditional `UPDATE`, checked by row count — no lock, and nothing can land between
the check and the write:

```python
def save_input_fields(instance, fields, base_version=None):
    if base_version is None:  # a client that sends no version: an ordinary narrow save
        instance.save(update_fields=[*fields, "updated_at"])
        return
    values = {name: getattr(instance, name) for name in fields}
    written = (
        type(instance).objects.filter(pk=instance.pk, updated_at=base_version)
        .update(**values, updated_at=timezone.now())  # update() skips auto_now: set it
    )
    if not written:
        raise ValidationError(
            {"updated_at": [ValidationError(_("This record changed since the edit's base version."), code="conflict")]}
        )
    instance.refresh_from_db(fields=["updated_at"])
```

- **Wire shape.** With `handle_django_errors` on, the conflict is an `OperationInfo` message
  `{ kind: VALIDATION, field: "updatedAt", code: "conflict" }`. Clients tell it from an ordinary field
  error by that field, so no new payload type is needed.
- **Omitted or `null` means no check**, so clients that don't send it keep working (deploy the server
  first).
- **Send the version back verbatim**, as the string the server returned. Round-tripping it through a
  JavaScript `Date` truncates microseconds and every edit would then conflict.
- **Prefer this conditional `UPDATE` to `select_for_update()`.** Both refuse a stale version — each
  compares against the client's base version — but a lock is held until the outermost transaction
  ends (with an atomic resolver, the rest of the request). Use the locked variant below only when
  the write must go through `save()`.
- Keep the resolver atomic if it does more than the one write (an M2M `sync_m2m`, stamping a related row):
  the conflict is raised before them and rolls the request back as a unit.
- Delete is unchanged; it has no `data` to carry a version.

### Limits of the versioned write

`QuerySet.update()` is one SQL `UPDATE`; Django's model layer takes no part. So when a
`base_version` is sent:

- the model's **`save()` override doesn't run** — whatever it derives (a `slug` from the `title`)
  or triggers (publishing an event) is skipped;
- **`pre_save` / `post_save` signals don't fire** — receivers that audit, invalidate a cache or
  reindex never hear of the edit;
- **no field's `pre_save()` runs**, so an `auto_now` field other than the version isn't advanced
  (the helper sets `updated_at` by hand, and nothing else).

One limit holds for **every** narrow write, `save(update_fields=…)` included: only the listed
columns reach the database. A column that `save()` derives, or a second `auto_now` field, is
computed in memory and then dropped unless it is listed — have the model's `save()` add it to
`update_fields` whenever its source field is there, or list it from the resolver.

When the first three matter for the model, keep the precondition but check it under a row lock,
then write through the ordinary narrow `save()` — overrides, signals and `auto_now` all run:

```python
def save_input_fields(instance, fields, base_version=None):
    with transaction.atomic():
        if base_version is not None:
            current = (
                type(instance).objects.select_for_update()
                .filter(pk=instance.pk).values_list("updated_at", flat=True).get()
            )
            if current != base_version:
                raise ValidationError(
                    {"updated_at": [ValidationError(_("This record changed since the edit's base version."), code="conflict")]}
                )
        instance.save(update_fields=[*fields, "updated_at"])
```

Same wire shape and the same guarantee — nothing lands between the check and the write while the
row is locked. The lock lasts until the outermost transaction commits. SQLite ignores
`select_for_update()`, so the guarantee holds only on a backend with row locks (PostgreSQL, MySQL,
Oracle).

## 3. Tests

Parametrize nothing here — one behaviour per test, each with its own query and response:

- a matching version applies, and the version advances;
- a stale version returns the `OperationInfo` (`len(messages) == 1`, `kind`/`field`/`message`/`code`)
  and **nothing is written** (assert the row, not just the response);
- no version, and an explicit `null`, both apply;
- the sent version is never written as a field;
- **a write landing between the read and the write** is not overwritten — simulate it by patching the
  step between them (e.g. `full_clean`) to commit another writer's `update()` first;
- a field that wasn't sent keeps a value written meanwhile (this one needs no version);
- with the locked variant, a versioned edit runs the model's `save()` and its signal receivers
  (assert the receiver's effect or the derived column).
