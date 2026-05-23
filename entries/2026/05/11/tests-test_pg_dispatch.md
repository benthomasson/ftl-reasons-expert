# File: tests/test_pg_dispatch.py

**Date:** 2026-05-11
**Time:** 12:51

## Purpose

`test_pg_dispatch.py` is the test suite for the **PostgreSQL backend dispatch layer** — the routing logic that lets the `reasons` CLI and API work against either a local SQLite database or a remote PostgreSQL instance. It verifies that when `pg_conninfo` and `project_id` are provided, every API function correctly delegates to `PgApi` methods instead of touching SQLite, and that the CLI argument resolution (`_backend_kwargs`, `_require_sqlite`) correctly negotiates between CLI flags, environment variables, and defaults.

This file does **not** test `PgApi` itself (that's `test_pg.py`). It tests the *dispatch plumbing* — the if/else branching in `api.py` and `cli.py` that decides which backend to use.

## Key Components

### `TestPgDispatch`
Tests the low-level `_pg_dispatch` helper in `api.py`. This function is the single chokepoint: given a connection string, project ID, and method name, it opens a `PgApi` context manager and calls the named method with forwarded kwargs. The two tests verify:
- The method is called and its return value is passed through.
- Keyword arguments are forwarded intact.

### `TestBackendKwargs`
Tests `cli._backend_kwargs`, which resolves a CLI `argparse.Namespace` into a dict of either `{"db_path": ...}` (SQLite) or `{"pg_conninfo": ..., "project_id": ...}` (Postgres). Five scenarios:
- **SQLite default**: no PG flags, no env vars → returns `db_path`.
- **PG with project ID**: both flags present → returns PG kwargs.
- **PG missing project ID**: `--pg` without `--project-id` and no `REASONS_PROJECT_ID` env var → `SystemExit`.
- **Env var fallback**: `REASONS_PG_CONNINFO` + `REASONS_PROJECT_ID` in env → returns PG kwargs.
- **CLI overrides env**: CLI flags take priority over environment variables.

### `TestRequireSqlite`
Tests `cli._require_sqlite`, a guard function that blocks certain subcommands (like `hash-sources`) from running against Postgres. Three cases: no PG passes, `--pg` flag exits, `REASONS_PG_CONNINFO` env var exits.

### `TestExportMarkdownPgPath`
The most data-rich test class. Uses a class-level `EXPORT_DATA` dict that models a small belief network (premise, derived node, gated node with outlist, blocker, and a nogood). Tests that `export_markdown()` correctly accepts `pg_conninfo`/`project_id` kwargs, delegates to `export_network`, and produces markdown containing node IDs, text, justification references, source info, nogoods, and dependent relationships. Also tests the empty-network edge case.

### `TestImportJsonDispatch`, `TestImportBeliefsDispatch`, `TestImportAgentDispatch`, `TestSyncAgentDispatch`
Each tests that the corresponding API function reads a file from disk, parses it, then delegates the parsed data to the appropriate `PgApi` method. `import_agent` has two variants: markdown and JSON input. These tests verify both the delegation and the data transformation (file → parsed structure → PgApi call).

### `TestHashSourcesDispatch`, `TestCheckStaleDispatch`, `TestLookupDispatch`
Test that maintenance API functions (`hash_sources`, `check_stale`, `lookup`) dispatch to PgApi with correct default and explicit arguments (e.g., `force`, `repos`, `upgrade_hashes`, `visible_to`).

### `TestAddRepoDispatch`, `TestListReposDispatch`, `TestListNegativeDispatch`
Test repo management and negative-belief listing dispatch, including optional parameters like `visible_to` and `model`.

### `Test*NoLongerBlocked` classes
Three classes — `TestImportCliNoLongerBlocked`, `TestMaintenanceCliNoLongerBlocked`, `TestRepoAndNegativeCliNoLongerBlocked` — serve as **regression guards**. These verify that commands previously gated behind `_require_sqlite` (and thus blocked from Postgres) now accept PG kwargs without error. These are the "acceptance tests" that confirm the PG backend was fully wired up for each operation.

## Patterns

**Uniform mock structure.** Every PG-path test follows the same pattern:
```python
mock_pg = MagicMock()
mock_pg.<method>.return_value = <expected>
with patch("reasons_lib.pg.PgApi") as MockPgApi:
    MockPgApi.return_value.__enter__ = MagicMock(return_value=mock_pg)
    MockPgApi.return_value.__exit__ = MagicMock(return_value=False)
    result = api_function(..., pg_conninfo="postgresql://...", project_id="test")
```
This mocks `PgApi` as a context manager without needing a real database. The pattern is repeated ~30 times, which is verbose but keeps each test self-contained and readable.

**Environment isolation.** Tests that check env-var behavior use `patch.dict("os.environ", ...)` with `clear=True` or explicit exclusion of `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` to prevent the host environment from leaking into assertions.

**`tmp_path` for file-based operations.** Import tests use pytest's `tmp_path` fixture to create temporary JSON/markdown files, avoiding filesystem pollution.

**SystemExit as validation.** Invalid configurations (PG without project ID, sqlite-only commands with PG) are tested via `pytest.raises(SystemExit)`, matching the `sys.exit()` calls in the production code.

## Dependencies

**Imports from production code:**
- `reasons_lib.api`: `_pg_dispatch`, `export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`
- `reasons_lib.cli`: `_backend_kwargs`, `_require_sqlite`

**Implicitly depends on (via mocking):**
- `reasons_lib.pg.PgApi` — the actual Postgres backend class. Every test patches this.
- `reasons_lib.api.export_network` — patched in the export-markdown tests.

**Nothing imports this file** — it's a leaf test module.

## Flow

1. Each test constructs a `MagicMock` standing in for a `PgApi` instance and configures its return value.
2. It patches `reasons_lib.pg.PgApi` so that constructing + entering the context manager yields the mock.
3. It calls the real API function with `pg_conninfo` and `project_id` kwargs — this triggers the PG dispatch path in the production code.
4. It asserts: (a) the mock method was called with expected args, and (b) the return value was passed through correctly.

For `_backend_kwargs` tests, the flow is: construct an `argparse.Namespace`, optionally set env vars, call `_backend_kwargs`, and assert the returned dict matches the expected backend selection.

## Invariants

- **`pg_conninfo` requires `project_id`.** If a PG connection string is present (via CLI or env) but `project_id` is absent, the system must exit. Tested in `TestBackendKwargs.test_pg_missing_project_id_exits`.
- **CLI flags override environment variables.** When both are present, the CLI wins. Tested in `test_cli_overrides_env`.
- **SQLite-only commands must reject PG.** `_require_sqlite` must `SystemExit` when PG is detected from either CLI or env. Tested in `TestRequireSqlite`.
- **File parsing happens before dispatch.** Import functions read and parse files in the API layer, then pass structured data to `PgApi`. The PgApi mock receives parsed dicts/strings, not file paths.
- **All API functions with PG paths return the PgApi method's return value unchanged.** Every test asserts the result matches the mock's return value.

## Error Handling

- **`FileNotFoundError` propagation**: `TestImportJsonDispatch.test_file_not_found` verifies that passing a nonexistent path raises `FileNotFoundError` — the API doesn't swallow it.
- **`SystemExit` for misconfiguration**: Missing `project_id` and sqlite-only commands with PG both produce `SystemExit`, tested with `pytest.raises(SystemExit)`.
- No tests for PG connection failures — those would belong in `test_pg.py` integration tests.

## Topics to Explore

- [file] `reasons_lib/api.py` — Contains `_pg_dispatch` and all the API functions whose PG dispatch paths are tested here; the branching logic is the core of what these tests validate
- [file] `reasons_lib/pg.py` — The `PgApi` class that these tests mock; understanding its interface explains why the mocks are shaped the way they are
- [function] `reasons_lib/cli.py:_backend_kwargs` — The argument resolution logic that negotiates between CLI flags, env vars, and SQLite defaults
- [file] `tests/test_pg.py` — The companion test file that tests `PgApi` itself (likely with a real or mocked database), complementing this dispatch-layer test
- [general] `pg-backend-wiring` — The overall strategy for making every `reasons` command dual-backend (SQLite + Postgres); the `*NoLongerBlocked` test classes trace the migration progress

## Beliefs

- `pg-dispatch-requires-project-id` — When `pg_conninfo` is provided (via CLI `--pg` or `REASONS_PG_CONNINFO` env var) without a corresponding `project_id`, the system calls `sys.exit()` rather than proceeding with an incomplete configuration
- `cli-flags-override-env-vars` — `_backend_kwargs` gives precedence to CLI `--pg`/`--project-id` flags over `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables when both are present
- `import-functions-parse-before-dispatch` — API import functions (`import_json`, `import_beliefs`, `import_agent`) read and parse files into structured data before delegating to `PgApi`, so `PgApi` never receives raw file paths
- `all-api-functions-support-pg-dispatch` — Every public API function tested here (`export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`) accepts `pg_conninfo`/`project_id` kwargs and routes to `PgApi` methods
- `require-sqlite-blocks-pg-for-guarded-commands` — `_require_sqlite` enforces that certain subcommands cannot run against a Postgres backend, checking both CLI flags and environment variables

