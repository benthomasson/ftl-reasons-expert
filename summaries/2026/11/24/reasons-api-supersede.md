---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: supersede in reasons/api.py

# `supersede` — `reasons/api.py`

## Purpose

`supersede` is the top-level, library-facing entry point for performing a **belief revision** in the Reason Maintenance System (RMS): it records that one node (`old_id`) has been replaced by another (`new_id`), so that the old belief transitions from `IN` to `OUT` while the new one remains `IN`.

Unlike `mark_superseded` (a related function in the same file), which uses a **string-based** approach (`metadata["superseded_by"] = new_id` plus `net.retract(...)`), `supersede` delegates the effect to `Network.supersede`, which — per the `supersede_with_text` docstring — uses the **outlist mechanism** so the change is *reversible* (asserting `new_id` OUT can bring `old_id` back IN).

The function exists to give callers (CLI, LangGraph tools, other Python code) a backend-agnostic, context-managed API that hides:
- `Storage`/`Network` lifecycle (load/save/close)
- The SQLite vs PostgreSQL dispatch decision
- Cache invalidation and pub/sub events on write

## Contract

**Preconditions (implicit — not enforced by this wrapper):**
- Both `old_id` and `new_id` must already exist in the network (or in the PG domain).
- `new_id` is the *replacement* — you generally must have added it (or used `supersede_with_text`) before calling `supersede`.

**Postconditions:**
- `old_id`'s truth value is `OUT` while `new_id` is `IN` (via outlist defeat — reversible).
- With `transitive=True`, every ancestor in the supersession chain is also defeated, so only the latest version remains `IN`.
- Returned `changed` lists every node whose truth value flipped during propagation (this can be larger than `{old_id}`).

**Invariants:**
- The change is applied inside a write transaction (`_with_network(db_path, write=True)`) so it is persisted on context exit.
- Pub/sub subscribers registered for `db_path` receive truth-value change events.

## Parameters

| Param | Type | Default | Meaning / edge cases |
|---|---|---|---|
| `old_id` | `str` | — | Node being superseded. Must exist (delegated key check happens in `Network.supersede`). |
| `new_id` | `str` | — | Successor node. Must exist; identical `old_id == new_id` is not guarded here — behavior is whatever `Network.supersede` does. |
| `db_path` | `str` | `"reasons.db"` (`DEFAULT_DB`) | SQLite file path. Ignored when `pg_conninfo` is set. |
| `transitive` | `bool` | `False` | When `True`, `Network.supersede` walks the supersession chain to defeat ancestors. **Not forwarded to the PostgreSQL backend** — see Contracts and Gotchas below. |
| `pg_conninfo` | — | `None` | If truthy, dispatch to `PgApi.supersede` via `_pg_dispatch`. Non-`None` selects the PG backend and the SQLite path is skipped. |
| `domain_id` | — | `None` | Only used for PG routing; ignored on SQLite. |

## Return Value

A `dict` produced by `Network.supersede`:

```python
{
    "old_id": str,
    "new_id": str,
    "changed": list[str],
}
```

- `changed` is the cascade result of the recompute — it may contain more than just `old_id` (dependents that fell out, siblings that entered IN via outlist-clearing, etc.). The caller must handle arbitrary-sized lists.
- If the node lookup fails inside `Network.supersede`, a `KeyError` propagates — no defensive dict is returned.

## Algorithm

1. **Backend branch.** `if pg_conninfo: return _pg_dispatch(pg_conninfo, domain_id, "supersede", old_id=..., new_id=...)`.
   - Notice **`transitive` is dropped** here — the PG backend always sees the default (`False` on the PG side). Not enforced by types.
2. **SQLite path.** Enter `_with_network(db_path, write=True)`:
   - Because `write=True`, `_with_network._Ctx.__init__` **bypasses the cache** and always calls `Storage(db_path).load()` for a fresh `Network`.
   - `__enter__` snapshots `truth_value` for every node if any pub/sub subscriber exists (to compute change events later).
3. **Call `net.supersede(old_id, new_id, transitive=transitive)`.** The underlying implementation (in `reasons/network.py`, outside this file):
   - Adds `new_id` to the outlist of `old_id`'s justification(s), defeating `old_id` when `new_id` goes `IN`.
   - With `transitive=True`, walks the supersession chain to defeat all ancestors.
   - Recomputes truth values and returns `{old_id, new_id, changed}`.
4. **Context exit** (`__exit__`), when no exception:
   - `Storage.save(self.network)` persists.
   - If a subscriber snapshot was captured, diff it against the post-state, compute `pubsub.compute_changes(...)`, and `pubsub.publish(...)`.
   - `Storage.close()`, then update `_network_cache[db_path]` with the new mtime.

