# Title

Config schema and loader with env-ref resolution

## Summary

Implement `src/config/schema.ts` (zod schema for the config file) and
`src/config/load.ts` (location resolution, strict validation, `${env:VAR}`
resolution with redactor registration), per DESIGN.md §9.1–§9.2 and ADR-006.

## Context

The config file is the only way servers enter a session. It must be strict (typo
protection), keep secrets out of file text via env refs, and never auto-load from
the current directory (threat T8).

## Scope

- `src/config/schema.ts`, `src/config/load.ts`, test fixture config files, unit
  tests. Also emit `config.schema.json` (JSON Schema generated from the zod schema
  at build time via a script `scripts/gen-config-schema.ts`, committed output).

## Detailed Requirements

1. Schema (zod, `strict()` everywhere — unknown keys fail with JSON path):
   - Top level: `{ $schema?: string, mcpServers?: Record<string, ServerConfig>,
     limits?: { maxOutputBytes?: number (int, ≥1024, default 10_000_000) },
     history?: { file?: string|null, maxEntries?: number (int ≥0, default 10000) } }`.
   - `ServerConfig` = stdio variant `{ command: string, args?: string[],
     env?: Record<string,string>, cwd?: string, enabled?: boolean,
     inheritEnv?: boolean, timeouts?: Timeouts, limits?: {maxOutputBytes?} }`
     XOR http variant `{ url: string (valid URL), headers?: Record<string,string>,
     allowInsecureHttp?: boolean, enabled?: boolean, timeouts?: Timeouts,
     limits?: {maxOutputBytes?} }`. Exactly one of `command`/`url` — both or
     neither → `E_CONFIG`.
   - `Timeouts = { connectMs?: number (int 100..600000, default 10000),
     callMs?: number (int 100..3600000, default 60000) }`.
   - Server alias keys: `^[A-Za-z_][A-Za-z0-9_-]*$`; collision with builtin names
     → `E_CONFIG`. The builtin name list lives in `src/builtins/names.ts` (a plain
     string-array constant with no imports): **create it in this issue** from the
     DESIGN.md §10 table if it does not exist yet; issue 19's registry meta-tests
     assert the registry stays consistent with it.
2. Loader `loadConfig(opts: { flagPath?: string }): ResolvedConfig`:
   - Location: `flagPath` > `process.env.MCPSH_CONFIG` >
     `$XDG_CONFIG_HOME/mcpsh/config.json` (default `~/.config/mcpsh/config.json`).
   - Missing file at the default location → empty config (no servers). Missing file
     at an explicit location (flag/env) → `E_CONFIG` (user asked for it).
   - Unreadable/invalid JSON/schema violation → `E_CONFIG` with file path and JSON
     path; zod issues mapped to a readable one-line summary.
   - `http:` URL for non-localhost host without `allowInsecureHttp: true` →
     `E_CONFIG` (localhost = `localhost`, `127.0.0.0/8`, `::1`). With the flag →
     accepted + `warn` log at load.
   - `inheritEnv: true` → `warn` log at load (§12.4).
3. Env refs: values in `env` and `headers` may contain `${env:VAR}` (whole-string
   or embedded, multiple allowed). Provide
   `resolveEnvRefs(template: string, context: string): string` used **at transport
   build time** (issues 09/10), not at load: unset var → `E_CONFIG` naming var and
   server but never the value; each resolved value is passed to
   `registerSecret` (issue 05) before returning. Loader only *syntax-checks* refs.
4. `ResolvedConfig` exposes: ordered server map with normalized defaults, global
   limits, history settings, and the config file path (or null) for diagnostics.

## Acceptance Criteria

- [ ] Fixtures + tests: valid full config; unknown key (path in message); bad alias;
      both `command`+`url`; neither; bad URL; non-localhost `http:` rejected;
      localhost `http:` accepted; `allowInsecureHttp` warning logged; env-ref
      syntax error; `resolveEnvRefs` unset-var error message contains var name and
      server alias but not any value; resolved values registered with redactor
      (spy); defaults applied (timeouts/limits/history).
- [ ] Precedence test: flag > env > default path (use temp dirs + env stubs).
- [ ] `scripts/gen-config-schema.ts` regenerates `config.schema.json`
      deterministically; CI-checked no-diff (wire into `npm run build` or a
      `check:schema` script executed by CI issue 02's `test` job — add script now,
      CI picks it up automatically via `npm test` if implemented as a test).

## Validation

`npm test -- config` green; manual: `MCPSH_CONFIG=fixtures/valid.json node -e ...`
loads.

## Dependencies

01, 04, 05.

## Non-goals

Project-local discovery/trust (v2, ADR-006), config import from other clients (v2),
transport construction (09/10), OAuth (v2).

## Design References

DESIGN.md §9.1, §9.2, §12.3 (T6, T8), §12.4, §12.5; ADR-006.
