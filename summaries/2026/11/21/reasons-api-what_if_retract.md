---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: what_if_retract in reasons/api.py

# `what_if_retract` — Read-Only Retraction Simulator

## 1. Purpose

`what_if_retract` is a **what-if / dry-run** counterpart to `retract_node`. It answers the question: *"If I retracted `node_id` right now, what would happen to the rest of the belief network?"* — without actually changing anything in the database or the in-memory cache.

It exists so callers (CLI, LangGraph tools, proposal review flows such as `propose_retraction`, `propose_update`, and `repair`) can preview the blast radius of a retraction and decide whether to commit it.

## 2. Contract

**Preconditions**
- The database at `db_path` (or the PG database/domain identified by `pg_conninfo`/`domain_id`) must exist and be readable.
- `node_id` **must exist** in the network; otherwise `KeyError` is raised.

**Postconditions (SQLite path)**
- Database file is **not modified** — the context manager is opened with `write=False` and the returned `Network` object is deep-copied before any mutation.
- The in-memory `_network_cache[db_path]` entry is **unchanged in content** — the copy protects it, and `__exit__` re-stores the *original* `self.network` (not the mutated copy).
- The response dict is JSON-serializable (strings, ints, bools, lists of dicts).

**Invariants**
- The target `node_id` is **never included** in `retracted` or `restored` lists (`if nid == node_id: continue`). It is only echoed back as the top-level `node_id`.
- Every `id` in `retracted` had `truth_value == "IN"` before and is `"OUT"` after the simulation; every `id` in `restored` was `"OUT"` before and is `"IN"` after.
- `total_affected == len(retracted) + len(restored)`.

**Assumptions not enforced by types**
- `net.retract()` only changes `truth_value` and never removes nodes or rewrites `dependents` — otherwise the `n.dependents` and `_cascade_depth` reads *after* retraction would report post-retraction state instead of the pre-retraction graph.
- The PG backend's `what_if_retract` returns a dict with the same key shape; the function does not normalize or validate it.

## 3. Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `node_id` | `str` | ID of the belief to simulate retracting | Missing → `KeyError("Node '...' not found")`. `""` also raises `KeyError` |
| `db_path` | `str` (default `DEFAULT_DB = "reasons.db"`) | SQLite DB path | Ignored when `pg_conninfo` is truthy |
| `pg_conninfo` | `None` or PG connection string | When truthy, dispatches to `PgApi.what_if_retract` | Bypasses all SQLite/cache logic |
| `domain_id` | `None` or anything | PG domain identifier | Only used when `pg_conninfo` is set |

## 4. Return Value

Two distinct shapes are returned:

**If node is already `OUT` (short-circuit):**
```python
{
    "node_id": node_id,
    "already_out": True,
    "retracted": [],
    "restored": [],
    "total_affected": 0,
}
```

**If node is `IN` (full simulation):**
```python
{
    "node_id": node_id,
    "already_out": False,
    "retracted": [{"id": ..., "text": ..., "depth": int, "dependents": int}, ...],
    "restored":  [{"id": ..., "text": ..., "depth": int, "dependents": int}, ...],
    "total_affected": int,
}
```

Both lists are sorted by `(depth, id)` ascending. `depth` is the shortest path length through the `dependents` graph from the retracted node. `total_affected` **excludes** the target node itself.

Callers must handle: (a) `KeyError` for unknown nodes, (b) an empty result struct via the `already_out` branch, (c) possible `AttributeError` if `PgApi` lacks the method (unlikely, but no defensive checking).

## 5. Algorithm

1. **PG dispatch** — if `pg_conninfo` is set, hand off to `_pg_dispatch(...)` which opens a `PgApi` and calls `what_if_retract(node_id=node_id)` on it. Return its result verbatim.

2. **Load read-only** — `with _with_network(db_path, write=False)`:
   - Calls `os.path.getmtime(db_path)`.
   - If the cache has an entry for `db_path` whose stored mtime matches, reuse the cached `Network` (no disk load).
   - Otherwise, open `Storage(db_path)` and `store.load()`.

3. **Existential check** — raise `KeyError` if `node_id not in net.nodes`.

4. **Short-circuit** — if the node is already `OUT`, return the reduced dict immediately (no cascade simulation needed).

5. **Deep copy** — `net = copy.deepcopy(net)`. This rebinds the *local* variable; the `_Ctx.__exit__` will re-cache `self.network` (the original object), not this copy. Without the copy, `net.retract()` below would corrupt the shared cache because `_with_network` caches by object identity (`_network_cache[db_path] = (mtime, self.network)`).

6. **Snapshot pre-state** — `before = {nid: n.truth_value for nid, n in net.nodes.items()}` on the copy (identical to original at this instant).

7. **Simulate** — `changed = net.retract(node_id)`. This is the only mutation, and it happens *only on the deep copy*.

