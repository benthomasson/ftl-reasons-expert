---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: build_wiki in reasons/api.py

# `build_wiki` — Interlinked Belief Wiki Generator

## Purpose

`build_wiki` is the functional API wrapper that turns the current belief
network into a directory of interlinked Markdown wiki pages. It exists so
that callers (CLI, LangGraph tools, scripts) can invoke wiki generation
without managing `Storage` lifecycles: it fetches nodes, enriches them
with full details, buckets them into topic or cluster groups, and hands
everything off to the lower-level `reasons.build_wiki.build_wiki` module
that actually writes the pages. It reads from the database but never
mutates it.

## Contract

**Preconditions**
- `db_path` points to a valid SQLite RMS database (no `pg_conninfo`
  parameter — this function does **not** support the PostgreSQL backend,
  unlike many sibling functions in the module).
- `output_dir`, if it exists, is writable. If not, its parents must be
  creatable or the caller must tolerate an empty-wiki failure.
- When `cluster=True`, the clustering dependencies (embeddings model if
  `embedding_model` is set, or the default sentence-transformers model)
  must be installed.

**Postconditions**
- Directory `output_dir` exists and contains at least `index.md`.
  Additional pages are written by `_build_wiki`.
- In the empty case, `index.md` contains the literal
  `# Belief Wiki\n\n*No beliefs found.*\n` and nothing else.
- The database is not modified.

**Invariants assumed but not enforced**
- Every node id appearing in `groups` should also be present as a key in
  `node_details` — the wrapper tries to guarantee this but silently
  swallows `show_node` failures, so a node id may be in `groups` but not
  in `node_details`.
- `_assign_topics` and `_build_wiki` accept the exact argument shapes
  used here. Their contracts are documented only in
  `reasons/build_wiki.py`.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `output_dir` | `str` | Directory for generated pages; created if missing. Default `"wiki"`. | Relative paths resolve against CWD. If `_build_wiki` writes additional files, they live here too. |
| `status` | `str \| None` | Truth-value filter (`"IN"`, `"OUT"`, etc.). `None` = no filter. Propagated to `list_nodes` and (via `status or ""`) to `list_clusters`. **Not** propagated to `topics()`. | `""` and `None` are equivalent for the cluster path. |
| `max_topics` | `int` | Cap on the number of topics extracted by `topics()`. Default 20. | `0` or negative → `topics()` returns no topics, so `_assign_topics` buckets everything into its fallback. |
| `cluster` | `bool` | If true, group by semantic clusters (`list_clusters`) instead of topics. | Mutually exclusive with the topic path. `n_clusters`, `seed`, `embedding_model` only take effect when `cluster=True`. |
| `n_clusters` | `int \| None` | Number of clusters requested (k-means style). Default `None` = library default. | Only used when `cluster=True`. |
| `seed` | `int \| None` | Random seed for reproducible clustering. | Only used when `cluster=True`. |
| `embedding_model` | `str \| None` | Sentence-transformers model name for clustering. | Only used when `cluster=True`. |
| `visible_to` | `list[str] \| None` | Access-tag subset filter. Applies to `list_nodes`, `show_node`, and (on the cluster path) `list_clusters`. **Not** applied to `topics()`. | Passed as `None` means "no visibility restriction". Nodes whose `access_tags` are not a subset are excluded. |
| `model` | `str` | LLM model identifier forwarded to `_build_wiki` (page summarization / content generation). Default `""` = library default. | Never used by this function directly; only forwarded. |
| `timeout` | `int` | LLM timeout in seconds. Default 300. | Only meaningful if `_build_wiki` invokes the LLM. |
| `parallel` | `int` | Worker/parallelism hint forwarded to `_build_wiki`. Default 0 = library default. | Forwarded only. |
| `db_path` | `str` | SQLite database path. Default `DEFAULT_DB = "reasons.db"`. | No `pg_conninfo` support. |

## Return Value

```python
{"output_dir": str, "pages": int, "total_nodes": int}
```

- Returned verbatim from `_build_wiki` on the success path — the wrapper
  does not add or reshape keys. Callers must trust the shape of
  `_build_wiki`'s own return.
- Empty case: `{"output_dir": output_dir, "pages": 0, "total_nodes": 0}`.

No exception is returned in-band; errors are raised.

## Algorithm

