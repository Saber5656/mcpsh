# mcpsh — v1 Issue Plan

- Status: Accepted. Derived from [DESIGN.md](./DESIGN.md); issue bodies live in
  `docs/issues/NN-*.md` and are the source for GitHub Issues (GitHub is derived
  state; this file and the issue files win on conflict).
- 34 issues, numbered 01–34. Issue file numbers are canonical identifiers;
  GitHub issue numbers may differ.

## v1 completion statement

v1 is complete when **all 34 issues below are implemented and their Validation
sections pass in CI**. At that point:

> A user can clone the repo, build it, configure stdio and Streamable-HTTP MCP
> servers in `~/.config/mcpsh/config.json`, and use both the interactive REPL
> (completion, history, cancellation, progress, elicitation) and one-shot
> `mcpsh -c '...'` mode to compose MCP tools, data/format builtins, and external
> Unix commands in structured pipelines — with the documented security defaults
> (no shell execution, env allowlist, output sanitization, secret redaction,
> size/time limits), full user documentation, a published threat model, CI
> gates, and a human-gated release workflow ready to publish once the npm name
> is decided (ADR-008).

Publishing to npm is explicitly **not** part of v1 completion (merge ≠ release).
Newly discovered implementation unknowns may add issues; they must be added to
this file first (see Known unknowns).

## Issue list (recommended execution order)

| # | File | Title | Depends on |
|---|---|---|---|
| 01 | issues/01-project-scaffolding.md | Project scaffolding: TypeScript/ESM package, toolchain, CLI stub | — |
| 02 | issues/02-ci-pipeline.md | CI pipeline: build/test matrix, CodeQL, dependency review, audit | 01 |
| 04 | issues/04-error-taxonomy.md | Error taxonomy: classes, exit codes, formatting | 01 |
| 05 | issues/05-logger-redaction.md | Logger with secret redaction | 01 |
| 06 | issues/06-output-sanitization.md | Terminal output sanitization module | 01 |
| 08 | issues/08-fixture-mcp-server.md | Fixture MCP server for integration testing | 01 |
| 17 | issues/17-lexer.md | Lexer: tokens, quoting, expansions, comments | 01, 04 |
| 03 | issues/03-value-model.md | Value model: types, path access, size, JSON bridge | 01, 04 |
| 07 | issues/07-config-schema-loader.md | Config schema and loader with env-ref resolution | 01, 04, 05 |
| 18 | issues/18-parser-ast.md | Parser and AST | 04, 17 |
| 09 | issues/09-mcp-stdio-lifecycle.md | MCP manager: stdio transport and lifecycle FSM | 04, 05, 07, 08 |
| 13 | issues/13-arg-coercion.md | Flag-to-argument mapping with schema-aware coercion | 03, 04 |
| 19 | issues/19-evaluator-core.md | Evaluator core + builtin registry framework | 03, 04, 17, 18 |
| 10 | issues/10-mcp-http-transport.md | MCP manager: Streamable HTTP transport | 07, 08, 09 |
| 11 | issues/11-tool-registry-namespacing.md | Tool registry: cache, namespacing, resolution rules | 08, 09 |
| 21 | issues/21-external-exec.md | External command execution with serialization boundary | 03, 04, 19 |
| 22 | issues/22-builtins-session.md | Session builtins | 03, 04, 19 |
| 24 | issues/24-builtins-data.md | Data builtins | 03, 19 |
| 25 | issues/25-builtins-format.md | Format builtins | 03, 19 |
| 12 | issues/12-tool-invocation-unwrapping.md | Tool invocation: timeouts, cancellation, unwrapping | 03, 04, 08, 09, 11 |
| 14 | issues/14-resources-support.md | Resources support: listing and reading | 03, 04, 08, 09, 11 |
| 15 | issues/15-prompts-support.md | Prompts support: listing and getting | 03, 04, 08, 09, 11, 12 |
| 16 | issues/16-interaction-callbacks.md | Elicitation, progress, logging, sampling refusal | 04, 05, 06, 08, 09, 12 |
| 20 | issues/20-command-resolution.md | Command resolution order with suggestions | 11, 12, 13, 19, 21 |
| 23 | issues/23-builtins-server.md | Server builtins | 09–15, 19, 20 |
| 26 | issues/26-cli-oneshot.md | CLI entry and one-shot mode | 05–07, 09, 12, 13, 19–25 |
| 27 | issues/27-repl-loop.md | REPL loop: readline, history, cancellation, rc, elicitation | 05, 06, 16, 22, 26 |
| 28 | issues/28-repl-completion.md | REPL tab completion | 11, 17, 27 |
| 29 | issues/29-rendering-progress.md | Rendering: pretty printer, table, progress | 03, 06, 16, 19, 26, 27 |
| 30 | issues/30-integration-e2e.md | End-to-end integration suite | 08, 26, 27, 29 |
| 31 | issues/31-security-test-suite.md | Adversarial security test suite | 30 (+ features under attack) |
| 32 | issues/32-user-docs.md | User documentation: README and USAGE reference | 26, 27, 29 (30 recommended) |
| 33 | issues/33-security-policy.md | Security policy and published threat model | 31, 32 |
| 34 | issues/34-release-packaging.md | Release packaging: human-gated publish workflow | 01, 02, 30–33 |

