# ADR-002: Hybrid structured pipeline semantics

- Status: Accepted (human decision, 2026-07-06)
- Deciders: repository owner

## Context

The defining question for a "Unix-pipeline-inspired shell for composing MCP tools"
is what flows through `|`. MCP tool results are structured (JSON `structuredContent`,
typed content blocks), while external Unix commands speak bytes.

## Decision

Pipes carry **structured values** (JSON-compatible: null, boolean, number, string,
list, record) between mcpsh-native stages (MCP tools and builtins). At the boundary
to an **external command**, values are serialized deterministically to bytes on the
child's stdin, and the child's stdout is captured as a string value. Explicit
conversion builtins (`from json`, `from jsonl`, `lines`, `to json`, ...) cross
between representations; no implicit parsing of external output.

Serialization rules, `$in` binding, and limits are specified in DESIGN.md §7.

## Consequences

- Positive: MCP results are manipulated natively (`get`, `where`, `select`) without
  jq round-trips; external tools still compose (`... | jq`, `... | grep`).
- Negative: requires a lexer/parser/evaluator and a value model — the largest single
  cost in v1. Mitigated by a deliberately tiny grammar (no control flow, ADR-003).
- Values are fully materialized between stages in v1 (no byte-streaming through
  mcpsh stages); `tail -f`-style streaming composition does not work through
  structured stages. Documented limitation; fd-level pass-through between adjacent
  external commands is a v2 optimization.

## Alternatives considered

- **Pure text pipes (POSIX)**: minimal implementation but the product collapses into
  existing `mcpc`/`mcpt` territory; every structured operation needs external jq.
- **Structured-only closed world**: no external command execution; smallest attack
  surface but abandons the Unix-composition concept.
