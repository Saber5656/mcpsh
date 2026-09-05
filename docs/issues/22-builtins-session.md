# Title

Session builtins: help, version, exit, vars, set, unset, alias, unalias, history

## Summary

Implement `src/builtins/session.ts`: the session/meta builtins per the DESIGN.md
§10 table, using the registry framework from issue 19.

## Context

These builtins manage the session itself. `help` doubles as the discoverability
surface; `set`/`vars` complete the variable feature (§7.5); `history` bridges to
the REPL's store via the host.

## Scope

- `src/builtins/session.ts`, unit tests with mock env/host.

## Detailed Requirements

Per the §10 table, exactly:

1. `help [name]` (ignores input): no arg → string listing every builtin
   (name — summary, grouped: session/server/data/format) plus a footer pointing at
   `tools`, `describe`, and USAGE.md. With arg: builtin → usage + summary + flags;
   qualified tool name → delegate to the registry cache (schema summary like
   `describe`, without connecting; not cached → hint to `connect`); unknown →
   `E_ARGS` with suggestions.
2. `version` (ignores) → string `mcpsh <version>`.
3. `exit` (ignores): host exposes `requestExit()`; in one-shot mode the host
   raises `E_ARGS` "exit is only available in the REPL".
4. `vars` (ignores) → list of `{name, type, size}` records sorted by name
   (type = null/boolean/number/string/list/record; size = `sizeOf`).
5. `set NAME [word]` (accepts): word form stores the expanded word's Value; sink
   form stores the input. Both yield the stored value (§10). Invalid NAME
   (`[A-Za-z_][A-Za-z0-9_]*`) → `E_ARGS`; reserved names `in`, `env` → `E_ARGS`.
6. `unset NAME` (ignores) → null; unknown name is a no-op (idempotent).
7. `alias [NAME words...]` (ignores): no args → list of `{name, expansion}`
   records; with args → define (expansion = the raw element spellings joined by
   single spaces, stored as parsed elements per §6.4); name rules as variables;
   name colliding with a builtin → `E_ARGS`.
8. `unalias NAME` (ignores) → null; unknown → `E_ARGS`.
9. `history [--max n]` (ignores) → list of the last n (default 50) history lines,
   oldest first, via `host.historyProvider`; absent provider (one-shot) →
   `E_ARGS` "history is only available in the REPL".

## Acceptance Criteria

- [ ] Unit tests per builtin covering the exact behaviors above, including: help
      output contains every registered builtin name (dynamic assertion, not a
      snapshot that rots); set sink vs word form; reserved-name rejections; alias
      list/define/collision; history via a stubbed provider and its one-shot
      error; exit via `requestExit` spy and its one-shot error.
- [ ] All builtins registered via `defineBuiltin` with correct `inputPolicy` (a
      meta-test iterates the registry and asserts the §10 table's policies for
      these nine names).

## Validation

`npm test -- builtins-session` green.

## Dependencies

19 (registry framework), 03, 04.

## Non-goals

Server builtins (23), REPL history storage itself (27), alias persistence
(rc file covers it, 27).

## Design References

DESIGN.md §6.4, §7.5, §10.
