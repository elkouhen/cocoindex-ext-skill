---
name: cccf
description: "This skill should be used for Java/Spring/Maven microservice audits with cccf: initializing the repo with the right Semgrep packs, indexing findings plus REST/Kafka endpoints, inspecting summary/endpoints/graph/findings, or running code search via ccc when needed. Trigger phrases include 'audit', 'microservice', 'Kafka', 'REST', 'cccf', 'Semgrep', 'finding', 'endpoint', 'graph', and 'search the codebase'."
---

> Adapted from cocoindex-code's own skill
> ([`skills/ccc/SKILL.md`](https://github.com/cocoindex-io/cocoindex-code/blob/main/skills/ccc/SKILL.md),
> Apache-2.0), renamed to `cccf` and extended to cover Semgrep findings.

# cccf - Microservice Audit, Indexing & Findings

`cccf` is the audit CLI layered on top of Semgrep and, for code search only,
`ccc` (CocoIndex Code). For the target stack of this skill — **Java + Spring +
Maven microservices** — it indexes:

- Semgrep findings (`cccf summary`, `cccf findings`)
- REST and Kafka endpoints (`cccf endpoints`)
- derived blocking signals such as outbound REST calls inside Kafka consumers
  (`cccf graph`)
- code search results enriched with findings (`cccf search`, via `ccc`)

## Ownership

The agent owns the `cccf` lifecycle for the current project — initialization, indexing, and searching. Do not ask the user to perform these steps; handle them automatically.

- **Initialization**: If `cccf search` or `cccf index` fails with an initialization error (e.g., "Not in an initialized project directory"), run `cccf init` from the project root directory (see **Default Rules** below for which `--rules` to pass), then `cccf index` to build the index, then retry the original command.
- **Index freshness**: Keep the index up to date by running `cccf index` (or `cccf search --refresh`) when the index may be stale — e.g., at the start of a session, or after making significant code changes (new files, refactors, renamed modules). There is no need to re-index between consecutive searches if no code was changed in between.
- **Installation**: If `cccf` itself is not found (command not found), refer to [management.md](references/management.md) for installation instructions and inform the user.

### Default Rules for Java Microservice Audits

This skill bundles five Semgrep rule packs:

- `default` ([`rules/default/`](rules/default/)) — Java rules for bounded,
  streaming-safe handling of files and Kafka events: bounded streaming for
  files (`a-memoire-fichiers.yaml`, R1-R3) and Kafka claim-check/delivery
  guarantees (`b-kafka.yaml`, R5-R10). The prose behind each rule ID
  (including the design-only rules not enforced by a Semgrep pattern — R4,
  R9, R11) lives in
  [`rules/default/design-rules.md`](rules/default/design-rules.md), which
  travels with the pack when copied into a target repo so the `reference:
  design-rules.md#rN` in each rule's metadata keeps resolving.
- `liveness` ([`rules/liveness/`](rules/liveness/)) — cross-cutting rules
  for distributed-system blocking points in a REST + Kafka microservices
  landscape (see `ccc-findings`'s `archive/BACKLOG-10.md` K8): missing HTTP
  timeouts, blocking waits without a timeout, a synchronous REST call
  inside a Kafka consumer handler, and a network call held under a lock —
  Java/Spring only (`java.yaml`: `RestTemplate`, `@KafkaListener`,
  `synchronized`), matching the target stack (Java + Spring + Maven).
- `rest` ([`rules/rest/`](rules/rest/)) — REST endpoint inventory rules for
  Spring controllers and `RestTemplate` client calls. These do **not** create
  findings; they populate `cccf endpoints` / `cccf graph`.
- `kafka` ([`rules/kafka/`](rules/kafka/)) — Kafka endpoint inventory rules
  for `@KafkaListener`, `KafkaTemplate.send(...)`, and `ProducerRecord(...)`.
  These also feed `cccf endpoints` / `cccf graph`, not the findings list.
- `kafka-security` ([`rules/kafka-security/`](rules/kafka-security/)) —
  Kafka security findings (see `ccc-findings`'s `archive/BACKLOG-10.md`
  K8, security volet): SASL credentials hardcoded in source,
  `security.protocol` set to `PLAINTEXT`, a `JsonDeserializer` trusting all
  packages (arbitrary class instantiation from an untrusted message), and
  raw Java deserialization (`ObjectInputStream.readObject`) — Java/Spring
  only.

Run all five packs **by default** on `cccf init`, unless the user
explicitly asks for a different rule set:

1. If `.cccf/config.yml` doesn't exist yet, copy the pack directories
   (`rules/default/`, `rules/liveness/`, `rules/rest/`, `rules/kafka/`,
   `rules/kafka-security/`) into the target repo (e.g.
   `.cccf/rules/default/`, `.cccf/rules/liveness/`, `.cccf/rules/rest/`,
   `.cccf/rules/kafka/`, `.cccf/rules/kafka-security/`) — never pass an
   absolute path back into the skill's own directory, since Semgrep derives
   rule identity from the `--config` path and an absolute path outside the
   repo breaks reproducibility across machines/checkouts.
2. Run `cccf init` with one `--rules` per copied file:
   ```bash
   cccf init \
     --rules .cccf/rules/default/a-memoire-fichiers.yaml \
     --rules .cccf/rules/default/b-kafka.yaml \
     --rules .cccf/rules/liveness/java.yaml \
     --rules .cccf/rules/rest/java.yaml \
     --rules .cccf/rules/kafka/java.yaml \
     --rules .cccf/rules/kafka-security/java.yaml
   ```
   This takes priority over `cccf init`'s own auto-detection (local
   `.semgrep.yml`/`semgrep.yml`/`.semgrep`, or the `p/security-audit`
   registry fallback) — explicit `--rules` always wins.
3. If the user names additional or different rules, merge them in with more
   `--rules` flags rather than dropping the default packs, unless the user
   asks to replace them.
4. If `.cccf/config.yml` already exists, leave it as-is (`cccf init` refuses
   to overwrite it) — the default packs only apply to fresh initialization.

## Audit Workflow

For microservice audits, prefer the following sequence:

1. `cccf summary` — quick posture: severity distribution and hot rules.
2. `cccf endpoints` — inspect the static REST/Kafka inventory.
3. `cccf graph` — look for outbound REST calls inside Kafka consumers first;
   use it before diving into individual findings when the symptom is
   distributed blocking or intermittent lock-up.
4. `cccf findings <query>` — search vulnerabilities / technical debt by
   description or rule.
5. `cccf search <query>` — only when you need semantic code search; this is
   the part that depends on `ccc`.

Do not jump straight to `cccf search` for every audit question: `summary`,
`endpoints`, `graph`, and `findings` are cheaper and more directly aligned with
architecture review.

## Searching the Codebase

To perform a semantic search:

```bash
cccf search <query terms>
```

The query should describe the concept, functionality, or behavior to find, not exact code syntax. For example:

```bash
cccf search database connection pooling
cccf search user authentication flow
cccf search error handling retry logic
```

### Filtering Results

- **By language** (`--lang`, repeatable): restrict results to specific languages.

  ```bash
  cccf search --lang python --lang markdown database schema
  ```

- **By path** (`--path`): restrict results to a glob pattern relative to project root. If omitted, defaults to the current working directory (only results under that subdirectory are returned).

  ```bash
  cccf search --path 'src/api/*' request validation
  ```

### Pagination

Results default to the first page. To retrieve additional results:

```bash
cccf search --offset 5 --limit 5 database schema
```

If all returned results look relevant, use `--offset` to fetch the next page — there are likely more useful matches beyond the first page.

### Working with Search Results

Search results include file paths and line ranges. To explore a result in more detail:

- Use the editor's built-in file reading capabilities (e.g., the `Read` tool) to load the matched file and read lines around the returned range for full context.
- When working in a terminal without a file-reading tool, use `sed -n '<start>,<end>p' <file>` to extract a specific line range.

## Searching Findings

`cccf search` (above) enriches code results with any Semgrep findings that
overlap them. To search the indexed findings **on their own** — in natural
language, without a code search — use `cccf findings` instead:

```bash
cccf findings <query terms>
```

Use this when the question is about a vulnerability, security issue, piece of
technical debt, or a specific rule/finding, rather than about code semantics.

### Filtering Results

- **By severity** (`--severity`): keep only findings at or above this level (`INFO` / `WARNING` / `ERROR`).
- **By rule** (`--rule`): keep only findings matching this exact `rule_id`.
- **By path** (`--path`): restrict to a glob pattern, same style as `cccf search --path`.

```bash
cccf findings sql injection --severity ERROR --path 'app/*'
```

### Pagination and Output

- `--limit` / `--offset` paginate results, same as `cccf search` (default limit 5).
- `--context` adds 5 lines of surrounding code (before/after) to each finding, when the source file is still readable.
- `--json` returns the stable `FindingHit` schema instead of the text render — this is also the shape returned by the MCP tool `search_findings`.

### Aggregated View

For a broad, low-cost overview instead of a targeted search (e.g. "what's our
security posture", "any critical findings"), prefer:

```bash
cccf summary
```

This returns totals by severity, the top 10 rules by count, and counts by
top-level directory — much cheaper than a full `cccf findings` search when the
question isn't about a specific file, rule, or vulnerability class. Add
`--json` for structured output (matches the MCP tool `findings_summary`).

### Choosing the Right Tool

Prefer the least expensive query that answers the question, escalating only
as needed:

1. **Architecture overview** — `cccf summary`, then `cccf endpoints`.
2. **Distributed blocking suspicion** — `cccf graph`.
3. **Targeted findings lookup** — `cccf findings <query>`.
4. **Code + findings** — `cccf search <query>` when the question is primarily about code semantics.

If `.cccf/findings.db` doesn't exist yet, both `cccf findings` and `cccf summary` exit with `Index absent. Lancez d'abord: cccf index` (exit code 2) — run `cccf index` and retry, same as the index-freshness rule under Ownership above.

## Settings

To view or edit embedding model configuration, include/exclude patterns, or language overrides, see [settings.md](references/settings.md).

## Management & Troubleshooting

For installation, initialization, daemon management, troubleshooting, and cleanup commands, see [management.md](references/management.md).
