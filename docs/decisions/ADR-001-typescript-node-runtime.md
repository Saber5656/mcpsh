# ADR-001: TypeScript on Node.js as implementation runtime

- Status: Accepted (human decision, 2026-07-06)
- Deciders: repository owner

## Context

mcpsh is a shell/CLI whose core dependency is an MCP client implementation. The
runtime choice determines MCP SDK maturity, distribution channel, startup latency,
and how mechanically implementation agents can execute the issue plan.

## Decision

Implement mcpsh in **TypeScript running on Node.js**.

- `engines.node: ">=22.12.0"`; CI on Node 22.x and 24.x (see research/mcp-protocol-notes.md).
- Strict TypeScript, ESM (`"type": "module"`, moduleResolution NodeNext).
- MCP support via the official `@modelcontextprotocol/sdk` v1.x (ADR-007).
- Runtime dependency policy: **only** `@modelcontextprotocol/sdk` and `zod`.
  Colors via `node:util.styleText`, CLI parsing and REPL via `node:` builtins.

## Consequences

- Positive: most mature MCP SDK (client-side sampling/elicitation/Streamable HTTP
  all first-party); largest example corpus for implementation agents; npm distribution.
- Negative: requires a Node runtime (no single-binary distribution); ~50–100 ms
  startup overhead per one-shot invocation. Accepted for v1; single-binary packaging
  (e.g. bun compile / SEA) is a v2 idea.
- Neutral: npm package name conflict must be resolved before publishing (ADR-008).

## Alternatives considered

- **Go**: single binary, fast startup; official SDK exists but client-side features
  and examples are thinner; JSON manipulation code is more verbose. Rejected for v1.
- **Rust**: best performance; highest implementation and review cost; poorest fit for
  granular delegation to implementation agents. Rejected.
- **Python**: mature SDK but slow REPL startup and weaker distribution story for a
  shell. Rejected.
