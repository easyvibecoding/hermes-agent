---
name: hermes-memory-bridge
description: Expose Hermes MEMORY.md, USER.md, and session history (state.db) to external MCP clients (Claude Code, Cursor, VS Code, etc.) via a FastMCP stdio server. Complements Hermes in-process memory providers by enabling out-of-process read/write access.
version: 1.0.0
author: easyvibecoding
license: MIT
metadata:
  hermes:
    tags: [MCP, Memory, Claude Code, Integration, Bridge]
    related_skills: [native-mcp, claude-code]
---

# Hermes Memory Bridge

Expose Hermes Agent's persistent memory system (MEMORY.md, USER.md) and session history (state.db) to any MCP-compatible client via a lightweight FastMCP stdio server.

## Problem

`hermes mcp serve` exposes conversation browsing tools but does **not** expose memory read/write or session transcript access. External agents like Claude Code cannot access Hermes memory without a custom bridge.

## How This Relates to Existing Memory Consolidation

Hermes already has **in-process** memory lifecycle hooks for session-end consolidation:

- **Honcho** — `on_session_end` flushes pending data; `sync_turn` for per-turn background sync
- **Holographic** — `on_session_end` with `auto_extract` uses regex to extract user preferences and project decisions into a local fact store; `on_memory_write` mirrors built-in memory writes
- **`on_pre_compress`** — all providers can inject text into the compression summary

These mechanisms run **inside** the Hermes process and are triggered automatically by agent lifecycle events. They do not require external access.

This bridge takes a **complementary** approach — it provides **out-of-process read/write access** for MCP clients that coexist alongside Hermes. The session tools expose the same `state.db` data that internal providers consume via `on_session_end(messages)`, but make it available to agents running **outside** the Hermes process.

## Tools (7)

### Memory

| Tool | Description |
|------|-------------|
| `read_memory(store)` | Read MEMORY.md, USER.md, or both |
| `add_memory_entry(store, entry, old_text)` | Append or substring-replace in a memory store |
| `remove_memory_entry(store, old_text)` | Remove a section or substring cleanly |
| `memory_status()` | Char usage, section count, state.db session count |

### Session

| Tool | Description |
|------|-------------|
| `recent_sessions(limit, source)` | List recent sessions |
| `session_read(session_id, last_n)` | Read full transcript of a session |
| `session_search(query, limit, source)` | FTS5 full-text search across all sessions |

## Safety

Aligned with Hermes built-in `tools/memory_tool.py`:

- **Atomic writes** — temp file + `os.replace()` + `fsync` prevents corruption on crash
- **Cross-platform file locking** — `fcntl.flock` (Unix) / `msvcrt.locking` (Windows) / graceful fallback
- **Prompt-injection scanning** — rejects role hijacking, instruction override, chat-template tokens, invisible Unicode
- **Read-only state.db** — all session queries use `?mode=ro`

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

### Important: Frozen Snapshot

Hermes injects memory as a frozen snapshot at session start. If an external MCP client writes to memory via this bridge, the changes are persisted to disk immediately but will **not** appear in the current Hermes session's context. They become visible when Hermes starts a new session.

### Dual Registration

Hermes and Claude Code read different config files:
- Hermes: `~/.hermes/config.yaml` → `mcp_servers`
- Claude Code: `~/.claude.json` → `mcpServers`

Register the server in both to enable bidirectional access.
