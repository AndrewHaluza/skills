---
name: agent-docs-formatter
description: Formats documentation into Agent-Optimized Markdown — namespaced key headers, ID-bound block delimiters, cross-reference links, and rationale kept below the rule — so coding agents read one block via grep/awk instead of the whole file. Use when the user asks to format, optimize, or rewrite documentation for AI coding agents.
---
# Agent-Optimized Docs Formatter

Rewrites human-readable Markdown into keyed blocks. An agent lists the keys with one `grep`, extracts one block with one `awk`, and follows `@ns:ID` links from block to block. It never reads the whole file.

## Inputs

- **File path** and **namespace** from the user. When either is missing, use namespace `def` and path `docs/DRAFT.md` and continue; mention the defaults in your reply.

## Transformation rules

Apply every rule to every block.

1. **Agent instructions header.** The file starts with this comment. Replace `<file_path>` with the literal path and `<ns:ID>` with a real key from the file:

    ```html
    <!-- AGENT INSTRUCTIONS:
    This file uses an agent-optimized block format. DO NOT read this file entirely.
    1. TABLE OF CONTENTS: Run this to list all available keys:
      grep -F "## [@" <file_path>

    2. EXTRACT A BLOCK: Run this to read one block (example key '<ns:ID>'):
      awk -v id="[@<ns:ID>]" 'index($0,"## " id)==1{p=1} p{print} index($0,"END_DOC_BLOCK: " id)==1{exit}' <file_path>

    3. FOLLOW A LINK: a reference `@ns:ID` names another block. Find its file with:
      grep -rlF "## [@ns:ID]" docs
    -->
    ```

    The `index()` extract matches the key literally (a `.` in `ui:12.7` stays a dot) and stops at the end marker.

2. **Namespaced primary keys.** Every `##` heading becomes `## [@<ns>:<ID>] <Title>`. The namespace scopes a domain (`arch`, `ui`, `core`) and is the same for the whole file.
    - `<ID>`: uppercase letters, digits, `-` and `.` — short sequential (`DB-01`, `12.7`) or a short descriptor (`AUTH`, `SCHEMA-CHECK`).
    - Each key is unique within its file. Give a key that another file also uses (`CONTENTS`) only when it is the file's table of contents.
    - *Before:* `## How to handle SQLite errors` → *After:* `## [@arch:DB-01] SQLite error handling`

3. **Heading hierarchy.** `##` is reserved for primary keys. Demote every other `##` to `###` or `####`; nested headings stay inside their block.

4. **ID-bound end marker.** Every block ends with its own key on a line by itself, exactly:

    ```markdown
    END_DOC_BLOCK: [@<ns>:<ID>]
    ```

    The marker's key equals the heading's key. Content between one end marker and the next `##` heading is a blank line only.

5. **Rule first, rationale last.** A block opens with the rule or fact an agent acts on, in short imperative sentences. History, reasoning and alternatives go in a `### Rationale` subsection at the end of the block, so an agent that stops reading early still has the rule.

6. **Cross-references as links.** When one block depends on another, link it in the text as `@<ns>:<ID>` (for example "see @arch:DB-01"). When the key exists in more than one file, qualify it: `docs/arch/a.md#@arch:CONTENTS`. Write the full `@ns:ID` form every time — a bare `DB-01` or `arch:DB-01` cannot be resolved by a link checker.

7. **Block size.** Keep each block under about 4 KB. Split a larger block into sibling keys (`DB-01`, `DB-02`) and link them.

8. **Code fences.** Keep example `## [@` headings and end markers inside fenced code blocks only, and keep them out of the real key list: write the example with a placeholder namespace (`@ns:ID`) so it never collides with a real key.

## Completion check

Run these on the output file and fix every mismatch before replying:

```sh
f=<file_path>
grep -c '^## \[@' "$f"; grep -c '^END_DOC_BLOCK: \[@' "$f"          # equal counts
diff <(grep -oE '^## \[@[^]]+\]' "$f" | sed 's/^## //') \
     <(grep -oE '^END_DOC_BLOCK: \[@[^]]+\]' "$f" | sed 's/^END_DOC_BLOCK: //')   # same keys, same order
grep -oE '^## \[@[^]]+\]' "$f" | sort | uniq -d                       # no duplicate keys
grep -E '^## ' "$f" | grep -v '^## \[@'                               # no unkeyed ## headings
```

Then confirm every `@ns:ID` link in the file resolves to a `## [@ns:ID]` heading in the docs tree. The work is done when all four commands show no mismatch and every link resolves.
