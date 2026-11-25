---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: backfill_hashes in reasons/api.py

## Explanation: `backfill_hashes` (reasons/api.py)

### 1. Purpose

A thin public-API wrapper that populates the Merkle hashes used for integrity verification (`text_hash` on nodes, `content_hash` on justifications, `source_hash` on nodes with source paths). It exists so callers (CLI, LangGraph tools, scripts) can invoke hashing without writing their own Storage/`Network` lifecycle code, mirroring the pattern used by every other function in `reasons/api.py`: open the DB, do the work, save, close.

It is the *write-side* companion to `check_integrity` (which only *verifies* existing hashes) — you run `backfill_hashes` first on a legacy/un-hashed database, then rely on `check_integrity` for tamper detection.

### 2. Contract

**Preconditions**
- `db_path` names a SQLite database that is (or can become) a valid RMS store — the function does *not* take `pg_conninfo`/`domain_id` like the rest of the API, so it is **SQLite-only**. This is an important asymmetry.
- The merkle module (`.merkle`) is importable in the current environment.

**Postconditions**
- Every node/justification missing a hash has one computed and persisted to disk (via `_with_network(write=True)` → `store.save(...)`).
- The in-memory `_network_cache[db_path]` is refreshed with the new mtime + network on success.
- If the function returns normally, the on-disk DB reflects the new hashes; if it raises, changes are discarded (see §7).

**Invariants / assumptions not enforced by types**
- It is assumed `merkle.backfill_hashes(net)` **mutates `net` in place**. The wrapper does nothing with the returned value beyond returning it — any persistence is entirely a side effect of the mutation plus the context-manager save.
- It is assumed hashing is **idempotent** — re-running on an already-hashed network should yield the same hashes (SHA-based), so double-invocation is safe.
- It hashes **all nodes regardless of `access_tags`/visibility** — there is no `visible_to` filter here, unlike `show_node`/`export_network`. Callers who care about tenant isolation must not point this at a shared DB lightly.
- No schema/version check is performed before mutation.

### 3. Parameters

- **`db_path: str = DEFAULT_DB`** — filesystem path to the SQLite RMS database. `DEFAULT_DB` is `"reasons.db"` (relative). If the file does not exist and Storage is unable to initialize it, the function will raise during `Storage(db_path).load()` (behavior delegated to `Storage`). No `~` expansion, no path normalization, no validation.
- **Implicit (missing) parameters**: unlike sibling functions it exposes no `pg_conninfo`, `domain_id`, `visible_to`, `force`, or `dry_run`. This is deliberate: Merkle hashing only makes sense against the local SQLite file.

### 4. Return Value

Returns the dict returned by `merkle.backfill_hashes(net)` — the wrapper does not introspect or reshape it. The docstring implies a summary of what was computed (e.g., counts of nodes/justifications hashed), but the exact shape is defined in `reasons/merkle.py` and is JSON-serializable, consistent with every other API function. The caller must handle whatever fields that dict contains (likely `{"nodes": int, "justifications": int, ...}`).

If the merkle function returns early (e.g., "nothing missing"), that dict is passed through unchanged.

### 5. Algorithm (step by step)

1. **Lazy import** `merkle.backfill_hashes` as `_backfill`. Done inside the function body to avoid a module-level import cycle (`merkle` likely imports from `network`, which `api` also imports) and to keep cold-start cheap.
2. **Enter `_with_network(db_path, write=True)`**:
   - Checks `_network_cache[db_path]` against the file's mtime; because `write=True`, the cache short-circuit is disabled → a fresh `Storage(db_path)` is always opened and `store.load()` produces a `Network`.
   - On `__enter__`, snapshots `{nid: n.truth_value}` into `_before` **only if** `pubsub.has_subscribers(db_path)` is truthy. (Hash backfilling never changes truth values, so the snapshot is pure overhead when subscribers exist — a wasted O(nodes) scan.)
3. **Call `_backfill(net)`**: walks the network, computes any missing `text_hash` / `source_hash` / justification `content_hash`, and sets them on the in-memory node/justification objects. Returns a summary dict.
4. **Return that dict**, still inside the `with` block. The context manager then runs `__exit__`.
5. **`__exit__` (success path)**: because `write=True and exc_type is None`, calls `store.save(self.network)` to persist the mutated hashes, then (if `_before` was captured) computes truth-value deltas via `pubsub.compute_changes` — which will be empty here — and publishes them. Then closes `store` and refreshes `_network_cache[db_path] = (fresh_mtime, net)`.
6. **Control returns** the merkle summary dict to the original caller.

### 6. Side Effects

