---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: retract_node in reasons/api.py

# `retract_node`

## Purpose

`retract_node` is the public, JSON-friendly API entry point for **retracting** a belief in the Reason Maintenance System. Retraction marks a node `OUT` and cascades the truth-value change through the justification graph — any dependents whose justifications are no longer satisfied also flip `OUT`. When a blocker (outlist entry) is retracted, previously gated beliefs can flip back `IN`. The function reports the full set of nodes whose truth value changed and annotates the `IN → OUT` transitions with "restoration hints" that help a caller understand how a node might be restored later.

It exists so CLI / LangGraph / script callers can retract without managing `Storage` lifecycle or the network cache directly — the function owns the open/save/close cycle via `_with_network`.

## Contract

**Preconditions**
- `node_id` names a node that exists in the loaded network, if the local SQLite path is taken. (The function itself doesn't check — it relies on `net.retract()` to raise `KeyError`.)
- If `pg_conninfo` is provided, the function expects a reachable PostgreSQL instance and schema managed by `reasons.pg.PgApi`.

**Postconditions (local backend)**
- The database at `db_path` has been re-saved with the retraction applied.
- The in-process network cache for `db_path` has been refreshed to the post-retract network (via `_with_network.__exit__`).
- If `pubsub.has_subscribers(db_path)` is true, change events have been published.
- Returned dict has exactly the keys `changed`, `went_out`, `went_in`, `restoration_hints`.

**Postconditions (PG backend)**
- Delegated entirely to `PgApi.retract_node`; the return shape is whatever that backend returns (the docstring's key set is the intended contract, but the local implementation is not run).

**Invariants**
- `went_out` and `went_in` are disjoint subsets of `changed`.
- Every ID in `went_out ∪ went_in` exists in `net.nodes` after the retract (verified indirectly by the indexing in the comprehensions).
- `node_id` itself is never listed in `restoration_hints` (early `continue`).

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `node_id` | `str` | The node to retract. | Not validated; `net.retract` will raise `KeyError` if absent. |
| `reason` | `str` (default `""`) | Free-form text passed to `net.retract` (typically stored on the node's metadata). | Not interpreted by this function — an empty string is valid. |
| `db_path` | `str` (default `DEFAULT_DB = "reasons.db"`) | Path to the SQLite RMS database. | If the file does not exist, `_with_network` (`Storage(db_path)`) will create it; retracting a non-existent node then raises `KeyError`. |
| `pg_conninfo` | anything truthy (default `None`) | Presence of this triggers PostgreSQL dispatch; the value is a libpq-style conninfo string. | Any truthy value warps to `_pg_dispatch` and `db_path` is ignored. |
| `domain_id` | anything (default `None`) | Passed to `PgApi(conninfo, domain_id)` — namespaces the PG backend. | Ignored when `pg_conninfo` is falsy. |

## Return Value

On the local backend the function returns a dict:

```python
{
  "changed": [str, ...],              # all node IDs whose truth_value flipped
  "went_out": [str, ...],             # subset of changed: IN  -> OUT
  "went_in":  [str, ...],             # subset of changed: OUT -> IN   (outlist restoration)
  "restoration_hints": [              # annotation, one entry per went_out node that qualifies
      {
        "node_id": str,
        "all_premises": [str, ...],       # the SL justification's antecedents
        "surviving_premises": [str, ...]  # antecedents still IN after the cascade
      }, ...
  ]
}
```

Caller obligations:
- Must handle `KeyError` / exceptions from `net.retract` (nothing is returned on failure).
- Should tolerate `changed` being empty (retracting an already-`OUT` node may not flip anything).
- If `pg_conninfo` is used, the shape is whatever the PG backend returns — callers should treat the docstring keys as the *intended* contract but test against the actual PG payload.

## Algorithm

1. **Backend selection** — if `pg_conninfo` is truthy, delegate to `_pg_dispatch(pg_conninfo, domain_id, "retract_node", node_id=..., reason=...)` and return. PG dispatch lazily imports `reasons.pg.PgApi` inside a `with` block, opens it, calls `getattr(pg, "retract_node")(...)`, and closes.
2. **Open a write transaction** with `_with_network(db_path, write=True)`. Because `write=True`, the cache is bypassed (fresh load), and on a clean exit the network is `store.save(net)`'d, the cache mtime is updated, and pubsub events are computed/published if subscribers exist. `__enter__` also captures a pre-snapshot for pubsub only if subscribers exist — that is *separate* from the `before` snapshot this function builds.
3. **Snapshot pre-truth-values** into `before: {node_id: truth_value}` for every node.
4. **Cascade retract**: `changed = net.retract(node_id, reason=reason)` — this mutates `net` in place and returns the list of affected node IDs. The `reason` string is stored by the network layer (typically on the node being retracted).
5. **Classify transitions**:
   - `went_out = [nid for nid in changed if before[nid] == "IN" and net.nodes[nid].truth_value == "OUT"]`
   - `went_in  = [nid for nid in changed if before[nid] == "OUT" and net.nodes[nid].truth_value == "IN"]`
   The use of `before.get(nid)` in the actual code (via `before.get`) is defensive — if `net.retract` ever returned an ID outside the snapshot, the comparison yields `False` rather than `KeyError`. (Note: the `net.nodes[nid]` access will still raise if the node can't be found *after* the retract, but a well-behaved `retract` only returns existing nodes.)
6. **Build restoration hints** for each node that went `OUT` other than the retracted node itself:
   - Iterate the node's justifications.
   - Find the first justification with `j.type == "SL"` and `len(j.antecedents) >= 2` (a conjunctive "ALL-of" rule with at least two premises — a node whose only SL had one premise, or whose only justifications are CP/single-antecedent SL, produces no hint).
   - Collect `still_in = [a for a in j.antecedents if a in net.nodes and net.nodes[a].truth_value == "IN"]`.
   - If any antecedent is still `IN`, append a hint `{node_id, all_premises, surviving_premises}`.
   - **`break`** unconditionally after the first qualifying SL justification — even if `still_in` was empty (in that case nothing is appended, and later justifications are not examined).
7. Return the aggregation inside the `with` block; the eventual `__exit__` runs after the return value is constructed (Python evaluates the return expression, then runs `__exit__`, then actually returns).

## Side Effects

- **Database write**: `Storage.save(network)` is invoked on clean exit; the entire network is rewritten.
- **Cache mutation**: `_network_cache[db_path]` is refreshed to `(new_mtime, network)`.
- **Pub/Sub**: if any subscriber is registered (via `reasons.pubsub.has_subscribers(db_path)`), `__enter__` snapshots truth values, and `__exit__` calls `pubsub.compute_changes(before, after, db_path)` followed by `pubsub.publish(events, db_path)`.
- **Node metadata mutation**: `net.retract` typically writes the `reason` onto node metadata / timestamps — the caller should expect that side effect, but it's handled in `reasons.network`, not here.
- **Postgres**: with `pg_conninfo`, a network round trip to the PG backend occurs (no local DB touched).
- **No `print`/`log`**: this function does not log. If `net.retract` raises, the `\__exit__` sees `exc_type` and skips save/publish.

## Error Handling

- **`KeyError`** — raised by `net.retract(node_id, ...)` when `node_id` isn't in `net.nodes`. Propagates uncaught; caller must catch.
- **Generic exceptions from `net.retract`** — any exception aborts the context manager cleanly: `_with_network.__exit__` sees a non-`None` `exc_type`, does **not** save or publish, still calls `store.close()`, and returns `False` (so the exception propagates). The cache is *also not updated* because the `if not (write and exc_type is not None):` guard is false.
- **Postgres errors** — any exception raised inside `PgApi` propagates; `PgApi.__exit__` is responsible for cleanup.
- **No swallowed errors** — the function does not catch or wrap exceptions; callers decide how to handle them.
- **`before.get(nid)` defensive read** — guards against a rogue `changed` list containing IDs not present in the snapshot; the subsequent `net.nodes[nid]` will still raise if that's truly inconsistent.

## Usage Patterns

Typical call from a tool/CLI wrapper:

```python
result = retract_node(
    "belief-42",
    reason="contradicted by source doc",
    db_path="reasons.db",
)
# Inspect result["went_out"] for cascade size, result["went_in"] for restored
# gated beliefs, and result["restoration_hints"] to offer "restore via X" UX.
```

- **Batch retraction**: call once per node; each call pays a full load/save cycle. If you need to retract many nodes, prefer doing the retract loop inside a single `_with_network(write=True)` context yourself rather than calling this function repeatedly.
- **Dry run first**: use `what_if_retract(node_id, ...)` — it performs the same cascade on a deep-copied network without saving. This function has no dry-run flag.
- **PG deployments**: pass `pg_conninfo` (and optionally `domain_id`); `db_path` is ignored. Note PG may not support every cascade nuance the SQLite backend does — check `reasons.pg.PgApi.retract_node`.
- **Caller obligation**: treat `changed` as the authoritative "what moved" list; don't re-derive by diffing before/after yourself — you don't have `before`.

## Dependencies

- `reasons.api._with_network` — context manager that handles `Storage.load`/`save`, cache, and pubsub.
- `reasons.api._pg_dispatch` — PG backend dispatcher.
- `reasons.network.Network.retract` — the actual cascade algorithm.
- `reasons.network.Network.nodes` / node `.truth_value` / node `.justifications` (with `.type` and `.antecedents`) — used by the classification and hint loops.
- `reasons.storage.Storage` — persistent store.
- `reasons.pubsub` — change event computation (only when subscribers are registered).
- Internal assumption: `Justification.type == "SL"` denotes a conjunctive ("ALL-of") rule and `len(antecedents) >= 2` means the premise set is non-trivial — this is a domain datamodel invariant, not enforced by any type annotation.

---

## Topics to Explore

- [function] `reasons/network.py:retract` — the actual cascade algorithm; tells you what `changed` really contains and how outlist restoration is triggered.
- [function] `reasons/api.py:what_if_retract` — read-only simulation counterpart; useful to compare with this write path.
- [file] `reasons/api.py` — `_with_network` is central to how retraction is persisted and how pubsub events are generated; read it before writing new write-path APIs.
- [file] `reasons/pg.py` — `PgApi.retract_node` is the backend your `pg_conninfo` path calls; verify the return-dict contract matches the local SQLite shape.
- [file] `tests/test_api.py` — asserts the return-dict shape for `retract_node` and related APIs; a good template if you add behavior.

## Beliefs

- `retract-node-returns-four-keys` — The local-backend return dict has exactly four keys: `changed`, `went_out`, `went_in`, `restoration_hints`.
- `retract-node-delegates-to-postgres-when-conninfo-set` — When `pg_conninfo` is truthy, the function bypasses the SQLite path entirely and dispatches to `PgApi.retract_node`.
- `retract-node-opens-write-transaction` — The SQLite path uses `_with_network(db_path, write=True)`, which always reloads from disk, re-saves on successful exit, refreshes the in-memory cache, and publishes pubsub events if subscribers are registered.
- `retraction-hints-scan-only-first-multi-antecedent-sl` — The hint loop `break`s after the first `SL` justification with `len(antecedents) >= 2`, so a node with multiple conjunctive justifications produces at most one hint.
- `retract-node-does-not-validate-node-id` — The function does not validate `node_id`; a missing node surfaces as a `KeyError` from `net.retract` only after the write context has opened (and the save is skipped on that exception).
- `restoration-hints-exclude-retracted-node` — The hint loop explicitly `continue`s on `nid == node_id`, so the retracted node itself never appears in `restoration_hints`.
