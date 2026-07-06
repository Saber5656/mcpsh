# Title

Resources support: listing and reading

## Summary

Implement `src/mcp/resources.ts`: list resources across servers and read a
resource by URI with server disambiguation, mapping contents to Values per
DESIGN.md §9.8.

## Context

Resources complete the "compose MCP" story beyond tools (goal G6). The builtins
(`resources`, `resource-read`) are thin wrappers added in issue 23's registry — the
protocol logic lives here.

## Scope

- `src/mcp/resources.ts`, integration tests with fixture.

## Detailed Requirements

1. `listResources(registry, alias?): Promise<Value>`:
   - With alias: ensure connected (explicit user action → lazy connect allowed),
     SDK `listResources` with pagination; map to records
     `{server, uri, name?, title?, mimeType?}` (omit absent fields).
   - Without alias: union over servers currently in `ready` state only (no
     connects); empty list + `info` log when none connected.
   - Cache per server, invalidated on `notifications/resources/list_changed` (add
     the handler; fixture support exists via SDK auto-capability) and on
     disconnect. Cache consulted by `resource-read` disambiguation and completion.
2. `readResource(manager, uri, opts: {server?: string}): Promise<Value>`:
   - Server selection: `opts.server` if given; else servers whose cached resource
     list contains `uri` — exactly one → use it; zero or multiple → `E_ARGS`
     with hint (`specify --server; candidates: a, b` / `no connected server lists
     this URI`).
   - SDK `readResource`; per-server `callMs` timeout and size cap apply (reuse
     patterns from issue 12).
   - Contents mapping: single text content → string; single blob content → blob
     record (§5.1) `{type:'blob', mimeType, data, bytes}`; multiple contents →
     list of the above mappings.
3. URI validation: must parse as a URI (`new URL` — accept custom schemes);
   invalid → `E_ARGS`.

## Acceptance Criteria

- [ ] Integration tests: list with alias returns both fixture resources with
      correct fields; union list with two aliases configured, one connected,
      returns only the connected one's resources (and does not connect the other —
      assert state); read `memo://greeting` → exact string; read `memo://blob` →
      blob record with matching base64 and bytes; ambiguous URI with two aliases
      of the same fixture → `E_ARGS` listing both; `--server` override resolves
      it; unknown URI → `E_ARGS`; invalid URI → `E_ARGS`; oversized read
      (add a `memo://big` resource to the fixture in this issue, parameterless
      fixed 1 MiB, and test with a lowered limit) → `E_LIMIT`.

## Validation

`npm test -- resources` green.

## Dependencies

03, 04, 09, 11 (registry/cache plumbing), 08.

## Non-goals

Resource templates (list/read of templated URIs beyond passing a full URI
through), subscriptions (`resources/subscribe`, v2), builtin registration (23).

## Design References

DESIGN.md §9.8, §5.1, §5.3.
