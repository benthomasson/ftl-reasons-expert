---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: build_prompt in reasons/derive.py

# `build_prompt` — `reasons/derive.py`

## Purpose

`build_prompt` assembles the complete LLM prompt used by the derivation workflow. It takes a belief-network node mapping, applies topic/structural/depth filters, computes graph statistics (depth, agents, counts), builds the “existing beliefs” and “existing derived conclusions” sections, formats everything into either the default `DERIVE_PROMPT` or a caller-supplied template, and returns the prompt text plus a stats dictionary.

It exists so the derive command can produce a bounded, filtered, contextualized prompt from the full network without duplicating filtering, sampling, clustering, agent-detection, and formatting logic.

---

## Contract

### Preconditions

- `nodes` is a mapping of `node_id -> node_data`.
- Each node data value is expected to behave like a dict and may contain:
  - `text`: string used in prompt sections.
  - `truth_value`: typically `"IN"` or `"OUT"`.
  - `justifications`: list of justification dicts.
  - `supporting_justification`: optional index into `justifications`.
- Each justification is expected to be dict-like and may contain:
  - `antecedents`: list of node IDs.
  - `outlist`: list of node IDs.
- If `prompt_template` is provided, it must be compatible with Python `str.format` and may only reference the placeholders supplied by this function.

### Postconditions

- Returns `(prompt_text, stats_dict)`.
- The returned prompt is formatted with the filtered graph’s data.
- `stats_dict` describes the filtered graph, not necessarily the original full graph.
- The input `nodes` dict is not mutated by this function.
- `cluster_cache`, if supplied and clustering is enabled, may be populated or reused by the clustering module.

### Important invariants and assumptions

- Topic filtering happens first, before depth, dependency, and agent calculations.
- Depth is computed by `_get_depth`, which depends on membership in the `derived` mapping passed to it. Its `nodes` argument is effectively unused.
- `derived` nodes are nodes with non-empty `justifications`, regardless of `truth_value`. So OUT derived nodes can appear in the derived section and in `total_derived`.
- `in_nodes` counts all nodes with `truth_value == "IN"`, including IN derived nodes.
- The beliefs section only lists non-derived IN nodes.
- The budget is not a strict total prompt limit: `_build_beliefs_section` uses `budget` for beliefs, and `_build_derived_section` separately uses `budget` for derived conclusions. In agent mode, the beliefs section can exceed `budget` because of per-agent minimum allocations.

---

## Parameters

| Parameter | Meaning | Edge cases / notes |
|---|---|---|
| `nodes` | Mapping of `node_id -> node_data` from `export_network`. | Missing `text` can raise `KeyError` in section builders. Malformed `justifications` entries can raise `AttributeError`. |
| `domain` | Optional domain description inserted into prompt context. | If provided, it takes precedence over agent-based context text. |
| `topic` | Optional case-insensitive keyword filter. Splits on whitespace and matches any keyword against node ID and text. | Applied before all other filtering. Whitespace-only `topic` is truthy but produces no keywords, so all nodes are filtered out. |
| `budget` | Maximum beliefs for the beliefs section and maximum derived conclusions for the derived section. | Not strictly enforced in agent mode. Negative values are not validated and can cause slicing or sampling problems. |
| `sample` | If true, randomly sample beliefs instead of alphabetical truncation. | Ignored when clustering is enabled. Uses `seed` for reproducibility. |
| `seed` | Random seed for sampling and clustering. | `None` means non-deterministic randomness. |
| `min_depth` | Only include nodes at this depth or deeper. | Triggers depth recomputation and filtering. Negative values are not validated. |
| `max_depth_filter` | Only include nodes at this depth or shallower. | Triggers depth recomputation and filtering. Negative values can filter everything. |
| `premises_only` | If true, keep only nodes without non-empty `justifications`. | Treats empty/`None` justifications as premises. Also causes `derived` to become empty. |
| `has_dependents` | If true, keep only nodes referenced by at least one antecedent or outlist entry. | Uses references from the topic-filtered graph before this filter is applied. |
| `cluster` | If true, use semantic clustering to select beliefs across domains. | Lazily imports `reasons.cluster` and may load embeddings/model. |
| `intra_cluster` | If true, focus the budget on one cluster per round. | Takes precedence over `cluster` inside `_build_beliefs_section`. |
| `round_num` | Round number used by intra-cluster rotation. | Only meaningful with `intra_cluster=True`. |
| `cluster_cache` | Optional cache for embedding reuse across rounds. | May be mutated by clustering code. |
| `embedding_model` | Sentence-transformers model name for clustering. | Defaults to `"all-MiniLM-L6-v2"` inside `_build_beliefs_section`. |
| `n_clusters` | Override automatic cluster count. | Passed to clustering module. |
| `prompt_template` | Optional custom prompt template. | Must use supported placeholders and escape literal braces. Unknown placeholders raise `ValueError`. |

