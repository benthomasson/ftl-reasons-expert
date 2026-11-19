---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _with_network in reasons/api.py

# `_with_network` — Load/Operate/Save context manager with mtime-keyed caching

## 1. Purpose

`_with_network` is the shared transactional boundary for every SQLite-backed API function in `reasons/api.py`. Its job is to be the *single* place that knows how to do the three-phase pattern:

1. **Open** the `Storage` and `load()` a `Network` (or reuse a cached one).
2. **Yield** the `Network` for the caller to read or mutate.
3. **Save** (write mode) / **close** / **update cache**.

It exists for three reasons:

- **Caching**: read-only calls across the codebase are extremely common (`get_status`, `list_nodes`, `show_node`, `search`, `lookup`, …). Re-reading and re-parsing the full SQLite DB on every call would be wasteful when the file hasn't changed. The cache keyed by `(db_path, mtime)` makes read-only access effectively free after the first load.
- **Correctness discipline**: writes must never operate on the cached `Network` (they could diverge from disk and skip save), so `write=True` bypasses the cache entirely.
- **Pub/sub integration**: on successful writes, the manager emits truth-value deltas to any subscribers — computed *before* and *after* the body, not inside each API function.

## 2. Contract

**Preconditions**
- `db_path` refers to an existing SQLite database created by `Storage`. If it does not, `Storage(db_path)`/`store.load()` will raise inside `__init__`.
- In `write=True` mode, the caller must treat the returned `Network` as exclusively owned by the context for its duration.
- In `write=False` mode, the caller must **not mutate** the returned `Network`, because it may be the cached instance shared with other read-only callers. Nothing enforces this by type — the `what_if_*` functions work around it with `copy.deepcopy`.

**Postconditions**
- **Read-only, cache hit**: no `Storage` is opened; the cached `Network` is returned; on exit the cache entry is refreshed with the current `mtime`.
- **Read-only, cache miss**: `Storage.load()` is used; on exit the cache is populated.
- **Write, no exception in body**: `store.save(network)` is called; if there were subscribers at `__enter__` time, a delta event is published; `store.close()` is called; cache is updated.
- **Write, exception in body**: no `save`, no cache update, `store.close()` runs, exception propagates (`return False`).

**Invariants**
- `_network_cache[db_path] == (mtime, network)` is only trusted for read-only paths.
- `self.store` is always non-`None` when `write=True`; it may be `None` on a cache-hit read-only path.
- `self._before` is non-`None` only when `write=True` **and** `pubsub.has_subscribers(db_path)` was true at `__enter__`.

## 3. Parameters

- **`db_path: str`** — filesystem path to the SQLite database. Used as both the `Storage` constructor arg and the cache dict key. An `OSError` from `os.path.getmtime` is swallowed (treated as "no mtime → cache miss") but the same missing path will fail inside `Storage(db_path)`, so this param is effectively required to point at a real file.
- **`write: bool = False`** — selects one of two entirely different code paths:
  - `False` ⇒ try cache first; only touch disk if `mtime` differs or cache is empty.
  - `True` ⇒ *always* open `Storage` and `load()` fresh (cache is ignored even on a hit); save on successful exit.

## 4. Return Value

Returns an instance of the inner `_Ctx` class (not a generator, not a `contextlib.contextmanager`). `_Ctx` implements `__enter__` (returns the `Network`) and `__exit__`.

Because `_Ctx()` is *called* in the `return` statement, `__init__` runs **eagerly** at the moment `_with_network(...)` is invoked, before the `with` block starts. Consequences:

- DB open and `load()` happen at factory-call time, not `__enter__` time. For the standard `with _with_network(...) as net:` pattern this is indistinguishable. For split use (`ctx = _with_network(p); ...; with ctx:`) the storage connection is held open across the gap.
- Any exception from `Storage`/`load()` propagates from the factory call, not from `with`.
- If `load()` raises after `Storage.__init__` assigns `self.store`, the connection is **leaked** — `__exit__` never runs because the `_Ctx` object is discarded.

