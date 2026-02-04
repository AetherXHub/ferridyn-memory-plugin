# FerridynDB Memory Plugin for Claude Code

Agentic memory plugin that gives Claude Code persistent, structured memory across sessions. Automatically retrieves relevant context before each prompt, saves important learnings before conversation compaction, and reflects on sessions at exit.

Backed by [FerridynDB](https://github.com/AetherXHub/ferridyndb) via the `fmemory` CLI with 3 hooks, 13 skills, and 13 commands.

## What It Does

**3 Automatic Hooks:**
- **UserPromptSubmit** — Before each prompt, discovers and injects relevant memories
- **PreCompact** — Before conversation compaction, extracts and persists key learnings
- **Stop** — At session exit, reflects and stores high-level insights

**13 Skills** for guided memory operations (slash commands):
- Core: `/ferridyn-memory:setup`, `remember`, `recall`, `forget`, `browse`, `learn`, `health`
- Proactive: `/ferridyn-memory:teach`, `reflect`, `context`, `update`, `decide`, `status`

**13 Commands** bridging Claude Code to the `fmemory` CLI.

## Prerequisites

All three must be installed and in your PATH:

1. **fmemory CLI**
   ```bash
   cargo install --git https://github.com/AetherXHub/ferridyn-memory
   ```

2. **ferridyn-server** (the database daemon)
   ```bash
   cargo install ferridyn-server --git https://github.com/AetherXHub/ferridyndb
   ```

3. **ANTHROPIC_API_KEY** environment variable
   ```bash
   export ANTHROPIC_API_KEY="sk-ant-..."
   ```

4. **Node.js 20+** (for building hook scripts)

## Install

```bash
/plugin marketplace add AetherXHub/ferridyn-memory-plugin
/ferridyn-memory:setup
```

The setup command verifies all prerequisites, builds hook scripts, starts the server, and runs a round-trip test.

## Hooks

| Hook | Event | Purpose |
|------|-------|---------|
| **UserPromptSubmit** | Before each prompt | Recalls relevant memories, injects proactive memory protocol |
| **PreCompact** | Before conversation compaction | Extracts and persists key learnings from transcript |
| **Stop** | At session exit | Reflects on conversation, stores high-level insights |

## Skills

### Core (User-Invoked)

| Skill | Purpose | When to Use |
|-------|---------|------------|
| `setup` | Bootstrap plugin — verify CLI, build hooks, start server | First-time setup only |
| `remember` | Guidance on what and how to store | Before storing a memory |
| `recall` | Precise and natural language retrieval | Searching for specific memories |
| `forget` | Safe memory removal workflow | Deleting outdated memories |
| `browse` | Interactive exploration of categories, schemas, indexes | Understanding memory structure |
| `learn` | Deep codebase exploration → persistent project memory | Learning new codebase patterns |
| `health` | Memory integrity diagnostics | Checking schema coverage, index usage |

### Proactive (Auto-Triggered + User-Invokable)

| Skill | Auto-Trigger | Purpose |
|-------|--------------|---------|
| `teach` | "remember that...", "note that..." | Parse natural language into structured memory |
| `reflect` | After completing significant work | Extract and persist learnings |
| `context` | Before starting complex work | Pull relevant memories; ask/store if missing |
| `update` | When information contradicts stored data | Find and replace stale memories |
| `decide` | When significant decision made | Log decision with rationale and alternatives |
| `status` | Session start, "what do you know about..." | Quick memory overview by category |

## Configuration

| Variable | Required | Purpose |
|----------|----------|---------|
| `ANTHROPIC_API_KEY` | Yes | Claude Haiku for schema inference, NL parsing, query resolution |
| `FERRIDYN_MEMORY_CLI` | No | Override `fmemory` binary path (default: looks up in PATH) |
| `FERRIDYN_MEMORY_SOCKET` | No | Override server socket path (default: `~/.local/share/ferridyn/server.sock`) |

## Development

### Install Dependencies
```bash
npm install
```

### Build Hook Scripts
```bash
npm run build
```

Compiles TypeScript in `scripts/src/*.ts` to `scripts/dist/*.mjs` via `tsup`.

### Test Scripts (Manual)
```bash
npm run test:retrieval   # Test UserPromptSubmit hook
npm run test:commit      # Test PreCompact hook
npm run test:reflect     # Test Stop hook
npm run test:health      # Test health diagnostics
npm run test:stats       # Test stats utility
```

## How It Works

1. Claude Code triggers a hook event (UserPromptSubmit, PreCompact, Stop)
2. `hooks.json` routes to a TypeScript script in `scripts/dist/`
3. Script calls `fmemory --json <subcommand>` via subprocess
4. Results processed and returned to Claude Code

## Repository Structure

```
scripts/src/           TypeScript source (config.ts bridges to fmemory CLI)
scripts/dist/          Compiled output (.mjs files)
skills/*/SKILL.md      13 skill definitions
commands/*.md          13 command definitions
hooks/hooks.json       3 hook configurations
.claude-plugin/        Plugin metadata (plugin.json, marketplace.json)
```

## Documentation

For detailed usage, see:
- **FerridynDB Memory CLI**: https://github.com/AetherXHub/ferridyn-memory
- **Plugin Homepage**: https://github.com/AetherXHub/ferridyn-memory-plugin
