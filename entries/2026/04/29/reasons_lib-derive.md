# File: reasons_lib/derive.py

**Date:** 2026-04-29
**Time:** 17:02

## `reasons_lib/derive.py` — LLM-Driven Belief Derivation Engine

### 1. Purpose

This module is the **inference expansion layer** of the reasons system. It takes the current belief network, constructs a prompt that teaches an LLM how the TMS works, sends it the existing beliefs, and parses back structured proposals for new derived conclusions. It owns the full pipeline from prompt construction through proposal validation, but delegates actual database insertion to `api.add_node()`.

Its job is to deepen the reasoning graph — combining depth-N beliefs into depth-N+1 conclusions, and creating GATE beliefs that connect positive claims to negative ones via outlist semantics.

### 2. Key Components

**`DERIVE_PROMPT` (template string)** — The core LLM prompt. It teaches the model about the three node kinds (premises, derived, outlist-gated), explains what makes a good derivation, and specifies an exact output format (`### DERIVE id` / `### GATE id`). The template has slots for domain context, belief listings, derived conclusions, stats, and optional cross-agent instructions.

**`build_prompt(nodes, ...)`** — Assembles the full prompt from a network export. This is the most parameter-rich function:
- `topic` — keyword filter applied before anything else via `_filter_by_topic`
- `budget` — caps how many beliefs the LLM sees (default 300)
- `sample` / `seed` — random sampling vs. alphabetical truncation
- `min_depth` / `max_depth_filter` — depth-band filtering
- `premises_only` / `has_dependents` — structural filters

Returns `(prompt_text, stats_dict)`.

**`parse_proposals(response)`** — Regex parser that handles two format generations: the current `### DERIVE belief-id` format and the older `### DERIVE: \`belief-id\`` format with bold markdown labels. Falls through to the old parser only if the new one finds nothing.

**`validate_proposals(proposals, nodes)`** — The safety gate. Rejects proposals where:
- Any antecedent or outlist node doesn't exist in the network
- The proposed ID already exists
- The proposed ID has ≥50% Jaccard similarity to any OUT (retracted) belief — this prevents the LLM from re-deriving conclusions that were previously retracted

Returns `(valid, skipped)` with skip reasons.

**`apply_proposals(valid, db_path)`** — Iterates through validated proposals, calling `api.add_node()` for each. Wraps each call in try/except so one failure doesn't abort the batch. Returns a list of `(proposal, result_or_error)` tuples.

**`write_proposals_file(valid, output_path)`** — Serializes proposals to a markdown file in the same format `parse_proposals()` can read back. This is the human-review path: write proposals, let a human delete unwanted ones, then `reasons accept` re-parses the file.

**Helper functions:**
- `_get_depth()` — Recursive depth computation with memoization and cycle guard (sets depth to 0 before recursing)
- `_detect_agents()` — Identifies multi-agent namespaces from colon-separated node IDs (e.g., `agent-a:belief-id`), skipping `:active` premise nodes
- `_filter_by_topic()` — Case-insensitive keyword matching on node ID + text
- `_sample_beliefs()` — Reservoir sampling for budget-constrained prompt building
- `_build_beliefs_section()` / `_build_derived_section()` — Format beliefs for the prompt, with agent-aware grouping and proportional budget allocation
- `_tokenize_id()` / `_jaccard()` — Jaccard similarity on hyphen-split ID tokens, used for retraction-similarity detection

### 3. Patterns

**Prompt-parse-validate-apply pipeline** — The module follows a strict four-phase pipeline: build the prompt, parse the LLM's response, validate against the live network, then apply. Each phase is a pure function (except `apply_proposals` which has side effects). This makes each stage independently testable.

**Budget-based prompt construction** — Rather than dumping all beliefs into the prompt, the module implements a budget system with two strategies: alphabetical truncation (deterministic, reproducible) and random sampling (better coverage across large networks). When agents are present, the budget is allocated proportionally by agent size.

**Format versioning in the parser** — `parse_proposals` tries the new format first, falls back to old only if zero matches. This is a clean forward-compatible strategy — old files still parse, new output is preferred.

**Human-in-the-loop round-trip** — `write_proposals_file` and `parse_proposals` form a serialization round-trip. The markdown format is both human-readable and machine-parseable, letting users delete lines before acceptance.

### 4. Dependencies

**Imports:**
- `api` (internal) — `add_node()` for applying proposals to the database
- `random` — sampling for budget constraints
- `re` — regex parsing of LLM responses
- `subprocess`, `sys` — imported but unused in this module (used by the CLI layer that calls into it)
- `defaultdict` — grouping beliefs by prefix/agent
- `Path` — imported but unused here

