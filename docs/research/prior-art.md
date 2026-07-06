# Research: Prior Art and Naming Collision

Status: Completed 2026-07-06. Informs DESIGN.md §2 (goals/differentiation) and ADR-008 (npm naming).

## Purpose

Survey existing MCP command-line clients to (a) position mcpsh, (b) avoid re-inventing
solved problems, and (c) detect naming collisions before public release.

## Naming collision (action required before npm release)

The npm package name `mcpsh` is **already taken**:

| Fact | Value |
|---|---|
| npm package | `mcpsh@0.2.0` |
| Description | "A minimal CLI client for the Model Context Protocol" |
| Author / repo | morinokami — <https://github.com/morinokami/mcpsh> |
| First published | 2025-03-30 (last modified 2025-04-03) |
| Scope | Interactive shell for **one** stdio server per invocation; raw JSON-RPC-style method invocation (`tools/call {"name": ...}`); tab completion of method names; TypeScript |

Consequences:

- Publishing this project to npm as `mcpsh` is impossible without the name being
  transferred or abandoned (npm dispute process is slow and not guaranteed).
- GitHub repo name `Saber5656/mcpsh` does not conflict technically, but search
  discoverability collides with `morinokami/mcpsh`.

Decision: see [ADR-008](../decisions/ADR-008-npm-name-collision.md) — v1 development
proceeds with binary name `mcpsh` and `"private": true` in package.json; the public
distribution name is a human release-gate decision.

## Comparative survey

| Tool | Language | Model | REPL | Own pipeline language | Multi-server session | Transports | LLM built in |
|---|---|---|---|---|---|---|---|
| `morinokami/mcpsh` (npm `mcpsh`) | TypeScript | Raw method invocation against one server | Yes (minimal) | No | No | stdio | No |
| `f/mcptools` (`mcpt`) | Go | Subcommands (`mcpt call ...`) + basic shell mode | Basic | No | No (one server per invocation) | stdio, HTTP | Optional (guard/proxy features) |
| `apify/mcpc` | TypeScript | Subcommands over persistent named sessions (`mcpc @sess tools-call ...`) | **No** (explicit non-goal) | **No** — composition delegated to bash/jq/xargs | Yes (named sessions via bridge processes, `~/.mcpc/sessions.json`, keychain creds) | stdio, Streamable HTTP, OAuth 2.1 | No |
| `chrishayuk/mcp-cli` | Python | Chat/interactive client oriented around LLM providers | Yes (chat) | No | Yes | stdio, HTTP | Yes (core feature) |
| `philschmid/mcp-cli`-style minimal CLIs | various | One-shot call helpers for agents | No | No | No | stdio | No |
| Nushell (non-MCP) | Rust | General shell with structured-data pipelines | Yes | Yes (reference design) | n/a | n/a | No |

Key observations:

1. **The gap is real.** Every existing MCP CLI either (a) has no REPL, (b) scopes a
   session to a single server, or (c) delegates all composition to bash + jq. None
   offers a shell where structured values flow *between MCP tools of different
   servers* as first-class pipeline stages.
2. `apify/mcpc` is the closest competitor and explicitly documents "no interactive
   REPL" and "no custom pipeline language" as design choices. mcpsh takes the
   opposite bet: composition ergonomics inside one process, Nushell-style.
3. Nushell is the strongest *design* reference for hybrid structured pipelines
   (structured values internally, serialization at external-command boundaries,
   `$in`, explicit `^` external sigil). mcpsh borrows these ideas at much smaller
   scope (no control flow, no custom types, no plugins in v1).
4. Session persistence across *processes* (mcpc's bridge daemons) is deliberately
   out of scope for v1: mcpsh sessions live and die with the shell process, which
   keeps the security and lifecycle model simple. Deferred to v2 evaluation.

## What mcpsh borrows

| Idea | Source | Adopted as |
|---|---|---|
| Structured values in pipes, serialize at process boundary | Nushell | DESIGN.md §5, §7 |
| `$in` pipeline-input reference | Nushell | DESIGN.md §7.2 |
| `^cmd` to force external command | Nushell | DESIGN.md §8 |
| `server.tool` namespacing | mcpt / mcpc flat command mapping | DESIGN.md §9.4 |
| `--json` escape hatch for tool args | mcpt, mcpc | DESIGN.md §9.6 |
| JSON-to-stdout for non-TTY composition | mcpc `--json` | DESIGN.md §11.4 |
| Env-var references instead of secrets in config | common MCP client config practice | DESIGN.md §12.5 |

## Sources

- <https://www.npmjs.com/package/mcpsh> (registry metadata verified via `registry.npmjs.org/mcpsh`, 2026-07-06)
- <https://github.com/morinokami/mcpsh>
- <https://github.com/f/mcptools>
- <https://github.com/apify/mcpc>
- <https://github.com/chrishayuk/mcp-cli>
- <https://www.philschmid.de/mcp-cli>
- <https://www.nushell.sh> (design reference)
