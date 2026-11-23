---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Inspect `reasons.pg.PgApi` to see whether constructor or `init_db()` failures leave open connections/resources.

## Verdict

`PgApi` is not fully exception-safe around resource acquisition/cleanup.

- **Constructor:** Mostly safe for `_require_psycopg()` / `psycopg.connect()` failures, because no connection has been assigned yet. But it is **not safe after a successful connect** if a later constructor statement fails, notably `str(domain_id)`. There is no `try/except`, no `close()`, and no `__del__`.
- **`init_db()`:** Cursor resources are cleaned up by `with self.conn.cursor() as cur`, but the **connection is not closed and not rolled back by `init_db()` on failure**. If called directly, the connection stays open and the transaction may be aborted. If called inside `with PgApi(...)`, `__exit__` will rollback and close, but only because of the context manager.

## Constructor behavior

Relevant code: `reasons/pg.py:104-112`

```python
def __init__(self, conninfo, domain_id):
    _require_psycopg()
    if isinstance(conninfo, str):
        self.conn = psycopg.connect(conninfo, autocommit=False)
        self._owns_conn = True
    else:
        self.conn = conninfo
        self._owns_conn = False
    self.domain_id = str(domain_id)
```

Failure cases:

1. **`_require_psycopg()` raises**
   - Happens before any connection is created.
   - No `PgApi`-owned resource exists.

2. **`psycopg.connect(...)` raises**
   - No connection is assigned to `self.conn`.
   - `PgApi` has nothing to close.
   - Any partial-connect cleanup is psycopg’s responsibility.

3. **`psycopg.connect(...)` succeeds, then `str(domain_id)` raises**
   - `self.conn` is already open.
   - `self._owns_conn` is already `True` for string `conninfo`.
   - `self.domain_id = str(domain_id)` is outside any cleanup block.
   - If `str(domain_id)` raises, the constructor does not complete, and no explicit `close()` is called.
   - This can leave an owned PostgreSQL connection open unless psycopg/GC finalization happens to clean it up, which is not an explicit guarantee in this class.

4. **Non-string `conninfo`**
   - `PgApi` treats it as an externally owned connection and sets `_owns_conn = False`.
   - If a later constructor step fails, `PgApi` would not close it anyway; the caller remains responsible.

So the constructor leak window is narrow but real: **only after `psycopg.connect()` succeeds and before `__init__` returns.** The main visible trigger is `str(domain_id)` failing.

## `init_db()` behavior

Relevant code: `reasons/pg.py:128-163`

```python
def init_db(self, project_name: str = ""):
    from .metadata import SCHEMA_VERSION
    with self.conn.cursor() as cur:
        cur.execute(SCHEMA)
        cur.execute(INDEXES)
        ...
    self.conn.commit()
    return {"domain_id": self.domain_id, "created": True}
```

Failure cases:

1. **`from .metadata import SCHEMA_VERSION` raises**
   - No cursor is opened.
   - The connection remains open.
   - `init_db()` does not close it or roll it back.

2. **Any SQL inside `with self.conn.cursor() as cur` raises**
   - The cursor context manager closes the cursor.
   - The connection itself remains open.
   - Because `autocommit=False`, the transaction may be left aborted.
   - `init_db()` has no `except` / `finally` that calls `self.conn.rollback()` or `self.close()`.

3. **`self.conn.commit()` raises**
   - The connection remains open.
   - No rollback or close is attempted in `init_db()`.

4. **Successful `init_db()`**
   - The connection remains open intentionally so the same `PgApi` instance can be reused.
   - That is not a leak by itself; it is the expected ownership model.

So for `init_db()`, the key distinction is:

- **Cursor resource:** cleaned up by `with self.conn.cursor()`.
- **Connection resource:** not cleaned up on failure.
- **Transaction state:** not rolled back by `init_db()` on failure.

If the caller writes:

```python
with PgApi(conninfo, domain_id) as api:
    api.init_db()
```

then `__exit__` at `reasons/pg.py:121-124` performs rollback and close on exception. Without that context manager or an explicit `try/finally`, an `init_db()` failure leaves the connection open.

## Context-manager cleanup

Relevant code: `reasons/pg.py:114-124`

```python
def close(self):
    if self._owns_conn:
        self.conn.close()

def __enter__(self):
    return self

def __exit__(self, exc_type, *exc):
    if exc_type is not None:
        self.conn.rollback()
    self.close()
```

The context manager is the only automatic cleanup path shown:

- On exception: `rollback()` then `close()`.
- On success: `close()` only.

One additional hazard: if `self.conn.rollback()` raises inside `__exit__`, then `self.close()` is skipped because there is no `try/finally`. That is another lifecycle edge case, though not asked specifically about constructor/`init_db()`.

## Test coverage note

The provided `tests/test_pg.py` excerpt shows functional tests using the `pg_api` fixture, but no constructor-failure or `init_db()`-failure cleanup tests in the first 400 lines. The fixture definition is in `tests/conftest.py`, which is not included in the observations. So we cannot verify from the excerpt whether the test fixture always closes or rolls back connections.

Also note: the observation greps for lifecycle methods and connection usage returned zero matches, which conflicts with the direct source read. The direct `reasons/pg.py` content is sufficient here; the zero-match grep results appear unreliable or incomplete.

## Topics to Explore

- [function] `reasons/pg.py:PgApi.__init__` — Review the post-connect failure window and whether `str(domain_id)` can raise in realistic call paths.
- [function] `reasons/pg.py:PgApi.init_db` — Check whether failed DDL should be rolled back immediately inside `init_db()` rather than left to the caller.
- [file] `tests/conftest.py` — Inspect the `pg_api` fixture to see whether tests consistently close connections and rollback aborted transactions.
- [general] `psycopg-connection-lifecycle` — Confirm psycopg v3 behavior for connection close, GC finalization, and aborted transactions after failed statements.
- [file] `reasons/pg.py` — Search later methods for similar `with self.conn.cursor()` / `commit()` patterns that omit rollback/close on failure.

## Beliefs

- `constructor-not-exception-safe-after-connect` — `PgApi.__init__` has no `try/except` or `close()` after `psycopg.connect()`, so a later `str(domain_id)` failure can leave an owned connection open.
- `init-db-does-not-close-connection-on-failure` — `init_db()` closes the cursor via context manager but never closes `self.conn` on exception.
- `init-db-does-not-rollback-on-failure` — `init_db()` commits only on success and has no exception handler that calls `rollback()`.
- `close-only-closes-owned-connections` — `PgApi.close()` closes the connection only when `_owns_conn` is true, which happens only for string `conninfo`.
- `context-manager-rolls-back-then-closes` — `PgApi.__exit__` calls `rollback()` followed by `close()` when an exception is raised.
