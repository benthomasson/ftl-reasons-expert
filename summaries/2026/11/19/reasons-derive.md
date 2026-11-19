---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/derive.py

# reasons/derive.py

## Purpose

`reasons/derive.py` owns the **derive-proposal workflow** for a Reason Maintenance System network. It takes an exported belief graph, builds an LLM prompt that asks for new higher-level derived beliefs or outlist-gated beliefs, parses the LLM response, validates proposals against the graph, and either writes them for human review or applies them to the database via `api.add_node`.

It does **not** call the LLM itself, load the database, or run the CLI. It is a library layer used by `reasons/api.py`, `reasons/cli.py`, tests, and the mirrored `reasons_lib/derive.py`.

The module also supports:

- Budgeting/sampling belief lists for large networks.
- Topic filtering.
- Depth filtering and premises/dependent filtering.
- Semantic clustering of beliefs via `reasons/cluster.py`.
- Agent-namespaced beliefs such as `agent-a:belief-id`.
- Cross-agent derivation prompts.
- Backward-compatible parsing of old and new LLM output formats.

---

## Key Components

### Constants

- `DERIVE_PROMPT`
  - The main prompt template.
  - Placeholders: `{domain_context}`, `{beliefs_section}`, `{derived_section}`, `{total_in}`, `{total_derived}`, `{max_depth}`, `{cross_agent_task}`, `{agents_stats}`.
  - Instructs the model to emit either `### DERIVE <id>` or `### GATE <id>` blocks with antecedents, optional unless-list, mode, and label.

- `CROSS_AGENT_TASK`
  - Additional prompt fragment inserted when agent-namespaced nodes are detected.
  - Encourages derivations that combine knowledge from multiple agents.

### Depth and filtering helpers

- `_get_depth(node_id, nodes, derived, memo=None)`
  - Computes reasoning depth recursively.
  - A node not in `derived` has depth `0`.
  - A derived node has depth `1 + max(depth(antecedent))`.
  - If `supporting_justification` points to a valid index, only that justification is followed; otherwise all justifications are followed.
  - Uses a memo dict and sets `memo[node_id] = 0` before recursion as a cycle guard.

- `_detect_agents(nodes)`
  - Detects agent namespaces by looking for `:` in node IDs.
  - `agent-a:belief-id` is grouped under `agent-a`.
  - IDs ending in `:active` are skipped.
  - Despite the docstring mentioning metadata, the implementation is ID-based only.

- `_filter_by_topic(nodes, topic)`
  - Splits `topic` on whitespace.
  - Keeps a node if **any** keyword appears as a case-insensitive substring of `f"{node_id} {text}"`.
  - This is OR semantics, not AND semantics.

- `_sample_beliefs(belief_ids, budget, rng=None)`
  - Returns all IDs if `len <= budget`.
  - Otherwise uses `rng.sample` to select up to `budget`.
  - The docstring says “reservoir sampling,” but the implementation is `random.Random.sample`, which is uniform sampling without replacement, not streaming reservoir sampling.

### Prompt-section builders

- `_build_beliefs_section(...)`
  - Builds the “Existing Beliefs” section.
  - Includes only nodes where `truth_value == "IN"` and `node_id not in derived`.
  - Therefore it lists base/non-derived IN beliefs; derived beliefs are rendered separately.
  - Supports three display modes:
    1. `intra_cluster=True`: delegates to `cluster_beliefs_intra`, focuses budget on one cluster per round.
    2. `cluster=True`: delegates to `cluster_beliefs`.
    3. Neither: plain grouping/truncation.
  - In agent-aware mode, groups by `agent_name:` prefix and allocates budget proportionally per agent.
  - In non-agent mode, groups by the first dash-separated ID prefix.
  - Truncates belief text to 120 characters in listings.
  - Returns `(section_text, cluster_stats)`, where `cluster_stats` is `None` unless clustering was used.
  - Important budget caveat: in agent mode it enforces at least 5 beliefs per agent and at least 5 local beliefs, so the total can exceed `max_beliefs`.

