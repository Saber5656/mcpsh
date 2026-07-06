# Title

CLI entry and one-shot mode

## Summary

Implement `src/cli.ts` for real: flag parsing, mode selection (REPL vs `-c` vs
stdin program), session assembly (config → manager → host), final-value output
rules, exit codes, and clean shutdown.

## Context

The one-shot surface is the CI/scripting contract (DESIGN.md §11.1–§11.4): stable
flags, stable exit codes, and predictable stdout are the product promise. The REPL
branch delegates to issue 27 (until then: a clear "REPL not yet implemented"
stderr + exit 1 behind the same mode logic).

## Scope

- `src/cli.ts` (+ `src/session.ts` assembling config/manager/registry/host);
  e2e tests spawning the built binary.

## Detailed Requirements

1. Flag parsing (hand-rolled over `process.argv`, §4.3 no-dependency policy),
   exactly the §11.1 set: `-c/--command <program>`, `--config <path>`,
   `--output auto|json|text` (default auto), `--log-level error|warn|info|debug|trace`
   (default warn), `--verbose` (implies log-level debug + stderr-tail surfacing),
   `--no-color`, `--version`, `--help`. Unknown flag / missing value / bad enum →
   usage line on stderr, exit 2. `--help` prints usage (flags + one example) to
   stdout, exit 0. `-c ''` → exit 2.
2. Mode selection per §11.2 exactly (TTY detection via `process.stdin.isTTY`);
   in `-c` mode stdin is never consumed by stages (document; stdin is not wired
   into the host).
3. Color: disabled when `--no-color`, `NO_COLOR` env set, or stdout not a TTY;
   wire `enableColor` (05) and renderer settings (29 when it lands; renderer
   consumed via a narrow interface so 26 merges before 29 with plain JSON output).
4. Session assembly: `loadConfig` (exit 3 on `E_CONFIG`) → manager → registry →
   null interaction host (auto-decline, §9.9) → `buildCommandHost` → parse →
   evalProgram with a SIGINT-wired AbortController (first Ctrl-C aborts; second
   force-exits 130).
5. Output per §11.4: value printing to stdout; `auto`: null → nothing, string →
   raw + `\n`, other → compact JSON + `\n`; `json`: always JSON (pretty iff stdout
   TTY); `text`: `to text` semantics. Sanitization applies only on TTY (§12.6).
6. Errors: `formatError` (04, wired with real redact/sanitize) to stderr; exit via
   `exitCodeFor`. `E_PARSE` before any connection work (parse first, then
   connect lazily anyway).
7. Shutdown: after program completes or fails, `closeAll()` with 3 s cap; process
   must exit even if a server child ignores SIGTERM (SIGKILL path, issue 09);
   explicit `process.exit(code)` after cleanup (no hanging handles).

## Acceptance Criteria

- [ ] e2e (built binary + fixture config): `-c 'fixture.echo --text hi'` → stdout
      `hi\n`, exit 0; `-c 'fixture.add --a 2 --b 3 | get sum'` → `5\n`;
      `echo 'version' | mcpsh` (non-TTY stdin) runs; `--output json` quotes
      strings; null result prints nothing; parse error → exit 2, `mcpsh: E_PARSE`
      on stderr; missing explicit config → exit 3; `fixture.fail --message x` →
      exit 1; SIGINT during `fixture.slow --ms 10000` → exit 130 within 4 s and no
      zombie fixture process; `--version`/`--help`/unknown-flag behaviors;
      NO_COLOR + non-TTY output contains no ANSI bytes.
- [ ] Every §11.3 exit code has at least one e2e assertion.

## Validation

`npm test -- e2e-cli` green on macOS/Linux CI.

## Dependencies

05, 06, 07, 09, 12, 13, 19, 20, 21, 22, 23, 24, 25 (full pipeline path); REPL not
required.

## Non-goals

REPL loop (27), completion (28), pretty rendering/table/progress (29), rc file
(27).

## Design References

DESIGN.md §11.1–§11.4, §12.6, §13.
