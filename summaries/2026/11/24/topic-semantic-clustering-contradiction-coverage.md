---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The tradeoff between topical grouping and missing contradictions that span different clusters.

## The tradeoff

`reasons/cluster.py` implements topical grouping by embedding belief text and running KMeans. `list_clusters` (`reasons/cluster.py:203-254`) returns a hard partition: every belief is assigned to exactly one cluster. `detect_contradictions_semantic` (`reasons/contradictions.py:182-256`) then uses that partition and checks each cluster independently.

The critical line is the loop at `reasons/contradictions.py:232-254`:

```python
for ci, cluster in enumerate(clusters, 1):
    cluster_ids = [b["id"] for b in cluster["beliefs"]]
    ...
    for i in range(0, len(cluster_ids), CONTRADICTION_BATCH_SIZE):
        batch = cluster_ids[i:i + CONTRADICTION_BATCH_SIZE]
```

Each LLM prompt is built only from `cluster_ids` for the current cluster. A contradiction whose claims land in two different clusters is therefore structurally invisible: the LLM never sees both claims in the same prompt.

### What topical grouping buys

Grouping semantically similar beliefs helps because contradiction detection is comparative. The LLM needs related claims in the same context. Clustering makes that more likely for local contradictions: direct negations, incompatible properties, scope conflicts, and quantitative mismatches that share vocabulary. It also keeps each prompt focused and reduces the amount of unrelated text the model must reason over.

`cluster_beliefs_intra` (`reasons/cluster.py:149-200`) is the clearest expression of this design intent: it focuses a round on one cluster to give the LLM “topically adjacent beliefs that are more likely to combine” (`reasons/cluster.py:152-155`). That is the benefit in pure form.

### What it costs

The cost is exactly the cross-cluster contradiction. Many real contradictions are not semantically local: one claim may be an architectural rule in one cluster, while the conflicting claim is an implementation detail in another. For example:

- “All API handlers must be idempotent.”
- “Retrying `POST /charge` increments the order counter.”

Those may embed into different clusters. `detect_contradictions_semantic` checks each cluster separately, so neither prompt contains both claims.

The non-semantic baseline, `detect_contradictions` (`reasons/contradictions.py:121-179`), avoids the hard partition by shuffling all IN belief IDs at `reasons/contradictions.py:153` and batching sequentially at `reasons/contradictions.py:160-161`. That can mix clusters, but only by chance. For large networks, two claims that belong together may not land in the same 50-item batch. So random batching preserves a chance of cross-cluster co-batching, but at the cost of noisy prompts and lower probability of catching subtle local contradictions.

### Compounding factors in the current implementation

- `detect_contradictions_semantic` skips clusters with fewer than two beliefs at `reasons/contradictions.py:234-235`. Singleton beliefs—often outliers or bridge concepts—are never checked against anything in that path.
- It uses the fixed constant `CONTRADICTION_BATCH_SIZE = 50` (`reasons/contradictions.py:12`) at `reasons/contradictions.py:240`; it does not accept a `batch_size` parameter. Large clusters are split into deterministic 50-item windows in sorted-ID order, because `ClusterCache.embed` sorts IDs (`reasons/cluster.py:51`) and `list_clusters` preserves that order (`reasons/cluster.py:232-243`). Even same-cluster contradictions can be split across batches.
- `detect_contradictions_semantic` does not expose `n_clusters` or `seed` (`reasons/contradictions.py:182-184`); it calls `list_clusters` at `reasons/contradictions.py:222-225` without those options. Cluster granularity is therefore fixed by `_auto_k` (`reasons/cluster.py:78-86`, called from `list_clusters` at `reasons/cluster.py:234`). You cannot tune the cross-cluster/local-density tradeoff without changing code.
- The codebase already has a cross-cluster sampler for the derive pipeline: `cluster_beliefs` (`reasons/cluster.py:89-146`) allocates budget across every cluster and samples members from each. It is explicitly tested for cross-domain selection in `tests/test_cluster.py:56-67`. But `detect_contradictions_semantic` uses `list_clusters`, not `cluster_beliefs`, so that cross-cluster sampling capability is not applied to contradiction checking.

The practical tradeoff is therefore: topical grouping raises recall for local contradictions and reduces prompt noise, but a hard partition guarantees that cross-cluster nogoods are missed unless you add a hybrid pass. A hybrid would run the existing within-cluster checks plus a cross-cluster pass that deliberately puts representatives from multiple clusters—or suspected bridge beliefs—into the same prompt.

### What the observations do not show

The provided `tests/test_contradictions.py` only includes lines 1-300 of 570, so any semantic-detector tests later in that file are not visible. The CLI/API call site that selects `detect_contradictions_semantic` and any options for cross-cluster checks are also not shown. The observations also do not include empirical cluster quality, model token limits, or how often cross-cluster contradictions occur in real networks.

## Topics to Explore

- [function] `reasons/contradictions.py:detect_contradictions_semantic` — See exactly where the per-cluster loop prevents cross-cluster prompts.
- [function] `reasons/cluster.py:cluster_beliefs` — Compare its cross-cluster budget allocation with the hard partition used by contradiction detection.
- [function] `reasons/cluster.py:list_clusters` — Understand how `_auto_k` and KMeans granularity affect how many contradictions become cross-cluster by construction.
- [file] `tests/test_contradictions.py` — Check the remaining lines 301-570 for semantic-detector coverage and any cross-cluster test cases.
- [general] `hybrid-cross-cluster-contradiction-check` — Design a two-pass detector that keeps within-cluster checks but adds mixed-cluster representative batches.

## Beliefs

- `detect-contradictions-semantic-precludes-cross-cluster-prompts` — `detect_contradictions_semantic` builds each prompt from one cluster’s IDs only, so claims in different clusters never share a prompt.
- `cluster-beliefs-implements-cross-cluster-sampling` — `cluster_beliefs` allocates budget across clusters and samples members from each, unlike `list_clusters`, which returns a hard partition.
- `semantic-contradiction-detector-skips-singleton-clusters` — `detect_contradictions_semantic` continues when a cluster has fewer than two beliefs.
- `semantic-contradiction-detector-has-fixed-cluster-granularity` — It does not expose `n_clusters` or `seed` and relies on `_auto_k` defaults via `list_clusters`.
- `contradiction-batch-size-is-constant-50` — `CONTRADICTION_BATCH_SIZE` is 50 and semantic contradiction detection uses it for within-cluster batching.
