# mcpsh — Design Document (v1)

- Status: Accepted design for v1. Canonical source of truth together with
  [ISSUE_PLAN.md](./ISSUE_PLAN.md), `docs/issues/*.md`, `docs/decisions/*.md`,
  `docs/research/*.md`.
- Scope decisions ratified by the repository owner on 2026-07-06:
  TypeScript/Node (ADR-001), hybrid structured pipelines (ADR-002),
  REPL + one-shot scope (ADR-003), no LLM features (ADR-004).
- Section numbers (§N) are stable identifiers referenced by issue files. Do not
  renumber sections without updating `docs/issues/*.md` and ISSUE_PLAN.md.

## §1 Overview

mcpsh is a Unix-pipeline-inspired shell for composing MCP (Model Context Protocol)
tools. It connects to multiple MCP servers in one session, exposes every tool as a
first-class command, and lets structured values flow through pipelines that freely
mix MCP tools, built-in data operators, and external Unix commands.

```
mcpsh> github.search_issues --repo modelcontextprotocol/typescript-sdk --state open \
       | get items | where labels contains bug | select number title | table
```

One sentence per design pillar:

- **It is a shell**, not a subcommand CLI: interactive REPL with completion plus a
  scriptable one-shot mode (ADR-003).
- **Pipes carry structured values** internally and serialize deterministically at
  external-command boundaries (ADR-002).
- **It is deterministic**: no LLM anywhere in v1 (ADR-004).
- **It is secure by default**: no shell interpolation, env allowlists, secret
  redaction, output sanitization, explicit config trust (§12).

Differentiation against existing MCP CLIs is documented in
[research/prior-art.md](./research/prior-art.md).

## §2 Goals and non-goals

### §2.1 v1 goals

| # | Goal |
|---|---|
| G1 | Connect to multiple MCP servers (stdio + Streamable HTTP) from one config file |
| G2 | Invoke any MCP tool as a command with schema-aware flag arguments |
| G3 | Compose tools, builtins, and external commands with `\|` over structured values |
| G4 | Interactive REPL: completion, history, cancellation, progress, elicitation |
| G5 | Non-interactive one-shot mode usable from scripts and CI with stable exit codes |
| G6 | List/read resources and list/get prompts from connected servers |
| G7 | Secure defaults per §12; documented threat model |
| G8 | Tested (unit + integration against a fixture MCP server) and CI-gated |

### §2.2 v1 non-goals (deferred to v2 — tracked in ISSUE_PLAN.md §Deferred)

- Control flow (`if`/`for`), functions, `.mcpsh` script files, command substitution,
  globbing, `&&`/`||`, redirection operators, multi-line REPL input (ADR-003)
- LLM features: sampling mediation, NL→pipeline (ADR-004)
- OAuth flows for remote servers (static headers only in v1)
- Project-local config auto-discovery and its trust model; config import (ADR-006)
- Cross-process persistent sessions (mcpc-style daemons)
- `completion/complete` (MCP argument autocompletion), roots capability
- Windows support (best-effort only; not CI-gated in v1)
- Plugins/custom builtins, themes, table styling options
- Single-binary packaging

### §2.3 Personas

1. **MCP server developer** — pokes at their server's tools/resources during
   development; wants fast iteration and readable errors (replaces Inspector for
   quick checks).
2. **Automation engineer** — wires MCP tools into cron/CI one-liners; wants stable
   exit codes and JSON output.
3. **Power user** — composes multi-server workflows interactively (GitHub → filter →
   filesystem, etc.).

## §3 User experience examples

These transcripts are normative UX targets; integration tests mirror them against
the fixture server (issue 30).

### §3.1 Interactive session

```
$ mcpsh
mcpsh 0.1.0 — 2 servers configured (github, fs). Type 'help' to start.
mcpsh> servers
 alias   transport  state         protocol    tools
 github  stdio      ready         2025-11-25  12
 fs      stdio      configured    -           -
mcpsh> tools github
 github.search_issues   Search issues in a repository
 github.create_issue    Create an issue
 ...
mcpsh> github.search_issues --repo owner/repo --state open | get items | first 3 | select number title
[
  { "number": 41, "title": "Fix flaky test" },
  ...
]
mcpsh> $it | set top3          # error: unknown variable $it (did you mean $in?)
mcpsh> github.search_issues --repo owner/repo --state open | get items | first 3 | set top3
mcpsh> $top3 | to jsonl | ^jq -r '.title'
Fix flaky test
...
mcpsh> exit
```

### §3.2 One-shot / CI usage

```
$ mcpsh -c 'github.search_issues --repo owner/repo --state open | get items | length'
17
$ mcpsh -c 'fs.read_file --path /etc/hostname' > hostname.txt
$ echo 'servers' | mcpsh            # program from stdin when stdin is not a TTY
$ mcpsh -c '...' ; echo $?          # 0 ok / 1 runtime / 2 parse / 3 config / 130 SIGINT
```

### §3.3 Elicitation (REPL only)

```
mcpsh> deploy.rollout --service api
deploy is asking: "Proceed with rollout to production?"
  confirm (boolean) [y/N]: y
{ "status": "started" }
```

Non-interactive mode auto-declines elicitation requests (§9.9).

## §4 Architecture and module map

### §4.1 Runtime shape

Single Node.js process. No daemons, no background state. MCP server subprocesses
(stdio) and HTTP connections live and die with the mcpsh process.

