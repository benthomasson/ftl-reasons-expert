# File: tests/conftest.py

**Date:** 2026-04-29
**Time:** 17:15



# `tests/conftest.py` — Shared Test Fixtures

## Purpose

This is pytest's automatic fixture-discovery file for the `tests/` package. Its sole responsibility is providing **PostgreSQL test infrastructure**: it registers a custom marker (`pg`), exposes a skip sentinel for tests that need Postgres, and supplies an isolated `PgApi` fixture that each test gets a fresh, uniquely-scoped database partition — cleaned up afterward regardless of test outcome.

## Key Components

### `pytest_configure(config)`

A pytest hook that registers the `pg` marker at collection time, so `@pytest.mark.pg` doesn't trigger an "unknown marker" warning. This is a declarative registration — it doesn't enforce anything itself.

### `pg_available` / `skip_no_pg`

Two module-level constants evaluated once at import time:

- **`pg_available`**: `True` if `DATABASE_URL` is in the environment; `False` otherwise.
- **`skip_no_pg`**: A `pytest.mark.skipif` decorator that test functions can apply (`@skip_no_pg`) to declaratively skip when Postgres is unavailable. This is the "decorator-style" counterpart to the imperative `pytest.skip()` inside the fixture.

### `pg_api` fixture

The main fixture. Contract:

- **Yields** a fully initialized `PgApi` instance backed by a unique `project_id` (UUID v4).
- **Precondition**: `DATABASE_URL` must be set — if not, the test is skipped imperatively.
- **Postcondition**: All rows inserted by the test into the five tracked tables (`rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_nodes`) are deleted, scoped by `project_id`. The connection is committed and closed.

## Patterns

**Tenant-isolation via `project_id`**: Rather than creating/dropping a throwaway database or using transactions for rollback, the fixture partitions data by a random UUID. This lets tests run against a shared database (even concurrently) without colliding, and cleanup is a set of targeted `DELETE` statements.

**Defensive rollback before cleanup**: The teardown calls `api.conn.rollback()` before the `DELETE` cascade. This handles the case where a test left the connection in a failed-transaction state (Postgres aborts all statements after an error until the transaction is rolled back). Without this, the cleanup `DELETE`s would themselves fail.

**Two skip mechanisms**: The module provides both `skip_no_pg` (declarative marker for test functions) and the imperative `pytest.skip()` inside the fixture. The fixture's skip is a safety net — if someone requests `pg_api` without also applying the marker, the test still skips gracefully.

## Dependencies

**Imports**: `os`, `uuid`, `pytest`, and conditionally `reasons_lib.pg.PgApi` (imported inside the fixture body to avoid import errors when Postgres dependencies aren't installed).

**Depended on by**: `tests/test_pg.py` uses the `pg_api` fixture. Any new test file that requests `pg_api` or references `skip_no_pg` automatically depends on this file.

## Flow

1. At **collection time**: `pytest_configure` registers the `pg` marker; `pg_available` and `skip_no_pg` are evaluated.
2. When a test **requests `pg_api`**: the fixture checks `pg_available`, imports `PgApi`, creates a UUID-scoped instance, calls `init_db()`, and yields it to the test.
3. After the test **finishes** (pass or fail): rollback any broken transaction, delete all rows for this `project_id` across five tables in dependency order (propagation log first, nodes last — foreign-key safe), commit, close.

## Invariants

- **Each test gets a unique `project_id`** — UUID v4 guarantees no collisions between concurrent or sequential tests.
- **Cleanup deletes exactly five tables** — if the schema adds a new `project_id`-scoped table, the fixture must be updated or test data will leak.
- **The `PgApi` import is deferred** — the module loads cleanly even without `psycopg2` installed, so SQLite-only test runs aren't broken by import errors.

## Error Handling

- Missing `DATABASE_URL`: handled by `pytest.skip()`, which raises a special exception pytest catches — the test is marked "skipped", not "failed."
- Failed transaction state at teardown: the `api.conn.rollback()` call absorbs this, allowing the cleanup `DELETE`s to proceed.
- No try/except around cleanup: if the cleanup itself fails (e.g., table doesn't exist), the fixture teardown will error and pytest will report it — this is intentional, since a broken cleanup is a real problem that shouldn't be silenced.

---

## Topics to Explore

- [file] `reasons_lib/pg.py` — The `PgApi` class whose lifecycle this fixture manages; understanding its `init_db()`, connection handling, and schema is essential context.
- [file] `tests/test_pg.py` — The primary consumer of `pg_api`; shows how the fixture is used in practice and what operations are tested against Postgres.
- [function] `reasons_lib/pg.py:PgApi.init_db` — What schema setup happens during `init_db()` and whether it's idempotent (matters for concurrent test runs).
- [general] `project-id-multitenancy` — The `project_id` partitioning pattern used here and in `PgApi` — how the five tables reference it and what foreign-key relationships exist between them.
- [file] `reasons_lib/storage.py` — The SQLite storage counterpart; comparing it with `PgApi` clarifies which tests need Postgres and which don't.

## Beliefs

- `pg-fixture-uses-uuid-isolation` — Each `pg_api` fixture instance partitions its data by a unique UUID v4 `project_id`, not by transactions or ephemeral databases.
- `pg-cleanup-covers-five-tables` — Teardown deletes from exactly `rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, and `rms_nodes` — a new project-scoped table would leak test data.
- `pg-import-is-deferred` — `PgApi` is imported inside the fixture body, not at module top-level, so the test suite loads without Postgres dependencies when running SQLite-only tests.
- `pg-teardown-rollback-before-delete` — The fixture calls `conn.rollback()` before cleanup deletes to recover from any failed-transaction state left by the test.

