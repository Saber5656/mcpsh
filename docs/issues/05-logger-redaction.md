# Title

Logger with secret redaction

## Summary

Implement `src/log/log.ts` (leveled stderr logger) and `src/log/redact.ts`
(secret redaction by registered values and key patterns), per DESIGN.md §12.5 and
threat T4.

## Context

Config env-refs resolve to secrets (tokens, Authorization headers). Any log line or
error message that could embed them must be scrubbed. The logger is used by every
runtime module; redaction must be impossible to bypass accidentally (the logger
applies it internally).

## Scope

- `src/log/log.ts`, `src/log/redact.ts`, unit tests.

## Detailed Requirements

1. `redact.ts`:
   - `registerSecret(value: string): void` — ignores values shorter than 4 chars
     (avoid redacting everything); stores exact strings in a module-level set.
   - `redact(text: string): string` — two passes:
     a. Replace every occurrence of each registered secret with `[redacted]`.
     b. Key-pattern pass: for text matching
        `/(token|secret|passwd|password|authorization|bearer|api[-_]?key)\s*[=:]\s*("[^"]*"|'[^']*'|\S+)/gi`,
        replace the value portion with `[redacted]`.
   - Longest-secret-first replacement (avoid partial-overlap leaks).
2. `log.ts`:
   - Levels: `error < warn < info < debug < trace`; default threshold `warn`
     (DESIGN.md §11.1). `setLevel(level)`, `getLevel()`.
   - `log.error/warn/info/debug/trace(msg: string)` → write
     `mcpsh [<level>] <redacted msg>\n` to stderr when level enabled. Redaction is
     applied inside the logger — callers cannot skip it.
   - First `trace`-level emission prints a one-time warning banner:
     `trace logging may include payloads and secrets-adjacent data` (§12.5).
   - No timestamps in v1 (stable test output); no file output.
   - Color: level tag styled via `util.styleText` only when stderr is a TTY and
     color enabled (accept an `enableColor(bool)` setter; wiring from CLI in 26).
3. Both modules: `node:` builtins only; no imports from other src modules
   (leaf-level; sanitization is applied by render/error layers, not here).

## Acceptance Criteria

- [ ] Tests: threshold filtering per level; registered-value replacement including
      multiple occurrences and overlapping registrations; key-pattern replacement
      for `Authorization: Bearer xyz`, `api_key=...`, `token: "..."` shapes;
      values <4 chars not registered; trace banner emitted exactly once.
- [ ] A test proves a message logged after `registerSecret("s3cr3t-value")`
      never contains the raw value at any level.

## Validation

`npm test -- log` green; issue 31 later attacks this end-to-end.

## Dependencies

01.

## Non-goals

Log files, structured/JSON logging, timestamps, log rotation, sanitization
(issue 06).

## Design References

DESIGN.md §11.1, §12.3 (T4), §12.5.
