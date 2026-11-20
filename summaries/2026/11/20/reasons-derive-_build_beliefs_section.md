---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _build_beliefs_section in reasons/derive.py

# `_build_beliefs_section` — Explanation

## Purpose

This function renders the **"Existing Beliefs"** block that gets injected into the derive prompt (`DERIVE_PROMPT.format(..., beliefs_section=...)`). It takes the full network and produces a compact, token-budgeted, markdown-formatted list of *base* beliefs (IN, non-derived nodes) so that an LLM can pick antecedents when proposing new derivations.

It is a **pure builder**: it does not mutate inputs, does not hit the DB, and does not call the LLM. It selects, groups, sorts, truncates/samples, and formats — then returns the text and (optionally) clustering metadata.

Because there are four different display strategies (plain truncation, random sampling, semantic clustering, intra-cluster focus) × two grouping modes (agents vs. code-prefix), this is essentially a **rendering dispatcher** with per-mode budget logic.

## Contract

**Preconditions / assumptions (not type-enforced):**
- `nodes` is a dict `{node_id: {"truth_value": "IN"|"OUT", "text": str, ...}}`. Every node that passes the IN filter is expected to have a `text` key; otherwise `v["text"]` raises `KeyError`.
- `derived` is a subset of `nodes.keys()`. The function only uses it for membership tests (`k not in derived`).
- `agents`, when truthy, is `{agent_name: [node_ids]}` produced by `_detect_agents`. Agent namespaces are encoded in node IDs as `"<agent>:<belief-id>"`.
- Non-agent belief IDs may contain hyphens; the "prefix" is defined as `k.split("-")[0]` (or the whole id if no hyphen). This is a **heuristic**, not a schema.
- `seed` is meaningful only when sampling/clustering is requested. If `sample=False` and no clustering, `seed` is ignored (rng is `None`).

**Postconditions:**
- Returns `(section_text, cluster_stats)`.
  - `section_text` is a string. It is `""` (not `None`) when nothing qualifies.
  - `cluster_stats` is a dict **if and only if** `cluster or intra_cluster` was truthy. Otherwise it is `None`. So the second return value is a reliable mode signal.
- When clustering: `selected_ids` is assumed to be at most `max_beliefs` items; **no further truncation** is applied at render time.
- When not clustering: truncation/sampling is applied, but budget can **exceed `max_beliefs`** because per-group budgets use `max(5, ...)` (see Algorithm).

## Parameters

| Param | Meaning | Edge cases |
|---|---|---|
| `nodes` | Full network dict | Missing `truth_value` → treated as OUT (not IN). Missing `text` for displayed node → `KeyError`. |
| `derived` | Set/dict of derived node IDs | Only membership-tested. Keys not in `nodes` are harmless. |
| `agents` | `{name: [ids]}` from `_detect_agents` | Empty `{}` is falsy → falls through to the code-prefix path. |
| `max_beliefs` | Soft budget | Exact-capped in the "no agents, no cluster" path; can be exceeded elsewhere via `max(5, ...)` floors. |
| `sample` | Use random sampling vs. alphabetical truncation | Unused when `cluster or intra_cluster` is set. |
| `seed` | Seed for reproducibility | `sample=True, seed=None` → non-deterministic (system entropy). |
| `cluster` | Turn on semantic cross-cluster sampling | Lazily imports `reasons.cluster.cluster_beliefs`. |
| `intra_cluster` | Focus budget on **one** cluster per round | Takes precedence over `cluster`; calls `cluster_beliefs_intra`. |
| `round_num` | Rotation index for intra-cluster | Only used by `cluster_beliefs_intra`. |
| `cluster_cache` | Optional embedding cache | Only used by clustering path. |
| `embedding_model` | Model name | Defaults to `"all-MiniLM-L6-v2"` when falsy. |
| `n_clusters` | Override auto cluster count | Only used by clustering path. |

## Return Value

`(section_text: str, cluster_stats: dict | None)`

- `section_text` is a `"\n".join(lines)` — a sequence of markdown `### <group>` headers each followed by `- \`<id>\`: <text[:120]>` bullets.
- In the plain "no cluster" truncation path (no agents, no sample), headers **omit** `showing N`; other paths include it.
- `cluster_stats` is forwarded verbatim from `cluster_beliefs[_intra]` when clustering was requested; the caller (`build_prompt`) uses it to populate `stats["cluster"]`, `stats["n_clusters"]`, etc.

The caller must tolerate an **empty string** beliefs section (e.g., when `in_nodes` is empty or all selections happen to be empty).

## Algorithm

