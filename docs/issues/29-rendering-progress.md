# Title

Rendering: pretty printer, table builtin, progress display

## Summary

Implement `src/render/render.ts`: the colored pretty renderer for REPL output, the
`table` builtin, duration display, and the progress spinner/status line driven by
progress notifications — all sanitization-integrated per DESIGN.md §11.5.

## Context

This issue makes structured output humane. Every data string crossing into the
terminal goes through `sanitize` (issue 06, threat T1). The renderer is consumed
via the narrow interface already stubbed by issues 26/27.

## Scope

- `src/render/render.ts`, `table` builtin registration, unit tests (string
  assertions with color disabled; targeted ANSI assertions with color forced).

## Detailed Requirements

1. `renderValue(value, opts: {color: boolean, width: number}): string`:
   - Top-level string → sanitized raw text.
   - Other values → 2-space-indented JSON; colors via `util.styleText` (keys cyan,
     strings green, numbers yellow, booleans magenta, null dim); all string
     content sanitized BEFORE styling (escape-proof ordering); blob records
     render `data` truncated to 32 base64 chars + `…(<bytes> bytes)`.
2. `table` builtin (`inputPolicy: 'requires'`, registered here):
   - Input list-of-records (else `E_TYPE`; empty list → dim `(empty)`).
   - Columns: union of keys in encounter order, max 8 (+ dim notice
     `(+N more columns)`); cells: strings raw-sanitized, other values compact
     JSON; cell truncation with `…` so the table fits `width` (min column width
     4); header row dim; no box-drawing characters (space-padded, one header
     separator line of `─`).
   - Returns the rendered string (renderer-independent value; REPL prints it
     as a top-level string; usable in one-shot too).
3. Progress display (`ProgressRenderer` implementing the loop's `onProgress`
   hook + a ticker):
   - Activates for calls running >500 ms on a TTY; single status line
     `⠋ <server>.<tool> … <elapsed>s [<pct>%] <message?>` rewritten in place
     (`\r` + clear-to-EOL), spinner frames braille set, 100 ms tick; percent from
     `progress/total` when total present; message sanitized and truncated to
     width; line fully cleared before the result/error prints.
   - Non-TTY: no output (progress already logged at debug by issue 16).
4. Duration line: calls >2 s append dim `(took X.Ys)` after the rendered value
   (loop consults timing recorded around evalProgram stages — expose a per-stage
   timing hook from the evaluator: add an optional `onStageComplete` callback to
   evalProgram opts; wire here).
5. Respect color enablement rules from issue 26 (`--no-color`, NO_COLOR, TTY).

## Acceptance Criteria

- [ ] Unit tests (color off): pretty JSON snapshot for a nested value; top-level
      string raw; blob truncation; table column union/order/8-cap/truncation/
      empty-list/width fitting (width 40 vs 120); `E_TYPE` for non-list-of-records.
- [ ] Sanitization: value containing `\x1b[31m` renders with `␛` not ESC (color
      on and off); table cell likewise.
- [ ] Progress: fake timer test — updates rewrite one line, clears on completion;
      no output when not TTY; message sanitized.
- [ ] Duration line appears for a stubbed 2.5 s stage, absent for 0.1 s.

## Validation

`npm test -- render` green; manual REPL check with fixture `slow`/`control_chars`
documented in the PR description.

## Dependencies

03, 06, 16 (progress plumbing), 19 (stage hook), 26, 27 (interfaces to satisfy).

## Non-goals

Themes/config for colors (v2), pager integration (v2), `table` sorting/styling
flags (v2), width-aware wrapping beyond truncation.

## Design References

DESIGN.md §10 (`table`), §11.5, §12.3 (T1), §12.6.
