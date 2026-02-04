# CLAUDE.md

## Project

FerridynDB Memory Plugin — Claude Code plugin providing agentic memory via 3 hooks, 13 skills, and 13 commands. The plugin invokes the `fmemory` CLI (installed separately via `cargo install --git https://github.com/AetherXHub/ferridyn-memory`) as a subprocess to store/retrieve memories from a local FerridynDB database. fmemory ships with 7 predefined categories (project, decisions, contacts, preferences, bugs, tools, notes) with typed attributes and secondary indexes.

## Build Commands

```bash
npm install                # Install dev dependencies (tsup, typescript)
npm run build              # Compile TypeScript hooks to scripts/dist/
```

No Rust commands — the plugin is pure TypeScript. The `fmemory` CLI and `ferridyn-server` daemon are external dependencies managed by the `setup` skill.

## Architecture

```
scripts/src/
  config.ts              Bridge to fmemory CLI (execFile wrapper, JSON parsing)
  memory-retrieval.ts    UserPromptSubmit hook (recall + inject context)
  memory-commit.ts       PreCompact hook (extract and save learnings)
  memory-reflect.ts      Stop hook (session reflection)
  memory-health.ts       Standalone health diagnostics
  memory-stats.ts        Standalone stats utility
  types.ts               Shared type definitions

scripts/dist/            Compiled .mjs files (produced by npm run build)

skills/*/SKILL.md        13 skill definitions (setup, remember, recall, forget, browse,
                         learn, health, teach, reflect, context, update, decide, status)

commands/*.md            13 command definitions (mimic skill names)

hooks/hooks.json         Hook configuration — 3 events (UserPromptSubmit, PreCompact, Stop)
                         each maps to a script in scripts/dist/

.claude-plugin/
  plugin.json            Plugin metadata (name, version, permissions)
  marketplace.json       Marketplace listing (description, tags, owner)
```

## How Hooks Work

1. **Event triggered** — Claude Code detects hook event (UserPromptSubmit, PreCompact, or Stop)
2. **Script invoked** — hooks.json routes to appropriate script in scripts/dist/
3. **Input received** — Hook input JSON passed via stdin (containing prompt, transcript_path, etc.)
4. **CLI invoked** — Script calls `fmemory --json <subcommand>` via execFile()
5. **Output processed** — Result JSON parsed and returned to Claude Code

Example flow (UserPromptSubmit):
```
Claude Code (event triggered)
  ↓
hooks.json["UserPromptSubmit"] → scripts/dist/memory-retrieval.mjs
  ↓
stdin: { "prompt": "...", "hook_event_name": "UserPromptSubmit" }
  ↓
execFile: fmemory --json recall --category project --limit 5
  ↓
stdout: { "memories": [...], "injected": true }
  ↓
Claude Code (injects context into prompt)
```

## Environment Variables

- **ANTHROPIC_API_KEY** (required) — Claude Haiku for NL parsing and query resolution (used by `fmemory`, not the plugin)
- **FERRIDYN_MEMORY_CLI** (optional) — Override `fmemory` binary path; default: looks up in PATH
- **FERRIDYN_MEMORY_SOCKET** (optional) — Override server socket path; default: `~/.local/share/ferridyn/server.sock`

## Development Process

1. **Install dependencies**
   ```bash
   npm install
   ```

2. **Edit TypeScript** — Modify files in `scripts/src/` as needed

3. **Build** — Compile to `scripts/dist/`
   ```bash
   npm run build
   ```
   Must produce 5 `.mjs` files:
   - memory-retrieval.mjs
   - memory-commit.mjs
   - memory-reflect.mjs
   - memory-health.mjs
   - memory-stats.mjs

4. **Test hooks manually** — Pipe JSON input to test a compiled script
   ```bash
   # Test UserPromptSubmit hook
   echo '{"prompt": "test", "hook_event_name": "UserPromptSubmit"}' | \
     node scripts/dist/memory-retrieval.mjs

   # Or use npm test scripts
   npm run test:retrieval
   npm run test:commit
   npm run test:reflect
   ```

## Key Files

- **config.ts** — Shared utilities
  - `runFMemory()` — Wrapper around execFile to invoke `fmemory --json`
  - `parseJsonOutput()` — Safe JSON parsing with error handling
  - `callHaiku()` — LLM calls for category selection, memory extraction, and NL parsing

- **memory-retrieval.ts** — UserPromptSubmit hook
  - Discovers category names from fmemory
  - Selects relevant categories via Haiku (given the prompt)
  - Retrieves items from selected categories
  - Injects structured context + memory protocol into the hook response

- **memory-commit.ts** — PreCompact hook
  - Reads transcript file (path provided by Claude Code)
  - Extracts key learnings via Haiku summarization
  - Persists learnings using NL-first `fmemory remember` syntax
  - Returns summary of saved memories

- **memory-reflect.ts** — Stop hook
  - Similar to memory-commit but optimized for session-level reflection
  - Focuses on high-level decisions, patterns, preferences
  - Complements PreCompact (granular facts vs. big-picture insights)

- **types.ts** — Shared interfaces
  - `HookInput` — Input structure from Claude Code
  - `HookOutput` — Output structure to Claude Code
  - `FMemoryItem` — Single memory entry
  - `FMemoryCategory` — Category metadata

## Testing

Hooks can be tested without full Claude Code setup:

```bash
# Set ANTHROPIC_API_KEY
export ANTHROPIC_API_KEY="sk-ant-..."

# Test retrieval hook
echo '{"prompt":"how to deploy","hook_event_name":"UserPromptSubmit"}' | \
  node scripts/dist/memory-retrieval.mjs

# Test commit hook (requires /dev/null or valid transcript)
echo '{"transcript_path":"/dev/null","hook_event_name":"PreCompact"}' | \
  node scripts/dist/memory-commit.mjs

# Test reflect hook
echo '{"transcript_path":"/dev/null","stop_hook_active":false,"hook_event_name":"Stop"}' | \
  node scripts/dist/memory-reflect.mjs
```

All tests require `fmemory` in PATH and `ferridyn-server` running.

## Troubleshooting

**"fmemory: command not found"**
- Install the CLI: `cargo install --git https://github.com/AetherXHub/ferridyn-memory`
- Or set `FERRIDYN_MEMORY_CLI` to the binary path

**"Cannot connect to server"**
- Ensure `ferridyn-server` is running: `ferridyn-server --db ~/.local/share/ferridyn/memory.db --socket ~/.local/share/ferridyn/server.sock`
- Or run `/ferridyn-memory:setup` to bootstrap

**"ANTHROPIC_API_KEY is required"**
- Set the environment variable: `export ANTHROPIC_API_KEY="sk-ant-..."`
- Some hooks (NL parsing, memory extraction) require Claude Haiku

## Resources

- **FerridynDB Memory CLI**: https://github.com/AetherXHub/ferridyn-memory
- **FerridynDB Database**: https://github.com/AetherXHub/ferridyndb
- **Plugin Homepage**: https://github.com/AetherXHub/ferridyn-memory-plugin
