---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: mtime-based caching implications when clustering after writes

## mtime-based caching implications when clustering after writes

### The two caches at play

There are two independent caches involved in the "cluster after a write" flow:

1. **Network cache (`reasons/api.py`)** — `_network_cache: dict[str, tuple[float, Network]]` is a module-level dict keyed by `db_path`. The docstring at line 119 and the logic in `_with_network` (roughly lines 113–175) show:
   - Line 127: `mtime = os.path.getmtime(db_path)` is captured on **entry**.
   - Line 131: the cached `Network` is reused only when `cached[0] == mtime` **and** `not write`.
   - Lines 163–164: on **exit**, `_network_cache[db_path] = (os.path.getmtime(db_path), self.network)` — the writer re-caches its own in-memory `Network` object under the post-save mtime.
   - `clear_network_cache` (just below `_with_network`) is the manual invalidation escape hatch.

2. **Embedding cache (`reasons/cluster.py`)** — `ClusterCache.embed` keys embeddings by `(node_id, sha256(text.encode()).hexdigest()[:16])`. This cache is **text-hash based, not mtime based**. If the text for a node changes, the key changes and the embedding is recomputed. This makes `ClusterCache` itself robust to writes — *provided* it sees the current text.

### How clustering consumes beliefs

`cluster_beliefs`, `cluster_beliefs_intra`, and `list_clusters` in `reasons/cluster.py` **do not open the database themselves**. They take a `beliefs: {node_id: text}` dict and rely entirely on the caller to supply fresh data. So the freshness question for clustering is really: **what `beliefs` dict did the caller build, and did it come from a fresh `_with_network` view?**

### Implications when clustering immediately after a write

**1. Same-process: writer → cluster is consistent, by design.**
After a `_with_network(write=True)` call finishes, `__exit__` (lines 163–164) stores the just-saved `Network` under the newly-read mtime. A subsequent read-only `_with_network` (the one a cluster caller would use) sees `cached[0] == mtime` and reuses that exact object — so it reflects the saved state without a round-trip to SQLite. This is the intended fast path.

**2. Cross-process / external writes: WAL can mask mtime changes.**
`Storage.__init__` sets `PRAGMA journal_mode=WAL` (`reasons/storage.py`). In WAL mode, writes land in the `-wal` sidecar and the *main* DB file's mtime often does **not** update until a checkpoint fires (on close of the last connection, or when the autocheckpoint threshold is hit). Consequences:
   - A reader in process B may see the same `os.path.getmtime(db_path)` as before a write in process A, and re-use its stale `Network`.
   - That stale `Network` feeds a stale `beliefs` dict into `cluster_beliefs` / `list_clusters`, and the `ClusterCache` will happily reuse the old embeddings for the old text (since the text-hash key matches the stale text). **There is no error signal** — clustering silently runs on pre-write state.
   - `clear_network_cache(db_path)` is the only explicit escape hatch; it is not called automatically after a foreign write.

**3. mtime granularity / same-tick collisions.**
`os.path.getmtime` returns a float but the underlying resolution depends on the filesystem (1s or 2s on some filesystems, coarse on many network mounts). Two writes within the same tick can produce identical mtimes, causing a false cache hit. The test at `tests/test_network_cache.py:83-99` exercises external modification invalidating the cache (`cached_mtime = _network_cache[db][0]` … `assert _network_cache[db][0] != cached_mtime`), which is the intended behavior — but the test must force an actual mtime change to be meaningful, and in production a fast writer→cluster→writer sequence can defeat this.

**4. The cluster embedding cache does not repair staleness.**
`ClusterCache.embed` is a pure function of the `(node_id, sha256(text)[:16])` key. If the `Network` was stale, the "text" it hashes is the old text, so the cached embedding is returned and the stale text is clustered without any indication that the DB has moved on. Conversely, after a *real* text change, the new hash forces a re-embed — so `ClusterCache` is only as fresh as the dict handed to it.

**5. Read-path mutation hazard.**
In `_with_network.__exit__`, the cache write happens for read-only calls too (the guard is `not (write and exc_type is not None)`). If a read-only caller mutates the `Network` object it received (nothing prevents this — it's the same object held in `_network_cache`), the cached entry can diverge from the on-disk DB. A later cluster call that reuses the cached object would cluster on mutated in-memory state that was never persisted.

### What's missing from the observations

- **The API-level entrypoint for clustering is not shown.** `find_cluster_entrypoints` returned zero matches, and `reasons/api.py` is truncated at line 400; the file is 5886 lines. We cannot verify whether the cluster caller uses `_with_network(db_path)` (read path), a `write=True` call, or an explicit `clear_network_cache`. That determines whether the same-process fast path applies or whether re-reads are forced.
- **The full `tests/test_network_cache.py`.** We only have line snippets (83–99). We don't see *how* the test forces the mtime change (e.g., `os.utime`, sleep, or a real write), which matters for assessing the granularity risk.
- **How `beliefs` are filtered** (IN-only? education on derived nodes?) upstream of clustering — this is what makes stale reads *matter* or not.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — the sole cache-invalidation point; confirm exactly when `mtime` is captured vs. when the cache is refreshed.
- [file] `reasons/api.py` — find the API that builds the `beliefs` dict and hands it to `cluster_beliefs`; verify read vs. write path and whether `clear_network_cache` is called.
- [file] `reasons/storage.py` — WAL pragma in `Storage.__init__`; check whether `save()` / `close()` triggers a checkpoint that updates the main file mtime.
- [general] `mtime-granularity` — understand OS/filesystem mtime resolution and how same-tick writes can defeat mtime caching.
- [file] `tests/test_network_cache.py` — see how the test forces an external mtime change and whether it covers the WAL scenario.

## Beliefs

- `network-cache-keyed-by-db-path-and-mtime` — `_network_cache` maps `db_path` to `(mtime, Network)`; the read path reuses a cached `Network` only when the file's `os.path.getmtime` is unchanged.
- `write-path-refreshes-network-cache` — After a successful `_with_network(write=True)`, `__exit__` stores the post-save mtime and the in-memory `Network` in `_network_cache`.
- `cluster-cache-uses-text-hash-not-mtime` — `ClusterCache.embed` keys embeddings on `(node_id, sha256(text)[:16])`, so text changes force re-embedding but stale text is reused silently.
- `cluster-functions-trust-input-dict` — `cluster_beliefs`, `cluster_beliefs_intra`, and `list_clusters` never open the DB; freshness depends entirely on the caller-supplied `beliefs` dict.
- `wal-mode-delays-main-file-mtime-update` — `Storage.__init__` sets `journal_mode=WAL`, so the main DB file's mtime can lag behind actual writes until a checkpoint, affecting cross-process cache invalidation.
