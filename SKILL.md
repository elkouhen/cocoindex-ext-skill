---
name: cccf
description: Query and fix a repository's security/quality debt through the ccc-findings Semgrep index. Triggers — vulnerability, security, semgrep, finding, debt, audit.
---

# ccc-findings (cccf)

Local Semgrep index, searchable in natural language and joined with code through
`ccc`. This skill helps the agent answer quickly, with little noise, without
rerunning a full scan every time a security question appears.

## UX golden rule

Start with the cheapest query that answers the question, then ask for more
context only when action is needed:

1. Global overview → `findings_summary()`.
2. Specific question or file → `search_findings(...)`.
3. Code exploration with associated debt → `search_code_with_findings(...)`.
4. After a patch → `reindex_findings()` then rerun the same search as before the patch.

Never drown the user in raw JSON: summarize the useful findings, their
files/lines, and the proposed or completed action.

## Choose the right tool

| User need | Tool to use | Expected agent output |
|---|---|---|
| "Give me a security overview" | `findings_summary()` | 3-5 lines: severities, main rules, hot spots. |
| "Is there a SQL injection?" | `search_findings(query="sql injection", limit=5)` | Relevant findings, sorted, without detailed context on the first call. |
| "I am going to edit this file" | `search_findings(path_glob="<file>*", limit=10)` | Warn about known findings on the file before patching. |
| "Show session code with its issues" | `search_code_with_findings(query="session management")` | Code results annotated with findings and max severity. |
| "Fix this finding" | `search_findings(..., include_context=true)` then patch | Read the context before any modification. |
| "Verify after the fix" | `reindex_findings()` then `search_findings(...)` | Confirm disappearance or explain the blocker. |

## Pleasant default path

When the request is broad or ambiguous:

1. Call `findings_summary()` to assess volume.
2. If `ERROR` findings exist, look at critical issues first:
   `search_findings(query="critical security", severity="ERROR", limit=5)`.
3. Propose or apply a fix only after retrieving the context of a precise finding
   with `include_context: true`.

When the request targets a file:

1. Call `search_findings(path_glob="<file>*", limit=10)`.
2. If no known finding exists, say so plainly and continue the task.
3. If findings exist, account for them before editing the file.

When the request asks for a fix:

1. Search for the finding with the most precise available filters.
2. Re-read the context (`include_context: true`) or the source file.
3. Patch the code, respecting `fix` if Semgrep provides one.
4. If the official Semgrep MCP is available, scan only the modified file for a
   fresh verification.
5. Call `reindex_findings()`.
6. Rerun the same search as at the start to confirm the finding disappeared.
7. After 2 unsuccessful attempts, stop and explain the blocker.

## Installation

Installing this skill file only makes these instructions available — it does
not install Semgrep, `cccf`, or register the MCP servers. Before the first use
of any tool below (`findings_summary`, `search_findings`, ...), check whether
they respond; if not, run the steps below yourself (or ask the user to).

0. **This skill**: `npx skills add https://github.com/elkouhen/ccc-findings-skill`
   (mono-skill repo — installs `cccf` directly, no `--skill` flag needed).
1. **Semgrep** (required by `cccf`): `pipx install semgrep` or
   `brew install semgrep`.
2. **cccf**: not published on PyPI — install from source:
   ```bash
   uv tool install git+https://github.com/elkouhen/ccc-findings
   # or: pipx install git+https://github.com/elkouhen/ccc-findings
   ```
3. **Embedding model**: downloaded automatically on the first `cccf index`
   (`sentence-transformers`, default model `Snowflake/snowflake-arctic-embed-xs`,
   ~100 MB). Network access is required once; later runs reuse the local cache
   (`~/.cache/huggingface`).
4. **Initialize and index the target repo**:
   ```bash
   cd <your-repo>
   cccf init                # detects .semgrep.yml/semgrep.yml/.semgrep;
                            # otherwise use --rules <path-or-pack>, or falls
                            # back automatically to the p/security-audit pack
   cccf index
   ```
