# Title

MCP connection manager: stdio transport and lifecycle FSM

## Summary

Implement `src/mcp/manager.ts`: the per-server connection state machine
(configured → connecting → ready → failed/closed), lazy connect, stdio transport
with env allowlist, stderr ring buffer, timeouts, and graceful shutdown.

## Context

This is the heart of the MCP layer (DESIGN.md §9.3). Security requirements: env
allowlist (threat T6), single-shot connects (T9), stderr capture for diagnosis
without noise.

## Scope

- `src/mcp/manager.ts` (FSM + stdio); HTTP variant is issue 10. Integration tests
  against the fixture server.

## Detailed Requirements

1. `class McpManager` constructed from `ResolvedConfig` (issue 07):
   - `list(): ServerInfo[]` — `{ alias, transport: 'stdio'|'http', state,
     protocolVersion?, toolCount?, lastError? }`.
   - `ensureReady(alias): Promise<Connection>` — lazy connect implementing the FSM
     per DESIGN.md §9.3; concurrent `ensureReady` calls for the same alias share
     one in-flight attempt.
   - `connect(alias)` (idempotent, single retry semantics per §9.3),
     `disconnect(alias)`, `closeAll()` (≤3 s total, parallel, never throws).
   - Unknown alias → `E_ARGS` with known aliases; `enabled: false` →
     `E_CONNECT` "server is disabled in config".
2. stdio transport:
   - SDK `StdioClientTransport` with `command`, `args`, `cwd`, and env built as:
     allowlist from parent (`PATH`, `HOME`, `USER`, `LOGNAME`, `SHELL`, `TERM`,
     `LANG`, `TMPDIR`, every `LC_*`) merged with config `env` (values through
     `resolveEnvRefs`, issue 07). `inheritEnv: true` → full parent env as base.
   - stderr: pipe and retain the last 100 lines per server (ring buffer);
     accessible via `getStderrTail(alias)`; on connect failure include the tail in
     the `ConnectError` `cause`; at `debug` level stream lines to the logger
     prefixed `[<alias>] `.
3. Initialization: SDK `Client` with `clientInfo { name: "mcpsh", version }`,
   capabilities: none beyond elicitation (registered by issue 16 — expose a
   capability-registration hook so 16 can add it without editing this file's
   logic). Record negotiated `protocolVersion`.
4. Connect timeout `connectMs` (per-server config): on timeout, kill transport,
   state `failed`, `E_CONNECT` message includes timeout value and stderr tail.
5. Crash/EOF while ready → state `failed` with stored error; next `ensureReady`
   attempts a fresh connect (this is the "single attempt per trigger" rule — no
   timer-based auto-reconnect).
6. Shutdown sequence per server: SDK `close()`; if the child survives 3 s →
   SIGTERM, +3 s → SIGKILL. `closeAll()` used by CLI/REPL exit paths.
7. Events: emit `stateChanged(alias, state)` (plain EventEmitter) — consumed later
   by REPL rendering; no direct render imports here.

## Acceptance Criteria

- [ ] Integration tests (fixture over stdio): lazy connect on first `ensureReady`;
      `list()` state transitions observed; protocol version recorded; disabled
      server error; unknown alias error; connect timeout (point config at
      `node -e 'setTimeout(...,60000)'` — a command that never speaks MCP) yields
      `E_CONNECT` within `connectMs`+500 ms and includes stderr tail; `env_dump`
      via a raw SDK call shows exactly the allowlist + configured env (canary
      `SECRET_CANARY` set in parent is absent; present with `inheritEnv: true`);
      crash recovery: kill the child, observe `failed`, `ensureReady` reconnects;
      `closeAll()` leaves no child processes (assert via pid liveness).
- [ ] No automatic retry without a new user-triggered call (test: after failure,
      state stays `failed` for 500 ms with no reconnect).

## Validation

`npm test -- manager` green on macOS/Linux; no zombie processes after suite
(asserted in test teardown).

## Dependencies

01, 04, 05, 07, 08.

## Non-goals

HTTP transport (10), tool listing/calls (11/12), elicitation handler content (16),
REPL display (27/29).

## Design References

DESIGN.md §9.3, §12.3 (T6, T9), §12.4; ADR-007.
