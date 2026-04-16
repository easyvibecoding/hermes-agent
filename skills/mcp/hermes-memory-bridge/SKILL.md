---
name: hermes-memory-bridge
description: Bidirectional memory bridge — exposes Hermes MEMORY.md, USER.md, and session history to any MCP client (Claude Code, Cursor, VS Code, etc.) via a FastMCP stdio server. Includes atomic writes, file locking, prompt-injection scanning, session browsing, and dream-consolidation workflow support.
version: 2.0.0
author: easyvibecoding
license: MIT
metadata:
  hermes:
    tags: [MCP, Memory, Claude Code, Integration, Bridge, Dream]
    related_skills: [native-mcp, claude-code]
---

# Hermes Memory Bridge

Expose Hermes Agent's persistent memory system (MEMORY.md, USER.md) and session history (state.db) to any MCP-compatible client via a lightweight FastMCP stdio server.

## Problem

`hermes mcp serve` exposes conversation browsing tools but does **not** expose memory read/write or session transcript access. External agents like Claude Code cannot access Hermes memory without a custom bridge.

## Solution

A self-contained FastMCP server (`mcp-servers/hermes-memory-mcp.py`) that provides 7 tools:

### Memory Tools

| Tool | Description |
|------|-------------|
| `read_memory(store)` | Read MEMORY.md, USER.md, or both |
| `add_memory_entry(store, entry, old_text)` | Append or substring-replace in a memory store |
| `remove_memory_entry(store, old_text)` | Remove a section or substring from a memory store |
| `memory_status()` | Char usage, section count, and state.db session count |

### Session Tools (Dream Workflow)

| Tool | Description |
|------|-------------|
| `recent_sessions(limit, source)` | List recent sessions for review |
| `session_read(session_id, last_n)` | Read transcript of a specific session |
| `session_search(query, limit, source)` | FTS5 full-text search across all sessions |

## Safety Features

Aligned with Hermes built-in `memory_tool.py` (`tools/memory_tool.py`):

- **Atomic writes** — temp file + `os.replace()` + `fsync` prevents corruption on crash
- **File locking** — `fcntl.flock()` on `.lock` files prevents race conditions between Hermes and MCP clients writing concurrently
- **Prompt-injection scanning** — rejects content containing role hijacking, instruction override, chat-template tokens, or invisible Unicode
- **Read-only state.db** — all session queries use `?mode=ro` URI, no accidental mutation

## Dream Workflow

The session tools enable a **dream consolidation** pattern inspired by Honcho's `on_session_end` and Holographic's auto-extraction:

1. **Review** — `recent_sessions()` to see what happened recently
2. **Read** — `session_read(id)` to get the full transcript
3. **Extract** — The LLM identifies key insights, decisions, or user preferences
4. **Consolidate** — `add_memory_entry()` to persist insights; `remove_memory_entry()` to prune stale entries
5. **Verify** — `memory_status()` to confirm capacity

This can be automated via Claude Code hooks, cron jobs, or manual `/dream` commands.

## Prerequisites

- **Python 3.10+**
- **uv** — for automatic dependency management (`brew install uv` on macOS)
- **mcp[cli]** — installed automatically by the runner script via `uv run --with`

## Quick Start

### 1. Register in Hermes config

Add to `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  hermes-memory:
    command: ~/.hermes/mcp-servers/hermes-memory-runner.sh
    args: []
    enabled: true
```

### 2. Register in Claude Code

Add to `~/.claude.json` (or project-level `.claude/settings.local.json`):

```json
{
  "mcpServers": {
    "hermes-memory": {
      "command": "/bin/bash",
      "args": ["~/.hermes/mcp-servers/hermes-memory-runner.sh"]
    }
  }
}
```

### 3. Verify

In Claude Code:

```
> memory_status()
MEMORY.md: 892/2,200 chars (40.5%) — 5 sections
USER.md: 461/1,375 chars (33.5%) — 5 sections
state.db: 127 sessions (searchable via session_search)
```

## Architecture

```
~/.hermes/
├── memories/
│   ├── MEMORY.md          # General memory (2,200 char limit)
│   └── USER.md            # User profile (1,375 char limit)
├── state.db               # SQLite with FTS5 session index
└── mcp-servers/
    ├── hermes-memory-mcp.py      # FastMCP server (7 tools)
    └── hermes-memory-runner.sh   # Shell wrapper for uv
```

### Memory Format

Sections in MEMORY.md and USER.md are separated by `\n§\n` (newline + section sign + newline). The tools respect this convention — new entries are appended with `§` separators, removals clean up dangling separators, and char limits are enforced before every write.

### Frozen Snapshot Behavior

Hermes injects memory as a frozen snapshot at session start. If Claude Code writes to memory via this bridge, the changes are persisted to disk immediately but will **not** appear in the current Hermes session's context. They will be available when Hermes starts a new session.

### Dual Registration

Hermes and Claude Code read different config files:
- Hermes: `~/.hermes/config.yaml` → `mcp_servers`
- Claude Code: `~/.claude.json` → `mcpServers`

Register the server in both to enable bidirectional access.
