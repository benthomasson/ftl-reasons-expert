---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/conftest.py

# `tests/conftest.py`

## Purpose

This is the **pytest shared-fixture entry point** for the entire `tests/` directory. In pytest's architecture, a `conftest.py` file is auto-loaded by the test runner before any test in that directory (or subdirectories) executes. Its job here is narrow but critical: it provides the **PostgreSQL test infrastructure** — registering a `pg` marker for test categorization and exposing a `pg_api` fixture that hands each test an isolated database session.

Because conftest fixtures are visible to *every* test file in the directory tree (not just `test_pg.py`), any test anywhere under `tests/` can request `pg_api` or the `skip_no_pg` marker without importing anything extra.

## Key Components

### `pytest_configure(config)`
```python
def pytest_configure(config):
    config.addinivalue_line("markers", "pg: requires PostgreSQL (set DATABASE_URL)")
```
Registers the `pg` marker with pytest. Without this, any test decorated with `@pytest.mark.pg` would trigger a `PytestUnknownMarkWarning`. This is the standard idiom for custom markers.

### `pg_available` / `skip_no_pg` (module-level constants)
```python
pg_available = bool(os.environ.get("DATABASE_URL"))
skip_no_pg = pytest.mark.skipif(not pg_available, reason="DATABASE_URL not set")
```
- **`pg_available`** is evaluated **once at import time**. It's a boolean snapshot of whether `DATABASE_URL` was set when pytest collected the test suite. This means you cannot toggle it mid-run without restarting pytest.
- **`skip_no_pg`** is a pre-built `skipif` marker. Tests can write `@pytest.mark.skipif(not pg_available, ...)` inline, or use `@skip_no_pg` directly. It's a convenience for tests that want a one-line skip guard.

### `pg_api` fixture (the core)
```python
@pytest.fixture
def pg_api():
    if not pg_available:
        pytest.skip("DATABASE_URL not set")

    from reasons.pg import PgApi
    conninfo = os.environ["DATABASE_URL"]
    domain_id = str(uuid.uuid4())
    api = PgApi(conninfo, domain_id)
    api.init_db()

    yield api

    api.conn.rollback()
    with api.conn.cursor() as cur:
        for table in ("rms_propagation_log", "rms_justifications",
                      "rms_nogoods", "rms_network_meta", "rms_nodes"):
            cur.execute(f"DELETE FROM {table} WHERE domain_id = %s", (domain_id,))
    api.conn.commit()
    api.close()
```

| Phase | What happens |
|---|---|
| **Guard** | If `pg_available` is falsy, the fixture skips the test immediately. |
| **Lazy import** | `PgApi` is imported *inside* the fixture body, not at module top-level. This avoids an `ImportError` at collection time if the `psycopg2`/`pg` driver isn't installed and the test would be skipped anyway. |
| **Isolation** | A fresh `uuid.uuid4()` is generated per test invocation. Every row written by the test is scoped to this `domain_id`, so parallel or sequential tests never see each other's data. |
| **Setup** | `api.init_db()` is called — presumably creates tables / extensions if they don't exist (idempotent). |
| **Teardown (yield)** | 1. `rollback()` — discards any uncommitted transaction left open by the test (defensive). 2. Deletes rows from 5 tables in FK-safe order (children before parents): `rms_propagation_log` → `rms_justifications` → `rms_nogoods` → `rms_network_meta` → `rms_nodes`. 3. `commit()` to persist the deletions. 4. `close()` to release the connection back to the pool / close it. |

## Patterns

- **Pytest yield-fixture with teardown**: The `yield` splits setup (above) from teardown (below). Pytest guarantees the teardown runs even if the test raises.
- **Conditional skip via env var**: The "opt-in heavy dependency" pattern — PG tests only run when `DATABASE_URL` is present. This lets CI run the full suite without a PG instance, and developers can opt in locally.
- **UUID-based row scoping**: Instead of `TRUNCATE` (which would block other sessions and require a superuser), each test gets a unique `domain_id`. Teardown deletes *only* its own rows. This is safe for concurrent test runs against a shared database.
- **Lazy import inside fixture**: Keeps the import graph flat at collection time. If `reasons.pg` has a heavy or missing dependency, it only blows up when a test actually requests `pg_api`.
- **Marker registration**: Follows the `pyproject.toml`-adjacent convention of declaring markers at configure-time rather than relying on `ini_options`.

## Dependencies

**Imports (this file → them):**
- `os` — read `DATABASE_URL` from the environment.
- `uuid` — generate the per-test `domain_id`.
- `pytest` — fixture decorator, `skip()`, `mark.skipif`, `config.addinivalue_line`.
- `reasons.pg.PgApi` — **lazy**, imported inside the fixture. This is the only application-code dependency.

**Imported by (them → this file):**
- Directly: `tests/test_pg.py` (the sole explicit consumer per the import graph).
- Indirectly: **every test file** in `tests/` can request `pg_api` or use `skip_no_pg` without importing conftest, because pytest injects conftest fixtures into the test namespace automatically. In practice, only PG-dependent tests use it.

## Flow

