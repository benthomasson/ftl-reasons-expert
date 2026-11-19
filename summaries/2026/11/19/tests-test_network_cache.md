---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_network_cache.py

# Explanation of `tests/test_network_cache.py`

## Purpose

This is a focused unit-test suite for the **in-memory network cache** used by `reasons/api.py`. It verifies the correctness of three exported cache primitives:

- `_network_cache` — the module-level cache dictionary
- `_with_network(db_path, write=...)` — the context manager that loads/caches a `Network` object
- `clear_network_cache(db_path=None)` — manual cache eviction

The file owns one responsibility: **guaranteeing that cache reads, cache bypasses on write, mtime-based invalidation, and mutation isolation all behave correctly** so higher-level API functions (`search`, `show_node`, `what_if_retract`, `what_if_assert`, `add_node`, `retract_node`) can safely lean on the cache.

---

## Key Components

### Global state under test
- **`_network_cache: dict[str, tuple[float, Network]]`** — maps `db_path` → `(mtime, network)`. The index `[0]` is the file's last-modified timestamp; `[1]` is the loaded `Network` object (with a `.nodes` dict whose values have `.text` and `.truth_value`).
- **`_with_network(db_path, write=False)`** — context manager.
  - `write=False` → return cached `Network` if the on-disk mtime matches the cached mtime; otherwise load fresh and store `(mtime, net)`.
  - `write=True` → **always** load a fresh `Network` (bypassing the cached object), yield it, and only *after* a successful exit store `(mtime, net)` back.
- **`clear_network_cache(db_path=None)`** — with an argument, pops a single entry; without, empties the dict.
- **`setup_function()`** — a pytest hook run before each test, clearing the entire cache to guarantee test isolation.

### The tests and their contracts
| Test | Contract enforced |
|---|---|
| `test_read_populates_cache` | A read (`write=False`) stores `_network_cache[db]` with `nodes["a"].text == "First belief"`. |
| `test_cached_read_skips_storage` | Two sequential reads return **the same object** (`net1 is net2`). |
| `test_write_loads_fresh` | A `write=True` context returns a **distinct** object (`net1 is not net2`). |
| `test_write_updates_cache` | After `add_node("b", …)`, the cache contains node `"b"` — writes must repopulate the cache. |
| `test_external_modification_invalidates_cache` | Touching the file (`os.utime`) changes mtime, so a subsequent read replaces the cache entry (`_network_cache[db][0] != cached_mtime`). |
| `test_clear_cache_specific_path` | `clear_network_cache(db1)` removes only `db1`, leaving `db2` intact. |
| `test_clear_cache_all` | `clear_network_cache()` empties the dict (`len == 0`). |
| `test_write_exception_does_not_poison_cache` | If the write body raises, the cache entry still holds the pre-write `"Original belief"` text — mutations on the temporary write copy never leak. |
| `test_search_uses_cache` | `search("widget")` returns the same result on the second (cached) call. |
| `test_show_node_uses_cache` | `show_node("test-node")` returns `{"id": "test-node", …}` on both cold and warm calls. |
| `test_what_if_retract_does_not_corrupt_cache` | `what_if_retract` mutates a copy; the cached `Network` still shows `premise`/`conclusion` with `truth_value == "IN"`. |
| `test_what_if_assert_does_not_corrupt_cache` | Symmetrically, `what_if_assert` leaves the cached (already `OUT`) node untouched. |

---

## Patterns

