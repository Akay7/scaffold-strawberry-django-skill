# Defining the `schema` object, extensions and limiter settings

Read this when the repo has no `schema` yet, or its extension list is missing one of the guards
below. Otherwise you are only adding fields to an existing `Query` / `Mutation`.

## The `schema` object

There is a single module-level variable named **`schema`** — an instance of `strawberry.Schema`
built from the `Query` and `Mutation` classes. It must already exist (you're adding fields to its
`Query`/`Mutation`, not creating it). If the repo has no schema yet, define it at the bottom of the
same module, after the `Query` and `Mutation` classes:

```python
import strawberry
from django.conf import settings
from graphql.validation import NoSchemaIntrospectionCustomRule
from strawberry.extensions import (
    AddValidationRules,
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
    # dev keeps GraphiQL/introspection working. Decided per request from the live
    # setting, not once at import, so a test that changes DEBUG gets the other answer.
    # (The same rule `DisableIntrospection` adds, minus the rule when DEBUG is on.)
    lambda: AddValidationRules([] if settings.DEBUG else [NoSchemaIntrospectionCustomRule]),
]

schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    # Field names default to camelCase (don't disable auto_camel_case); datetimes
    # are ISO 8601 via the built-in DateTime scalar.
    extensions=extensions,
)
```

Type the list when the repo runs a type checker:
`extensions: list[type[SchemaExtension] | Callable[[], SchemaExtension]] = [...]`
(`SchemaExtension` from `strawberry.extensions`).

