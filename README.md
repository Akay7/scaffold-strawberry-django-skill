# scaffold-strawberry-django

An [Agent Skill](https://agentskills.io/specification) that scaffolds the GraphQL layer for a
Django model using [Strawberry-Django](https://strawberry.rocks/docs/django): a Type + Filter,
Create/Update/Delete Inputs + Mutations, and the schema wiring to connect them.

It's a `SKILL.md` of instructions for an LLM coding agent, not a code generator you run — the
agent reads your existing models and conventions and writes matching code. It encodes a few
non-obvious production gotchas: the `handle_django_errors` union-name collision on bare
`create`/`update` method names, a parallel enum-name collision on same-named `TextChoices` classes,
`annotate=` vs. per-row resolvers for computed fields, the delete-mutation pk-restore trick, and a
no-leak-guarantee test pattern for authorization-sensitive relation inputs.

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

Copy or symlink `scaffold-strawberry-django/` into your tool's skills directory, e.g.
`.claude/skills/` (Claude Code), `.opencode/skills/` (OpenCode), or `.agents/skills/` (generic) —
project-local or the equivalent `~/.config/...` / `~/.claude/...` global path.

```bash
cp -r scaffold-strawberry-django /path/to/your/project/.claude/skills/
# or, globally, to track updates from this repo's clone:
ln -s "$(pwd)/scaffold-strawberry-django" ~/.claude/skills/scaffold-strawberry-django       # Claude Code, supported by OpenCode
ln -s "$(pwd)/scaffold-strawberry-django" ~/.opencode/skills/scaffold-strawberry-django  # OpenCode
```

## License

MIT — see [LICENSE](LICENSE).