---

## Return Value

Returns a two-element tuple:

```python
(prompt_text, stats_dict)
```

`prompt_text` is the fully formatted prompt string.

`stats_dict` always contains:

- `total_in`: count of filtered nodes with `truth_value == "IN"`, including IN derived nodes.
- `total_derived`: count of filtered nodes with non-empty justifications.
- `max_depth`: maximum depth computed according to filtering path.
- `agents`: number of detected agent namespaces.
- `agent_names`: sorted agent names, or `[]`.
- `budget`: the supplied budget.
- `sample`: the supplied sampling flag.

It conditionally contains:

- `topic`, if `topic` is truthy.
- `min_depth`, if not `None`.
- `max_depth_filter`, if not `None`.
- Cluster-related keys if clustering was requested and clustering returned non-empty stats:
  - `cluster`
  - `intra_cluster`
  - `n_clusters`
  - `cluster_sizes`
  - `embedding_model`
  - optional `focus_cluster`

Callers must handle the fact that `stats_dict` describes the filtered graph, not the original full graph.

---

## Algorithm

1. **Apply topic filter first.**  
   If `topic` is truthy, replace `nodes` with `_filter_by_topic(nodes, topic)`. That function lowercases node ID and text, splits the topic on whitespace, and keeps nodes where any keyword appears. This is an OR match, not phrase matching.

2. **Compute pre-filter depth and dependency information.**  
   - `all_derived` is every current node with non-empty `justifications`.
   - `max_depth` is the maximum `_get_depth` over `all_derived`, defaulting to `0`.
   - `referenced` is the set of all node IDs appearing in any `antecedents` or `outlist` of any current node’s justifications.

3. **Apply structural and depth filters.**  
   If any of `premises_only`, `has_dependents`, or depth bounds are active:
   - Skip nodes with non-empty justifications when `premises_only=True`.
   - Skip nodes not in `referenced` when `has_dependents=True`.
   - Compute each node’s depth using `_get_depth(k, nodes, all_derived, memo)` and skip nodes outside `min_depth` / `max_depth_filter`.
   - Replace `nodes` with the filtered dict.

4. **Recompute derived and IN sets from filtered nodes.**  
   - `derived = {k: v for k, v in nodes.items() if v.get("justifications")}`.
   - `in_nodes = {k: v for k, v in nodes.items() if v.get("truth_value") == "IN"}`.

5. **Recompute `max_depth` when structural/depth filters were used.**  
   If any of `premises_only`, `has_dependents`, or depth bounds were active, recompute `max_depth` as the maximum `_get_depth` over filtered `derived`, using the shared `memo` and original `all_derived`. If `premises_only=True`, `derived` is empty and `max_depth` becomes `0`.

6. **Detect agents.**  
   `_detect_agents(nodes)` treats any node ID containing `:` as agent-namespaced, splitting at the first colon and skipping IDs ending in `:active`. It returns `{agent_name: [node_ids]}`.

7. **Build context strings.**  
   - If `domain` is provided, use `"The beliefs in this network are about: {domain}"`.
   - Else if agents were detected, list agent names and explain multi-agent context.
   - Else empty context.
   - If `topic` was provided, append a “Filtered to beliefs matching: …” note.
   - Set `cross_agent_task = CROSS_AGENT_TASK` if agents were detected, else empty string.
   - Build `agents_stats` with total agent count and per-agent node counts.

8. **Build belief and derived sections.**  
   - `_build_beliefs_section(...)` receives filtered nodes, derived IDs, agent info, budget, sampling, clustering options. It only lists IN non-derived beliefs and groups them by agent or ID prefix. It returns section text and optional cluster stats.
   - `_build_derived_section(...)` lists up to `budget` derived nodes sorted by descending depth. It shows status, depth, ID, text, first justification’s antecedents, and optional unless list. It includes OUT derived nodes too.

9. **Format the template.**  
   Choose `prompt_template` if provided, otherwise `DERIVE_PROMPT`. Call `.format(...)` with:
   - `domain_context`
   - `beliefs_section`
   - `derived_section`
   - `total_in`
   - `total_derived`
   - `max_depth`
   - `cross_agent_task`
   - `agents_stats`

10. **Build and return stats.**  
    Populate the base stats, conditionally add topic/depth/cluster metadata, and return `(prompt, stats)`.

---

## Side Effects

- No direct file or network I/O in `build_prompt` itself.
- It does not mutate the input `nodes` mapping; it reassigns local `nodes` to new filtered dicts.
- If clustering is enabled, `_build_beliefs_section` lazily imports `reasons.cluster` and may:
  - load a sentence-transformers embedding model,
  - populate or reuse `cluster_cache`,
  - perform CPU/GPU work,
  - raise import or model-loading errors.
- Randomness is local via `random.Random(seed)`; global random state is not modified by `build_prompt` directly.

---