- **Database writes**: `store.save(network)` rewrites the persisted network (nodes/justifications hashes, plus whatever else Storage persists).
- **In-process global state**: `_network_cache[db_path]` is replaced with the new network object keyed by the post-save mtime.
- **Pubsub**: if any subscribers are registered for `db_path`, `pubsub.publish` is invoked — but with an empty event list, since truth values don't change. If there are no subscribers, no snapshot/pubsub work is performed at all (hasher path only).
- **File I/O**: reads the DB (load) and writes it (save). Updates file mtime.

### 7. Error Handling

- Exceptions from `merkle.backfill_hashes` (e.g., a node with a malformed `source` making a hash impossible to compute) propagate outward and are **not** caught here.
- Because `write=True` and `exc_type is not None`, `__exit__` **skips `store.save`** — so a mid-run failure leaves the on-disk DB untouched (changes discarded). It also **skips the `_network_cache` update** (the `not (write and exc_type is not None)` guard), so stale cache entries with an old mtime survive and will be invalidated by the next `os.path.getmtime` check anyway. `store.close()` still runs.
- `_with_network.__exit__` returns `False`, so the exception is **re-raised**, not swallowed.
- File-missing handling is delegated entirely to `Storage`: `os.path.getmtime` failing inside `_Ctx.__init__` is caught (`OSError`), but the subsequent `Storage(db_path).load()` may raise or create. This function performs no explicit existence check, unlike `init_db`'s `FileExistsError` guard.
- **No idempotency guard** of its own — re-running is safe only because hashing is deterministic.

### 8. Usage Patterns

Typical callers:

```python
from reasons.api import backfill_hashes
result = backfill_hashes(db_path="reasons.db")
print(result)  # e.g. {"nodes": 42, "justifications": 17, ...}
```

- **Migration step** — run once against any pre-Merkle or imported DB before relying on `check_integrity`.
- **Post-import fixup** — after `import_json` / `import_hf` / `import_agent` if the serialized form omitted hashes.
- **CLI** — wired to a `reasons backfill-hashes` subcommand.

**Caller obligations**
- Serialize writes to `db_path` — no explicit lock is taken; two concurrent writers can clobber each other's saves.
- Don't assume the return dict shape is stable across versions.
- Don't rely on pubsub events from this function; if you have subscribers, expect a spurious empty publish and an O(nodes) truth-value snapshot cost.
- This does **not** touch `pg_conninfo` domains — do not route PG callers here.

### 9. Dependencies

- `reasons.merkle.backfill_hashes` — the real worker; imported lazily.
- `_with_network` (same file) — the network lifecycle context manager; provides fresh-load-for-write, atomic save-on-success semantics, cache update, and pubsub.
- `reasons.storage.Storage` — transitively, via `_with_network`.
- `reasons.pubsub` — transitively, on `__exit__` (only if subscribers exist).
- `DEFAULT_DB` constant (`"reasons.db"`) in the same module.

---

## Topics to Explore

- [function] `reasons/merkle.py:backfill_hashes` — The real worker; understand what it mutates and returns, and whether it hashes justifications or only nodes.
- [function] `reasons/api.py:_with_network` — The shared lifecycle contract that all mutation functions depend on; the save/cache/pubsub semantics are subtle and explained in one place.
- [file] `reasons/storage.py` — `Storage.save`/`load` define what actually lands on disk; confirm hash columns are persisted and mtime updates as expected.
- [file] `tests/test_merkle.py` — Shows the expected dict shape and the idempotency guarantee callers can rely on.
- [general] `merkle-integrity-model` — How `text_hash`, `content_hash`, and `source_hash` chain together; critical before discussion of `check_integrity` tamper detection.

## Beliefs

- `backfill-hashes-is-sqlite-only` — Unlike nearly every other function in `reasons/api.py`, `backfill_hashes` has no `pg_conninfo`/`domain_id` parameters and cannot be dispatched to the PostgreSQL backend.
- `backfill-hashes-requires-with-network-write-mode` — Hashes are only persisted because `_with_network(db_path, write=True)` calls `store.save(network)` in `__exit__`; the merkle function itself performs no I/O.
- `write-with-network-exception-discards-changes` — If `merkle.backfill_hashes` raises, `_with_network.__exit__` skips `store.save` and skips the `_network_cache` mtime refresh, so on-disk hashes are unchanged.
- `backfill-hashes-produces-no-pubsub-events` — Hash backfilling never alters `truth_value`, so even with subscribers registered, `pubsub.compute_changes` returns an empty event list.
- `backfill-hashes-assumes-in-place-mutation` — The wrapper returns the merkle dict unmodified and relies on `merkle.backfill_hashes` mutating the passed `Network` in place; that contract is not enforced by types.
