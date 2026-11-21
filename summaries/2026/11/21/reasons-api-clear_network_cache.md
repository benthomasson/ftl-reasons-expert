---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: clear_network_cache in reasons/api.py

## Purpose

`clear_network_cache` is the manual invalidation hook for the module-global `_network_cache`, which is defined as:

```python
_network_cache: dict[str, tuple[float, Network]] = {}
```

The cache is populated by `_with_network`, which reuses a cached `Network` for read-only operations when the cached file mtime matches the current DB file mtime. Writes always load fresh, save, then update the cache.

`clear_network_cache` exists so callers can explicitly discard cached `Network` objects when the cache may be stale, when the DB file was changed outside this process, or between tests. It does not interact with the database itself; it only mutates the in-memory dictionary.

## Contract

Preconditions:

- `db_path` is either `None` or intended to be the exact string key used in `_network_cache`.
- The DB file does not need to exist.
- No lock is held or acquired.

Postconditions:

- If `db_path is None`, `_network_cache` is empty afterward.
- If `db_path` is a string, the entry with that exact key is removed if present.
- If the key is absent, nothing happens.
- Other cache entries are untouched when a specific `db_path` is passed.
- The function always returns `None`.

Important invariants / assumptions:

- Cache keys are **exact strings**. There is no path normalization and no equivalence between `"reasons.db"`, `"./reasons.db"`, and an absolute path.
- Passing a `pathlib.Path` will not match a cache entry keyed by its string form unless the cache was also keyed by a `Path` object.
- Clearing the cache does not affect any `Network` object already in use by an active `_with_network` context. On context exit, `_with_network` may repopulate the cache.
- There is no synchronization. Concurrent calls to `clear_network_cache` and `_with_network` can race.

## Parameters

### `db_path: str | None = None`

- `None`: clear the entire `_network_cache`.
- `str`: remove only the entry whose key is exactly equal to `db_path`.
- Missing key: silently ignored via `dict.pop(db_path, None)`.
- Empty string: treated as a valid key; it will only remove the `""` entry.
- Non-string hashable object: technically works if it matches an existing key, but the type contract expects `str`.
- Unhashable object such as a `list`: `dict.pop` raises `TypeError`.
- `Path` object: hashable, but it generally will not match a string key.

## Return Value

Always returns `None`.

The caller cannot tell whether a specific entry existed or whether anything was actually removed. There is no count returned and no exception for missing entries.

## Algorithm

1. If `db_path is None`:
   - Call `_network_cache.clear()`.
   - This removes every cached `(mtime, Network)` pair.
2. Otherwise:
   - Call `_network_cache.pop(db_path, None)`.
   - This removes the matching key if present.
   - The `None` default means a missing key is ignored.

## Side Effects

- Mutates the module-global `_network_cache` dictionary.
- Drops references to cached `Network` objects.
- May allow cached objects to be garbage-collected if no active context still references them.
- Does **not** close `Storage`, save data, delete files, or touch SQLite/PostgreSQL.
- Does **not** invalidate any other cache, such as proposal caches or external process state.
- Does **not** prevent an in-flight `_with_network` context from reinserting its `Network` into `_network_cache` on exit.

## Error Handling

The function has no explicit `try`/`except`.

Possible errors:

- Unhashable `db_path`: `TypeError` from `dict.pop`.
- Extremely low-level failures such as `MemoryError`: propagate as usual.

Missing keys are deliberately swallowed by `pop(..., None)`. There is no validation that `db_path` points to a real database or that it is a `str`.

## Usage Patterns

Typical uses:

- Clear the entire cache after out-of-band database modifications:
  ```python
  clear_network_cache()
  ```
- Invalidate one database entry:
  ```python
  clear_network_cache("reasons.db")
  ```
- Reset cache state between tests.
- Force a reload when the mtime-based check in `_with_network` is insufficient, such as coarse filesystem timestamps, external writers, WAL behavior, or rapid replacement of the DB file.

Caller obligations:

- Pass the same string form used as the cache key.
- Avoid relying on this function to close or persist anything.
- Avoid calling it concurrently with `_with_network` if deterministic cache state matters.
- If a `with _with_network(...)` block is active, expect the cache to be repopulated when that block exits.

## Dependencies

- Depends only on the module-level `_network_cache` dictionary.
- Indirectly relates to `_with_network`, `Storage.load`, `Storage.save`, and `os.path.getmtime`.
- No external module calls, no filesystem I/O, no database access.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The cache producer/consumer that determines when mtime-based reuse occurs and when cached networks are replaced.
- [file] `tests/test_network_cache.py` — Shows the intended invalidation and cache-reuse behavior under test.
- [file] `reasons/storage.py` — Explains how `Storage.load` and `Storage.save` create and persist the `Network` objects stored in the cache.
- [general] `mtime-based-cache-invalidation` — Explore edge cases where file mtime is insufficient to detect external database changes.
- [general] `cache-concurrency` — Understand why unsynchronized cache clearing can race with `_with_network` context cleanup.

## Beliefs

- `clear-network-cache-clears-global-cache` — Calling `clear_network_cache(None)` removes every entry from `_network_cache`.
- `clear-network-cache-specific-path-is-exact-key` — Calling `clear_network_cache(path)` removes only the entry whose key exactly equals that string.
- `clear-network-cache-does-not-touch-storage` — The function performs no database or filesystem I/O and does not save or close any `Network`.
- `network-cache-reuse-requires-mtime-match` — `_with_network` reuses a cached read-only `Network` only when the cached mtime equals the current file mtime.
- `clear-network-cache-has-no-synchronization` — The function uses no lock, so an active `_with_network` context can repopulate the cache after it is cleared.