```
┌────────────────────────────── mcpsh process ─────────────────────────────┐
│ cli.ts ──► repl/ (interactive)  or  one-shot runner                      │
│                │                                                          │
│                ▼                                                          │
│ lang/lexer ─► lang/parser ─► lang/eval ─► builtins/*                     │
│                                  │  │                                     │
│                                  │  └─► pipeline/external (spawn argv)   │
│                                  ▼                                        │
│                             mcp/manager ─► SDK Client per server         │
│                                  │            │ stdio: child process     │
│ config/ ─► (servers, limits)     │            │ http: fetch/stream       │
│ log/    ◄── redaction ──────────┴─ render/ ◄─ sanitize                  │
└──────────────────────────────────────────────────────────────────────────┘
```

### §4.2 Source layout (normative)

| Path | Responsibility | Issues |
|---|---|---|
| `src/cli.ts` | argv parsing, mode selection, exit codes | 26 |
| `src/repl/loop.ts` | readline loop, history, cancellation, rc file | 27 |
| `src/repl/complete.ts` | tab completion | 28 |
| `src/render/render.ts` | value rendering, table, progress display | 29 |
| `src/render/sanitize.ts` | control-char/ANSI neutralization | 06 |
| `src/lang/lexer.ts` | tokens | 17 |
| `src/lang/ast.ts`, `src/lang/parser.ts` | AST + parser | 18 |
| `src/lang/eval.ts` | evaluator, variable store, `$in` | 19 |
| `src/lang/resolve.ts` | command resolution order | 20 |
| `src/pipeline/external.ts` | external process execution + serialization | 21 |
| `src/builtins/registry.ts` | builtin metadata/registry framework | 19 |
| `src/builtins/session.ts` | help/version/exit/vars/set/unset/alias/history | 22 |
| `src/builtins/server.ts` | servers/connect/disconnect/tools/describe/call | 23 |
| `src/builtins/data.ts` | get/select/where/first/last/length/flatten/sort-by/reverse | 24 |
| `src/builtins/format.ts` | from/to/lines/echo/save/open | 25 |
| `src/mcp/manager.ts` | connection lifecycle FSM, registry of servers | 09, 10 |
| `src/mcp/tools.ts` | tool list cache, namespacing, invocation, unwrap | 11, 12 |
| `src/mcp/args.ts` | flag→JSON argument mapping and coercion | 13 |
| `src/mcp/resources.ts`, `src/mcp/prompts.ts` | resources/prompts | 14, 15 |
| `src/mcp/interactions.ts` | elicitation/progress/logging/cancellation glue | 16 |
| `src/config/schema.ts`, `src/config/load.ts` | config schema + loader + env refs | 07 |
| `src/values/value.ts`, `src/values/path.ts` | value model, path access, size caps | 03 |
| `src/errors.ts` | error taxonomy, exit-code mapping | 04 |
| `src/log/log.ts`, `src/log/redact.ts` | logger + redaction | 05 |
| `test/fixture-server/` | fixture MCP server for tests | 08 |
| `test/e2e/` | end-to-end tests via built CLI | 30, 31 |

Dependency rule: `lang/` must not import `mcp/` directly; the evaluator receives a
`CommandHost` interface (resolution + invocation callbacks) so language and MCP
layers are independently testable.

### §4.3 Runtime dependency policy

Exactly two runtime dependencies: `@modelcontextprotocol/sdk` (^1.29, ADR-007) and
`zod`. Everything else uses `node:` builtins (`readline/promises`, `util.styleText`,
`child_process`, `fs`). Adding a runtime dependency requires an ADR.

## §5 Value model

### §5.1 Types

A pipeline value is one of:

| Type | JS representation | Notes |
|---|---|---|
| null | `null` | absence of value; also the input to the first stage |
| boolean | `boolean` | |
| number | `number` | finite doubles only; `NaN`/`Infinity` are a `E_TYPE` error at creation |
| string | `string` | UTF-8 text |
| list | `Value[]` | heterogeneous allowed |
| record | `{ [key: string]: Value }` | string keys, insertion-ordered |

There is no first-class binary type in v1. Binary payloads (resource blobs, image
content) are represented as records:
`{ "type": "blob", "mimeType": string, "data": <base64 string>, "bytes": number }`.

### §5.2 Operations (module `src/values`)

- `pathGet(value, path)` — path syntax `a.b.0.c` (identifiers and non-negative
  integer indexes separated by `.`). Missing segment returns a sentinel
  `MISSING` (distinct from null) so callers choose error vs. skip semantics.
- `sizeOf(value)` — approximate byte size; used to enforce `limits.maxOutputBytes`.
- `toJson(value, {pretty})` / `fromJson(text)` — wrap `JSON.parse` errors in
  `E_TYPE` with a 1-line excerpt of the offending text (sanitized, max 120 chars).
- `checkValue(x)` — validates/normalizes foreign JSON into a Value (rejects
  non-finite numbers, non-plain objects).

### §5.3 Size limits

Every stage boundary enforces `limits.maxOutputBytes` (default **10_000_000**).
Applies to: MCP tool results, resource reads, external-command stdout capture, and
`open`. Exceeding it raises `E_LIMIT` naming the stage and the limit. Configurable
globally and per-server (§9.2).

## §6 Language: lexical structure and grammar

### §6.1 Tokens (module `src/lang/lexer.ts`)