## Side Effects

- **Disk I/O**: SQLite DB is saved; write lock is held during the context body.
- **Cache write**: `_network_cache[db_path]` is repopulated with the post-supersede `Network` and current mtime.
- **Pub/sub**: If `pubsub.has_subscribers(db_path)`, an event dict describing truth-value transitions is published.
- **Fresh load**: even if a cached `Network` for the same `db_path` exists, it is *not* used — `write=True` forces a re-load (`_with_network._Ctx.__init__`).
- **PG path**: opens a `PgApi(pg_conninfo, domain_id)` connection via `_pg_dispatch` and delegates — a distinct connection/lifecycle from SQLite.

## Error Handling

- **`KeyError`** — from `Network.supersede` if `old_id` or `new_id` isn't found. This propagates untouched (no wrapping).
- **`sqlite3.DatabaseError` / `OperationalError`** — from `Storage.load`/`save` if the file is corrupted or locked.
- **`FileNotFoundError`** — noted: `_with_network` uses `os.path.getmtime` with `except OSError` at load, so a missing DB yields an empty `Storage` state (`Storage(db_path)` will typically create/lookup the schema); downstream the supersede will then raise `KeyError`.
- **Silent swallow when exception inside the body:** `_with_network.__exit__` skips `save()` if `exc_type is not None`, so a failure mid-supersede leaves the on-disk DB untouched. The exception itself propagates.
- **`transitive` is silently ignored on PG** — not an exception, but a behavioral difference callers can be surprised by.

## Usage Patterns

```python
# 1. New node must already exist
add_node("fact-v2", "Updated text", db_path="reasons.db")
result = supersede("fact-v1", "fact-v2")
# result -> {"old_id": "fact-v1", "new_id": "fact-v2", "changed": [...]}

# 2. With transitive — defeats every ancestor in the chain
supersede("fact-v1", "fact-v3", transitive=True)

# 3. PostgreSQL backend
supersede("fact-v1", "fact-v2", pg_conninfo="postgresql://...", domain_id="proj")
```

**Caller obligations:**
- Add `new_id` first, or use `supersede_with_text` (which internally calls this function).
- Treat `changed` as conservative — dependents not in the list did not flip.
- Do **not** expect `transitive=True` to have an effect when `pg_conninfo` is supplied.

## Dependencies

- `reasons.storage.Storage` — SQLite persistence (`load`/`save`/`close`).
- `reasons.network.Network` — holds `net.supersede(old_id, new_id, transitive=...)` (the real implementation).
- `reasons.api._with_network` — the load/operate/save context manager (write-mode; skips cache).
- `reasons.api._pg_dispatch` — lazily imports `reasons.pg.PgApi` for PG backends.
- `reasons.pubsub` — optional change-event publication.
- `reasons.api.DEFAULT_DB` — default file name `"reasons.db"`.

---

## Topics to Explore

- [function] `reasons/network.py:Network.supersede` — The actual supersede implementation: how outlist entries cause `old_id` to go `OUT`, and how `transitive=True` walks the supersession chain.
- [function] `reasons/api.py:supersede_with_text` — The convenience wrapper that creates the new node then calls this function; shows the canonical "add-then-supersede" usage pattern.
- [function] `reasons/api.py:mark_superseded` — The non-reversible alternative that uses `retract` + `metadata["superseded_by"]`; comparing it clarifies the two revision strategies.
- [general] `outlist-mechanism` — How outlists enable reversible defeat (vs. retraction) and why this matters for the belief-revision workflow.
- [file] `reasons/pg.py` — The PostgreSQL backend side of `_pg_dispatch`; check whether `PgApi.supersede` accepts a `transitive` kwarg (the wrapper drops it).

## Beliefs

- `supersede-uses-outlist-not-retraction` — `supersede` marks `old_id` OUT via the outlist mechanism (reversible), unlike `mark_superseded` which calls `net.retract` and records `superseded_by` metadata.
- `supersede-transitive-not-forwarded-to-pg` — The `transitive` flag is silently dropped when `pg_conninfo` is set; only the SQLite path honors it.
- `supersede-requires-existing-new-id` — The function does not add `new_id`; callers must add it first (or use `supersede_with_text`).
- `supersede-write-context-bypasses-cache` — Every call enters `_with_network(db_path, write=True)`, which reloads `Storage.load()` even if a fresh cached `Network` exists.
- `supersede-returns-propagation-cascade` — The returned `changed` list reflects all nodes whose truth value flipped during recompute, not just the `old_id`/`new_id` pair.
