---
name: setup
description: Configure the FerridynDB memory plugin — verify CLI installation, build hooks, and start the server. Run this once to bootstrap everything.
---

# FerridynDB Memory Setup

This is the setup command for the FerridynDB memory plugin. Run it once after installing the plugin to verify prerequisites, build hook scripts, start the server, and activate hooks.

## Graceful Interrupt Handling

**IMPORTANT**: This setup process saves progress after each step. If interrupted, setup can resume from where it left off.

### State File Location
- `.omc/state/ferridyn-setup-state.json` - Tracks completed steps

### Resume Detection (Step 0)

Before starting any step, check for existing state:

```bash
PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT:-$(cd "$(dirname "$0")/../.." && pwd)}"
STATE_DIR=".omc/state"
STATE_FILE="${STATE_DIR}/ferridyn-setup-state.json"

if [ -f "$STATE_FILE" ]; then
  LAST_STEP=$(jq -r ".lastCompletedStep // 0" "$STATE_FILE" 2>/dev/null || echo "0")
  TIMESTAMP=$(jq -r .timestamp "$STATE_FILE" 2>/dev/null || echo "unknown")

  # Check if state is stale (older than 24 hours)
  TIMESTAMP_RAW=$(jq -r '.timestamp // empty' "$STATE_FILE" 2>/dev/null)
  if [ -n "$TIMESTAMP_RAW" ]; then
    NOW_EPOCH=$(date +%s)
    TS_EPOCH=$(date -d "$TIMESTAMP_RAW" +%s 2>/dev/null || date -j -f "%Y-%m-%dT%H:%M:%S" "$(echo "$TIMESTAMP_RAW" | cut -dT -f1-2 | sed 's/+.*//')" +%s 2>/dev/null || echo "0")
    STATE_AGE=$((NOW_EPOCH - TS_EPOCH))
  else
    STATE_AGE=999999
  fi

  if [ "$STATE_AGE" -gt 86400 ]; then
    echo "Previous setup state is more than 24 hours old. Starting fresh."
    rm -f "$STATE_FILE"
  else
    echo "Found previous setup session (Step $LAST_STEP completed at $TIMESTAMP)"
  fi
fi
```

If state exists and is fresh, use AskUserQuestion to prompt:

**Question:** "Found a previous setup session. Resume or start fresh?"

**Options:**
1. **Resume from step $LAST_STEP** - Continue where you left off
2. **Start fresh** - Begin from the beginning

If user chooses "Start fresh":
```bash
rm -f "$STATE_FILE"
```

### Save Progress Helper

After completing each major step, save progress:

```bash
save_setup_progress() {
  mkdir -p "$STATE_DIR"
  cat > "$STATE_FILE" << EOF
{
  "lastCompletedStep": $1,
  "timestamp": "$(date -Iseconds)"
}
EOF
}
```

## Step 1: Check Prerequisites

Verify the CLI, server, Node.js, and API key are available before attempting anything:

```bash
command -v fmemory >/dev/null 2>&1 && echo "fmemory: OK" || echo "fmemory: MISSING"
command -v ferridyn-server >/dev/null 2>&1 && echo "ferridyn-server: OK" || echo "ferridyn-server: MISSING"
command -v node >/dev/null 2>&1 && echo "node: OK" || echo "node: MISSING"
command -v jq >/dev/null 2>&1 && echo "jq: OK" || echo "jq: MISSING (optional, used for state tracking)"
[ -n "$ANTHROPIC_API_KEY" ] && echo "ANTHROPIC_API_KEY: OK" || echo "ANTHROPIC_API_KEY: MISSING"
```

If `fmemory` is missing, stop and tell the user:

> **The fmemory CLI is not installed.** Install it with Cargo:
> ```
> cargo install --git https://github.com/AetherXHub/ferridyn-memory
> ```
> Then re-run `/ferridyn-memory:setup`.

If `ferridyn-server` is missing, stop and tell the user:

> **The ferridyn-server daemon is not installed.** Install it with Cargo:
> ```
> cargo install ferridyn-server --git https://github.com/AetherXHub/ferridyndb
> ```
> Then re-run `/ferridyn-memory:setup`.

If `node` is missing, warn the user:

> **Node.js is required for hook scripts.** Install Node.js 20+ first.
> The CLI (fmemory) will work without it, but hooks (auto-retrieval, auto-save, session reflection) will not.

If `ANTHROPIC_API_KEY` is missing, warn the user:

> **ANTHROPIC_API_KEY is required.** fmemory uses Claude Haiku for natural language parsing and query resolution.
> Set it in your environment:
> ```
> export ANTHROPIC_API_KEY="sk-ant-..."
> ```
> fmemory will refuse to start without this key.

Use AskUserQuestion:

**Question:** "ANTHROPIC_API_KEY is not set. How would you like to proceed?"

**Options:**
1. **I'll set it now** — Pause and wait for user to set the key, then re-check
2. **Continue anyway** — Setup will complete but hooks won't work until the key is set

Save progress after prerequisites pass:
```bash
save_setup_progress 1
```

## Step 2: Build Hook Scripts

Install Node.js dependencies and build the TypeScript hook scripts:

```bash
cd "$PLUGIN_ROOT"
npm install
npm run build
```

Verify the built hook scripts exist:

```bash
ls -l "$PLUGIN_ROOT/scripts/dist/"
```

Expect to see 5 `.mjs` files:
- `memory-retrieval.mjs` (UserPromptSubmit hook)
- `memory-commit.mjs` (PreCompact hook)
- `memory-reflect.mjs` (Stop hook)
- `memory-health.mjs` (diagnostics utility)
- `memory-stats.mjs` (stats utility)

If any are missing, stop and report the error. Do not continue.

Save progress:
```bash
save_setup_progress 2
```

## Step 3: Create Data Directory and Start Server

```bash
mkdir -p ~/.local/share/ferridyn
```

Check if the server is already running:

```bash
if [ -S ~/.local/share/ferridyn/server.sock ]; then
  # Socket exists — test if server is responsive
  timeout 2 fmemory discover >/dev/null 2>&1
  if [ $? -eq 0 ]; then
    echo "Server already running and responsive."
  else
    echo "Stale socket found. Restarting server..."
    rm -f ~/.local/share/ferridyn/server.sock
  fi
else
  echo "No server running. Starting..."
fi
```

If the server is not running (socket missing or stale), start it:

```bash
nohup ferridyn-server \
  --db ~/.local/share/ferridyn/memory.db \
  --socket ~/.local/share/ferridyn/server.sock \
  > /dev/null 2>&1 &

# Wait for socket to appear (up to 3 seconds)
for i in 1 2 3; do
  [ -S ~/.local/share/ferridyn/server.sock ] && break
  sleep 1
done
```

Verify the socket was created:

```bash
[ -S ~/.local/share/ferridyn/server.sock ] && echo "Server started." || echo "ERROR: Server failed to start."
```

If the server failed to start, stop and report. Save progress:
```bash
save_setup_progress 3
```

## Step 4: Initialize Predefined Categories and Verify

Initialize the 7 predefined category schemas, then test a round-trip:

```bash
fmemory init
fmemory remember --category _setup-test "Setup verification test"
fmemory recall --category _setup-test
fmemory forget --category _setup-test --key setup-verification-test
```

All commands must succeed. If any fails, stop and report. Save progress:
```bash
save_setup_progress 4
```

## Step 5: Clear State and Show Welcome

Clear the setup state file:

```bash
rm -f "$STATE_FILE"
```

Display the following:

```
FerridynDB Memory Setup Complete!

SERVER
  Socket:   ~/.local/share/ferridyn/server.sock
  Database: ~/.local/share/ferridyn/memory.db

CLI
  fmemory - Command-line interface for memory operations
  Path: $(command -v fmemory)

SKILLS (slash commands)
  /ferridyn-memory:remember  - Guidance on what and how to store
  /ferridyn-memory:recall    - Precise and natural language retrieval
  /ferridyn-memory:forget    - Safe memory removal workflow
  /ferridyn-memory:browse    - Interactive memory exploration
  /ferridyn-memory:learn     - Deep codebase exploration → persistent memory
  /ferridyn-memory:teach     - Conversational memory capture ("remember that...")
  /ferridyn-memory:reflect   - Post-task learning extraction
  /ferridyn-memory:context   - Pre-task memory retrieval
  /ferridyn-memory:update    - Correct stale memories
  /ferridyn-memory:decide    - Log decisions with rationale
  /ferridyn-memory:status    - Quick memory overview
  /ferridyn-memory:health    - Memory diagnostics

HOOKS (activate on next Claude Code restart)
  UserPromptSubmit - Recalls memories + injects proactive memory protocol
  PreCompact       - Saves important learnings before conversation compaction
  Stop             - Reflects on session and persists high-level learnings

NEXT STEP
  Restart Claude Code for hooks to take effect.

For more info: https://github.com/AetherXHub/ferridyn-memory-plugin
```

If `ANTHROPIC_API_KEY` is set in the environment, add:

> **Structured memory is active.** fmemory uses 7 predefined category schemas (typed attributes and secondary indexes) with Claude Haiku for natural language parsing and query resolution.

## Help Text

When user runs `/ferridyn-memory:setup --help`, display:

```
FerridynDB Memory Setup

USAGE:
  /ferridyn-memory:setup         Run full setup (verify CLI, build hooks, start server)
  /ferridyn-memory:setup --help  Show this help

WHAT IT DOES:
  1. Checks that fmemory and ferridyn-server are installed in PATH
  2. Checks for Node.js (npm) and ANTHROPIC_API_KEY
  3. Installs npm deps and builds TypeScript hook scripts (tsup)
  4. Creates data directory (~/.local/share/ferridyn)
  5. Starts the FerridynDB server daemon (required — fmemory is server-only)
  6. Verifies round-trip memory storage
  7. 7 predefined category schemas are created automatically on first use (or via `fmemory init`)

PREREQUISITES:
  - fmemory CLI: cargo install --git https://github.com/AetherXHub/ferridyn-memory
  - ferridyn-server: cargo install ferridyn-server --git https://github.com/AetherXHub/ferridyndb
  - Node.js 20+ (for hook scripts)
  - ANTHROPIC_API_KEY environment variable (for NL parsing and query resolution)

AFTER SETUP:
  Restart Claude Code for hooks to activate.

For more info: https://github.com/AetherXHub/ferridyn-memory-plugin
```
