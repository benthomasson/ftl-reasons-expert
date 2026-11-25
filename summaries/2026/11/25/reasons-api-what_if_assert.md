---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: what_if_assert in reasons/api.py

# `what_if_assert` — Simulation of asserting an OUT node

## Purpose

`what_if_assert` is a **read-only what-if simulator** for the Reason Maintenance System's TMS assertion operation. It answers the question:

> *"If I were to assert this currently-OUT node back to IN, what else would change — and in which direction?"*

It exists so callers (CLI, MCP server, agent loops like `propose_retraction`'s sibling workflows) can preview the effect of a truth-value mutation **before** committing it. It is the counterpart to `what_if_retract`: retraction pushes things OUT; assertion pushes things back IN — but assertion can also push things OUT when it activates an **outlist gate** (an OUT node referenced in another node's outlist must stay OUT for that node's justification to hold; once the blocker becomes IN, the gated belief loses its justification and goes OUT).

It never touches the database and never publishes pubsub events.

## Contract

**Preconditions**
- `db_path` (SQLite path) or a live `pg_conninfo` must be supplied.
- `node_id` must exist in the network. A missing ID raises `KeyError` and is *not* treated as "already retracted."
- The `Network` loaded from disk must be in a consistent, recomputed state (the TMS's `assert_node` assumes `truth_value`s reflect the last propagation).

**Postconditions**
- The on-disk database is byte-identical (barring concurrent writers); no `Storage.save` is called.
- The shared `_network_cache[db_path]` entry is *not* mutated (guaranteed by `copy.deepcopy`).
- Returns a dict describing the predicted deltas. Two states are possible:
  - `already_in == True`: the node is already IN; `assert_node` would be a no-op.
  - `already_in == False`: the simulation was actually executed.

**Invariants**
- `node_id` is deliberately **excluded** from both `retracted` and `restored`. The caller already knows about that node.
- `total_affected == len(retracted) + len(restored)`.
- Lists are sorted by `(depth, id)`.

## Parameters

| Parameter | Type | Meaning |
|---|---|---|
| `node_id` | `str` | The node to simulate asserting. Must be currently `OUT` to trigger a real simulation. |
| `db_path` | `str` | SQLite database path. Defaults to `DEFAULT_DB = "reasons.db"`. Ignored when `pg_conninfo` is set. |
| `pg_conninfo` | `object`/`None` | When truthy, dispatches to the Postgres backend via `_pg_dispatch` and returns whatever `PgApi.what_if_assert` returns — **without ever touching `db_path`**. |
| `domain_id` | `str`/`None` | Passed to the Postgres dispatcher; ignored for SQLite. |

Edge cases:
- Asserting a node that's already IN is a legitimate call that short-circuits.
- A missing `node_id` raises `KeyError` rather than returning an empty result — callers must pre-filter.
- Passing both `pg_conninfo` and a valid `db_path` silently prefers Postgres.

## Return Value

```python
{
    "node_id": str,
    "already_in": bool,
    "retracted": [{"id": str, "text": str, "depth": int, "dependents": int}, ...],
    "restored":  [{"id": str, "text": str, "depth": int, "dependents": int}, ...],
    "total_affected": int,
}
```

- **`already_in == True`** path returns `{node_id, already_in: True, retracted: [], restored: [], total_affected: 0}`. Note the absence of `total_affected` in the docstring is misleading — it *is* present.
- **`retracted`**: nodes that were `IN` before and `OUT` after — these are almost always outlist-gated beliefs (relatives of the `asserted` node's dependents via outlist edges) that lost support.
- **`restored`**: nodes that were `OUT` before and `IN` after — the "restoration cascade" propagated through SL/CP justifications.
- **`depth`**: BFS shortest-path length in the `dependents` reverse index from `node_id` to the affected node (`_cascade_depth`). `0` when the node is unreachable via `dependents` (possible in principle if changes arrived via outlist-only paths outside the BFS frontier).
- **`dependents`**: raw count `len(n.dependents)` at the *post-assertion* moment — the reverse edge set, which is topology and does not change during propagation.

Callers must handle: `KeyError` (node missing), any `RuntimeError`/exceptions propagated from `assert_node`.

## Algorithm

1. **Backend branch**
   ```python
   if pg_conninfo:
       return _pg_dispatch(pg_conninfo, domain_id, "what_if_assert", node_id=node_id)
   ```
   Straight to `PgApi.what_if_assert`. All further logic below is SQLite-only.

2. **Open the network read-only**
   ```python
   with _with_network(db_path, write=False) as net:
   ```
   - The context manager checks `_network_cache[db_path]` against `os.path.getmtime`. A hit reuses the cached `Network`; a miss loads a fresh one via `Storage(db_path).load()`.
   - `write=False` means `__exit__` will **not** call `store.save`, and `__enter__` will not snapshot `_before` for pubsub.

3. **Existence check**
   ```python
   if node_id not in net.nodes:
       raise KeyError(f"Node '{node_id}' not found")
   ```

4. **Early-out for already-IN**
   ```python
   if node.truth_value == "IN":
       return {..., "already_in": True, "retracted": [], "restored": [], "total_affected": 0}
   ```
   No copy, no simulation, no `changed` iteration.

5. **Deep copy the network**
   ```python
   net = copy.deepcopy(net)
   ```
   This is *the* safety-critical step. The `Network` from `_with_network` may be a cached object shared across many read-only calls in the process (see `tests/test_network_cache.py`). Without this, `assert_node` would mutate the cached `Node.truth_value` fields and subsequent reads would return wrong status until the file's mtime changed.

6. **Snapshot the pre-state**
   ```python
   before = {nid: n.truth_value for nid, n in net.nodes.items()}
   ```
   Materializes `{node_id: "IN"|"OUT"}` for the whole network. Needed because `assert_node` returns the list of *changed* IDs but not the direction.

7. **Simulate the assertion**
   ```python
   changed = net.assert_node(node_id)
   ```
   Runs the TMS propagation in memory against the copy. `changed` contains the IDs of every node whose `truth_value` flipped — including `node_id` itself (OUT → IN).

8. **Classify each change**
   ```python
   for nid in changed:
       if nid == node_id:      # skip the asserted node itself
           continue
       info = {
           "id": nid,
           "text": net.nodes[nid].text,
           "depth": _cascade_depth(net, nid, node_id),
           "dependents": len(net.nodes[nid].dependents),
       }
       if before[nid] == "IN" and net.nodes[nid].truth_value == "OUT":
           retracted.append(info)
       elif before[nid] == "OUT" and net.nodes[nid].truth_value == "IN":
           restored.append(info)
   ```
   - `_cascade_depth` is a BFS over `dependents` inside the *post-assertion* network. Since `assert_node` only flips truth values (not edge structure), the BFS traverses the same topology as the pre-assertion network — depth reflects the causal hop count.
   - **Any `changed` entry not matching either branch is silently dropped.** This is an invariant: `assert_node` should only return nodes whose truth values actually flipped, so the `if/elif` is exhaustive in practice. If `assert_node` ever returned a node neither IN→OUT nor OUT→IN, that node is quietly excluded from both lists and from `total_affected`.

9. **Sort for deterministic output**
   ```python
   retracted.sort(key=lambda c: (c["depth"], c["id"]))
   restored.sort(key=lambda c: (c["depth"], c["id"]))
   ```
   Shallowest first, tie-broken by ID. Depth-first ordering matches the mental model "direct consequences first."

10. **Return**
    ```python
    return {
        "node_id": node_id,
        "already_in": False,
        "retracted": retracted,
        "restored": restored,
        "total_affected": len(retracted) + len(restored),
    }
    ```

## Side Effects

- **No database writes.** `write=False` skips `store.save()` in `__exit__`.
- **No pubsub dispatch.** `pubsub.has_subscribers` / `pubsub.publish` are only invoked when `write=True`.
- **`_network_cache` is updated on exit.** Even read-only calls stash `(mtime, self.network)` in the cache. But the *cached* object is the original (not the deep-copied simulation target), so cache correctness is preserved.
- **Process memory:** a full `Network` deep copy is materialized. For large networks this is the dominant cost — one copy per call.
- **`_cascade_depth` BFS:** O(V+E) per affected node in the worst case. Overall worst-case cost of step 8 is O(V·(V+E)).
- **No concurrency safety:** the mtime-based cache invalidation is not a lock. Two processes writing while this reads can produce a stale simulation.

## Error Handling

| Condition | Behavior |
|---|---|
| `node_id` not in network | `KeyError(f"Node '{node_id}' not found")` — propagates up. |
| `db_path` points to a missing/corrupt DB | `Storage(db_path).load()` raises from the storage layer. |
| `pg_conninfo` set but unreachable | `_pg_dispatch` → `PgApi(...)` raises the DB driver's exception. |
| `assert_node` raises | Propagates uncaught; no rollback needed since `write=False` guarantees no state was written. |
| Node already IN | Return dict with `already_in: True`; no simulation. |

Errors are **not** swallowed; there is no try/except inside the function. Callers like `propose_retraction` wrap calls in try/except for their own purposes.

The function does not validate:
- That `assert_node`'s returned `changed` set contains only actual truth-value flips (implicit invariant).
- That `_cascade_depth` finds a path; unreachable returns `0`.

## Usage Patterns

```python
# Preview before committing
preview = what_if_assert("belief-42")
if preview["already_in"]:
    ...
elif preview["total_affected"] > 20:
    print("Large cascade — reconsider")
else:
    assert_node("belief-42")   # commit

# Inside proposal tooling: preview a retraction impact
impact = what_if_retract(target_id)   # symmetric counterpart
```

Caller obligations:
- Check `already_in` before interpreting the empty lists — an empty `retracted`/`restored` is *not* the same as "no effect."
- Do not assume `retracted + restored` preserves original traversal order — it's re-sorted.
- Do not rely on `dependents` counting only currently-IN dependents; it counts all edges.

## Dependencies

| Symbol | Location | Role |
|---|---|---|
| `_with_network` | `reasons/api.py` | Opens/caches a `Network` in read or write mode |
| `_pg_dispatch` | `reasons/api.py` | Postgres backend routing via `PgApi` |
| `_cascade_depth` | `reasons/api.py` (defined *below* `what_if_assert` — module-level so callable at runtime) | BFS shortest path over `dependents` |
| `Network.assert_node` | `reasons/network.py` | The TMS propagation routine |
| `copy.deepcopy` | stdlib | Isolates the simulation from the cached `Network` |
| `Storage` | `reasons/storage.py` (used indirectly via `_with_network`) | Loads the `Network` from SQLite |
| `pubsub` | `reasons/pubsub.py` | Not invoked in this path (only on `write=True`) |

## Assumptions Not Enforced by Types

1. **`assert_node(node_id)` returns every node whose `truth_value` changed**, no more and no less. If it returned unchanged nodes, they'd be silently dropped by the `if/elif` — an observable undercount of `total_affected`.
2. **`before` contains `node_id` and every entry in `changed`.** Guaranteed by iterating `net.nodes.items()` after the copy, but nothing in the type system prevents `assert_node` from returning an ID not in `net.nodes`.
3. **The deep copy is shallow-copy-safe for all fields that `assert_node` mutates.** If `assert_node` mutated any object shared with the cache (e.g., a justification object), the copy would be insufficient. The current `Node.truth_value`/`supporting_justification` tests pass, but the invariant is implicit.
4. **The `dependents` reverse index is maintained by `assert_node` (and prior writes).** If it drifted, `_cascade_depth` would silently return `0`.
5. **`write=False` on `_with_network` means "no subscriber snapshot" pre-`__enter__` and "no save" post-`__exit__`.** That's an implementation detail other code could change.

---

## Topics to Explore

- [function] `reasons/network.py:Network.assert_node` — The TMS propagation that this simulator wraps; its outlist-gating semantics explain why `retracted` can be non-empty when asserting.
- [function] `reasons/api.py:what_if_retract` — Mirrors this function; reading both makes the IN↔OUT symmetry of the TMS explicit.
- [function] `reasons/api.py:_cascade_depth` — The BFS that computes the `depth` field; important because it traces `dependents` (which unify antecedent and outlist edges).
- [function] `reasons/api.py:_with_network` — The caching/transaction context; its `write=False` behavior is why this function is safe to call in loops without touching DB state.
- [file] `tests/test_network_cache.py` — Specifically exercises the deep-copy invariant this function relies on; would catch a regression if `copy.deepcopy` were dropped.

## Beliefs

- `what-if-assert-never-writes-database` — The function opens `_with_network(write=False)` and never calls `store.save` or `pubsub.publish` on the SQLite path.
- `what-if-assert-requires-deep-copy` — The `copy.deepcopy(net)` call is essential to avoid corrupting the process-wide `_network_cache` entry, since read-only loads may return a shared cached `Network`.
- `asserted-node-is-excluded-from-results` — The `if nid == node_id: continue` guard removes the asserted node from both `retracted` and `restored`, so callers must add `1` if counting total nodes affected.
- `assert-can-retract-outlist-gated-beliefs` — Asserting an OUT node can push other nodes OUT via outlist gating, so `what_if_assert.retracted` may be non-empty despite the "assertion" framing.
- `cascade-depth-uses-dependents-bfs` — The `depth` field is the shortest-path hop count in the `dependents` reverse index, or `0` if the changed node is unreachable from `node_id`.
- `pg-conninfo-short-circuits-sqlite-branch` — If `pg_conninfo` is truthy, the function returns `_pg_dispatch(...)` before any SQLite code runs, so `db_path` is ignored.
