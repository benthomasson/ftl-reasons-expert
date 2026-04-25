# File: reasons_lib/derive.py

**Date:** 2026-04-24
**Time:** 16:48

## `reasons_lib/derive.py` — LLM-Assisted Belief Derivation

### 1. Purpose

This file is the **automated reasoning engine** for the belief network. It takes the current state of the network, builds an LLM prompt that presents existing beliefs and derived conclusions, sends it to a model, then parses and validates the model's proposals for new derived beliefs. It owns the full pipeline: prompt construction → response parsing → validation → application to the database (or writing to a file for human review).

Think of it as a "reasoning amplifier" — it doesn't do the reasoning itself, it orchestrates an LLM to find emergent connections that the network doesn't yet contain.

### 2. Key Components

**`DERIVE_PROMPT` (template string)** — The mega-prompt sent to the LLM. It teaches the model what an RMS is, defines two output formats (`### DERIVE` for standard derived beliefs, `### GATE` for outlist-gated beliefs), and injects the current network state via format placeholders. The prompt is carefully structured to encourage deeper chains (depth N+1 from depth N) over shallow groupings.

**`build_prompt(nodes, ...)`** — The main prompt assembly function. Takes a raw `nodes` dict (from `api.export_network`) and numerous filtering/budget parameters. Returns `(prompt_text, stats_dict)`. This is where all the filtering logic lives: topic filtering, depth filtering, `premises_only`, `has_dependents`, and budget-capped belief selection. It detects multi-agent networks and adjusts the prompt accordingly.

**`parse_proposals(response)`** — Regex-based parser for LLM output. Supports two format generations: the current `### DERIVE id` format and a legacy `### DERIVE: \`id\`` format with bold field names. Returns a list of proposal dicts with keys `kind`, `id`, `text`, `antecedents`, `unless`, `label`.

**`validate_proposals(proposals, nodes)`** — The safety gate. Checks three things: (1) all referenced antecedents/outlist nodes exist, (2) the proposed ID doesn't already exist, (3) the proposed ID isn't suspiciously similar to a retracted (OUT) belief. Returns `(valid, skipped)` with skip reasons.

**`apply_proposals(valid, db_path)`** — Commits valid proposals to the database via `api.add_node`. Each proposal becomes a real node with SL justification. Errors are caught per-proposal and returned alongside successes.

**`write_proposals_file(valid, output_path)`** — Alternative to `apply_proposals` — writes proposals to a markdown file in the same format `parse_proposals` can read, enabling a human-in-the-loop review workflow.

### 3. Patterns

**Prompt engineering as a module** — The entire file is structured around building, sending, parsing, and validating LLM output. The prompt template is a first-class artifact with its own internal documentation.

**Budget-capped prompt construction** — `_build_beliefs_section` enforces a `max_beliefs` budget to keep prompts within token limits. Two strategies: alphabetical truncation (default) or random sampling (`sample=True` with optional `seed` for reproducibility). When agents are present, the budget is allocated proportionally across agents.

**Graceful format migration** — `parse_proposals` tries the new regex first; only falls back to the old format if zero matches are found. This lets old LLM outputs or saved proposal files still work.

**Jaccard similarity for retraction protection** — `find_similar_out` tokenizes belief IDs into word sets and computes Jaccard similarity. This prevents the LLM from re-deriving beliefs that were previously retracted (a key TMS invariant — retraction should stick until explicitly undone).

**Memoized depth computation** — `_get_depth` uses a `memo` dict passed through the recursion to avoid recomputing depths in the DAG. It also sets `memo[node_id] = 0` before recursing as a cycle guard.

### 4. Dependencies

**Imports:**
- `api` — the only internal dependency. Used by `apply_proposals` to call `api.add_node`.
- `random` — for budget sampling.
- `re` — for parsing LLM responses.
- `subprocess`, `sys`, `Path` — imported but **unused in the current code**. Likely remnants from when this module invoked the LLM directly (that responsibility has since moved to the CLI or calling code).

**Imported by:**
- `reasons_lib/api.py` — exposes derive functions through the public API layer.
- `reasons_lib/cli.py` — wires `build_prompt`, `parse_proposals`, `validate_proposals`, and `apply_proposals` into the `reasons derive` CLI command.
- `tests/test_derive.py`, `tests/test_derive_budget.py` — unit tests.

### 5. Flow

The typical execution path:

1. **Caller exports network** → `api.export_network()` returns `{"nodes": {...}}`.
2. **`build_prompt`** filters nodes (topic, depth, premises-only), detects agents, builds the beliefs and derived sections, and assembles the full prompt.
3. **Caller sends prompt to LLM** (this module doesn't do the LLM call itself).
4. **`parse_proposals`** extracts structured proposals from the LLM's text response.
5. **`validate_proposals`** checks each proposal against the live network state — missing antecedents, duplicates, similarity to retracted beliefs.
6. **`apply_proposals`** or **`write_proposals_file`** — either commit directly or write for review.

Data flows as: `nodes dict → prompt string → LLM response string → proposal dicts → validated dicts → database writes or markdown file`.

### 6. Invariants

- **Antecedents must exist**: A proposal is rejected if any antecedent or outlist node is not in the network. No dangling references.
- **No duplicates**: A proposal whose ID already exists in the network is skipped.
- **No resurrection of retracted beliefs**: If a proposed ID has ≥50% Jaccard similarity with any OUT node's ID, it's skipped. This preserves the TMS invariant that retraction is meaningful.
- **Budget is hard-capped**: `_build_beliefs_section` never emits more than `max_beliefs` entries, with a floor of 5 per group.
- **Depth is well-defined**: Premises have depth 0; derived nodes have depth = max(antecedent depths) + 1. Cycles are guarded by pre-setting `memo[node_id] = 0`.

### 7. Error Handling

- `apply_proposals` wraps each `api.add_node` call in a try/except and returns the error string alongside the proposal — one bad proposal doesn't abort the batch.
- `validate_proposals` doesn't raise; it partitions proposals into valid and skipped with explanatory reasons.
- `parse_proposals` returns an empty list on no matches — no error, just nothing to do.
- There's no validation that the LLM response is well-formed beyond the regex; malformed output is silently ignored.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — Where `build_prompt`, `parse_proposals`, `validate_proposals`, and `apply_proposals` are re-exported as the public API; also where `add_node` lives
- [file] `tests/test_derive.py` — Exercises the parse/validate/apply pipeline with fixture networks; shows what proposal formats are expected
- [file] `tests/test_derive_budget.py` — Tests the budget-capping and sampling behavior in `_build_beliefs_section`
- [function] `reasons_lib/cli.py:derive` — The CLI handler that actually calls the LLM and orchestrates the full derive flow
- [general] `outlist-semantics` — How GATE proposals create non-monotonic beliefs ("X holds unless Y is IN") and how retraction/restoration interacts with them in `network.py`

---

## Beliefs

- `derive-no-llm-call` — `derive.py` builds prompts and parses responses but never calls an LLM itself; the caller (CLI or API) is responsible for the model invocation
- `derive-retraction-guard` — Proposals with ≥50% Jaccard token overlap against any OUT node ID are rejected, preventing re-derivation of retracted beliefs
- `derive-validate-before-apply` — `apply_proposals` trusts its input; callers must run `validate_proposals` first or risk database errors from missing antecedents
- `derive-budget-floor-five` — Each agent group and the non-agent group are guaranteed at least 5 belief slots regardless of proportional allocation
- `derive-unused-imports` — `subprocess`, `sys`, and `Path` are imported but not used in the current code

