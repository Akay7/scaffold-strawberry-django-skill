# Test-writing conventions

Read this when adding tests for the generated queries/mutations.

**Defer to the repo first.** If the project has a GraphQL-testing skill or a conventions doc
(`docs/graphql.md`), follow it and use only the generic rules below where it is silent. Layout,
client fixture and auth fixtures differ per repo — read one existing GraphQL test before writing.

## Layout and client

- **Layout**: match where the repo's tests live and how its runner is invoked (its
  CLAUDE.md/README "run a single test" example). Some repos keep one flat top-level `tests/`
  directory named per capability (`tests/test_<capability>.py`), others a per-app `tests/`
  package (`<app>/tests/test_<stem>.py`). Don't impose one.
- **Client**: use the repo's shared fixture. Strawberry-Django ships one:
  `strawberry_django.test.client.TestClient`. `res = gql_client.query(QUERY, variables)` returns
  `Response(data, errors, extensions)` (`res.data`, `res.errors`) and asserts there are no top-level
  errors unless called with `assert_no_errors=False`; it also takes `headers=` and
  `files={"file": fileobj}` (with `{"file": None}` in the variables) for multipart uploads.
  **If the repo has no such fixture, add one** to the root `conftest.py` instead of hand-rolling
  `client.post(..., json.dumps(...), content_type=...)` in each file:

  ```python
  from strawberry_django.test.client import TestClient

  @pytest.fixture
  def gql_client(client):
      return TestClient("/graphql/", client=client)  # same Django client: client.force_login(user) applies
  ```

  If the repo type-checks its tests, `TestClient.query()` is declared to return
  `Coroutine[Any, Any, Response] | Response` (it shares the method with the async client), so every
  `res.data` is flagged. Narrow it once in a tiny subclass instead of ignoring it at each call:

  ```python
  class GraphQLTestClient(TestClient):
      def query(self, query, variables=None, headers=None, files=None, assert_no_errors=True) -> Response:
          return cast(Response, super().query(query, variables, headers, files, assert_no_errors))
  ```

  Drop to a raw Django client only for transport concerns (status codes, CSRF, headers).
- Tests that touch the DB need `@pytest.mark.django_db` (or the repo's equivalent fixture).

## Style

- One `@pytest.mark.django_db()` function per case — **no test classes**, even though pytest and
  pytest-django both support (and their own docs often default to) grouping cases in a
  `class Test<Something>:` container. That default is easy to reach for out of habit; the
  convention here is flat, independent functions per file, grouped only by filename.
- Keep all GraphQL document strings as **constants at the top of the file** (queries and
  mutations), then *data-setup* helpers, then the test functions. Never a helper that sends the
  request or unpacks the response (`gql(...)`, `create_x(...)`, `update_x(...)`, `_upload(...)`): each
  test posts its own query and keeps the response in view, or the wrappers drift from file to file and
  hide what is asserted. Repeated defaults belong in a **module constant** merged at the call site —
  `gql_client.query(CREATE, {"data": {**DEFAULTS, "name": "Pear"}})` — and uploads pass their file
  straight in (`files={"file": fileobj}`).
- Always assign the query result to a variable (`res = gql_client.query(...)`) and inspect it —
  never discard it. On success rely on the client's default: don't pass `assert_no_errors=False` and
  then `assert res.errors is None` — that is the default written out twice. Pass
  `assert_no_errors=False` only where an error is *expected*, and inspect `res.errors`.
- Use named local variables for values that appear in both setup and assertions (e.g.
  `initial_name = "Apple"`). If the mutation changes a value, name that too (e.g.
  `new_name = "Pear"`) and assert it in both the response and the DB.
- For `OperationInfo` message assertions: assert `len(messages) == N`, then index into each
  message and check `kind`/`field`/`message`/`code` explicitly. Never `any()`, a `for` loop, or
  other iteration to search for a matching message — an exact count plus exact fields is what
  proves the response can't be hiding an extra or different error.
- **One behavior per test**; name tests for the behavior, not the mechanism.
- Permissions: grant object-level access with `assign_perm(PERM, user, obj)` (django-guardian) if
  the project uses it. If it scopes access some other way (`is_staff`, a role/group field, a
  membership table), match the existing permission-gated tests — check one before writing a new one.

## Shapes to assert

**With `handle_django_errors` on** (the default this skill assumes — including delete): the mutation
field returns a union, so every create/update/delete selection needs `__typename` plus both
`... on <ModelName>Type { ... }` and `... on OperationInfo { messages { kind field message code } }`
inline fragments.

- Success → no `errors`, and `__typename == "<ModelName>Type"`.
- Validation failure → still no `errors`; `__typename == "OperationInfo"`, `len(messages) == 1`,
  then check `field` (and `kind`/`message`/`code` if the test cares).
- **Auth failure** on a mutation gated per-mutation with the shared extension (SKILL.md step 3) →
  also no `errors`: an `OperationInfo` with `kind == "PERMISSION"` and `code == "UNAUTHENTICATED"`,
  and the resolver never ran (assert the DB is unchanged). Only if the repo gates the *namespace*
  wrapper instead is denial a top-level `errors` entry (with `extensions.code ==
  "UNAUTHENTICATED"`) and a null namespace.
- Not-found, and a malformed id (integer or UUID pk), on update/delete → the same
  `OperationInfo` (`kind: ERROR`), not a raised error.
- Not-yours on update/delete: when callers can see rows they may not change (`mutations.md`, write
  scope), the refused caller is one who **can see** the row (a teammate), not only a stranger. A
  stranger is refused even if the mutation wrongly looks the row up in the read scope, so only the
  teammate case catches that; assert the `OperationInfo` and that the row is unchanged.
- Delete of a row a `PROTECT` FK points at → `OperationInfo` on the relation's message, row still
  present.
- A to-many input (`sync_m2m`), one test per behavior: `set` replaces the list; `add` and
  `remove` each keep the rest; both in one request apply both; an omitted field changes nothing —
  on create too, where it means "none". `set` with `add`/`remove`, and an unknown id, are each one
  `OperationInfo` (`kind: VALIDATION`, `field` = the input field) with the relation unchanged —
  and on create, no row left behind. Compare a multi-row list as a set of ids unless the related
  model has a default ordering.
- A query field denied by `IsAuthenticated`: an `OffsetPaginated` field returns an empty page
  (`totalCount: 0`, `results: []`), a plain list field `[]` and an optional field `null`, all with
  **no** `errors`; only a non-null, non-list field gives a top-level `errors`
  entry (and a null parent), with `extensions.code == "UNAUTHENTICATED"`.
- The code is the contract, so pin it, not the message: one refused document per gate shape,
  parametrized over every language the app serves (`headers={"Accept-Language": …}`), each asserting
  the code; and one signed-in failure asserting the code is **absent**. Single-object query with an unknown or malformed id → `null`, no
  `errors`.

**Without it** (a mutation that opts out with `handle_django_errors=False` for its own reason):
success → fields directly on `data["<model>"]["createFruit"]`; not-found/validation/auth failure →
`errors`.

## Pinning "no existence oracle"

Parametrize over every branch that must be indistinguishable (e.g. `id_exists` real vs `uuid4()`,
plus "malformed id") and have each case assert the *same literal* response dict — `__typename`,
`messages` with kind/field/message/code. See `authorization-and-leaks.md`.
