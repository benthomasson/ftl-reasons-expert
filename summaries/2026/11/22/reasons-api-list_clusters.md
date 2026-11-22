---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: list_clusters in reasons/api.py

## `list_clusters` — Semantic clustering of beliefs

### 1. Purpose

`list_clusters` is a thin orchestration wrapper that:

1. Loads the belief network read-only from SQLite.
2. Filters nodes by truth value (`status`) and access tags (`visible_to`).
3. Hands the surviving `{node_id: text}` mapping to a downstream embedding-based clusterer in `reasons/cluster.py`.

It exists as an API layer entry point (used by CLI and by `build_wiki`) so callers don't have to touch `Storage`, the visibility check, or the sentence-transformer model lifecycle themselves.

### 2. Contract

- **Precondition:** the SQLite file at `db_path` exists (otherwise `Storage` / `_with_network` will raise). The `reasons.cluster` module and its dependencies (a sentence-transformers model) must be importable.
- **Postcondition:** returns a JSON-serializable dict describing semantic clusters. The database is not mutated.
- **Invariant:** filters are applied *before* embedding — the downstream model never sees excluded nodes, so clustering can't leak hidden beliefs via shared semantics.

### 3. Parameters

| Param | Type | Meaning / edge cases |
|---|---|---|
| `status` | `str`, default `"IN"` | Truth-value filter. The check is `if status and n.truth_value != status:` — so **`""` disables filtering entirely** (all nodes, IN and OUT). `None` behaves the same as `""` due to truthiness. |
| `n_clusters` | `int \| None` | Passed through to the clusterer; downstream decides auto-detection when `None`. No validation here (e.g. `n_clusters > len(beliefs)` is not checked). |
| `seed` | `int \| None` | Reproducibility seed forwarded to the clusterer. |
| `embedding_model` | `str \| None` | Sentence-transformers model name. `None` → `DEFAULT_MODEL` from `reasons.cluster`. Also echoed back in the empty-result response. |
| `visible_to` | `list[str] \| None` | Access-tag allow-list used by `_is_visible`. `None` = no filtering. `[]` = only nodes with **no** `access_tags` pass, since tagged nodes require all their tags ⊆ `visible_to`. |
| `db_path` | `str`, default `DEFAULT_DB` | Path to the SQLite file. |

### 4. Return Value

- **Empty case** (no beliefs survived filters): `{"clusters": [], "n_clusters": 0, "embedding_model": <resolved model>}`.
- **Normal case**: whatever `reasons.cluster.list_clusters` returns — from the `build_wiki` caller we can infer the shape is `{"clusters": [{"id": ..., "beliefs": [{"id": <node_id>, ...}, ...], ...}, ...]}` plus likely `n_clusters` and `embedding_model` keys.

Callers must be prepared for either shape; `build_wiki` guards with `cluster_result["clusters"]` (always present) but the per-cluster `id` key is an assumption not enforced by the type system.

### 5. Algorithm

1. `from .cluster import list_clusters as _list_clusters, DEFAULT_MODEL` — **lazy import** so the heavy sentence-transformers dependency is only loaded when this API is actually called.
2. Enter `_with_network(db_path)` with default `write=False` → either reuses the mtime-validated cache or does a fresh `Storage.load()`.
3. Iterate nodes `sorted(net.nodes.items())` (deterministic order):
   - Skip if `status` is truthy and `node.truth_value != status`.
   - Skip if `visible_to is not None` and `not _is_visible(node, visible_to)`.
   - Accumulate `beliefs[nid] = n.text`.
4. Exit the context manager (storage closed, cache updated).
5. If `beliefs` is empty, short-circuit and return `{"clusters": [], "n_clusters": 0, "embedding_model": embedding_model or DEFAULT_MODEL}` — note it *does not* invoke the model in this branch.
6. Otherwise delegate: `_list_clusters(beliefs, n_clusters=..., seed=..., model_name=embedding_model or DEFAULT_MODEL)` and return the result unchanged.

### 6. Side Effects

- Opens and closes a SQLite connection (via `Storage` inside `_with_network`).
- May populate or reuse `_network_cache[db_path]` (keyed by path + mtime).
- May **download and load** a sentence-transformer model on first use (network I/O to HuggingFace, memory cost, wall-clock latency).
- No mutations to the DB — `write=False` means `Storage.save` is skipped.

### 7. Error Handling

- **No try/except**: all exceptions propagate.
- `FileNotFoundError` / `sqlite3.OperationalError` if the DB is missing or corrupt.
- `ImportError` if `sentence-transformers` isn't installed (raised from inside `reasons.cluster`).
- Network/model-download errors bubble out.
- A file modified between the pre-call mtime check and the load is *not* specifically guarded — that's the same race any read path has via `_with_network`.

### 8. Usage Patterns

- Called from `build_wiki(cluster=True, ...)`, which passes `status=status or ""` — a deliberate trick to flip `None` into "cluster everything".
- Likely exposed via CLI (`reasons clusters`) with defaults.
- **Caller obligations:**
  - Ensure the DB exists.
  - Expect high latency on the first call (model load).
  - For determinism, provide `seed`.
  - For consistent output across runs against an evolving DB, be aware that clustering is over *all* filtered beliefs, so adding/removing any node can reshuffle cluster membership.

### 9. Dependencies

- **Internal:** `reasons.cluster.list_clusters`, `reasons.cluster.DEFAULT_MODEL`, `reasons.storage.Storage` (via `_with_network`), `reasons.network.Network`, `_is_visible` helper, `_network_cache`.
- **External:** SQLite (`sqlite3`), and transitively `sentence-transformers` (via `reasons.cluster`).

### Assumptions not enforced by the type system

- `status=""` silently means "all states" — an empty string is not documented in the signature, only in the truthiness check.
- Downstream return shape (`clusters`, and each cluster having `id` + `beliefs[].id`) is assumed by callers but not typed.
- `n_clusters` is not validated against `len(beliefs)` — behavior when it exceeds the corpus size is delegated downstream (and would be worth checking).
- The empty-result branch returns `embedding_model` as the resolved default so downstream code can still report which model *would* have been used.

## Topics to Explore

- [file] `reasons/cluster.py` — The actual embedding + clustering implementation; defines `DEFAULT_MODEL` and the response shape callers depend on
- [function] `reasons/api.py:_is_visible` — Access-tag semantics used to filter beliefs before clustering
- [function] `reasons/api.py:build_wiki` — Primary internal caller; shows how cluster labels are derived and how `status or ""` is used
- [function] `reasons/api.py:_with_network` — Cache + read/write lifecycle that governs whether a call reuses the in-memory network
- [general] `network-cache-invalidation` — mtime-based caching implications when clustering after writes

## Beliefs

- `list-clusters-delegates-to-cluster-module` — The API wrapper forwards all filtering results to `reasons.cluster.list_clusters`; it never clusters inline
- `status-empty-string-disables-truth-filter` — Passing `status=""` (or `None`) is equivalent to "include IN and OUT nodes" because the guard is `if status and ...`
- `list-clusters-is-read-only` — The function enters `_with_network(db_path)` without `write=True`, so no `Storage.save` occurs
- `list-clusters-short-circuits-empty-set` — When no nodes survive filters, it returns `{"clusters": [], "n_clusters": 0, "embedding_model": ...}` without invoking the embedding model
- `visible-to-requires-subset-tags` — A node is clustered only if all of its `access_tags` are in `visible_to`, matching `_is_visible`'s subset rule