The returned `Network` remains valid after the `with` block; nothing invalidates it. Callers that keep references past the `with` are implicitly relying on later writes being able to see the same instance — which is *only* true if the cache entry still points at it.

## 5. Algorithm

**`__init__`**
1. `self.store = None`.
2. Fetch `mtime = os.path.getmtime(db_path)`; on `OSError` set `mtime = None`.
3. Look up `cached = _network_cache.get(db_path)`.
4. If `cached` exists, `mtime` is not `None`, `cached[0] == mtime`, **and** `not write` → `self.network = cached[1]` (do *not* open `Storage`).
5. Otherwise → `self.store = Storage(db_path); self.network = self.store.load()`.
6. `self._before = None` (snapshot populated later).

**`__enter__`**
1. If `write` **and** `pubsub.has_subscribers(db_path)`, snapshot `{nid: n.truth_value}`.
   - `has_subscribers` runs here, not in `__init__`. Subscribers appearing between factory call and `with` *will* be captured; those appearing after `__enter__` will *not* trigger a delta event.
2. Return `self.network`.

**`__exit__(exc_type, exc_val, exc_tb)`**
1. If `write` **and** `exc_type is None`:
   - If `self.store` truthy (always true for write): `self.store.save(self.network)`.
   - If `_before` was captured: build `after` snapshot, call `pubsub.compute_changes(self._before, after, db_path)`, then `pubsub.publish(events, db_path)`.
2. If `self.store`: `self.store.close()`.
3. If **not** (`write and exc_type is not None`):
   - Re-read `mtime`; on success store `(mtime, self.network)` in `_network_cache`.
   - This runs for read-only exits *and* successful write exits.
4. `return False` — do not suppress body exceptions.

The last cache update step means: it is skipped *only* when a write body raised. That's deliberate — you don't want to cache mutations that weren't saved to disk.

## 6. Side Effects

- **Disk I/O**: `os.path.getmtime` (twice per call — `__init__` and `__exit__`), `Storage.load` (reads DB), `Storage.save` (writes DB, only on successful write).
- **Connection lifecycle**: `Storage` opened in `__init__`, closed in `__exit__`. Skipped entirely on cache-hit read-only paths.
- **Global state**: `_network_cache[db_path]` is written on every exit except failed writes.
- **Pub/sub**: `compute_changes` + `publish` on successful writes *only when subscribers existed at `__enter__`*.
- **Object aliasing**: the caller of a read-only context receives a reference to the same `Network` other read-only callers may also hold. This is intentionally shared memory.

## 7. Error Handling

- `os.path.getmtime` failure at either call site is swallowed (`mtime=None` or `pass`). This is a *development* convenience, not a correctness guarantee — the subsequent `Storage(db_path)` will fail if the file is truly missing.
- `Storage(db_path)` and `store.load()` exceptions propagate from the factory call. Note the leak hazard: if `load()` raises after `Storage()` succeeded, `self.store` has an open SQLite connection that `__exit__` never closes.
- `store.save()` exceptions in `__exit__` propagate, but they also leave `self.store.close()` at risk — the close call is *not* in a `try/finally`, so a save failure leaks the connection. This is a latent resource-leak path.
- `pubsub.compute_changes`/`publish` exceptions propagate similarly, after `save` succeeded but before `close`.
- Body exceptions propagate (`return False`). Cache is not updated, save is skipped — so a partially-mutated in-memory `Network` is discarded, which is correct.
- **Concurrent writer race (read-only path)**: if another process writes to the DB between `__init__` (which stamped `mtime`) and `__exit__` (which re-reads `mtime`), the exit hook stores the *stale in-memory* `Network` under the *new* `mtime`. The next reader sees a matching mtime and gets a stale cache hit. `mtime` is coarse (often 1s resolution) so intra-second writes may not even bump it one way or another.