Notes:
- Issue 07 creates `src/builtins/names.ts` (canonical builtin name constant) if
  absent; 19/22–25 meta-tests keep the registry consistent with it.
- Issue 20 may land with `runExternal` wired as a throwing stub if 21 has not
  merged (noted in 20's test plan); final wiring completes when both are in.

## Implementation waves

| Wave | Issues (parallelizable within a wave) | Intra-wave ordering |
|---|---|---|
| W1 Scaffold | 01 | — |
| W2 Foundations | 02, 04, 05, 06, 08, 17 | none (fully parallel) |
| W3 Building blocks | 03, 07, 18 | none |
| W4 Engines | 09, 13, 19 | none |
| W5 Capabilities | 10, 11, 21, 22, 24, 25 | none |
| W6 Tool & resource calls | 12, 14 | none |
| W7 Feature completion | 15, 16, 20 | none |
| W8 Surfaces I | 23, 26 | 23 → 26 |
| W9 Surfaces II | 27, 28, 29 | 27 → {28, 29} |
| W10 Journeys | 30 | — |
| W11 Hardening & docs | 31, 32 | none |
| W12 Release readiness | 33, 34 | 33 → 34 |

Each wave merges only when its issues' Validation sections pass in CI (wave gate).
One issue = one branch = one PR, per repository policy.

## Dependency graph

```
01 ─┬─ 02
    ├─ 04 ─┬─ 03 ─┬─ 13 ── 20 ─┬─ 23 ── 26 ── 27 ─┬─ 28
    │      │      ├─ 19 ─┬─ 21 ─┘(20)             ├─ 29 ── 30 ─┬─ 31 ── 33 ── 34
    │      │      │      ├─ 22, 24, 25 ──────► 26 │            └─(32)
    │      │      │      └────────────────────────┘
    │      ├─ 17 ── 18 ── 19
    │      └─ 07 ── 09 ─┬─ 10
    ├─ 05 ──┘(07)       ├─ 11 ─┬─ 12 ─┬─ 15, 16 ── 27
    ├─ 06 ─────────► 16, 29    ├─ 14  └─ 20
    └─ 08 ─────────► 09, 11, 12, 14, 15, 16, 30, 31
32 ← 26, 27, 29        (arrows point from prerequisite to dependent)
```

The table above is authoritative where the ASCII graph is ambiguous.

## Coverage table (DESIGN.md § → issues)

| DESIGN.md section | Covered by issues |
|---|---|
| §1–§2 Overview, goals | whole plan (this file) |
| §3 UX transcripts | 26, 27, 30 (executable contracts) |
| §4 Architecture, module map, dep rule | 01, 19, 20 (host pattern), all module issues |
| §5 Value model | 03 |
| §6 Lexical structure, grammar, expansion, aliases | 17, 18, 19, 22 |
| §7 Pipeline semantics, $in, serialization, externals, variables | 19, 21, 22 |
| §8 Command resolution | 20, 23 (`call`) |
| §9.1–§9.2 Config | 07 |
| §9.3 Lifecycle FSM | 09, 10 |
| §9.4 Namespacing/registry | 11 |
| §9.5–§9.7 Invocation, args, unwrap | 12, 13 |
| §9.8 Resources, prompts | 14, 15, 23 |
| §9.9 Server-initiated interactions | 16, 27 |
| §10 Builtins | 22, 23, 24, 25, 29 (`table`) |
| §11.1–§11.4 CLI, modes, exit codes, output | 26 |
| §11.5 Rendering | 29 |
| §11.6 REPL behaviors | 27 |
| §11.7 Completion | 28 |
| §12 Security model | 05, 06, 07, 09, 10, 16, 21, 25, 27, 28 (mechanisms); 02, 34 (T12); 31 (adversarial proof); 33 (publication) |
| §13 Error taxonomy | 04, 26 |
| §14 Testing strategy | 02, 08, 30, 31 (+ per-issue Validation) |
| §15 Distribution/release | 01, 34 |
| §16 Known unknowns | this file (below) |

Every DESIGN.md behavioral section maps to at least one issue; no v1 behavior
lives only in prose outside this plan.

## Validation strategy (whole product)

1. **Per-issue**: each issue's Validation section is the merge gate (unit or
   integration tests named there; CI matrix from issue 02).
