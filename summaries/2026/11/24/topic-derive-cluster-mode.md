---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How `round_num`, `cache`, and clustering strategy are wired into the derive pipeline.

# How `round_num`, `cache`, and clustering strategy are wired into the derive pipeline

Three orthogonal concerns are threaded through the derive pipeline:

- **`round_num`** — an integer round counter, meaningful **only** in `--intra-cluster` mode where it selects which cluster to focus on this round.
- **`cluster_cache`** — a `ClusterCache` object (optional) that reuses embeddings across rounds.
- **clustering strategy** — two booleans, `cluster` (spread) and `intra_cluster` (focus on one cluster per round), selected in `_build_beliefs_section`.

All three are parameters that flow downward, unchanged, from the CLI entry point all the way into `reasons/cluster.py`.

---

## 1. The CLI entry point and exhaust loop (`reasons/cli.py`)

```
for round_num in range(1, max_rounds + 1):              # line 1734
    print(f"Round {round_num}/{max_rounds}", ...)        # line 1736
    added = _derive_one_round(args, round_num=round_num, # line 1738
                              ...)
```

`_derive_one_round` (defined at line 1485) is responsible for translating CLI args into the derive call:

- Line 1502 uses `round_num` only as a **log prefix**: `f"[round {round_num}] "`.
- Line 1522 forwards it to the library API as `derive(..., round_num=round_num or 0, cluster_cache=cluster_cache, ...)`. The `or 0` coerces `None` (single-shot path) to `0`, so `focus_idx = 0 % k = 0` on a single invocation. In the exhaust loop the first value is `1`, so the first round already picks `focus_idx = 1 % k` — a subtle quirk worth knowing.
- Lines 1581 and 1598 stash it in the report state as `"round": round_num or 1` — note the **different default** (`1` vs `0`) than the derive call.

⚠️ **Gap:** The snippet at line 1485 is `def _derive_one_round(args, round_num=None, report_state=None,` — truncated. So we don't see the rest of the signature or whether `cluster_cache` / `cluster` / `intra_cluster` flags are plumbed here from `argparse`. Likewise, the body shows `cluster_cache=cluster_cache` being **passed in**, but not where `ClusterCache` is instantiated before the loop. That is the missing piece required to confirm cache reuse across rounds.

---

## 2. `derive()` — the library entry (`reasons/derive.py`)

At line 441 the function signature includes:

```python
cluster=False, intra_cluster=False, round_num=0,
```

and line 534 forwards them to `_build_beliefs_section`:

```python
round_num=round_num, cluster_cache=cluster_cache, ...
```

So `derive()` is a thin pass-through for these three concerns — it doesn't interpret them itself; it only packages the prompt and calls the parser.

---

## 3. `_build_beliefs_section` — the strategy switch (`reasons/derive.py:187`)

This is where the wiring actually branches. The signature includes:

```python
cluster=False, intra_cluster=False,
round_num=0, cluster_cache=None,
embedding_model=None, n_clusters=None,
```

The dispatch logic:

```python
if intra_cluster:                                             # wins over `cluster`
    from .cluster import cluster_beliefs_intra as _cluster_intra
    belief_texts = {k: v["text"] for k, v in in_nodes.items()}
    selected_ids, cluster_stats = _cluster_intra(
        belief_texts, max_beliefs, round_num=round_num, seed=seed,
        n_clusters=n_clusters, cache=cluster_cache,
        model_name=embedding_model or "all-MiniLM-L6-v2",
    )                                                          # line ~213–219
elif cluster:
    from .cluster import cluster_beliefs as _cluster
    ...
    selected_ids, cluster_stats = _cluster(
        belief_texts, max_beliefs, seed=seed, n_clusters=n_clusters,
        cache=cluster_cache, model_name=embedding_model or "all-MiniLM-L6-v2",
    )                                                          # line ~221–226
```

Key properties of this wiring:

1. **`intra_cluster` supersedes `cluster`** via the `if / elif`. Both flags set → intra wins.
2. The `.cluster` import is **lazy** (inside the branch), so the optional `sentence-transformers` / `scikit-learn` deps are only imported when clustering is enabled.
3. The same `cluster_cache` object is passed to whichever strategy runs.
4. When clustering is active, the function returns early with `(section_text, cluster_stats)` and skips the older agent-budget / prefix-grouping paths.

---

## 4. The cluster module (`reasons/cluster.py`)

### `ClusterCache`

- Holds a `SentenceTransformer` model plus a dict `_cache`.
- Cache key is `(node_id, sha256(text.encode()).hexdigest()[:16])`. So editing a belief's text invalidates its cache entry.
- `embed()` sorts belief IDs, only encodes uncached ones, and reassembles outputs in the same sorted order — critical so that `KMeans` predictions stay aligned with `ids`.

### `cluster_beliefs` — spread strategy

1. `if cache is None: cache = ClusterCache(model_name)` (lazy creation).
2. `cache.embed(beliefs)` → `ids, embeddings`.
3. `k = _auto_k(...)` → roughly `max(2, min(n // 5, 20))`, further capped at `budget // 3` by the caller.
4. `KMeans(n_clusters=k, random_state=seed, n_init=10)`.
5. Allocate `budget // k + remainder` to each cluster, sampled with `random.Random(seed)`.