1. **Import lazily** — `_assign_topics` and `build_wiki` from
   `.build_wiki` are imported inside the function to avoid pulling in
   LLM/embedding dependencies at module import time.

2. **Fetch candidate nodes** —
   `list_nodes(status=status, visible_to=visible_to, db_path=db_path)`.
   Note this only forwards `status` and `visible_to`; all other `list_nodes`
   filters (`namespace`, `tag`, `limit`, etc.) are unavailable here.
   `list_nodes` returns `{"nodes": [...], "count": int}`, and only the
   `id` values are kept in `node_ids`.

3. **Empty short-circuit** —
   If `node_ids` is empty, create `output_dir`, write a single
   `index.md` placeholder, and return `pages=0`. This guarantees the
   caller always gets a valid `output_dir` back regardless of whether
   any nodes exist.

4. **Detail enrichment** —
   For every node id, call `show_node(nid, visible_to=visible_to,
   db_path=db_path)` and store its full detail dict in `node_details`.
   `KeyError` and `PermissionError` are caught and ignored — a
   filtered/vanished node simply has no detail entry but may still be
   referenced from `groups`. This is the main "silent narrowing" point.

5. **Assign groups** — two mutually exclusive branches:

   - **`cluster=True`**
     - Call `list_clusters(status=status or "", n_clusters=…, seed=…,
       embedding_model=…, visible_to=visible_to, db_path=db_path)`.
       Because `list_clusters` treats `""` as "no filter", `status=None`
       and `status=""` here both mean "cluster all nodes" — consistent
       with `list_nodes(status=None)` returning all nodes.
     - For each cluster, collect member ids.
     - Derive a human-readable **page label** by splitting every member
       id on `[-._:]`, dropping fragments with `len <= 2` or in
       `_TOPIC_STOP_WORDS`, and picking the most frequent surviving
       fragment. Fallback is `f"cluster-{c['id']}"`.
     - Collision handling: if the derived label already exists in
       `groups`, append `-{c['id']}` so two clusters sharing a
       token don't collide.
     - `groups[label] = [ids]`.

   - **`cluster=False`**
     - Call `topics(limit=max_topics, db_path=db_path)` to get the top-N
       word-frequency topics.
     - **Caveat**: `topics()` is called with no `status` or `visible_to`
       arguments, so it derives topic vocabulary from **all nodes in the
       network**, including OUT nodes and nodes the caller shouldn't see.
       The node ids that get grouped, however, still come from the
       filtered `node_ids`, so unfiltered topics can only affect *labels*
       (and pages that end up empty), not membership.
     - `_assign_topics(node_ids, topics_result["topics"])` returns a
       `dict[group_label, list[node_id]]`. Nodes whose ids don't match
       any topic are handled by `_assign_topics` (typically an
       "other"/"uncategorized" bucket — see `reasons/build_wiki.py`).

6. **Delegate page writing** — call `_build_wiki(node_details, groups,
   output_dir, model=model, timeout=timeout, parallel=parallel)` and
   return its result dict unchanged.

## Side Effects

- **Filesystem writes**: `os.makedirs(output_dir, exist_ok=True)` and
  `index.md` in the empty case; `_build_wiki` writes all other pages
  (index + per-topic / per-cluster pages + belief stubs, presumably).
- **Read-only DB access**: this wrapper never passes `write=True` into
  `_with_network`; `show_node`, `list_nodes`, `topics`, and
  `list_clusters` all open read-only network views (cached when mtime
  matches).
- **Potential LLM/embedding calls**: `_build_wiki` receives `model`,
  `timeout`, and `parallel`, which implies per-page LLM summarization.
  `list_clusters` loads an embedding model if `embedding_model` is set.
- **Network cache**: `_with_network` may populate `_network_cache` on
  the read path; no invalidation is performed here.

## Error Handling

- **Swallowed**: `KeyError` and `PermissionError` from `show_node`. A
  node that has been deleted between `list_nodes` and `show_node`, or
  that fails the access-tag check despite `visible_to` filtering in
  `list_nodes`, is silently dropped from `node_details` while possibly
  still appearing in `groups`.
