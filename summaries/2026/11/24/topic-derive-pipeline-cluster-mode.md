---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Explains how `cluster_beliefs` and `cluster_beliefs_intra` are used by the derive pipeline.

## How `cluster_beliefs` and `cluster_beliefs_intra` fit into the derive pipeline

Clustering is used as an optional belief-selection strategy inside the derive prompt builder, specifically in `_build_beliefs_section` in `reasons/derive.py`. Instead of alphabetically truncating or randomly sampling IN beliefs, derive can let semantic clustering choose which beliefs the LLM sees.

### 1. Input set passed to clustering

`_build_beliefs_section` first computes:

```python
in_nodes = {k: v for k, v in nodes.items()
            if v.get("truth_value") == "IN" and k not in derived}
```

It then converts this to `{node_id: text}`:

```python
belief_texts = {k: v["text"] for k, v in in_nodes.items()}
```

So both clustering functions receive only IN, non-derived beliefs, as a mapping from belief ID to belief text.

### 2. Mode dispatch in `reasons/derive.py`

The derive builder supports two clustered modes:

- If `intra_cluster=True`, it lazily imports `cluster_beliefs_intra as _cluster_intra` at `reasons/derive.py:211` and calls it with `round_num`.
- Else if `cluster=True`, it lazily imports `cluster_beliefs as _cluster` at `reasons/derive.py:219` and calls it without `round_num`.
- Because the code uses `if intra_cluster: ... elif cluster: ...`, `intra_cluster` takes precedence if both flags are set.
- If neither mode is active, neither clustering function is imported or called, and the normal sampling/truncation logic is used.

The calls pass through the derive-level options:

```python
selected_ids, cluster_stats = _cluster_intra(
    belief_texts, max_beliefs, round_num=round_num, seed=seed,
    n_clusters=n_clusters, cache=cluster_cache,
    model_name=embedding_model or "all-MiniLM-L6-v2",
)
```

and similarly:

```python
selected_ids, cluster_stats = _cluster(
    belief_texts, max_beliefs, seed=seed, n_clusters=n_clusters,
    cache=cluster_cache, model_name=embedding_model or "all-MiniLM-L6-v2",
)
```

Here `max_beliefs` becomes the clustering `budget`.

### 3. What each clustering function does

`cluster_beliefs` is defined at `reasons/cluster.py:90`. It clusters all belief texts and spreads the budget across clusters, so the selected set is more likely to cover multiple semantic domains. It computes `base_per = budget // k`, distributes any remainder, and samples from clusters sorted by size.

`cluster_beliefs_intra` is defined at `reasons/cluster.py:150`. It uses the same embedding/KMeans setup but focuses the budget on one cluster per round:

```python
focus_idx = round_num % k
focus_label = sorted_labels[focus_idx]
members = clusters[focus_label]
selected = rng.sample(members, min(budget, len(members)))
```

So `cluster_beliefs_intra` gives the LLM topically adjacent beliefs for a given derive round, then rotates to another cluster as `round_num` increases.

Both functions:

- Use `ClusterCache` when provided, enabling embedding reuse across rounds.
- Use `KMeans(..., random_state=seed, n_init=10)`.
- Return `(selected_ids, cluster_stats)`.
- Include stats such as `n_clusters`, `cluster_sizes`, and `embedding_model`; intra-cluster mode also includes `focus_cluster`.

### 4. How derive returns cluster stats

`_build_beliefs_section` returns:

```python
return "\n".join(lines), cluster_stats
```

when clustering is active. The selected IDs are formatted into prompt sections grouped by agent namespace or by the prefix before `-` in the belief ID. The `cluster_stats` object is returned alongside the rendered section so the caller can surface or log clustering information. When clustering is off, `cluster_stats` is `None`.

### 5. Test coverage

`tests/test_cluster.py` covers `cluster_beliefs` behavior, including budget selection, reproducibility, cross-domain sampling, cache reuse, stats shape, and `n_clusters` override.

`tests/test_derive.py:1065-1068` directly imports and calls `cluster_beliefs_intra` with `round_num=0` and `round_num=1`, indicating that round-based cluster rotation is part of the derive test surface.

### What is not visible in the supplied observations

The exact CLI flag parsing and the caller of `_build_beliefs_section` are not included in the returned portion of `reasons/derive.py`. Those lines likely show how `cluster`, `intra_cluster`, `round_num`, `cluster_cache`, and `embedding_model` are supplied from the derive command or exhaustive-round loop.

## Topics to Explore

- [function] `reasons/derive.py:_build_beliefs_section` — Central dispatch that filters IN beliefs, calls either clustering function, formats the prompt section, and returns cluster stats.
- [function] `reasons/cluster.py:cluster_beliefs` — Implements cross-cluster budget allocation for broad semantic coverage.
- [function] `reasons/cluster.py:cluster_beliefs_intra` — Implements per-round single-cluster focus and rotation via `round_num % k`.
- [function] `reasons/cluster.py:ClusterCache.embed` — Reuses embeddings across derive rounds, which matters for exhaustive or multi-round derive runs.
- [file] `reasons/derive.py` — Shows where `cluster_stats` is surfaced and how CLI/build-prompt options are wired; relevant lines extend beyond the observed 400 lines.

## Beliefs

- `derive-clustering-uses-in-non-derived-beliefs` — `_build_beliefs_section` filters to `truth_value == "IN"` nodes excluding derived nodes before passing `{id: text}` to clustering.
- `derive-lazily-imports-cluster-functions` — `reasons/derive.py` imports `cluster_beliefs_intra` at line 211 and `cluster_beliefs` at line 219 only inside the active clustering branch.
- `intra-cluster-mode-takes-precedence` — In `_build_beliefs_section`, `if intra_cluster ... elif cluster` means setting both flags selects `cluster_beliefs_intra`.
- `cluster-beliefs-spreads-budget-across-clusters` — `cluster_beliefs` allocates `budget // k` plus remainder across sorted clusters to sample across semantic domains.
- `cluster-beliefs-intra-rotates-focus-cluster` — `cluster_beliefs_intra` selects `focus_idx = round_num % k` and returns it as `cluster_stats["focus_cluster"]`.
