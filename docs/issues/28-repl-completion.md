# Title

REPL tab completion

## Summary

Implement `src/repl/complete.ts`: the readline completer per the DESIGN.md §11.7
table — builtins, aliases, server prefixes, cached tool names, schema-derived
flags, variables, and file paths — with zero side effects.

## Context

Completion is the discoverability engine of the shell. The hard requirement is
threat T11: completion must never connect servers or spawn processes; it reads
caches only.

## Scope

- `src/repl/complete.ts`, unit tests (pure function over session state).

## Detailed Requirements

1. Signature compatible with readline: `completer(line: string): [string[],
   string]` — implement as a pure function over injected state
   `{builtins, aliases, vars, servers (alias+state), toolCache, resourceCache,
   schemaFor(qualified)}` so tests need no readline.
2. Tokenize the line with the real lexer in a lenient mode (unterminated final
   token treated as the completion target; lexer errors before the cursor → no
   completions). Completion target = trailing partial word (possibly empty after
   whitespace).
3. Candidate rules per §11.7 (first-word vs flag position vs path arguments vs
   `$`-prefix), plus:
   - `server.` prefix: after a configured alias + dot, complete that server's
     cached tools; not connected → return no candidates (the once-per-server dim
     notice is rendered by the loop, not the completer — expose
     `lastMiss: {server} | null` in the result for the loop to consume).
   - Flag position (current command resolved to a tool with cached schema or a
     builtin): complete `--property` names minus flags already present in the
     command; boolean properties complete without trailing space hint (client
     detail: return name only).
   - Path completion for `open`/`save` first positional and any `--config` flag
     value: filesystem readdir of the dirname prefix (this filesystem read is
     allowed — it is the user's own machine and standard shell behavior; no
     network/subprocess).
   - `$` prefix: defined vars + `in` + `env.` (no env var name enumeration —
     avoid dumping the environment; complete `env.` literally then stop).
4. Sorting: case-sensitive alpha; builtins before tools on the first word when
   both match (stable groups: builtins, aliases, tools, server-prefixes).
5. Never throw: any internal error → `[[], line]` (completion silently empty);
   assert via fuzz test.

## Acceptance Criteria

- [ ] Unit tests: first-word completion contains builtins + configured
      `server.`-prefixes + cached qualified names; `fixture.ec` → `fixture.echo`;
      disconnected server yields empty + `lastMiss`; flag completion from `add`'s
      schema minus already-typed flags; builtin flag completion (`tools --re` →
      `--refresh`); path completion in a temp dir (subdir + file, prefix filter);
      `$` completions; empty-line completion; mid-token cursor not supported
      (readline gives whole line — document); fuzz: 500 random lines → no throw.
- [ ] Side-effect proof: state object instrumented — no method besides cache
      reads/readdir is called (spy assertions; manager absent from the interface
      entirely).

## Validation

`npm test -- complete` green.

## Dependencies

17 (lexer), 11 (cache shapes), 27 (loop wiring — completer lands as pure module;
one-line wiring change in loop.ts included here).

## Non-goals

MCP `completion/complete` argument-value completion (v2), fuzzy matching,
history-based suggestions.

## Design References

DESIGN.md §11.7, §12.3 (T11).
