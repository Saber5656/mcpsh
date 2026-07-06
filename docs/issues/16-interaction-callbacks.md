# Title

Server-initiated interactions: elicitation, progress, logging, sampling refusal

## Summary

Implement `src/mcp/interactions.ts`: the elicitation request handler (interactive
prompt vs auto-decline), progress and logging notification routing, and verify
sampling requests fail because the capability is not declared (ADR-004), per
DESIGN.md §9.9.

## Context

These are the only paths where a server initiates action toward the user —
security-sensitive (threats T10, T1) and mode-dependent (REPL vs one-shot).

## Scope

- `src/mcp/interactions.ts`; capability registration via the issue-09 hook;
  integration tests with fixture.

## Detailed Requirements

1. `InteractionHost` interface (implemented by REPL in issue 27; a null host for
   one-shot mode):
   `{ interactive: boolean, promptElicitation(server: string, message: string,
   schema: FlatSchema): Promise<{action:'accept', content: Record<string,Value>} |
   {action:'decline'} | {action:'cancel'}>, onProgress(server, update): void }`.
2. Elicitation handler (registered as client capability + request handler):
   - Non-interactive host → respond `{action:'decline'}` immediately; `debug` log.
   - Interactive: validate `requestedSchema` is a flat object schema whose
     properties are `string | number | integer | boolean | enum` — anything else →
     auto-decline + one-line notice via logger (`warn`).
   - Sanitize (issue 06) the server-provided message and all property
     names/descriptions before display; prefix display with the server alias
     (mitigates T10; actual rendering in REPL issue 27 — this module passes
     pre-sanitized strings).
   - User cancel (Ctrl-C during prompt) → `{action:'cancel'}`.
   - Response content values validated against the flat schema (type coercion:
     `y/n` → boolean for boolean props; numeric parse; enum membership) before
     sending; invalid input re-prompts up to 3 times then declines.
3. Progress: subscribe per-call (issue 12's `onProgress`) and route to
   `InteractionHost.onProgress` — carrying `{server, progress, total?, message?}`
   with message sanitized here.
4. Logging notifications (`notifications/message`): map server levels
   `debug|info|notice|warning|error|critical|alert|emergency` →
   mcpsh `debug|info|info|warn|error|error|error|error`; prefix `[<alias>]`;
   sanitize; forward to logger (issue 05).
5. Sampling: do NOT register the capability or any handler. Add an integration
   test that a fixture-initiated `sampling/createMessage` (add a fixture tool
   `try_sampling {}` that attempts it and reports the error it received — extend
   the fixture in this issue) results in a method-not-found error at the server
   side and no crash in mcpsh.
6. Roots: assert the client declares no roots capability (initialize result
   inspection in a test).

## Acceptance Criteria

- [ ] Integration tests: `ask_user` with a scripted interactive host → accept path
      round-trips the answer; decline path; cancel path; non-interactive host
      auto-declines; nested schema auto-declines with warning; control chars in
      elicitation message arrive sanitized at the host; `log_spam` at each server
      level lands at the mapped mcpsh level (logger spy); `try_sampling` reports
      method-not-found; `slow` progress reaches the host with sanitized message.
- [ ] Re-prompt-then-decline after 3 invalid inputs (scripted host returning bad
      values).

## Validation

`npm test -- interactions` green.

## Dependencies

04, 05, 06, 08, 09, 12.

## Non-goals

REPL rendering of prompts/spinners (27, 29), sampling mediation (v2, ADR-004),
elicitation for nested/complex schemas (v2).

## Design References

DESIGN.md §9.9, §12.3 (T1, T10); ADR-004.
