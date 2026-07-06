# ADR-004: No LLM features in v1 (deterministic shell)

- Status: Accepted (human decision, 2026-07-06)
- Deciders: repository owner

## Context

MCP servers may request LLM completions from the client (`sampling/createMessage`).
Separately, AI-assisted UX (natural language → pipeline) is a plausible feature.
Both would pull API-key management, cost controls, and approval UX into scope and
significantly enlarge the attack surface of an otherwise deterministic tool.

## Decision

v1 is a **fully deterministic shell**:

- The client does **not** declare the `sampling` capability. Incoming
  `sampling/createMessage` requests fail at the protocol layer (JSON-RPC
  method-not-found). mcpsh documents this behavior so users understand why
  sampling-dependent servers degrade.
- No LLM API keys are read, stored, or transmitted by mcpsh.
- No natural-language command synthesis.

## Consequences

- Positive: no secret custody beyond user-configured server headers; reproducible
  behavior; simpler threat model (DESIGN.md §12); trivially testable.
- Negative: MCP servers that hard-require sampling are partially or fully unusable
  from mcpsh v1. Accepted; such servers are a minority and degrade with a clear error.
- v2 design item (documented, not planned): opt-in sampling mediation via a
  user-configured provider with per-request interactive approval.
