# ADR-008: npm name collision — develop as private `mcpsh`, defer public name to release gate

- Status: Accepted (design decision, 2026-07-06); final name is a **human decision**
- Deciders: design agent; escalated to repository owner before first publish

## Context

`mcpsh` on npm is taken by an unrelated minimal MCP client
(`mcpsh@0.2.0`, morinokami, published 2025-03; see docs/research/prior-art.md).
The GitHub repository `Saber5656/mcpsh` and the binary name `mcpsh` are not blocked,
but the npm distribution name is.

## Decision

For all v1 development:

- `package.json` uses `"name": "mcpsh"` **with `"private": true`**, preventing any
  accidental publish under a colliding or unintended name.
- The binary name remains `mcpsh` (`"bin": { "mcpsh": "dist/cli.js" }`).
- The release-packaging issue (34) includes a hard gate: the publish workflow fails
  while `private: true`, and flipping it requires choosing the public name.

Options for the human release decision (not decided here):

| Option | Example | Trade-off |
|---|---|---|
| A. Scoped package, same binary | `@saber5656/mcpsh` | Zero rename cost; scoped names are less discoverable; binary still collides conceptually with the existing `mcpsh` package's `npx` usage |
| B. New unscoped name + binary | e.g. `mcpipe`, `mshell`, ... | Full discoverability; requires repo/binary rename and README/docs sweep |
| C. Attempt npm name dispute/transfer | `mcpsh` | Slow, uncertain; existing package is recent and legitimately used |

## Consequences

- v1 implementation is completely insulated from the naming outcome; only issue 34
  and README install instructions depend on it.
- The decision must be made before the first `npm publish` (which is itself a human
  gate per repository policy: merge ≠ release).
