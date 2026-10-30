# Agent-Optimized Docs Formatter (`agent-docs-formatter`)

Transforms standard, human-readable documentation into a strictly structured, database-like Markdown format. Terminal-based AI coding agents can then search and extract context cheaply with standard UNIX tools (`grep` and `awk`), saving tokens and keeping context relevant.

## Why use this?

Terminal AI agents (Claude Code, Cursor, local CLI agents) often read an entire documentation file to find one rule. That consumes context window and dilutes relevance.

With namespaces, primary keys and deterministic block delimiters, an agent can:

1. **Discover** the available rules with one cheap `grep`.
2. **Extract** only the block it needs with one targeted `awk`.

## Performance Impact & Metrics

Measured on the karst repo: 38 keyed docs, 3–45 KB each. Figures are bytes read (TOC `grep` + one block via `awk`) versus reading the whole file. Tokens are roughly 1/4–1/3 of bytes.

| Metric | Standard docs (full read) | Agent-optimized (grep + awk) | Measured impact |
|---|---|---|---|
| Context per lookup | O(file size) | O(#keys) for the TOC + O(block size) | Large docs (≥13 KB): ~5–12% of file. Small docs (~3 KB): ~35–40% |
| Typical cost | 3–45 KB per read | 0.1–0.7 KB TOC + 0.9–5 KB block | 62–95% fewer bytes, depending on doc size |
| Best / worst case | n/a | Best: 935 B of 20,957 B (4.5%). Worst: 1,090 B of 2,839 B (38%) | Savings grow with file size |

### Sample lookups

| Doc / key | Keyed read | Whole file | Share |
|---|---|---|---|
| `artifacts spec` / `ART-14` | 935 B | 20,957 B | 4.5% |
| `store-and-schema` / `arch:SCHEMA-CHECK` | 1,397 B | 24,646 B | 6% |
| `glossary` / `GL-01` | 3,240 B | 41,785 B | 7.8% |
| `HOOK-CONTRACT` / `HOOK-04` | 1,415 B | 13,938 B | 10% |
| `agent-cores` / `arch:AC-13` | 4,459 B | 44,991 B | 10% |
| `uat-stuck` / `UAT-03` | 1,090 B | 2,839 B | 38% |

### Caveats

- Cost is not constant. The TOC grep scales with the number of keys and is the largest share of the cost on most lookups.
- Block size matters. A 7.6 KB block in a 24 KB file saves little, so split blocks over about 4 KB.
- Tiny files (under ~5 KB) and files with 2 blocks gain little from keying.
- Latency and answer accuracy were not measured here. Smaller context should help both, but treat that as a hypothesis until tested.
- Count blocks with `grep -c '^END_DOC_BLOCK'`. The header example line also contains the marker and inflates a plain count by one.

## Features

- **Invisible Agent API:** Injects an HTML comment at the top of the file with the exact `grep` and `awk` commands, teaching the agent how to query the document.
- **Namespace Governance:** Enforces namespaces (e.g. `arch`, `ui`, `core`) so primary keys never collide across files.
- **Canonical Key Headers:** Rewrites broad `##` headings into unique, searchable keys (e.g. `## [@ui:AUTH-01] Token Management`).
- **Heading Normalization:** Demotes minor headings to `###` or `####`, so `##` is reserved for top-level queryable blocks.
- **Strict Block Delimiters:** Closes every block with an end marker (e.g. `END_DOC_BLOCK: [@ui:AUTH-01]`) so `awk` extractions never bleed into the next section.

## Usage

Invoke the skill on a raw Markdown file or draft:

> "Format my architecture rules using the agent-docs-formatter. Use the namespace `arch` and the file path `docs/architecture.md`."

If you give no namespace or file path, the formatter defaults to namespace `def` and file path `docs/DRAFT.md`.

## Example Transformation

### Before (standard Markdown)

```markdown
## Handling Database Errors
Always catch SQLite errors at the repository layer. Don't let them bubble up to the express routers.

## Redis Caching Strategy
Cache user sessions for 24 hours. Invalidate on password reset.
```

### After (agent-optimized Markdown)

```markdown
<!-- AGENT INSTRUCTIONS:
This file uses an agent-optimized block format. DO NOT read this file entirely.
1. TABLE OF CONTENTS: Run this to list all available keys:
  grep -F "## [@" docs/architecture.md

2. EXTRACT A BLOCK: Run this to read one block (example key 'arch:DB-01'):
  awk -v id="[@arch:DB-01]" 'index($0,"## " id)==1{p=1} p{print} index($0,"END_DOC_BLOCK: " id)==1{exit}' docs/architecture.md

3. FOLLOW A LINK: a reference `@ns:ID` names another block. Find its file with:
  grep -rlF "## [@ns:ID]" docs
-->

## [@arch:DB-01] Handling Database Errors
Always catch SQLite errors at the repository layer. Don't let them bubble up to the express routers.
END_DOC_BLOCK: [@arch:DB-01]

## [@arch:CACHE-01] Redis Caching Strategy
Cache user sessions for 24 hours. Invalidate on password reset.
END_DOC_BLOCK: [@arch:CACHE-01]
```

### Extraction notes

- Brackets must be escaped in the awk regex (`\[` and `\]`). Unescaped, they form a character class and the pattern fails.
- IDs containing regex characters (e.g. `ui:11.12`) match loosely. A literal-match extract avoids that:

  ```sh
  awk -v id="[@arch:DB-01]" 'index($0,"## " id)==1{p=1} p{print} index($0,"END_DOC_BLOCK: " id)==1{exit}' docs/architecture.md
  ```
