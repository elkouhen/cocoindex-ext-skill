---
name: cccr
description: "Use this skill for Java/Spring architecture exploration with cccr: initialize and index a project, inspect microservices, Kafka topics, HTTP APIs, MongoDB collections and modules, analyze dependencies, audit Semgrep findings, export architecture graphs, or use semantic code search through ccc. Trigger phrases include 'audit', 'microservice', 'Kafka', 'REST', 'MongoDB', 'cccr', 'Semgrep', 'finding', 'architecture graph', and 'search the codebase'."
---

# cccr Architecture Explorer

`cccr` indexes a Java/Spring project into architecture objects and Semgrep
findings. Use its object commands before retrieving code: they describe
microservices, modules, HTTP APIs, Kafka topics, MongoDB collections, and the
relationships between them. `ccc` (CocoIndex Code) is optional and only needed
for semantic code search.

## Documentation Language

Write and update user-facing documentation, generated report text, and examples
in English. Preserve exact CLI output, source-code snippets, and user-provided
text in their original language when quoting them.

## Ownership

The agent owns the `cccr` lifecycle for the current project. Do not ask the
user to initialize or index it manually.

- If an architecture command reports that the project is not initialized, run
  `cccr init`, then `cccr index`, and retry the command.
- Run `cccr index` after relevant source changes, renamed modules, or a major
  refactor. There is no need to re-index between read-only queries.
- Before drawing architectural conclusions, run `cccr doctor`. Missing REST or
  Kafka packs invalidate the topology; missing `ccc` only disables `search`.
- If `cccr` is unavailable, follow
  [management.md](references/management.md).

## Default Rules

This skill bundles five Java/Spring Semgrep packs:

- `default` - bounded file handling and Kafka delivery design checks.
- `liveness` - HTTP timeouts, blocking waits, synchronous calls in consumers,
  and network calls held under locks.
- `rest` - HTTP API inventory; it feeds architecture indexing rather than the
  findings list.
- `kafka` - Kafka producer and consumer inventory; it feeds architecture
  indexing rather than the findings list.
- `kafka-security` - insecure Kafka settings and unsafe deserialization.

For a fresh Java/Spring project, set `CCCR_RULES_ROOT` to this skill's
`rules` directory, then initialize and verify the project:

```bash
export CCCR_RULES_ROOT="/path/to/ccc-radar-skill/skills/cccr/rules"
cccr init
cccr doctor
cccr index
```

`cccr init` copies the packs into the target project and records their source
hashes, so the audit remains reproducible after this skill is upgraded. The
command also enables the valid Semgrep registry rulesets `p/security-audit`,
`p/java`, `p/owasp-top-ten`, and `p/secrets`. Do not add `p/spring`: it is not
a valid Semgrep registry ruleset.

If `.cccr/config.yml` already exists, do not overwrite it. Retain its explicit
rules and add the required local packs when they are absent.

## Architecture Workflow

Start with the indexed architecture, then move toward source only when needed:

1. `cccr microservices` - discover services and their main integrations.
2. `cccr microservices show <name>` - understand one service without reading
   configuration or source files.
3. Explore its linked objects with `cccr microservices topics <name>`,
   `cccr microservices apis <name>`, and `cccr microservices mongodb <name>`.
4. Explore a shared object from the other direction with `cccr topics`,
   `cccr apis`, or `cccr mongodb`.
5. Use `cccr analyze audit` before manually interpreting a dense graph.
6. Use `cccr analyze microservices impact <name>` or `path <source> <target>`
   for impact and dependency-path questions.
7. Use `cccr search <query>` only when the answer requires code semantics.

The default output is intentionally concise and never dumps application files.
Use `implementation`, `properties`, or `openapi` only when implementation or a
contract is explicitly required.

### Object Commands

Each object family supports `list`, `show`, `neighbors`, and `search` where
applicable. Navigate through relationships instead of file paths:

```bash
cccr microservices show order-service
cccr microservices topics order-service
cccr topics consumers orders.created
cccr apis consumers 'POST /payments'
cccr mongodb services orders
cccr modules show shared-domain
cccr modules graph
```

`cccr topics show <topic>` and microservice summaries include Java payload
types for Kafka messages when they are statically explicit. Treat a missing
type as unknown: do not derive it from a topic name or serializer setting.

When a project uses `getTopics().getXxx()` for producers and
`${kafka.topics.xxx.name}` in `@KafkaListener` annotations, index it with:

```bash
cccr index --topic-strategy strategy1
```

This opt-in manual-engine strategy maps `getAbcDefGhiJkl()` and `abc_def_ghi_jkl`
to physical topic `ABC_DEF_GHI_JKL`, replaces standard extraction at the covered
source locations, and triggers a full refresh when the strategy changes. In a
`Rest*Config*` class it also recognizes configured HTTP dependencies: uppercase
domain constants name an indexed peer, while
`getRest().get("xxx")` adds `xxx` as an external
microservice linked by HTTP; visual exports render it as a triangle. It is
unavailable with `--engine cocoindex`.

Use `cccr integrations` for a compact project-wide HTTP/Kafka inventory. Use
`cccr graph` for the microservice interaction graph; it is distinct from
module build dependencies.

### Analysis and Exports

```bash
cccr analyze audit
cccr analyze microservices calls order-service
cccr analyze microservices dependencies order-service
cccr analyze microservices impact order-service
cccr analyze microservices path order-service payment-service
cccr analyze microservices orphan-integrations

cccr export microservices --drawio architecture.drawio
cccr export microservices --html architecture.html
cccr export microservices --c4 likec4-project
cccr export modules --drawio module-dependencies.drawio
cccr export modules --html module-dependencies.html
```

The microservice export covers HTTP, Kafka, and MongoDB relationships. The
module export is a separate build-dependency view. Use `--workspace` only when
the services live in separately indexed repositories.

## Findings

Use `cccr summary` for the broad Semgrep posture, then filter indexed findings
with `cccr findings`:

```bash
cccr findings "missing HTTP timeout"
cccr findings "hardcoded SASL credentials"
cccr findings "CWE-89" --severity ERROR
cccr findings "PLAINTEXT kafka security.protocol" --path 'app/*'
```

`cccr findings` uses a precision-first lexical query: all query terms must be
present. Prefer an exact rule identifier or CWE when known. `--context`
explicitly includes source context; otherwise findings remain a concise
architecture-level view.

## Code Search

Use semantic search only after the architecture-level commands above, or when
the question is specifically about implementation:

```bash
cccr search database connection pooling
cccr search --lang java request validation
cccr search --path 'src/main/java/**' retry handling
cccr search --offset 5 --limit 5 database schema
```

`--lang` accepts one language. `--path` is forwarded to `ccc`. Search results
identify a file and range; inspect that source only when it is needed to answer
the question.

## References

- [settings.md](references/settings.md): project configuration and `ccc`
  settings.
- [management.md](references/management.md): installation, initialization,
  index refresh, and troubleshooting.
