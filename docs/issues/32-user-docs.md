# Title

User documentation: README and USAGE reference

## Summary

Rewrite `README.md` (install, quickstart, worked examples, positioning) and write
`docs/USAGE.md` (complete reference: grammar, builtins, config, flags, exit codes,
environment variables, security notes for users).

## Context

Docs are part of the v1 completion definition (public OSS repo). USAGE.md is the
user-facing distillation of DESIGN.md — normative for behavior, but written for
users, not implementers.

## Scope

- `README.md`, `docs/USAGE.md`. English. No changes to DESIGN.md.

## Detailed Requirements

1. README sections, in order:
   - One-paragraph pitch + the §1 example pipeline.
   - Status banner: pre-release, npm name pending (ADR-008) — install via
     `git clone && npm ci && npm run build && npm link` until first release.
   - Quickstart: minimal config file (one stdio server, one HTTP server with
     `${env:VAR}` header), first REPL session transcript, first one-shot command.
   - Three worked examples: multi-server pipeline; external-command mix
     (`to jsonl | ^jq ...`); CI usage with exit codes.
   - Comparison table (one line each vs mcpc / mcptools / morinokami-mcpsh /
     plain Inspector) derived from docs/research/prior-art.md.
   - Security highlights (5 bullets from §12: no shell, env allowlist, sanitized
     output, env-ref secrets, explicit config) linking to USAGE security section.
   - Link map: USAGE.md, DESIGN.md, CONTRIBUTING-less note (contributions:
     "design docs in docs/ are canonical; see ISSUE_PLAN.md"), LICENSE.
2. USAGE.md sections:
   - Invocation and flags (§11.1), modes (§11.2), exit codes table (§11.3),
     output rules (§11.4).
   - Language reference: tokens/quoting/escapes, grammar (EBNF from §6.2),
     expansions and interpolation table (§6.3), aliases, variables, `$in` rules
     (§7.2) with the `E_UNBOUND_INPUT` rationale, serialization table (§7.3).
   - Command resolution order + `^` + `call` (§8).
   - Builtins: the full §10 table transcribed user-legibly, one short example per
     builtin (every builtin in the §10 table, including `table`, gets exactly one).
   - Config reference: full annotated example (§9.1), field table (§9.2), env
     refs, timeouts/limits, TLS rules.
   - MCP behavior: unwrap rules in user terms (§9.7), elicitation UX, progress,
     what happens with sampling-requiring servers (ADR-004 user-facing note).
   - Security notes for users: history caveat (secrets typed in commands persist
     unless space-prefixed — T5), server trust guidance (§12.2 B3, §12.8), PATH
     hygiene, `--verbose`/trace payload warning.
   - Troubleshooting: 8 common errors (`E_` codes) with causes and fixes.
3. Every code sample in both files must be executable against the fixture server
   or a public reference server — mark each sample with its verification method;
   samples used in tests (issue 30) noted as such.
4. Keep line width ≤100; tables per repository doc style.

## Acceptance Criteria

- [ ] README quickstart followed verbatim on a clean checkout works (validated in
      PR by a recorded transcript).
- [ ] USAGE.md documents every builtin (registry-name sweep — no missing, no
      phantom), every CLI flag, every exit code, every `E_` code.
- [ ] No contradiction with DESIGN.md (reviewer checklist item; spot-check §6/§7
      tables match).

## Validation

Docs review against the running product; `help` output cross-checked with
USAGE.md builtin list.

## Dependencies

26, 27, 29 (behavior frozen); 30 recommended first (samples ↔ tests alignment).

## Non-goals

Man pages, website, tutorials/blog posts, CONTRIBUTING.md (v2), translated docs.

## Design References

DESIGN.md §3, §6–§13; ADR-004, ADR-008; research/prior-art.md.
