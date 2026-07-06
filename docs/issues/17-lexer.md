# Title

Lexer: tokens, quoting, expansions, comments

## Summary

Implement `src/lang/lexer.ts` producing the token stream defined in DESIGN.md §6.1
with precise position tracking and error reporting.

## Context

First half of the language front end. The token definitions (bare-word charset,
`--` flags only, `$` expansions, adjacency concatenation) encode several security
and UX decisions — implement them exactly.

## Scope

- `src/lang/lexer.ts`, unit tests.

## Detailed Requirements

1. Token kinds: `WORD`, `STRING_SQ`, `STRING_DQ` (with parts), `VAR`, `IN`, `ENV`,
   `FLAG`, `PIPE`, `SEMI`, `CARET`, `EOF`. Newline lexes as `SEMI` (§6.1). Every
   token carries `{start: {line, col}, end: {line, col}, raw: string}` (1-based).
2. WORD: maximal run of chars not in `{whitespace, |, ;, #, ^, $, ", '}` and not
   C0/C1 controls. `^` is CARET only at command-start position — the lexer does
   not know command positions, so: lex `^` as CARET token always; the parser
   validates placement (§6.2). Same for FLAG: a WORD starting with `--` followed by
   `[A-Za-z0-9]` lexes as FLAG with name `[A-Za-z0-9][A-Za-z0-9_-]*`; optional
   inline `=` splits: after `=` lex a word-expr continuation (parts until
   delimiter). A bare `--` lexes as WORD `--`.
3. STRING_SQ: `'...'`; content literal; no escapes (a `'` cannot appear inside);
   unterminated → `E_PARSE` at the opening quote.
4. STRING_DQ: `"..."` with escapes `\\ \" \$ \n \t` (others → `E_PARSE` "unknown
   escape"); interpolation parts: `$name`, `$in`, `$env.NAME` (NAME:
   `[A-Za-z_][A-Za-z0-9_]*`); `$` not followed by a valid form → `E_PARSE`
   (require `\$` for a literal dollar). Token carries
   `parts: (Lit|Var|In|Env)[]`.
5. Expansions outside strings: `$name` → VAR, `$in` → IN, `$env.NAME` → ENV;
   `$` alone or `$1` → `E_PARSE` with hint.
6. Adjacency: the lexer emits tokens; the parser concatenates adjacent
   word-like tokens (WORD/STRING_*/VAR/IN/ENV with no whitespace between) into one
   word-expr — to support this, tokens must record exact end/start positions so
   adjacency is detectable. (Concatenation itself is issue 18.)
7. COMMENT: `#` at a position where a new token would start → skip to end of line
   (a `#` inside a WORD run, e.g. `a#b`, stays in the WORD).
8. Errors: `ParseError` with line/col; message templates exactly:
   `unterminated string`, `unknown escape \\x`, `invalid expansion`,
   `invalid flag name`.

## Acceptance Criteria

- [ ] Table-driven tests: token streams (kinds + raws + positions) for at least 30
      inputs covering every token kind, adjacency cases (`pre$x"y $z"'lit'`),
      flags with/without inline values, `--` terminator word, `-x` as WORD,
      newline-as-SEMI, comments (start-of-token vs mid-word `#`), `$env.HOME`,
      `weird.name` dotted words, empty input, whitespace-only input.
- [ ] Error tests: each error template with exact line/col assertions (including
      multi-line input positions).
- [ ] Fuzz-ish test: lexing any printable-ASCII string either succeeds or throws
      `ParseError` (never a non-McpshError).

## Validation

`npm test -- lexer` green.

## Dependencies

01, 04.

## Non-goals

Parsing/AST (18), expansion evaluation (19), globbing (v2), multi-line
continuations (v2).

## Design References

DESIGN.md §6.1, §6.3 (token-level parts only).
