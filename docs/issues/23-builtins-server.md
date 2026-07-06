# Title

Server builtins: servers, connect, disconnect, tools, describe, call, resources, prompts

## Summary

Implement `src/builtins/server.ts`: the MCP-facing builtins from the DESIGN.md §10
table, as thin adapters over the manager (09/10), registry (11), invocation
(12/13), resources (14), and prompts (15) modules.

## Context

These builtins are the user-visible MCP surface. All protocol logic already exists
in the mcp/ modules; this issue is wiring plus argument handling plus output
shaping (records/lists per §10, rendering-agnostic).

## Scope

- `src/builtins/server.ts`, integration tests with fixture (through evalProgram +
  real host).

## Detailed Requirements

Exactly per §10:

1. `servers` (ignores) → list of records
   `{alias, transport, target?, state, protocol?, tools?}` — `target` = command
   basename (stdio) or host (http); `tools` = cached count or absent; sorted by
   alias. Never connects.
2. `connect <alias>` (ignores) → the server's record (as above) after
   `manager.connect`; errors propagate (`E_CONNECT` etc.).
3. `disconnect <alias>` (ignores) → null.
4. `tools [alias] [--refresh]` (ignores) → list of
   `{server, name, title?, description?}`; with alias → that server (lazy connect
   allowed); without → union of **connected** servers' caches (no connects),
   sorted by server then name.
5. `describe <server.tool>` (ignores) → record `{server, name, title?,
   description?, inputSchema?, outputSchema?, annotations?}` from cache (connect
   allowed for qualified form — it is an explicit user action).
6. `call <name> [flags...] [--raw-result]` (input per §7.2) — forces MCP
   interpretation; `<name>` may be any word (quoted names with spaces/dots work);
   qualified required unless unqualified-unique (reuse resolution, issue 20);
   args mapping via issue 13 with the same reserved flags; `$in` behavior
   identical to direct tool invocation (the evaluator treats `call`'s trailing
   elements as the tool's elements — implement via a registry escape hatch:
   `call` is registered with `rawElements: true` so the framework hands it
   unparsed elements).
7. `resources [alias]`, `resource-read <uri> [--server a]`, `prompts [alias]`,
   `prompt-get <server.name> [--arg k=v ...]` — direct adapters to issues 14/15;
   `--arg` may repeat; malformed `k=v` → `E_ARGS`.

## Acceptance Criteria

- [ ] Integration tests (fixture, via full evalProgram): `servers` before/after
      connect shows state change and never connects by itself; `tools` union
      excludes disconnected server; `--refresh` observed; `describe` includes
      inputSchema for `add`; `call fixture.echo --text hi` → `"hi"`;
      `call 'fixture.weird.name'` works; `"x" | call fixture.echo --text $in` →
      `"x"`; `--raw-result` returns envelope for `fail` without throwing;
      `resource-read memo://greeting` and ambiguity error path; `prompt-get
      fixture.greet --arg name=World` → messages record; every builtin rejects
      unknown flags.
- [ ] Meta-test: §10 inputPolicy table asserted for these builtins.

## Validation

`npm test -- builtins-server` green.

## Dependencies

09, 10, 11, 12, 13, 14, 15, 19, 20.

## Non-goals

Rendering (`servers` table formatting is issue 29's `table` + §11.5; here it
returns records), completion (28).

## Design References

DESIGN.md §8, §9.4–§9.8, §10.
