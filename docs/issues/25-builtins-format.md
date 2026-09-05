# Title

Format builtins: from, to, lines, echo, save, open

## Summary

Implement `src/builtins/format.ts`: representation conversions between strings and
structured values, plus safe file read/write, per the DESIGN.md §10 table.

## Context

These builtins cross the text/structure boundary (ADR-002) and touch the
filesystem (`save`/`open` — threat T13 safe-overwrite rule and size caps apply).

## Scope

- `src/builtins/format.ts`, unit tests (temp dirs for file ops).

## Detailed Requirements

Exactly per §10:

1. `from <format>` (requires string input): formats `json` (→ `fromJson`),
   `jsonl` (split lines, drop trailing empty line, parse each — error names the
   line number), `lines` (split on `\n`, drop one trailing empty element).
   Unknown format → `E_ARGS` listing formats. Non-string input → `E_TYPE`.
2. `to <format>` (requires): `json [--pretty]` (compact default; pretty = 2-space),
   `jsonl` (list input required; each element compact + `\n`, returned as one
   string), `text` (string → as-is; other → compact JSON). Unknown → `E_ARGS`.
3. `lines` (requires string) — exact alias of `from lines`.
4. `echo [words...]` (accepts): 0 args → passthrough input (null stays null);
   1 arg → that word's Value (bare `$var` passes typed value per §6.3); n args →
   list of Values.
5. `save <path> [--raw] [--force]` (requires): default mode writes
   `toJson(value, {pretty: true})` + `\n`; `--raw` requires string input
   (else `E_TYPE`) and writes it verbatim (no sanitization — §12.6 exempts
   `save --raw`); existing path without `--force` → `E_ARGS`
   `file exists: <path> (use --force)`; parent dir must exist (no mkdir -p);
   write is atomic-ish: write to `<path>.tmp-<pid>` then rename; mode 0644;
   returns null.
6. `open <path>` (ignores): reads file; enforce `limits.maxOutputBytes` BEFORE
   reading via `fs.stat` (→ `E_LIMIT`); decode strict UTF-8 — invalid bytes →
   `E_TYPE` `not valid UTF-8: <path>` (hint: binary files are unsupported in v1);
   returns string. ENOENT/EACCES → `E_ARGS` with path.
7. Paths: expand a leading `~/` to `os.homedir()` (both builtins); relative paths
   resolve against `process.cwd()`.

## Acceptance Criteria

- [ ] Unit tests: from json/jsonl (incl. line-numbered error)/lines trailing-empty
      rule; to json compact/pretty/jsonl/text; lines alias equivalence; echo
      0/1/n args incl. typed `$var` passthrough (via evaluator harness); save
      default JSON+newline bytes asserted, `--raw` verbatim (bytes with ANSI kept),
      exists/`--force`, tmp-rename (no partial file on injected write failure),
      missing parent dir error; open size-cap via stat, UTF-8 rejection (write
      invalid bytes first), `~/` expansion (HOME stubbed), ENOENT.
- [ ] Meta-test: inputPolicy per §10 for these six builtins.

## Validation

`npm test -- builtins-format` green.

## Dependencies

03, 19.

## Non-goals

CSV/TSV/YAML formats (v2), binary reads (v2), globbing (v2), sanitization of
saved bytes (§12.6 exemption is intentional).

## Design References

DESIGN.md §5.3, §10, §12.3 (T13), §12.6; ADR-002.
