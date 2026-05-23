# File: reasons_lib/derive.py

**Date:** 2026-05-11
**Time:** 13:01

## Purpose

`derive.py` is the **LLM-driven reasoning chain builder** for the `ftl-reasons` Truth Maintenance System. It takes the current belief network, formats it into a structured prompt, sends it to an LLM, parses the LLM's proposed new derived beliefs, validates them against the existing network, and either applies them directly or writes them to a file for human review.

It owns the entire derive pipeline: prompt construction, proposal parsing, validation, deduplication (via Jaccard similarity against retracted beliefs), and persistence.

## Key Components

### Constants

**`DERIVE_PROMPT`** — The system prompt template. It teaches the LLM about the three node types (premises, derived, outlist-gated), explains retraction cascades, and instructs it to output proposals in a strict `### DERIVE` / `### GATE` format. Placeholders: `{beliefs_section}`, `{derived_section}`, `{total_in}`, `{total_derived}`, `{max_depth}`, `{domain_context}`, `{cross_agent_task}`, `{agents_stats}`.

**`CROSS_AGENT_TASK`** — Injected into the prompt only when multi-agent beliefs are detected. Adds a fourth task: derive cross-agent beliefs that combine knowledge across agent namespaces.

### Graph Utilities

**`_get_depth(node_id, nodes, derived, memo)`** — Computes the depth of a node in the derivation DAG via recursive traversal. Base premises are depth 0; derived nodes are 1 + max(antecedent depths). Uses `memo` dict for memoization and as a cycle guard (sets depth to 0 before recursing).

**`_detect_agents(nodes)`** — Identifies agent namespaces from colon-separated node IDs (e.g., `agent-name:belief-id`). Skips `:active` sentinel nodes. Returns `{agent_name: [node_ids]}`.

**`_filter_by_topic(nodes, topic)`** — Keyword filter. Splits `topic` on spaces, matches case-insensitively against node ID + text. Returns matching subset.

### Budget & Sampling

**`_sample_beliefs(belief_ids, budget, rng)`** — Reservoir sampling via `rng.sample()`. Returns all beliefs if under budget.

**`_build_beliefs_section(nodes, derived, ...)`** — The most complex function. Builds the beliefs portion of the prompt with three strategies:
1. **Clustering** (`cluster=True`): Calls `cluster.cluster_beliefs()` for semantic sampling across domains. Groups output by agent or by ID prefix.
2. **Random sampling** (`sample=True`): Uniform random selection within proportional per-agent budgets.
3. **Alphabetical truncation** (default): First N beliefs alphabetically, grouped by ID prefix.

Returns `(section_text, cluster_stats)` — `cluster_stats` is `None` unless clustering was used.

**`_build_derived_section(nodes, derived)`** — Formats existing derived conclusions sorted by depth (deepest first), showing status, antecedents, and outlist.

### Prompt Assembly

**`build_prompt(nodes, ...)`** — The main entry point for prompt construction. Orchestrates:
1. Topic filtering
2. Depth/filter computation on the full graph
3. Node filtering (premises-only, has-dependents, depth range)
4. Agent detection
5. Domain context and cross-agent task injection
6. Beliefs and derived section building
7. Template formatting with error handling for custom templates

Returns `(prompt_text, stats_dict)`. The stats dict captures everything needed to reproduce or audit the prompt (budget, sample mode, cluster info, filters applied).

### Proposal Parsing

**`parse_proposals(response)`** — Two-format parser:
1. **New format** (v0.10+): `### DERIVE belief-id` / `### GATE belief-id` with plain-text field labels
2. **Old format** (v0.9): `### DERIVE: \`belief-id\`` with bold-markdown field labels

Tries new format first; falls back to old only if no new-format matches are found. Returns list of proposal dicts with keys: `kind`, `id`, `text`, `antecedents`, `unless`, `label`.

### Validation & Deduplication

**`_tokenize_id(node_id)`** / **`_jaccard(a, b)`** — Tokenizes belief IDs on hyphens/colons and computes Jaccard set similarity.

**`find_similar_out(proposal_id, nodes, threshold=0.5)`** — Searches for retracted (OUT) beliefs with similar IDs. Prevents re-deriving beliefs that were previously retracted.

**`validate_proposals(proposals, nodes)`** — Three checks per proposal:
1. All antecedents and outlist nodes must exist in the network
2. The proposed ID must not already exist
3. The proposed ID must not be similar to any retracted belief (Jaccard >= 0.5)

Returns `(valid, skipped)` with skip reasons.

### Persistence

**`apply_proposals(valid, db_path)`** — Calls `api.add_node()` for each valid proposal. Formats antecedents and outlist as comma-separated strings. Catches exceptions per-proposal and returns `(proposal, result_or_error)` tuples.

