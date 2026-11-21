---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: remove_justification in reasons/api.py

# `remove_justification` — Analysis

## Purpose

`remove_justification` is a thin **backend-dispatching facade** that removes a single justification (by 0-based index) from an existing node and re-propagates the network. It lives in the "functional Python API" layer (`reasons/api.py`), whose job is to abstract away the `Storage` lifecycle and the choice between SQLite and PostgreSQL backends, so callers (CLI, LangGraph tools, scripts) can call plain functions that return JSON-serializable dicts and manage their own transaction boundaries.

The function itself contains **no business logic**: it exists solely to (a) dispatch to PG when `pg_conninfo` is given, and (b) wrap the SQLite path in the load-operate-save lifecycle via `_with_network`.

## Contract

**Preconditions (implicit, not type-enforced):**
- `node_id` names an existing node in the network at `db_path`.
- `index` is a **0-based** integer in `[0, len(node.justifications))`.
- The justification at `index` is currently present on the node.
- `net.remove_justification` exists on `Network` with the signature `(node_id: str, index: int) -> dict`.
- The method on `PgApi` has matching keyword parameters `node_id` and `index`.

**Postconditions (on success):**
- The justification is removed from the node in persisted storage (SQLite) or PG.
- The whole network is re-propagated so dependent truth values ("IN"/"OUT") reflect the loss of that justification.
- The cached `Network` object for `db_path` is refreshed (SQLite path only, on success).
- If there are subscribers, pubsub change events are computed and published (SQLite path only).
- A dict describing the result is returned.

**Invariants:**
- On exception, **nothing is persisted** — `_with_network.__exit__` only calls `store.save(...)` when `exc_type is None`, so failures roll back cleanly (the in-memory mutation is discarded; the cached network is also not refreshed).
- A failed call leaves the database file byte-for-byte unchanged (with respect to this operation) and does **not** invalidate the cache.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `node_id` | `str` | The node whose justification is being removed | Must be fully qualified (`ns:id`) — this function does **not** take a `namespace` kwarg. Missing node → `KeyError` from the network layer. |
| `index` | `int` | 0-based index into `node.justifications` | Negative or out-of-range indices → `IndexError` (or equivalent) from `Network.remove_justification`. `bool` is technically a subclass of `int` in Python but is not intended. |
| `db_path` | `str` | SQLite database path | Ignored when `pg_conninfo` is provided. Default `"reasons.db"`. |
| `pg_conninfo` | `str \| None` | PostgreSQL connection info | Falsy → SQLite path. Truthy → PG dispatch. |
| `domain_id` | `Any` | PG tenant/domain selector | Only consumed by `_pg_dispatch`. |

**Assumption not enforced by types:** `node.justifications` may be **empty** (a premise). Passing `index=0` to a premise node should fail — but nothing in this function checks that; the caller must know the node has justifications. Removing the *only* justification effectively turns the node into a premise, which is a meaningful state change the caller probably wants to be aware of.

## Return Value

Returns the **unmodified** return value of `Network.remove_justification` (SQLite) or `PgApi.remove_justification` (PG). Based on the docstring and the pattern used by sibling functions (`add_justification`, `assert_node`, `retract_node`), the returned dict almost certainly contains keys such as `"changed"`, `"old_truth_value"`, `"new_truth_value"`, or `"went_out"`/`"went_in"` lists. **Don't depend on that here** — this file does not create the dict; it passes whatever the network layer produces straight through.

**Caller obligations:**
- Catch `KeyError` (missing node), `IndexError`/`ValueError` (bad index), and DB exceptions (`sqlite3.*`, `psycopg.*`) — this function does not translate or swallow any of them.
- Do not mutate the returned dict.

## Algorithm

```
1. If pg_conninfo is truthy:
   a. Import PgApi lazily (from .pg).
   b. Open PgApi(pg_conninfo, domain_id).
   c. Call PgApi.remove_justification(node_id=..., index=...).
   d. Return the result.
   (No caching, no pubsub, no separate propagation call — PG backend handles it.)

2. Otherwise (SQLite path):
   a. Enter _with_network(db_path, write=True):
      - write=True forces a FRESH Storage(db_path).load() — the in-memory
        cache is bypassed even if the file mtime matches.
      - __enter__: if pubsub.has_subscribers(db_path), snapshot
        {nid: truth_value} for all nodes (pre-change baseline).
   b. Inside the with-block, call net.remove_justification(node_id, index).
      - Under the hood: validates node/index, removes justification, calls
        recompute_all() to re-derive truth values, updates node.updated_at,
        cleans up the dependents reverse-index.
   c. Return that dict from the with-block.
   d. __exit__ (success path):
      - store.save(net) — persists ALL nodes, justifications, nogoods.
      - Recompute after-snapshot, diff via pubsub.compute_changes,
        publish to subscribers.
      - store.close().
      - Refresh _network_cache[db_path] = (fresh mtime, net).
   e. __exit__ (exception path):
      - Skip save, skip pubsub, skip cache update.
      - Still close the Storage connection.
      - Re-raise the original exception.
```

## Side Effects

