# Title

Fixture MCP server for integration testing

## Summary

Build an in-repo MCP server (`test/fixture-server/`) exercising every protocol
feature mcpsh consumes: assorted tools (including adversarial ones), resources,
prompts, elicitation, progress, logging notifications — runnable over stdio and
Streamable HTTP.

## Context

All MCP-layer and e2e issues (09–16, 30, 31) test against this server. It uses the
same SDK (server side) pinned by ADR-007, so protocol behavior matches reality
without network dependencies.

## Scope

- `test/fixture-server/server.ts` (shared logic), `stdio.ts` (bin entry),
  `http.ts` (Streamable HTTP entry printing its ephemeral port), minimal unit test
  proving it starts and lists tools.

## Detailed Requirements

1. Tools (names and behavior are contract — tests depend on them):
   - `echo { text: string }` → single text block with `text`.
   - `add { a: number, b: number }` → `structuredContent: { sum }` (+ outputSchema).
   - `get_structured { kind: "record"|"list"|"nested" }` → structuredContent
     variants (fixed literals for snapshot tests).
   - `fail { message: string }` → `isError: true`, text = message.
   - `slow { ms: number }` → sleeps; emits `notifications/progress` every 100 ms
     when a progressToken is provided; returns `done`.
   - `big { bytes: number }` → text of exactly `bytes` ASCII `x`s (cap 64 MiB;
     larger request → isError).
   - `control_chars {}` → text containing `\x1b[31m`, OSC52 sequence, `\r`, BEL,
     backspace, C1 U+009B.
   - `env_dump {}` → structuredContent `{ env: {<name>: <value>...} }` of the
     server process env.
   - `ask_user { question: string }` → sends `elicitation/create` with
     `requestedSchema: { type: "object", properties: { answer: { type: "string" } },
     required: ["answer"] }`; returns the user's answer or `declined`.
   - `multi_content {}` → two text blocks + one 1×1 PNG image block.
   - `log_spam { level: string, message: string }` → emits a
     `notifications/message` at that level, returns `ok`.
   - `weird.name { }` → returns text `weird` (tests qualified parsing with dots
     and `call` quoting).
2. Resources: `memo://greeting` (text/plain "hello from fixture"),
   `memo://blob` (image/png, small fixed base64). Resource list returns both.
3. Prompts: `greet { name: string (required) }` → one user message
   `Hello, <name>!`.
4. Server declares tools/resources/prompts capabilities incl. `listChanged` where
   applicable; a special tool `mutate_tools {}` toggles registration of an extra
   tool `extra.tool` and emits `notifications/tools/list_changed` (for issue 11
   cache-invalidation tests).
5. stdio entry: plain `node dist-test/.../stdio.js` runnable after build; also
   emits one line to **stderr** at startup (`fixture-server started`) for
   stderr-capture tests.
6. http entry: starts Streamable HTTP server on port 0, prints
   `PORT=<port>` to stdout, serves until SIGTERM.
7. Keep the fixture in devDependencies-only territory (may use the SDK server API;
   no other new deps). TypeScript, built by the normal `tsc` build into a
   test-only output dir (adjust tsconfig with a secondary project file
   `tsconfig.test.json` if needed — document the chosen mechanism in the README
   section of the fixture directory).

## Acceptance Criteria

- [ ] A vitest integration test connects over stdio via the SDK client, lists 13
      tools / 2 resources / 1 prompt, calls `echo`, and closes cleanly.
- [ ] Same over Streamable HTTP using the printed port.
- [ ] `control_chars`, `env_dump`, `ask_user`, `mutate_tools` behave exactly as
      specified (asserted in tests here or deferred to consuming issues with a
      smoke assertion here).

## Validation

`npm test -- fixture` green on macOS and Linux CI.

## Dependencies

01.

## Non-goals

Testing mcpsh's own client code (09+), OAuth/auth on the HTTP fixture, performance
realism.

## Design References

DESIGN.md §14; research/mcp-protocol-notes.md; ADR-007.
