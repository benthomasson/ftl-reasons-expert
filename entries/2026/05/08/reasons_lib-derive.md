# File: reasons_lib/derive.py

**Date:** 2026-05-08
**Time:** 14:17

## Purpose

`derive.py` is the LLM-driven reasoning chain builder for the `reasons` TMS. It takes the existing belief network, constructs a prompt that presents all current beliefs and derived conclusions to an LLM, parses the LLM's proposed new derivations, validates them against the live network, and either applies them directly to the database or writes them to a file for human review.

It owns one responsibility: **turning a flat or shallow belief graph into a deeper one** by finding opportunities to combine existing beliefs into higher-level conclusions. It does not own the LLM call itself (that's handled by callers via `api.py` or `cli.py`), but it owns prompt construction, response parsing, validation, and application.

## Key Components

### Constants

**`DERIVE_PROMPT`** — The system prompt template. It teaches the LLM the RMS model (premises, derived, outlist-gated), defines output format (`### DERIVE` / `### GATE`), and gets filled with the actual belief data. The prompt enforces rules: minimum 2 antecedents, prefer deeper chains over grouping base beliefs, antecedents must be related.

**`CROSS_AGENT_TASK`** — An additional instruction block injected only when multi-agent beliefs are detected, encouraging cross-agent derivations.

### Core Functions

**`build_prompt(nodes, ...)`** — The main entry point. Takes the raw `nodes` dict from a network export and returns `(prompt_text, stats_dict)`. It orchestrates: topic filtering → depth computation → node filtering (by depth, premises-only, has-dependents) → agent detection → beliefs section construction → derived section construction → template interpolation. The `stats` dict captures what went into the prompt for audit trails.

**`parse_proposals(response)`** — Regex parser for LLM output. Handles two format generations: the current `### DERIVE belief-id` format and a legacy `### DERIVE: \`belief-id\`` format with bold markdown labels. Returns a list of proposal dicts with keys `kind`, `id`, `text`, `antecedents`, `unless`, `label`.

**`validate_proposals(proposals, nodes)`** — Checks each proposal against the network. Three rejection criteria: (1) any antecedent or unless node doesn't exist, (2) the proposed ID already exists, (3) the proposed ID is Jaccard-similar (≥50%) to a retracted (OUT) belief. Returns `(valid, skipped)` with skip reasons.

**`apply_proposals(valid, db_path)`** — Writes valid proposals into the SQLite database via `api.add_node`. Catches exceptions per-proposal so one failure doesn't abort the batch.

**`write_proposals_file(valid, output_path)`** — Serializes proposals to a markdown file in the same `### DERIVE` / `### GATE` format that `parse_proposals` can re-read, enabling a human-in-the-loop review cycle.

### Helper Functions

**`_get_depth(node_id, nodes, derived, memo)`** — Memoized recursive depth computation. Premises are depth 0; derived nodes are 1 + max(antecedent depths). Includes a cycle guard (sets `memo[node_id] = 0` before recursion).

**`_detect_agents(nodes)`** — Identifies multi-agent namespaces by splitting on `:` in node IDs. Skips `:active` sentinel nodes.

**`_filter_by_topic(nodes, topic)`** — Keyword filter over node ID + text. Space-separated keywords are OR'd (any match passes).

**`_sample_beliefs(belief_ids, budget, rng)`** — Reservoir-style random sampling to stay within token budget.

**`_build_beliefs_section(nodes, derived, ...)`** — The most complex helper. Three code paths based on flags:
1. **Cluster mode**: Uses `cluster.cluster_beliefs` for semantic sampling across domains, then groups output by agent or by ID prefix.
2. **Agent mode (no clustering)**: Allocates budget proportionally across agents, then alphabetically or randomly within each allocation.
3. **Default mode**: Groups beliefs by kebab-case prefix, outputs by group size (largest first), truncates at budget.

**`find_similar_out(proposal_id, nodes, threshold)`** — Jaccard similarity on tokenized belief IDs to detect proposals that are too close to previously retracted beliefs.

## Patterns

**Prompt-parse-validate-apply pipeline**: The module follows a strict four-stage pipeline. `build_prompt` → (external LLM call) → `parse_proposals` → `validate_proposals` → `apply_proposals` or `write_proposals_file`. Each stage is independently testable.

**Budget management**: The prompt can't contain all beliefs in a large network. The module offers three strategies — alphabetical truncation (default), random sampling (`sample=True`), and semantic clustering (`cluster=True`) — all governed by a single `budget` parameter.

**Format backward compatibility**: `parse_proposals` tries the new format first; if it finds nothing, falls back to the old format. This lets the module consume responses from different LLM versions or historical proposal files.

**Fail-soft per item**: Both `validate_proposals` and `apply_proposals` process each proposal independently. A bad proposal is skipped or logged, not fatal.

## Dependencies

**Imports**:
- `api` (from the same package) — `api.add_node` is the only database write path
- `cluster` (conditional import inside `_build_beliefs_section`) — semantic clustering with sentence-transformers; only loaded when `cluster=True`
- `subprocess`, `sys`, `Path` are imported but unused in the current code (likely remnants from an earlier version or used by callers that import from this module)

**Imported by**:
- `reasons_lib/api.py` and `reasons_lib/cli.py` — the CLI `derive` command calls `build_prompt`, `parse_proposals`, `validate_proposals`, and `apply_proposals`
- `tests/test_derive.py` and `tests/test_derive_budget.py` — unit tests for prompt building, parsing, and validation
- The sympy entries in the "imported by" list are false positives from a name collision (sympy's `derive` is unrelated)

## Flow

A typical derive invocation:

1. Caller loads the network via `api.export_network()` → `nodes` dict
2. `build_prompt(nodes, domain=..., budget=300)` filters nodes, detects agents, builds both the beliefs section and derived section, interpolates into `DERIVE_PROMPT`, returns `(prompt, stats)`
3. Caller sends `prompt` to an LLM, gets `response` text
4. `parse_proposals(response)` → list of proposal dicts
5. `validate_proposals(proposals, nodes)` → `(valid, skipped)` — checks existence, duplicates, similarity to retracted beliefs
6. Either `apply_proposals(valid)` writes to SQLite, or `write_proposals_file(valid, path)` writes a reviewable markdown file

Data transforms: `nodes` dict → filtered subset → formatted text sections → prompt string → (LLM) → raw text → parsed proposal dicts → validated subset → database rows or markdown file.

## Invariants

- **Every proposal must have ≥2 antecedents** (enforced by the prompt instructions, not by code — the code does not reject single-antecedent proposals)
- **All antecedents and unless nodes must exist in the network** (`validate_proposals` enforces this)
- **No duplicate belief IDs** (`validate_proposals` rejects proposals whose ID already exists)
- **Retraction-aware**: proposals similar to OUT beliefs (Jaccard ≥0.5 on tokenized IDs) are rejected to prevent re-deriving retracted conclusions
- **Depth is acyclic**: `_get_depth` assumes the justification graph is a DAG; the cycle guard prevents infinite recursion but would produce depth 1 for cyclic nodes
- **Budget is always respected**: `_build_beliefs_section` never emits more than `max_beliefs` entries regardless of code path

## Error Handling

- `apply_proposals` wraps each `api.add_node` call in a try/except, capturing the exception as a string in the results list. No proposal failure aborts the batch.
- `validate_proposals` never raises — invalid proposals are collected in the `skipped` list with human-readable reasons.
- `parse_proposals` returns an empty list on unparseable input (no exceptions).
- `_get_depth` silently returns 0 for cycles rather than raising.
- The conditional `from .cluster import cluster_beliefs` will raise `ImportError` at runtime if sentence-transformers isn't installed, but only when `cluster=True` is explicitly requested.

## Topics to Explore

- [file] `reasons_lib/api.py` — Where `add_node` is defined; the actual database write contract and how SL/outlist justifications are stored
- [file] `reasons_lib/cluster.py` — Semantic clustering implementation used by the `cluster=True` path; understanding how beliefs are grouped across domains
- [file] `tests/test_derive.py` — Test cases for prompt building, parsing both format generations, and validation logic including the Jaccard similarity guard
- [function] `reasons_lib/cli.py:derive` — The CLI entrypoint that orchestrates the full derive pipeline including LLM invocation and `--auto`/`--review` modes
- [general] `outlist-gate-semantics` — How GATE beliefs interact with retraction cascades in the underlying TMS; the known issue where outlist nodes aren't tracked in the dependents index

## Beliefs

- `derive-parse-proposals-backward-compat` — `parse_proposals` tries the new `### DERIVE id` format first and falls back to the old `### DERIVE: \`id\`` format, so it can consume outputs from any version
- `derive-validate-rejects-similar-retracted` — `validate_proposals` rejects any proposed belief whose tokenized ID has ≥50% Jaccard overlap with an existing OUT belief, preventing re-derivation of retracted conclusions
- `derive-apply-fail-soft` — `apply_proposals` catches exceptions per-proposal and returns them as strings in the results list; one failure never aborts the remaining proposals
- `derive-budget-three-strategies` — `_build_beliefs_section` supports three budget strategies: alphabetical truncation (default), random sampling (`sample=True`), and semantic clustering (`cluster=True`), all controlled by a single `max_beliefs` parameter
- `derive-depth-cycle-guard` — `_get_depth` sets `memo[node_id] = 0` before recursing into antecedents, preventing infinite recursion on cyclic justification graphs but silently computing depth 1 for any node in a cycle

