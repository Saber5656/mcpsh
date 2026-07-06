# Title

External command execution with serialization boundary

## Summary

Implement `src/pipeline/external.ts`: spawn external commands from argv arrays
(never a shell), serialize pipeline input to stdin per the §7.3 table, capture
stdout with size cap, pass stderr through, and map failures and cancellation.

## Context

This is the Unix half of the hybrid pipeline (ADR-002) and the module where
ADR-005's no-shell guarantee lives (threat T3). Serialization rules must match
DESIGN.md §7.3 exactly — they are part of the language contract.

## Scope

- `src/pipeline/external.ts`, unit/integration tests (portable commands only:
  `cat`, `sort`, `tr`, `sh -c` where explicitly testing user-written shell,
  `node -e`).

## Detailed Requirements

1. `runExternal(spec: { command: string, argv: string[], stdinValue: Value | null,
   stdinClosed: boolean, limits, signal?: AbortSignal }): Promise<Value>`.
   The evaluator/host builds `argv` from element expansion + raw flag spellings
   (§7.4); this module never re-tokenizes anything (ADR-005).
2. Spawn: `child_process.spawn(command, argv, { shell: false, env: process.env,
   stdio: ['pipe'|'ignore', 'pipe', 'inherit'] })`. PATH resolution is spawn's.
   ENOENT → `E_UNKNOWN_COMMAND` (command name; happens when PATH probe raced),
   EACCES → `E_EXTERNAL_FAILED` with a permissions hint.
3. stdin per §7.3: `stdinClosed` or null value → close immediately; string →
   write verbatim (no added newline); list of all-strings → each + `\n`; list of
   records → JSONL; anything else → compact JSON + `\n`. Write errors after child
   exit (EPIPE) are ignored (grep -q semantics).
4. stdout: accumulate with `limits.maxOutputBytes` cap → on breach, kill child
   (SIGTERM→SIGKILL 3 s), throw `E_LIMIT` (stage = command name). Decode UTF-8
   with replacement chars; strip exactly one trailing `\n` if present; result is a
   string Value.
5. stderr: inherited (live passthrough). Document: stderr is not part of pipeline
   data.
6. Exit: code 0 → return value; non-zero → `E_EXTERNAL_FAILED{command, exitCode}`
   message `<cmd> exited with code N`; killed by signal → include signal name.
7. Cancellation: `signal` abort → SIGTERM, SIGKILL after 3 s → `E_CANCELLED`.
8. Zombie prevention: always `await` full child close (not just exit) before
   resolving; unref nothing.

## Acceptance Criteria

- [ ] Tests: `cat` round-trips a string exactly (no added newline — verify with a
      string lacking trailing newline); list-of-strings → `sort` → expected order;
      list-of-records → `cat` → JSONL bytes asserted; record → `cat` → compact
      JSON + `\n`; trailing-newline strip (echo output); non-zero exit →
      `E_EXTERNAL_FAILED` with code; ENOENT → `E_UNKNOWN_COMMAND`; cap breach with
      `node -e 'console.log("x".repeat(...))'` → `E_LIMIT` and child gone; abort
      mid-`sleep`-equivalent (`node -e 'setTimeout(()=>{},10000)'`) → `E_CANCELLED`
      within 4 s and child gone; **no-shell proof**: argv `["echo; touch /tmp/pwned"]`
      to `/bin/echo` prints the literal string and creates no file; stdinClosed
      with `cat` → empty string result.
- [ ] Zombie check in suite teardown (no lingering children by pgid or pid list).

## Validation

`npm test -- external` green on macOS and Linux.

## Dependencies

03, 04, 19 (spec shape agreed with evaluator/host).

## Non-goals

fd-level streaming between adjacent externals (v2, ADR-002), PTY allocation for
interactive externals (v1 externals are non-interactive; document), Windows.

## Design References

DESIGN.md §7.3, §7.4, §12.3 (T2, T3); ADR-002, ADR-005.
