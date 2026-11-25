---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore how the SQLite and PG paths stay consistent, especially where arguments like `db_path` are silently ignored.

The SQLite and PG paths are kept consistent by a **facade/dispatch layer**, not by sharing a storage implementation. SQLite uses the stateful `Storage` class (`reasons/storage.py`) with full-network `save()`/`load()`. PG uses `PgApi` (`reasons/pg.py`) with one transaction per operation. The API/CLI layer chooses exactly one backend and forwards only that backend’s kwargs.

## Backend selection: where `db_path` is silently ignored

The clearest evidence is in `tests/test_pg_dispatch.py`.

- SQLite default path:
  - `test_sqlite_default` creates `args = Namespace(db="reasons.db", pg=None, domain_id=None)`.
  - `_backend_kwargs(args)` returns exactly `{"db_path": "reasons.db"}`.
  - See `tests/test_pg_dispatch.py:40-46`.

- PG path:
  - `test_pg_with_domain_id` still sets `args.db="reasons.db"`, but with `args.pg` set.
  - `_backend_kwargs(args)` returns only `{"pg_conninfo": "...", "domain_id": "abc-123"}`.
  - `db_path` is completely absent. See `tests/test_pg_dispatch.py:48-53`.
  - The same happens with env-var fallback: `REASONS_PG_CONNINFO` + `REASONS_PROJECT_ID` produce only PG kwargs, not `db_path` (`tests/test_pg_dispatch.py:64-69`).

So the ignore is deliberate and structural: once PG is selected, the CLI/API kwargs are built for PG only. The default `--db reasons.db` is not rejected, warned about, or forwarded. It is simply dropped before the API call.

There is also a guard for commands that are still SQLite-only. `_require_sqlite` exits when PG is configured, preventing a silent fallback to `db_path` for unsupported commands. Tests cover this at `tests/test_pg_dispatch.py:80-98`.

Important caveat: the grep observations claiming `db_path` appears zero times in `reasons/` and tests are inconsistent with the supplied file contents. `db_path` clearly appears in `Storage.__init__` (`reasons/storage.py:125-126`) and in `tests/test_storage.py:12-14`. Those grep results should not be trusted.

## How the paths stay semantically aligned

### 1. Unified dispatch surface

`_pg_dispatch` is the PG gateway. It constructs `PgApi`, enters its context manager, and calls the requested method with forwarded kwargs. Tests show:

- `_pg_dispatch("postgresql://...", "proj-1", "get_status")` calls `PgApi.get_status()`.
- `_pg_dispatch(..., "show_node", node_id="a")` calls `PgApi.show_node(node_id="a")`.
- See `tests/test_pg_dispatch.py:19-35`.