- **Propagated**:
  - `FileNotFoundError`/`sqlite3.DatabaseError` from opening a missing
    database (via `_with_network` / `Storage`).
  - `OSError` from `os.makedirs` / `open` in the empty case.
  - Any exception raised by `list_clusters` (missing embeddings
    dependency, network error, etc.).
  - Any exception raised by `_build_wiki` (LLM timeout, write failure,
    missing templates, unexpected group membership).
  - `ValueError` etc. from `_assign_topics` if its contract is violated.
- **Not caught**: the function has no try/except around the heavy
  branch. Partial output may be left on disk if `_build_wiki` fails
  mid-write.

## Usage Patterns

- **Typical CLI**: `reasons build-wiki --status IN --cluster
  --n-clusters 8 --model claude --parallel 4`. The wrapper is the
  SQLite-only convenience entry point.
- **Programmatic**: called from scripts to produce a static site for
  review or publishing; the caller typically just consumes
  `result["pages"]` for logging and trusts `result["output_dir"]`.
- **Caller obligations**:
  - Ensure `output_dir` is safe to overwrite — this function recreates
    `index.md` unconditionally in the empty case and lets `_build_wiki`
    write into the same directory on the non-empty path, so stale pages
    from a previous run may persist.
  - Do **not** call this if the caller needs PostgreSQL support —
    there is no `pg_conninfo` dispatch here.
  - If access separation matters, know that topic labels are derived
    from the whole network, while membership respects `visible_to`.
    For strict access separation, prefer `cluster=True`, which applies
    `visible_to` to both membership and cluster discovery.

## Dependencies

- `reasons.build_wiki.build_wiki` — actual page writer; consumes a
  `dict[str, node_detail]`, a `dict[str, list[str]]` of groups, and the
  output directory.
- `reasons.build_wiki._assign_topics` — buckets filtered node ids into
  topic groups.
- `reasons.api.list_nodes` — filtered node enumeration (used for both
  the node list and, indirectly via `show_node`, detail).
- `reasons.api.show_node` — per-node detail extraction.
- `reasons.api.topics` — word-frequency topic extraction from node ids.
- `reasons.api.list_clusters` — semantic k-means clustering with
  visibility filtering.
- `reasons.api._TOPIC_STOP_WORDS` — shared stopword set used when
  deriving cluster page labels.
- `reasons.api._with_network` / `Storage` (transitively) — read-only
  network handle.

## Topics to Explore

- [file] `reasons/build_wiki.py` — The page-template layer; explains exactly what `_build_wiki` writes and how `model`/`timeout`/`parallel` shape page content.
- [function] `reasons/build_wiki.py:_assign_topics` — The bucketing rule that decides whether an unmatched node ends up under "Other" or is dropped; critical for understanding surprising page groupings.
- [function] `reasons/api.py:topics` — The topic extractor that is called *without* `status`/`visible_to`; explains why topic vocabulary can leak information from OUT or gated nodes.
- [function] `reasons/api.py:list_clusters` — The clustering helper used when `cluster=True`; its intersection with `visible_to` is what makes the cluster path access-safe.
- [function] `reasons/api.py:list_nodes` — The upstream filter; documents what filters `build_wiki` could expose but currently ignores (`namespace`, `tag`, `by_impact`, etc.).

## Beliefs

- `build-wiki-does-not-support-postgresql` — `build_wiki` has no `pg_conninfo`/`domain_id` parameters, unlike most sibling functions in `reasons/api.py`, so it is SQLite-only.
- `build-wiki-topic-labels-bypass-visibility-filter` — The `cluster=False` branch calls `topics(limit=max_topics, db_path=db_path)` without `visible_to` or `status`, so topic vocabulary is derived from the entire network even when the requesting caller has restricted access tags.
- `build-wiki-empty-network-produces-placeholder-index` — When the filtered node set is empty, `build_wiki` creates `output_dir`, writes a single `index.md` containing `*No beliefs found.*`, and returns `{"pages": 0, "total_nodes": 0}` without invoking `_build_wiki`.
- `build-wiki-swallows-node-detail-errors` — `KeyError` and `PermissionError` from `show_node` are caught and ignored, meaning `node_details` can be a strict subset of `groups`' referenced ids.
- `build-wiki-cluster-labels-derived-from-id-token-frequency` — Cluster page labels are chosen as the most frequent non-stopword token after splitting member ids on `[-._:]`, with a `cluster-{id}` fallback and `-{id}` collision suffixing.
