---
name: agent-docs-formatter
description: Formats documentation into Agent-Optimized Markdown with canonical key headers, strict block delimiters, and separated rationale for cheap CLI search. Use when the user asks to format, optimize, or rewrite documentation for AI coding agents.
---
# Agent-Optimized Docs Formatter

Transforms standard, human-readable documentation into a strictly structured format optimized for terminal-based AI coding agents to search cheaply using standard text-processing commands.

## When to Use

- The user provides a markdown file or raw documentation and asks to make it "agent-optimized", "agent-ready", or "searchable".
- The user asks to apply the deterministic documentation format.

## Transformation Rules

When rewriting the documentation, you MUST apply these strict structural rules:

1. **Invisible Agent Instructions (The API):**
    
    At the very top of the file, inject this exact HTML comment block. You MUST replace `<file_path>` inside the comment block with the literal, hardcoded file path provided by the user (or the default). Do not leave `<file_path>` as a literal string in the output.
    
    ```html
    <!-- AGENT INSTRUCTIONS:
    This file uses an agent-optimized block format. DO NOT read this file entirely.
    1. TABLE OF CONTENTS: Run this to list all available keys:
      grep -F "## [@" <file_path>
    
    2. EXTRACT A RULE: Run this to read a specific block (Example for ID 'ui:R01'):
      awk "/^## \[@ui:R01\]/,/END_DOC_BLOCK: \[@ui:R01\]/" <file_path>
    -->
    ```

2. **Strict Namespace Governance & File Path:**
    
    **Concept:** Namespaces act as a scoping prefix (e.g., `arch`, `ui`, `core`) to categorize domains and prevent primary key collisions across different files. Every primary key MUST include a namespace prefix (e.g., `[@ui:R01]`, `[@arch:DB-01]`).
    
    **Default Fallback:** If the user does not explicitly provide a namespace and file path, do NOT interrupt them to ask. Instead, automatically use a short default namespace (e.g., `def` or `sys`) and the default file path `docs/DRAFT.md`.

3. **Canonical Key Headers (Primary Keys):**
    
    Rewrite every major sub-heading (`##`) so it starts with a bracketed, unique primary key using the required namespace. Generate concise, sequential IDs (e.g., `01`, `02`) or short alphabetical descriptors (e.g., `AUTH`, `API`) to append to the namespace.
    
    - *Bad:* `## How to handle SQLite errors`
    - *Good:* `## [@arch:DB-01] SQLite ABI Resolution`

4. **Heading Hierarchy Normalization:**
    
    If the user's original text uses `##` for sub-sections or minor points, you must demote them to `###` or `####` to ensure the document ONLY uses `##` for primary keys. Leave nested headers (`###`, `####`) inside the block; do not turn them into primary keys.

5. **Strict ID-Bound Block Delimiters:**
    
    Every conceptual block MUST end with an exact