5. **Register both required MCP servers** with the client (for example
   `.mcp.json` at the project root for Claude Code, or your client's
   equivalent). Both are needed: `cccf` for the indexed findings search
   (source: https://github.com/elkouhen/ccc-findings) and the official
   Semgrep MCP for fresh post-patch verification (step 4 of Workflow 3):
   ```json
   {
     "mcpServers": {
       "cccf": {"command": "cccf", "args": ["mcp"]},
       "semgrep": {"command": "uvx", "args": ["semgrep-mcp"]}
     }
   }
   ```
   If the Semgrep MCP is unreachable in a given environment, that
   verification step is simply skipped: `reindex_findings` + `search_findings`
   (steps 5-6 of Workflow 3) are enough to confirm a finding disappeared,
   with lower confidence than a fresh scan.

## Configure the Semgrep rules to analyze

The `rules` field in `.cccf/config.yml` controls what is analyzed. If nothing
is specified, `cccf init` already uses the default registry pack
`p/security-audit` (see Installation, step 4). This section is for
customization. Valid sources can be mixed in the same list:

- **Local rules file**, for example `rules/rules.yml`, in standard Semgrep
  format (`rules: [{id, pattern, message, severity, languages,
  metadata: {cwe, owasp}}, ...]`).
- **Rules directory**: path to a directory containing several YAML files, loaded
  together.
- **Semgrep registry pack**, for example `p/security-audit`, `p/python`,
  `p/owasp-top-ten`. Network access is required the first time Semgrep downloads
  and caches the pack.

Define rules at init time (`--rules` is repeatable):
```bash
cccf init --rules rules/rules.yml --rules p/security-audit
```
or edit `.cccf/config.yml` directly afterward:
```yaml
rules:
  - rules/rules.yml
  - p/security-audit
```
Then apply the change to the whole repo with a full scan:
```bash
cccf index --full
```
**Agent trap**: the MCP tool `reindex_findings` is always incremental; it only
rescans modified files. After changing `rules`, `reindex_findings` alone does
NOT reanalyze files already indexed with the old config. Use `cccf index --full`
from the CLI instead; there is no MCP `--full` equivalent.

`min_severity` (`INFO`/`WARNING`/`ERROR`, same file) filters what is kept in the
database among the findings discovered by the rules. It is independent from the
choice of rules themselves.

## Reference workflows

### Workflow 1 — Explore known issues

1. Call `search_findings(query="<issue description>", limit=5)`.
2. For the selected finding, call `search_findings` again with the same filter
   and `include_context: true` to retrieve the surrounding code.

### Workflow 2 — Before editing a file

1. Call `search_findings(path_glob="<file>*")` to list known findings on that
   file before patching it.

### Workflow 3 — Fix loop

1. `search_findings(query="...", severity="ERROR", path_glob="...")` to list
   findings to fix.
2. Re-read each finding's context (`include_context: true`).
3. Patch the code, respecting the finding's `fix` field if provided.
4. If the official Semgrep MCP is available, call its `semgrep_scan` tool on the
   modified file only for immediate fresh verification. Do not scan the whole repo.
5. Call `reindex_findings` to update the `cccf` index.
6. Call `search_findings` again with the same filter as step 1 to confirm the
   finding disappeared.
7. If the finding still exists after 2 fix attempts, do not try a 3rd time:
   report the blocker to the user.

### Workflow 4 — Overview

1. Call `findings_summary()` for a low-token aggregate state (severities, top rules).

## Cross-search code + findings

To explore code while accounting for its security debt, prefer
`search_code_with_findings(query="...")` over a plain `ccc` search: it annotates
each code result with known Semgrep findings that overlap it.

## Anti-patterns

- Do not scan the whole repo through the official Semgrep MCP (`semgrep_scan`
  without a precise target): use the `cccf` index (`search_findings`,
  `findings_summary`) for everything already indexed, and reserve the official
  Semgrep MCP for post-patch verification of one precise file.
- Do not fix a finding before reading its context (`include_context: true` or
  direct source file reading).
- Do not remove an existing `# nosemgrep` comment: it represents an existing
  decision about a false positive.
- Do not display the raw JSON response unless the user explicitly asks for JSON:
  turn results into an actionable summary.
- Do not block if `ccc` is unavailable: `search_code_with_findings` returns a
  findings fallback that can be used to continue the analysis.

## Recommended response format

Answer briefly and decision-first:

```text
I found 2 ERROR findings in src/payments/.
- src/payments/db.py:42 — possible SQL injection (CWE-89), fixed and reindexed.
- src/payments/token.py:18 — weak token generation, still needs a fix.
```

If nothing is found:

```text
No known Semgrep finding on src/payments/*. The index is up to date after reindex_findings.
```