## Error Handling

- **Unknown placeholder in custom template:** `.format` raises `KeyError`, which is converted into `ValueError` with a message listing available placeholders.
- **Malformed braces in custom template:** `.format` raises `ValueError`, which is re-raised as a custom `ValueError` explaining brace escaping.
- **Malformed default template:** `ValueError` is not wrapped because `prompt_template` is falsy.
- **Other formatting errors:** Exceptions such as `AttributeError`, `IndexError`, or `TypeError` from unusual format expressions are not caught.
- **Missing node fields:** `_build_beliefs_section` and `_build_derived_section` assume `v["text"]` exists for nodes they include. Missing `text` raises `KeyError`.
- **Malformed justifications:** code assumes each justification is dict-like. A non-dict element causes `AttributeError`.
- **Clustering errors:** Import failures or embedding/model failures propagate to the caller.
- **Negative budget with sampling:** `_build_beliefs_section` may pass a negative sample size to `random.Random.sample`, which can raise `ValueError`.

---

## Usage Patterns

Typical callers are derive workflow commands or round loops that:

1. Export the network into a `nodes` dict.
2. Call `build_prompt` with a budget, optional topic/depth filters, sampling seed, and clustering options.
3. Send the returned prompt to an LLM.
4. Parse proposals with `parse_proposals`.
5. Validate and apply them.

Common patterns:

- Non-clustered single-round prompt: `build_prompt(nodes, domain=..., budget=300)`.
- Reproducible sampling: `build_prompt(nodes, sample=True, seed=42, budget=200)`.
- Multi-round clustering: pass `cluster=True` or `intra_cluster=True`, plus `round_num`, `cluster_cache`, `embedding_model`, and `n_clusters`.
- Custom prompt: pass `prompt_template` using only supported placeholders and escaped literal braces.

Caller obligations:

- Provide a node schema compatible with `_build_beliefs_section` and `_build_derived_section`.
- Do not assume the budget is a strict total token or belief limit.
- Handle the fact that stats reflect filtered results.
- Handle clustering dependencies if clustering is enabled.

---

## Dependencies

Internal helpers in `reasons/derive.py`:

- `_filter_by_topic`
- `_get_depth`
- `_detect_agents`
- `_build_beliefs_section`
- `_build_derived_section`
- `DERIVE_PROMPT`
- `CROSS_AGENT_TASK`

Optional external/internal clustering:

- `reasons.cluster.cluster_beliefs`
- `reasons.cluster.cluster_beliefs_intra`
- sentence-transformers model loading via the clustering module

Standard library:

- `random` for sampling.

---

## Additional Implementation Notes

- `_get_depth` ignores its `nodes` argument. Depth is determined entirely by whether an ID is present in the `derived` mapping passed to it, and by `supporting_justification` when present and valid.
- Because `_get_depth` uses `supporting_justification` if valid, depth can be based on a single selected justification rather than all justifications.
- `has_dependents` considers both `antecedents` and `outlist` references.
- In agent mode, `_build_beliefs_section` allocates at least 5 beliefs per agent with `max(5, int(max_beliefs * len(agent_beliefs) / total_all))`. This can make the total exceed `budget`.
- `_build_derived_section` displays only the first justification’s antecedents and unless list for each derived node.
- If clustering is requested, `sample` is effectively ignored for belief selection; clustering controls selection instead.
- `agents_stats` counts all detected non-`:active` agent-namespaced nodes in the filtered graph, not only IN beliefs.

---

## Topics to Explore

- [file] `reasons/cluster.py` — Implements semantic clustering and intra-cluster rotation used when `cluster` or `intra_cluster` is set.
- [function] `reasons/derive.py:_build_beliefs_section` — Contains budget allocation, sampling, agent grouping, and cluster integration.
- [function] `reasons/derive.py:_get_depth` — Defines depth semantics, supporting-justification behavior, and cycle guard logic.
- [file] `tests/test_derive_budget.py` — Exercises budget, sampling, and truncation behavior.
- [general] `custom-prompt-template-contract` — Explores supported placeholders and brace-escaping behavior for `prompt_template`.

## Beliefs

- `build-prompt-returns-prompt-and-stats` — `build_prompt` returns a `(prompt_text, stats_dict)` tuple and builds stats from the filtered graph.
- `topic-filter-runs-before-depth-and-dependency-calculations` — Topic filtering is applied before depth, dependency, and agent calculations.
- `derived-section-includes-out-nodes` — The derived section includes all filtered nodes with non-empty justifications, including OUT nodes, and labels their status.
- `agent-budget-allocation-can-exceed-budget` — In non-clustered agent mode, `_build_beliefs_section` allocates at least 5 beliefs per agent, so total beliefs can exceed `budget`.
- `custom-template-unknown-placeholder-raises-valueerror` — Custom prompt templates with unknown placeholders raise `ValueError` listing the supported placeholders.
