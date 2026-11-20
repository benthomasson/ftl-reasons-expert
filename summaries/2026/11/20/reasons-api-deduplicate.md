---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: deduplicate in reasons/api.py

# `deduplicate` (`reasons/api.py`)

## Purpose

Finds sets of **currently-IN beliefs that look like duplicates** of each other and returns them as clusters — optionally auto-retracting all but one member of each cluster. It is the foundation for two higher-level workflows:

1. **Human-reviewed dedup** — call `deduplicate(...)`, feed the clusters into `write_dedup_plan()`, let a human edit the plan, then `parse_dedup_plan()` → `apply_dedup_plan()`.
2. **Automatic dedup** — call `deduplicate(auto=True)` and let the function itself retract and rewire.

The similarity metric is *either* a cheap lexicographic signal (Jaccard over tokens of the node **ID**) or an expensive semantic signal (cosine similarity over text embeddings).

## Contract

- **Precondition**: `db_path` points at an initialized RMS SQLite database. Not enforced — missing tables/rows surface as `sqlite3` / `KeyError` errors.
- **Postcondition (auto=False)**: pure read. Returns clusters of size ≥ 2 whose pairwise similarity (under the chosen metric) is ≥ `threshold`. No mutations.
- **Postcondition (auto=True)**: for each cluster, the non-`kept` members are retracted and any node that referenced them has its justifications rewritten to point at the `kept` node. The updated network is persisted on `_with_network` exit.
- **Invariants**:
  - Only nodes with `truth_value == "IN"` are considered (OUT nodes are invisible to this function).
  - Clusters are non-overlapping by construction (union-find guarantees disjoint sets).
  - Every returned cluster has `size >= 2`.
  - `retracted` is always present in the return dict (empty list when `auto=False`).
  - Within a cluster, the chosen `kept` node is a member of `beliefs`.

## Parameters

| Name | Type | Default | Meaning |
|---|---|---|---|
| `threshold` | `float` | `0.5` | Similarity cutoff. Jaccard range is `[0,1]`; cosine range is typically `[-1,1]` but sentence-transformer outputs are usually `[0,1]`. Values > 1 mean *nothing clusters* in either mode. Negative values cluster everything. |
| `auto` | `bool` | `False` | If True, performs retraction + dependent rewriting. This flips `_with_network(..., write=True)`, bypasses the read cache, and saves at exit. |
| `semantic` | `bool` | `False` | Chooses embedding cosine similarity over node **text** instead of Jaccard over node **ID tokens**. Also switches the imported dependencies. |
| `embedding_model` | `str \| None` | `None` | Only meaningful when `semantic=True`; otherwise ignored. Falls back to `cluster.DEFAULT_MODEL`. |
| `db_path` | `str` | `"reasons.db"` | Path to the SQLite DB. |

**Edge cases not enforced by types**:
- `threshold` above 1.0 is not rejected — silently yields zero clusters.
- `embedding_model` is silently ignored when `semantic=False`.
- If `cache.embed` returns a different ID set or order than `in_nodes`, the union-find `find(ids[i])` will raise `KeyError` — no defensive check.
- `auto=True` with `semantic=True` requires the model; a missing model / missing `numpy` raises.

## Return Value

```python
{
    "clusters": [
        {
            "beliefs": [{"id": str, "text": str, "dependents": int}, ...],
            "size": int,
            "kept": str,          # node to preserve if auto
        },
        ...
    ],
    "retracted": [str, ...],      # node IDs retracted (empty when auto=False)
}
```

- Clusters are sorted by `size` **descending**.
- `beliefs` is sorted by node ID.
- Empty network (no IN nodes, or zero similar pairs) returns `{"clusters": [], "retracted": []}`.
- The `kept` member is *also present in the `beliefs` list* — callers must not treat it as "to-delete".

## Algorithm

1. **Mode dispatch**: import `ClusterCache, DEFAULT_MODEL, numpy` when `semantic=True`, otherwise `_tokenize_id, _jaccard` from `derive`.
2. **Open network** via `_with_network(db_path, write=auto)`.
3. **Snapshot**: `in_nodes = [(nid, node) for nid, node in sorted(net.nodes.items()) if node.truth_value == "IN"]`. Empty → immediate return.
4. **Compute similarity inputs**:
   - Semantic: build `{id: text}`, ask `ClusterCache(embedding_model or DEFAULT_MODEL).embed(...)` for `(ids, embeddings)`, L2-normalize rows (with zero-norm guard), then `sim_matrix = normed @ normed.T`.
   - Token: `tokens = {nid: _tokenize_id(nid)}` for each IN node.
5. **Union-Find** initialization: `parent[nid] = nid` for every IN node. Two closures `find` (path-halving) and `union` (simple attach-root).
6. **Pairwise join**: O(n²). Semantic: `sim_matrix[i,j] >= threshold → union(ids[i], ids[j])`. Token: `_jaccard(tokens[a], tokens[b]) >= threshold → union(a, b)`.
7. **Group**: `defaultdict(list)` keyed by `find(nid)`.
8. **Filter/emit**: skip groups of size 1; otherwise build `beliefs` list (id, text, dependents-count from `net.nodes[nid].dependents`), compute `kept = max(members, key=lambda nid: (len(net.nodes[nid].dependents), nid))`.
   - **Tiebreak subtlety**: `max(..., (dependents, nid))` picks the *largest* depends-count; on ties it picks the *largest* node ID (not the smallest — easy to misread).