Higher-level API functions then behave like a facade: `export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, etc. Tests assert that when `pg_conninfo` is supplied they dispatch to the matching `PgApi` method with normalized kwargs, e.g.:

- `import_json(... pg_conninfo=..., domain_id=...)` calls `mock_pg.import_json(data=data)` (`tests/test_pg_dispatch.py:237+`).
- `hash_sources(... pg_conninfo=...)` calls `mock_pg.hash_sources(force=..., repos=...)` (`tests/test_pg_dispatch.py:400+`).
- `check_stale(... pg_conninfo=...)` calls `mock_pg.check_stale(repos=..., upgrade_hashes=..., git_aware=...)` (`tests/test_pg_dispatch.py:430+`).

That keeps backend-specific argument translation localized to the dispatch layer.

### 2. Parallel data models

Both backends persist the same core concepts: nodes, justifications, nogoods, propagation log, and network metadata.

- SQLite schema: `reasons/storage.py:16-92`.
- PG schema: `reasons/pg.py:21-75`.

Both model truth values as `IN`/`OUT`, source fields, timestamps, and JSON-ish metadata. PG adds `domain_id` to every table and uses composite keys; SQLite isolates by database file and uses single-column keys. That is the core mapping:

- SQLite identity/isolation comes from `db_path`.
- PG identity/isolation comes from `domain_id`.

This shows up in defaults:

- SQLite defaults `project_name` from `db_path.stem` (`reasons/storage.py:190-192`, and again at `reasons/storage.py:326`).
- PG defaults `project_name` from the supplied project name or `domain_id` (`reasons/pg.py:147-159`).

So when PG is selected, `db_path` has no PG equivalent; `domain_id` takes over the isolation role.

### 3. Transaction model differences are intentional

SQLite uses whole-network snapshot persistence:

- `Storage.save()` enters `with self.conn:`, clears and rewrites tables, and writes the entire `Network` (`reasons/storage.py:205-346`).
- `Storage.load()` reconstructs nodes, justifications, nogoods, metadata, repos, and log, then rebuilds dependents via `network._rebuild_dependents()` (`reasons/storage.py:348-477`, especially line 432).
- Schema migration is handled in `_init_schema()` via `PRAGMA table_info` and `ALTER TABLE` (`reasons/storage.py:134-203`).

PG uses per-operation transactions:

- `PgApi.__init__` stores a connection and `domain_id` (`reasons/pg.py:103-111`).
- `__exit__` rolls back on exception (`reasons/pg.py:120-123`).
- `add_node()` and `add_justification()` commit individually (`reasons/pg.py:281`, `reasons/pg.py:348`).
- `retract_node()` and `assert_node()` also propagate and commit per operation (`reasons/pg.py:356-434`).
- `what_if_retract()` deliberately rolls back in `finally` because it simulates changes (`reasons/pg.py:438-473`).

So consistency is maintained at the **API contract level**: the same operation names and result shapes, but different persistence mechanics.

### 4. Schema-version metadata is seeded similarly

Both backends seed `schema_version`, `project_name`, `created_at`, and `updated_at` into their metadata tables:

- SQLite: `reasons/storage.py:190-202`.
- PG: `reasons/pg.py:147-160`.

SQLite uses `INSERT OR IGNORE`; PG uses `ON CONFLICT ... DO NOTHING`. That keeps metadata initialization idempotent on both sides.

## Where consistency is not fully proven by these observations

The observations do not include the actual implementations of `reasons/api.py` or `reasons/cli.py`, only tests. So we can infer the branch behavior but cannot see the exact conditionals or signatures.

Also, only the first 500 lines of `reasons/pg.py` are provided. The visible PG schema does not show `repos`, `node_tags`, `node_sources`, or FTS equivalents that SQLite has (`reasons/storage.py:52-91`). Those may exist later in `pg.py`, but their absence from the provided excerpt means full cross-backend schema parity is unverified.

Finally, the tests are mostly dispatch tests with mocked `PgApi`, plus SQLite round-trip tests in `tests/test_storage.py`. There is no visible test that runs the same sequence against both SQLite and PG and compares exported networks.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect `_backend_kwargs` and `_require_sqlite` to confirm exactly when `db_path` is dropped and which commands remain SQLite-only.
- [file] `reasons/api.py` — Inspect how public API functions branch between `Storage`/`Network` and `_pg_dispatch`, and whether conflicting `db_path` + `pg_conninfo` inputs are validated.
- [file] `reasons/pg.py` — Review the remaining PG methods and schema to verify parity for repos, tags, sources, search, export, and import.
- [general] `cross-backend-parity-tests` — Look for tests that exercise the same operations on SQLite and PG and compare semantic results, not just mock dispatch.
- [function] `reasons/storage.py:Storage.save` — Understand the whole-network snapshot strategy and the external tag/source preservation logic.

## Beliefs

- `pg-backend-selection-ignores-db-path` — When PG is selected, `_backend_kwargs` returns only `pg_conninfo` and `domain_id`, omitting `db_path` even when `args.db` is set (`tests/test_pg_dispatch.py:48-53`).
- `sqlite-storage-is-whole-network-snapshot` — `Storage.save` clears and rewrites all SQLite tables inside one transaction, and `Storage.load` reconstructs the network and rebuilds dependents (`reasons/storage.py:205-346`, `reasons/storage.py:348-477`).
- `pgapi-is-transaction-per-operation` — `PgApi` commits each operation individually and rolls back on context-manager exceptions; it has no full-network `save`/`load` (`reasons/pg.py:97-123`, `reasons/pg.py:281`, `reasons/pg.py:348`).
- `pg-domain-id-maps-to-sqlite-db-file` — SQLite defaults `project_name` from `db_path.stem`, while PG defaults it from `project_name` or `domain_id` (`reasons/storage.py:190-192`, `reasons/pg.py:147-159`).
- `require-sqlite-prevents-silent-sqlite-fallback` — `_require_sqlite` raises `SystemExit` when PG is configured, preventing SQLite-only CLI commands from silently using `db_path` (`tests/test_pg_dispatch.py:80-98`).
