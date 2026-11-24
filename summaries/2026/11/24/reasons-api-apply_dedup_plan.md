---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: apply_dedup_plan in reasons/api.py

# `apply_dedup_plan` — Explanation

```python
def apply_dedup_plan(
    plan: list[dict],
    db_path: str = DEFAULT_DB,
) -> dict:
    with _with_network(db_path, write=True) as net:
        retracted = []
        errors = []
        for cluster in plan:
            keep = cluster["keep"]
            if keep not in net.nodes:
                errors.append(f"keep node not found: {keep}")
                continue
            for old_id in cluster["retract"]:
                if old_id not in net.nodes:
                    errors.append(f"retract node not found: {old_id}")
                    continue
                if net.nodes[old_id].truth_value == "OUT":
                    continue
                _rewrite_dependents(net, old_id=old_id, new_id=keep)
                net.retract(old_id)
                retracted.append(old_id)
        return {"applied": len(plan), "retracted": retracted, "errors": errors}
```

## 1. Purpose

`apply_dedup_plan` is the "commit" step of a two-phase deduplication workflow:

1. `deduplicate` finds clusters of duplicate beliefs (by ID-token Jaccard or embedding cosine) and their auto-chosen survivor (`kept`).
2. `write_dedup_plan` writes those clusters to a markdown file that a human edits (`KEEP` vs `RETRACT` per line).
3. `parse_dedup_plan` turns the reviewed markdown back into a structured list of `{"keep": ..., "retract": [...]}` dicts.
4. **`apply_dedup_plan` consumes that list** and actually does the work: rewires dependency edges so no belief loses justification, then retracts the duplicates.

The function exists precisely so that the decision (which node wins) can be reviewed by a human without touching the TMS, while the mechanical rewiring and retraction can be done atomically in a single `_with_network(..., write=True)` transaction.

## 2. Contract

**Preconditions** (mostly not enforced by types):
- `plan` must be a list of dicts, each with keys `"keep"` (str) and `"retract"` (list[str]). The type hint only says `list[dict]`; the shapes are assumed.
- `keep` must exist in the network *at loop time*. If not, the entire cluster is skipped (all its retract targets are abandoned).
- Each `retract` id must exist in the network; if not, an error string is appended and that id is skipped.
- `old_id`'s truth value must not already be `"OUT"` for it to be processed (already-OUT nodes are silently skipped).
- `db_path` must reference a valid RMS SQLite DB.

**Postconditions** (assuming no exception):
- The `keep` node's `dependents` set and its `justifications` on other nodes have been updated to reference `keep` in place of each retracted node's id (in antecedents *and* outlists).
- Each processed `retract` id is now `"OUT"` and its `dependents` set has been cleared of the rewritten nodes.
- The network is saved to disk; pubsub events are published; the `_network_cache` is refreshed.
- `return_value["retracted"]` contains the ids actually retracted (excluding ones skipped for already-OUT / missing).
- `return_value["applied"]` equals `len(plan)` — see warnings below.

**Invariants**:
- All plan entries are processed in order. Order matters **if** the same node appears as `keep` in one cluster and `retract` in another, or if `retract` lists overlap across clusters.
- Rewrite-before-retract ordering is essential: `_rewrite_dependents` is called *before* `net.retract` so that the TMS recompute doesn't cascade OUT dependents of the newly-retracted duplicate.

## 3. Parameters

- **`plan: list[dict]`** — Each element is expected to look like `{"keep": "b-node-keep", "retract": ["b-node-dup-1", "b-node-dup-2"]}`. This shape mirrors `parse_dedup_plan`'s output. Keys are accessed directly (`cluster["keep"]`, `cluster["retract"]`), so a missing key raises `KeyError`. Empty `retract` lists are allowed and simply do nothing.
- **`db_path: str`** — Path to the SQLite DB, defaults to `"reasons.db"`. It is passed straight to `_with_network(db_path, write=True)`, which loads a `Storage`, loads the `Network`, and will save it on successful exit. If the file doesn't exist or isn't a valid DB, `Storage(db_path)` will raise.

