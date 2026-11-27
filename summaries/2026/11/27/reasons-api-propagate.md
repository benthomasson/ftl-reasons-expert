---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: propagate in reasons/api.py

## `propagate` — `reasons/api.py`

### 1. Purpose

`propagate` is a functional wrapper that forces the belief network through a full truth-value recomputation and persists the result. It exists so any caller (CLI, LangGraph tools, scripts, ad-hoc Python) can canonicalize the network's truth values in one call without manually opening `Storage`, loading `Network`, calling `recompute_all`, saving, and closing.

It is the "make the network consistent again" entry point. It's what you call when you suspect the persisted truth values are out of sync with the justification graph — e.g., after an out-of-band DB write, after a partial/buggy propagation, after a manual `Storage`-level import, or as a safety net following many mutations.

It is *not* a mutation primitive — callers use `add_node`, `retract_node`, `assert_node`, etc. Those functions each open their own write context. `propagate` is for bulk reconciliation.

### 2. Contract

**Preconditions**
- If `pg_conninfo` is falsy: `db_path` must point to a SQLite file that `Storage(db_path).load()` can read. The file's presence isn't guarded here — whether a missing file raises, creates an empty DB, or returns an empty network is determined by `Storage`, not by `propagate`.
- If `pg_conninfo` is truthy: `db_path` is ignored entirely; the remote backend is authoritative.
- The network's node structures must be internally consistent enough for `Network.recompute_all()` to walk them (dangling antecedents/outlists are tolerated in the rest of the codebase — see tests like `test_dangling_dependents.py`).

**Postconditions**
- Local mode: the `Network` loaded from disk has had `recompute_all()` run; if the context exits cleanly, the mutated network is written back via `Storage.save`. The cached `Network` for that path is replaced with the freshly saved one (keyed on file mtime).
- Pg mode: the remote `PgApi.propagate()` has been executed; local state is untouched.
- Return value always is a JSON-serializable dict with the shape (`{"changed": ...}`) from whichever backend ran.

**Invariants**
- No partial writes: `Storage.save` is only invoked when the with-block exits without exception (see `_with_network.__exit__`).
- Fresh-read: because `write=True`, `_with_network` bypasses `_network_cache` on entry, so any in-memory drift is discarded and disk state is authoritative for this call.
- The `changed` list reflects nodes whose `truth_value` differed between the pre-recompute state and the post-recompute state (as produced by `recompute_all()`), regardless of whether any event was consumed downstream.

### 3. Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `db_path` | `str` | Path to the SQLite RMS database. Default `DEFAULT_DB = "reasons.db"`. | Ignored entirely when `pg_conninfo` is set. If the file is missing, behavior depends on `Storage` — do not assume a silent no-op. |
| `pg_conninfo` | any (typically `str` / psycopg connection string) | If truthy, dispatches to `PgApi`. | Any falsy value (None, `""`, `0`) means SQLite mode. The code only tests truthiness — an empty string would be treated as SQLite. |
| `domain_id` | any | Passed through to `PgApi(pg_conninfo, domain_id)` in pg mode. | Ignored in SQLite mode. **There is no warning** if the caller passes `domain_id` without `pg_conninfo`; it silently has no effect. |

### 4. Return Value

Always a `dict`.

- **SQLite mode:** `{"changed": <list>}` where `<list>` is exactly what `Network.recompute_all()` returns — by pattern in this file (`mark_duplicate`, `defeat_justification`, `retract_node`, `assert_node` all iterate it as node IDs) a `list[str]`.
- **Pg mode:** the raw return value of `PgApi.propagate()` — not re-shaped by `propagate` at all. Callers relying on `result["changed"]` are effectively relying on the pg backend to return the same shape.

Callers must handle an empty `changed` list — it means the network was already consistent.

### 5. Algorithm

1. Coarse backend check: if `pg_conninfo` is truthy, delegate and return immediately:
   ```python
   return _pg_dispatch(pg_conninfo, domain_id, "propagate")
   ```
   `_pg_dispatch` opens `PgApi(pg_conninfo, domain_id)` as a context manager and calls `getattr(pg, "propagate")()`. No kwargs forwarded.

2. Otherwise open the SQLite write context:
   ```python
   with _with_network(db_path, write=True) as net:
       changed = net.recompute_all()
   ```
   `_with_network(write=True)` does the following:
   - Reads `os.path.getmtime(db_path)` (an `OSError` → `mtime = None`).
   - Checks `_network_cache[db_path]`. If cached and `mtime` matches **and** `write` were False, it would reuse the cache. Because `write=True`, it *always* instantiates a fresh `Storage(db_path)` and calls `.load()`.
   - `__enter__`: if there are pubsub subscribers (`pubsub.has_subscribers(db_path)`), snapshots `{node_id: truth_value}` into `self._before`.
   - Executes `net.recompute_all()` — a full propagation pass, returning the list of node IDs whose truth values flipped.
   - `__exit__` (clean): `store.save(network)` writes to disk; if `_before` was captured, computes diff via `pubsub.compute_changes(before, after, db_path)` and calls `pubsub.publish(events, db_path)`. Then closes the store. Then updates `_network_cache[db_path] = (new_mtime, network)`.
   - `__exit__` (exception): skips save, skips pubsub, still closes the store, **does not** update the cache (the guard is `not (write and exc_type is not None)`), returns `False` so the exception propagates.

3. Return `{"changed": changed}`.

### 6. Side Effects

