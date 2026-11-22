---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: cmd_show in reasons/cli.py

## `cmd_show` — Explanation

### Purpose

`cmd_show` is the CLI handler for `reasons show <node_id>`. It fetches a single belief node from the API layer (`api.show_node`), then formats and prints its full record — core identity, provenance, justifications, metadata, timestamps, tags, sources, and dependents — to stdout. It exists to give users a "full inspection" view of one belief, complementing the summary view (`cmd_status`) and the reasoning view (`cmd_explain`).

### Contract

- **Preconditions**
  - `args` is the `argparse.Namespace` produced by the `show` subparser (i.e. it has `node_id`; optionally `visible_to`, `db`, `pg`, `domain_id`, `project_id`).
  - `_backend_kwargs(args)` must resolve cleanly (i.e. if `--pg`/`REASONS_PG_CONNINFO` is set, `--domain-id`/`REASONS_DOMAIN_ID` must also be present).
  - `api.show_node` must return a dict that includes **at minimum** the keys `id`, `text`, `truth_value`, `source`, `source_hash`, `metadata`, `justifications`, `dependents` (these are accessed via `node[...]`, not `.get`). `metadata` must be a `dict` (not `None`), and `justifications`/`dependents` must be lists.
- **Postconditions**
  - On success: writes a formatted multi-line report to **stdout** and returns `None` (process exits 0).
  - On `KeyError`/`PermissionError` from `api.show_node`: writes an error line to **stderr** and exits 1.
- **Invariants**
  - The `try/except` only wraps the `api.show_node(...)` call. Any `KeyError` raised while *formatting* the returned dict (e.g. missing `source`) escapes as an uncaught traceback.

### Parameters

- `args` (`argparse.Namespace`)
  - `node_id` (`str`, required) — the belief ID to fetch.
  - `visible_to` (`str`, optional) — comma-separated tag set; parsed by `_parse_visible_to`. If omitted, visibility is unrestricted (argument is `None`).
  - `db` (`str`, `_backend_kwargs` uses it when not on PG) — path to the SQLite database.
  - `pg` (`str`, optional) and `domain_id`/`project_id` — passed through `_backend_kwargs`; a `--pg` without a domain id triggers `sys.exit(1)` *before* the try block finishes evaluating.

Edge cases:
- `_parse_visible_to` returns `None` when `visible_to` is missing/empty; this is forwarded as-is to `api.show_node`, which presumably treats `None` as "no filtering."
- Missing `node_id` is a parser-level error (argparse rejects it before this function runs).

### Return Value

Returns `None` on success (implicit end-of-function). It never returns a value; the caller (`main`) just returns the process exit code 0. On error paths, it calls `sys.exit(1)` — the function itself never returns a non-success code.

### Algorithm

1. **Parse visibility**: `visible_to = _parse_visible_to(args)`.
2. **Fetch node** inside a `try`:
   - Calls `api.show_node(args.node_id, visible_to=visible_to, **_backend_kwargs(args))`.
   - `**_backend_kwargs(args)` is evaluated *as part of the call expression*, so a missing PG domain id would call `sys.exit(1)` there; `SystemExit` is not caught by the excepts and terminates normally.
   - `KeyError` → print `Error: ...` to stderr, exit 1.
   - `PermissionError` → print `Access denied: ...` to stderr, exit 1.
3. **Print core identity** (always):
   ```
   ID:     <node['id']>
   Text:   <node['text']>
   Status: <node['truth_value']>
   ```
4. **Print provenance** conditionally (truthiness of value):
   - `source`, `source_url` (via `.get`), `source_hash` (via `[]`).
   - From `metadata`: `source_type`, `accepted_pr`, `pinned_sha` (truncated to first 12 chars, with ` (lines N)` suffix when `pinned_lines` is set).
5. **Print justifications**:
   - If `node["justifications"]` is non-empty: header `Justifications (N):` then one line per justification as `type(antecedent1, antecedent2) [label]` (label omitted when empty).
   - Otherwise print the literal line `Premise (no justifications)`.
6. **Print metadata extras**: `retract_reason` (prefixed by a blank line), `example` (newlines are re-indented two spaces).
7. **Print timestamps** by iterating a fixed ordered tuple `("created_at", "updated_at", "reviewed_at", "verified_at", "retracted_at")` using `node.get(key, "")`. Labels are derived: `ts_key.replace("_", " ").title().replace(" At", "")`, e.g. `created_at` → `Created`. A blank line is emitted first only if at least one timestamp exists. **Missing timestamps are skipped**.
8. **Print tags** if `node.get("tags")` is truthy.
9. **Print sources** if `node.get("sources")` is truthy: each is `source_ref [source_type] (label)` (brackets/parens omitted if the sub-field is empty).
10. **Print dependents** if `node["dependents"]` is truthy.

