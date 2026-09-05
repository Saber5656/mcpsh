# Title

Value model: types, path access, size accounting, JSON bridge

## Summary

Implement `src/values/value.ts` and `src/values/path.ts`: the pipeline Value type,
validation/normalization of foreign JSON, `a.b.0.c` path access with a MISSING
sentinel, approximate size accounting for limits, and JSON encode/decode wrappers.

## Context

Every stage of every pipeline exchanges Values (DESIGN.md §5). This module is pure
(no I/O) and must be rock-solid; the evaluator, MCP layer, builtins, and renderer
all consume it.

## Scope

- `src/values/value.ts`, `src/values/path.ts`, unit tests.

## Detailed Requirements

1. Types per DESIGN.md §5.1: `type Value = null | boolean | number | string |
   Value[] | { [k: string]: Value }`. Export type guards `isRecord`, `isList`,
   `isBlobRecord` (checks the §5.1 blob convention shape).
2. `checkValue(x: unknown): Value` — accepts JSON-compatible input; rejects with
   `E_TYPE` (`McpshError` from issue 04): non-finite numbers, functions, symbols,
   class instances (non-plain objects), `undefined` inside arrays (converted to
   null? **No** — reject; explicitness over coercion). Plain objects are shallow-
   copied into insertion-ordered records; nested structures validated recursively.
3. `pathGet(value: Value, path: string): Value | MISSING`:
   - Path grammar: segments separated by `.`; segment = identifier
     (`[A-Za-z_][A-Za-z0-9_-]*`) or non-negative integer. Empty path or malformed
     segment → `E_TYPE` (this is a caller/program error, not MISSING).
   - Integer segment on a list indexes it; identifier segment on a record reads the
     key; any segment on other types, out-of-range index, or absent key → `MISSING`
     (exported unique symbol).
4. `sizeOf(value: Value): number` — approximate UTF-8 byte size: strings
   `Buffer.byteLength`, numbers/booleans/null fixed 8, lists/records sum of
   children + 2 per entry + key sizes. Deterministic; documented as approximate.
5. `assertWithinLimit(value, limitBytes, stageLabel)` → throws `E_LIMIT` naming the
   stage and limit when `sizeOf` exceeds it.
6. `toJson(value, opts?: {pretty?: boolean}): string` and
   `fromJson(text: string): Value` — `fromJson` wraps parse errors in `E_TYPE`
   including a sanitized excerpt (≤120 chars) of the offending text (use issue 06
   `sanitize`; a plain truncation fallback is acceptable until 06 lands, replaced
   when integrating).
7. No dependencies outside `node:` builtins and `src/errors`.

## Acceptance Criteria

- [ ] Unit tests cover: all type guards; `checkValue` acceptance and each rejection
      class; `pathGet` for records/lists/mixed paths/MISSING cases/malformed path
      errors; `sizeOf` monotonicity (superset ≥ subset); `assertWithinLimit` throw
      and pass; `toJson`/`fromJson` round-trip; `fromJson` error excerpt truncation.
- [ ] 100% of exported functions have tests; `npm test` green.

## Validation

`npm test -- values` green on Node 22/24; review that MISSING never leaks into
Values (type-level: `pathGet` return type is `Value | typeof MISSING`).

## Dependencies

01, 04 (error classes; may be developed in parallel against the agreed class names).

## Non-goals

Path *setting*, deep equality, streaming, binary type (blob is a record convention).

## Design References

DESIGN.md §5; ADR-002.
