# scaffold-strawberry-django

An [Agent Skill](https://agentskills.io/specification) that scaffolds the GraphQL layer for a
Django model using [Strawberry-Django](https://strawberry.rocks/docs/django): a Type + Filter,
Create/Update/Delete Inputs + Mutations, and the schema wiring to connect them.

It's a `SKILL.md` of instructions for an LLM coding agent, not a code generator you run — the
agent reads your existing models and conventions and writes matching code. It encodes the
non-obvious production gotchas:

**Reading**

- one Type per model, with who-sees-which-rows in its `get_queryset`; every Type that other rows
  point at filters its own rows, and a relation to a scoped Type is `| None` — otherwise one hidden
  related row nulls the whole response;
- list fields are `OffsetPaginated`: a plain `list[...]`, nested ones included, returns every row
  unless the client sends `pagination`;
- the single-object query returns a QuerySet, not `.first()` — a bare instance skips the
  optimizer, so `annotate=` fields crash and a nested Type's scope is bypassed;
- `annotate=` / named `QuerySet` methods vs. per-row resolvers for computed fields;
- choice fields are an enum only with `django-choices-field` or `GENERATE_ENUMS_FROM_CHOICES` (off
  by default); the two name and serialize the enum differently, the generated one must be written
  back by `.value`, and same-named choices classes collide on one enum name.

**Writing**

- small shared helpers instead of per-model code: `apply_input` (FKs stay `auto` and
  `full_clean()` validates them; `field_name=` renames honoured), `sync_m2m` (`set`, or
  `add`/`remove`; unknown ids refused before anything is written), `get_by_id` (a malformed id is
  not-found, not an unconverted `ValueError`) and `reread` (create/update payloads as stored,
  loaded through the optimizer);
- update/delete look the row up in what the caller may *change*, which can be narrower than what
  they may see;
- updates write only the fields sent (`save(update_fields=...)`), with an optional `updatedAt`
  precondition, checked by one conditional `UPDATE`, for rows with several writers — and the
  limits of that `UPDATE` (no `save()` override, no signals);
- a create that is safe to retry from an offline queue, keyed by the client's UUID, and what that
  exposes;
- the `handle_django_errors` union-name collision on bare `create`/`update` method names;
- delete returns the model's own type with the pk-restore trick, and `ProtectedError` from a
  `PROTECT` FK is not converted automatically;
- a `null=True` string field without `blank=True` is silently required by `full_clean()`.

**Errors and access**

- gating each mutation (not just the namespace) makes denial a typed `OperationInfo`, and every
  not-signed-in refusal carries the code `UNAUTHENTICATED` — clients match the code, never the
  translatable message;
- `handle_django_errors` covers mutations only — a Django `ValidationError` in a *query* is masked
  or can leak `"['…']"`, so a readable query failure is a `GraphQLError`;
- an error-masking recipe that hides unplanned failures (DB outages, a `ValidationError` carrying
  connection details) but keeps refusals and a user-scoped not-found readable, and error logging by
  cause — faults at ERROR with their traceback, clients' mistakes at INFO, no query arguments in
  the log;
- introspection off outside `DEBUG`, decided per request so tests can switch it;
- "no existence oracle" error design, pinned by parametrized identical-literal tests.

`SKILL.md` holds the workflow, the mutation template and a rules checklist. The long recipes live
in `skills/scaffold-strawberry-django/references/` and are read on demand: the shared helper code,
Type/Filter design (relations, computed fields, choice fields), mutation details, the
schema/limiter setup with error masking and logging, concurrency, authorization-and-leak patterns
and test conventions.

It assumes current releases (strawberry-graphql-django ≥ 0.89, Django 6 / Python 3.12+); its
recipes were checked against strawberry-graphql-django 0.93 and Django 6.1.

It targets a plain object/list-field API (offset-paginated list fields, or a plain list for small
fixed sets) — not Relay's **Node interface** (Global Object Identification via a global
`node(id: ID!)` root field) or cursor-based **Connections** (`edges`/`pageInfo`). If your project's
GraphQL layer is Relay-routed, its patterns need translating to that shape rather than applying
as-is.

## Open standard

Written to the [Agent Skills specification](https://agentskills.io/specification) — YAML
frontmatter (`name`, `description`) plus a Markdown instruction body, supported by Claude Code,
Cursor, Codex CLI, OpenCode, Gemini CLI/Antigravity, JetBrains Junie, AWS Kiro, Block's Goose, and
others, under Linux Foundation AAIF governance.

The `/scaffold-strawberry-django <ModelName>` invocation syntax is a Claude Code slash-command
example — other agents match the `description` field against user intent instead. The
frontmatter's `allowed-tools` field is a Claude Code extension; other agents ignore it.

## Installing

### Via gh tool (recommended)

```bash
gh skill install Akay7/scaffold-strawberry-django-skill
# or pin version
gh skill install Akay7/scaffold-strawberry-django-skill scaffold-strawberry-django --pin v1.0.0
```

### Manual

Clone or download this repo.

Copy or symlink `skills/scaffold-strawberry-django/` into your tool's skills directory, e.g.
`.claude/skills/` (Claude Code), `.opencode/skills/` (OpenCode), or `.agents/skills/` (generic) —
project-local or the equivalent `~/.config/...` / `~/.claude/...` global path.

```bash
cp -r skills/scaffold-strawberry-django /path/to/your/project/.claude/skills/
# or, globally, to track updates from this repo's clone:
ln -s "$(pwd)/skills/scaffold-strawberry-django" ~/.claude/skills/scaffold-strawberry-django       # Claude Code, supported by OpenCode
ln -s "$(pwd)/skills/scaffold-strawberry-django" ~/.opencode/skills/scaffold-strawberry-django  # OpenCode
```

## License

MIT — see [LICENSE](LICENSE).
