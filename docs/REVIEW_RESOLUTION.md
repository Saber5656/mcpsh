# Review resolution addendum

- Repository: `Saber5656/mcpsh`
- Pull request: #1
- Original PR head before this resolution addendum: `996339adfc8d0ffa87c6555713c4f2c0bc066ac0`
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates that evidence and requires a fresh review.
- Scope: each exact review thread below has a normative design contract and a focused verification gate.
- This is design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTN394M6OlJHX` — Enforce an absolute tool-call timeout

**Normative resolution**: `callMs`/progress resets never extend the hard absolute deadline. `resetTimeoutOnProgress` may control an idle timer only within the configured maximum, and the request is terminated at that maximum.

**Focused verification gate**: Emit progress indefinitely with reset enabled and assert termination at the absolute cap; test tiny caps, no progress, and normal completion.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 2. Thread `PRRT_kwDOTN394M6OlJHc` — Redact full bearer and JSON-style secrets

**Normative resolution**: The redactor consumes complete bearer credentials (`Authorization: Bearer <value>`) and quoted/unquoted JSON-style key values such as `api_key`, with multiline and truncation-safe handling.

**Focused verification gate**: Pass headers, JSON, form, multiline, and mixed-case secrets through verbose/trace logging; assert the complete value is replaced and non-secret text remains.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 3. Thread `PRRT_kwDOTN394M6OlJHf` — Keep stdio shutdown within the total cap

**Normative resolution**: `closeAll()` owns one shared absolute shutdown deadline of 3 seconds for all stdio servers; graceful close, SIGTERM, and SIGKILL consume the remaining budget rather than restarting a 3-second wait per server.

**Focused verification gate**: Use one and multiple non-exiting servers, measure total close time, and assert escalation plus final cleanup stay within the shared cap.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 4. Thread `PRRT_kwDOTN394M6OlJHm` — Check tool errors before normal unwrapping

**Normative resolution**: Pipeline normalization checks `isError === true` before inspecting structured/text content and emits `E_TOOL_FAILED`; an error result is never promoted to a successful value because it also contains payload data.

**Focused verification gate**: Return error results with structured content, text, both, and neither; assert all map to the error path before unwrapping.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 5. Thread `PRRT_kwDOTN394M6OlJHp` — Enforce the open cap while reading

**Normative resolution**: File/tool reads retain the stat fast path but stream or chunk data and stop at `maxOutputBytes`; special files, growing files, and `/proc` cannot bypass the limit or block indefinitely.

**Focused verification gate**: Read `/dev/zero`, a FIFO, a growing file, an oversized regular file, and an exact-limit file; assert bounded bytes, timeout/limit error, and no unbounded memory.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 6. Thread `PRRT_kwDOTN394M6OlJHv` — Treat mid-word # as literal

**Normative resolution**: The lexer treats `#` as a comment delimiter only at token start (after the defined boundary); once a WORD has begun, `a#b` and URI fragments remain literal.

**Focused verification gate**: Tokenize `#comment`, `a#b`, `https://x/#frag`, quoted hashes, and whitespace variants; assert exact tokens/comments.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 7. Thread `PRRT_kwDOTN394M6OlJHw` — Do not execute unresolved bare dotted names

**Normative resolution**: An unresolved dotted bare name returns `E_UNKNOWN_COMMAND`; only the explicit external-execution escape (such as `^foo.bar`) can request PATH execution, subject to the normal policy.

**Focused verification gate**: Resolve configured aliases, unknown dotted names, explicit escapes, and ordinary commands; assert no implicit external process for a typo.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 8. Thread `PRRT_kwDOTN394M6OlNjM` — Tag the fenced examples.

**Normative resolution**: Every Markdown fenced block in the issue is labeled with the correct language (`bash`, `json`, `ts`, `text`, or equivalent), including repeated/later snippets.

**Focused verification gate**: Run markdownlint/MD040 over the document and assert zero unlabeled fences without suppressing the rule.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 9. Thread `PRRT_kwDOTN394M6OlNjP` — Clarify record key ordering.