### Side Effects

- **stdout I/O**: multiple `print()` calls.
- **stderr I/O**: only on `KeyError`/`PermissionError` paths.
- **Process termination**: `sys.exit(1)` on error; `SystemExit` raised via `_backend_kwargs` is *also* terminal.
- **No DB mutation**: `show` is read-only; it does not call `propagate`, `retract`, or stash any state.
- **No persistent caching**: repeated calls re-hit the API.

### Error Handling

- **Caught**: `KeyError` from `api.show_node` (typical "node not found" signal) and `PermissionError` (raised when `node.access_tags` ⊄ `visible_to`). Both print to stderr and exit 1.
- **Not caught**: any other exception from `api.show_node` (e.g. DB operational errors, `ValueError`); any `KeyError` from the *formatting* path when a required top-level key is missing; `AttributeError` if `node["metadata"]` is `None`.
- **Escalation**: `SystemExit` raised inside `_backend_kwargs` bypasses the excepts (it derives from `BaseException`), which is the intended behavior.

> **Assumption not enforced by types**: `api.show_node` must always return a dict containing the keys strictly accessed via `node[k]` (`source`, `source_hash`, `metadata`, `justifications`, `dependents`), and `metadata` must be a dict — otherwise the function crashes with a traceback instead of the friendly error.

### Usage Patterns

- Called by `main()` through the dispatch map: `commands["show"] = cmd_show`. The caller is `argparse`-responsible for the `show` subparser (`node_id` positional and optional `--visible-to`).
- Typical invocation:
  ```bash
  reasons show my.belief.id
  reasons show my.belief.id --visible-to team-a,public
  reasons show my.belief.id --pg "postgresql://..." --domain-id my.domain
  ```
- **Caller obligations**: none beyond providing a valid `Namespace`; script writers quoting the output should be aware that "Premise (no justifications)" is printed for premises and that some fields only appear when non-empty.

### Dependencies

- `reasons.api.show_node` — the only external data source; defines the schema of the returned dict and the visibility semantics.
- `_parse_visible_to(args)` — same module helper; returns `list[str] | None`.
- `_backend_kwargs(args)` — same module; picks SQLite vs PG kwargs (and may exit early).
- `sys` — for `stderr` and `sys.exit`.

---

## Topics to Explore

- [function] `reasons/api.py:show_node` — Defines the exact return shape `cmd_show` relies on (especially which keys are always present vs. optional, and when `PermissionError` is raised).
- [function] `reasons/cli.py:_backend_kwargs` — Explains how SQLite vs. PostgreSQL kwargs are built and why a `--pg` without a domain id terminates before `api.show_node` is reached.
- [function] `reasons/cli.py:_parse_visible_to` — The visibility tag-set parser that feeds `visible_to`; note its `None` vs. empty-list distinction.
- [file] `tests/test_cli.py` — Contains the regression tests for `cmd_show` output and error handling; useful to see expected stdout/stderr shapes.
- [file] `tests/test_source_type.py` — Exercises the `source_type` metadata field that `cmd_show` prints as `Source type:`.

## Beliefs

- `cmd-show-only-catches-lookup-errors` — `cmd_show` catches `KeyError`/`PermissionError` only around the `api.show_node` call; formatting-time `KeyError`s propagate as uncaught tracebacks.
- `cmd-show-requires-non-optional-top-level-keys` — `cmd_show` accesses `node['source']`, `node['source_hash']`, `node['metadata']`, `node['justifications']`, and `node['dependents']` without `.get`, so `api.show_node` must always return them.
- `cmd-show-assumes-metadata-is-a-dict` — `cmd_show` assumes `node["metadata"]` is a dict (not `None`); calling `.get` on a `None` value would raise `AttributeError`.
- `cmd-show-truncates-pinned-sha` — When `metadata["pinned_sha"]` is truthy, `cmd_show` displays only its first 12 characters.
- `cmd-show-uses-fixed-timestamp-order` — Timestamp display order is hard-coded to `created_at, updated_at, reviewed_at, verified_at, retracted_at` and entries with falsy values are silently skipped.
