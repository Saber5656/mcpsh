# Title

Release packaging: publish workflow (human-gated), CHANGELOG, package hygiene

## Summary

Prepare everything needed to publish — npm publish workflow with provenance
(manual dispatch, hard-gated on the naming decision), CHANGELOG, package metadata
polish — WITHOUT publishing. Publishing remains a human action per ADR-008 and
repository policy (merge ≠ release).

## Context

v1 completion means "ready to release", not "released". The npm name is undecided
(ADR-008); the workflow must make accidental publication impossible while making
intentional publication one decision + one click for the owner.

## Scope

- `.github/workflows/release.yml`, `CHANGELOG.md`, package.json final polish,
  `docs/RELEASING.md`.

## Detailed Requirements

1. `release.yml`:
   - Trigger: `workflow_dispatch` only, with an input `confirm` requiring the
     literal string `publish`.
   - Steps: checkout → setup-node (registry-url set) → `npm ci` → full test suite
     → `npm run build` → `npm pack` → assert `package.json` `private` is absent
     and `name` is not bare `mcpsh`-with-private (i.e., the ADR-008 decision has
     been executed) — else fail with a message linking ADR-008 →
     `npm publish --provenance --access public` using `NPM_TOKEN` secret.
   - `permissions: { contents: read, id-token: write }` (provenance requirement).
   - The workflow must fail cleanly when `NPM_TOKEN` is absent (secret is
     configured manually by the owner, never by agents).
2. `CHANGELOG.md`: keep-a-changelog format; `[Unreleased]` section; retroactive
   `0.1.0` entry summarizing v1 feature set (one line per DESIGN.md §2.1 goal).
3. package.json polish: `description`, `keywords` (`mcp`, `model-context-protocol`,
   `shell`, `cli`, `pipeline`), `repository`, `bugs`, `homepage` fields;
   `"files": ["dist", "config.schema.json"]`; verify `npm pack --dry-run` file
   list contains exactly dist + schema + README + LICENSE + package.json (test
   asserted in CI via a script `scripts/check-pack.ts`).
4. `docs/RELEASING.md`: owner runbook — decide name (ADR-008 options), flip
   `private`/`name`, set `NPM_TOKEN` (manual), update CHANGELOG + version,
   tag `vX.Y.Z`, dispatch workflow, post-publish smoke (`npx <name> --version`).
   Explicit note: merging PRs never publishes; only this workflow does.
5. Version policy statement in RELEASING.md per DESIGN.md §15 (0.x → 0.9 RC at
   v1 completion → 1.0.0 after naming + owner sign-off).

## Acceptance Criteria

- [ ] Workflow dry-run (dispatch on a fork or with publish step behind the gate)
      demonstrates: gate fails while `private: true`; all pre-publish steps green.
- [ ] `scripts/check-pack.ts` runs in CI and passes; adding a stray file to
      `files` fails it (demonstrated then reverted).
- [ ] CHANGELOG and RELEASING exist with all sections; RELEASING contains zero
      agent-executable secret steps (review).
- [ ] No publish occurs (verify npm registry unchanged).

## Validation

CI green including check-pack; manual dispatch test evidence in PR.

## Dependencies

01, 02, 30–33 (full validation before release readiness).

## Non-goals

Actually publishing (human gate), Homebrew/other channels (v2), single-binary
builds (v2), release notes automation.

## Design References

DESIGN.md §12.3 (T12), §15; ADR-008.