**Normative resolution**: The record contract uses an ordered entry list/map for serialized order, or explicitly defines a canonical sort; plain JavaScript object enumeration is not used as insertion-order evidence for integer-like keys.

**Focused verification gate**: Serialize integer-like and ordinary keys, parse/re-render, and assert the documented canonical order is stable across runtimes.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 10. Thread `PRRT_kwDOTN394M6OlNjT` — Add `inheritEnv` to the config schema.

**Normative resolution**: `inheritEnv` is a typed schema field with its default and security semantics, matching §12.4/T6 and the loader's unknown-key validation.

**Focused verification gate**: Validate configs with omitted, true, false, and invalid `inheritEnv`; assert child environment behavior and unknown-key failures match the schema.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 11. Thread `PRRT_kwDOTN394M6OlNji` — Label the dependency graph fence.

**Normative resolution**: The dependency graph fence is labeled `text` (or an equivalent non-code language) so markdownlint and readers treat it as a diagram.

**Focused verification gate**: Run markdownlint on the issue and assert MD040 is clear for this and every fence.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 12. Thread `PRRT_kwDOTN394M6OlNjq` — Add issue 06 to the dependency chain.

**Normative resolution**: The Dependencies section explicitly lists issues 01, 04, and 06, because `fromJson` relies on the sanitizer from issue 06.

**Focused verification gate**: Run dependency/reference consistency checks and assert the issue cannot be scheduled without its direct sanitizer prerequisite.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 13. Thread `PRRT_kwDOTN394M6OlNjt` — Redact the full `Authorization: Bearer ...` value.

**Normative resolution**: Header redaction special-cases bearer authentication or consumes the complete header value after `Bearer`, replacing the scheme and credential together.

**Focused verification gate**: Test bearer headers with spaces, punctuation, empty values, mixed case, and adjacent fields; assert no credential suffix remains.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 14. Thread `PRRT_kwDOTN394M6OlNj1` — Make ESC a documented exception to the C0 strip.

**Normative resolution**: The sanitizer order is explicit: preserve/recognize ESC long enough to replace it with the safe `␛` marker (or equivalent), while removing other C0 controls except the separately retained newline/tab rules.

**Focused verification gate**: Test ESC, ANSI sequences, all C0 controls, newline/tab, and C1 controls; assert deterministic safe output and no contradictory rule.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 15. Thread `PRRT_kwDOTN394M6OlNj9` — Pin down the schema-drift check.

**Normative resolution**: CI runs one named schema-drift command that regenerates `config.schema.json` and fails on a diff, alongside the documented build/test commands; the generated file is committed and cannot silently drift.

**Focused verification gate**: Modify the schema generator and fixture, run the exact CI command, and assert drift fails while a regenerated identical file passes.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 16. Thread `PRRT_kwDOTN394M6OlNkH` — Reconcile the shutdown budget.

**Normative resolution**: The shutdown contract defines one total deadline and derives per-server graceful/escalation waits from remaining time; the ≤3s `closeAll()` promise and per-server behavior are no longer contradictory.

**Focused verification gate**: Test one, many, graceful, and non-cooperative servers and assert total elapsed time never exceeds the canonical deadline.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 17. Thread `PRRT_kwDOTN394M6OlNkM` — Make `listTools()` cache-first.

**Normative resolution**: `listTools(alias)` returns a valid cached list on a cache hit; only `refresh: true` or an absent/invalid cache fetches from the server, then atomically replaces the cache.

**Focused verification gate**: Call repeatedly with/without refresh, simulate stale/missing/corrupt cache and server failure, and assert fetch count and fallback behavior.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 18. Thread `PRRT_kwDOTN394M6OlNkV` — Resolve the `string` coercion rule.

**Normative resolution**: The coercion table follows DESIGN §9.6: declared `string` properties remain as-is, while schema-less/no-schema values use the explicit string conversion rule; the issue text and examples use one precedence.

