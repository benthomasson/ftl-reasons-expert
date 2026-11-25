---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Understand why unsynchronized cache clearing can race with `_with_network` context cleanup.

## Understanding why unsynchronized cache clearing can race with `_with_network` cleanup

### The pieces involved

| Piece | Where | Role |
|---|---|---|
| `_network_cache` | `reasons/api.py:33` | Module-level dict `{db_path: (mtime, Network)}`. Process-wide, shared by all threads. |
| `_with_network` enter/read | `reasons/api.py:130` | Looks up `cached = _network_cache.get(db_path)` and uses it if mtime matches. |
| `_with_network` exit/write | `reasons/api.py:164` | Assigns `_network_cache[db_path] = (mtime, self.network)` after a successful write context. |
| `clear_network_cache` | `reasons/api.py:172–181` | Calls `_network_cache.clear()` (line 179) or `_network_cache.pop(db_path, None)` (line 181). |

`_with_network` is used ~94 times across `reasons/api.py` (see `find_with_network_usages`), and the write path ends with an **unconditional** assignment back into the cache. That assignment is the "cleanup" step: after a write context finishes, the in-memory network is (re)published as the cached truth source.

### Why the cleanup can race with a clear

Consider two concurrent paths on the same `db_path`:

1. **Thread A** enters `with _with_network(db, write=True) as net:` and mutates `net`.
2. **Thread B** calls `clear_network_cache(db)`. This executes `_network_cache.pop(db, None)` at line 181.
3. Thread A exits cleanly. Its `__exit__` writes to storage and then runs `_network_cache[db_path] = (mtime, self.network)` (line 164).

B's invalidation is undone by A's assignment. The resulting cache entry may hold a network that B believed was invalidated. If B's clear was triggered by an **external** write to the `.db` file (which is exactly what `test_external_modification_invalidates_cache` exercises), A's `os.stat`-derived mtime can be captured at a moment that no longer reflects the file, or the re-populated entry never gets mtime-invalidated at line 130 because the mtime matches what A just stat'd.

### Why this is technically a race and not just "last writer wins"

Neither `_with_network` nor `clear_network_cache` takes any lock. In CPython the individual `dict.pop` at line 181 and the `dict.__setitem__` at line 164 are atomic *as operations*, but the **check-then-act** logic inside `_with_network.__exit__` (persist, capture mtime, insert) is not atomic with respect to `clear_network_cache`. Lines 130 and 164 bracket the entire body of a `_with_network` call; a call to `clear_network_cache` at any point in between sees one of:

- **Before line 164**: cache is empty, then A re-inserts. Reader for `write=False` after B's clear sees A's write. Correct if A's write was intended, wrong if B was invalidating stale state.
- **After line 164**: work was already re-inserted and B doesn't clear it. The clear is a no-op.
- **Interleaved with mtime capture**: the mtime recorded in the tuple at line 164 can be captured *before* B's `os.utime`/external write, so the entry claims a fresh mtime while holding an old network, or vice versa.

### Semantics the tests encode (and the race violates)

From `tests/test_network_cache.py`:

- `test_write_updates_cache` — after `add_node`, `"b"` must appear in `_network_cache[db][1].nodes`. So line 164 is a **required** cleanup step.
- `test_write_exception_does_not_poison_cache` — if the write context raises, `_network_cache[db][1]` must still be the *original* network. That means line 164 is skipped on exception; the pre-existing cache entry must be preserved.
- `test_clear_cache_specific_path` — clearing one `db_path` must not touch others. `pop(db_path)` at line 181 implements that.

All three tests are single-threaded, so they pass. There is no test that interleaves `clear_network_cache` with an in-flight `_with_network(write=True)` body — which is exactly the case where the race manifests. The cleanup contract the tests enforce ("on success, cache reflects the write; on failure, cache is untouched") is not enforced against **concurrent clears**, only against the sequential exception path.

### Why `mtime` doesn't save you

The read path at line 130 uses `mtime` to detect stale cache entries. Two properties undermine that defence once a concurrency bug exists:

1. **Coarse mtime granularity** on some filesystems means a write-and-clear can land in the same mtime tick, so line 130's check treats the entry as fresh.
2. **Ordering:** if `clear_network_cache` happens between A's storage write and A's `os.stat`, A records the mtime it observed — not necessarily the file's current mtime after B has possibly triggered a further external write. The entry then looks "fresh" while holding state that never matched the file.

### What the observations don't yet show

To justify a fix (rather than just explain the theoretical race), you'd still want:

- The **full body** of `_with_network` (`reasons/api.py:116`), specifically the ordering inside `__exit__` between the storage write, the mtime capture, and line 164.
- All **call sites of `clear_network_cache`** — particularly whether any run inside a `_with_network` body (`reasons/api.py:1611` is a comment noting a nested context, which is a hint).
- Whether any of the ~94 `_with_network` usages are ever entered concurrently for the same `db_path` in the same process (tests, workers, threads).

Without those, the claim "unsynchronized cache clearing races with `_with_network` cleanup" is well-grounded in the observed structure but not yet demonstrated by a reproducer.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Confirm the exact ordering of storage-write → mtime-capture → line 164 insertion inside `__exit__`
- [function] `reasons/api.py:clear_network_cache` — Enumerate call sites; check whether any are reachable from inside a `_with_network` body (see `reasons/api.py:1611` note about a nested context)
- [file] `tests/test_network_cache.py` — Add a concurrency test that runs `clear_network_cache` in a separate thread while a `write=True` body is in flight
- [general] `cache-invalidation-locking` — Evaluate a `threading.Lock` around `_network_cache` (or per-`db_path` `RLock`) as the minimal fix
- [general] `mtime-invalidation-correctness` — Whether mtime-granularity collisions can mask the race even after locking

## Beliefs

- `_network_cache-is-process-wide-and-shared` — `_network_cache` at `reasons/api.py:33` is a module-level dict keyed by `db_path`, so all threads/contexts in the process share one cache instance.
- `write-context-updates-cache-unconditionally` — On successful exit of a `write=True` `_with_network` context, `_network_cache[db_path] = (mtime, self.network)` at line 164 is executed with no check that the cache was cleared concurrently.
- `clear_network_cache-operates-without-lock` — `clear_network_cache` (lines 172–181) uses bare `dict.clear`/`dict.pop` with no synchronization against `_with_network`'s insert at line 164.
- `read-path-relies-on-mtime-for-invalidation` — `_with_network`'s read branch at line 130 treats a cached entry as valid iff its stored mtime matches the current file mtime.
- `clear-race-can-repopulate-invalidated-entries` — A `clear_network_cache` call interleaved with a concurrent `write=True` `_with_network` exit can leave a fresh cache entry whose network/mtime pair does not reflect the state the clearer intended to invalidate.