### `cluster_beliefs_intra` — focus strategy (where `round_num` matters)

Identical setup through `KMeans`, then:

```python
sorted_labels = sorted(clusters, key=lambda c: -len(clusters[c]))  # by size desc
focus_idx = round_num % k                                          # line 188
focus_label = sorted_labels[focus_idx]
members = clusters[focus_label]
selected = rng.sample(members, min(budget, len(members)))
```

So `round_num` is used as a **rotation counter across the size-ordered cluster list** — every round the LLM sees beliefs from exactly one topic, cycling through all `k` topics before repeating.

### `_auto_k` (shared)

```python
if n_clusters is not None:
    k = max(n_clusters, 1)
else:
    k = min(n_beliefs // 5, 20)
    k = max(k, 2)
return min(k, n_beliefs)
```

The two cluster functions additionally cap this at `budget // 3`, so the sampling budget constrains how fine-grained the clustering can be.

---

## 5. Summary of data flow

```
argparse  (not shown)
   │
   ▼
_derive_one_round(args, round_num=..., ...)     cli.py:1485
   │  round_num or 0 ; cluster_cache=...
   ▼
derive(..., cluster=?, intra_cluster=?, round_num=0,
       cluster_cache=None, embedding_model=?, n_clusters=?)   derive.py:441
   │  forwards
   ▼
_build_beliefs_section(..., round_num, cluster_cache,
                       cluster, intra_cluster,
                       embedding_model, n_clusters)           derive.py:187
   │
   ├── intra_cluster →  cluster_beliefs_intra(..., round_num=round_num,
   │                                          cache=cluster_cache, ...)
   └── cluster      →  cluster_beliefs(..., cache=cluster_cache, ...)
                                                              cluster.py
```

- **`round_num` reaches its destination only at `cluster_beliefs_intra` line 188.** In spread-cluster mode it's ignored; with neither flag it's unused entirely.
- **`cluster_cache` is threaded through every layer unchanged.** If `None`, each cluster call constructs its own cache — which defeats the whole point in `--exhaust` mode, so the caller must build one before the loop.
- **Strategy** is decided by the if/elif in `_build_beliefs_section`; `embedding_model` and `n_clusters` ride along to both strategies.

---

## What's missing from the observations

- Where `ClusterCache(...)` is actually instantiated — probably just before the `for round_num in ...` loop at `reasons/cli.py:1734`, but the snippet doesn't show it.
- The full `_derive_one_round` signature and its `argparse`-to-kwargs mapping (line 1485 truncates).
- The complete `derive(...)` call at `reasons/cli.py:1522` — only `round_num` and `cluster_cache` are visible; `cluster`, `intra_cluster`, `embedding_model`, and `n_clusters` are presumably forwarded too but not shown.
- Lines 400–697 of `reasons/derive.py` — specifically how `cluster_stats` is used in prompt metadata and what `derive()` returns.
- Note the discrepancy: `locate_cache_references_in_derive` reports **0 matches** for `"cache"`, but the actual file uses `cluster_cache` at lines 187, 214, and 534. The pattern was almost certainly evaluated with a word boundary (`\bcache\b`), so `cluster_cache` didn't match. Trust the file contents over that search result.

---

## Topics to Explore

- [function] `reasons/cli.py:_derive_one_round` — Full signature and how `argparse` values (including `cluster_cache`) are plumbed through to the `derive(...)` call at line 1522
- [file] `reasons/derive.py` — Read lines 400–697 to see how `cluster_stats` is embedded in prompt metadata and returned to callers
- [function] `reasons/cluster.py:ClusterCache.embed` — Understand cache invalidation semantics (`sha256(text)[:16]` key) and ordering guarantees
- [general] `exhaust-mode-cache-lifecycle` — Trace where `ClusterCache` is instantiated so a single cache object survives all `--exhaust` rounds
- [file] `reasons/cli.py` — Inspect `argparse` definitions for `--cluster`, `--intra-cluster`, `--embedding-model`, and `--n-clusters` flags

---

## Beliefs

- `round-num-drives-intra-cluster-focus` — In `cluster_beliefs_intra`, `round_num % k` selects `focus_idx` from size-sorted clusters, rotating the per-round topical focus.
- `intra-cluster-supersedes-cluster` — `_build_beliefs_section` uses `if intra_cluster: … elif cluster: …`, so both flags set means intra-cluster wins.
- `cluster-cache-is-keyed-by-node-id-and-text-hash` — `ClusterCache._cache` keys on `(node_id, sha256(text)[:16])`, so editing belief text invalidates its cached embedding.
- `round-num-defaults-diverge-between-derive-and-report` — `_derive_one_round` passes `round_num or 0` to `derive()` but stores `round_num or 1` in report state.
- `cluster-functions-are-lazily-imported` — `_build_beliefs_section` imports `cluster_beliefs` / `cluster_beliefs_intra` inside the branch, so `sentence-transformers` and `scikit-learn` are only required when clustering is on.
