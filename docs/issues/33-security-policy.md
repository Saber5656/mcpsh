# Title

Security policy and published threat model

## Summary

Add `SECURITY.md` (vulnerability reporting policy) and `docs/SECURITY-MODEL.md`
(user/researcher-facing threat model derived from DESIGN.md §12), and run the
secure-defaults verification checklist with recorded evidence.

## Context

Public OSS release requires a disclosure path and an honest statement of the
security model — including residual risks (§12.8) — so users can make informed
trust decisions and researchers know scope.

## Scope

- `SECURITY.md`, `docs/SECURITY-MODEL.md`, checklist execution evidence in the PR.

## Detailed Requirements

1. `SECURITY.md`:
   - Supported versions table (pre-1.0: latest minor only).
   - Private reporting channel: GitHub private vulnerability reporting (enable the
     repo setting — note for the repo owner in the PR if not already enabled;
     agents must not change repo settings themselves).
   - Response targets: acknowledge ≤7 days, fix-or-plan ≤90 days.
   - Scope statement: in-scope (mcpsh code, defaults, docs promising behavior);
     out-of-scope (vulnerabilities in user-configured MCP servers, Node.js,
     dependencies — report upstream; §12.8 residual risks by design).
2. `docs/SECURITY-MODEL.md` (≤200 lines, user-facing):
   - Trust boundary summary (§12.2 table, reworded for users).
   - What mcpsh guarantees: the §12.3 mitigations in plain language, each linked
     to its test in `test/security/` (issue 31) as evidence.
   - What mcpsh does NOT guarantee: §12.8 verbatim-equivalent + "servers run with
     your privileges — choose them like you choose shell scripts to run".
   - Secret-handling guidance for users (§12.5 user half: env refs, history
     space-prefix, trace warning).
3. Secure-defaults verification checklist (executed manually once, transcript in
   PR description; each row also mapped to an automated test where one exists):
   - fresh install defaults: no config → no connects; history 0600; sanitized
     render on by default; `--verbose` off; external stdin serialization exact;
     HTTPS enforcement on a non-localhost `http:` config → `E_CONFIG`.
4. README security-highlights section (issue 32) linked to SECURITY-MODEL.md —
   add the link if 32 already merged, else coordinate ordering in the PR.

## Acceptance Criteria

- [ ] Both files exist with all sections above; SECURITY-MODEL.md links resolve
      (docs + test files).
- [ ] Checklist executed with evidence (commands + outputs) attached to the PR.
- [ ] No claim in SECURITY-MODEL.md lacks either a test reference or an explicit
      "not automatically verified" tag.

## Validation

Docs review; link check; cross-review against DESIGN.md §12 for drift.

## Dependencies

31 (tests to cite), 32 (link coordination).

## Non-goals

Bug bounty program, security audits (post-v1), signing/SLSA beyond npm provenance
(34).

## Design References

DESIGN.md §12 (entire), §15.
