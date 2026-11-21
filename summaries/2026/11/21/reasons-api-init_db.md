---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: init_db in reasons/api.py

# `init_db` — Explanation

## Purpose

`init_db` is the **entry point for creating a fresh Reason Maintenance System (RMS) database**. It exists so callers (CLI, LangGraph tools, scripts) can obtain a clean, schema-initialized database without touching `Storage` internals or argparse. It is the deliberate counterpart to `_with_network` (which **loads** an existing DB); `init_db` only creates.

It supports two backends through a single function signature: **SQLite** (the default, file-based) and **PostgreSQL** (via `pg_conninfo`, dispatched to `PgApi`).

## Contract

- **Precondition (SQLite path):** `db_path` points to a writable location and either (a) does not exist, or (b) exists with `force=True`. The parent directory is assumed to already exist.
- **Precondition (Postgres path):** `pg_conninfo` is a truthy, valid DSN understood by `PgApi`.
- **Postcondition (SQLite):** After return, `db_path` exists and contains a valid Storage schema; `Storage` was instantiated and closed cleanly. Return value is `{"db_path": str, "created": True}`.
- **Postcondition (Postgres):** Returned value is whatever `PgApi.init_db(project_name=...)` returns; the SQLite file is **not** touched.
- **Invariant:** If `db_path` exists and `force=False`, the function raises before any filesystem mutation — the caller's existing database is untouched.
- **Invariant:** When the SQLite path succeeds, `created` is **always** `True`; the function never reports `created: False`. Success == creation.

## Parameters

| Parameter | Type | Default | Meaning & edge cases |
|---|---|---|---|
| `db_path` | `str` | `DEFAULT_DB` (`"reasons.db"`) | Path to the SQLite file. Can be relative or absolute. **Ignored** when `pg_conninfo` is truthy. Passing a directory raises `IsADirectoryError` from `Path.unlink()` when `force=True`, or reaches `Storage` and fails there otherwise. |
| `force` | `bool` | `False` | If `True` and the file exists, the existing database (all beliefs) is **deleted** via `p.unlink()` before creation. If the file does not exist, `force=True` is a no-op. **Ignored** on the Postgres path. |
| `project_name` | `str` | `""` | Passed through to `Storage(...)`. The docstring says empty falls back to the DB filename stem. That fallback (if it happens) is implemented inside `Storage`, **not** here — `init_db` never inspects the filename for the project name. On the Postgres path it is forwarded to `PgApi.init_db`. |
| `pg_conninfo` | any (usually `str`) | `None` | Truthiness check only. Empty string falls through to SQLite — **not** a separate error. Any truthy value (even a non-string) will attempt Postgres dispatch. |
| `domain_id` | any | `None` | Only forwarded to `_pg_dispatch`; unused on the SQLite path. |

## Return Value

```python
{"db_path": str, "created": True}   # SQLite path only
```

- `db_path` is `str(Path(db_path))`, which normalizes the path representation but does **not** resolve to an absolute path (so `"./reasons.db"` → `"reasons.db"` via `Path.__str__`).
- On the Postgres path, the return value comes from `PgApi.init_db` and is **not guaranteed** to match the shape `{"db_path", "created"}` — callers integrating against both backends should handle whatever `PgApi` returns.
- There is no "already existed" success state; the caller either gets a fresh DB or an exception.

## Algorithm

1. **Postgres short-circuit.** If `pg_conninfo` is truthy:
   - Call `_pg_dispatch(pg_conninfo, domain_id, "init_db", project_name=project_name)`.
   - Note: `force` and `db_path` are **silently not forwarded**. Postgres has no file semantics and `force` is meaningless there.
   - Return whatever `PgApi.init_db` returns.
2. **Materialize** `p = Path(db_path)`.
3. **Existing file check (no force).** If `p.exists()` and `not force`: raise `FileExistsError`.
4. **Existing file check (force).** If `p.exists()` and `force`: `p.unlink()` — destroys the file and all data.
5. **Create.** `store = Storage(db_path, project_name=project_name)` — this is where the file and schema are actually created (constructor side effect).
6. **Close.** `store.close()` — flushes/commits and releases locks.
7. **Report.** Return `{"db_path": str(p), "created": True}`.

## Side Effects

- **Destructive:** With `force=True`, `p.unlink()` irreversibly deletes the entire SQLite database — no backup, no confirmation, no transactional rollback.
- **Filesystem write:** Creates a new SQLite database file at `db_path`, whose schema fixtures (nodes, justifications, node_tags, node_sources, FTS tables, etc.) are initialized by `Storage.__init__`.
- **No cache invalidation.** `_network_cache` is **not** cleared. Because `_with_network` compares `os.path.getmtime(db_path)` against the cached mtime, a force-recreated file normally gets an updated mtime and invalidates the cache **on the next load**. However, on filesystems with coarse (1-second) mtime resolution, a delete+recreate within the same tick could give the false impression of an unchanged file. This is a latent assumption, not enforced here.
- **No directory creation.** If the parent directory of `db_path` doesn't exist, either `Path.unlink()` or `Storage.db_path.open()` will raise. `init_db` does not `mkdir(parents=True)`.
- **No `force` on the Postgres side.** Callers cannot force-drop a Postgres domain through this entry point; Postgres dispatch simply omits `force`.
- **Logging:** None — the function relies entirely on exceptions.

