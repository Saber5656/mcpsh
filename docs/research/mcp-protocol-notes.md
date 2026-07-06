# Research: MCP Protocol and SDK Status

Status: Completed 2026-07-06. Informs DESIGN.md §9 and ADR-007 (spec/SDK pinning).
Facts below are time-sensitive; re-verify at implementation start of Wave 1.

## Specification status (as of 2026-07-06)

| Item | Status |
|---|---|
| Current stable spec revision | **2025-11-25** |
| Next revision | **2026-07-28**, currently a release candidate; final planned for 2026-07-28 |
| Headline changes in 2026-07-28 RC | Stateless protocol core, Extensions framework, Tasks, MCP Apps (sandboxed HTML UIs), authorization hardening, formal feature-lifecycle/deprecation policy (≥12 months deprecated before removal) |
| Legacy HTTP+SSE transport | Deprecated in favor of Streamable HTTP (since 2025-03-26 revision) |

Implication for mcpsh v1: target the **2025-11-25** revision through the stable SDK.
The SDK performs protocol-version negotiation during `initialize`, so servers speaking
older revisions continue to work. The 2026-07-28 changes (tasks, extensions, stateless
core) are consumed later via an SDK upgrade — tracked as a known unknown, not a v1
requirement.

## TypeScript SDK status (as of 2026-07-06)

| Item | Status |
|---|---|
| Package | `@modelcontextprotocol/sdk` |
| Current stable | **v1.29.0** |
| v2 | Beta, targets the 2026-07-28 spec; stable planned to land alongside the spec (2026-07-28) |
| v1.x support window | Bug fixes and security updates for **at least 6 months after v2 ships** |

Decision (ADR-007): pin `@modelcontextprotocol/sdk@^1.29` for all of v1. Do not build
on the v2 beta: API may still change, and v1.x remains supported well past our v1
window. Plan an explicit v2-SDK migration task after v1 ships.

## Client features mcpsh must handle (2025-11-25 revision)

| Protocol feature | Direction | mcpsh v1 stance |
|---|---|---|
| `initialize` / version negotiation | client→server | SDK-managed; record negotiated version per connection |
| `tools/list`, `tools/call` | client→server | Core feature |
| `notifications/tools/list_changed` | server→client | Invalidate cached tool list |
| `resources/list`, `resources/read` | client→server | Supported via builtins |
| `prompts/list`, `prompts/get` | client→server | Supported via builtins |
| `sampling/createMessage` | server→client | **Capability not declared**; requests fail with JSON-RPC method-not-found (ADR-004) |
| `elicitation/create` | server→client | Declared in interactive REPL only; auto-decline in non-interactive mode |
| `roots/list` | server→client | **Capability not declared** in v1 |
| `notifications/progress` | server→client | Rendered in REPL (spinner/progress line) |
| `notifications/message` (logging) | server→client | Mapped to mcpsh logger levels; visible with `--verbose` |
| `ping` | both | SDK auto-responds |
| `notifications/cancelled` | client→server | Sent on Ctrl-C via AbortSignal |
| `completion/complete` (argument autocompletion) | client→server | Deferred to v2 |
| Tool output: `structuredContent` + `outputSchema` | server→client | Preferred unwrap source (DESIGN.md §9.7) |
| Content block types: text, image, audio, resource_link, embedded resource | server→client | Mapped to records (DESIGN.md §9.7) |

## Transports for v1

| Transport | v1 support | Notes |
|---|---|---|
| stdio | Yes | Subprocess with env allowlist (DESIGN.md §12.4) |
| Streamable HTTP | Yes | HTTPS required for non-localhost; static headers from env refs; OAuth deferred |
| HTTP+SSE (legacy) | No | Deprecated upstream; document as unsupported |

## Node.js runtime baseline (verified 2026-07-06 via endoflife.date)

| Node line | LTS status | EOL |
|---|---|---|
| 26 | LTS starts 2026-10-28 | 2029-04-30 |
| 24 | Active LTS (since 2025-10-28) | 2028-04-30 |
| 22 | Maintenance LTS | 2027-04-30 |

Decision: `engines.node: ">=22.12.0"`, CI matrix on 22.x and 24.x. `util.styleText`
(stable in these lines) replaces any color dependency.

## Sources

- <https://modelcontextprotocol.io/specification/2025-11-25>
- <https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/>
- <https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/>
- <https://github.com/modelcontextprotocol/typescript-sdk> / <https://www.npmjs.com/package/@modelcontextprotocol/sdk>
- <https://ts.sdk.modelcontextprotocol.io/> (v1) / <https://ts.sdk.modelcontextprotocol.io/v2/> (v2 beta)
- <https://endoflife.date/nodejs>
