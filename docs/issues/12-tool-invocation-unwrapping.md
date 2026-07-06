# Title

Tool invocation: timeouts, cancellation, result unwrapping

## Summary

Implement the invocation half of `src/mcp/tools.ts`: `callTool` with per-server
timeout and AbortSignal, size caps, and the §9.7 result-unwrapping rules including
`isError` → `E_TOOL_FAILED` and the `--raw-result` envelope mode.

## Context

This function is the product's core action. Unwrap rules decide day-to-day
ergonomics; timeout/cancellation decide whether Ctrl-C always works (threat T9);
caps decide memory safety (T2).

## Scope

- `callTool` + unwrap helpers in `src/mcp/tools.ts`; integration tests.

## Detailed Requirements

1. `callTool(alias, tool, args: Record<string, Value>, opts: { raw?: boolean,
   signal?: AbortSignal, onProgress?: (p) => void }): Promise<Value>`:
   - `ensureReady(alias)`, then SDK `callTool` with `timeout: callMs` (per-server
     config), `resetTimeoutOnProgress: true`, progress handler forwarded.
   - External `signal` abort → SDK cancellation (`notifications/cancelled`) and
     `E_CANCELLED`.
   - SDK timeout → `E_PROTOCOL` message `call timed out after <callMs> ms`
     (hint: raise `timeouts.callMs` for <alias>).
   - Tool-not-found / protocol errors → `E_PROTOCOL` (server included).
2. Unwrap (exact order, DESIGN.md §9.7):
   1. `raw` mode → envelope record always (including `isError: true` results —
      no throw), shape:
      `{ content: BlockRecord[], structuredContent?: Value, isError: boolean }`.
   2. `isError` truthy → `ToolFailedError` with server/tool, message = sanitized
      concatenation of text blocks (truncate 500 chars), envelope attached.
   3. `structuredContent` present → `checkValue(structuredContent)`.
   4. Exactly one block and it is `text` → that string.
   5. Else envelope record (as in raw, `isError: false`).
   - `BlockRecord` mapping: text → `{type:'text', text}`; image/audio →
     `{type, mimeType, data, bytes}` (bytes = decoded base64 length; do NOT decode
     into memory — compute from base64 length arithmetic); resource_link →
     `{type:'resource_link', uri, name?, description?}`; embedded resource →
     `{type:'resource', resource:{uri, mimeType?, text?|data?}}`. Unknown block
     types → `{type:<type>, raw:<checkValue'd best effort>}` (forward-compat).
3. Size cap: `assertWithinLimit` (issue 03) on the unwrapped value AND on the
   envelope path, using per-server `limits.maxOutputBytes` override else global.
   Stage label: `<alias>.<tool>`.
4. Result values pass `checkValue` (issue 03); invalid → `E_PROTOCOL` naming the
   server (server sent non-JSON-compatible data).

## Acceptance Criteria

- [ ] Integration tests (fixture): `echo` → string; `add` → structuredContent
      record `{sum:5}`; `multi_content` → envelope with 3 blocks (image block has
      correct mimeType and bytes, data intact base64); `get_structured` variants;
      `fail` → `E_TOOL_FAILED` with message and attached envelope; `fail` with
      raw → envelope with `isError: true`, no throw; `big` over the configured
      limit → `E_LIMIT` naming `<alias>.big`; `slow` with an aborted signal →
      `E_CANCELLED` and (SDK-level) cancellation sent; `slow` beyond a tiny
      `callMs` → timeout `E_PROTOCOL` with hint; progress callback receives ≥1
      update from `slow` with progressToken.
- [ ] Unit tests for unwrap ordering with hand-built results (all 5 rules).

## Validation

`npm test -- tools-call` green; timing-sensitive tests use generous bounds
(timeout 200 ms vs slow 2000 ms).

## Dependencies

03, 04, 09, 11, 08.

## Non-goals

Flag→args mapping (13), `$in` binding (19), rendering (29).

## Design References

DESIGN.md §5.3, §9.5, §9.7, §12.3 (T2, T9, T14).