- `_build_derived_section(nodes, derived, max_derived=300)`
  - Builds the “Existing Derived Conclusions” section.
  - Sorts derived nodes by descending depth.
  - Displays status, depth, node ID, truncated text, antecedents, and optional unless-list.
  - Uses `derived[k]["justifications"][0]` for antecedents/outlist, not necessarily the `supporting_justification` selected by `_get_depth`.
  - Stops after `max_derived` entries and emits an omission notice.

### Parsing

- `parse_proposals(response)`
  - Parses LLM output into proposal dictionaries:
    ```python
    {
        "kind": "derive" | "gate",
        "id": str,
        "text": str,
        "antecedents": list[str],
        "unless": list[str],
        "mode": "all" | "any",
        "label": str,
    }
    ```
  - New format:
    ```text
    ### DERIVE belief-id
    claim text
    - Antecedents: a, b
    - Unless: c
    - Mode: ALL
    - Label: rationale
    ```
  - New-format regex requires:
    - `### DERIVE|GATE <id>`
    - One-line claim text.
    - `- Antecedents:` line.
    - Optional `- Unless:` line.
    - Optional `- Mode: ALL|ANY` line; only uppercase.
    - `- Label:` line.
  - If the optional mode is missing in the new format, mode defaults to `"all"`.
  - Old legacy format is parsed only if **zero** new-format proposals matched:
    - `### DERIVE: \`id\``
    - `### GATE (outlist): \`id\``
    - Old proposals do not include a `"mode"` key.
  - Old-format kind detection uses a 30-character window around the match and checks whether `"GATE"` appears in that window.

### Prompt construction

- `build_prompt(...)`
  - Main entry point for prompt creation.
  - Applies topic filtering first.
  - Computes derived nodes, max depth, and referenced nodes.
  - Applies optional filters:
    - `premises_only`: keep nodes without truthy justifications.
    - `has_dependents`: keep only nodes referenced as antecedent or outlist.
    - `min_depth` / `max_depth_filter`: filter by `_get_depth`.
  - Recomputes `derived`, `in_nodes`, and agents after filtering.
  - Builds domain context, agent stats, cross-agent instructions, beliefs section, and derived section.
  - Formats the prompt using `prompt_template or DERIVE_PROMPT`.
  - Raises `ValueError` for custom-template unknown placeholders or malformed braces.
  - Returns `(prompt_text, stats_dict)`.

### Similarity and validation

- `_tokenize_id(node_id)`
  - Lowercases, replaces `:` with `-`, splits on `-`, and returns a set of tokens.

- `_jaccard(a, b)`
  - Returns Jaccard similarity between two token sets.

- `find_similar_out(proposal_id, nodes, threshold=0.5)`
  - Finds `OUT` beliefs whose tokenized ID similarity is at least `threshold`.
  - Returns `(out_id, similarity)` tuples sorted descending by similarity.

- `validate_proposals(proposals, nodes)`
  - Returns `(valid, skipped)`.
  - Skips proposals when:
    - Any antecedent or unless ID is missing from `nodes`.
    - The proposal ID already exists.
    - There are fewer than 2 antecedents.
    - The proposal ID is too similar to a retracted `OUT` belief (`>= 0.5` Jaccard).
  - Does **not** validate that unless IDs are actually `OUT`, that IDs are kebab-case, or that `kind == "gate"` has a non-empty unless list.

### Application and review output

- `apply_proposals(valid, db_path="reasons.db")`
  - Converts each proposal into `api.add_node` arguments:
    - `sl=",".join(antecedents)`
    - `unless=",".join(unless)` or empty
    - `source_type="derived"`
    - `any_mode=(proposal.get("mode") == "any")`
  - Catches exceptions per proposal and converts them to string results.
  - Always uses `source_type="derived"`; GATE behavior is carried by `unless`, not by a distinct source type.

- `write_proposals_file(valid, output_path)`
  - Writes proposals to a markdown file for human review.
  - Uses the same `### DERIVE` / `### GATE` format that `parse_proposals` can read, so `reasons accept proposed-derivations.md` can parse the file directly.
  - Returns `output_path`.

