---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Which `api.py` functions lack a `pg_conninfo` dispatch branch (this one, plus tag/source helpers), and what that implies for Postgres deployments.

## Explanation

Based on the observed slices, the key mechanism is `_pg_dispatch` in `reasons/api.py:184-188`:

```python
def _pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs):
    from .pg import PgApi
    with PgApi(pg_conninfo, domain_id) as pg:
        return getattr(pg, method_name)(**kwargs)
```

A public `api.py` function only routes to PostgreSQL if it has an explicit branch like:

```python
if pg_conninfo:
    return _pg_dispatch(pg_conninfo, domain_id, "some_method", ...)
```

The observed public functions that do have this branch include:

- `init_db` — `reasons/api.py:193`, branch at `203-204`
- `ensure_namespace` — `reasons/api.py:217`, branch at `226-227`
- `list_namespaces` — `reasons/api.py:243`, branch at `251-252`
- `add_node` — `reasons/api.py:293`, branch at `312-315`
- `add_justification` — `reasons/api.py:415`, branch at `431-434`
- `retract_node` — `reasons/api.py:474`, branch at `484-485`
- `what_if_retract` — `reasons/api.py:513`, branch at `523-524`
- `what_if_assert` — `reasons/api.py:581`, branch at `590-591`
- `what_if_supersede` — `reasons/api.py:649`, branch at `658-659`

The tests also explicitly verify PG dispatch for:

- `hash_sources` — `tests/test_pg_dispatch.py::TestHashSourcesDispatch`
- `check_stale` — `tests/test_pg_dispatch.py::TestCheckStaleDispatch`

So those source-oriented helpers are not confirmed missing.

### Confirmed helpers without a direct `pg_conninfo` dispatch branch

From the observed `reasons/api.py` snippet:

- `_get_node_tags` — opens `sqlite3.connect(db_path)` and queries the `node_tags` table. It has no `pg_conninfo` parameter or PG branch.
- `_filter_by_tags` — depends on `_get_node_tags`, so it is SQLite-only as written.
- `_is_visible` — pure helper over `node.metadata["access_tags"]`; no backend access, so no dispatch needed, but it only works on a loaded `Network` object.
- `_with_network` — SQLite load/save/cache context manager; no PG equivalent.
- `clear_network_cache` / `_network_cache` — SQLite cache management only.
- `_resolve_namespace` — pure string helper; no dispatch needed.
- `_pg_dispatch` — the dispatcher itself, not a public API function.
- `export_markdown` — as exercised by `tests/test_pg_dispatch.py::TestExportMarkdownPgPath`, it does not call `_pg_dispatch` directly. It accepts `pg_conninfo` and delegates to `export_network`; the test patches `export_network`, so PG correctness depends on `export_network` having its own dispatch branch.

Important caveat: the observations only include the first 700 lines of `reasons/api.py` out of 5,886 total lines, and `list_api_functions` returned 0 matches. So this is **not a complete audit** of every `api.py` function. The match at `reasons/api.py:756` shows another function signature with `pg_conninfo`, but the snippet ends before its body, so its dispatch behavior is unconfirmed.

### What this implies for Postgres deployments

1. **`pg_conninfo` can be silently ignored.**
   If a public function accepts `pg_conninfo` but lacks the `if pg_conninfo:` branch, it will continue down the SQLite path and use `db_path` / `reasons.db`. In a Postgres deployment, that can read or write a stale local SQLite database while the rest of the system uses Postgres.

2. **Tag filtering is not PG-compatible as written.**
   `_get_node_tags` and `_filter_by_tags` call `sqlite3.connect(db_path)` and query `node_tags`. The observed Postgres schema in `reasons/pg.py:SCHEMA` has `rms_nodes.metadata JSONB` but no `node_tags` table. Any PG code path that reaches these helpers will either fail with “no such table” or operate against the wrong backend.

3. **Access-tag visibility is backend-sensitive.**
   `_is_visible` checks `node.metadata["access_tags"]`, which can work on a loaded `Network`, but the SQLite tag helpers are not mirrored by a PG `node_tags` query in the observed schema. PG tag filtering must be implemented against `metadata JSONB` or a new PG table.

4. **Source helpers are partially covered, but not all paths are proven.**
   `hash_sources` and `check_stale` are tested to dispatch to `PgApi`. However, CLI gating may still block some commands: `tests/test_pg_dispatch.py::TestRequireSqlite` expects `_require_sqlite(args, "hash-sources")` to exit when PG is configured. That is a CLI/API coverage mismatch worth checking.

5. **The safest design is explicit failure or explicit dispatch.**
   For any function that is intentionally SQLite-only, passing `pg_conninfo` should raise `NotImplementedError` rather than silently using SQLite. For functions that should support Postgres, add `if pg_conninfo: return _pg_dispatch(...)` and corresponding `PgApi` methods/tests.

## Topics to Explore

- [file] `reasons/api.py` — Full 5,886-line file; enumerate every public function and confirm which ones have `if pg_conninfo:` branches.
- [function] `reasons/api.py:_get_node_tags` — SQLite-only tag loader; trace all callers to see which PG paths depend on it.
- [function] `reasons/pg.py:PgApi` — Inspect methods for tag filtering, lookup, source hashing, and staleness to see which API paths have PG equivalents.
- [file] `tests/test_pg_dispatch.py` — Existing dispatch tests show which paths are covered and which are still missing.
- [general] `pg-tag-filtering-design` — Decide whether PG tags should be queried from `metadata JSONB` or a dedicated table, and how filters/access checks map to SQL.

## Beliefs

- `pg-conninfo-dispatch-requires-explicit-branch` — A public `api.py` function only routes to PostgreSQL when it has an explicit `if pg_conninfo: return _pg_dispatch(...)` branch.
- `tag-helpers-are-sqlite-only` — `_get_node_tags` and `_filter_by_tags` use `sqlite3.connect(db_path)` and query `node_tags`, so they cannot operate on PostgreSQL as written.
- `pg-schema-has-no-node-tags-table` — The observed `reasons/pg.py` schema defines `rms_nodes.metadata JSONB` but no `node_tags` table.
- `hash-sources-and-check-stale-dispatch-to-pgapi` — `tests/test_pg_dispatch.py::TestHashSourcesDispatch` and `TestCheckStaleDispatch` verify PG dispatch for `hash_sources` and `check_stale`.
- `export-markdown-delegates-pg-routing` — `export_markdown` is tested with `pg_conninfo` by patching `export_network`, so it relies on `export_network` for PG dispatch rather than dispatching directly.
