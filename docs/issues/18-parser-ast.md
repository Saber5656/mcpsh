# Title

Parser and AST: programs, pipelines, commands, elements

## Summary

Implement `src/lang/ast.ts` and `src/lang/parser.ts`: turn the token stream into
the AST defined in DESIGN.md §6.2, including adjacency concatenation into
word-exprs, `^` placement validation, and faithful `raw` preservation.

## Context

Second half of the language front end. The parser stays consumer-agnostic: it does
not decide flag-value binding (§6.2) and does not evaluate expansions.

## Scope

- `src/lang/ast.ts`, `src/lang/parser.ts`, unit tests.

## Detailed Requirements

1. AST node shapes exactly as DESIGN.md §6.2 (Program/Pipeline/Command/Element with
   WordExpr/Flag and Part = Lit/Var/In/Env), each carrying `pos` (start of node)
   and Elements carrying `raw` (exact source spelling reconstructed from token
   raws, used for external argv §7.4).
2. `parse(source: string): Program`:
   - Grammar per §6.2: leading/trailing/duplicate SEMIs tolerated; empty program →
     `Program{pipelines: []}`.
   - Pipeline: commands separated by PIPE; PIPE with missing command on either
     side → `E_PARSE` `unexpected '|'` / `missing command after '|'`.
   - CARET: allowed only as the first token of a command, immediately followed
     (adjacent or not) by at least one element → sets `Command.external = true`;
     CARET elsewhere → `E_PARSE` `unexpected '^'`.
   - Adjacency concatenation: word-like tokens (WORD, STRING_SQ, STRING_DQ, VAR,
     IN, ENV) whose positions touch merge into a single WordExpr; STRING_DQ
     contributes its parts inline; STRING_SQ and WORD contribute Lit parts.
   - FLAG tokens become Flag elements; inline `=` value (word-expr continuation
     from the lexer) becomes `inline` (with its own adjacency handling).
   - A FLAG appearing after a `--` WORD element in the same command is converted
     to a WordExpr with a single Lit part (flag-terminator rule §6.1); the `--`
     itself is kept as a WordExpr element (consumers/externals see it verbatim;
     builtins/tools skip a bare `--`).
3. Parser never inspects semantic meaning (no builtin/tool knowledge).
4. Error recovery: none in v1 (first error throws). REPL catches and displays.

## Acceptance Criteria

- [ ] Snapshot/table tests: ≥25 programs → full AST including positions and raws,
      covering: multi-pipeline programs with `;` and newlines; `^cmd` external;
      `--flag=value`, `--flag value` (two elements), boolean-style lone flags;
      `--` terminator behavior; adjacency merges (`a$x"b"`); value-source command
      (`$var | next`); comments; empty program; dotted names.
- [ ] Error tests: `unexpected '|'`, `missing command after '|'`,
      `unexpected '^'`, propagated lexer errors — with positions.
- [ ] Round-trip property: for every test program, concatenating element `raw`s
      with single spaces re-parses to a structurally equal AST (raw fidelity).

## Validation

`npm test -- parser` green.

## Dependencies

17, 04.

## Non-goals

Evaluation (19), resolution (20), any control-flow syntax (v2, ADR-003).

## Design References

DESIGN.md §6.1, §6.2, §7.4.