| Token | Form |
|---|---|
| WORD | bare run of chars excluding whitespace and `\| ; # ^ $ " '` and control chars. Allowed: letters, digits, `. / \\ - _ : @ + = , ( ) [ ] { } < > ~ * ? !` etc. |
| STRING_SQ | `'...'` — literal, no escapes except `''` → nothing (no interpolation); may contain any char except `'` |
| STRING_DQ | `"..."` — interpolated; escapes `\\` `\"` `\$` `\n` `\t`; contains literal parts and expansion parts |
| EXPANSION | `$name` (identifier: `[A-Za-z_][A-Za-z0-9_]*`), `$in`, `$env.NAME` |
| FLAG | `--name` or `--name=<word-or-string>`; name: `[A-Za-z0-9][A-Za-z0-9_-]*` |
| PIPE | `\|` |
| SEMI | `;` and newline (equivalent statement separators) |
| CARET | `^` only as first char of a command's first element |
| COMMENT | `#` at token start position → skip to end of line |

- A single `-x` or `-abc` is a WORD, not a flag (only `--` introduces flags); this
  keeps external commands like `ls -la` natural.
- A lone `--` WORD terminates flag parsing for builtins/tools (subsequent `--x` are
  words); passed through verbatim to external commands.
- Adjacent parts without whitespace concatenate into one word expression:
  `pre$x'lit'"y $z"` is one WORD-EXPR with 4 parts.
- Lexer errors (`E_PARSE`): unterminated string, invalid `$` expansion, bad flag
  name — all reported with 1-based line and column.

### §6.2 Grammar (EBNF)

```
program   ::= sep* pipeline (sep+ pipeline)* sep*
sep       ::= ";" | NEWLINE
pipeline  ::= command ("|" command)*
command   ::= ["^"] element+
element   ::= word-expr | flag
flag      ::= FLAG            (* may carry inline "=value" word-expr *)
word-expr ::= (WORD | STRING_SQ | STRING_DQ | EXPANSION)+   (* adjacency-concatenated *)
```

The parser (issue 18) produces:

```ts
Program  { pipelines: Pipeline[] }
Pipeline { commands: Command[], pos }
Command  { external: boolean /* ^ */, elements: Element[], pos }
Element  = WordExpr { parts: Part[], raw: string }
         | Flag     { name: string, inline?: WordExpr, raw: string }
Part     = Lit { text } | Var { name } | In {} | Env { name }
```

`raw` preserves the original spelling so external commands receive tokens verbatim
(§7.4). Flag-value binding ("does `--limit` consume the next word?") is **not**
decided by the parser; the consumer decides (§9.6 for tools, §10 for builtins,
verbatim for externals).

### §6.3 Expansion and interpolation semantics

| Context | Rule |
|---|---|
| Word-expr that is exactly one `Var`/`In` part (bare `$x` / `$in`) | Expands to the **value itself**, unstringified. As a command's only element it is a value source (§7.1). |
| `$env.NAME` bare | Expands to a string (env var value); unset → `E_TYPE` error naming the variable |
| Inside STRING_DQ or concatenated word-expr | Stringify: string → as-is; null → empty string; boolean/number → JSON; list/record → compact JSON |
| Unknown `$name` | `E_TYPE` error "unknown variable $name" (+ did-you-mean over defined vars and `$in`) |
| STRING_SQ | Never expands |

Expansion output is never re-tokenized (ADR-005): one word-expr → exactly one
argument/value regardless of content.

### §6.4 Aliases

`alias NAME <word-expr...>` defines a session alias. Expansion: if the first element
of a command is a WORD equal to an alias name, replace it with the alias's stored
elements (textual AST splice, recursion depth max 8 → `E_PARSE`). Aliases never
expand in non-initial position. `unalias NAME` removes. Aliases are session-only in
v1 (persist via rc file).

## §7 Pipeline execution semantics

### §7.1 Evaluation

- Pipelines in a program run sequentially; a pipeline error aborts the remainder of
  the program (REPL prints the error and continues the session; one-shot exits with
  the mapped code, §13).
- Within a pipeline, stages run **sequentially**: stage N completes and its value
  becomes stage N+1's input. v1 fully materializes values between stages (ADR-002).
- The first stage receives input `null`. A command whose only element is a bare
  `$var`/`$in` expansion is a **value source** yielding that value.
- The final value of the last pipeline is the program result, rendered per §11.4.

### §7.2 Input binding (`$in`)

| Stage kind | Behavior with pipeline input |
|---|---|
| Builtin | Declares `inputPolicy: 'requires' \| 'accepts' \| 'ignores'`. `requires` + null input → `E_TYPE`; `ignores` + non-null input → `E_TYPE` ("does not accept pipeline input") |
| MCP tool | If any element references `$in`, input is consumed there. Otherwise non-null input → **`E_UNBOUND_INPUT` error** (silent data loss is forbidden). Null input + no `$in` → fine |
| External command | If any element references `$in`, input binds there and **stdin is closed**. Otherwise input is serialized to the child's stdin (§7.3) |

### §7.3 Serialization to external stdin

| Input value | Bytes written to child stdin |
|---|---|
| null | stdin closed immediately |
| string | the string, verbatim (no added newline) |
| list of strings | each element + `\n` |
| list of records | JSONL: compact JSON per element + `\n` |
| any other value | compact JSON + `\n` |

### §7.4 External command execution (module `src/pipeline/external.ts`)

- argv construction: elements in order; word-exprs expand (§6.3); flags use their
  `raw` spelling. No shell, ever (ADR-005). PATH lookup via `spawn` default.
- stdout: captured (cap §5.3), decoded UTF-8 (invalid sequences → U+FFFD), one
  trailing `\n` stripped (like `$(...)`), result is a string value.
- stderr: passed through to the mcpsh stderr live (not captured, not part of value).
- Exit ≠ 0 → `E_EXTERNAL_FAILED` (message includes command name and exit code or
  signal). The pipeline aborts.
