# Title

Terminal output sanitization module

## Summary

Implement `src/render/sanitize.ts`: neutralize control characters and ANSI escape
sequences in data-derived strings before they reach the user's terminal
(threat T1, DESIGN.md §12.6).

## Context

MCP servers are semi-trusted; their output is untrusted data. Raw ANSI/OSC
sequences in tool results can rewrite the screen, spoof prompts, or (OSC 52) write
the clipboard. Every rendering path routes through this module.

## Scope

- `src/render/sanitize.ts`, unit tests.

## Detailed Requirements

1. `sanitize(text: string): string`:
   - Remove C0 control chars (U+0000–U+001F) **except** `\n` (U+000A) and `\t`
     (U+0009). `\r` is removed (prevents line-rewrite spoofing).
   - Remove C1 control chars (U+0080–U+009F).
   - Replace every ESC (U+001B) with `␛` (U+241B, SYMBOL FOR ESCAPE) — this
     neutralizes CSI/OSC/DCS sequences while keeping evidence visible.
   - Remove U+007F (DEL).
   - Preserve all other characters including emoji and combining marks; must not
     split surrogate pairs (operate on code points, not UTF-16 units).
2. `sanitizeMultiline(text: string): string` — alias documented for readability
   (same behavior; exists so call sites are greppable).
3. Performance: single pass; no regex backtracking risk (use a character-class
   regex or manual scan); handles 10 MB strings in <100 ms on CI hardware.
4. Module is leaf-level: no imports from other `src/` modules.

## Acceptance Criteria

- [ ] Tests: strips `\r`, BEL, backspace, vertical tab; keeps `\n`/`\t`; replaces
      ESC in `\x1b[31mred\x1b[0m` and OSC52 `\x1b]52;c;...\x07` (BEL also
      stripped); C1 range removed; DEL removed; emoji/surrogate pairs intact;
      idempotence (`sanitize(sanitize(x)) === sanitize(x)`); empty string.
- [ ] Property-style test: for strings of random code points, output contains no
      code point in the forbidden sets.
- [ ] 10 MB benchmark test under 100 ms (marked non-flaky: generous bound, skip on
      unusual environments only with justification).

## Validation

`npm test -- sanitize`; issue 31 validates end-to-end via the fixture server's
`control_chars` tool.

## Dependencies

01.

## Non-goals

Deciding *where* sanitize applies (render/error layers own that: issues 04, 29);
markdown/HTML escaping; width/truncation logic (29).

## Design References

DESIGN.md §12.3 (T1), §12.6.