```
pytest collects tests/
  │
  ├─ conftest.py is imported
  │    ├─ pg_available = bool(os.environ.get("DATABASE_URL"))   # once, at import
  │    ├─ skip_no_pg = pytest.mark.skipif(...)                   # once, at import
  │    └─ pytest_configure() runs → registers "pg" marker
  │
  └─ for each test requesting `pg_api`:
       │
       ├─ pg_available is False?
       │    └─ pytest.skip("DATABASE_URL not set")   ← test marked skipped
       │
       ├─ import reasons.pg.PgApi                     ← lazy
       ├─ domain_id = uuid4()                         ← unique per test
       ├─ api = PgApi(DATABASE_URL, domain_id)
       ├─ api.init_db()                               ← ensure schema exists
       │
       ├─ yield api    ←── test body runs here ──►
       │
       ├─ api.conn.rollback()                         ← discard any dirty txn
       ├─ DELETE FROM rms_propagation_log  WHERE domain_id = ?
       ├─ DELETE FROM rms_justifications   WHERE domain_id = ?
       ├─ DELETE FROM rms_nogoods          WHERE domain_id = ?
       ├─ DELETE FROM rms_network_meta     WHERE domain_id = ?
       ├─ DELETE FROM rms_nodes            WHERE domain_id = ?
       ├─ api.conn.commit()
       └─ api.close()
```

The data flow is strictly **write-scoped to a UUID**: the test writes rows tagged with its `domain_id`, and teardown deletes exactly those rows. No other session's data is touched.

## Invariants

1. **Test isolation via UUID**: Every `pg_api` invocation gets a *new* `uuid.uuid4()`. Two concurrent tests can never share a `domain_id` (collision probability ≈ 0).
2. **FK-safe deletion order**: The 5 `DELETE` statements go from most-dependent to least-dependent (`rms_nodes` is the root; `rms_propagation_log` is a leaf). Reversing this order would violate foreign-key constraints.
3. **Rollback-before-delete**: The `rollback()` call ensures the connection is in a clean state before the cleanup `DELETE`s run. If the test left an open (and failed) transaction, the `DELETE` statements would be part of that aborted transaction and silently not commit.
4. **Marker registration**: The `pg` marker is registered *before* any test is collected, so `@pytest.mark.pg` decorators on test functions are recognized.
5. **`pg_available` is immutable after import**: Because it's a module-level `bool`, it cannot change during a test session. If `DATABASE_URL` is set *after* pytest starts, it has no effect.

## Error Handling

| Scenario | Behavior |
|---|---|
| `DATABASE_URL` not set | `pytest.skip()` is called inside the fixture. The test is reported as **skipped**, not failed. No exception escapes. |
| `PgApi` constructor fails (bad DSN, connection refused) | Exception propagates out of the fixture setup → pytest reports the test as **errored** (not failed). The teardown (below `yield`) does **not** run because `yield` was never reached. |
| `api.init_db()` fails (e.g., missing permission) | Same as above — setup error, no teardown. |
| Test body raises | Pytest catches the test exception. Teardown **still runs** (that's the contract of yield-fixtures). The `rollback()` + `DELETE` cleanup executes. |
| A `DELETE` in teardown fails (e.g., connection dropped) | Exception propagates as a **fixture teardown error**. Pytest reports it alongside the original test result. The remaining `DELETE`s and `commit()`/`close()` are **not** reached — you may leak rows for that `domain_id`. There is no `try/finally` guard around the cleanup loop. |
| `api.conn.rollback()` on an already-clean connection | No-op. Safe. |

The one gap worth noting: there is no `try/except` around the teardown `DELETE` loop. A mid-loop failure (say, the connection is killed after the second delete) would leave orphan rows in the later tables. In practice this is rare, but a `try/finally` wrapping at least the `close()` would make teardown more robust.

## Topics to Explore

- [file] `reasons/pg.py` — The `PgApi` class this fixture instantiates. Understanding its constructor contract, `init_db()` semantics, and the schema it manages (the 5 `rms_*` tables) is essential for interpreting the teardown logic.
- [file] `tests/test_pg.py` — The primary consumer of `pg_api`. Shows the expected usage pattern: how tests request the fixture, what operations they perform, and how they rely on the `domain_id` scoping.
- [general] `fk-safe-table-ordering` — The 5 `rms_*` tables and their foreign-key relationships. Knowing the full dependency graph (not just the deletion order in conftest) helps when adding new tables that must be included in teardown.
- [file] `pyproject.toml` — May contain pytest configuration (e.g., `addopts`, `markers`, `testpaths`) that interacts with the `pytest_configure` registration in this file.
- [function] `reasons/pg.py:init_db` — Whether it's idempotent (CREATE TABLE IF NOT EXISTS) or destructive. This determines whether the fixture is safe to run against a database that already has data from other domains.

## Beliefs

- `pg-fixture-scopes-by-uuid` — Each `pg_api` fixture invocation generates a unique `uuid.uuid4()` as `domain_id`, guaranteeing that no two tests (even running concurrently) share database rows.
- `pg-teardown-deletes-in-fk-order` — The teardown deletes rows in the order `rms_propagation_log → rms_justifications → rms_nogoods → rms_network_meta → rms_nodes`, which is child-before-parent and is required to avoid foreign-key constraint violations.
- `pg-fixture-lazy-imports-pgapi` — `reasons.pg.PgApi` is imported inside the fixture body (not at module level), so a missing `psycopg2` dependency does not break pytest collection for tests that don't need PostgreSQL.
- `pg-available-frozen-at-import` — `pg_available` is evaluated once at conftest import time; setting `DATABASE_URL` after pytest has started has no effect on whether PG tests run or skip.
- `pg-teardown-lacks-finally-guard` — The cleanup `DELETE` loop and `close()` are not wrapped in `try/finally`, so a mid-teardown exception (e.g., connection loss) will skip remaining deletes and the connection close, potentially leaking orphan rows.