- Cancellation (Ctrl-C): SIGTERM to child, SIGKILL after 3 s grace.
- Spawn env: external commands inherit the full mcpsh environment (they are
  user-invoked programs, same trust as any shell); only MCP stdio servers get the
  allowlist treatment (§12.4).

### §7.5 Variables

- `set NAME` as a sink stores the pipeline input under NAME; `set NAME <word-expr>`
  stores the expanded value. Names: `[A-Za-z_][A-Za-z0-9_]*`, case-sensitive.
- `$NAME` reads (§6.3); `unset NAME` deletes; `vars` lists name/type/size preview.
- Variables are session-scoped, never persisted. `$env.*` is read-only; v1 has no
  way to set process env from the language.

## §8 Command resolution

For a command whose first element is a word-expr (after alias expansion, §6.4):

1. `^` prefix → **external command**, skip all other resolution.
2. First element expands to a string `name` (if it is a bare `$var` value source,
   resolution does not apply — §7.1).
3. Resolution order:
   1. **Builtin** exact match (`help`, `servers`, two-word forms per §10 are handled
      by their primary word).
   2. **Alias** (already expanded before resolution; listed here for the mental model).
   3. **MCP tool**:
      - Qualified `server.tool`: text before the first `.` matches a configured
        server alias → that server; the remainder (may contain dots) is the tool
        name. Triggers lazy connect (§9.3) if needed.
      - Unqualified: `name` matches exactly one tool among **connected** servers'
        cached tool lists → that tool. Ambiguous → `E_UNKNOWN_COMMAND` listing the
        candidates. No auto-connect for unqualified names.
   4. **External command** if `name` contains no `.` that matched above; PATH lookup
      happens at spawn time.
4. Nothing matches → `E_UNKNOWN_COMMAND` with did-you-mean suggestions (Levenshtein
   ≤ 2 over builtins, aliases, and cached tool names).

`call server.tool` (builtin) always forces MCP-tool interpretation and accepts tool
names that are not lexable as bare words (quoted argument).

## §9 MCP integration

### §9.1 Config file

Location and precedence per ADR-006: `--config` > `MCPSH_CONFIG` >
`$XDG_CONFIG_HOME/mcpsh/config.json` (default `~/.config/mcpsh/config.json`).
Missing file → empty server set (REPL still starts; `servers` explains how to
configure). Malformed file → `E_CONFIG`, exit 3.

```json
{
  "$schema": "https://raw.githubusercontent.com/Saber5656/mcpsh/main/config.schema.json",
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${env:GITHUB_TOKEN}" },
      "cwd": "/optional/working/dir",
      "timeouts": { "connectMs": 10000, "callMs": 60000 }
    },
    "remote": {
      "url": "https://mcp.example.com/mcp",
      "headers": { "Authorization": "Bearer ${env:EXAMPLE_TOKEN}" },
      "allowInsecureHttp": false
    },
    "noisy": { "command": "some-server", "enabled": false }
  },
  "limits": { "maxOutputBytes": 10000000 },
  "history": { "file": null, "maxEntries": 10000 }
}
```

### §9.2 Config schema rules (module `src/config`)

| Field | Rules |
|---|---|
| server alias (map key) | `^[A-Za-z_][A-Za-z0-9_-]*$`, unique, must not collide with builtin names → `E_CONFIG` |
| `command`/`args`/`env`/`cwd` | stdio transport. `env` values may use `${env:VAR}` |
| `url`/`headers` | Streamable HTTP transport. Exactly one of `command`/`url` per server |
| `headers` values | may use `${env:VAR}` |
| `allowInsecureHttp` | default false; `http:` URLs allowed only for localhost (`localhost`, `127.0.0.0/8`, `::1`) or when true (startup warning) |
| `enabled` | default true; disabled servers are listed but not connectable |
| `timeouts.connectMs` / `timeouts.callMs` | defaults 10000 / 60000; per-server override |
| `limits.maxOutputBytes` | default 10000000; global, per-server override allowed |
| Unknown keys | `E_CONFIG` with JSON path (strict schema; typo protection) |

`${env:VAR}`: resolved when building the transport (not at load); unset var →
`E_CONFIG` naming the variable and server (but not the would-be value). Every
resolved value is registered with the redactor (§12.5) before first use.

### §9.3 Connection lifecycle (module `src/mcp/manager.ts`)

Per-server FSM:

```
configured ──connect()──► connecting ──initialize ok──► ready
    ▲                        │  │ timeout/error                │ close()/EOF/crash
    │                        ▼  ▼                              ▼
    └────── disconnect() ── failed ◄──────────────────────── closed
```

- States: `configured`, `connecting`, `ready`, `failed` (with stored error),
  `closed`. `servers` shows the state, negotiated protocol version, and tool count.
- **Lazy connect**: first use of `server.tool`, `tools server`, `resources server`,
  `prompts server`, or explicit `connect server` transitions configured→connecting.
- `connect` is idempotent: on `ready` it is a no-op; on `failed`/`closed` it retries
  (single attempt per invocation; no automatic retry loops — §12.7).
- stdio: subprocess spawned with env allowlist (§12.4), `cwd` from config; stderr
  captured into a 100-line ring buffer, shown on connect failure and with
  `--verbose`. Shutdown: SDK `close()`, then SIGTERM, SIGKILL after 3 s.
- HTTP: Streamable HTTP transport with configured headers. TLS rules per §9.2.
- Connect timeout `connectMs` → state `failed`, `E_CONNECT`.
- On mcpsh exit: all connections closed gracefully (best effort, 3 s cap total).

### §9.4 Namespacing and tool registry (module `src/mcp/tools.ts`)