- **Disk I/O:** full SQLite load and (on clean exit) full save. This replaces the persisted network wholesale, not per-node.
- **Cache mutation:** `_network_cache[db_path]` is repopulated with the freshly saved `Network`. Any prior cached instance is discarded even for read-only callers.
- **Pubsub:** if subscribers exist for `db_path`, a diff of truth-value changes is published as events. This is gated by `has_subscribers`, so cost is only paid when needed.
- **Pg mode:** remote state is mutated; the local `_network_cache` is *not* touched (nothing local is loaded or saved).
- **No global locks:** SQLite locking is handled inside `Storage`; `_with_network` itself does not serialize concurrent callers.

### 7. Error Handling

`propagate` has **no** try/except. It relies entirely on:

- `_with_network.__exit__` to guarantee `store.close()` even on failure (`if self.store: self.store.close()` runs unconditionally).
- `__exit__` returning `False`, so any exception — from `Storage.load`, `recompute_all`, or `Storage.save` — propagates to the caller.
- `_pg_dispatch` to raise whatever `PgApi` raises (connection errors, remote failures).

Concrete failure surfaces a caller should expect:
- `sqlite3.OperationalError` (missing DB, locked DB, schema drift) from `Storage.load` / `save`.
- Whatever `Network.recompute_all()` raises if the graph has a structural invariant violation the propagation code asserts on.
- On pg mode, DB-driver exceptions, auth errors, etc. from `PgApi`.

Because save happens in `__exit__` after `recompute_all()` returned, a mid-save exception (e.g., disk full) leaves the DB in whatever state `Storage.save` left it — this function does not attempt rollback.

### 8. Usage Patterns

Typical:
```python
result = propagate()                       # default reasons.db
changed = result["changed"]
if changed:
    log.info("propagation flipped %d nodes", len(changed))
```
```python
# After an out-of-band import or partial run:
propagate(db_path="/data/rms.db")
```
```python
# Remote-backend call — db_path irrelevant:
propagate(pg_conninfo="postgresql://...", domain_id="tenant-a")
```

Caller obligations:
- Treat `changed` as a signal only; do not assume any particular ordering.
- Do not pass `domain_id` without `pg_conninfo` and expect it to matter.
- Do not expect "no change" to mean "no work" — a full recompute and full SQLite save happen regardless of whether the network was already in fixpoint.
- Serialize calls if the workload is write-heavy; each `propagate` is a full network rewrite.

Common bug to avoid: calling `propagate()` immediately after a mutation that already opened `_with_network(write=True)` (e.g., `retract_node`). Those mutations propagate what they need; an extra `propagate` is only a hedge, not a requirement.

### 9. Dependencies

- `_pg_dispatch` → imports `reasons.pg.PgApi` lazily; wraps `PgApi` in a context manager.
- `_with_network` → `reasons.storage.Storage`, `reasons.pubsub`, module-global `_network_cache`, `os.path.getmtime`.
- `Network.recompute_all()` → `reasons.network.Network`, i.e., the TMS fixpoint algorithm.
- `pubsub.has_subscribers`, `pubsub.compute_changes`, `pubsub.publish` → event fan-out on write.
- `DEFAULT_DB = "reasons.db"` for the default path.
- No direct import of `pg` or `storage` lifecycles at module load — the pg import is deferred to call time.

### Assumptions not enforced by types

- `recompute_all()` is assumed to return an iterable of node ID strings. The type annotation is absent — a future refactor returning `dict` would still pass the return and only fail later in consumers.
- `db_path` being a valid file is assumed, not checked. `propagate(db_path="nonexistent.db")` will pass through to `Storage`'s behavior (create vs. raise depends on `Storage`).
- `pg_conninfo` is treated as a truthy/falsy discriminator only. Any object with `__bool__` overriding → non-obvious. An empty string `""` is *not* pg mode.
- Both backends are assumed to return `{"changed": [...]}`. `propagate` does not normalize the pg response shape — the contract with pg is delegated.

## Topics to Explore

- [function] `reasons/network.py:recompute_all` — The actual propagation algorithm, including outlist/defeater semantics and the "changed" set definition.
- [function] `reasons/api.py:_with_network` — Cache-by-mtime, write-mode bypass, and the pubsub subscriber diff.
- [file] `reasons/pubsub.py` — How truth-value diffs become events and how subscribers are keyed by `db_path`.
- [function] `reasons/pg.py:PgApi.propagate` — What the remote backend's `propagate` returns and how `domain_id` scopes it.
- [file] `reasons/storage.py` — Load/save granularity, schema, and whether a missing file is created or raises.

## Beliefs

- `propagate-dispatches-to-pg-via-conninfo` — When `pg_conninfo` is truthy, `propagate` returns `_pg_dispatch(...)` and never opens the SQLite `_with_network` context.
- `propagate-forces-write-context` — `propagate` calls `_with_network(db_path, write=True)`, which bypasses the `_network_cache` on load and triggers `Storage.save` + cache refresh on clean exit.
- `propagate-produces-changed-list` — The `"changed"` key is the verbatim return of `Network.recompute_all()`, so its element type (node IDs) is a convention, not a type-enforced contract.
- `propagate-enables-pubsub-notification` — Because it opens a `write=True` context, if `pubsub.has_subscribers(db_path)` is true, truth-value diffs are published on successful exit.
- `propagate-skips-save-on-exception` — `_with_network.__exit__` returns `False` and only saves when `exc_type is None`, so exceptions from `recompute_all` propagate without a partial write and without updating `_network_cache`.
- `propagate-domain-id-requires-pg_conninfo` — `domain_id` is silently ignored in SQLite mode; it is only forwarded to `PgApi` when `pg_conninfo` is truthy.