**Imported by:**
- `reasons_lib/cli.py` — orchestrates the full derive pipeline including LLM invocation
- `reasons_lib/api.py` — uses `_tokenize_id` and `_jaccard` for deduplication
- `tests/test_derive.py`, `tests/test_derive_budget.py` — unit tests

### 5. Flow

The end-to-end flow (orchestrated by the CLI, not this module):

1. **Export** — `api.export_network()` produces a `{node_id: node_data}` dict
2. **Filter** — `build_prompt` optionally narrows by topic, depth band, or structural properties
3. **Budget** — `_build_beliefs_section` caps the number of beliefs shown to the LLM, either by truncation or sampling
4. **Prompt** — The template is filled with beliefs, derived conclusions, stats, and optional agent/domain context
5. **LLM call** — (happens in CLI, not here) — the prompt is sent to `claude` or `gemini` via subprocess
6. **Parse** — `parse_proposals` extracts structured proposals from the LLM's markdown response
7. **Validate** — `validate_proposals` checks antecedent existence, ID uniqueness, and retraction similarity
8. **Apply or Write** — Either `apply_proposals` adds them to the DB immediately, or `write_proposals_file` saves them for human review

Within `_build_beliefs_section`, there's a branching path: if agents are detected, beliefs are grouped by agent with proportional budget allocation. Otherwise, they're grouped by ID prefix (the first hyphen-separated segment).

### 6. Invariants

- **Antecedent existence**: Every antecedent and outlist reference in a proposal must exist in the current network, or the proposal is rejected.
- **ID uniqueness**: A proposal whose ID already exists in the network is skipped.
- **Retraction guard**: A proposal whose ID has ≥50% Jaccard token overlap with any OUT belief is skipped. This prevents the LLM from re-deriving beliefs that were previously retracted for good reason.
- **Minimum 2 antecedents**: Enforced by the prompt instructions (not code-validated), preventing trivial restatements.
- **Cycle guard in depth computation**: `_get_depth` sets `memo[node_id] = 0` before recursing, so circular justification chains don't infinite-loop — they just report depth 0.
- **Budget cap**: The beliefs section never exceeds `max_beliefs` entries, regardless of network size.

### 7. Error Handling

`apply_proposals` is the only function with explicit error handling — it wraps each `api.add_node()` call in try/except and collects errors as strings alongside successes. This means a malformed proposal doesn't abort the entire batch.

`validate_proposals` doesn't raise — it returns skip reasons as strings in the `skipped` list, letting callers report all problems at once rather than failing on the first one.

`parse_proposals` silently returns an empty list if the LLM response doesn't match either format — no error, no warning. The caller (CLI) is responsible for checking `len(proposals) == 0`.

`_get_depth` handles cycles silently via the memo sentinel rather than raising. This is intentional — in a TMS, circular justifications are a valid (if degenerate) state.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — Contains `add_node()` which actually creates TMS nodes with justifications and triggers propagation; also reuses `_tokenize_id`/`_jaccard` for its deduplication workflow
- [function] `reasons_lib/cli.py:cmd_derive` — The CLI orchestrator that shells out to LLM subprocesses, manages the `--exhaust` loop, and decides between auto-apply and human-review paths
- [file] `tests/test_derive.py` — Covers the full pipeline with fixture networks, including both format versions, validation edge cases, and the Jaccard retraction guard
- [general] `outlist-gate-semantics` — How GATE beliefs flip truth value when their outlist nodes go OUT; this is the core non-monotonic reasoning mechanism that `derive.py` creates proposals for
- [file] `reasons_lib/network.py` — The underlying TMS propagation engine that `add_node` triggers after a derived belief is inserted

## Beliefs

- `derive-validate-rejects-jaccard-similar-to-out` — `validate_proposals` skips any proposal whose ID has ≥50% Jaccard token similarity to an existing OUT belief, preventing re-derivation of retracted conclusions
- `derive-parse-supports-two-format-versions` — `parse_proposals` tries the new `### DERIVE id` format first, falling back to the older `### DERIVE: \`id\`` format only when the new parser returns zero matches
- `derive-apply-never-aborts-batch-on-single-failure` — `apply_proposals` catches exceptions per-proposal and accumulates `(proposal, error_string)` tuples, so one bad proposal doesn't prevent others from being applied
- `derive-depth-cycle-guard-returns-zero` — `_get_depth` sets `memo[node_id] = 0` before recursing into antecedents, so circular justification chains terminate with depth 0 rather than infinite recursion
- `derive-budget-allocation-is-proportional-by-agent` — When agent namespaces are detected, `_build_beliefs_section` allocates the prompt budget proportionally to each agent's belief count, with a floor of 5 per agent

