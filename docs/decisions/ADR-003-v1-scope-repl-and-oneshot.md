# ADR-003: v1 scope — REPL plus one-shot execution, no control flow

- Status: Accepted (human decision, 2026-07-06)
- Deciders: repository owner

## Context

Scope options ranged from a non-interactive CLI to a full scripting language with
files, control flow, and functions. v1 must be shippable and reviewable while still
delivering the "shell" concept.

## Decision

v1 ships exactly two execution surfaces:

1. **Interactive REPL** — completion, persistent history, Ctrl-C pipeline
   cancellation, progress display, elicitation prompts.
2. **One-shot mode** — `mcpsh -c '<program>'` and program-on-stdin when stdin is not
   a TTY; JSON-friendly output rules and stable exit codes for scripting/CI.

Included language features: pipelines (`|`), sequencing (`;`), comments (`#`),
session variables (`set` / `$name`), read-only `$env.NAME`, `$in`, aliases, quoting
and interpolation. An rc file (`~/.config/mcpsh/init.mcpsh`, executed line-by-line at
REPL start) is included because it reuses the exact one-shot program semantics.

Excluded from v1 (deferred to v2): control flow (`if`/`for`/`while`), function
definitions, `.mcpsh` script files with shebang, command substitution `$(...)`,
globbing, `&&`/`||`, redirection operators (use `save`), multi-line REPL input,
job control/background execution.

## Consequences

- Positive: the parser/evaluator stays small enough for granular issue delegation;
  the v1 completion definition is crisp; CI usage works from day one.
- Negative: loops over data must be expressed with data builtins (or external
  `xargs`-style patterns); acceptable for v1.
- The grammar is designed so the v2 additions are extensions, not rewrites
  (statement-level constructs on top of the existing pipeline node).
