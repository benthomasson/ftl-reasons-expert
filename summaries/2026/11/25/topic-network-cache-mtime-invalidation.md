---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How `_network_cache` interacts with concurrent writers and why read-only report APIs are effectively a single load per call.

## `_network_cache` and `_with_network`

`_network_cache` is a process-local dictionary declared at `reasons/api.py:33`:

```python
_network_cache: dict[str, tuple[float, Network]] = {}
```

It is keyed by database path. Each value is `(mtime, Network)`: the database file’s modification time plus the fully loaded in-memory `Network`.

The cache is used by `_with_network(db_path, write=False)`:

- On entry, `_with_network` reads the current DB mtime and checks `_network_cache.get(db_path)` at `reasons/api.py:130`.
- If there is a cache entry, the mtime matches, and `write` is `False`, it reuses the cached `Network`.
- Otherwise it opens `Storage(db_path)` and calls `self.store.load()`.
- On exit, if the operation is not a failed write, it updates `_network_cache[db_path] = (mtime, self.network)` at `reasons/api.py:164`.
- `clear_network_cache()` removes one path or clears the entire dictionary at `reasons/api.py:172-181`.

This means read-only calls see an mtime-validated snapshot. If the database file has not changed, they reuse the same `Network` object. If the file mtime changed, the next read-only call performs one fresh `Storage.load()` and then updates the cache.

## Concurrent writers

The important property is that `write=True` bypasses the cache:

```python
if cached and mtime is not None and cached[0] == mtime and not write:
    self.network = cached[1]
else:
    self.store = Storage(db_path)
    self.network = self.store.load()
```

So a writer always loads a fresh `Network` instead of mutating the cached object directly. This is verified by `tests/test_network_cache.py::test_write_loads_fresh`, which asserts that a read context and a write context return different `Network` objects.

On a successful write, `_with_network.__exit__` saves the network and then refreshes the cache. If the write raises, the save is skipped and the cache update is also skipped:

```python
if not (write and exc_type is not None):
    ...
    _network_cache[db_path] = (mtime, self.network)
```

That behavior is covered by `tests/test_network_cache.py::test_write_exception_does_not_poison_cache`, which verifies that a failed write does not replace the cached original with a corrupted network.

However, there is no visible locking around `_network_cache` or `_with_network`. The cache is a plain `dict`, and the check-then-load/save-then-cache sequence is not atomic. Consequences:

- Two same-process writers can both load fresh copies from the same initial DB state, mutate independently, and save whole-network snapshots. The later save wins, potentially discarding the other writer’s changes.
- Cache updates can race. For example, writer A can save, writer B can save, then writer A can update the cache with its own older `Network` object while reading the newer file mtime. A subsequent read-only call would see an mtime match but reuse a stale network.
- Across processes, each process has its own `_network_cache`, so the mtime check only detects external changes at the start of a `_with_network` context. There is no cross-process cache invalidation or visible file-lock coordination in the observed code.

So the cache improves read reuse but does not make concurrent writers safe by itself. It is an mtime-based, process-local cache, not a concurrency-control mechanism.

## Why read-only report APIs are effectively a single load per call

The report formatter modules `reasons/report_belief.py` and `reasons/report_gated.py` operate on data passed into them. For example, `report_belief(...)` takes `node_detail`, `explain_steps`, `premises_data`, and `source_chunks`, then either builds structured markdown in `_build_structured_report` or formats prompt text in `_format_data_for_prompt`. Those functions do not open `Storage` or read the database themselves.

Therefore, if a read-only report API opens one `_with_network(db_path)` context, the network is resolved once for that API call:

- cache hit: no `Storage.load()` is needed;
- cache miss or mtime change: exactly one `Storage.load()` happens at context creation.

All report generation then works against the already-loaded in-memory `Network` and passed data. Repeated read-only report calls reuse the cached `Network` until the DB mtime changes. Tests such as `tests/test_network_cache.py::test_cached_read_skips_storage` assert that two read contexts return the same `Network` object, and `test_external_modification_invalidates_cache` confirms that an mtime change causes the cache to be refreshed.

One caveat: the actual bodies of the `report_gated` and `report_belief` API functions at `reasons/api.py:3671` and `reasons/api.py:3732` are not included in the observations. I can confirm the cache behavior of `_with_network` and that the report modules are pure formatters, but I cannot verify from the provided snippets whether each report API opens exactly one `_with_network` context. That is the main missing piece if you need to prove the “single load per call” property end-to-end.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Verify the mtime check, cache update, write bypass, and failed-write behavior in the full source.
- [function] `reasons/api.py:report_belief` — Confirm whether the report API opens one `_with_network` context and passes loaded data to `reasons/report_belief.py`.
- [function] `reasons/api.py:report_gated` — Confirm the same call pattern for gated reports.
- [general] `storage-save-concurrency` — Explore `Storage.save`/`Storage.load` transaction and locking behavior to understand cross-process writer safety.
- [file] `tests/test_network_cache.py` — Tests document expected cache invalidation, write freshness, failure poisoning, and what-if copy isolation.

## Beliefs

- `network-cache-is-mtime-keyed` — `_network_cache` stores `(mtime, Network)` values keyed by `db_path` at `reasons/api.py:33`.
- `read-only-with-network-requires-mtime-match` — `_with_network(..., write=False)` reuses a cached `Network` only when the cached mtime equals the current DB file mtime.
- `writes-bypass-cache-and-load-fresh` — `_with_network(..., write=True)` always loads a fresh `Network` instead of reusing the cached object.
- `failed-write-prevents-cache-update` — A write context that raises skips both `Storage.save` and the `_network_cache` update.
- `network-cache-has-no-explicit-locking` — The observed `_network_cache` and `_with_network` code contain no synchronization primitive around cache reads/writes.
