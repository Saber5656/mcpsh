# Title

Data builtins: get, select, where, first, last, length, flatten, sort-by, reverse

## Summary

Implement `src/builtins/data.ts`: the structured-data operators from the
DESIGN.md §10 table — the jq-replacement core of the product.

## Context

These operators are why structured pipes beat text pipes (ADR-002). Semantics are
fixed by §10 (including list-mapping `get`, MISSING handling differences between
`get` and `where`/`select`).

## Scope

- `src/builtins/data.ts`, unit tests (pure; mock env).

## Detailed Requirements

All `inputPolicy: 'requires'`. Exactly per §10:

1. `get <path>`: path via `pathGet` (issue 03). Input record/scalar → direct
   access; input **list** with a path whose first segment is not an integer → map
   over elements (each element's MISSING → `E_TYPE` naming path and index).
   Direct MISSING → `E_TYPE` `no value at path 'x.y'` (hint: `where`/`select`
   skip missing). Path argument must be a single word; missing → `E_ARGS`.
2. `select <p1> [p2...]`: input record → record with the named paths' values
   (last path segment becomes the key; duplicate result keys → `E_ARGS`);
   input list-of-records → map; MISSING → key omitted; input scalar/mixed list →
   `E_TYPE`. ≥1 path required.
3. `where <path> <op> <literal>`: input list only. Ops:
   `== != > >= < <= contains startswith`. Literal parsing: `true`/`false`/`null`
   → those; numeric-looking → number; else string (quoted words arrive as strings
   already; document that `where x == "5"` cannot force string comparison in v1 —
   the literal rule wins; known limitation note in help text). Comparison rules:
   `==`/`!=` deep-equal; ordering ops require both sides same primitive type
   (number/string) else element excluded; `contains`: string haystack (substring),
   list haystack (deep-equal membership), record haystack (key presence);
   `startswith`: strings only. MISSING path → element excluded.
4. `first [n]` / `last [n]`: list input; no arg → single element (empty list →
   `E_TYPE` "empty list"); with n (positive int; else `E_ARGS`) → list of ≤n.
5. `length`: list → element count; record → key count; string → code-point count;
   other → `E_TYPE`.
6. `flatten`: list → one level flattened (non-list elements kept as-is).
7. `sort-by <path> [--desc]`: list; keys via `pathGet` per element; MISSING →
   element sorts last (stable); mixed key types: numbers before strings before
   booleans before null before others (document fixed order); stable sort.
8. `reverse`: list → reversed copy.

## Acceptance Criteria

- [ ] Table-driven unit tests per builtin covering every rule above, including:
      get list-mapping and its indexed error; select duplicate-key error and
      MISSING omission; where each op × (string, number) plus contains on
      string/list/record, literal parsing edge (`where x == true`), excluded
      mixed-type ordering; first/last empty and n variants; sort-by stability
      (equal keys keep order), --desc, MISSING-last; flatten one-level-only proof.
- [ ] Every error is `E_TYPE`/`E_ARGS` with messages naming the builtin.

## Validation

`npm test -- builtins-data` green.

## Dependencies

03, 19.

## Non-goals

Arbitrary expressions/closures in `where` (v2), `group-by`/`uniq` (v2), parallel
map (v2).

## Design References

DESIGN.md §5.2, §10; ADR-002.
