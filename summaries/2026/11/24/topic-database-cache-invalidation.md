---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Whether destructive operations like `init_db(force=True)` should proactively call `clear_network_cache(db_path)`, and whether that gap has real consequences on coarse-mtime filesystems.

## Short answer

Yes. `init_db(force=True)` should explicitly invalidate the cache for its `db_path`, ideally by calling `clear_network_cache(db_path)`. The current usage scan shows `clear_network_cache` is defined at `reasons/api.py:172`, while `init_db` is defined at `reasons/api.py:191`, and there is no production call from `init_db` to `clear_network_cache`; the only usages are in `tests/test_network_cache.py` (for example, lines 20, 28, 114, 128, 158, 173).

## Why this matters

The cache appears to be keyed by database path and validated by file mtime:

- `test_read_populates_cache` asserts `_network_cache[db][1].nodes["a"]`.
- `test_cached_read_skips_storage` shows repeated reads reuse the same `Network`.
- `test_external_modification_invalidates_cache` captures `_network_cache[db][0]` as the cached mtime, calls `os.utime(db, None)`, and expects the cache entry’s mtime to change.
- `test_clear_cache_specific_path` at `tests/test_network_cache.py:114` confirms `clear_network_cache(db1)` clears only that path.

So if `init_db(force=True)` deletes/recreates the DB but the cache entry survives, `_with_network(db)` can return the pre-force in-memory `Network`.

## Real consequence on coarse-mtime filesystems

Yes, the gap is real on filesystems with coarse mtime granularity, such as some FAT/network filesystems with 1-second or 2-second resolution.

Example:

1. A cached read records `_network_cache[db][0] == 1700000000`.
2. `init_db(force=True)` deletes and recreates the DB at `1700000000.7`.
3. On a 1-second-resolution filesystem, the new file’s `st_mtime` is also `1700000000`.
4. `_with_network(db)` compares cached mtime with current mtime, sees them equal, and treats the cache as valid.
5. `search`, `show_node`, and `what_if_*` then read stale nodes from before the reset.

A later `_with_network(db, write=True)` appears to load fresh state, but reads before that can return data that should have been wiped. Also, `test_external_modification_invalidates_cache` sleeps only `0.05` seconds before `os.utime`, which is not enough to guarantee an mtime change on 1s/2s filesystems. That test can therefore be flaky and does not cover `init_db(force=True)`.

Cross-process invalidation is also a concern: clearing the cache inside the `init_db` process does not help other processes that already cached the old `Network`; they still rely on mtime and can be stale on coarse filesystems.

## Recommendation

On the `force=True` path in `init_db`, call `clear_network_cache(db_path)`. More generally, any operation that replaces, truncates, restores, or recreates the database at `db_path` should explicitly clear that path’s cache. At minimum, clear after successful reinitialization before returning. If concurrent access is possible, clear before the destructive work and again after success.

Caveat: the observations do not include the body of `init_db` or `_with_network`, so the exact invalidation check is inferred from tests. If `_with_network` also checks inode or file size, delete/recreate may mitigate some cases, but the tests only expose mtime behavior. Path normalization between `init_db` and the cache key is also not shown.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Confirm exactly how cache validity is checked and whether it uses only `st_mtime` or also inode/size.
- [function] `reasons/api.py:init_db` — Inspect the `force=True` branch and add `clear_network_cache(db_path)` there.
- [file] `tests/test_network_cache.py` — Add a regression test where `init_db(force=True)` runs within the same mtime bucket as the cached entry.
- [general] `cache-invalidation-token` — Evaluate replacing mtime-only invalidation with a stronger token such as inode, size, schema version, or an explicit generation counter.
- [file] `reasons/api.py` — Audit other destructive DB operations that may bypass `_with_network(..., write=True)` and leave stale caches.

## Beliefs

- `clear-network-cache-is-defined-in-reasons-api` — `clear_network_cache` is defined at `reasons/api.py:172` and supports clearing one database path or all cached networks.
- `init-db-does-not-call-clear-network-cache` — The symbol usage scan shows `clear_network_cache` is not called from `init_db`; only tests invoke it.
- `network-cache-invalidation-is-mtime-based` — `_network_cache[db][0]` holds a cached mtime, and `test_external_modification_invalidates_cache` expects `os.utime` to invalidate the cache.
- `write-context-loads-fresh-network` — `_with_network(db, write=True)` returns a different `Network` object from the cached read, so stale reads can persist until a write happens.
- `coarse-mtime-collision-can-preserve-stale-cache` — If a forced reinitialization produces the same coarse `st_mtime` as the cached value, `_with_network` can return the pre-reinitialization `Network`.
