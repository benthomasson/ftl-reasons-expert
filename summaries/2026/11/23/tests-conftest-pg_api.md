---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: pg_api in tests/conftest.py

# `pg_api`

## Purpose

`pg_api` is a pytest fixture that provides a test-scoped `PgApi` instance connected to PostgreSQL via `DATABASE_URL`. Each fixture invocation creates a fresh UUID `domain_id`, initializes the database through `PgApi.init_db()`, yields the API object to the test, and then attempts to delete all rows associated with that `domain_id` from a fixed set of `rms_*` tables before closing the connection.

It exists to isolate PostgreSQL-backed tests from each other without creating a separate database per test. Isolation is domain-based: every test gets a unique `domain_id`, and teardown removes rows for that domain only.

## Contract

### Preconditions

- `DATABASE_URL` must have been set when `tests/conftest.py` was imported. `pg_available` is computed once at module import time:
  ```python
  pg_available = bool(os.environ.get("DATABASE_URL"))
  ```
- If `DATABASE_URL` is not available, the fixture calls `pytest.skip(...)`.
- `reasons.pg.PgApi` must be importable.
- `PgApi` must support:
  - `PgApi(conninfo, domain_id)`
  - `.init_db()`
  - `.conn` as a psycopg-style connection with `.rollback()`, `.cursor()`, `.commit()`, and the API must have `.close()`
- The target database must contain the tables listed in teardown:
  - `rms_propagation_log`
  - `rms_justifications`
  - `rms_nogoods`
  - `rms_network_meta`
  - `rms_nodes`
- Each listed table must have a `domain_id` column compatible with the generated UUID string.

### Postconditions

On successful teardown:

- Rows in the five listed tables whose `domain_id` equals the generated UUID are deleted.
- The cleanup transaction is committed.
- The `PgApi` instance is closed.

### Invariants

- The fixture is function-scoped by default because `@pytest.fixture` has no `scope=...`.
- A new `PgApi` and a new UUID `domain_id` are created per test that requests this fixture.
- Cleanup uses the local `domain_id` variable, not `api.domain_id`. If the test mutates or ignores `api.domain_id`, teardown still deletes by the original generated UUID.

### Important assumptions not enforced by the type system

- `DATABASE_URL` remains present after module import. If it was set at import time but removed before the test runs, `os.environ["DATABASE_URL"]` raises `KeyError` because `pg_available` is still `True`.
- Conversely, if `DATABASE_URL` is set after module import, the fixture remains skipped because `pg_available` was already computed as `False`.
- The test uses the fixture's `domain_id` for any rows it wants cleaned up.
- The listed tables are the only tables that need cleanup, or all other dependent data cascades or is otherwise removed.
- The cleanup `DELETE` order is compatible with foreign-key constraints.
- `api.conn` remains open and usable until teardown. If the test closes or replaces it, teardown can fail.
- `api.init_db()` is safe/idempotent enough to run per test and does not conflict destructively with concurrent tests.
- `api.close()` is reached only if rollback, deletes, and commit all succeed.

## Parameters

`pg_api` takes no explicit function parameters. Pytest injects it when a test declares `pg_api` as an argument.

Implicit inputs:

- `DATABASE_URL`: environment variable used as the PostgreSQL connection string.
- `pg_available`: module-level boolean computed from `DATABASE_URL` at import time.
- `uuid.uuid4()`: generates the per-test `domain_id`.

Edge cases:

- Missing `DATABASE_URL`: test is skipped.
- Invalid `DATABASE_URL`: connection or `init_db()` failure propagates as a setup error.
- Connection closed during the test: teardown rollback/delete may fail.
- Test writes rows with a different `domain_id`: those rows are not cleaned up.
- Test writes to tables not listed in the cleanup tuple: those rows are not cleaned up.

## Return Value

Because this is a generator fixture using `yield`, pytest provides the yielded `PgApi` instance to the test. The fixture itself does not return a value to the test in the ordinary function sense.

If `DATABASE_URL` is unavailable, the fixture raises pytest's skip signal via `pytest.skip(...)`, and the test is reported as skipped rather than receiving a `PgApi`.

The cleanup path returns nothing. If cleanup fails, the failure is reported as a teardown error.

## Algorithm

1. At module import time, compute:
   ```python
   pg_available = bool(os.environ.get("DATABASE_URL"))
   ```
2. When a test requests `pg_api`, pytest invokes the fixture.
3. Check `pg_available`. If false, call:
   ```python
   pytest.skip("DATABASE_URL not set")
   ```
4. Lazily import `PgApi`:
   ```python
   from reasons.pg import PgApi
   ```
5. Read the connection string:
   ```python
   conninfo = os.environ["DATABASE_URL"]
   ```
