# ccc-radar-skill

Claude Code skill for `cccr` (CocoIndex Code + Semgrep findings): semantic
code search, index management, and Semgrep findings lookup, driven
automatically by the agent.

- [`skills/cccr/SKILL.md`](skills/cccr/SKILL.md) — the skill itself:
  ownership rules, searching, filtering, pagination.
- [`skills/cccr/references/settings.md`](skills/cccr/references/settings.md) —
  embedding model, include/exclude patterns, language overrides.
- [`skills/cccr/references/management.md`](skills/cccr/references/management.md) —
  installation, initialization, daemon management, troubleshooting.
- [`skills/cccr/rules/default/`](skills/cccr/rules/default/) — bundled
  Semgrep rule pack (Java) for bounded file streaming and Kafka
  claim-check/delivery guarantees, run by default on `cccr init` (see
  **Default Rules** in `SKILL.md`); `design-rules.md` in the same directory
  documents the full rule set in prose (including the rules with no
  Semgrep-detectable pattern).
- [`skills/cccr/rules/liveness/`](skills/cccr/rules/liveness/) — bundled
  Semgrep rule pack (Java) for distributed-system blocking points: missing
  HTTP timeouts, blocking waits, synchronous REST calls inside Kafka
  consumers, network calls under a lock — also run by default on
  `cccr init` (see **Default Rules** in `SKILL.md`).
- [`skills/cccr/rules/rest/`](skills/cccr/rules/rest/) — bundled Semgrep
  rule pack (Java) that inventories REST endpoints for `cccr
  endpoints`/`cccr graph`: Spring routes exposed (`@GetMapping`/.../
  `@RequestMapping` for any HTTP verb), `RestTemplate` call sites, Feign
  declarative clients (`@FeignClient` interface methods, distinguished from
  server routes by their missing method body), and `WebClient` fluent calls
  (`.get().uri(...)`, ...) — not a findings pack, run by default on `cccr
  init` (see **Default Rules** in `SKILL.md`; see `ccc-radar`'s
  `archive/BACKLOG-10.md` K11).
- [`skills/cccr/rules/kafka/`](skills/cccr/rules/kafka/) — bundled Semgrep
  rule pack (Java/Spring, plus raw `kafka-clients`) that inventories Kafka
  producers/consumers (`@KafkaListener`, `KafkaTemplate.send`,
  `ProducerRecord`, `KafkaConsumer.subscribe(...)` for non-Spring
  consumers, and `StreamsBuilder.stream(...)`/`KStream.to(...)` for Kafka
  Streams) for `cccr endpoints`/`cccr graph`, resolving topic names given
  as Spring properties (`${app.kafka.topics.orders}`, including via a
  `@Value`-annotated field referenced by variable) against
  `application.yml`/`.properties` when present — not a findings pack, run
  by default on `cccr init` (see
  `ccc-radar`'s `archive/BACKLOG-10.md` K2, and `BACKLOG.md` Q25 for the
  Kafka Streams rules).
- [`skills/cccr/rules/kafka-security/`](skills/cccr/rules/kafka-security/)
  — bundled Semgrep rule pack (Java/Spring) for Kafka security findings:
  hardcoded SASL credentials, `PLAINTEXT` security protocol, a
  `JsonDeserializer` trusting all packages, raw Java deserialization — run
  by default on `cccr init` (see `ccc-radar`'s `archive/BACKLOG-10.md`
  K8, security volet).

## Installation

```bash
npx skills add elkouhen/ccc-radar-skill
```

This installs the `cccr` skill (`skills/cccr/`) for your coding agent. It
still requires the `cccr` CLI itself — see
[Installation in `ccc-radar`](https://github.com/elkouhen/ccc-radar#installation).

## Related projects

- [`ccc-radar`](https://github.com/elkouhen/ccc-radar) (`cccr`) — the
  CLI and MCP server this skill drives. It indexes Semgrep findings locally
  and joins them to code search results from `ccc`.
- [`cocoindex-code`](https://github.com/cocoindex-io/cocoindex-code) (`ccc`)
  — the underlying AST-based semantic code search tool that `cccr` extends
  as a companion package (no fork, no internal import at the CLI/MCP level —
  see `ccc-radar`'s ADR-1).

## Provenance

This skill started as an adaptation of cocoindex-code's own
[`skills/ccc/`](https://github.com/cocoindex-io/cocoindex-code/tree/main/skills/ccc)
skill (Apache-2.0): `SKILL.md` is renamed to `cccr` and extended to cover
Semgrep findings, while `references/settings.md` and
`references/management.md` are carried over unmodified since they document
`ccc` itself, which `cccr` relies on unchanged. Each file links back to its
source. See [LICENSE](LICENSE) for the terms this carries over.

## License

[Apache License 2.0](LICENSE), matching the upstream
[`cocoindex-code`](https://github.com/cocoindex-io/cocoindex-code) project.
