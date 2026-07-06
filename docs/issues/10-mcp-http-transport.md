# Title

MCP connection manager: Streamable HTTP transport

## Summary

Extend `src/mcp/manager.ts` with the Streamable HTTP transport: configured headers
with env-ref resolution, TLS enforcement rules, and the same lifecycle semantics as
stdio.

## Context

Remote MCP servers use Streamable HTTP (legacy HTTP+SSE is deprecated and
unsupported, research/mcp-protocol-notes.md). Security: HTTPS off-localhost
(threat T7), header secrets via env refs only (T4).

## Scope

- HTTP branch in `src/mcp/manager.ts` (or `src/mcp/http.ts` if cleaner);
  integration tests against the fixture HTTP entry (issue 08).

## Detailed Requirements

1. Transport: SDK `StreamableHTTPClientTransport` with `url` and
   `requestInit.headers` built from config `headers` through `resolveEnvRefs`
   (issue 07; registers secrets with the redactor).
2. TLS rule enforcement happens at config load (issue 07); this issue must add a
   defense-in-depth assertion at connect time (`E_CONNECT` if a non-localhost
   `http:` URL reaches the transport builder without `allowInsecureHttp`).
3. Same FSM semantics as stdio: `connectMs` timeout, single-shot attempts, `failed`
   state with stored error, `closeAll()` support. `getStderrTail` returns `[]` for
   HTTP servers.
4. Error mapping: DNS/TCP/TLS failures, HTTP 401/403 (message: "authentication
   failed (check headers for <alias>)"), 404, and protocol-level initialize
   failures all → `E_CONNECT` with cause; message never includes header values
   (redaction + construct messages from status codes only).
5. HTTP-specific info in `list()`: `transport: 'http'`, host shown in `servers`
   builtin output later (store `displayTarget = new URL(url).host`).

## Acceptance Criteria

- [ ] Integration tests (fixture over HTTP): connect/list/call `echo` round-trip;
      lifecycle states; connect timeout against a TCP server that accepts and
      stalls (net.createServer with no response); 401 mapping (wrap fixture with a
      auth-checking proxy or a stub HTTP server returning 401); header env-ref
      resolution (fixture http entry extended to optionally require a bearer
      token via env `FIXTURE_REQUIRE_TOKEN` — add that option in this issue);
      error messages contain no token value (assert on thrown message and logs).
- [ ] Defense-in-depth http:// assertion unit-tested by constructing a manager
      from a hand-built (schema-bypassing) config object.

## Validation

`npm test -- http` green on macOS/Linux CI (no external network use — everything
on 127.0.0.1).

## Dependencies

01, 04, 05, 07, 08, 09.

## Non-goals

OAuth 2.1 flows (v2), legacy SSE transport, retries/backoff, proxies.

## Design References

DESIGN.md §9.2, §9.3, §12.3 (T4, T7); research/mcp-protocol-notes.md.