- **Disk write:** The entire network is serialized to `db_path` on success (SQLite path). This is a full save, not an in-place patch.
- **In-memory cache:** `_network_cache[db_path]` is replaced with the post-save `Network`.
- **Pubsub:** If `pubsub.has_subscribers(db_path)` is true at entry, before/after snapshots are diffed and `pubsub.publish(events, db_path)` is called. Callers who subscribed will see change notifications for every node whose truth value moved.
- **Network internal state:** The node's `justifications` list is mutated, `dependents` reverse-pointers on antecedent nodes are cleaned, `updated_at` is bumped, and dependent truth values may flip during propagation.
- **PG path:** A connection/handshake is opened and closed via `_pg_dispatch`; no local cache or pubsub involvement.
- **Thread/process safety:** The cache and pubsub registry are module-level globals (`_network_cache`, `pubsub`'s internal state) — concurrent callers within one process share them.

## Error Handling

**No `try/except` in this function.** Exceptions propagate unmodified:

| Exception | When |
|---|---|
| `KeyError` | `node_id` not present in `net.nodes` |
| `IndexError` / `ValueError` | `index` out of range (raised by `Network.remove_justification`) |
| `FileNotFoundError` / `OSError` | `db_path` does not exist / not readable (from `Storage(db_path)` construction or `os.path.getmtime`) |
| `sqlite3.OperationalError` | Corruption, locked DB, etc., during load or save |
| PG driver exceptions | Bazillions of them, from `_pg_dispatch` |

**Rollback semantics on the SQLite path:** Because `_with_network.__exit__` checks `exc_type is None` before saving, any exception thrown inside the `with` block leaves the DB untouched. However, **in-memory mutations are not undone** — `self.network` was mutated. But `__exit__` also skips the cache refresh in that case, so the *cached* Network (if any) retains its old state; next read either loads fresh (if write=True) or reuses stale cache (write=False). This is a subtle asymmetry worth knowing.

## Usage Patterns

Typical call from a CLI or agent tool:

```python
# Safely remove the 0th justification, then report what cascaded OUT.
try:
    result = remove_justification("claim-42", index=0, db_path="reasons.db")
    print("Changed:", result.get("changed", []))
except (KeyError, IndexError) as e:
    print(f"Cannot remove justification: {e}")
```

Typical pre-inspection pattern (recommended):

```python
detail = show_node("claim-42")
n_justifications = len(detail["justifications"])
if 0 <= i < n_justifications:
    remove_justification("claim-42", i)
```

**Caller obligations:**
- Must pass `node_id` fully qualified when the node is namespaced (unlike `add_justification`, this function has **no `namespace` parameter** to prefix IDs for you).
- Must catch exceptions; a return dict does not indicate "error" — the function either succeeds or raises.
- Should be aware that a successful call does not mean "the node went OUT" — removing one justification may leave the node IN if other justifications support it.
- Should not call this from inside a `with _with_network(...)` block — that would create a second Storage instance and the inner save could clobber changes from the outer one.

## Dependencies

**Internal:**
- `reasons/api.py:_with_network` — load/save lifecycle context manager.
- `reasons/api.py:_pg_dispatch` — PG backend dispatcher.
- `reasons/storage.py:Storage` — SQLite persistence.
- `reasons/network.py:Network.remove_justification` — the real implementation.
- `reasons/pubsub` — subscriber registry and event computation.
- `reasons/pg.PgApi` (lazily imported only when `pg_conninfo` is truthy).

**External:**
- `sqlite3` (transitively via `Storage`).
- Whatever the PG driver is (`psycopg`, etc.) via `PgApi`.
- `os.path.getmtime` in `_with_network`.

---

## Topics to Explore

- [function] `reasons/network.py:Network.remove_justification` — The real logic: how `recompute_all()` handles the case where the removed justification was the *supporting* one, and whether it deletes the justification in-place or rebuilds the list.
- [file] `tests/test_remove_justification.py` — Concrete expected behavior for edge cases: removing the only justification of a premise, removing a justification that no dependents rely on, out-of-range indices.
- [file] `reasons/pubsub.py` — How `compute_changes` derives IN→OUT / OUT→IN events from the before/after truth-value snapshots, and how subscribers are registered per `db_path`.
- [function] `reasons/api.py:_with_network` — The cache-then-load-then-save dance; specifically why `write=True` bypasses the cache and why cache refresh is skipped on exception.
- [function] `reasons/pg.py:PgApi.remove_justification` — Whether the PG implementation propagates internally or is a no-op stub, and how connection pooling interacts with the returned dict.

## Beliefs

- `remove-justification-forces-fresh-load` — The SQLite path passes `write=True` to `_with_network`, so it always calls `Storage.load()` and never reuses the cached `Network` even when the file mtime is unchanged.
- `remove-justification-skips-save-on-error` — `_with_network.__exit__` only calls `store.save(...)` when `exc_type is None`; therefore any exception thrown inside `Network.remove_justification` leaves the SQLite database unmodified (but the in-memory `Network` object is not rolled back).
- `remove-justification-has-no-namespace-parameter` — Unlike `add_justification` and `add_node`, `remove_justification` has no `namespace` kwarg, so callers must pass fully qualified IDs for namespaced nodes.
- `remove-justification-publishes-only-with-subscribers` — Pubsub events are computed and published only when `pubsub.has_subscribers(db_path)` returns truthy at `__enter__` time; otherwise no before/after snapshot is even taken.
- `remove-justification-passes-through-return-value` — The function returns the raw dict produced by `Network.remove_justification` or `PgApi.remove_justification` without adding, renaming, or filtering any keys.
