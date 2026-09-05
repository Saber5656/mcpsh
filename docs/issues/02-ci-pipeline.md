# Title

CI pipeline: build/test matrix, CodeQL, dependency review, audit

## Summary

Add GitHub Actions workflows that gate every PR on lint, typecheck, tests, and
`npm pack` across an OS/Node matrix, plus security scanning (CodeQL, dependency
review, npm audit report).

## Context

The repository is public and intended for OSS release; CI is the enforcement point
for the validation strategy (DESIGN.md §14) and supply-chain posture (§12.3 T12).

## Scope

- `.github/workflows/ci.yml`
- `.github/workflows/codeql.yml`

## Detailed Requirements

1. `ci.yml`:
   - Triggers: `pull_request`, `push` to `main`.
   - Job `test`: matrix `os: [ubuntu-latest, macos-latest]` ×
     `node: ['22', '24']`; steps: checkout, setup-node (with npm cache),
     `npm ci`, `npm run lint`, `npm run format`, `npm run typecheck`,
     `npm run build`, `npm test`, `npm pack --dry-run`.
   - Job `audit` (ubuntu only, non-blocking: `continue-on-error: true`):
     `npm audit --omit=dev`; failure must not block merge but must be visible.
   - Top-level `permissions: { contents: read }`. Pin all actions to major version
     tags at minimum (e.g. `actions/checkout@v4`).
   - `concurrency` group per ref with `cancel-in-progress: true`.
2. `codeql.yml`: CodeQL default setup for `javascript-typescript`; triggers:
   `push` to `main`, `pull_request`, weekly `schedule`. Permissions per CodeQL docs
   (`security-events: write`, `contents: read`).
3. Dependency review: add a `dependency-review` job in `ci.yml` using
   `actions/dependency-review-action` on `pull_request` (fails on critical/high
   vulnerabilities in newly added deps).
4. Do not add coverage gates (deterministic CI; DESIGN.md §14).

## Acceptance Criteria

- [ ] A PR touching source runs the full matrix and all jobs pass on the current
      codebase.
- [ ] CodeQL completes and uploads results (visible under Security tab).
- [ ] `audit` job failure does not fail the workflow run.
- [ ] All workflows have explicit `permissions` blocks.

## Validation

Open a draft PR with a trivial change; verify all jobs run and pass; verify a
deliberately broken lint rule fails the `test` job (then revert).

## Dependencies

01

## Non-goals

Release/publish workflow (34), Windows CI (v2), coverage reporting services.

## Design References

DESIGN.md §12.3 (T12), §14, §15.
