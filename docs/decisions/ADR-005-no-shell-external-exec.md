# ADR-005: External commands run without a shell; expansions never re-split

- Status: Accepted (design decision, 2026-07-06)
- Deciders: design agent (conservative default; security-motivated)

## Context

mcpsh pipes untrusted data (MCP tool output) into and around external commands. The
classic injection vectors in shells are (a) building command lines through
`sh -c` string concatenation and (b) word-splitting/glob-expanding variable
contents.

## Decision

1. External commands are spawned **directly with an argv array**
   (`child_process.spawn(cmd, args, { shell: false })`). mcpsh never invokes
   `/bin/sh -c`. If a user wants shell behavior they must write `sh -c '...'`
   themselves, explicitly.
2. Expansion results are **never re-parsed**: a `$var`/`$in`/`$env.X` expansion (or
   an interpolated string) always yields exactly one argument, regardless of the
   spaces, quotes, semicolons, or newlines it contains. Equivalent to always writing
   `"$var"` in POSIX shells.
3. There is no glob expansion in v1, so no expansion-triggered filesystem access.

## Consequences

- Positive: tool output containing `; rm -rf ~` or backticks is inert data end to
  end. Verified by an adversarial test (issue 31).
- Negative: users cannot rely on `$var` expanding to multiple arguments; a future
  explicit spread mechanism (v2) would be required for that pattern.
- PATH lookup still applies to external commands, as in any shell; PATH hygiene
  remains the user's responsibility (documented in DESIGN.md §12.8).
