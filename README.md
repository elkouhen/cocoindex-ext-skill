# ccc-findings-skill

Claude Code skill for `cccf` (CocoIndex Code + Semgrep findings): semantic
code search, index management, and Semgrep findings lookup, driven
automatically by the agent.

- [`skills/cccf/SKILL.md`](skills/cccf/SKILL.md) — the skill itself:
  ownership rules, searching, filtering, pagination.
- [`skills/cccf/references/settings.md`](skills/cccf/references/settings.md) —
  embedding model, include/exclude patterns, language overrides.
- [`skills/cccf/references/management.md`](skills/cccf/references/management.md) —
  installation, initialization, daemon management, troubleshooting.

## Related projects

- [`ccc-findings`](https://github.com/elkouhen/ccc-findings) (`cccf`) — the
  CLI and MCP server this skill drives. It indexes Semgrep findings locally
  and joins them to code search results from `ccc`.
- [`cocoindex-code`](https://github.com/cocoindex-io/cocoindex-code) (`ccc`)
  — the underlying AST-based semantic code search tool that `cccf` extends
  as a companion package (no fork, no internal import — see `ccc-findings`'s
  ADR-1).

## License

[Apache License 2.0](LICENSE), matching the upstream
[`cocoindex-code`](https://github.com/cocoindex-io/cocoindex-code) project.