- Canonical form `server.tool` (§8). Tool lists fetched on `ready`, cached.
- Cache invalidation: `notifications/tools/list_changed` → drop cache, refetch
  lazily; `tools --refresh` forces refetch.
- Tool metadata kept: name, title, description, inputSchema, outputSchema,
  annotations (readOnlyHint/destructiveHint shown by `describe`).

### §9.5 Tool invocation (module `src/mcp/tools.ts`)

- Arguments object built per §9.6; validated minimally client-side: required
  properties present, primitive type mismatches rejected (`E_ARGS`) — full JSON
  Schema validation stays server-side.
- Call with per-server `callMs` timeout and AbortSignal; Ctrl-C sends
  `notifications/cancelled` (SDK) and raises `E_CANCELLED`.
- Result size cap per §5.3.

### §9.6 Flag → argument mapping (module `src/mcp/args.ts`)

- Reserved mcpsh flags (never forwarded): `--json <word>`, `--raw-result`.
  If a tool schema declares a property named `json` or `raw-result`, it must be set
  via `--json '{...}'`.
- `--name` where schema property `name` is boolean → `true`; explicit false via
  `--name=false`. Boolean flags never consume the next word.
- `--name <word>` / `--name=<word>` for non-boolean properties; coercion by schema
  type: string → as-is; number/integer → numeric parse (`E_ARGS` on failure);
  array/object → JSON parse (`E_ARGS` with excerpt on failure). Schema-less
  property or no schema → string.
- Word-exprs expand before coercion; a bare `$var` flag value passes the value
  unconverted (already typed).
- `--json '<object>'` merges as the base object; explicit flags override individual
  keys **only if** the key is absent in the JSON; duplicate specification (same key
  in both, or the same flag twice) → `E_ARGS`.
