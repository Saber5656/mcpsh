# Title

Tool registry: listing cache, namespacing, resolution rules

## Summary

Implement `src/mcp/tools.ts` (registry half): fetch and cache per-server tool
lists, resolve `server.tool` and unqualified tool names per DESIGN.md §8/§9.4, and
invalidate caches on `tools/list_changed`.

## Context

Command resolution (issue 20) and completion (28) both consult this registry. The
rules that prevent surprising behavior: unqualified names resolve only against
*connected* servers' caches, and never trigger connects.

## Scope

- Registry portion of `src/mcp/tools.ts`; integration tests with fixture.

## Detailed Requirements

1. `class ToolRegistry(manager)`:
   - `listTools(alias, opts?: {refresh?: boolean}): Promise<ToolInfo[]>` — ensures
     connected (lazy connect allowed here: explicit user action), fetches via SDK
     `listTools` (paginate until complete), caches per server.
     `ToolInfo = { server, name, title?, description?, inputSchema?, outputSchema?,
     annotations? }`.
   - `cachedTools(alias): ToolInfo[] | undefined` — cache read, never connects.
   - `resolveQualified(name): { alias, tool } | undefined` — split at the FIRST
     `.`; left side must equal a configured server alias; right side non-empty
     (may contain further dots). Does NOT check tool existence (invocation reports
     tool-not-found from the server or from cache when available).
   - `resolveUnqualified(name): Resolution` where `Resolution` is
     `{kind:'unique', alias, tool}` | `{kind:'ambiguous', candidates: string[]}` |
     `{kind:'none'}` — searches only caches of servers in `ready` state.
   - `allCachedQualifiedNames(): string[]` for completion and did-you-mean.
2. `notifications/tools/list_changed` handler registered per connection (hook via
   manager from issue 09): drop that server's cache (refetch happens lazily on
   next need). Log at `debug`.
3. Cache also drops on disconnect/failed/closed transitions.
4. Alias/builtin collision is prevented at config load (issue 07); registry may
   assert it defensively.

## Acceptance Criteria

- [ ] Integration tests: `listTools` caches (second call does not re-request —
      assert via fixture request counting or SDK-level spy); `--refresh` refetches;
      `mutate_tools` + `list_changed` notification drops cache and next list shows
      `extra.tool`; `resolveQualified("github.some.tool")` splits at first dot;
      unqualified unique/ambiguous/none cases across two configured fixture
      instances (same fixture registered under two aliases → every shared tool
      name is ambiguous); unqualified resolution returns `none` when servers are
      configured but not connected (no connect side effect — assert states
      unchanged).
- [ ] Pagination: fixture exposes ≥3 tools; force page size via SDK options if
      available, else document single-page coverage and add a unit test for the
      pagination loop with a mocked client.

## Validation

`npm test -- tools-registry` green.

## Dependencies

09 (manager, hooks), 08.

## Non-goals

Invocation/unwrapping (12), argument mapping (13), command resolution order
(20 — it consumes this registry), completion UI (28).

## Design References

DESIGN.md §8, §9.4, §11.7.
