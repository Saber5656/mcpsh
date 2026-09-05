# Title

Command resolution: builtin → tool → external order with suggestions

## Summary

Implement `src/lang/resolve.ts`: the production `CommandHost.resolve`
implementation wiring the builtin registry, tool registry, and external fallback
per DESIGN.md §8, including `^` handling and did-you-mean suggestions.

## Context

Resolution order is a UX and security decision (no auto-connect for unqualified
names, no accidental external execution of typo'd tool names — suggestions guide
users). This glues the language layer to the MCP layer while keeping both
import-independent (host pattern, §4.2).

## Scope

- `src/lang/resolve.ts` + `buildCommandHost(...)` factory assembling resolve +
  callTool (12/13) + runExternal (21); unit + integration tests.

## Detailed Requirements

1. `resolve(name, {external}: {external: boolean})`:
   - `external` (from `^`) → `{kind:'external', command: name}` unconditionally.
   - Builtin registry exact match → `{kind:'builtin', builtin}`.
   - Qualified tool: `registry.resolveQualified(name)` (issue 11) →
     `{kind:'tool', alias, tool}` (lazy connect happens at call time).
   - Unqualified: `registry.resolveUnqualified(name)`:
     unique → tool; ambiguous → `E_UNKNOWN_COMMAND` with message listing qualified
     candidates (sorted) and hint `qualify as <alias>.<name>`.
   - Else → `{kind:'external', command: name}` — BUT only when `name` looks
     runnable: contains `/` or is found on PATH (check via `node:fs` X_OK scan of
     PATH entries; no spawning). Not found → `E_UNKNOWN_COMMAND` with suggestions.
2. Suggestions: Levenshtein distance ≤2 candidates from: builtin names, alias
   names, cached qualified tool names, cached unambiguous unqualified tool names.
   Cap 5, sorted by distance then alpha. Implement distance locally (~20 lines, no
   dependency).
3. `buildCommandHost(config, manager, registry, interactionHost, opts)` factory:
   returns the full `CommandHost` used by CLI/REPL; wires `callTool` to
   issue-12/13 modules (args mapping applied to tool commands' elements — the
   evaluator hands the host pre-expanded elements for tool calls; define this
   boundary here precisely: host receives `{alias, tool, elements}` and performs
   args mapping via issue 13, obtaining the schema from the registry cache when
   available, undefined otherwise).
4. PATH probing must be cached per resolve call (not per session) and must not
   follow symlink loops (rely on fs.accessSync semantics).

## Acceptance Criteria

- [ ] Tests with two fixture aliases: builtin shadowing (a tool named `help` on a
      server resolves to the builtin; qualified `srv.help` reaches the tool);
      qualified dotted tool `srv.weird.name` resolves; unqualified unique/
      ambiguous/none paths; ambiguous error message lists both qualified names;
      unknown name with a near-miss builtin (`serverz` → `servers`) suggests
      correctly; `^ls` forces external even when an `ls` tool exists; unqualified
      name matching a *disconnected* server's tool resolves external-or-unknown
      (never connects — assert manager states); PATH probe: name not on PATH →
      `E_UNKNOWN_COMMAND`; name on PATH (use `sh`) → external.
- [ ] `buildCommandHost` integration test: end-to-end `fixture.echo --text hi`
      through evalProgram returns `"hi"`.

## Validation

`npm test -- resolve` green.

## Dependencies

11, 12, 13, 19 (host interface), 21 (runExternal wiring may land as a stub that
throws until 21 merges — acceptable, noted in the test plan).

## Non-goals

Completion (28), alias definition semantics (19/22), spawning logic (21).

## Design References

DESIGN.md §4.2, §8, §11.7 (no side effects from resolution).
