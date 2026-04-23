# File: reasons_lib/derive.py

**Date:** 2026-04-23
**Time:** 16:41

# `reasons_lib/derive.py` — Automated Derivation Engine

## Purpose

This file owns the **derivation pipeline** — the process of discovering new, higher-level beliefs by combining existing ones. It constructs an LLM prompt from the current belief network, parses the LLM's structured proposals, validates them against the live network, and either applies them directly or writes them to a review file. It's the "reasoning upward" mechanism: turning depth-0 premises and depth-1 conclusions into deeper chains.

## Key Components

### Constants

**`DERIVE_PROMPT`** — A large template string that instructs an LLM to act as a "reasoning architect." It teaches the LLM the three node kinds (base premises, derived conclusions, outlist-gated conclusions), defines output format (`### DERIVE` / `### GATE`), and gets filled with the current belief state. The prompt is carefully designed so its output can be round-tripped through `parse_proposals()`.

**`CROSS_AGENT_TASK`** — An addendum injected into the prompt when multi-agent beliefs are detected, encouraging cross-agent derivations (combining knowledge from different imported agents).

### Graph Traversal

**`_get_depth(node_id, nodes, derived, memo)`** — Recursive depth computation with memoization. A base premise is depth-0; a derived node is `max(antecedent depths) + 1`. The `memo[node_id] = 0` assignment before recursion acts as a cycle guard — if a cycle exists, the node gets depth 0 rather than infinite recursion.

### Filtering & Sampling

**`_detect_agents(nodes)`** — Scans for colon-namespaced IDs (e.g., `agent-name:belief-id`) to identify multi-agent networks. Skips `:active` premise nodes.

**`_filter_by_topic(nodes, topic)`** — Keyword filter: space-separated terms matched case-insensitively against node ID + text. Any keyword match includes the node (OR semantics).

**`_sample_beliefs(belief_ids, budget, rng)`** — Reservoir-style random sampling to stay within token budgets. Falls through to identity when the list is already under budget.

### Prompt Construction

**`_build_beliefs_section(nodes, derived, agents, ...)`** — Builds the base-beliefs portion of the prompt. Two modes:
- **With agents**: Allocates budget proportionally across agents, renders per-agent sections.
- **Without agents**: Groups by ID prefix (first segment before `-`), renders per-group.

Supports both alphabetical truncation (default) and random sampling (`sample=True`).

**`_build_derived_section(nodes, derived)`** — Renders existing derived conclusions sorted by depth (deepest first), showing status, antecedents, and outlist.

**`build_prompt(nodes, ...)`** — The main entry point for prompt construction. Orchestrates filtering (topic, depth, premises-only, has-dependents), computes stats, and assembles the final prompt string. Returns `(prompt_text, stats_dict)`.

### Proposal Parsing

**`parse_proposals(response)`** — Regex parser that extracts `DERIVE` and `GATE` proposals from LLM output. Supports two formats:
- **New format** (v0.10+): `### DERIVE belief-id` with plain-text field labels
- **Old format** (v0.9): `### DERIVE: \`belief-id\`` with bold field labels

Falls back to the old format only if the new format yields no matches. Each proposal is a dict with keys: `kind`, `id`, `text`, `antecedents`, `unless`, `label`.

### Validation & Similarity

**`_tokenize_id` / `_jaccard`** — Utility functions for comparing belief IDs by Jaccard similarity over their tokenized segments.

**`find_similar_out(proposal_id, nodes, threshold=0.5)`** — Finds OUT (retracted) beliefs whose ID is similar to a proposed ID. This prevents re-deriving beliefs that were previously retracted — a critical consistency check.

**`validate_proposals(proposals, nodes)`** — Returns `(valid, skipped)`. A proposal is skipped if:
1. Any antecedent or unless-node doesn't exist in the network
2. The proposed ID already exists
3. The proposed ID is similar to a retracted (OUT) belief (Jaccard >= 0.5)

### Application

**`apply_proposals(valid, db_path)`** — Commits valid proposals to the database by calling `api.add_node()` for each. Catches exceptions per-proposal so one failure doesn't block others.

**`write_proposals_file(valid, output_path)`** — Writes proposals to a markdown file in the same `### DERIVE` / `### GATE` format that `parse_proposals()` can read, enabling a human review loop: generate → edit file → `reasons accept`.

## Patterns