## 4. Return Value

Always a plain dict with three keys:

| key | type | meaning |
|---|---|---|
| `applied` | `int` | `len(plan)` — **not** the number of clusters that succeeded. See the caveat below. |
| `retracted` | `list[str]` | ids that were actually flipped OUT by this call. Already-OUT and missing ids are not included. |
| `errors` | `list[str]` | human-readable error strings for missing `keep` or missing `retract` ids. These do **not** abort the whole call; processing continues with remaining clusters/entries. |

The caller must handle:
- Repeated ids: nothing de-dupes `retracted` if the same node appears in two clusters (it would be OUT after the first, so the second pass would skip it, but `errors`/order can surprise).
- `applied != retracted count` — a successful "no error, nothing retracted" call returns `applied=1, retracted=[]`.
- `errors` strings are free-text; consumers scanning for structure will need to parse them.

## 5. Algorithm (step-by-step)

1. Enter `_with_network(db_path, write=True)`. This:
   - Attempts to reuse the cached `Network` only if `not write`; because `write=True` it *always* opens a fresh `Storage` and loads from disk.
   - Records `truth_value` snapshot for pubsub diffing.
   - Returns the loaded `Network` to the body.

2. Initialize `retracted = []` and `errors = []`.

3. For each `cluster` in `plan`:
   - `keep = cluster["keep"]` — direct key access; raises `KeyError` if the key is missing.
   - If `keep` is not in `net.nodes`: append `"keep node not found: {keep}"`, `continue` — so all of this cluster's retract targets are dropped.
   - For each `old_id` in `cluster["retract"]`:
     - If `old_id` missing: append `"retract node not found: {old_id}"`, `continue`.
     - If `net.nodes[old_id].truth_value == "OUT"`: `continue` silently (idempotency for re-applying plans).
     - Call `_rewrite_dependents(net, old_id=old_id, new_id=keep)`. This:
       - Iterates `old_node.dependents`.
       - For each dependent `dep`, rewrites each justification, replacing `old_id` with `keep` in both `antecedents` and `outlist`.
       - Adds the dependent to `keep.dependents` and discards it from `old_node.dependents`.
       - Note: it does **not** re-point *justifications of the old node itself* — the old node is about to be retracted anyway.
     - Call `net.retract(old_id)`. Because dependents were already rewritten, this retraction should not cascade OUT any former dependent.
     - Append `old_id` to `retracted`.
4. Return the summary dict. On successful exit of the `with` block, `_with_network.__exit__`:
   - `store.save(self.network)` — flushes changes to disk.
   - Computes before/after truth values and `pubsub.publish(events, db_path)`.
   - Closes `Storage`.
   - Updates `_network_cache[db_path] = (mtime, network)`.

## 6. Side Effects