**Focused verification gate**: Exercise declared string, schema-less, null, numeric, boolean, array, and object values and assert exact output/error behavior.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 19. Thread `PRRT_kwDOTN394M6OlNka` — Resolve the `#` tokenization rule mismatch.

**Normative resolution**: The lexer grammar removes `#` from the WORD exclusion set for already-started words and keeps comment detection at token start, making the grammar and examples identical.

**Focused verification gate**: Run the lexer conformance matrix for inline/start/quoted hashes and assert no tokenization regression.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 20. Thread `PRRT_kwDOTN394M6OlNkj` — Keep the `servers` record contract identical to §10.

**Normative resolution**: The `servers` record uses the exact §10 fields and optionality (`target`, optional `protocol`, `tools?`) and no divergent shape is introduced in this issue.

**Focused verification gate**: Compare schema/types/parser fixtures to §10 and run round-trip validation for minimal and fully populated records.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 21. Thread `PRRT_kwDOTN394M6OlNkn` — Don't add `--raw-result` here unless §10 changes too.

**Normative resolution**: The `call` command remains `call <name> [flags...]` as defined by §10; `--raw-result` is removed from this issue or is first added to the canonical command contract and all dependent tests.

**Focused verification gate**: Compare CLI help, parser, docs, and completion behavior; assert unsupported `--raw-result` is rejected without an undocumented escape hatch.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 22. Thread `PRRT_kwDOTN394M6OlNkt` — Preserve the scalar return for `first`/`last` when `n=1`.

**Normative resolution**: `first`/`last` return one scalar for default `n=1` and a list only for `n>1`, with the exact §10 behavior documented for empty results.

**Focused verification gate**: Test n omitted, 1, >1, zero matches, and multiple matches; assert return shape and error behavior.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 23. Thread `PRRT_kwDOTN394M6OlNkw` — Use a unique, exclusive temp file for `save`.

**Normative resolution**: `save` creates a cryptographically/randomly unique temp file in the target directory with exclusive creation and restrictive mode, fsyncs/renames atomically, and cleans abandoned temps by policy.

**Focused verification gate**: Run concurrent saves and collision injection under permissive umask; assert no clobber, partial target, or predictable shared temp file.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 24. Thread `PRRT_kwDOTN394M6OlNk2` — Resolve the completer return-shape mismatch.

**Normative resolution**: The pure completer returns one documented richer completion object containing `lastMiss`, and the readline wrapper adapts it to its tuple; no caller is promised two incompatible shapes.

**Focused verification gate**: Test hit, miss, prefix, and last-miss cases through both pure helper and readline adapter; assert shape compatibility.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 25. Thread `PRRT_kwDOTN394M6OlNk5` — Preserve trusted `table` output from the top-level string sanitizer.

**Normative resolution**: `table` returns a typed rendered value or uses a dedicated trusted-render path; the top-level sanitizer distinguishes trusted table markup from untrusted raw strings and never creates an arbitrary bypass.

**Focused verification gate**: Render ANSI table styling, untrusted cells, terminal output, and JSON output; assert styling is preserved only in the trusted path and cell content is escaped.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 26. Thread `PRRT_kwDOTN394M6OlNk8` — Make the RSS guard capture peak `mcpsh` memory.

**Normative resolution**: The RSS guard samples/records peak process RSS over the entire command/server lifetime, not only a final snapshot, and applies the documented threshold to that peak.

**Focused verification gate**: Allocate then release a large buffer before exit and assert the recorded peak still exceeds the threshold; test sampling interval and normal low-memory runs.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 27. Thread `PRRT_kwDOTN394M6OlNk-` — Add the allowed localhost HTTP case.

**Normative resolution**: The security checklist includes both rejection of non-local plain HTTP and an explicit allowed localhost case (`localhost`, loopback IPv4, `::1`) under the documented `allowInsecureHttp` policy and warning.

**Focused verification gate**: Test HTTP localhost variants, non-loopback HTTP, HTTPS, DNS aliases, and default/override settings; assert exact allow/reject/warning results.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.