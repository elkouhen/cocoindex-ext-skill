# cccr Management

`cccr` depends on three executables for the full workflow:

- `cccr` — findings / endpoints / graph / MCP
- `semgrep` — scan engine used by `cccr index`
- `ccc` — semantic code search used by `cccr search`

## Installation

Install all three when the environment is missing them:

```bash
uv tool install ccc-radar
uv tool install cocoindex-code
pipx install semgrep
```

`cccr search` is the only command that requires `ccc`; `cccr summary`,
`cccr endpoints`, `cccr graph`, `cccr findings`, and `cccr index` do not.

## Project Initialization

For the Java/Spring/Maven audit workflow owned by this skill:

1. Copy the local rule packs into the target repo under `.cccr/rules/`.
2. Run `cccr init` with explicit `--rules` for `default`, `liveness`, `rest`,
   and `kafka`.
3. Run `cccr index`.

Example:

```bash
cccr init \
  --rules .cccr/rules/default/a-memoire-fichiers.yaml \
  --rules .cccr/rules/default/b-kafka.yaml \
  --rules .cccr/rules/liveness/java.yaml \
  --rules .cccr/rules/rest/java.yaml \
  --rules .cccr/rules/kafka/java.yaml
cccr index
```

If `.cccr/config.yml` already exists, do not recreate it silently; reuse it.

## Refreshing the Index

- After code changes: run `cccr index`.
- After asking architecture-level questions: prefer `cccr summary`,
  `cccr endpoints`, `cccr graph`, or `cccr findings` before `cccr search`.
- After major refactors that affect code search too: refresh `cccr index`, then
  use `cccr search`.

## Troubleshooting

- `cccr` missing: install `ccc-radar`.
- `semgrep` missing: install `semgrep`.
- `ccc` missing: install `cocoindex-code`; only blocks `cccr search`.
- `Index absent. Lancez d'abord: cccr index`: initialize/configure the repo,
  then run `cccr index`.
- Embedding signature / dimension mismatch: re-run `cccr index --full`.
