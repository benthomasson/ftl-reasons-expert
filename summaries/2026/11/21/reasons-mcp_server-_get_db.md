---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _get_db in reasons/mcp_server.py

## `_get_db` in `reasons/mcp_server.py`

`_get_db` is a small lazy-initialization helper that resolves and caches the path to the `reasons.db` database file. It is not an MCP tool itself; it is used by the MCP tool functions to supply `db_path=` to the `reasons.api` layer.

### Purpose

- Avoid re-running database discovery on every MCP tool call.
- Defer discovery until it is actually needed when the module is imported and tools are invoked directly.
- Provide a single process-wide resolved database path for all MCP tools.
- Normal server startup pre-seeds this cache in `main()` via `_db = _find_db()`.

### Contract

- Precondition: module global `_db` exists and is initialized to `None` at import time.
- Postcondition on success: returns the resolved database path as a string.
- Cache invariant: if `_db is None`, the path has not yet been resolved. After a successful resolution, `_db` is non-`None` and later calls return the cached value without calling `_find_db()` again.
- The function does not guarantee that the returned path exists, is readable, or points to a valid SQLite database.
- The type annotation says it returns `str`, but runtime behavior depends on `_find_db()` returning a string. If `_find_db()` returned `None`, `_get_db()` would return `None` and would retry on the next call because `_db` would still be `None`.
- If `_find_db()` returns an empty string, that empty string is cached because the check is `is None`, not truthiness.

### Parameters

None.

### Return Value

Returns the module-global cached database path, resolved by `_find_db()` on first use.

Callers should treat it as a path string to pass into `reasons.api` functions. They must not assume the path exists or is valid; the API/database layer is responsible for handling filesystem or SQLite errors.

### Algorithm

1. Read the module-global `_db`.
2. If `_db is None`, call `_find_db()` and assign its return value to `_db`.
3. Return `_db`.

This is a classic lazy memoization pattern.

### Side Effects

- Mutates the module-global `_db` on first successful resolution.
- Indirectly reads environment and filesystem state through `_find_db()`:
  - Checks `REASONS_DB`.
  - Walks upward from `os.getcwd()` looking for `reasons.db`.
  - Falls back to `api.DEFAULT_DB`.
- Does not create, open, or modify the database.
- The cached path persists for the lifetime of the process. If the environment variable or current working directory changes after the first call, `_get_db()` will not notice.

### Error Handling

`_get_db()` has no `try`/`except`. It propagates exceptions from `_find_db()`.

Examples of possible propagated errors include:

- `OSError` from `os.getcwd()`.
- `AttributeError` if `api.DEFAULT_DB` is missing.
- Any other exception raised by `_find_db()`.

If lazy initialization fails, `_db` remains `None`, so a later call will retry discovery.

### Usage Patterns

All MCP tool wrappers call `_get_db()` when invoking `reasons.api` functions, for example:

```python
api.search(query, db_path=_get_db(), ...)
```

Normal server startup runs:

```python
def main():
    global _db
    _db = _find_db()
    mcp.run()
```

So in the normal `reasons-mcp` entry point, the cache is pre-seeded and `_get_db()` is effectively just a cached global read. Lazy discovery matters when the module is imported and tool functions are called directly, such as in tests.

Tests that change `REASONS_DB`, change the working directory, or monkeypatch `_find_db()` must reset `mcp_server._db = None` between cases to avoid stale cache leakage.

### Dependencies

- `_find_db` in the same module.
- Module-global `_db`.
- Indirectly `reasons.api.DEFAULT_DB` through `_find_db`.
- Indirectly `os` through `_find_db`.

### Assumptions Not Enforced by Types

- `_find_db()` is assumed to return a valid `str`; the annotation is not runtime-enforced.
- `api.DEFAULT_DB` is assumed to exist and be a string.
- The discovered path is assumed to remain valid for the process lifetime.
- The function is assumed to be called from a single-threaded or otherwise benign concurrency context. There is no lock, so two threads can both observe `_db is None` and call `_find_db()` redundantly.

## Topics to Explore

- [function] `reasons/mcp_server.py:_find_db` — Implements the actual `REASONS_DB`, walk-up, and default fallback discovery logic.
- [function] `reasons/mcp_server.py:main` — Shows how the server pre-seeds the `_db` cache before `mcp.run()`.
- [file] `tests/test_mcp_server.py` — Demonstrates how database discovery and the module-global cache are tested or reset.
- [general] `module-global-caching-in-tests` — Useful for understanding stale-state risks when testing `_db`.
- [file] `reasons/api.py` — The API layer that consumes the `db_path` returned by `_get_db`.

## Beliefs

- `get-db-caches-resolution` — `_get_db` calls `_find_db` only when the module global `_db` is `None`, then returns the cached value on later calls.
- `get-db-delegates-discovery` — `_get_db` contains no discovery logic itself; all environment, working-directory, and default-path resolution is delegated to `_find_db`.
- `main-prewarms-db-cache` — `main` assigns `_db = _find_db()` before starting the MCP server, bypassing lazy initialization during normal server startup.
- `get-db-does-not-validate-path` — `_get_db` returns the discovered path without checking that the database file exists or is readable.
- `get-db-propagates-find-db-errors` — `_get_db` has no exception handling, so errors from `_find_db` propagate and leave `_db` unset when lazy initialization fails.
