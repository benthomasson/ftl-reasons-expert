---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/derive.py

# `reasons/derive.py` — Deriving Deeper Reasoning Chains

## 1. Purpose

This module is the **LLM-powered derivation engine** for the Reasons system. Its job is to look at an existing belief network (nodes with justifications, truth values, and depth) and propose *new* higher-level conclusions by combining existing ones. It operates in a pipeline:

1. **Build a prompt** that presents the current network state to an LLM.
2. **Parse the LLM's response** into structured `derive` / `gate` proposals.
3. **Validate** those proposals against the actual graph (do antecedents exist? is the ID already taken? is it re-proposing a retracted belief?).
4. **Persist** accepted proposals into the reasons database via `api.add_node()`, or write them to a markdown file for human review.

It owns the concern of *what* to ask the LLM (prompt construction, budgeting, sampling strategy) and *how* to interpret the answer (parsing, validation). It does **not** own the LLM call itself — that lives in `api.py` or `cli.py`, which import this module.

## 2. Key Components

### Constants

| Name | Role |
|---|---|
| `DERIVE_PROMPT` | The main instruction template. Contains placeholders `{beliefs_section}`, `{derived_section}`, `{total_in}`, `{total_derived}`, `{max_depth}`, `{domain_context}`, `{cross_agent_task}`, `{agents_stats}`. Defines the output contract (`### DERIVE <id>` / `### GATE <id>`). |
| `CROSS_AGENT_TASK` | Injected into the prompt only when agent-namespaced nodes are detected. Instructs the LLM to produce cross-agent beliefs. |

### Functions

| Function | Contract |
|---|---|
| `_get_depth(node_id, nodes, derived, memo)` | Returns the integer depth of a node. Base premises → 0. Recursive with a **cycle guard** (sets `memo[node_id] = 0` before recursing). Respects `supporting_justification` index if present. |
| `_detect_agents(nodes)` | Scans node IDs for the `agent-name:belief-id` convention. Returns `{agent_name: [node_ids]}`. Skips `:active` suffix nodes. |
| `_filter_by_topic(nodes, topic)` | Case-insensitive substring match on space-separated keywords against ID + text. |
| `_sample_beliefs(belief_ids, budget, rng)` | If `len ≤ budget`, returns as-is. Otherwise uses `rng.sample()` for a uniform random subset. |
| `_build_beliefs_section(...)` | The most complex function. Produces the human-readable beliefs block for the prompt. Supports **five modes**: plain prefix-grouping, proportional agent allocation, random sampling, semantic clustering (`cluster=True`), and intra-cluster rotation (`intra_cluster=True`). Returns `(section_text, cluster_stats|None)`. |
| `_build_derived_section(nodes, derived, max_derived)` | Lists existing derived conclusions sorted by depth (deepest first), with antecedents, outlist, and truth value. Truncates at `max_derived`. |
| `parse_proposals(response)` | Parses LLM text into a list of dicts. Tries the **v0.10+ regex** first; falls back to the **v0.9 regex** if nothing matches. Each proposal has keys: `kind`, `id`, `text`, `antecedents`, `unless`, `mode`, `label`. |
| `build_prompt(nodes, ...)` | **Primary entry point.** Orchestrates filtering, agent detection, section building, template filling. Returns `(prompt_text, stats_dict)`. Accepts an optional `prompt_template` override. |
| `find_similar_out(proposal_id, nodes, threshold)` | Jaccard similarity on tokenized kebab-case IDs. Returns OUT beliefs above `threshold` (default 0.5), sorted descending. |
| `validate_proposals(proposals, nodes)` | Returns `(valid, skipped)`. Skips if antecedents/unless reference missing nodes, ID already exists, or ID is similar to a retracted belief. |
| `apply_proposals(valid, db_path)` | Calls `api.add_node()` per proposal with `source_type="derived"`. Catches exceptions per-item; returns `(proposal, result|error_string)` tuples. |
| `write_proposals_file(valid, output_path)` | Writes a markdown file in the same `### DERIVE` / `### GATE` format that `parse_proposals` reads, so `reasons accept` can round-trip it. |

## 3. Patterns & Conventions

- **Pipeline / staged pipeline**: `build_prompt → [LLM call elsewhere] → parse_proposals → validate_proposals → apply_proposals`. Each stage is a pure-ish function with a clear input/output contract, making the pipeline testable in isolation (see `test_derive.py`).