- **DB reads**: opens the SQLite DB and loads the full `Network`.
- **In-memory mutation**: `_rewrite_dependents` mutates `Justification.antecedents`, `Justification.outlist`, and `Node.dependents` across the whole graph; `net.retract` mutates truth values (and calls TMS recompute).
- **DB writes**: `Storage.save` on successful exit. If `net.retract` triggers cascading truth changes (shouldn't, per the rewrite, but possible if the rewrite was incomplete), those are persisted too.
- **Event publishing**: `pubsub.compute_changes` + `pubsub.publish` fires for every node whose truth value changed.
- **Cache mutation**: updates `_network_cache[db_path]` so subsequent read-only calls reuse the fresh network.
- **No IPC/network** — purely local SQLite.

## 7. Error Handling

- **Unhandled exceptions**: 
  - `KeyError` on `cluster["keep"]` or `cluster["retract"]` if the plan dicts are malformed. This propagates out of the `with` block; because `exc_type is not None`, `_with_network.__exit__` **does not save**, so all in-memory changes are discarded. This is effectively an all-or-nothing abort, but the DB is left untouched (good) and the cache is not updated.
  - `Storage(...)` / `store.load()` errors if `db_path` is invalid.
  - Any exception raised inside `net.retract` (rare, e.g. malformed graph) is also surfaced and cancels the save.
- **Handled "errors"** (returned in `errors`, do not abort):
  - Missing `keep` node → whole cluster skipped.
  - Missing `retract` node → that id skipped.
- **Silently ignored**: already-OUT targets. Callers relying on `retracted` to detect "did anything change" should be aware.

## 8. Usage Patterns

Canonical CLI flow (implied by the sibling functions):

```python
# 1. Find duplicates and write a plan
result = deduplicate(threshold=0.6, auto=False, db_path=DB)
write_dedup_plan(result["clusters"], "proposed-dedup.md")

# 2. Human edits the markdown (KEEP vs RETRACT, deletions)

# 3. Parse and apply
plan_text = Path("proposed-dedup.md").read_text()
plan = parse_dedup_plan(plan_text)      # -> [{"keep":..., "retract":[...]}, ...]
summary = apply_dedup_plan(plan, db_path=DB)
```

Caller obligations:
- **Review the plan first** — this function does no semantic validation of the "keep" choice; it trusts the plan.
- **Pass already-parsed dicts**, not raw markdown — use `parse_dedup_plan`.
- **Do not rely on `applied` as a success counter** — use `len(retracted)` and `errors`.
- **Order clusters if nodes overlap** — the function processes them in listed order.

## 9. Dependencies

- `_with_network(db_path, write=True)` — module-level context manager; provides load/save lifecycle, pubsub, cache.
- `Network.retract(node_id)` — TMS retraction and propagation.
- `_rewrite_dependents(net, old_id, new_id)` — module helper that does the justification surgery; the semantic heart of the dedup.
- `Network.nodes` — dict of `Node` objects with `.truth_value`, `.dependents`, `.justifications`.
- Indirectly: `Storage` (disk I/O), `pubsub` (event fan-out), `DEFAULT_DB` constant, `_network_cache`.

### Assumptions the type system does not enforce

- `plan` entries carry `"keep"` and `"retract"` keys.
- Ids in the plan refer to real nodes at *loop time* — nothing prevents a plan that was valid at write time from drifting.
- `keep` is not also a member of `retract` for the same cluster. If it were, `_rewrite_dependents` would iterate `keep.dependents` while pointing `new_node == old_node`, and the subsequent `net.retract(keep)` would retract the survivor. Not caught.
- The reviewer actually intendend each `RETRACT` line — no semantic similarity is re-checked here.
- Sub-clusters are disjoint in a way that ordering doesn't matter.

---

## Topics to Explore

- [function] `reasons/api.py:parse_dedup_plan` — The exact inverse of `write_dedup_plan`; explains the shape `apply_dedup_plan` consumes and what lines it silently ignores.
- [function] `reasons/api.py:_rewrite_dependents` — The justification-rewrite helper; understanding its `dependents` swap is essential to why the post-rewrite `retract` is safe.
- [function] `reasons/api.py:deduplicate` — Produces the clusters consumed here, including the `kept` field that becomes `"keep"` in the plan.
- [file] `tests/test_api.py` — Shows canonical call sequences and any edge cases (already-OUT nodes, missing ids) exercised in CI.
- [general] `tms-retract-semantics` — How `Network.retract` cascades and why rewrite-before-retract avoids losing derived beliefs.

## Beliefs

- `apply-dedup-plan-reports-applied-as-len-plan` — `applied` equals `len(plan)` regardless of per-cluster success, so it is not a success counter.
- `apply-dedup-plan-rewrites-before-retract` — `_rewrite_dependents` is always called before `net.retract` on the same id, redirecting antecedents and outlists to the `keep` node first.
- `apply-dedup-plan-skips-already-out-nodes` — Retract targets whose truth value is already `"OUT"` are silently skipped and not listed in `retracted`.
- `apply-dedup-plan-missing-keep-aborts-cluster` — A missing `keep` node causes the entire cluster (all its retract targets) to be skipped with one error string.
- `apply-dedup-plan-requires-writable-db` — The function opens the DB via `_with_network(..., write=True)` and only persists changes when the `with` block exits without an exception.
