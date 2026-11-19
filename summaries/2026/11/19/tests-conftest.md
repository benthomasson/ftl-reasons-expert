---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/conftest.py

# Explanation: `tests/conftest.py`

## Purpose

`tests/conftest.py` is pytest's conventional shared-configuration module for the `tests/` tree. Pytest auto-discloses it during collection, so anything defined here — hooks, fixtures, module-level constants — becomes available to every test module in `tests/` (and subdirectories) **without** needing an explicit `import`. It is not "imported by" a test file in the Python sense; pytest loads it before any test module is collected.

Its concrete responsibilities in this repo are narrow and focused on the PostgreSQL-backed code path:

1. Register a custom `pg` marker so tests that need a live database can be marked and filtered (`-m pg`).
2. Expose a reusable `skip_no_pg` decorator for tests that must be skipped when `DATABASE_URL` is absent.
3. Provide the `pg_api` fixture — a per-test `PgApi` instance bound to a fresh, UUID-scoped `domain_id`, with automatic cleanup.

Everything else (HTTP mocks, temp directories, etc.) lives inside the individual test modules, not here.

---

## Key Components

### `pytest_configure(config)`
```python
def pytest_configure(config):
    config.addinivalue_line("markers", "pg: requires PostgreSQL (set DATABASE_URL)")
```
A standard pytest hook called once at startup. It registers the `pg` marker with a human-readable description, which silences `PytestUnknownMarkWarning` when a test uses `@pytest.mark.pg` and enables `pytest -m pg` / `-m "not pg"` selection. Note that registering the marker does **not** itself skip anything — skipping is done separately.

### `pg_available: bool`
```python
pg_available = bool(os.environ.get("DATABASE_URL"))
```
Evaluated once at import time. Empty strings and unset are both `False`. This is a snapshot — if a test mutates `os.environ["DATABASE_URL"]` mid-session, this flag does not update. Treat it as a session-level feature flag.

### `skip_no_pg`
```python
skip_no_pg = pytest.mark.skipif(not pg_available, reason="DATABASE_URL not set")
```
A ready-to-apply marker. Tests intended to be skipped in a DB-less environment decorate with `@skip_no_pg` and are reported as **skipped**, not failed or errored. This is the "opt-in" half of the skip strategy.

### `pg_api` fixture
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
A **generator fixture** with default function scope. Contract for consumers:

- Type: yields a `reasons.pg.PgApi` instance.
- Isolation: every test gets a brand-new `domain_id` (UUID4), so concurrent or sequential tests never collide on rows.
- Preconditions: `DATABASE_URL` is set in the environment, and the target DB schema is creatable (via `init_db()`).
- Postconditions: the test is fully torn down — all five `rms_*` tables are purged for that `domain_id`, transaction is committed, connection is closed.

The order of `DELETE FROM` matters: leaves first, then parents. `rms_propagation_log`, `rms_justifications`, and `rms_nogoods` reference rows in `rms_nodes`/`rms_network_meta`, so they must be cleared first or FK constraints will bite.

---

## Patterns & Idioms

- **Conftest auto-discovery**: No explicit import in `test_pg.py` (or any other test) is required — pytest injects `pg_api` by name. This is why the "Imported By" relationship in the manifest lists `tests/test_pg.py`: that's the fixture's consumer, not a Python import.
- **Lazy import**: `from reasons.pg import PgApi` is deferred into the fixture body rather than placed at the top of the file. This avoids importing `reasons.pg` (and any of its drivers like `psycopg`) at *collection* time, which would otherwise crash a DB-less CI run before any `skipif` could fire.
- **Generator fixture for setup/teardown**: `yield` splits setup from cleanup in one function; pytest guarantees the post-`yield` code runs even if the test raises.
- **Belt-and-braces skip**: two independent guards — the `skip_no_pg` marker for decorating tests, and a runtime `pytest.skip()` inside the fixture for tests that just request `pg_api` without decorating. Either path yields a clean "skipped" status.
- **Per-test scoping**: default function scope + unique `domain_id` = isolation without schema-level truncation.
- **Rollback-before-cleanup**: `api.conn.rollback()` at the top of teardown clears any aborted transaction state left by a failed `cur.execute` inside the test proper. Without it, the subsequent `DELETE` would fail with "current transaction is aborted".

---

## Dependencies

**Imports**
- `os` — for `DATABASE_URL` lookup.
- `uuid` — to mint a fresh `domain_id` per test.
- `pytest` — hooks, markers, fixtures, `pytest.skip`.
- `reasons.pg.PgApi` — imported lazily; must expose `conn`, `init_db()`, and `close()`.

**Consumers**
- Every test file under `tests/` benefits from the registered `pg` marker and the `skip_no_pg` decorator (whether or not they use them).
- `tests/test_pg.py` is the declared primary consumer of the `pg_api` fixture. `tests/test_pg_dispatch.py` also lives in the same directory and would inherit the auto-applied conftest, though the manifest lists only `test_pg.py` as directly coupled — worth verifying.