- **Strategy selection by flags**: `_build_beliefs_section` branches on `sample`, `cluster`, `intra_cluster` to pick different selection strategies. The caller passes a flat set of boolean/int kwargs rather than passing a strategy object — practical for a CLI-oriented codebase.

- **Memoized recursion with cycle guard**: `_get_depth` sets `memo[node_id] = 0` *before* recursing into antecedents. This prevents infinite recursion if the justification graph has a cycle (which shouldn't happen in a well-formed RMS, but the guard is cheap).

- **Backward-compatible parsing**: `parse_proposals` tries two regex patterns. The new format (`### DERIVE belief-id`) is tried first; the old format (`### DERIVE: \`belief-id\``) is the fallback. This is a pragmatic migration strategy — the v0.9 format is never emitted by the current `DERIVE_PROMPT`, but old LLM responses or user-edited files might still use it.

- **Proportional budget allocation**: When multiple agents are present, the belief budget is split proportionally (`max_beliefs * len(agent_beliefs) / total_all`), with a floor of 5 per agent. This prevents one large agent from starving smaller ones.

- **Template with explicit placeholder documentation**: The `KeyError` catch in `build_prompt` lists all valid placeholders, turning a silent template bug into an actionable error message.

## 4. Dependencies

### This file imports
- **`api`** (sibling module) — `apply_proposals` calls `api.add_node()`. This is the only runtime dependency on the broader package.
- **`.cluster`** (sibling module) — lazily imported inside `_build_beliefs_section` when `cluster=True` or `intra_cluster=True`. This avoids pulling in sentence-transformers unless clustering is actually requested.
- **Stdlib**: `random` (sampling), `re` (parsing), `defaultdict` (grouping). Note: `subprocess`, `sys`, `Path` are imported but **unused** in this file — likely leftover from a prior version that shelled out or wrote files directly.

### This file is imported by
- **`reasons/api.py`** — the high-level API surface calls `build_prompt` and the pipeline functions.
- **`reasons/cli.py`** — the `reasons derive` CLI command.
- **`reasons_lib/derive.py`** — a duplicate/shim copy (the repo has a parallel `reasons_lib/` tree; see the directory listing).
- **Tests**: `test_derive.py`, `test_derive_budget.py`, `test_source_type.py`.

## 5. Flow (Control & Data)

```
build_prompt(nodes, domain, topic, budget, ...)
│
├─ _filter_by_topic(nodes, topic)          # narrow by keyword
├─ compute all_derived, max_depth          # from full graph pre-filter
├─ filter by premises_only / has_dependents / min_depth / max_depth_filter
├─ _detect_agents(nodes)                   # split on ":"
├─ _build_beliefs_section(...)             # sample / cluster / group
│    └─ (optional) cluster_beliefs / cluster_beliefs_intra
├─ _build_derived_section(...)             # sort by depth desc
├─ template.format(...)                    # fill DERIVE_PROMPT
│    └─ KeyError → ValueError with placeholder list
└─ return (prompt, stats)

[LLM call happens in api.py or cli.py — not in this file]

parse_proposals(llm_response)
│
├─ try new_pattern regex  → proposals?  → return
└─ try old_pattern regex  → proposals?  → return

validate_proposals(proposals, nodes)
│
├─ for each proposal:
│   ├─ check all antecedents ∈ nodes
│   ├─ check all unless     ∈ nodes
│   ├─ check id ∉ nodes
│   └─ find_similar_out(id) → skip if Jaccard ≥ 0.5
└─ return (valid, skipped)

apply_proposals(valid)
│
└─ for each proposal:
    └─ api.add_node(id, text, sl, unless, label,
                    source_type="derived", any_mode=(mode=="any"))
       catch Exception → record error string

write_proposals_file(valid, path)          # alternative to apply
└─ write markdown in ### DERIVE / ### GATE format
```

## 6. Invariants

- **Antecedent existence**: `validate_proposals` rejects any proposal whose `antecedents` or `unless` list contains a node ID not present in the network. The LLM is instructed to only use existing IDs, but validation is the actual enforcement point.
- **No ID collision**: A proposal whose `id` already exists in `nodes` is skipped. This prevents duplicate belief creation.
- **Retraction awareness**: `find_similar_out` uses token-level Jaccard similarity (threshold 0.5) to catch the case where the LLM re-proposes a belief that was previously retracted (OUT). This is a soft check — it only looks at the *ID*, not the text.
- **ALL vs. ANY semantics**: The `mode` field controls how `api.add_node` wires the justification. `ALL` means retracting any single antecedent retracts the conclusion; `ANY` means the conclusion survives as long as at least one antecedent holds. The default is `ALL` when the field is absent.
- **Minimum antecedents**: The prompt instructs the LLM to use ≥ 2 antecedents, but `validate_proposals` does **not** enforce this. It's a soft constraint on the LLM, not a hard invariant in code.
- **Depth computation is acyclic by convention**: The cycle guard in `_get_depth` is defensive; a well-formed RMS justification graph should be a DAG.

## 7. Error Handling

| Location | Strategy |
|---|---|
| `_get_depth` | Cycle guard via early memo write. No exception raised. |
| `build_prompt` template formatting | Catches `KeyError` (unknown placeholder) and `ValueError` (malformed braces in a custom template). Both re-raise as `ValueError` with a human-readable message listing valid placeholders. |
| `validate_proposals` | Does **not** raise. Returns a `(valid, skipped)` tuple; each skipped entry carries a reason string. The caller decides what to do (log, report, etc.). |
| `apply_proposals` | Wraps each `api.add_node()` call in `try/except Exception`. Failures are captured as a string in the results list rather than aborting the batch. A single bad proposal doesn't prevent the rest from being applied. |
| `parse_proposals` | If neither regex matches, returns an empty list. No exception. The caller is expected to handle "LLM produced no parseable proposals." |
| Unused imports (`subprocess`, `sys`, `Path`) | Dead code. Safe to remove but currently harmless. |

---

## Topics to Explore

- [file] `reasons/cluster.py` — Implements `cluster_beliefs` and `cluster_beliefs_intra`, the semantic-clustering backends that `_build_beliefs_section` delegates to. Understanding how embeddings and cluster rotation work is essential to the `intra_cluster` multi-round mode.
- [function] `reasons/api.py:add_node` — The persistence target for every applied proposal. You need to understand its `sl`, `unless`, `any_mode`, and `source_type` parameters to fully reason about what `apply_proposals` actually writes to the database.
- [file] `tests/test_derive.py` and `tests/test_derive_budget.py` — The test suite that pins down the prompt-building contract, budget allocation math, sampling determinism, and parsing edge cases. Reading these alongside the source reveals the intended invariants that aren't always enforced in code.
- [general] `reasons-vs-reasons-lib-duplication` — The repo contains a parallel `reasons_lib/` tree with a `derive.py` of its own. Understanding whether this is a vendored copy, a build artifact, or a maintenance concern affects how you approach refactoring either file.
- [file] `reasons/cli.py` — The `reasons derive` command wires together `build_prompt`, the LLM call, `parse_proposals`, `validate_proposals`, and the `apply`/`write` decision. This is where the full pipeline is orchestrated and where user-facing flags map to `build_prompt` parameters.

---

## Beliefs

- `derive-validate-requires-existence` — `validate_proposals` rejects any proposal whose `antecedents` or `unless` list references a node ID absent from the `nodes` dict; this is the sole enforcement of the "use existing IDs" rule, not the LLM prompt.
- `derive-retractions-block-reproposal` — A proposed belief ID that has ≥ 50 % token overlap (Jaccard) with any OUT belief's ID is skipped, preventing the LLM from re-deriving a previously retracted conclusion.
- `derive-mode-defaults-to-all` — When the LLM omits the `Mode:` field or it fails to parse, `parse_proposals` defaults `mode` to `"all"`, meaning the justification is conjunctive (retracting any antecedent retracts the conclusion).
- `derive-cluster-lazy-import` — The `cluster` module is imported inside `_build_beliefs_section` only when `cluster=True` or `intra_cluster=True`, so sentence-transformers is never loaded unless semantic clustering is explicitly requested.
- `derive-write-roundtrips-parse` — `write_proposals_file` emits the exact `### DERIVE` / `### GATE` markdown format that `parse_proposals` reads, enabling a `reasons accept file.md` round-trip without re-invoking the LLM.