The variable **must be called `schema`** — Django/Strawberry config points at it by dotted path
(e.g. the `GraphQLView`'s `schema=`, typically `"<module>.schema:schema"`).

## The four base extensions are required, plus the introspection gate

- `DjangoOptimizerExtension` — makes the "no manual `select_related`/`prefetch_related`" rule work.
- `MaxTokensLimiter`, `MaxAliasesLimiter`, `QueryDepthLimiter` — abuse/DoS guards against oversized,
  alias-bombed, or deeply-nested queries. Never drop them. (The depth limiter exempts the
  introspection fields `__schema` / `__type`.)
- The introspection gate, active whenever `settings.DEBUG` is off — introspection lets a caller read
  the entire schema (every type, field, and enum value) without any query of their own; disabling it
  in production removes that reconnaissance surface while leaving GraphiQL/introspection-based
  tooling working in dev. Any product feature that needs client-visible reference data (e.g. an
  enum's valid values) must ship as its own explicit query field rather than relying on
  introspection to expose it. Gate it the same way the repo gates other dev-only surfaces (check
  `settings.py` for an existing `DEBUG`-gated toggle, e.g. a debug toolbar, and mirror that
  condition rather than inventing a new one).
  - **Decide it per request**, inside the factory callable, as above. The common
    `*([] if settings.DEBUG else [DisableIntrospection])` reads `DEBUG` once, when the schema module
    is imported: the answer is then fixed for the process, a test's `settings.DEBUG = …` (or
    `override_settings`) changes nothing, and a schema imported under the wrong setting stays that
    way.
  - Pin both sides with tests: `DEBUG = False` → `{ __schema { queryType { name } } }` is refused
    ("GraphQL introspection has been disabled…") and `data` is `null`; `DEBUG = True` → it answers.
    Run the `DEBUG = True` one on `schema.execute_sync` rather than over HTTP if turning `DEBUG` on
    also switches on dev-only middleware whose URLs aren't registered in tests (a debug toolbar).

## Limiter values via environment variables

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

## Mutation error handling setting

```python
STRAWBERRY_DJANGO = {"MUTATIONS_DEFAULT_HANDLE_ERRORS": True}
```

Check whether it is already set before assuming either behaviour (`grep -rn
MUTATIONS_DEFAULT_HANDLE_ERRORS .`). If the repo hasn't adopted it, follow the repo's existing
pattern instead of turning it on as a side effect of scaffolding one model.

## Choice-field enum setting

```python
STRAWBERRY_DJANGO = {..., "GENERATE_ENUMS_FROM_CHOICES": True}
```

Off by default: a plain `CharField(choices=...)` annotated `auto` is then a `String`. On, it becomes
an enum named `<App><Model><Field>Enum`, serialized by value. `django-choices-field` fields are
enums either way. Like the error-handling setting, check it (`grep -rn GENERATE_ENUMS_FROM_CHOICES
.`) and never flip it while scaffolding — it changes the wire type of every existing choice field.

## Masking unexpected errors (optional, recommended)

Strawberry serialises an exception's text straight into `errors[].message`, so a database outage
leaks its host, port and role to the client. `MaskErrors` replaces those with one generic message
while the original is still logged, and lets deliberate errors through:

```python
from django.core.exceptions import ObjectDoesNotExist, PermissionDenied
from graphql import GraphQLError
from strawberry.extensions import MaskErrors

def should_mask_error(error: GraphQLError) -> bool:
    """True when the error came from an exception no resolver raised on purpose."""
    original = error.original_error
    return original is not None and not isinstance(
        original, GraphQLError | PermissionDenied | ObjectDoesNotExist
    )

extensions = [
    ...,
    lambda: MaskErrors(should_mask_error=should_mask_error, error_message="Unexpected error."),
]
```

Errors with no `original_error` (syntax, validation, limiter rejections) stay readable. A mutation's
`ValidationError` / `PermissionDenied` / `ObjectDoesNotExist` never reach the masker, because
`handle_django_errors` turns them into `OperationInfo` first; from a **query** each Django exception
gets its own rule:

- **`PermissionDenied` — readable.** It is what `IsStaff` and other permission checks raise, with a
  fixed message that carries nothing sensitive. The shared `IsAuthenticated` (`shared-modules.md`,
  `permissions.py`) raises a `GraphQLError` with `extensions.code: "UNAUTHENTICATED"` instead, which
  the masker passes too. Clients tell "session gone, stop" from "this call failed" by that code —
  masked, a client retrying a batch records every remaining item as failed instead of stopping.
- **`ObjectDoesNotExist` — readable.** Not-found is a deliberate answer, not a server fault. The
  anti-enumeration guarantee comes from the **user-scoped lookup** (`.get(user=…, pk=…)` or the
  Type's `get_queryset`), which raises the same "`<Model>` matching query does not exist." for a
  missing row and for another user's row — masking adds nothing to that, and a lookup that isn't
  scoped already leaks the row on success. The message does name the model; if that is unwanted,
  raise `GraphQLError("Not found.")` in the resolver (or return `null`, as the single-object query
  does).
- **`ValidationError` — masked.** A deliberate one belongs in a mutation, where it is already an
  `OperationInfo`. One reaching the masker from a query may have been built from a database or
  downstream failure (a validator that queries, a wrapped exception) and carry its host, port and
  role. A query that wants its message shown raises `GraphQLError`.

Pin each rule with a test: an `OperationalError` and a `ValidationError` built from a realistic
connection-failure message both come back as "Unexpected error." with no host/role/password in the
raw response; an unauthenticated non-null field keeps "User is not authenticated." and
`extensions.code == "UNAUTHENTICATED"`; and a
user-scoped lookup gives an **identical** not-found message for another user's row and for a random
id. Don't gate masking on `DEBUG`: the behaviour must be exercised before production.


## Logging errors by cause (optional, recommended)

Strawberry logs **every** GraphQL error at ERROR on the `strawberry.execution` logger: a client's
syntax error, a limiter rejection, a deliberate `GraphQLError`, a refused permission — next to the
real faults, so an alert on ERROR fires on clients' mistakes. And it logs `str(error)`, which
appends an excerpt of the query document to the message, inline arguments included:
`lookup(secret: "hunter2")` puts `hunter2` in the log.

Override `process_errors` on a `strawberry.Schema` subclass and split by the masking rule above, so
an error is logged as a fault exactly when its cause was hidden from the client — the operator
gets the traceback the client was refused:

```python
import logging

import strawberry
from graphql import GraphQLError
from strawberry.types import ExecutionContext


class Schema(strawberry.Schema):
    """Log server faults at ERROR with their traceback, everything else at INFO."""

    def process_errors(
        self,
        errors: list[GraphQLError],
        execution_context: ExecutionContext | None = None,
    ) -> None:
        logger = logging.getLogger("strawberry.execution")
        for error in errors:
            # `error.message`, never `str(error)`: that appends an excerpt of the query
            # document — the client's inline arguments included — to every record.
            extra = {"graphql_path": ".".join(map(str, error.path))} if error.path else {}
            if should_mask_error(error):  # the masking rule: a fault the client never saw
                logger.error("%s", error.message, exc_info=error.original_error, extra=extra)
            else:  # the client's own request: syntax, limits, refusals, deliberate errors
                logger.info("%s", error.message, extra=extra)


schema = Schema(query=Query, mutation=Mutation, extensions=extensions)
```

- The class replaces `strawberry.Schema` in the `schema = …` line at the top of this file; nothing
  else changes.
- `graphql_path` (`fruit.createFruit`) says which field failed without logging the document.
  Structured-logging setups pick it up as a field; a plain formatter needs `%(graphql_path)s` (or
  ignores it).
- A fault's message still goes to the log — that is the point: a connection error names its host
  and role there, where the operator needs them, while the client got "Unexpected error.".
- Pin it with `caplog`: a database failure is one ERROR record with `exc_info`; a deliberate
  `GraphQLError` and an unknown field are INFO; and no record of a query with an inline argument
  contains that argument's value.
