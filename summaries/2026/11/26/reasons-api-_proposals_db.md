---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: _proposals_db in reasons/api.py

## `_proposals_db`

### Purpose

`_proposals_db` is an internal helper that opens and prepares an SQLite connection for the proposal subsystem. It exists so every proposal-related API call can rely on the `proposals` schema being present before it reads or writes proposals.

It is not the same as `_with_network`: it does not load a `Network`, does not recompute truth values, and does not use the network cache. It only manages the proposal table stored in the same SQLite database.

It is used by proposal APIs such as `propose_retraction`, `propose_supersession`, `propose_addition`, `propose_nogood`, `list_proposals`, `show_proposal`, `accept_proposal`, `reject_proposal`, and `withdraw_proposal`.

### Contract

Preconditions:

- `db_path` must be a usable SQLite database path, or a path SQLite can create.
- The parent directory must exist and be writable if the database file does not exist.
- `PROPOSALS_SCHEMA` must be valid SQL and must create the `proposals` table.
- `PROPOSALS_SCHEMA` should be idempotent because this helper executes it on every call.
- The existing `proposals` table, if present, must already have all columns expected by callers except possibly `tags_json`.

Postconditions on success:

- Returns an open `sqlite3.Connection`.
- The `proposals` table exists.
- The `tags_json` column exists, with default `'[]'` if it had to be added.
- Schema changes are committed before the connection is returned.
- Foreign key enforcement is enabled for that connection.
- The database journal mode is set to WAL, or SQLite's closest applicable mode for special databases such as `:memory:`.

Invariants and assumptions not enforced by the type system:

- The caller owns the returned connection and must close it.
- The caller must commit its own INSERT/UPDATE changes; `_proposals_db` only commits schema setup/migration.
- The column check is an exact string comparison against `"tags_json"` after `PRAGMA table_info`.
- Only `tags_json` is migrated. Missing other columns are not repaired.
- Default `sqlite3` thread checking applies; the connection should normally be used on the thread that created it.
- No `busy_timeout` is configured, so concurrent writers can surface `database is locked`.
- If the database file does not exist, simply calling this helper can create it.

### Parameters

- `db_path: str` — Path to the SQLite database.

Edge cases:

- A non-existent database file is created by `sqlite3.connect`.
- A non-existent parent directory causes `sqlite3.OperationalError`.
- A read-only database or directory can fail during `PRAGMA journal_mode=WAL`, `executescript`, or `ALTER TABLE`.
- `:memory:` works as an SQLite special database, but separate calls create separate in-memory databases, which is usually not what proposal callers want.
- Passing a `pathlib.Path` works at runtime despite the `str` annotation because `sqlite3.connect` accepts path-like objects.
- An existing malformed schema can fail later because only `tags_json` is checked.

### Return Value

Returns a live `sqlite3.Connection`.

It returns normally only if all setup steps succeed. It never returns `None` or an error dict. Callers must:

- close the connection in a `finally` block;
- commit their own writes;
- handle any SQLite exceptions propagated by this helper.

The returned connection uses SQLite's default row factory, so rows are tuples unless the caller changes it.

### Algorithm

1. Open or create the SQLite database with `sqlite3.connect(db_path)`.
2. Set `PRAGMA journal_mode=WAL` so file-backed databases use write-ahead logging.
3. Set `PRAGMA foreign_keys=ON` for this connection.
4. Execute `PROPOSALS_SCHEMA` via `executescript`, creating proposal tables/indexes if needed.
5. Run `PRAGMA table_info(proposals)` and collect the second field of each row, which is the column name.
6. If `tags_json` is not in that column-name list, run:

   ```sql
   ALTER TABLE proposals ADD COLUMN tags_json TEXT DEFAULT '[]'
   ```

7. Commit the schema/migration work.
8. Return the open connection.

### Side Effects

- May create the SQLite database file.
- May create or update the proposal schema.
- May alter the existing `proposals` table by adding `tags_json`.
- May switch the database journal mode to WAL.
- Enables foreign-key checks on the returned connection.
- Writes schema changes to disk via commit.
- May create SQLite WAL-related files while the connection is open.
- Does not close the connection; caller is responsible for that.
- Has no network or subprocess side effects.

### Error Handling

There is no exception handling here. SQLite errors propagate to the caller, including:

- `sqlite3.OperationalError` for unable-to-open, read-only database, locked database, missing table, duplicate column, or invalid schema SQL;
- `sqlite3.DatabaseError` for corrupt or invalid database files;
- `TypeError` for unsupported `db_path` values.

If an error occurs after `sqlite3.connect`, this helper does not explicitly close the local connection. The connection may be cleaned up by Python garbage collection, but that is not deterministic. The helper also does not rollback explicitly; partial schema effects depend on SQLite and `executescript` behavior.

### Usage Patterns

Typical call shape:

```python
conn = _proposals_db(db_path)
try:
    conn.execute(...)
    conn.commit()
finally:
    conn.close()
```

For read-only proposal queries, callers still close the connection but do not need to commit.

This helper is called repeatedly during a proposal lifecycle. For example, `accept_proposal` opens a proposal connection to read and validate, later mutates the network through `_with_network`, and then opens another proposal connection to mark the proposal accepted. Therefore `_proposals_db` itself does not provide cross-connection atomicity between network changes and proposal status updates.

### Dependencies

- Python standard library `sqlite3`.
- `PROPOSALS_SCHEMA` from `reasons.storage`.
- SQLite's support for WAL mode, `PRAGMA table_info`, and `ALTER TABLE ... ADD COLUMN`.

## Topics to Explore

- [file] `reasons/storage.py` — Contains `PROPOSALS_SCHEMA`, which defines the actual tables and indexes `_proposals_db` ensures.
- [function] `reasons/api.py:accept_proposal` — Shows how proposal rows are revalidated, applied, and marked accepted across multiple connections.
- [function] `reasons/api.py:propose_retraction` — Demonstrates how proposal rows are created and how connection ownership is handled by callers.
- [general] `sqlite-wal-concurrency` — Explore writer locking, `database is locked` behavior, and why no `busy_timeout` is configured here.
- [general] `proposal-transaction-boundaries` — Investigate how proposal mutations and network truth-value changes are or are not atomic across separate SQLite connections.

## Beliefs

- `_proposals_db-executes-idempotent-schema-on-every-call` — It always runs `PROPOSALS_SCHEMA`, so that schema must be safe to execute when tables and indexes already exist.
- `_proposals_db-adds-tags-json-when-missing` — It inspects `PRAGMA table_info(proposals)` and adds a `tags_json` column only when the exact string `tags_json` is absent.
- `_proposals_db-returns-caller-owned-connection` — It returns an open `sqlite3.Connection`; callers close it in `finally` and commit their own proposal writes.
- `_proposals_db-sets-wal-and-foreign-keys` — It sets WAL journal mode for the database and enables foreign-key enforcement for the returned connection.
- `_proposals_db-does-not-close-on-error` — SQLite exceptions from schema setup or migration propagate before the caller receives a connection, and the helper has no `try/finally` close.