2. **Per-wave**: a wave closes only with CI green on the merged combination
   (`npm test` full suite — no issue may disable another's tests).
3. **Journey level**: issue 30 pins the DESIGN §3 transcripts as executable
   contracts; runs in the same CI matrix ({ubuntu, macos} × {node 22, 24}).
4. **Adversarial level**: issue 31 attacks every §12.3 mitigation; its suite is a
   required check from the moment it merges.
5. **Documentation level**: issues 32/33 are validated by sweep checks (every
   builtin/flag/exit code documented; every security claim test-linked or tagged
   unverified).
6. **Release readiness**: issue 34's gates (pack contents, private-flag guard,
   human-only publish) close v1.
7. Aspirational, non-gating: ≥80% line coverage in `src/lang`, `src/values`,
   `src/mcp` (reported, not enforced — deterministic CI per DESIGN §14).

## Deferred v2 items

From ADR-003/004/006 and DESIGN §2.2 — recorded here so nothing silently drops:

| Area | Deferred item |
|---|---|
| Language | control flow (`if`/`for`), functions, `.mcpsh` scripts + shebang, `$(...)`, globbing, `&&`/`\|\|`, redirection, multi-line REPL input, explicit arg-spread for vars (ADR-005 consequence) |
| MCP | sampling mediation with per-request approval (ADR-004), OAuth 2.1 for HTTP servers, `completion/complete`, roots capability, resource subscriptions, resource templates UX, SDK v2 / spec 2026-07-28 migration (ADR-007) |
| Config | project-local config discovery with direnv-style trust/pinning, `mcpsh config import` from Claude Desktop/Code/Cursor (ADR-006) |
| Execution | fd-level streaming between adjacent externals, per-stage parallelism, `uniq`/`group-by` and richer data builtins, CSV/YAML formats, binary reads |
| Surfaces | persistent cross-process sessions, `mcpsh doctor`, themes, pager, Windows CI + support, single-binary packaging, Homebrew distribution |
| Security | OS sandboxing wrappers for stdio servers, continuous fuzzing |

## Known unknowns (may create additional issues during implementation)

| # | Unknown | Trigger to act | Likely consequence |
|---|---|---|---|
| U1 | Final public npm/package name (ADR-008) | Owner decision before first publish | README/34 updates; possibly repo rename sweep |
| U2 | SDK v1.x behavior gaps vs 2025-11-25 spec discovered during Wave 4–7 | Any SDK bug/missing client API | Workaround issue or pinned-version bump |
| U3 | SDK v2 / spec 2026-07-28 release timing (lands mid-implementation) | ADR-007 says: stay on v1.x regardless | Post-v1 migration issue (already deferred) |
| U4 | node-pty install reliability in CI for the REPL smoke test | Flaky/failed installs | Drop pty test to local-only; REPL coverage stays at unit level (30) |
| U5 | Streamable HTTP behavior variance across real servers (sessions, 404 re-init, auth quirks) | First real-server testing after W6 | Compatibility fixes in 10/12; possible new issue |
| U6 | `zod` major version compatibility with SDK peer expectations | npm resolution conflicts at 01 | Pin adjustment in 01 |
| U7 | Terminal-width/encoding edge cases in table/progress rendering on macOS Terminal vs iTerm vs Linux | Manual QA during W9 | Rendering fixes in 29 |
| U8 | GitHub private vulnerability reporting availability on the repo | Owner settings check during 33 | Fallback: security contact email in SECURITY.md |

Adding an issue: append here (list + wave + dependencies), create
`docs/issues/NN-*.md` with the standard sections, then open the GitHub issue —
in that order (docs first; GitHub is derived).