- **Prompt-as-contract**: The LLM prompt defines a structured output format, and `parse_proposals()` is the parser for that contract. The `write_proposals_file()` function also emits the same format, creating a three-way compatibility: LLM output ↔ review file ↔ parser.
- **Budget-based prompt construction**: Token budget management via `max_beliefs` parameter with proportional allocation across agents — avoids blowing up context windows on large networks.
- **Fail-soft validation**: `validate_proposals` filters rather than raises; `apply_proposals` catches per-item exceptions. The pipeline never fails entirely due to one bad proposal.
- **Retraction awareness**: `find_similar_out` prevents re-deriving retracted conclusions via fuzzy ID matching — an important property for a truth maintenance system where retraction carries semantic weight.

## Dependencies

**Imports**: Only `api` from the package (for `add_node`). Uses stdlib `random`, `re`, `defaultdict`. Note: `subprocess`, `sys`, and `Path` are imported but unused — likely vestigial.

**Imported by**: `reasons_lib/api.py` (which wraps `build_prompt`/`parse_proposals` into the public API), `reasons_lib/cli.py` (CLI commands), and `tests/test_derive.py`.

## Flow

The typical execution path:

1. **`build_prompt()`** receives a `nodes` dict (from `export_network`), applies filters (topic, depth, premises-only), detects agents, builds the beliefs and derived sections, and returns a formatted prompt string + stats.
2. An external caller sends the prompt to an LLM and gets a text response.
3. **`parse_proposals()`** extracts structured proposals from the response.
4. **`validate_proposals()`** checks each proposal against the live network, splitting into valid and skipped.
5. Either **`apply_proposals()`** commits to the DB, or **`write_proposals_file()`** writes a review file.

## Invariants

- Every proposal must have **at least 2 antecedents** (enforced by the prompt, not by code — the code trusts the LLM here).
- A proposal is rejected if **any referenced node doesn't exist** in the current network.
- A proposal is rejected if its ID has **Jaccard similarity >= 0.5** with any OUT node — preventing re-derivation of retracted beliefs.
- Depth computation is **cycle-safe**: the `memo[node_id] = 0` guard before recursion ensures cycles don't cause infinite recursion (they resolve to depth 0).
- Budget allocation across agents is **proportional to agent size**, with a minimum of 5 beliefs per agent.

## Error Handling

- `apply_proposals` wraps each `api.add_node` call in a try/except, appending the error string alongside the proposal. Callers receive a mixed list of `(proposal, result_dict)` and `(proposal, error_string)` tuples.
- `validate_proposals` never raises — it separates proposals into valid/skipped with reason strings.
- `_get_depth` silently handles cycles via the memo guard rather than raising.
- There is a bug in `_build_beliefs_section` with agents: `count += len(belief_ids)` is inside the per-belief loop instead of outside it, inflating the count. This means the non-agent budget (`remaining = max(5, max_beliefs - count)`) may be smaller than intended.

## Topics to Explore

- [file] `reasons_lib/api.py` — The public API layer that wraps `build_prompt`, `parse_proposals`, and `apply_proposals` into user-facing commands
- [file] `tests/test_derive.py` — Test cases that reveal edge cases in prompt building, parsing, and validation
- [function] `reasons_lib/network.py:export_network` — Produces the `nodes` dict that `build_prompt` consumes; understanding its schema is essential
- [general] `outlist-semantics` — How the GATE/outlist mechanism implements defeasible reasoning (beliefs that retract when a negative condition becomes true)
- [file] `reasons_lib/import_agent.py` — How agent-namespaced nodes get created, which determines how `_detect_agents` and cross-agent derivation work

## Beliefs

- `derive-retraction-guard-uses-jaccard` — `validate_proposals` rejects any proposed ID with Jaccard similarity >= 0.5 to an existing OUT node, preventing re-derivation of retracted beliefs
- `derive-prompt-roundtrips-through-parser` — The `### DERIVE` / `### GATE` format is shared between `DERIVE_PROMPT` output, `parse_proposals()` input, and `write_proposals_file()` output, forming a closed loop
- `derive-depth-cycle-guard` — `_get_depth` sets `memo[node_id] = 0` before recursing to prevent infinite recursion on cyclic justification chains
- `derive-agent-budget-proportional` — When agents are present, `_build_beliefs_section` allocates prompt budget proportionally to agent belief count with a floor of 5

