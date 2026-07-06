# Title

REPL loop: readline, history, cancellation, rc file, elicitation prompts

## Summary

Implement `src/repl/loop.ts`: the interactive session per DESIGN.md §11.6 —
prompt, persistent history with hygiene rules, Ctrl-C/Ctrl-D semantics, rc-file
execution, and the interactive `InteractionHost` (elicitation prompts).

## Context

The REPL is the flagship surface (goal G4). Design details that matter: history
file permissions and skip rules (threat T5), cancellation that always returns the
prompt (T9), lazy startup (§11.6: no connects at start).

## Scope

- `src/repl/loop.ts`, `src/repl/history.ts`; unit tests via injected
  readline-compatible interface; one guarded pty smoke test lands in issue 30.

## Detailed Requirements

1. Loop: `node:readline/promises` over stdin/stdout with `terminal: true`;
   prompt `mcpsh> `; single-line input (§11.6). Empty/whitespace line → new
   prompt. Each line: alias-expand → parse → eval → render via the renderer
   interface (pretty when 29 lands; JSON fallback until then) → prompt. Errors:
   `formatError` to stderr, session continues (all codes including `E_CONFIG`
   from late env-ref resolution).
2. Banner: `mcpsh <version> — N servers configured (a, b, ...). Type 'help' to
   start.` (0 servers → hint to create the config file with its path).
3. History (`src/repl/history.ts`):
   - File: `history.file` config override else `$XDG_STATE_HOME/mcpsh/history`
     (default `~/.local/state/mcpsh/history`); create parent dirs `0700`, file
     `0600` (chmod existing files to 0600 with a warn if wrong); load at start
     (tolerate missing/unreadable → warn once, in-memory only).
   - Append on accepted lines except: leading-space lines (T5) and duplicates of
     the immediately previous entry. Trim to `history.maxEntries` on save.
     Persist on each accepted line (append + fsync not required; document).
   - Expose provider for the `history` builtin (issue 22) and readline's
     up-arrow via injecting loaded entries.
4. Cancellation: while a pipeline runs, SIGINT/Ctrl-C aborts its
   AbortController; loop prints `^C` and re-prompts; at an empty prompt Ctrl-C
   prints hint `(Ctrl-D to exit)`; Ctrl-D (EOF) → graceful exit (closeAll, exit 0).
   `exit` builtin → same path via `requestExit()`.
5. rc file: `$XDG_CONFIG_HOME/mcpsh/init.mcpsh` if it exists — before the first
   prompt, execute line-by-line as programs (skip blank/comment lines); per-line
   errors print (prefixed `rc:<lineno>:`) and execution continues; rc lines are
   not appended to history; rc output values are rendered normally.
6. Interactive `InteractionHost` (issue 16 interface): elicitation prompts per
   §9.9 — display `<alias> is asking: <message>` then one readline question per
   property (`name (type) [default]: `, enum shows options; boolean accepts
   y/n/true/false); empty input on required → re-ask (3× → decline); Ctrl-C →
   cancel. `onProgress` is a no-op until issue 29 wires the spinner (narrow
   interface).
7. No server connects at startup or during completion (§11.6, §11.7).

## Acceptance Criteria

- [ ] Unit tests with injected I/O: line→eval→render loop; error keeps session;
      banner variants (0 and 2 servers); history append/skip-space/dedup/trim/
      provider; history file created 0600 and parent 0700 (assert modes);
      Ctrl-C during a long `slow` call aborts and re-prompts (scripted signal);
      Ctrl-D exits cleanly with closeAll spy called; rc file execution order,
      error-continue, non-persistence in history; elicitation accept/decline/
      cancel/re-ask flows via scripted answers (drives fixture `ask_user`
      end-to-end with manager+interactions wired).
- [ ] `exit` builtin terminates loop via requestExit (spy + integration).

## Validation

`npm test -- repl` green; interactive manual check documented in the PR
description (transcript of §3.1 against the fixture config).

## Dependencies

16, 22, 26 (session assembly reuse), 05, 06.

## Non-goals

Completion (28), pretty rendering/table/progress/spinner (29), multi-line input
(v2), persistent aliases beyond rc (v2).

## Design References

DESIGN.md §3.1, §3.3, §9.9, §11.2, §11.6, §12.3 (T5, T9).
