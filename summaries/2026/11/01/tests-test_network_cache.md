---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_network_cache.py

# `tests/test_network_cache.py`

## Purpose

This file is the **integration test suite for the in-memory network cache** that wraps the belief-network object in `reasons.api`. Its single responsibility is to verify the caching contract of `_with_network()` and `clear_network_cache()`: that reads are served from cache, writes invalidate and refresh it, external file changes are detected via mtime, and—critically—that **partial or failed mutations never leak into the cached state**.

The belief-network (a graph of nodes with IN/OUT truth values, justifications, and structural links) is expensive to rebuild from SQLite. The cache exists to amortize that cost across repeated read-only API calls (`search`, `show_node`, `what_if_*`) within a process. This test file is the safety net ensuring that amortization never silently serves stale or corrupted data.

## Key Components

### Under-test symbols (from `reasons.api`)

| Symbol | Contract |
|---|---|
| `_network_cache` | Module-level `dict[str, tuple[int, Network]]` mapping a resolved DB path → `(mtime, network_object)`. The tests import it directly to inspect internal state. |
| `_with_network(db_path, write=False)` | Context manager. **Read mode** (`write=False`): returns the cached `Network` if the mtime matches, otherwise loads from disk and populates the cache. **Write mode** (`write=True`): always loads a fresh copy from disk; on clean exit, stores it back into the cache; on exception, discards it. |
| `clear_network_cache(path=None)` | If `path` is given, deletes only that entry. If `None`, empties the entire cache dict. |
| `add_node`, `search`, `show_node`, `what_if_retract`, `what_if_assert`, `retract_node` | Higher-level API functions that internally call `_with_network`. The tests use them as black-box consumers to verify cache interaction. |

### Test fixtures / helpers

- **`setup_function()`** (pytest hook): calls `clear_network_cache()` before every test to guarantee isolation. Without this, a leaked cache entry from a prior test would cause false positives/negatives.
- **`tempfile.TemporaryDirectory()`**: each test creates a fresh SQLite file so tests don't share on-disk state.

### Test functions and what each pins down

| Test | Invariant verified |
|---|---|
| `test_read_populates_cache` | First read in read-mode inserts `(mtime, net)` into `_network_cache`. |
| `test_cached_read_skips_storage` | Second read returns the **same object identity** (`net1 is net2`), proving no re-load from disk. |
| `test_write_loads_fresh` | `write=True` breaks identity (`net1 is not net2`), confirming a fresh load. |
| `test_write_updates_cache` | After `add_node` (a write), the cache reflects the new node. |
| `test_external_modification_invalidates_cache` | Bumping the file mtime via `os.utime` causes the next read to detect the change (`_network_cache[db][0] != cached_mtime`). |
| `test_clear_cache_specific_path` | `clear_network_cache(db1)` removes only `db1`; `db2` survives. |
| `test_clear_cache_all` | `clear_network_cache()` with no arg empties the dict. |
| `test_write_exception_does_not_poison_cache` | A `RuntimeError` inside a write context leaves the **pre-exception** cached network intact. This is the most safety-critical test. |
| `test_search_uses_cache` / `test_show_node_uses_cache` | Read-only API calls produce correct results across two invocations (implicitly using the cache). |
| `test_what_if_retract_does_not_corrupt_cache` | `what_if_retract` mutates a *copy*; the cached original still shows both nodes as `IN`. |
| `test_what_if_assert_does_not_corrupt_cache` | Same guarantee for the assert direction; the cached node remains `OUT` after a `what_if_assert`. |

## Patterns

- **Stateful unit-test isolation via `setup_function`**: rather than a pytest fixture with yield/teardown, the module uses the older `setup_function` hook. This is a pytest 4-era idiom; each test also clears the cache explicitly, giving belt-and-suspenders isolation.
- **Temporary directory per test**: `tempfile.TemporaryDirectory` as a context manager ensures the SQLite file is created and destroyed within the test's scope. No shared state across tests.
- **Identity-based assertion** (`net1 is net2`): used to prove the cache short-circuits disk I/O. This is stronger than `==` because it confirms object reuse, not just value equality.
- **Mutation-then-verify pattern**: several tests mutate a network inside a `with` block, then immediately inspect `_network_cache[db][1].nodes` to confirm the cached copy was *not* touched. This tests the "copy-on-write" discipline of the read-path.
- **Negative testing via exception injection**: `test_write_exception_does_not_poison_cache` raises `RuntimeError` inside the context to simulate a crash mid-write, then asserts the cache is pristine.

## Dependencies

### Imports (what this file depends on)

- **`tempfile`, `pathlib.Path`** — standard-library utilities for isolated DB files.
- **`reasons.api`** — the production module under test. All tested symbols (`_network_cache`, `_with_network`, `add_node`, `clear_network_cache`, `init_db`, `search`, `show_node`, `what_if_retract`, `what_if_assert`, `retract_node`) come from here.
- **`os`, `time`** (imported inline in `test_external_modification_invalidates_cache`) — to bump mtime and add a small sleep so the mtime actually changes on disk.
- **`reasons.network`** (transitively) — the `Network` class whose `.nodes` dict and `.text`/`.truth_value` attributes are asserted on.

### Importers (what depends on this file)