## 8. Usage Patterns

Standard read:
```python
with _with_network(db_path) as net:
    for nid, node in net.nodes.items():
        ...
```

Standard write:
```python
with _with_network(db_path, write=True) as net:
    net.add_node(...)
    net.recompute_all()
# on normal exit: saved + cache refreshed + pub/sub fired if any subscribers
```

Mutate-but-don't-save (simulation):
```python
with _with_network(db_path) as net:
    net = copy.deepcopy(net)      # <-- required; do NOT mutate cache
    net.retract(node_id)
```
The write functions in the `what_if_*` family follow this pattern exactly, precisely because the read-only context may hand back the shared cached object.

Caller obligations not enforced by types:
- Do not mutate the `Network` in read-only mode without deep-copying.
- Do not rely on the returned `Network` after `with` for consistency with later external writes — the cache may be replaced.
- If you need to invalidate the cache — e.g. after mutating the DB out-of-band (like `add_tags`, `add_source`, `import_json` which write via raw `sqlite3` and then close) — call `clear_network_cache(db_path)` or ensure your edit bumps `mtime`.

## 9. Dependencies

- `os.path.getmtime` (stdlib) for freshness check.
- `Storage` from `.storage` — the SQLite persistence layer (`__init__` opens, `.load()` returns `Network`, `.save(net)` writes, `.close()` releases).
- `Network` from `.network` — the object yielded to the caller; must expose `.nodes: dict[str, Node]` where each `Node` has `.truth_value: str`.
- `pubsub` from `.` (module) — `has_subscribers`, `compute_changes`, `publish`.
- Module-level `_network_cache: dict[str, tuple[float, Network]]` — the cache itself.

---

## Topics to Explore

- [file] `reasons/storage.py` — Defines `Storage.load()`/`save()` and what invariants the loaded `Network` carries; the manager relies on `save` being able to persist every mutation the caller makes.
- [file] `reasons/pubsub.py` — Semantics of `has_subscribers`, `compute_changes`, and `publish`; especially whether publish is synchronous and whether subscriber additions mid-write are visible.
- [file] `tests/test_network_cache.py` — The intended contract for mtime hits/misses and the write bypass; reveals what the maintainers consider out-of-scope (e.g. concurrent writers).
- [function] `reasons/api.py:clear_network_cache` — The escape hatch for external writers that don't go through `_with_network`; explore which API functions (add_tags, add_source, import_json) require calling it or rely on mtime.
- [general] `deepcopy-read-only-network-pattern` — The `what_if_*` functions depend on this pattern; investigate whether the invariant "never mutate read-only `Network`" is checked anywhere or is a purely social contract.

## Beliefs

- `_with-network-caches-network-keyed-by-db-path-and-mtime` — The cache value is exactly the tuple `(mtime, Network)` keyed by `db_path`; changing either signature element elsewhere breaks cache hits.
- `_with-network-write-mode-bypasses-cache-hit` — A `write=True` call always reopens `Storage` and `load()`s fresh, even if `_network_cache` has a matching entry.
- `_with-network-skips-save-and-cache-update-on-body-exception` — When the `with` body raises, `store.save` is skipped, no pub/sub event fires, and the cache is intentionally left untouched.
- `_with-network-requires-caller-to-avoid-mutating-read-only-network` — The read-only path may return the shared cached `Network`; callers must `copy.deepcopy` before mutating (as `what_if_retract` and `what_if_assert` do).
- `_with-network-publishes-deltas-only-when-subscribers-exist-at-enter` — `pubsub.has_subscribers` is checked inside `__enter__`; if false, no `_before` snapshot is captured and no events are published on exit even if a subscriber arrives mid-write.
- `_with-network-save-and-publish-are-not-in-try-finally` — A failure in `store.save()` or `pubsub.publish()` short-circuits `store.close()`, leaving the SQLite connection open.