- Positional words (non-flag) in a tool command: `E_ARGS` ("tools take --flag
  arguments"), except the `$in`-reference rule (§7.2) which may appear anywhere a
  word-expr is allowed.
- Unknown flag (not in schema, not reserved) → `E_ARGS` with the list of known
  property names.

### §9.7 Result unwrapping

For a successful `tools/call` result:

1. `structuredContent` present → it becomes the pipeline value (validated as Value).
2. Else exactly one content block of type `text` → its text as a string value.
3. Else → envelope record:
   `{ "content": [<block records>], "isError": false }` where block records are
   `{type:"text", text}`, `{type:"image"|"audio", mimeType, data, bytes}` (blob
   convention §5.1), `{type:"resource_link", uri, name?, description?}`,
   `{type:"resource", resource: {uri, mimeType?, text? | data?}}`.
4. `isError: true` → `E_TOOL_FAILED`; message = concatenated text blocks (sanitized,
   truncated to 500 chars); the envelope is attached to the error for `--verbose`.
5. `--raw-result` on the invocation (or `call`) skips 1–2 and always yields the
   envelope record with `isError` included (tool errors do **not** raise when
   `--raw-result` is set — enables inspecting failures in pipelines).

### §9.8 Resources and prompts (modules `src/mcp/resources.ts`, `src/mcp/prompts.ts`)

- `resources [server]` → list of records `{server, uri, name?, title?, mimeType?}`.
  Without `server`: union across **connected** servers.
- `resource-read <uri> [--server <alias>]`: server chosen by `--server`, else the
  unique connected server listing that URI, else `E_ARGS` (ambiguity or unknown).
  Text contents → string; blob contents → blob record (§5.1); multi-content
  results → list. Size cap applies.
- `prompts [server]` → list of `{server, name, title?, description?, arguments: [...]}`.
- `prompt-get <server.name> [--arg key=value ...]` → record
  `{description?, messages: [{role, content: <block record>}]}`.

### §9.9 Server-initiated interactions (module `src/mcp/interactions.ts`)

| Request/notification | REPL behavior | One-shot behavior |
|---|---|---|
| `elicitation/create` | Render `<server> is asking: <message>`; prompt per top-level property of `requestedSchema` (string/number/boolean/enum only); Ctrl-C or empty required → decline. Nested schemas → auto-decline with a printed notice | Auto-decline |
| `sampling/createMessage` | Not declared; protocol-level method-not-found (ADR-004) | Same |
| `notifications/progress` | Progress line/spinner via render layer (§11.5) | Ignored (logged at debug) |
| `notifications/message` | Mapped to logger: server `debug/info/notice/warning/error/...` → mcpsh `debug/info/info/warn/error`; shown per `--log-level` | Same |
| `ping` | SDK auto-response | Same |
| `roots/list` | Capability not declared | Same |

## §10 Builtins reference

Conventions: `input → output` describes pipeline value flow. `inputPolicy` per §7.2.
Builtins with a second command word (e.g. `resource-read`) are single hyphenated
names, keeping resolution single-token. All builtins reject unknown flags (`E_ARGS`).

| Builtin | Signature | Input → output | Notes |
|---|---|---|---|
| `help` | `help [name]` | ignores → string | general help or per-command usage |
| `version` | `version` | ignores → string | |
| `exit` | `exit` | ignores → (terminates) | REPL only; `E_ARGS` in one-shot |
| `servers` | `servers` | ignores → list of records | alias, transport, state, protocol, toolCount |
| `connect` | `connect <alias>` | ignores → record (server row) | idempotent (§9.3) |
| `disconnect` | `disconnect <alias>` | ignores → null | |
| `tools` | `tools [alias] [--refresh]` | ignores → list of records | `{server, name, title?, description?}` |
| `describe` | `describe <server.tool>` | ignores → record | schema, annotations |
| `call` | `call <name> [flags...]` | per §7.2 → per §9.7 | forces MCP; `<name>` may be quoted |
| `resources` | `resources [alias]` | ignores → list | §9.8 |
| `resource-read` | `resource-read <uri> [--server a]` | ignores → value | §9.8 |
| `prompts` | `prompts [alias]` | ignores → list | §9.8 |
| `prompt-get` | `prompt-get <server.name> [--arg k=v ...]` | ignores → record | §9.8 |
| `get` | `get <path>` | requires → value | on list without leading index: map over elements; MISSING → `E_TYPE` naming path |
| `select` | `select <p1> [p2...]` | requires → record/list | record or list-of-records; missing key → omitted |
| `where` | `where <path> <op> <literal>` | requires list → list | ops: `== != > >= < <= contains startswith`; MISSING path → element excluded |
| `first` | `first [n=1]` | requires list → value/list | n=1 → element; n>1 → list |
| `last` | `last [n=1]` | requires list → value/list | |
| `length` | `length` | requires → number | list length, record key count, string char count |
| `flatten` | `flatten` | requires list → list | one level |
| `sort-by` | `sort-by <path> [--desc]` | requires list → list | numbers < strings; stable |
| `reverse` | `reverse` | requires list → list | |
| `from` | `from json\|jsonl\|lines` | requires string → value | `lines` splits, drops trailing empty |
| `to` | `to json [--pretty]\|jsonl\|text` | requires → string | `text`: strings as-is, else compact JSON |
| `lines` | `lines` | requires string → list | shorthand for `from lines` |
| `echo` | `echo [words...]` | accepts → value | 0 args: passthrough input; 1 arg: that value; n args: list |
| `set` | `set NAME [word]` | accepts → value | sink form stores input and passes it through; word form stores and yields it |
| `unset` | `unset NAME` | ignores → null | |
| `vars` | `vars` | ignores → list of records | name, type, approx size |
| `alias` | `alias [NAME words...]` | ignores → list/null | no args: list aliases |
| `unalias` | `unalias NAME` | ignores → null | |
| `history` | `history [--max n=50]` | ignores → list of strings | REPL only |
| `save` | `save <path> [--raw] [--force]` | requires → null | default: pretty JSON + `\n`; `--raw` requires string, writes verbatim; existing file without `--force` → `E_ARGS` |
| `open` | `open <path>` | ignores → string | UTF-8 only (invalid → `E_TYPE`); size cap |
| `table` | `table` | requires list of records → string | §11.5 |

## §11 CLI and REPL surfaces

### §11.1 CLI flags (module `src/cli.ts`)

```
mcpsh [options]
  -c, --command <program>   run program and exit
  --config <path>           config file (ADR-006)
  --output auto|json|text   final-value rendering in one-shot mode (default auto)
  --log-level error|warn|info|debug|trace   (default warn)
  --verbose                 shorthand: --log-level debug + surface server stderr
  --no-color                disable colors (also: NO_COLOR env, non-TTY default)
  --version, --help
```

Unknown flag → usage message on stderr, exit 2. `-c` with empty string → exit 2.

### §11.2 Mode selection

| stdin | `-c` given | Mode |
|---|---|---|
| TTY | no | REPL |
| TTY | yes | one-shot: run `-c` program |
| not a TTY | no | one-shot: read program from stdin (entire input, then execute) |
| not a TTY | yes | one-shot: run `-c` program (stdin available to stages? **No** — v1 reserves stdin; document: external stages and `$in` never read mcpsh's own stdin in `-c` mode) |

### §11.3 Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | runtime error (`E_CONNECT`, `E_TOOL_FAILED`, `E_EXTERNAL_FAILED`, `E_TYPE`, `E_LIMIT`, `E_UNBOUND_INPUT`, `E_UNKNOWN_COMMAND`, `E_ARGS`, `E_PROTOCOL`) |
| 2 | usage/parse error (`E_PARSE`, bad CLI flags) |
| 3 | configuration error (`E_CONFIG`) |
| 130 | interrupted (`E_CANCELLED` / SIGINT) |

### §11.4 Output rules (one-shot)

Final pipeline value → stdout:

| `--output` | Behavior |
|---|---|
| `auto` (default) | null → nothing; string → raw + `\n`; other → compact JSON + `\n` |
| `json` | JSON always (strings quoted); compact when stdout is not a TTY, pretty when TTY |
| `text` | like `to text` |

Errors → stderr, single line `mcpsh: <CODE>: <message>` (+ hint line when
applicable). All data-derived text passes sanitization (§12.6). In the REPL, every
pipeline's final value renders pretty (§11.5) unless null.

### §11.5 REPL rendering (module `src/render/render.ts`)

- Pretty renderer: 2-space-indented JSON with `util.styleText` colors (keys cyan,
  strings green, numbers yellow, null dim). Top-level strings print raw (sanitized).
- `table`: list-of-records → aligned columns; column set = union of keys (first 8,
  rest elided with notice); cell = compact JSON (strings raw); truncate to terminal
  width; header dim. Non-list-of-records input → `E_TYPE`.
- Progress: single status line (`⠋ github.search_issues … 3.2s [45%] message`)
  rewritten in place, TTY only, cleared on completion; driven by progress
  notifications and a 100 ms ticker for calls > 500 ms.
- Durations: calls slower than 2 s print a dim `(took 3.4s)` line after the value.

### §11.6 REPL behaviors (module `src/repl/loop.ts`)

- Prompt `mcpsh> `. Single-line input (no continuation in v1, ADR-003).
- History: `$XDG_STATE_HOME/mcpsh/history` (default `~/.local/state/mcpsh/history`);
  dir created `0700`, file `0600`; `history.maxEntries` (default 10000) enforced on
  save; leading-space input is not persisted (secret hygiene, §12.5); duplicates of
  the immediately previous entry collapsed.
- Ctrl-C: with a running pipeline → abort it (§9.5, §7.4), print `^C`, keep session;
  at empty prompt → print hint "(Ctrl-D to exit)"; Ctrl-D → exit 0.
- rc file: `$XDG_CONFIG_HOME/mcpsh/init.mcpsh`, if present, executed line-by-line
  as programs before the first prompt; each line's error prints and execution
  continues (rc errors never abort startup); rc lines are not added to history.
- REPL start does **not** connect servers (lazy, §9.3).

### §11.7 Completion (module `src/repl/complete.ts`)

Completion is side-effect-free: it never connects servers or spawns processes.

| Position | Candidates |
|---|---|
| first word | builtins, aliases, `alias.`-style server prefixes (`github.`), cached tool names (qualified + unambiguous unqualified), variable refs `$name` |
| after `server.` | cached tool names of that server (empty if not connected — a dim notice `(connect github to complete tools)` is shown once) |
| flag position of resolved tool/builtin | `--property` names from inputSchema / builtin metadata |
| argument of `open`, `save`, `--config` | filesystem paths |
| after `$` | variable names, `in`, `env.` |

## §12 Security model

### §12.1 Assets

User credentials (env-referenced tokens/headers), local filesystem contents, the
user's terminal (escape-sequence integrity), availability of the mcpsh process.

### §12.2 Trust boundaries

| # | Boundary | Trust stance |
|---|---|---|
| B1 | User input (REPL/`-c`/rc) | Trusted (it is the user's shell) |
| B2 | Config file | Trusted content, but only loaded from explicit locations (ADR-006); env refs keep secrets out of it |
| B3 | MCP servers (processes/URLs the user configured) | Semi-trusted: user chose them, but their **output is untrusted data** and their behavior may be buggy/compromised |
| B4 | External commands | Trusted (user-invoked, like any shell); their output is untrusted data when it re-enters pipelines |
| B5 | Network (HTTP transport) | Untrusted; TLS required off-localhost |
| B6 | Local files written/read (`save`/`open`, history) | Paths are user-chosen; mcpsh adds no implicit file access |

### §12.3 Threat table (normative — each mitigation maps to an issue)

| ID | Threat | Mitigation | Spec | Issue |
|---|---|---|---|---|
| T1 | Malicious server output injects terminal escapes (clobber scrollback, fake prompts, clipboard OSC52) | Sanitize all data-derived strings at render time: strip C0 (except `\n\t`), C1, and ESC (replaced with `␛`) | §12.6 | 06, 29, 31 |
| T2 | Server floods output → memory exhaustion | `limits.maxOutputBytes` on every capture point | §5.3 | 03, 12, 21, 31 |
| T3 | Tool output containing shell metacharacters executes commands | No shell spawns; single-argument expansion | ADR-005 | 21, 31 |
| T4 | Secrets leak into logs/errors | Redactor: registered env-ref values + key-pattern matching (`token/secret/key/password/authorization/bearer/api[-_]?key`, case-insensitive) replaced with `[redacted]` in all log lines and error messages | §12.5 | 05, 31 |
| T5 | Secrets leak into history file | `0600` perms; leading-space skip; documented | §11.6 | 27, 31 |
| T6 | stdio server inherits sensitive env vars | Allowlist (§12.4); opt-out per server `inheritEnv: true` | §12.4 | 09, 31 |
| T7 | MITM of remote server | HTTPS required off-localhost; `allowInsecureHttp` explicit + warning | §9.2 | 10 |
| T8 | Malicious repo config auto-executes servers | No project-local config discovery | ADR-006 | 07 |
| T9 | Hung server/tool blocks shell or CI forever | connect/call timeouts; Ctrl-C cancellation; no auto-retry loops | §9.3, §9.5 | 09, 12, 30 |
| T10 | Elicitation social-engineering (server impersonates mcpsh UI) | Elicitation prompts always prefixed with the server alias; content sanitized (T1); non-interactive auto-decline | §9.9 | 16 |
| T11 | Completion triggers side effects (connect/spawn) on TAB | Completion reads caches only | §11.7 | 28 |
| T12 | Supply-chain compromise of dependencies | 2-dependency policy (§4.3); lockfile committed; CI audit + CodeQL + dependency review; npm provenance at publish | §15 | 01, 02, 34 |
| T13 | Accidental overwrite of user files | `save` refuses existing files without `--force` | §10 | 25 |
| T14 | Malformed server JSON/protocol data crashes shell | SDK validation + `checkValue` normalization; protocol errors → `E_PROTOCOL`, session survives in REPL | §5.2 | 03, 12 |

### §12.4 stdio server environment allowlist

Default env for spawned MCP servers: `PATH`, `HOME`, `USER`, `LOGNAME`, `SHELL`,
`TERM`, `LANG`, `TMPDIR`, all `LC_*` — plus the config `env` map (with resolved
`${env:VAR}` refs). Nothing else from the parent env. Per-server
`"inheritEnv": true` opts into full inheritance (config-time warning log).
Rationale: server processes should not silently receive `AWS_*`, `GITHUB_TOKEN`,
etc. unless the user passes them explicitly.

### §12.5 Secret handling rules

- Secrets live in the process environment, never in config file text (env refs,
  §9.2) and never in mcpsh-written files.
- Every resolved env-ref value is registered with the redactor before first use.
- Log lines and error messages pass redaction (T4). Payload logging (tool args &
  results) happens only at `trace` level, with a warning banner on first use.
- History hygiene per §11.6.

### §12.6 Output sanitization

`sanitize(text)` (module `src/render/sanitize.ts`): remove C0 controls except
`\n` and `\t`, remove C1 (U+0080–U+009F), replace ESC (U+001B) with `␛` (U+241B).
Applied to: rendered values, error messages, elicitation prompts, progress messages,
log lines containing server-derived text, and table cells. Not applied to: bytes
written to external stdin, `save --raw` output, `--output` stdout in one-shot mode
when stdout is not a TTY (data fidelity for pipes) — **except** `--output auto`
pretty rendering on a TTY, which sanitizes.

### §12.7 Availability rules

No unbounded retry loops anywhere: connect attempts are single-shot per trigger;
timeouts everywhere (§9.3, §9.5); Ctrl-C always regains the prompt within 3 s
(child-kill grace).

### §12.8 Residual risks (documented, not mitigated in v1)

- MCP stdio servers run with full user privileges (no OS sandboxing) — inherent to
  the MCP local model; users choose their servers (B3). v2 idea: optional sandbox
  wrappers.
- PATH hygiene for external commands is the user's responsibility (as in any shell).
- A malicious server can return misleading *content* (phishing text); mcpsh
  guarantees rendering integrity (T1), not content truthfulness.

## §13 Error taxonomy and exit codes

Error class hierarchy (module `src/errors.ts`): `McpshError { code, message,
hint?, cause? }`.

| Code | Class | Typical source | Exit |
|---|---|---|---|
| `E_PARSE` | ParseError (line, col) | lexer/parser, alias depth | 2 |
| `E_CONFIG` | ConfigError (jsonPath?) | schema violation, env ref unset, file unreadable | 3 |
| `E_CONNECT` | ConnectError (server) | spawn failure, timeout, HTTP/TLS errors | 1 |
| `E_PROTOCOL` | ProtocolError (server) | SDK/JSON-RPC failures after connect | 1 |
| `E_UNKNOWN_COMMAND` | UnknownCommandError (name, suggestions) | resolution §8 | 1 |
| `E_ARGS` | ArgsError | flag mapping §9.6, builtin misuse | 1 |
| `E_TYPE` | PipelineTypeError | input policy, path errors, `from json` failure | 1 |
| `E_UNBOUND_INPUT` | UnboundInputError | §7.2 | 1 |
| `E_TOOL_FAILED` | ToolFailedError (server, tool, envelope) | `isError` results | 1 |
| `E_EXTERNAL_FAILED` | ExternalFailedError (cmd, exitCode/signal) | §7.4 | 1 |
| `E_LIMIT` | LimitError (limit, stage) | §5.3 | 1 |
| `E_CANCELLED` | CancelledError | Ctrl-C | 130 |

Formatting: one line `mcpsh: E_CODE: message` (+ optional `hint: ...` second line);
`--verbose` adds cause chain and (for tool/connect errors) the captured
envelope/stderr tail. All formatted text passes redaction and sanitization.

## §14 Testing and validation strategy

| Layer | Approach | Issues |
|---|---|---|
| values/errors/log/sanitize | pure unit tests (vitest), property-style cases for sanitize/redact | 03–06 |
| lexer/parser | table-driven token/AST snapshots incl. error positions | 17, 18 |
| evaluator/builtins | unit tests with a mock `CommandHost`; no MCP dependency | 19–25 |
| config | fixture files: valid, unknown-key, bad-alias, env-ref-unset, insecure-http | 07 |
| MCP client layer | integration against the fixture server (issue 08) over stdio **and** Streamable HTTP | 09–16 |
| CLI e2e | spawn built `dist/cli.js` with `-c` programs + fixture config; assert stdout/stderr/exit codes; SIGINT test | 30 |
| REPL | unit-level: injected readline interface driving `loop.ts`; one guarded pty smoke test (skipped when node-pty unavailable) | 27, 30 |
| Security | adversarial suite per threat table (T1–T6, T11, T13) | 31 |
| Static | `tsc --noEmit` strict, eslint, prettier check | 01, 02 |
| CI | GitHub Actions matrix {ubuntu, macos} × {node 22, 24}; CodeQL; dependency review; npm audit report | 02 |

Definition of validated: every issue's Validation section passes locally and in CI.
Aspirational (non-gating) coverage target: ≥80% lines in `src/lang`, `src/values`,
`src/mcp`.

## §15 Distribution and release

- Package: `"name": "mcpsh"`, `"private": true` during v1 development (ADR-008);
  `"bin": {"mcpsh": "dist/cli.js"}`; `files: ["dist"]`; license **MIT** (owner may
  veto before publish); `engines.node >= 22.12`.
- Build: `tsc` to `dist/` (no bundler in v1). `npm pack` smoke-tested in CI.
- Release flow (issue 34): manual `workflow_dispatch` publish workflow with npm
  provenance; hard-fails while `private: true`; version + CHANGELOG (keep-a-changelog)
  updated by the release PR. Publishing itself is a human gate (merge ≠ release);
  npm token is configured by the human owner, never by agents.
- Versioning: 0.x during v1 development; v1 completion (ISSUE_PLAN.md) tags 0.9.x
  RC; 1.0.0 after the naming decision (ADR-008) and owner sign-off.

## §16 Known unknowns

Tracked in ISSUE_PLAN.md §Known unknowns; headline items: final npm/public name
(ADR-008), SDK v2 / spec 2026-07-28 migration timing (ADR-007), REPL pty testing
reliability in CI, Streamable HTTP server behavior variance across real-world
servers, Windows quirks (path/env/spawn) if/when Windows becomes a target.
