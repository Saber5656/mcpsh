# Title

Error taxonomy: McpshError classes, exit-code mapping, formatting

## Summary

Implement `src/errors.ts`: the `McpshError` base class, one subclass per error code
from DESIGN.md §13, the error→exit-code mapping, and the single-line formatter with
optional hint and verbose cause chain.

## Context

Every module raises typed errors; the CLI and REPL render them uniformly and map
them to exit codes (DESIGN.md §11.3, §13). Centralizing this early prevents ad-hoc
`throw new Error` drift across issues.

## Scope

- `src/errors.ts`, unit tests.

## Detailed Requirements

1. `class McpshError extends Error { code: ErrorCode; hint?: string;
   cause?: unknown }` with `ErrorCode` union exactly: `E_PARSE`, `E_CONFIG`,
   `E_CONNECT`, `E_PROTOCOL`, `E_UNKNOWN_COMMAND`, `E_ARGS`, `E_TYPE`,
   `E_UNBOUND_INPUT`, `E_TOOL_FAILED`, `E_EXTERNAL_FAILED`, `E_LIMIT`,
   `E_CANCELLED`.
2. Subclasses with typed extra fields (DESIGN.md §13 table):
   `ParseError{line,col}`, `ConfigError{jsonPath?}`, `ConnectError{server}`,
   `ProtocolError{server}`, `UnknownCommandError{name,suggestions:string[]}`,
   `ArgsError`, `PipelineTypeError`, `UnboundInputError`, `ToolFailedError{server,
   tool, envelope?}`, `ExternalFailedError{command, exitCode?, signal?}`,
   `LimitError{limitBytes, stage}`, `CancelledError`.
3. `exitCodeFor(err: unknown): number` — `E_PARSE`→2, `E_CONFIG`→3,
   `E_CANCELLED`→130, all other `McpshError`→1, non-McpshError→1.
4. `formatError(err: unknown, opts: {verbose: boolean, sanitize: (s:string)=>string,
   redact: (s:string)=>string}): string`:
   - Line 1: `mcpsh: <CODE>: <message>`; line 2 (if hint): `hint: <hint>`.
   - `ParseError` message prefixed with `line <line>, col <col>: `.
   - verbose adds `caused by: ...` chain (max depth 5) and, for `ToolFailedError`,
     a pretty-printed envelope; for `ConnectError` with an attached stderr tail
     (string on `cause`), that tail.
   - Every emitted string passes through the injected `redact` then `sanitize`
     callbacks (dependency-injected so this module stays leaf-level; real
     implementations from issues 05/06 are wired by callers).
   - Non-McpshError input: `mcpsh: E_INTERNAL: unexpected error: <message>` —
     also add `E_INTERNAL` handling to `exitCodeFor` (→1) but NOT to the public
     `ErrorCode` union (internal-only formatting path).
5. `wrap(err: unknown, fallback: () => McpshError): McpshError` helper: returns
   `err` if already `McpshError`, else fallback with `cause` set.

## Acceptance Criteria

- [ ] Unit tests: one construction + `exitCodeFor` + `formatError` case per code;
      hint rendering; verbose cause chain depth cap; redact/sanitize callbacks
      demonstrably applied (spy functions); non-McpshError path.
- [ ] No imports besides `node:` builtins (leaf module).

## Validation

`npm test -- errors` green; grep shows no other module defines error classes.

## Dependencies

01.

## Non-goals

Actual redaction/sanitization logic (05/06), logging (05), process.exit calls
(CLI, 26).

## Design References

DESIGN.md §11.3, §13.
