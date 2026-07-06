# Title

End-to-end integration suite

## Summary

Build the cross-cutting e2e suite (`test/e2e/`): full user journeys through the
built CLI against the fixture server (stdio + HTTP), mirroring the DESIGN.md §3
transcripts, covering failure modes, signals, and a guarded REPL pty smoke test.

## Context

Unit/module tests (issues 03–29) prove parts; this suite proves the composition
the user actually runs, and pins the §3 transcripts as executable contracts
(DESIGN.md §14).

## Scope

- `test/e2e/journeys.test.ts`, `test/e2e/failures.test.ts`,
  `test/e2e/repl-pty.test.ts`, shared harness `test/e2e/harness.ts`.

## Detailed Requirements

1. Harness: build once (`npm run build` precondition — vitest globalSetup);
   `runMcpsh(args, {stdin?, env?, timeoutMs})` → `{stdout, stderr, code}`;
   temp config generator for fixture servers (stdio and HTTP variants; HTTP
   harness starts the fixture http entry and injects the port).
2. Journeys (each is one test, asserting stdout bytes and exit 0):
   - §3.2 analog: `fixture.get_structured --kind list | where n > 1 | select n
     name | to json` (adjust to fixture data contract fixed in issue 08).
   - Multi-server: two aliases, tool from each in one pipeline.
   - External mix: `fixture.get_structured --kind list | to jsonl | ^cat |
     from jsonl | length`.
   - Variables: `-c 'fixture.echo --text hi | set x; $x'` → `hi`.
   - Resources/prompts journeys (`resource-read`, `prompt-get`).
   - HTTP transport journey (same echo pipeline over the HTTP fixture).
   - stdin program mode (`echo '...' | mcpsh`).
3. Failure modes (assert exit code + stderr `E_` code):
   - Unknown command (with suggestion text), parse error (exit 2), config schema
     error (exit 3), unreachable stdio server (bad command), unreachable HTTP
     (closed port), tool `isError` (exit 1), `E_UNBOUND_INPUT` case, external
     non-zero exit, `E_LIMIT` via `big` with lowered config limit, callMs timeout
     via `slow`.
   - SIGINT: send after 300 ms during `slow --ms 10000` → exit 130 < 4 s; fixture
     child process confirmed gone (poll pid).
4. REPL pty smoke (`repl-pty.test.ts`): uses `node-pty` as an **optional**
   devDependency; `describe.skipIf(!ptyAvailable)`. Script: start REPL with
   fixture config → wait for banner → send `fixture.echo --text hi\r` → expect
   `hi` in output → send `exit\r` → clean exit. CI: attempt install; test skips
   (visibly, with a logged reason) where unavailable. This is the only pty test
   in v1 (DESIGN.md §14).
5. Every test hermetic: temp HOME/XDG dirs, no network beyond 127.0.0.1, no
   reliance on developer machine state; suite runtime target <90 s.

## Acceptance Criteria

- [ ] All journeys and failure tests pass on macOS and Linux CI on Node 22/24.
- [ ] Transcript examples in DESIGN.md §3 that are fixture-expressible exist as
      tests (§3.1 REPL flow via pty where available; §3.2 fully).
- [ ] Suite leaves no processes or temp files (teardown asserts).

## Validation

`npm test -- e2e` green in CI matrix; runtime under target.

## Dependencies

08, 26, 27 (REPL smoke), 29 (rendering-stable assertions use --no-color), and
transitively the full feature set.

## Non-goals

Adversarial security tests (31), performance benchmarks, Windows.

## Design References

DESIGN.md §3, §11.2–§11.4, §14.
