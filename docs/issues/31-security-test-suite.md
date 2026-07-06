# Title

Adversarial security test suite

## Summary

Build `test/security/`: adversarial end-to-end tests that attack each mitigated
threat from the DESIGN.md §12.3 table (T1–T6, T11, T13) through real product
surfaces, so regressions in security behavior fail CI loudly.

## Context

Unit tests in feature issues verify mechanisms in isolation; this suite attacks
the assembled product the way a malicious or compromised MCP server would
(threat model §12). It is a release gate: the security posture documented in
issue 33 cites this suite as evidence.

## Scope

- `test/security/*.test.ts` reusing the e2e harness (issue 30). New fixture
  behaviors only if a listed test needs one.

## Detailed Requirements

One spec per threat, minimum:

1. **T1 terminal injection**: REPL-path render of `fixture.control_chars` output
   (unit-level render call + pty when available) and one-shot TTY-emulated path:
   assert output contains no raw ESC/C1/`\r`/BEL bytes; OSC52 payload
   neutralized. Also: error messages carrying server text (`fail` with ANSI in
   message) are sanitized.
2. **T2 output flood**: `big` beyond limit → `E_LIMIT`, RSS of the mcpsh process
   stays under 512 MiB (coarse guard via `/usr/bin/time -l|-v` or
   `process.memoryUsage` reporting hook — pick the portable option and document);
   external `node -e` flood → same.
3. **T3 shell injection**: pipeline value `"; touch <tmp>/pwned; echo "` piped as
   `$in` into an external `echo` argument and into a tool arg echoed back —
   assert no file created and literal bytes preserved end-to-end.
4. **T4 secret redaction**: config with `Authorization: Bearer ${env:TOK}`;
   force connect failure (401 stub from issue 10 harness) with `--verbose trace`
   logging → assert token value absent from ALL of stdout/stderr; `env` map
   secret likewise (stdio spawn failure path).
5. **T5 history hygiene**: REPL history file mode 0600, parent 0700; leading-space
   line absent from file; secret typed in a normal line IS in the file —
   documented reality check asserting the docs' claim (test cites USAGE security
   note; see issue 32).
6. **T6 env allowlist**: parent env with `SECRET_CANARY`, `AWS_SECRET_ACCESS_KEY`
   → `env_dump` result contains neither; with `inheritEnv: true` contains both
   (opt-out works); config `env` passthrough works.
7. **T11 completion side effects**: completer invoked over a state where a server
   is configured-but-disconnected → manager spy records zero connect attempts;
   no child processes spawned during a completion burst (process-count guard).
8. **T13 overwrite safety**: `save` onto an existing file → `E_ARGS`, file
   unchanged (bytes compared); `--force` replaces atomically (no tmp residue).
9. Negative-control hygiene: each test first proves the attack vector is live
   (e.g., control chars really present in fixture output pre-sanitization via
   `--raw-result` + `to json`), so a broken fixture cannot green-wash the suite.

## Acceptance Criteria

- [ ] All specs pass on macOS/Linux CI; each maps to its threat ID in the test
      name (`T1 ...`).
- [ ] A `test/security/README.md` table: threat ID → spec file → DESIGN.md §12.3
      row (kept in sync check: test iterates the table and asserts every listed
      spec exists).

## Validation

`npm test -- security` green in CI matrix.

## Dependencies

30 (harness), and the features under attack: 05, 06, 07, 09, 10, 21, 25, 27, 28.

## Non-goals

Fuzzing campaigns (v2/continuous), dependency CVE scanning (CI, issue 02),
OS sandboxing of servers (§12.8 residual risk).

## Design References

DESIGN.md §12 (entire), §14.