**`write_proposals_file(valid, output_path)`** — Writes proposals in the same `### DERIVE` / `### GATE` markdown format that `parse_proposals()` can re-read. This creates a human-review checkpoint: edit the file, delete unwanted proposals, then `reasons accept` parses and applies the remainder.

## Patterns

- **Prompt-as-contract**: The LLM output format is rigidly specified in the prompt and parsed with regexes. The same format is used for file-based review, creating a single serialization format for proposals.
- **Budget-proportional allocation**: When multiple agents contribute beliefs, prompt space is divided proportionally to each agent's belief count, with a minimum floor of 5.
- **Graduated sampling**: Three tiers — alphabetical truncation (deterministic, cheap), random sampling (diverse, cheap), semantic clustering (diverse, expensive — requires embeddings).
- **Dedup via Jaccard on IDs**: Rather than comparing belief text, similarity is computed on tokenized IDs. This is fast and catches re-derivations of retracted beliefs even when the text is rephrased.
- **Cycle guard in depth computation**: `memo[node_id] = 0` is set before recursing, so cycles return depth 0 rather than infinite-looping.

## Dependencies

**Imports**: `api` (for `add_node`), `cluster` (lazy import inside `_build_beliefs_section` only when `cluster=True`). `random`, `re`, `defaultdict` from stdlib. `subprocess`, `sys`, and `Path` are imported but unused in the current code.

**Imported by**: `reasons_lib/api.py` (which exposes derive as part of the public API), `reasons_lib/cli.py` (CLI commands), `tests/test_derive.py` and `tests/test_derive_budget.py`. The sympy imports in `.venv/` are false positives — unrelated `derive` symbols.

## Flow

A typical derive invocation:

1. Caller loads the network (`export_network()`) and passes `nodes` dict to `build_prompt()`
2. `build_prompt()` filters nodes, detects agents, builds the beliefs and derived sections, fills the template, returns `(prompt, stats)`
3. Caller sends prompt to an LLM, gets back text
4. `parse_proposals(response)` extracts structured proposals from the LLM text
5. `validate_proposals(proposals, nodes)` filters out invalid/duplicate/similar-to-retracted proposals
6. Either `apply_proposals(valid)` writes directly to the database, or `write_proposals_file(valid, path)` creates a review checkpoint

## Invariants

- Every proposal must have **at least 2 antecedents** (enforced by prompt instruction, not code)
- All antecedent and outlist IDs must exist in the current network (enforced by `validate_proposals`)
- Proposed IDs must not collide with existing nodes (enforced by `validate_proposals`)
- Proposed IDs must not be Jaccard-similar (>= 0.5) to any OUT node (enforced by `validate_proposals`)
- Budget is a hard cap on beliefs in the prompt, not on proposals returned
- `_get_depth` returns 0 for premises and for cycles (via the memo cycle guard)
- `write_proposals_file` output is round-trippable through `parse_proposals`

## Error Handling

- `build_prompt()` catches `KeyError` and `ValueError` from `str.format()` to give actionable errors when custom prompt templates reference unknown or malformed placeholders. The `from None` suppresses the original traceback.
- `apply_proposals()` wraps each `api.add_node()` call in a try/except and returns the error string alongside the proposal — one bad proposal doesn't abort the batch.
- `validate_proposals()` doesn't raise; it partitions proposals into valid/skipped with human-readable skip reasons.

## Topics to Explore

- [file] `reasons_lib/cluster.py` — The semantic clustering implementation (`cluster_beliefs`) that powers the `cluster=True` sampling strategy
- [function] `reasons_lib/api.py:add_node` — The database-level function that `apply_proposals` delegates to; defines how SL justifications and outlists are stored
- [file] `tests/test_derive.py` — Tests for prompt building, proposal parsing (both formats), validation, and the Jaccard dedup logic
- [function] `reasons_lib/derive.py:_build_beliefs_section` — The most complex function here; worth tracing each of the three sampling paths (cluster, random, truncation) independently
- [general] `outlist-gated-belief-lifecycle` — How GATE beliefs flip between IN and OUT as their outlist nodes change truth value, and the known issue where outlist nodes aren't tracked in the dependents index

## Beliefs

- `derive-prompt-round-trippable` — `write_proposals_file` output can be parsed back by `parse_proposals` with no data loss, enabling a write-review-accept cycle
- `derive-validate-blocks-retracted-rediscovery` — `validate_proposals` rejects any proposal whose ID has >= 50% Jaccard token overlap with an existing OUT belief, preventing re-derivation of retracted conclusions
- `derive-depth-cycle-guard` — `_get_depth` uses a memo dict pre-set to 0 before recursion to prevent infinite loops on cyclic justification graphs, treating cycles as depth 0
- `derive-apply-is-fault-tolerant` — `apply_proposals` catches exceptions per-proposal and continues, never aborting the batch on a single failure
- `derive-unused-imports` — `subprocess`, `sys`, and `Path` are imported but not referenced anywhere in the module