---

## Flow

1. **Collection start**: pytest discovers `tests/conftest.py`, imports it once.
2. **Import-time side effects**: `pg_available` is computed, `skip_no_pg` is built, and `pytest_configure` is called, registering the `pg` marker.
3. **Test execution with `pg_api`**:
   - Fixture body runs. If `DATABASE_URL` is missing → `pytest.skip()` (test shows as *skipped*).
   - Otherwise `PgApi(conninfo, domain_id)` is constructed, `init_db()` is called to ensure schema exists, then control yields to the test.
   - The test manipulates `api` freely.
   - Regardless of test outcome, teardown executes: `rollback()` → loop of `DELETE FROM <table> WHERE domain_id = %s` → `commit()` → `close()`.

---

## Invariants

- **Uniqueness**: `domain_id` is a UUID4, freshly generated per test invocation. No two tests share a domain.
- **Isolation guarantee**: after `pg_api` tears down, no row with that `domain_id` remains in any of the five `rms_*` tables listed.
- **FK-safe deletion order**: leaves-to-root ordering of `("rms_propagation_log", "rms_justifications", "rms_nogoods", "rms_network_meta", "rms_nodes")` is required; reordering will trigger foreign-key violations.
- **Connection lifecycle**: `api.close()` must be reachable after the fixture's teardown block; if a `DELETE` raises, the connection may leak. In practice the `rollback()` prelude makes this unlikely.
- **Pre-requisite for execution**: `DATABASE_URL` must be set *and* valid; otherwise the fixture short-circuits with `pytest.skip`.
- **Snapshot semantics**: `pg_available` reflects the environment at **collection time**, not per-test.

---

## Error Handling

- **Missing DB**: handled gracefully via `pytest.skip` inside the fixture and via the `skip_no_pg` decorator externally. This is the primary, intentional error path.
- **Constructor / `init_db()` failure** (bad URL, DB down, permission denied): exception propagates out of the fixture **before** `yield`. Pytest reports this as a fixture error (test does not run), and no teardown is executed.
- **Exception inside the test**: pytest propagates it, but generator-fixture semantics guarantee the post-`yield` teardown still runs. `rollback()` neutralizes any `psycopg` aborted-transaction state left by the failing test statement; then the deletes succeed.
- **Error inside teardown's `DELETE`**: not caught — raises out of the fixture, potentially preventing `api.close()`. This is a minor resource-leak risk in pathological cases; acceptable for test scaffolding.
- **No `try/finally`**: the code relies entirely on the generator fixture contract, not on explicit `try/finally`. If someone rewrites `pg_api` as a plain function, this guarantee is lost.

---

## Topics to Explore

- [file] `reasons/pg.py` — Source of `PgApi`; inspect `init_db`, `conn`, `close`, and how `domain_id` scopes every query to confirm the isolation strategy assumed here.
- [file] `tests/test_pg.py` — The declared consumer of `pg_api`; useful for seeing real usage patterns, expected table names, and whether teardown relies on any additional state.
- [file] `tests/test_pg_dispatch.py` — Check whether it also uses `pg_api` (conftest auto-applies regardless), and whether its needs are captured by the same fixture.
- [file] `pyproject.toml` — Confirm `[tool.pytest.ini_options]` settings (`testpaths`, `markers`, `addopts`) and whether the `pg` marker is also declared there; duplicate marker registration is harmless but worth knowing.
- [general] `pytest-fixture-lifecycle` — Review pytest's generator-fixture semantics (setup/yield/teardown, function vs module scope, interaction with `pytest.skip` in setup) to fully understand the failure modes described above.

---

## Beliefs

- `conftest-fixtures-are-auto-injected` — The `pg_api` fixture is exposed to all test modules under `tests/` without any explicit `import`, via pytest's conftest auto-discovery.
- `pg-marker-requires-pytest-configure` — The `pg` marker is registered by `pytest_configure`; without that hook, `@pytest.mark.pg` would trigger unknown-marker warnings.
- `pg-api-fixture-requires-database-url` — `pg_api` skips (does not fail) when `DATABASE_URL` is unset, via `pytest.skip` inside the fixture body.
- `ldomain-isolation-uses-uuid4` — Each `pg_api` invocation mints a UUID4 `domain_id`, and all rows created during the test are scoped to that ID, guaranteeing cross-test isolation without table truncation.
- `cleanup-order-respects-foreign-keys` — The teardown `DELETE` loop orders tables leaves-first (`rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_nodes`) to satisfy foreign-key constraints.
- `rollback-prevents-aborted-transaction-errors` — `api.conn.rollback()` at the start of teardown resets failed-transaction state left by the test so the subsequent `DELETE`s succeed.
- `pg-api-import-is-deferred` — `from reasons.pg import PgApi` is executed inside the fixture body (not at module top) to avoid importing DB drivers at collection time when `DATABASE_URL` is absent.
