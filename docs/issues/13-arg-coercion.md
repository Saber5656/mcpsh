# Title

Flag-to-argument mapping with schema-aware coercion

## Summary

Implement `src/mcp/args.ts`: convert parsed command elements (flags + reserved
flags + `--json`) into a JSON arguments object for a tool call, coercing values by
the tool's inputSchema, with precise `E_ARGS` errors.

## Context

DESIGN.md §9.6 fixes the mapping rules (boolean flags never consume words,
`--json` merge semantics, duplicate detection, reserved flags). This is a pure
module: it receives already-expanded values, no I/O.

## Scope

- `src/mcp/args.ts`, unit tests. (Evaluator wires it in issue 19/20; `call`
  builtin in 23.)

## Detailed Requirements

1. Input: `elements: Array<{kind:'word', value: Value} | {kind:'flag',
   name: string, inline?: Value}>` (post-expansion — a bare `$var` flag value
   arrives as its typed Value), plus `inputSchema?: JsonSchemaish` (object with
   `properties`/`required`; treat absent/malformed schema as schema-less).
2. Reserved flags handled first: `--json <value>` (exactly one; string value → must
   parse as a JSON object via `fromJson`, non-object → `E_ARGS`; record value
   accepted as-is), `--raw-result` (boolean, returned as separate option, never in
   args).
3. Mapping loop over remaining elements:
   - `word` elements → `E_ARGS` "tools take --flag arguments (positional words are
     not supported)" — EXCEPT words that were `$in` references, which the evaluator
     resolves before this module (it never sees them; document the contract).
   - flag with schema property of type `boolean`: no inline → `true`; inline
     `true`/`false` → parsed; inline anything else → `E_ARGS`. Never consumes the
     next element.
   - flag with non-boolean schema property: value = inline if present, else the
     next `word` element (consumed); missing → `E_ARGS` "flag --x requires a value".
     Coercion by schema `type`: `string` → stringify rule only if the value is
     already a string, else if Value is non-string primitive → `E_ARGS` unless it
     is a string (strictness: only strings pass; numbers are NOT auto-stringified —
     users quote when needed... **Decision**: accept number/boolean and convert via
     JSON stringification for `string`-typed properties; note the leniency in a
     code comment referencing this issue); `number`/`integer` → if Value is number
     use it (integer check for `integer`), if string parse with `Number()` strict
     (trim, reject NaN/empty) else `E_ARGS`; `array`/`object` → if Value is
     list/record use as-is, if string parse via `fromJson` else `E_ARGS`; no
     `type` in property or property unknown to schema-less tools → pass string
     values as strings, typed Values as-is.
   - Flag not in schema (when schema has properties) and not reserved → `E_ARGS`
     listing known property names (sorted).
4. Duplicates: same flag twice, or flag key also present in `--json` object →
   `E_ARGS` "duplicate argument: x". (`--json` is the base; explicit flags fill
   only keys absent from it — but per DESIGN.md §9.6 the same key in both is an
   error, not an override.)
5. Required check: schema `required` properties missing from the final object →
   `E_ARGS` "missing required: a, b".
6. Output: `{ args: Record<string, Value>, raw: boolean }`.

## Acceptance Criteria

- [ ] Unit tests: every rule above including — boolean non-consumption
      (`--flag next-word` leaves the word to trigger the positional error);
      inline `=false`; number coercion from string and from typed Value; integer
      rejection of 1.5; array/object JSON parse success+failure (error includes
      excerpt); unknown flag error lists properties; duplicate flag; key collision
      with `--json`; `--json` non-object rejection; required-missing message;
      schema-less tool passes arbitrary flags as strings; `--raw-result` extraction.
- [ ] Table-driven test structure (one table entry per rule) for reviewability.

## Validation

`npm test -- args` green.

## Dependencies

03, 04.

## Non-goals

Client-side full JSON Schema validation (server validates; only the checks above),
`$in` resolution (19), builtin flag parsing (builtins declare their own simple
flags; 22–25).

## Design References

DESIGN.md §9.6.