8. **Classify** — iterate `changed`. Skip the target itself. For each remaining `nid`:
   - Build `info = {id, text, depth, dependents}`. `dependents` is `len(n.dependents)` on the mutated copy; because `retract` doesn't touch the reverse index, this equals the pre-retraction count.
   - If `before[nid] == "IN"` and `n.truth_value == "OUT"` → append to `retracted`.
   - elif `before[nid] == "OUT"` and `n.truth_value == "IN"` → append to `restored` (this is the outlist "restoration" case — a beleif was gated by the now-retracted node's OUT status and can now flip back IN).

9. **Decorate depth** — `_cascade_depth(net, nid, node_id)` performs a BFS from `node_id` across `dependents` edges and returns the shortest hop count to `nid`. Returns `0` if `nid` is unreachable from `node_id` via dependents (shouldn't happen for cascade members, but is silently tolerated).

10. **Sort and return** — both lists sorted by `(depth, id)`.

## 6. Side Effects

- **Filesystem reads**: `os.path.getmtime(db_path)`; possibly a full `Storage.load()` on cache miss.
- **Memory writes**: `_network_cache[db_path]` may be *repopulated* with `(mtime, original_network)` by `_with_network.__exit__`. Content is stable because the retract happened on a deepcopy.
- **CPU**: a full `copy.deepcopy` of the network, plus one cascade traversal. On large networks this dominates.
- **No DB writes**. No pubsub events (which are only emitted when `write=True`).

## 7. Error Handling

| Raised | Condition |
|---|---|
| `KeyError` | `node_id` not in `net.nodes` |
| Propagated from `Storage` | Corrupt/missing DB, permissions |
| Propagated from `PgApi` | Any PG-side failure (no try/except here) |
| Propagated from `_cascade_depth` | If `net.nodes[current_id]` raises `KeyError` — but this is pre-validated by the BFS start node and visited set |

**Swallowed**: `OSError` from `os.path.getmtime` inside `_Ctx.__init__` and `__exit__` is caught and silently ignored. Also, restoration classification failures for nodes present in `changed` but missing from `before` would raise `KeyError` — but `before` is a snapshot of *all* nodes, so this can't occur.

No non-`Exception` return value signals errors. Callers must use try/except for missing nodes.

## 8. Usage Patterns

- **Ad-hoc preview**: `reasons what-if-retract some-belief-id` in CLI produces the JSON dict for inspection.
- **Proposal gating**: `propose_retraction` calls `what_if_retract` to compute `impact_json` stored alongside the pending proposal, giving reviewers a blast-radius estimate before `accept_proposal` commits. See `propose_update`, whose `cascades` dict maps proposal IDs to `what_if_retract` outputs.
- **Repair triage**: `repair` / `repair_smuggled` flows can call it to decide whether a retraction is safe (small cascade) or too broad (needs a supersede instead).

**Caller obligations**: none beyond existing on disk. Because it never persists, callers may call it in tight loops — the only cost is repeated `deepcopy` + BFS. To avoid the deepcopy overhead across many calls, prefer a single `what_if_retract` per candidate node, or operate on `export_network()` results directly.

## 9. Dependencies

- `copy.deepcopy` — protects the cached `Network`.
- `reasons.api._with_network` — read-only cached context manager.
- `reasons.api._pg_dispatch` → `reasons.pg.PgApi` — PG backend dispatch.
- `reasons.api._cascade_depth` — dependents-graph BFS.
- `reasons.network.Network.retract` — performs the TMS cascade on the copy.
- `reasons.storage.Storage` — only on cache miss.
- `reasons.api._network_cache` — the module-level cache dict that must remain uncorrupted.

## Topics to Explore

- [function] `reasons/api.py:_cascade_depth` — The BFS that assigns each affected node its "distance" from the retracted node; understanding it clarifies what `depth` in the result means and why unreachable nodes surface as `0`.
- [function] `reasons/network.py:Network.retract` — The TMS cascade engine. `what_if_retract` is a thin wrapper around it; the interesting semantics (outlist gating, restoration chains, monotonicity) all live here.
- [function] `reasons/api.py:_with_network` — The cache contract. The reason `what_if_retract` must deep-copy is that `__exit__` unconditionally re-stores `self.network` — a shared object.
- [function] `reasons/api.py:what_if_assert` — Symmetric dry-run; comparing the two exposes how the "restored via outlist" and "retracted via outlist" directions mirror each other.
- [general] `proposal-impact-flow` — Trace how `what_if_retract`'s result is persisted in `proposals.impact_json` by `propose_retraction` and later consulted during `accept_proposal` drift checks.

## Beliefs

- `what-if-retract-preserves-cache-integrity` — `what_if_retract` deep-copies the loaded `Network` before calling `retract`, which is necessary because `_with_network.__exit__` writes `self.network` (the original object) back into `_network_cache` regardless of mutations.
- `what-if-retract-excludes-target-node` — The node being retracted is filtered out of both `retracted` and `restored` (`if nid == node_id: continue`), so `total_affected` counts only cascaded changes, never the target itself.
- `what-if-retract-short-circuits-on-out-node` — Retracting an already-OUT node returns immediately with `already_out: True` and empty lists, and does not invoke `net.retract` at all.
- `what-if-retract-distinguishes-restored-from-retracted` — A node in `changed` is classified as `restored` when it went `OUT → IN` (outlist-gate removal), which is distinct from the more common `IN → OUT` cascade.
- `what-if-retract-is-a-thin-wrapper-around-network-retract` — Nearly all TMS semantics (cascade, outlist gating, restoration) live in `Network.retract`; `what_if_retract` only adds preconditions, copy-isolation, categorization, depth annotation, and sorting.