## Error Handling

- **`FileExistsError`** — raised with the message `f"Database already exists: {db_path}"` when `p.exists()` and `force=False`. This is the primary "expected" error and callers usually convert it into a prompt/`--force` suggestion.
- **`OSError` / `IsADirectoryError`** — from `Path.unlink()` (permission denied, path is a directory, mount read-only).
- **`sqlite3.Error`** — propagated from `Storage(...)` if the file can't be created or the schema fails to apply.
- **Postgres errors** — whatever `PgApi.init_db` raises (e.g., connection failure, permission denied, domain already exists). `init_db` does not catch or translate these.
- **No swallowed errors.** The only branch that could pass silently is: the file is created but `Storage.__init__` doesn't actually create a schema — then `init_db` returns `created: True` on a hollow database. Nothing here asserts the resulting DB is non-empty.

## Usage Patterns

- **One-shot setup** — called at the top of a session before the first `add_node` / `with_network`-based call.
- **CLI `reasons init`** — with `--force` to reset a dev DB.
- **Implicit init from import flows** — `import_hf` and `import_api` invoke `init_db(db_path=..., force=False, project_name=..., pg_conninfo=pg_conninfo, domain_id=domain_id)` when `init=True` (or when using Postgres) and the file doesn't already exist.
- **Caller obligations:**
  - Catch `FileExistsError` if the caller wants to offer `--force` instead of crashing.
  - Verify `result["created"] is True` if the caller branches on freshness (always true on SQLite).
  - Do **not** assume `force=True` works against Postgres — it is silently ignored.
  - After `force=True`, callers should be aware the previously-loaded `Network` object in `_network_cache` (if any) is now stale but still holds memory; the next `_with_network` load will refresh it.

## Dependencies

- **`pathlib.Path`** — for existence check and unlink.
- **`Storage`** (`from .storage import Storage`) — actually creates the file and schema. `init_db` is essentially `Path`-guard + `Storage(...)` + `close`.
- **`_pg_dispatch`** (local helper) — delegates to `PgApi.init_db` for the Postgres path.
- **`DEFAULT_DB`** (`"reasons.db"`) — the fallback default path.

## Assumptions Not Enforced by Types

1. **`Storage.__init__` creates the DB and schema as a side effect.** Nothing else in `init_db` writes files, so this is load-bearing. If `Storage` merely opened a handle without creating the schema, `init_db` would return success on an unusable DB.
2. **`pg_conninfo` truthiness implies validity.** An empty string, `0`, or an `int` are treated surprisingly — empty string goes to SQLite; a non-empty non-DSN will fail deep inside `PgApi`.
3. **`force` is silently ignored on the Postgres path.** A caller passing `force=True, pg_conninfo="..."` will **not** get drop-and-recreate semantics; if the domain exists, `PgApi.init_db` may fail or be idempotent — caller has no way to know from this signature.
4. **Parent directory exists.** No `mkdir(parents=True)`.
5. **`db_path` points to a file, not a directory.** `p.exists()` is true for directories too; `force=True` on a directory raises `IsADirectoryError` rather than a friendlier error.
6. **`project_name=""` means "derive from filename"** — this behavior lives in `Storage` (or is at least claimed by the docstring); `init_db` never applies that fallback itself.
7. **Cache safety relies on mtime granularity.** `_network_cache` is not cleared; correctness depends on `getmtime` changing after `unlink()+create`, which is true on most filesystems but not universally guaranteed.

## Topics to Explore

- [file] `reasons/storage.py` — See what `Storage.__init__` actually creates (tables, FTS index, `project_name` fallback), and confirm `init_db` relies on it for schema creation.
- [function] `reasons/api.py:_pg_dispatch` — Understand how `init_db` is re-dispatched to `PgApi.init_db` and what each backend's `init_db` returns.
- [file] `reasons/pg.py` — Inspect `PgApi.init_db` to learn whether `force`, idempotency, or domain collisions are handled on the Postgres side.
- [function] `reasons/api.py:_with_network` — See how mtime-based cache lookup interacts with a freshly recreated database, and where `_network_cache` is (or isn't) cleared.
- [general] `database-cache-invalidation` — Whether destructive operations like `init_db(force=True)` should proactively call `clear_network_cache(db_path)`, and whether that gap has real consequences on coarse-mtime filesystems.

## Beliefs

- `init-db-raises-on-existing-file-unless-forced` — When `db_path` exists and `force=False`, `init_db` raises `FileExistsError` without mutating the existing database.
- `init-db-force-deletes-existing-database` — With `force=True`, `init_db` unconditionally calls `Path.unlink()` on the existing file, discarding all prior beliefs before recreating it.
- `init-db-ignores-db-path-and-force-when-pg-conninfo-set` — When `pg_conninfo` is truthy, `init_db` delegates to `_pg_dispatch` and does not forward `db_path` or `force` to the Postgres backend.
- `init-db-always-reports-created-true` — On the SQLite path, `init_db` always returns `{"created": True}` on success; it never returns `created: False`.
- `init-db-depends-on-storage-constructor-for-schema-creation` — `init_db` performs no direct schema writes; the file and schema are created as a side effect of instantiating `Storage`.
- `init-db-does-not-clear-network-cache` — `init_db` never calls `clear_network_cache`, so any cached `Network` for `db_path` is only invalidated on the next `_with_network` load via mtime comparison.