---

## Patterns

- **Template-driven prompt generation**
  - Prompt content is assembled from modular sections and inserted into a format string.
  - Custom `prompt_template` support allows callers to override the default prompt while preserving placeholder contracts.

- **Lazy optional dependency loading**
  - `cluster_beliefs` and `cluster_beliefs_intra` are imported inside `_build_beliefs_section` only when clustering is requested.
  - This avoids importing embedding/clustering dependencies for normal derive runs.

- **Filter pipeline**
  - Topic filtering → derived/depth/reference computation → premises/dependent/depth filtering → agent detection → section building.
  - The order matters because depth and references are computed before some filters.

- **Budgeted sampling**
  - Non-cluster sampling uses `random.Random(seed)` and `rng.sample`.
  - Cluster sampling delegates budget selection to `cluster.py`.

- **Agent namespace grouping**
  - IDs containing `:` are treated as agent-scoped.
  - Prompt stats and sections are adjusted to show per-agent budgets and cross-agent instructions.

- **LLM response parsing with backward compatibility**
  - New structured markdown format is preferred.
  - Old v0.9 format is supported as a fallback.

- **Validate-then-apply separation**
  - Parsing, validation, review-file writing, and database application are separate functions.
  - This supports human review workflows before mutation.

- **Per-item error capture**
  - `apply_proposals` continues after individual failures and records error strings.

---

## Dependencies

### Internal imports

- `from . import api`
  - Used only in `apply_proposals` via `api.add_node`.

- Lazy imports from `.cluster`:
  - `cluster_beliefs_intra`
  - `cluster_beliefs`
  - Used only when `intra_cluster` or `cluster` is true.

### Standard-library imports

- `random` — sampling.
- `re` — parsing LLM output.
- `collections.defaultdict` — grouping agents, prefixes, clusters.
- `subprocess`, `sys`, `pathlib.Path` — currently unused in this file.

### Imported by

- `reasons/api.py`
- `reasons/cli.py`
- `reasons_lib/derive.py`
- Tests including `tests/test_derive.py`, `tests/test_derive_budget.py`, `tests/test_source_type.py`

The repository contains a mirrored `reasons_lib/derive.py`; maintainers should check whether changes must be synchronized between `reasons/` and `reasons_lib/`.

---

## Flow

Typical derive workflow:

1. Caller loads/export the belief network as a `nodes` dict.
2. Caller invokes `build_prompt(...)` with options such as `budget`, `topic`, `sample`, `cluster`, depth filters, etc.
3. `build_prompt`:
   - Filters nodes.
   - Computes depth/reference metadata.
   - Detects agent namespaces.
   - Builds belief and derived-conclusion sections.
   - Formats `DERIVE_PROMPT` or a custom template.
   - Returns `(prompt, stats)`.
4. Caller sends the prompt to an LLM.
5. Caller passes the LLM response to `parse_proposals(response)`.
6. Caller passes parsed proposals to `validate_proposals(proposals, nodes)`.
7. Caller either:
   - Calls `write_proposals_file(valid, path)` for human review, or
   - Calls `apply_proposals(valid, db_path=...)` to mutate the network.
8. `apply_proposals` calls `api.add_node` once per proposal and collects per-proposal results.

Inside `build_prompt`, the important internal ordering is:

```text
topic filter
→ compute all_derived
→ compute max_depth
→ compute referenced nodes
→ apply premises_only / has_dependents / depth filters
→ recompute derived and in_nodes
→ detect agents
→ build prompt sections
→ format template
→ return prompt and stats
```

---

## Invariants

- **Beliefs section vs. derived section**
  - `_build_beliefs_section` excludes derived nodes and only lists `truth_value == "IN"` base beliefs.
  - Derived nodes are listed by `_build_derived_section`, including their `IN`/`OUT` status.

