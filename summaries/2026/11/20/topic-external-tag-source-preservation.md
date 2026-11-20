---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Trace `add_tags` / `add_source` callers to understand the out-of-band write pattern `save` is protecting against.

## Explanation

### Caller trace

`add_tags` and `add_source` are public API functions, and the observed callers are split across CLI entry points and tests.

- `add_tags` is defined at `reasons/api.py:938`:
  - signature: `def add_tags(node_id: str, tags: list[str], db_path: str = DEFAULT_DB, ...)`
  - it raises `NotImplementedError("add_tags is not supported with PostgreSQL yet")` at `reasons/api.py:942`.

- `add_source` is defined at `reasons/api.py:1007`:
  - signature starts with `def add_source(node_id: str, source_ref: str, source_type: str = "", ...)`
  - it raises `NotImplementedError("add_source is not supported with PostgreSQL yet")` at `reasons/api.py:1013`.

The CLI is the main production caller:

- `reasons/cli.py:590` calls:
  - `result = api.add_tags(args.node_id, args.tags, **_backend_kwargs(args))`
- `reasons/cli.py:621` defines `cmd_add_source`, and `reasons/cli.py:623` calls `api.add_source(...)`.
- `reasons/cli.py:3942` registers the command mapping `"add-source": cmd_add_source`.

So users can mutate tags and sources through CLI commands without going through whatever normal node persistence path exists.

Tests call both APIs directly and heavily:

- `add_tags` callers appear throughout `tests/test_api.py` from around `:1797` through `:2024`.
- `add_source` callers appear from around `tests/test_api.py:1871` through `:1933`.
- Tests cover cases such as `test_add_tags`, `test_add_tags_idempotent`, `test_add_tags_missing_node`, `test_add_source`, and `test_add_source_missing_node`.

### The out-of-band write pattern

The key evidence is the storage comment:

- `reasons/storage.py:210`: `# add_tags()/add_source() write directly to these tables,`

That comment indicates `add_tags` and `add_source` are not ordinary in-memory mutations that are later persisted only through a parent `save` operation. They write directly to the relevant tag/source storage tables. This is the “out-of-band” pattern: a node may exist, then tags or sources are added through a separate API/CLI path that bypasses the normal object-save workflow.

The protection `save` is likely responsible for is therefore stale-state reconciliation:

- A node object may have been loaded with an older tag/source view.
- `add_tags` / `add_source` may then write new rows directly to the tables.
- If `save` later persists that stale node object, it could accidentally overwrite, drop, or ignore those direct table writes.

In other words, `save` must not assume that the in-memory node object owns the complete tag/source state. It needs to either merge, reload, or avoid clobbering direct writes made by `add_tags` / `add_source`.

### What cannot be verified from the observations

The observations are insufficient to trace the exact `save` mechanism:

- `locate_save_definition` found no `save` definition.
- `search_out_of_band_comments` found no matches.
- `search_storage_write_methods` returned no matches.
- The `reasons/storage.py:210` comment is truncated and only proves that direct writes exist; it does not show the exact reconciliation logic.

To fully understand what `save` protects against, we would need the body of `save`, the surrounding table/schema definitions in `reasons/storage.py`, and the implementations of `add_tags` / `add_source` after their PostgreSQL guards.

## Topics to Explore

- [file] `reasons/storage.py` — Contains the direct-write comment at `:210` and likely the persistence code that needs to reconcile with `add_tags` / `add_source`.
- [function] `reasons/api.py:add_tags` — Inspect the implementation after `:942` to see exactly which tables are written and whether any transaction/locking behavior exists.
- [function] `reasons/api.py:add_source` — Inspect the implementation after `:1013` to compare source-table writes with tag-table writes.
- [file] `tests/test_api.py` — Review tests around `:1797`–`:2024` and `:1871`–`:1933` for expected idempotency, missing-node, and save-survival behavior.
- [general] `save-direct-write-reconciliation` — Search for the actual `save` implementation and compare its tag/source handling to the direct writes performed by `add_tags` / `add_source`.

## Beliefs

- `add-tags-cli-calls-api-add-tags` — `reasons/cli.py:590` calls `api.add_tags(args.node_id, args.tags, **_backend_kwargs(args))`.
- `add-source-cli-registered-as-add-source` — `reasons/cli.py:3942` maps `"add-source"` to `cmd_add_source`, which calls `api.add_source` at `reasons/cli.py:623`.
- `add-tags-and-add-source-write-directly-to-tables` — `reasons/storage.py:210` states that `add_tags()` and `add_source()` write directly to the referenced storage tables.
- `postgresql-rejects-add-tags-and-add-source` — `reasons/api.py:942` and `reasons/api.py:1013` raise `NotImplementedError` for PostgreSQL.
- `save-implementation-not-observed` — The observations contain no `save` definition and no out-of-band comment matches, so the exact reconciliation logic remains unverified.
