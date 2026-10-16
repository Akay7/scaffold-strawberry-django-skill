# scaffold-strawberry-django

An [Agent Skill](https://agentskills.io/specification) that scaffolds the GraphQL layer for a
Django model using [Strawberry-Django](https://strawberry.rocks/docs/django): a Type + Filter,
Create/Update/Delete Inputs + Mutations, and the schema wiring to connect them.

It's a `SKILL.md` of instructions for an LLM coding agent, not a code generator you run — the
agent reads your existing models and conventions and writes matching code. It encodes a few
non-obvious production gotchas:

- the `handle_django_errors` union-name collision on bare `create`/`update` method names, and a
  parallel enum-name collision on same-named choices classes with `django-choices-field`;
- choice fields are an enum only with `django-choices-field` or `GENERATE_ENUMS_FROM_CHOICES` (off
  by default), and the two name and serialize the enum differently;
- gating each mutation (not just the namespace) makes denial a typed `OperationInfo`;
- delete returns the model's own type with the pk-restore trick, and `ProtectedError` from a
  `PROTECT` FK is not converted automatically;
- one small shared `apply_input` applies create/update inputs (FKs stay `auto`, `full_clean()`
  validates them) instead of per-model field lists or private lookup helpers, and a shared
  `get_by_id` makes a malformed integer id not-found instead of an unconverted `ValueError`;
- updates write only the fields the input set (`save(update_fields=...)`, not a whole-row save that
  reverts a concurrent writer's columns), with an optional `updatedAt` precondition, checked by one
  conditional `UPDATE`, for rows that have several writers;
- `handle_django_errors` covers mutations only — a Django `ValidationError` in a *query* is masked or
  can leak `"['…']"`, so a readable query failure is a `GraphQLError`;
- an error-masking recipe that hides unplanned failures (DB outages, a `ValidationError` carrying
  connection details) but keeps `PermissionDenied` and a user-scoped not-found readable;
- one Type per model, with audience-varying rows in `get_queryset`; update/delete reuse it;
- `annotate=` / named `QuerySet` methods vs. per-row resolvers for computed fields;
- a `null=True` string field without `blank=True` is silently required by `full_clean()`;
- "no existence oracle" error design, pinned by parametrized identical-literal tests.

The schema/limiter setup, concurrency, authorization-and-leak patterns and test conventions live in
`skills/scaffold-strawberry-django/references/` and are read on demand, outside `SKILL.md`.

It assumes current releases (strawberry-graphql-django ≥ 0.89, Django 6 / Python 3.12+).

It targets a plain object/list-field API (query fields returning `list[<Type>]` or an
offset-paginated field) — not Relay's **Node interface** (Global Object Identification via a global
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