- **Pytest `setup_function` fixture** — no `@pytest.fixture` decorator; the bare `setup_function()` name is the legacy xUnit-style hook pytest auto-invokes before every test in the module. This is the single source of cache isolation.
- **Temporary DB directories** — every test wraps its DB in `with tempfile.TemporaryDirectory() as tmpdir:` so files are discarded after the test, but the **cache is not** (that's what `setup_function` cleans).
- **Identity assertions for cache semantics** — `net1 is net2` / `net1 is not net2` are the strongest possible assertions for "was this actually cached/reloaded?". They depend on `_with_network` returning the *same* object on a hit and a *different* one on a bypass — an intentional contract.
- **Mutation-isolation tests** — `test_write_exception_does_not_poison_cache`, `test_what_if_retract…`, and `test_what_if_assert…` all demonstrate that copies, not the cached instance, are mutated. This is the defense-in-depth rule: **the cached network is immutable from the caller's perspective**.
- **Index-based inspection of a tuple value** — `_network_cache[db][0]` (mtime) and `[1]` (network) are read directly rather than through an accessor. Simple, at the cost of coupling tests to the tuple shape.

---

## Dependencies

### Imports
- **stdlib**: `tempfile`, `pathlib.Path`, `os`, `time`
- **`reasons.api`** (the package under test):
  - `_network_cache`, `_with_network`, `clear_network_cache` — internals
  - `init_db`, `add_node` — setup helpers
  - `search`, `show_node` — read-path coverage
  - `what_if_retract`, `what_if_assert` — copy-only simulation coverage
  - `retract_node` — imported **inside** `test_what_if_assert_does_not_corrupt_cache` to avoid a module-level import that the other tests don't need

### What depends on this file
Nothing in the shipping code depends on tests. Within the test suite, `test_network_cache.py` is a dedicated regression suite for `reasons/api.py`; closely related is `tests/test_network.py` (which presumably exercises networking/merkle/MCP, not the cache).

> Note: the repository has both `reasons/api.py` and `reasons_lib/api.py`. This test imports from `reasons`, so it validates that copy; a change to `reasons_lib/api.py` alone would not be covered here.

---

## Flow

The typical test shape is:

1. `setup_function()` empties `_network_cache`.
2. `with tempfile.TemporaryDirectory() as tmpdir:` creates a throwaway DB path.
3. `init_db(db_path=db)` — creates schema; may or may not populate the cache.
4. `add_node(..., db_path=db)` — a `write=True` operation; after this the cache holds the new network.
5. `clear_network_cache()` (in tests that want a cold start).
6. Exercise the API:
   - **Read path** (`_with_network(db)`, `search`, `show_node`) → `_with_network` compares on-disk mtime to `_network_cache[db][0]`, returning the cached `[1]` when they match.
   - **Write path** (`add_node`, explicit `_with_network(db, write=True)`) → fresh load, yield, and on success write `(mtime, net)` back.
   - **Simulation path** (`what_if_retract`, `what_if_assert`) → mutate a fresh-copy, never repopulate the cache.
7. Inspect `_network_cache` directly to confirm entry existence, mtime, identity, and network contents.

---

## Invariants

1. **Cache key is the exact `db_path` string** — `db1` and `db2` are distinct keys (`test_clear_cache_specific_path`).
2. **Cache value is a 2-tuple `(mtime, network)`** — anchoring mtime to the file was last loaded is what makes external-modification invalidation possible.
3. **Read is "cache-hit if mtime matches"** — a read with a stale mtime must produce a *different* object; the test asserts by identity and by mtime value (`test_external_modification_invalidates_cache`).
4. **Write bypasses cache and reloads fresh** — `test_write_loads_fresh`.
5. **Successful writes update the cache** — `test_write_updates_cache`.
6. **Failed writes do NOT update the cache** — the exception path in `_with_network` must not store `(mtime, net)`.
7. **Simulations never mutate the cached `Network`** — `what_if_*` must `deepcopy` / reload, then discard; the cached `truth_value` must be unchanged after the call.
8. **The cache is global per-process** — `setup_function()` and `clear_network_cache()` are the only eviction mechanisms besides mtime mismatch.

---

## Error Handling

- The only exception the tests *induce* is `RuntimeError("simulated failure")` inside `test_write_exception_does_not_poison_cache`. The test wraps the `_with_network` block in `try/except RuntimeError: pass`, then asserts the cache still reflects the pre-write state.
- This encodes an **error-semantics invariant**: `_with_network(write=True)` must treat any exception as "abort write" — the cache must remain unchanged (or absent), and the exception must propagate to the caller. It must **not** swallow the exception, and it must **not** commit a partially-mutated copy.
- No other test exercises error paths; `search`, `show_node`, and `what_if_*` are assumed non-throwing for existing nodes. (If those functions ever start raising on missing nodes, additional tests would be needed.)

---

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The context manager this entire file is validating; inspect exactly how it branches on `write`, when it calls `os.path.getmtime`, and where the tuple is inserted.
- [function] `reasons/api.py:clear_network_cache` — Verify the `db_path=None` vs `db_path="…"` semantics and whether it also affects any secondary caches (e.g. per-node indexes).
- [function] `reasons/api.py:what_if_retract` — Check how it makes a deep-copy (or fresh reload) of the network so that `test_what_if_retract_does_not_corrupt_cache` holds.
- [file] `tests/test_network.py` — Compare how the non-cache networking tests construct/inspect networks, to see whether they assume a warm or cold cache.
- [general] `reasons-vs-reasons_lib-duplication` — Determine whether `reasons_lib/api.py` mirrors this cache logic and whether the same test should be added there.

## Beliefs

- `network-cache-keyed-by-db-path` — `_network_cache` is a dict keyed by DB path strings whose values are `(mtime, network)` tuples.
- `read-with-matching-mtime-returns-cached-object` — `_with_network(db, write=False)` returns the identical cached `Network` object when the on-disk mtime matches the cached mtime, proven by `net1 is net2`.
- `write-context-bypasses-cache` — `_with_network(db, write=True)` always yields a freshly loaded `Network` distinct from any cached one, proven by `net1 is not net2`.
- `failed-write-prevents-cache-population` — An exception raised inside a `write=True` context leaves the prior cache entry untouched (`test_write_exception_does_not_poison_cache`).
- `what-if-operations-mutate-copies` — `what_if_retract` and `what_if_assert` operate on a copy of the network and never mutate `_network_cache[db][1]`.
- `cache-preconditions-require-clear-between-tests` — All tests rely on the pytest `setup_function` hook to empty `_network_cache` before execution; without it, entries would leak because `TemporaryDirectory` only cleans the filesystem, not the dict.
