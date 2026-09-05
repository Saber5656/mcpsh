# Title

Prompts support: listing and getting

## Summary

Implement `src/mcp/prompts.ts`: list prompts and fetch a prompt's messages as
structured Values per DESIGN.md §9.8.

## Context

Prompts round out MCP feature coverage (goal G6). Even without an LLM (ADR-004),
rendered prompt messages are useful data (inspection, piping into files or other
tools).

## Scope

- `src/mcp/prompts.ts`, integration tests with fixture.

## Detailed Requirements

1. `listPrompts(registry, alias?): Promise<Value>` — same alias/union/caching
   semantics as `listResources` (issue 14), records:
   `{server, name, title?, description?, arguments: [{name, description?,
   required?}...]}` (arguments always a list, possibly empty). Cache invalidated on
   `notifications/prompts/list_changed` and disconnect.
2. `getPrompt(manager, qualifiedName, args: Record<string,string>): Promise<Value>`:
   - `qualifiedName` = `server.prompt` (first-dot split, same rule as tools §8);
     unqualified is NOT supported (prompts are namespaced explicitly; keep v1
     simple) → `E_ARGS` with hint when no dot present.
   - All argument values are strings (MCP prompt arguments are string-typed);
     `--arg key=value` parsing happens in the builtin (issue 23) — this module
     receives the record.
   - Result mapping: `{description?, messages: [{role, content: <BlockRecord>}]}`
     reusing the block mapping from issue 12 (export it there).
   - Missing required argument → server error mapped to `E_PROTOCOL` as-is
     (server's message), plus client-side pre-check against the cached prompt
     definition when available → `E_ARGS` "missing required argument: name".
3. Timeout and size cap as in issue 12.

## Acceptance Criteria

- [ ] Integration tests: list returns `greet` with its argument metadata; get
      `fixture.greet --arg name=World` (through the module API) → record with one
      user message whose text block equals `Hello, World!`; missing required arg →
      `E_ARGS` (cached) and `E_PROTOCOL` (cache-less path — clear cache first);
      unqualified name → `E_ARGS`; unknown server alias → `E_ARGS`.

## Validation

`npm test -- prompts` green.

## Dependencies

03, 04, 09, 11, 12 (block mapping export), 08.

## Non-goals

Prompt argument completion (`completion/complete`, v2), builtin registration (23),
any LLM execution of prompt messages (ADR-004).

## Design References

DESIGN.md §9.8; ADR-004.