Nothing imports this test file. It is a leaf in the dependency graph. However, the symbols it tests (`_with_network`, `_network_cache`) are consumed by nearly every public API function in `reasons.api` (`search`, `ask`, `review`, `derive`, `repair`, `what_if_*`, etc.), so a regression caught here would protect the entire API surface.

## Flow

```
setup_function()
  └─ clear_network_cache()          ← reset global state

test body:
  ├─ init_db(db_path=db)            ← create empty SQLite
  ├─ add_node("a", …, db_path=db)   ← writes through _with_network(write=True)
  │    └─ _with_network loads fresh net, mutates, stores in _network_cache
  ├─ clear_network_cache()          ← (some tests) force a cold start
  │
  ├─ with _with_network(db) as net: ← read mode
  │    ├─ mtime check: os.stat(db).st_mtime == _network_cache[db][0]?
  │    │    ├─ YES → return _network_cache[db][1]   (cache hit)
  │    │    └─ NO  → load from disk, store (mtime, net), return
  │    └─ test asserts on net.nodes / identity
  │
  └─ inspect _network_cache[db]     ← verify internal bookkeeping
```

The critical branch is in **write mode with exception**:

```
with _with_network(db, write=True) as net:
    net.nodes["a"].text = "CORRUPTED"
    raise RuntimeError(...)
    # ↓  __exit__ detects exception
    #    → does NOT update _network_cache
    #    → original cached net is preserved
```

## Invariants

1. **Cache key is the DB file path string.** Two different paths (even to the same file via symlink) would be separate cache entries. The tests use absolute temp paths so this is unambiguous.
2. **Read-mode returns the exact same `Network` object** on repeated calls (identity, not equality). This means any accidental in-place mutation by a caller corrupts the shared cache — which is exactly why `what_if_*` functions must copy.
3. **Write-mode is transactional with respect to the cache.** Either the full `__exit__` succeeds and the new network replaces the old, or an exception causes the new network to be discarded and the old cache entry to remain untouched.
4. **mtime is the invalidation mechanism.** The cache does not use file descriptors, inode numbers, or content hashes. A file whose mtime is unchanged but whose content changed (e.g., `truncate` + rewrite in the same tick) would be a false cache hit. The `time.sleep(0.05)` in the mtime test guards against filesystem timestamp granularity.
5. **`setup_function` runs before every test**, but individual tests also call `clear_network_cache()` after setup. This double-clear is defensive: it protects against a test that *populates* the cache in its setup phase and then expects a cold start.

## Error Handling

The only error-handling path under test is **exception propagation out of a write context**:

- `test_write_exception_does_not_poison_cache` raises `RuntimeError` inside `with _with_network(db, write=True)`. The context manager's `__exit__` receives `(exc_type, exc, tb)` and must skip the cache-update step. The test then asserts the cached node text is still `"Original belief"`, not `"CORRUPTED"`.
- No other test injects errors. There is no test for what happens if `init_db` fails, if the DB file is deleted between cache-check and load, or if the SQLite file is locked. These are gaps a reviewer might flag.
- `test_external_modification_invalidates_cache` does not actually cause an error — it only verifies that the mtime bookkeeping updates. It does **not** verify that the *network content* is reloaded (it could assert `"a" in net.nodes` to close that loop).

A notable absence: there is no test for concurrent access (two threads calling `_with_network` simultaneously), no test for the case where `db_path` is a relative path (which would break the mtime-key lookup), and no test for cache behavior when the network is very large (memory pressure).

---

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The context manager under test; understanding its `__enter__`/`__exit__` logic (especially the exception path) is essential to interpreting every test in this file.
- [file] `reasons/network.py` — Defines the `Network` class, its `.nodes` dict, and truth-value semantics (`IN`/`OUT`) that the tests assert on. Needed to understand what "corrupting the cache" means structurally.
- [function] `reasons/api.py:what_if_retract` — The "safe copy" pattern: this function must deep-copy (or at least shallow-copy) the cached network before mutating it. This test file verifies the *effect* of that copy but not the *mechanism*.
- [file] `tests/test_network.py` — The companion test file for the `Network` class itself (node CRUD, justification links, structural integrity). Reading it alongside this file clarifies the boundary between network-internal correctness and cache-management correctness.
- [general] `cache-invalidation-strategy` — The project uses mtime-based invalidation exclusively. Understanding whether this is sufficient for all deployment targets (e.g., network-mounted filesystems with coarse timestamps, or in-memory SQLite) is a latent risk worth investigating.

## Beliefs

- `_with_network-write-mode-must-be-atomic-with-respect-to-cache` — If the `with` block raises any exception, the previously cached `Network` object is guaranteed to remain unchanged; the in-flight mutated copy is discarded.
- `_with_network-read-mode-returns-same-object-identity` — Repeated read-mode calls for the same `db_path` (with unchanged mtime) return the identical `Network` instance, so callers must not mutate it in place.
- `_network_cache-keyed-by-db-path-string` — The cache is a plain `dict` keyed by the string form of the database path; it is not thread-safe and does not use locks or atomic operations.
- `what-if-operations-copy-before-mutate` — `what_if_retract` and `what_if_assert` operate on a copy of the cached network, leaving the canonical cached object's `truth_value` fields unaltered.
- `mtime-is-sole-invalidation-signal` — The cache has no fallback invalidation mechanism (no version counter, no content hash, no reference-counting); a change that does not alter the file's mtime will produce a stale cache hit.
