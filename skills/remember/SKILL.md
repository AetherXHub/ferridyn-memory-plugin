---
name: remember
description: Store a memory using the FerridynDB memory system. Guides the agent on what to store, how to choose categories, and when NOT to store.
---

# Remember — Store a Memory

Use the `fmemory remember` command to persist knowledge across sessions. fmemory is NL-first — describe what you want to store in natural language and let the system extract structure.

## When to Store

- Architecture decisions and rationale
- User preferences and workflows
- Bug patterns and their fixes
- Project-specific knowledge (conventions, configs, gotchas)
- Contact information and relationships
- Tool configurations and environment details
- Recurring task patterns

## When NOT to Store

- Trivial exchanges ("hello", "thanks")
- Temporary debugging state (use notepad instead)
- Information already in CLAUDE.md or AGENTS.md
- Large code blocks (store a summary + file path instead)
- Speculative or unconfirmed information

## NL-First Syntax

The primary interface is natural language. Just describe what you want to remember:

```bash
fmemory remember "Toby is a backend engineer, email toby@example.com"
```

Optionally specify a category:

```bash
fmemory remember --category contacts "Toby is a backend engineer, email toby@example.com"
```

Or provide both category and key for full control:

```bash
fmemory remember --category contacts --key toby "backend engineer at Example Corp"
```

## Predefined Categories

fmemory ships with 7 built-in categories, created automatically on first use or via `fmemory init`:

- `project` — codebase knowledge, architecture, conventions, gotchas
- `decisions` — architectural and design decisions with rationale
- `contacts` — people, roles, contact info
- `preferences` — user preferences, workflow patterns, directives
- `bugs` — bug patterns, root causes, fixes, workarounds
- `tools` — endpoints, configs, infrastructure, CI/CD, environments
- `notes` — general-purpose catch-all for anything that doesn't fit elsewhere

Custom categories can be added with `fmemory define`.

## Structured Data

Items have typed attributes (name, email, role, etc.), not flat content strings. Keys are simple identifiers (`toby`), not hierarchical formats. Every item gets a `created_at` timestamp (ISO 8601, UTC) injected automatically.

**Before (old):** `{ "category": "people", "key": "toby#email", "content": "toby@example.com" }`

**Now:** `{ "category": "contacts", "key": "toby", "name": "Toby", "email": "toby@example.com", "role": "backend engineer", "created_at": "2025-01-15T10:30:00Z" }`

Each predefined category has typed attributes and secondary indexes for fast lookups. When `--category` is omitted, fmemory selects the best matching category from the predefined list automatically.

## Tips

- Store descriptive NL text — let the system extract structure
- Store the **why** not just the **what** — "chose JWT because sessions don't work with serverless" > "using JWT"
- Prefer atomic memories (one fact per entry) over large blobs
- No need to worry about key formats — the system generates appropriate keys from your NL input