- **Depth semantics**
  - Non-derived nodes have depth `0`.
  - Derived nodes have depth `1 + max(antecedent depth)`.
  - `_get_depth` prefers `supporting_justification` when its index is valid.
  - `_build_derived_section` displays antecedents from `justifications[0]`, not necessarily the supporting justification.

- **Filtering semantics**
  - `premises_only` keeps nodes with no truthy justifications.
  - `has_dependents` keeps nodes referenced by any antecedent or outlist.
  - `min_depth` and `max_depth_filter` apply to all kept nodes, not only derived nodes.
  - Topic filtering uses OR across space-separated keywords.

- **Proposal validation**
  - A proposal must have at least 2 existing antecedents.
  - All antecedent and unless IDs must exist in the network.
  - Proposal IDs must be new.
  - Proposal IDs with Jaccard token similarity `>= 0.5` to any `OUT` belief are rejected as retracted-belief duplicates.
  - Unless IDs are not required to be `OUT` by validation.

- **Parsing strictness**
  - New-format mode only matches uppercase `ALL` or `ANY`.
  - Missing mode in the new format defaults to `"all"`.
  - Old-format proposals omit the `mode` key entirely.
  - If any new-format proposal parses, old-format parsing is skipped.

- **Application behavior**
  - `apply_proposals` always calls `api.add_node` with `source_type="derived"`.
  - `any_mode` is true only when `proposal.get("mode") == "any"`.
  - Errors from `api.add_node` are captured as result strings, not raised.

- **Budget behavior**
  - `budget` is used for both `max_beliefs` and `max_derived`.
  - Agent-mode belief budget is not strict because each agent gets at least 5 and local beliefs get at least 5.

- **Agent detection**
  - Agent scope is inferred solely from the first `:`-separated ID segment.
  - IDs ending in `:active` are excluded from agent lists.

---

## Error Handling

- `build_prompt` validates custom template formatting:
  - Unknown placeholder → `ValueError` with available placeholder names.
  - Malformed braces in a custom template → `ValueError` explaining `{{`/`}}`.
  - Errors from the default `DERIVE_PROMPT` are re-raised.

- `parse_proposals` does not raise on malformed model output; it simply returns the proposals it can match, possibly `[]`.

- `validate_proposals` does not raise on invalid proposals; it returns them in `skipped` with a reason string.

- `apply_proposals` catches all exceptions from `api.add_node` per proposal and appends the exception string to results. Partial application is possible.

- `write_proposals_file` does not catch I/O errors; file-open/write failures propagate.

- `_get_depth` guards against recursion cycles by setting `memo[node_id] = 0` before recursing, though this can produce an approximate depth for cyclic graphs.

---

## Topics to Explore

- [file] `reasons/api.py` — See how derive functions are exposed and orchestrated by the API layer.
- [function] `reasons/api.py:add_node` — Understand the `sl`, `unless`, `any_mode`, and `source_type` contract used by `apply_proposals`.
- [file] `reasons/cluster.py` — Learn the contracts of `cluster_beliefs` and `cluster_beliefs_intra`, including cluster stats and cache behavior.
- [file] `tests/test_derive.py` — See expected prompt construction, parsing, and proposal validation behavior.
- [general] `outlist-gating-semantics` — Understand how `unless`/OUT beliefs flip truth values and why GATE proposals are represented through `unless`.

## Beliefs

- `beliefs-section-excludes-derived-nodes` — `_build_beliefs_section` only lists nodes that are `IN` and not in the derived set; derived nodes are listed separately.
- `validate-proposals-requires-at-least-two-antecedents` — `validate_proposals` skips proposals with fewer than two antecedent IDs.
- `validate-proposals-rejects-similar-retracted-beliefs` — `validate_proposals` rejects a proposal whose ID has Jaccard token similarity `>= 0.5` with any `OUT` node.
- `apply-proposals-converts-api-errors-to-result-strings` — `apply_proposals` catches exceptions from `api.add_node` per proposal and returns them as strings instead of raising.
- `gate-proposals-still-use-derived-source-type` — `apply_proposals` always passes `source_type="derived"`; GATE behavior is represented by the `unless` argument.
