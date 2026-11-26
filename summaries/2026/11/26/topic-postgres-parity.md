---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Which APIs have real `PgApi` implementations vs. inferring behavior from `pg_conninfo` branching; the skipped stamping path is one of several asymmetries

## Backend dispatch model

`reasons/api.py` supports two backends:

- SQLite/Network path: the normal function body runs when `pg_conninfo` is absent.
- Postgres path: a function with `if pg_conninfo:` calls `_pg_dispatch(...)`.

`_pg_dispatch` is defined at `reasons/api.py:184`. It constructs `PgApi(pg_conninfo, domain_id)` at `reasons/api.py:187` and, per the dispatch tests, invokes the same-named method on that object. For example, `_pg_dispatch("postgresql://...", "proj-1", "get_status")` calls `mock_pg.get_status()` and `_pg_dispatch(..., "show_node", node_id="a")` calls `mock_pg.show_node(node_id="a")` (`tests/test_pg_dispatch.py:23-38`).

So a `pg_conninfo` branch is only a dispatch promise. It does **not** prove that the real `PgApi` class implements that method. The real implementation must exist as a method on `reasons/pg.py:PgApi` or the call fails with `AttributeError`.

## What is confirmed as a real `PgApi` implementation

From the provided `reasons/pg.py` slice and `tests/test_pg.py`, these are directly implemented or directly exercised against real Postgres:

| API surface | Evidence |
|---|---|
| `PgApi.__init__`, `close`, `__enter__`, `__exit__` | `reasons/pg.py:98-130` |
| `init_db` | `reasons/pg.py` implementation in the first 400 lines; dispatched at `reasons/api.py:203-204` |
| `add_node` | `reasons/pg.py` implementation; dispatched at `reasons/api.py:312-315` |
| `add_justification` | `reasons/pg.py` implementation; dispatched at `reasons/api.py:431-434` |
| `retract_node` | `reasons/pg.py` implementation; dispatched at `reasons/api.py:484-485` |
| `assert_node` | Directly tested in `tests/test_pg.py` `TestAssertNode` |
| `show_node` | Directly tested in `tests/test_pg.py` `TestShowNode` |
| `get_status` | Directly tested in `tests/test_pg.py` `TestGetStatus` |
| `search` | Directly tested in `tests/test_pg.py` `TestSearch` |
| `list_nodes` | Directly tested in `tests/test_pg.py` `TestListNodes` |
| `list_gated` | Directly tested in `tests/test_pg.py` `TestListGated` |

`add_node` also calls `self.ensure_namespace(namespace)` internally in `reasons/pg.py`, so a real `ensure_namespace` method must exist somewhere beyond the returned 400-line slice. The API dispatches `ensure_namespace` at `reasons/api.py:226-227` and `list_namespaces` at `reasons/api.py:251-252`, but the full method bodies are not in the provided slice.

## What is only inferred from dispatch wiring

These have evidence of API dispatch or mocked dispatch tests, but the provided `reasons/pg.py` slice does not show the real implementation:

| API surface | Dispatch evidence | Caveat |
|---|---|---|
| `ensure_namespace` | `reasons/api.py:226-227` | Called internally by `PgApi.add_node`; implementation not shown |
| `list_namespaces` | `reasons/api.py:251-252` | Implementation not shown |
| `what_if_retract` | `reasons/api.py:523-524` | No real `PgApi` test shown |
| `what_if_assert` | `reasons/api.py:590-591` | No real `PgApi` test shown |
| `what_if_supersede` | `reasons/api.py:658-659` | No real `PgApi` test shown |
| `import_json` | `tests/test_pg_dispatch.py` `TestImportJsonDispatch` | Test patches `reasons.pg.PgApi` with `MagicMock` at `tests/test_pg_dispatch.py:226-228` |
| `import_beliefs` | `tests/test_pg_dispatch.py` `TestImportBeliefsDispatch` | Mock-based dispatch test |
| `import_agent` | `tests/test_pg_dispatch.py` `TestImportAgentDispatch` | Mock-based dispatch test |
| `sync_agent` | `tests/test_pg_dispatch.py` `TestSyncAgentDispatch` | Mock-based dispatch test |
| `hash_sources` | `tests/test_pg_dispatch.py` `TestHashSourcesDispatch` | Mock-based dispatch test |
| `check_stale` | `tests/test_pg_dispatch.py` `TestCheckStaleDispatch` | Mock-based dispatch test |

This is the key trap: `tests/test_pg_dispatch.py:23-28`, `:226-228`, `:251-253`, etc. replace `reasons.pg.PgApi` with a `MagicMock`. Those tests prove that `_pg_dispatch` invokes the expected method name and kwargs. They do **not** prove that the real `PgApi` class has that method, nor that its semantics match SQLite.

## Known “no PgApi implementation” paths

The CLI explicitly rejects four commands when `--pg` is used:

- `mark-duplicate` — `reasons/cli.py:157`
- `mark-superseded` — `reasons/cli.py:179`
- `defeat-justification` — `reasons/cli.py:201`
- `migrate-defeaters` — `reasons/cli.py:287`

