# ADR-006: User-level config only in v1 — no automatic project-local config discovery

- Status: Accepted (design decision, 2026-07-06)
- Deciders: design agent (conservative default; security-motivated)

## Context

MCP client configs conventionally live both per-user and per-project (`.mcp.json`).
Automatically loading a project-local config means that entering a cloned repository
and running mcpsh can **spawn attacker-chosen processes** (stdio servers are
arbitrary commands) — the direnv problem. Doing this safely requires a trust/pinning
flow (hash-based approval per file), which is real design and UX work.

## Decision

v1 loads configuration from exactly one file, resolved in this order:

1. `--config <path>` CLI flag
2. `MCPSH_CONFIG` environment variable
3. `$XDG_CONFIG_HOME/mcpsh/config.json` (default `~/.config/mcpsh/config.json`)

There is **no** automatic discovery of `./.mcpsh.json`, `./.mcp.json`, or any other
file in the current directory or its ancestors. Users may point at a project file
explicitly (`mcpsh --config ./.mcp.json`), which is an explicit trust action.

## Consequences

- Positive: `cd untrusted-repo && mcpsh` cannot execute code from that repo.
- Negative: per-project workflows need an explicit flag or env var in v1.
- v2 items (documented): project config discovery gated by a direnv-style
  allow/pinning flow; `mcpsh config import` from Claude Desktop / Claude Code /
  Cursor config formats. The config schema already uses the conventional
  `mcpServers` map to keep such files copy-paste compatible.