6. Generate a fresh domain identifier:
   ```python
   domain_id = str(uuid.uuid4())
   ```
7. Construct the API:
   ```python
   api = PgApi(conninfo, domain_id)
   ```
8. Initialize the database:
   ```python
   api.init_db()
   ```
9. Yield `api` to the test. The test runs while the fixture is suspended at `yield`.
10. After the test finishes, begin teardown by rolling back any open transaction:
    ```python
    api.conn.rollback()
    ```
11. Open a cursor and delete rows from each table where `domain_id` matches:
    ```python
    with api.conn.cursor() as cur:
        for table in (...):
            cur.execute(f"DELETE FROM {table} WHERE domain_id = %s", (domain_id,))
    ```
12. Commit the cleanup transaction:
    ```python
    api.conn.commit()
    ```
13. Close the API/connection:
    ```python
    api.close()
    ```

## Side Effects

- Reads `DATABASE_URL` from the environment.
- Imports `reasons.pg.PgApi`.
- Generates a UUID.
- Opens a PostgreSQL connection through `PgApi`.
- Calls `api.init_db()`, which may create or modify database schema/data.
- Mutates database state through the test itself.
- On teardown:
  - Rolls back any open transaction on `api.conn`.
  - Issues five `DELETE` statements against shared PostgreSQL tables.
  - Commits those deletes.
  - Closes the API/connection if cleanup succeeds.

## Error Handling

- Missing `DATABASE_URL` at module import: fixture skips the test via `pytest.skip`.
- `DATABASE_URL` removed after import while `pg_available` is true: `os.environ["DATABASE_URL"]` raises `KeyError`.
- Import failure for `reasons.pg.PgApi`: propagates as a setup error.
- Constructor failure: propagates as a setup error.
- `init_db()` failure: propagates as a setup error.
- Test failure: pytest still attempts fixture teardown.
- `api.conn.rollback()` failure: propagates as a teardown error; subsequent deletes and `api.close()` are skipped.
- `DELETE` failure: propagates as a teardown error; `commit()` and `api.close()` are skipped.
- `commit()` failure: propagates as a teardown error; `api.close()` is skipped.
- Cleanup is not wrapped in `try/finally`, so `api.close()` is not guaranteed on teardown errors.
- Table names are interpolated with an f-string, but they come from a hard-coded tuple. This is safe as written, but becomes risky if the tuple is ever populated dynamically.

## Usage Patterns

Typical usage is to request the fixture by name:

```python
def test_something(pg_api):
    pg_api.some_operation()
```

Tests should:

- Use `pg_api` rather than constructing their own `PgApi`.
- Avoid closing or replacing `pg_api.conn` during the test.
- Use the fixture's domain scoping for data that should be cleaned up.
- Avoid relying on rows from other domains.
- Avoid assuming cleanup covers tables outside the hard-coded tuple.
- Remember that the `pg` marker is only registered by `pytest_configure`; it does not by itself skip tests. The fixture does the skipping. Tests not using this fixture need to use `skip_no_pg` or another skip mechanism if they require PostgreSQL.

## Dependencies

- Standard library:
  - `os`
  - `uuid`
- Third-party:
  - `pytest`
- Internal:
  - `reasons.pg.PgApi`
- External:
  - PostgreSQL reachable via `DATABASE_URL`
- Database objects:
  - Tables `rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_nodes`
  - A `domain_id` column on each table

## Topics to Explore

- [file] `tests/test_pg.py` — Shows how `pg_api` is requested and what database operations are exercised.
- [function] `reasons/pg.py:PgApi` — Defines the constructor, `init_db`, connection lifecycle, and domain-scoped APIs that this fixture assumes.
- [general] `postgres-test-isolation` — Explores whether `domain_id` cleanup fully isolates tests that share the same tables and database.
- [general] `pytest-yield-fixtures` — Explains setup/yield/teardown semantics and how setup, test, and teardown failures are reported.
- [general] `rms-table-foreign-keys` — Determines whether the cleanup delete order is valid under the actual schema constraints and cascades.

## Beliefs

- `pg-api-requires-database-url` — `pg_api` skips unless `DATABASE_URL` was set when `tests/conftest.py` was imported.
- `pg-api-is-function-scoped` — `pg_api` uses `@pytest.fixture` with no scope, so pytest creates a fresh `PgApi` and UUID per test.
- `pg-api-cleanup-deletes-by-domain-id` — Teardown deletes rows from five hard-coded `rms_*` tables matching the generated `domain_id`.
- `pg-api-teardown-rolls-back-before-delete` — Teardown calls `api.conn.rollback()` before issuing cleanup `DELETE` statements.
- `pg-api-close-not-guaranteed-on-teardown-errors` — `api.close()` is skipped if rollback, cleanup deletes, or commit raise an exception.