9. **Auto path** (only if `auto=True`): for each non-kept member, call `_rewrite_dependents(net, old_id=nid, new_id=keep)` **first**, then `net.retract(nid)` and append `nid` to `retracted`.
   - Ordering matters: rewriting first means dependents' justifications already point at `keep` by the time `retract` operates.
10. Sort `clusters` by `-size` and return.

## Side Effects

- **`auto=True`**:
  - Mutates `Network` in-memory: rewrites `antecedents`/`outlist` in dependent justifications, updates `dependents` reverse index, retracts members.
  - `_with_network(write=True)` → persists via `Storage.save`, publishes `pubsub` truth-value-change events.
  - No `recompute_all()` is called inside `deduplicate` — it relies on `net.retract()` doing its own propagation. This is unlike `defeat_justification`/`defeat_with_scope`, which explicitly call `net.recompute_all()`.
- **`auto=False`**: no DB writes, but `_with_network` will refresh the in-memory cache entry for `db_path` as a read-caching side effect.
- **Semantic mode**: may download / load a sentence-transformers model on first call (via `ClusterCache`).
- **No logging / printing**.

## Error Handling

No `try/except` — everything propagates:
- `sqlite3.OperationalError` / `DatabaseError` if the DB is corrupt or lacks expected schema.
- `KeyError` if a Node ID from `ClusterCache` isn't in `parent` (misalignment between embeddings and IN nodes).
- `ImportError` for `numpy` or sentence-transformers if `semantic=True` and deps missing.
- `Exception` from model loading / embedding computation.
- Empty network is handled gracefully (not an error).

## Usage Patterns

```python
# Human review path
result = deduplicate(threshold=0.6, db_path=db)
write_dedup_plan(result["clusters"], "proposed-dedup.md")
# ... human edits ...
plan = parse_dedup_plan(Path("proposed-dedup.md").read_text())
apply_dedup_plan(plan, db_path=db)

# Auto path (irreversible; treat with care)
result = deduplicate(threshold=0.8, auto=True, db_path=db)
```

Caller obligations:
- **Never assume** the `kept` node is *only* in the `kept` field — it's also in `beliefs`.
- When `auto=True`, downstream users should expect derived facts to have been **rewired** not retracted — that's the whole point of `_rewrite_dependents`, but it means siblings' justifications silently changed.
- The threshold's semantics change with `semantic=True` — a value tuned for Jaccard does not transfer to cosine.

## Dependencies

- **`_with_network`** (same file) — load/save context manager.
- **`reasons.derive._tokenize_id`, `reasons.derive._jaccard`** — token similarity primitives.
- **`reasons.cluster.ClusterCache`, `DEFAULT_MODEL`** — embedding cache (semantic mode).
- **`numpy`** — normalized embedding matrix + dot-product similarity.
- **`_rewrite_dependents`** (same file, defined above) — rewrites antecedents/outlist and repairs reverse index.
- **`Network.retract`** — truth-value cascade; called once per retracted node.

## Assumptions Not Enforced by Types

1. **`ClusterCache.embed` preserves correspondence between returned `ids` and `embeddings`, and both match the input `beliefs` IDs.** If not, `find(ids[i])` raises `KeyError` mid-loop.
2. **`net.retract(nid)` is sufficient** on its own to leave the network in a consistent state — `deduplicate` does *not* explicitly call `net.recompute_all()` after retracting.
3. **`_rewrite_dependents(nid → keep)` is done before `retract(nid)`**; any change to that ordering would cascade-OUT dependents.
4. **`threshold` unit consistency** — no validation that `semantic=True` callers are using a cosine scale rather than a Jaccard scale.
5. **`threshold` ∈ `[0, 1]`** — negative values would cluster every pair (useful only for debugging).
6. **Node equality is by ID** — there's no protection against ID collisions or the `kept` node being retracted later by a different cluster (impossible in practice because union-find disjoints clusters, but structurally undefended).

## Topics to Explore

- [function] `reasons/api.py:_rewrite_dependents` — Rewrites justifications to point at the surviving node; understand exactly which references are touched.
- [function] `reasons/network.py:Network.retract` — Whether it recomputes/redistributes truth values, or just flips the flag.
- [function] `reasons/derive.py_something:_jaccard` — Tokenization strategy for IDs (determines what "similar ID" means in practice).
- [file] `reasons/cluster.py` — `ClusterCache`, `DEFAULT_MODEL`, and how embeddings are cached/loaded.
- [file] `tests/test_api.py` — Existing coverage of `deduplicate` return shape and `auto=True` semantics.

## Beliefs

- `deduplicate-consideres-only-in-nodes` — Only nodes with `truth_value == "IN"` are candidates; OUT nodes are silently excluded from every cluster.
- `auto-true-enables-dependents-rewrite` — When `auto=True`, dependents' justifications are rewritten to reference the kept node *before* the duplicate is retracted, preserving derived beliefs.
- `kept-node-tiebreaks-by-largest-id` — On ties in dependents count, `max(..., key=(dependents, nid))` picks the *largest* node ID, not the smallest.
- `deduplicate-does-not-call-recompute-all` — Unlike `defeat_justification`, this function does not call `net.recompute_all()` after retractions; consistency relies on `net.retract` alone.
- `semantic-mode-requires-numpy-and-cluster-cache` — `semantic=True` imports `numpy` and `reasons.cluster.ClusterCache`, both of which can fail lazily at call time.
