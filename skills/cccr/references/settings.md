# cccr Settings

Two configuration layers matter in practice:

## 1. Project audit config: `.cccr/config.yml`

This is the file created by `cccr init` and consumed by `cccr index`.

```yaml
rules:
  - .cccr/rules/default/a-memoire-fichiers.yaml
  - .cccr/rules/default/b-kafka.yaml
  - .cccr/rules/liveness/java.yaml
  - .cccr/rules/rest/java.yaml
  - .cccr/rules/kafka/java.yaml
include:
  - "**/*"
exclude:
  - ".git/**"
  - ".venv/**"
  - "node_modules/**"
  - ".cccr/**"
min_severity: INFO
embedding_model: Snowflake/snowflake-arctic-embed-xs
semgrep_timeout_s: 120
```

Relevant fields for this skill:

- `rules`: must include the copied local packs for the microservice audit
  workflow if you want `cccr endpoints` / `cccr graph` to work.
- `include` / `exclude`: file perimeter for indexing.
- `min_severity`: applies to findings, not to endpoint-inventory rules.
- `embedding_model`: used by `cccr findings` and the local findings index.

After editing `.cccr/config.yml`, run `cccr index`.

## 2. ccc user-level config

`cccr search` relies on `ccc`, which keeps its own settings outside the
repository (for example `~/.cocoindex_code/global_settings.yml` for embedding
provider/model). Those settings matter only for the semantic code-search part.

If the question is about architecture inventory, Kafka/REST coupling, or known
findings, you can often avoid touching `ccc` settings entirely by using:

1. `cccr summary`
2. `cccr endpoints`
3. `cccr graph`
4. `cccr findings`

Use `cccr search` only when you truly need semantic code retrieval.
