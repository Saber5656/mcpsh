# Title

Evaluator core: pipeline execution, expansions, $in, variables, builtin registry

## Summary

Implement `src/lang/eval.ts` (program/pipeline evaluation, expansion semantics,
`$in` binding rules, variable store) and `src/builtins/registry.ts` (builtin
metadata framework) against an abstract `CommandHost` — no MCP or process I/O in
this issue.

## Context

This is the semantic heart of the language (DESIGN.md §7). It must be testable
without MCP: the `CommandHost` interface (§4.2 dependency rule) supplies
resolution and invocation callbacks, mocked in tests here and implemented by
issues 20/21/23.

## Scope

- `src/lang/eval.ts`, `src/builtins/registry.ts`, unit tests with a mock host.

## Detailed Requirements

1. `CommandHost` interface:
   `{ resolve(name: string): Resolved, callTool(ref, args, opts): Promise<Value>,
   runExternal(spec): Promise<Value>, host-provided limits, signal?: AbortSignal }`
   where `Resolved` = builtin | tool ref | external | unknown(suggestions).
   Exact shape designed here and consumed by 20/21; keep it minimal and typed.
2. Builtin registry (`src/builtins/registry.ts`):
   - `defineBuiltin({ name, summary, usage, flags: FlagSpec[], inputPolicy:
     'requires'|'accepts'|'ignores', run(ctx): Promise<Value> })`;
     `ctx = { input: Value|null, args: parsed words, flags: parsed flag values,
     env: EvalEnv, host: CommandHost }`.
   - Generic builtin flag parsing per FlagSpec (boolean vs value-taking; unknown
     flag → `E_ARGS`; `--` terminator respected).
   - Registry lookups by name; `listBuiltins()` for help/completion/config
     collision checks (issue 07 imports the name list from here — export a
     `BUILTIN_NAMES` constant without importing implementations, e.g. a separate
     `names.ts`, to avoid dependency cycles).
3. `EvalEnv`: variable store (Map), `getVar/setVar/unsetVar/listVars`; `$env.NAME`
   reads `process.env` (injected as a readonly record for testability); aliases
   map (used by parser-level expansion — evaluator exposes it, expansion happens
   pre-resolution per §6.4 — implement alias splice here at evaluation entry:
   after parsing, before resolving each command).
4. `evalProgram(program, env, host, opts): Promise<Value>` per §7.1: sequential
   pipelines, first-stage input null, value-source commands (bare `$var`/`$in`
   sole element), final value returned. Abort signal checked between stages →
   `E_CANCELLED`.
5. Expansion (§6.3): implement exactly — bare sole Var/In → Value passthrough;
   concatenated/quoted → stringification rules (null → "", string as-is, others
   compact JSON); unknown var → `E_TYPE` + did-you-mean; `$env.X` unset →
   `E_TYPE`; ENV parts always stringify (they are strings).
6. `$in` binding (§7.2): for tool commands — scan elements for In parts; present →
   substitute; absent + non-null input → `E_UNBOUND_INPUT` (message names the
   tool, hint mentions `$in`). For builtins — enforce `inputPolicy`. For externals
   — In present → substitute and mark stdin-closed; else pass input for stdin
   serialization (the actual serialization is issue 21; evaluator passes
   `{stdinValue}` in the spec).
7. Alias expansion: first-word match → splice stored elements; depth cap 8 →
   `E_PARSE` "alias expansion too deep".
8. Wire builtin execution: resolution returns builtin → parse its flags/words per
   registry spec → `run(ctx)`; result `checkValue`d.

## Acceptance Criteria

- [ ] Mock-host unit tests: sequential pipelines and `;`; value threading through
      3-stage pipelines; value-source `$var | ...` and `$in` passthrough
      (`"x" | echo $in`-equivalent with a test builtin); every expansion rule incl.
      null→""; unknown-var suggestions; `E_UNBOUND_INPUT` for tool with input and
      no `$in`; inputPolicy enforcement (requires/ignores violations); alias
      splice + depth-8 error; abort between stages → `E_CANCELLED`; builtin flag
      parsing (boolean, value, unknown, `--` terminator).
- [ ] A test-only builtin defined via the registry (`test-upper`) demonstrates the
      full ctx contract.
- [ ] No imports from `src/mcp/**` or `node:child_process` (architecture test —
      assert via a lint rule or a unit test reading the file's import list).

## Validation

`npm test -- eval` green; dependency-direction check passes.

## Dependencies

03, 04, 17, 18.

## Non-goals

Real resolution order (20), external spawning (21), MCP calls (12), any
control flow (v2).

## Design References

DESIGN.md §4.2 (dependency rule), §6.3, §6.4, §7.1, §7.2, §7.5.
