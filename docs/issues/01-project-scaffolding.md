# Title

Project scaffolding: TypeScript/ESM package, toolchain, and CLI stub

## Summary

Create the initial Node.js/TypeScript project skeleton with strict compiler
settings, lint/format/test toolchain, package metadata per the design, and a
`mcpsh` bin stub that prints the version. No product features.

## Context

Empty repository (README only). Every other issue builds on this skeleton.
Runtime/toolchain decisions are fixed by ADR-001 (TypeScript/Node ≥22.12, ESM) and
DESIGN.md §4.3 (dependency policy) and §15 (package shape, `private: true` per
ADR-008).

## Scope

- `package.json`, `tsconfig.json`, eslint + prettier config, vitest config,
  `.gitignore`, `.editorconfig`, `LICENSE` (MIT), `src/cli.ts` stub, `src/version.ts`.
- npm scripts: `build`, `test`, `lint`, `format`, `typecheck`.

## Detailed Requirements

1. `package.json`:
   - `"name": "mcpsh"`, `"private": true`, `"version": "0.1.0"`, `"type": "module"`,
     `"license": "MIT"`, `"engines": { "node": ">=22.12.0" }`,
     `"bin": { "mcpsh": "dist/cli.js" }`, `"files": ["dist"]`.
   - Runtime dependencies: exactly `@modelcontextprotocol/sdk@^1.29` and `zod`
     (install latest zod major compatible with the SDK). No other runtime deps.
   - Dev dependencies: `typescript`, `vitest`, `@types/node`, `eslint` (flat
     config) + `typescript-eslint`, `prettier`.
   - Scripts: `build` → `tsc -p tsconfig.json`; `typecheck` → `tsc --noEmit`;
     `test` → `vitest run`; `lint` → `eslint .`; `format` → `prettier --check .`;
     `format:fix` → `prettier --write .`.
2. `tsconfig.json`: `strict: true`, `module: "NodeNext"`,
   `moduleResolution: "NodeNext"`, `target: "ES2023"`, `outDir: "dist"`,
   `rootDir: "src"`, `declaration: false`, `sourceMap: true`,
   `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`.
3. `src/cli.ts`: shebang `#!/usr/bin/env node`; prints `mcpsh <version>` and exits 0
   when invoked with `--version`; any other invocation prints
   `mcpsh: not implemented yet` to stderr and exits 1. Version read from
   `src/version.ts` (a constant kept in sync with package.json; add a unit test that
   compares it with `package.json` version).
4. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `*.tsbuildinfo`.
5. `LICENSE`: MIT, copyright holder "mcpsh contributors".
6. Commit `package-lock.json` (lockfile required, DESIGN.md §12.3 T12).
7. One example unit test (`test/version.test.ts`) proving vitest wiring.

## Acceptance Criteria

- [ ] `npm ci && npm run build && npm test && npm run lint && npm run typecheck &&
      npm run format` all succeed on Node 22 and 24.
- [ ] `node dist/cli.js --version` prints `mcpsh 0.1.0` and exits 0.
- [ ] `npm ls --omit=dev --all` shows only `@modelcontextprotocol/sdk`, `zod`, and
      their transitive deps.
- [ ] `package.json` contains `"private": true`.

## Validation

Run the command chain above locally; CI enforcement arrives with issue 02.

## Dependencies

None (first issue).

## Non-goals

CI workflows (02), any parsing/MCP/REPL logic, README rewrite (32), publish
configuration beyond `private: true` (34).

## Design References

DESIGN.md §4.2, §4.3, §15; ADR-001, ADR-008.