Each prints an error like:

```text
Error: mark-duplicate is not supported with --pg (no PgApi implementation)
```

So those are confirmed SQLite-only at the CLI boundary.

There is also `_require_sqlite` in `reasons/cli.py`, tested in `tests/test_pg_dispatch.py` `TestRequireSqlite`. It exits when `--pg` or `REASONS_PG_CONNINFO` is set. The test uses `"hash-sources"` as the checked command, which is an asymmetry worth noting: the API-level `hash_sources` has a PG dispatch test in `TestHashSourcesDispatch`, but the CLI-level `_require_sqlite` test expects `"hash-sources"` to exit under PG. That suggests API capability and CLI exposure do not always agree.

## The skipped stamping path as an asymmetry

The stamping bookkeeping lives in `reasons/api.py`:

- `stamp_failed = []` at `reasons/api.py:3868`
- failed node IDs appended at `reasons/api.py:3895`
- fallback `stamp_failed = list(verified)` at `reasons/api.py:3897`
- `result["stamp_failed"] = stamp_failed` at `reasons/api.py:3909-3910`
- CLI warns on it at `reasons/cli.py:2273-2274`

There is also a related timestamp-preservation comment at `reasons/api.py:2186`:

```python
# Restore exact timestamps AFTER retract/assert to avoid overwrite
```

The provided `search_stamping_paths` results do **not** show a corresponding `PgApi` stamping implementation. The only PG stamping-related references shown are the `verified_at` column in the schema at `reasons/pg.py:64` and a propagation-log query at `reasons/pg.py:1810`.

So the asymmetry is:

1. The shared/SQLite path performs stamping and returns `stamp_failed`.
2. If the enclosing API function has an early `if pg_conninfo: return _pg_dispatch(...)`, the PG path bypasses that stamping block entirely.
3. Unless the `PgApi` method duplicates the stamping behavior and returns its own `stamp_failed`, PG mode silently skips that contract.
4. The CLI warning at `reasons/cli.py:2273-2274` would then never fire for PG mode because `result["stamp_failed"]` would be absent.

The observations do not show the enclosing function name around `reasons/api.py:3868`, nor a full `PgApi` method that updates `verified_at`. That is the missing piece needed to confirm whether this is a live bug or an intentionally duplicated PgApi behavior.

## What is missing

The observations are not sufficient for a complete API-by-API matrix:

- `search_pgapi_usages` has `total_count: 90` but only about 30 matches are shown.
- `search_pg_conninfo_branches` has `total_count: 236` but only the first 30 or so are shown.
- `read_pg_module` returns only the first 400 of 3736 lines of `reasons/pg.py`.
- `read_pg_tests` returns only the first 400 of 1313 lines of `tests/test_pg.py`.
- The full `reasons/api.py` around the stamping block and the full `PgApi` class are needed to know which dispatch names are real methods vs. mocked assumptions.

## Topics to Explore

- [file] `reasons/pg.py` — Read the full `PgApi` class to enumerate every real method beyond the first 400 lines, especially `ensure_namespace`, `list_namespaces`, `what_if_*`, `import_*`, `sync_agent`, `hash_sources`, `check_stale`, and any `verified_at` stamping logic.
- [function] `reasons/api.py:_pg_dispatch` — Confirm exactly how the method name is resolved and whether missing `PgApi` methods surface as `AttributeError`.
- [file] `tests/test_pg.py` — Read beyond line 400 for real Postgres tests covering import/sync/hash/stale and any stamping-related behavior.
- [general] `pg-conninfo-branch-audit` — Compare all 236 `pg_conninfo` references against actual `PgApi` methods to find dispatch branches without real implementations.
- [file] `reasons/api.py` — Inspect around line 3868 and line 2186 to identify the enclosing API function and whether its PG branch bypasses stamping/timestamp restoration.

## Beliefs

- `pg-dispatch-requires-pgapi-method` — `_pg_dispatch` delegates API calls to a same-named `PgApi` method; a `pg_conninfo` branch alone does not prove that method exists.
- `dispatch-tests-mock-pgapi` — `tests/test_pg_dispatch.py` patches `reasons.pg.PgApi` with `MagicMock`, so those tests verify dispatch wiring, not real `PgApi` method existence or semantic parity.
- `cli-blocks-four-pg-commands` — `mark-duplicate`, `mark-superseded`, `defeat-justification`, and `migrate-defeaters` exit under `--pg` with “no PgApi implementation” at `reasons/cli.py:157,179,201,287`.
- `stamp-failed-is-api-only-in-observations` — `stamp_failed` is produced in `reasons/api.py:3868-3910` and consumed in `reasons/cli.py:2273-2274`; no `PgApi` stamping implementation appears in the provided `reasons/pg.py` slice.
- `pgapi-real-methods-in-slice` — `PgApi` directly implements `init_db`, `add_node`, `add_justification`, and `retract_node` in `reasons/pg.py:98-400`, while real PG tests exercise `assert_node`, `show_node`, `get_status`, `search`, `list_nodes`, and `list_gated`.
