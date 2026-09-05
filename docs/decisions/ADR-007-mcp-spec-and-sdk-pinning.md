# ADR-007: Target MCP spec 2025-11-25 via @modelcontextprotocol/sdk v1.x

- Status: Accepted (design decision, 2026-07-06)
- Deciders: design agent (based on docs/research/mcp-protocol-notes.md)

## Context

As of 2026-07-06 the stable MCP revision is 2025-11-25. The 2026-07-28 revision
(stateless core, extensions, tasks, MCP Apps) is a release candidate shipping ~3
weeks after this decision, alongside TypeScript SDK v2 (currently beta). SDK v1.x
(current: 1.29.0) remains supported for at least 6 months after v2 ships.

## Decision

- Depend on `@modelcontextprotocol/sdk@^1.29` for all v1 issues. Do not use the v2
  beta anywhere.
- Treat the protocol revision as an SDK concern: mcpsh records the negotiated
  protocol version per connection (`servers` builtin shows it) but contains no
  revision-specific branching of its own.
- Re-verify SDK/spec status at the start of implementation Wave 1 (the research doc
  is dated); if SDK v2 has gone stable by then, **still ship v1 on SDK v1.x** —
  migration is its own post-v1 task with its own test pass.

## Consequences

- Positive: implementation agents build against a stable, documented API surface;
  no rework risk from v2-beta API churn during the v1 window.
- Negative: 2026-07-28 features (tasks, extensions) are unavailable in v1; a
  planned SDK-v2 migration task is required later (listed under known unknowns in
  ISSUE_PLAN.md).