### Phase 1 — Filter to base IN beliefs
```python
in_nodes = {k: v for k, v in nodes.items()
            if v.get("truth_value") == "IN" and k not in derived}
```
**Derived beliefs are excluded** from this section — they are rendered separately by `_build_derived_section`. Missing `truth_value` is treated as not IN.

### Phase 2 — Optional clustering
- If `intra_cluster`: lazy import `cluster_beliefs_intra`; build `belief_texts = {k: v["text"] ...}`; call with `round_num`, `seed`, `n_clusters`, `cache`, `model_name`.
- Elif `cluster`: lazy import `cluster_beliefs`; same idea without `round_num`.
- Both return `(selected_ids, cluster_stats)`. `selected_ids` is treated as a collection of IDs (uniqueness assumed).

### Phase 3A — Render cluster results (early return)
If `intra_cluster or cluster`:
- **With agents:**
  - Sort agents by descending `len(agents[a])` (⚠️ total node count, not the IN-base count that's actually displayed — see "Gotchas").
  - For each agent, filter `selected_ids` by `k.startswith(f"{agent}:")`, sort, skip if empty.
  - Header: `### Agent: <name> (<agent_total> beliefs, showing <len>)` where `agent_total` counts **IN base** beliefs.
  - **Local beliefs** are the selected IDs with no `:` — rendered under `### Local beliefs (...)`.
- **Without agents:**
  - Build two groupings keyed by `k.split("-")[0]`: `groups` (only selected IDs) and `all_groups` (all `in_nodes`).
  - Sort prefixes by descending total size from `all_groups`.
  - Header: `### <prefix> (<total> beliefs, showing <selected count>)`.
  - Return `("\n".join(lines), cluster_stats)`.

### Phase 3B — No-cluster, with agents
- Compute `total_agent_beliefs` (IN base, per agent namespace) + `non_agent` (IN base, no colon) → `total_all`.
- Iterate agents by descending `len(agents[a])`:
  - `agent_beliefs` = IN base filtered by prefix.
  - `agent_budget = max(5, int(max_beliefs * len(agent_beliefs) / total_all))`.
  - If `sample`: `_sample_beliefs(ids, budget, rng)` then re-sort alphabetically. Else: `ids[:budget]` (alphabetical).
  - Track `count += len(belief_ids)`.
- **Local beliefs** after agents: `remaining = max(5, max_beliefs - count)`; same sample/truncate logic.

### Phase 3C — No-cluster, no agents
Group `in_nodes` by `k.split("-")[0]`.

- **`sample=True`:**
  - Flatten all `(id, text)` into one list; call `_sample_beliefs(all_items, max_beliefs, rng)` (note: it receives tuples, then the ID is extracted via `set(k for k, _ in ...)`).
  - Regroup into `prefix_items` filtered to sampled keys; render each prefix with `showing N`. `count` is incremented but **never read** in this branch (dead variable).
- **`sample=False`:**
  - Iterate prefixes by descending group size; outer `break` when `count >= max_beliefs`; inner `break` mid-prefix when budget reached.
  - Header omits `showing N` — only `(N beliefs)`.

### Phase 4 — Return
```python
return "\n".join(lines), None
```

## Side Effects

- **Reads** `nodes` and `derived`. No mutations of inputs (`in_nodes`, `groups`, `all_groups`, `belief_texts` are all new objects).
- **Mutates** the local `random.Random` state (`rng`), which is deterministic per `seed`. When `sample=True` and `agents` are present, **all agents share one `rng`** sequentially, so a given agent's sample depends on prior agent calls' rng state. Reproducibility holds given a seed, but samples aren't independent of agent iteration order.
- **Lazy imports** `reasons.cluster` only when clustering is requested — this avoids pulling in `sentence-transformers` for the common path (and avoids a potential circular import at module load time).
- No I/O, no logging, no DB writes.

## Error Handling

The function is **fail-loud** — no `try/except` inside:
- `from .cluster import cluster_beliefs` → `ImportError` if `reasons/cluster.py` requires missing optional deps (e.g., `sentence-transformers`). Propagates.
- `v["text"]` / `agent_beliefs[id]["text"]` → `KeyError` if a node lacks `text`. This is a hard precondition.
- `cluster_beliefs_*` may raise (e.g., on empty `belief_texts`, or on model load failure); propagates.
- `_sample_beliefs` with `budget < 0` would raise `ValueError` from `random.sample`, but the `max(5, ...)` floors prevent that in the paths where it's reachable.

The caller `build_prompt` does **not** wrap this call — the only `try/except` in `build_prompt` is around `template.format(...)`.

## Usage Patterns

Called once, from `build_prompt`:

```python
beliefs_section, cluster_stats = _build_beliefs_section(
    nodes, derived, agents, max_beliefs=budget,
    sample=sample, seed=seed,
    cluster=cluster, intra_cluster=intra_cluster,
    round_num=round_num, cluster_cache=cluster_cache,
    embedding_model=embedding_model, n_clusters=n_clusters,
)
```

Caller obligations:
- Must pass `agents=_detect_agents(nodes)` (or `{}`) — the function does **not** re-derive agents.
- Must pass `derived` computed against the **same filtered** `nodes` (in `build_prompt`, `derived` is recomputed after topic/depth filters, so `in_nodes` and `derived` are consistent).
- Must handle `cluster_stats is None` (it will be, in the non-cluster case).
- Must handle empty `beliefs_section` (e.g., all beliefs are OUT, or topic filter matched nothing).

## Dependencies

- **Stdlib:** `random` (for `random.Random`), `collections.defaultdict` (already imported at module top).
- **Internal:** `_sample_beliefs` (same module). Lazy: `reasons/cluster.py::cluster_beliefs` and `reasons/cluster.py::cluster_beliefs_intra`.
- **External (optional, only via clustering):** `sentence-transformers` (implied by `embedding_model`), `numpy`/`scikit-learn` (implied by clustering generally).

## Gotchas / Observations

1. **Budget is soft, not hard.** In the agent branch, `max(5, int(max_beliefs * share))` per agent plus `max(5, max_beliefs - count)` for local means the aggregate output can exceed `max_beliefs` by up to roughly `5 * (num_agents + 1)`. The "budget" in the docstring is really a *target*, not a cap.
2. **Sort key vs. display count mismatch.** Agents are ordered by `-len(agents[a])` (all nodes: IN, OUT, derived), but headers and budget math use `in_nodes` counts (IN, non-derived). An agent with many OUT or derived beliefs can sort first while displaying fewer than the next agent.
3. **`sample` is ignored under clustering.** `cluster` / `intra_cluster` short-circuit before any `sample` logic; sampling is delegated to the cluster module via `seed`.
4. **Prefix grouping is `k.split("-")[0]`.** `bug-in-auth` and `bug-in-login` collapse to `bug`; a hyphen-free `standalone` becomes its own singleton group. This is a heuristic that can create surprising buckets.
5. **`count` in the sample/no-agent branch is dead code** — incremented but never read.
6. **`_sample_beliefs` uses `rng.sample`, not reservoir sampling** despite its docstring. Because `all_items` in the sample/no-agent branch is a list of `(id, text)` tuples, the function actually samples tuples, and the caller extracts IDs — this works, but is a subtle coupling.
7. **Cluster renders assume `selected_ids` is already budget-limited** — the render does no further truncation. If `cluster_beliefs` ever returned more than `max_beliefs`, the budget would be silently blown.

## Topics to Explore

- [file] `reasons/cluster.py` — Defines `cluster_beliefs` and `cluster_beliefs_intra`; determines the real budget behavior and embedding-cache contract that Phase 3A relies on.
- [function] `reasons/derive.py:_sample_beliefs` — The shared sampling helper; verify its `rng.sample` behavior and the docstring/implementation mismatch noted above.
- [function] `reasons/derive.py:_detect_agents` — Constructs the `agents` dict consumed here; explains why the sort key (`len(agents[a])`) diverges from the display count (`in_nodes`).
- [function] `reasons/derive.py:build_prompt` — The sole caller; shows how `cluster_stats` is folded into `stats` and how empty/partitioned inputs are handled.
- [file] `tests/test_derive_budget.py` — Likely the authoritative spec for the budget floors and overflow behavior discussed above.

## Beliefs

- `in-nodes-exclude-derived` — `_build_beliefs_section` only includes base beliefs: nodes with `truth_value == "IN"` whose ID is **not** in `derived`.
- `budget-floor-can-exceed-max` — Per-agent and local budgets use `max(5, ...)`, so the total displayed beliefs can exceed `max_beliefs` by up to ~5 per agent plus 5 for locals.
- `cluster-mode-ignores-sample-flag` — When `cluster` or `intra_cluster` is truthy, the `sample` parameter is not consulted; sampling is handled inside `reasons.cluster`.
- `cluster-stats-presence-signals-cluster-mode` — The second return value is a dict iff clustering ran (early-return path); otherwise it is `None`, even when the section is non-empty.
- `agent-sort-uses-total-node-count` — Agent ordering sorts by `len(agents[a])` (all namespaced nodes, including OUT and derived), while headers and budget math use the filtered `in_nodes` counts.
