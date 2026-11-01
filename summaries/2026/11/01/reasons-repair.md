---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/repair.py

# `reasons/repair.py` — Repairing Invalid Beliefs in a TMS

## 1. Purpose

This module is the **remediation engine** for a Truth Maintenance System. When `review-beliefs` flags a derived belief as invalid (its conclusion smuggles in a factual claim not supported by its antecedents), this file decides *how* to fix it and then executes that fix.

It owns three (really four) repair strategies:

| Pattern | When to use | Action |
|---|---|---|
| `search_and_link` | Belief is sound but missing an antecedent | Find a premise elsewhere in the network, wire it in |
| `soften` | Belief overstates the evidence | Rewrite the text to match what antecedents actually support |
| `abandon` | Dependency chain is too broken | Retract the belief entirely |
| `research` | Plausible but unconfirmable | Tag it for human follow-up, leave it in place |

The entry point `repair_beliefs` triages each invalid belief via LLM, then dispatches to the appropriate pattern. There's also `repair_smuggled_beliefs`, which is the older, narrower version that *only* does search-and-link (no triage).

## 2. Key Components

### Prompt Templates (module-level constants)

- **`EXTRACT_PROMPT`** — Asks the LLM to identify the *specific* factual claim the conclusion assumes but the antecedents don't provide. The response is a single sentence, not JSON.
- **`MATCH_PROMPT`** — Given the smuggled claim and a list of candidate premises, asks the LLM to return a JSON object `{"matched_ids": [...], "rationale": "..."}`.
- **`TRIAGE_PROMPT`** — The decision router. Gives the LLM the belief context, review comment, derivation depth, and count of flagged ancestors. Returns `{"pattern": "...", "rationale": "..."}`.
- **`SOFTEN_PROMPT`** — Rewrites an overstated belief. Returns `{"softened_text": "...", "rationale": "..."}`.

### `VALID_PATTERNS` (line ~130)

```python
VALID_PATTERNS = {"search_and_link", "soften", "abandon", "research"}
```

The whitelist used by `parse_triage_response` to reject malformed LLM output.

### Parser Functions

All four parsers follow the same idiom: **scan for `{`, attempt `json.JSONDecoder().raw_decode()`, validate shape, return a dict or a safe default.**

| Function | Input | Returns on success | Returns on failure |
|---|---|---|---|
| `parse_extract_response` | raw LLM text | stripped claim string (peels off `"Smuggled claim:"` prefix, strips quotes) | `""` |
| `parse_match_response` | raw LLM text + `valid_ids` set | `{"matched_ids": [filtered], "rationale": str}` | `{"matched_ids": [], "rationale": ""}` |
| `parse_triage_response` | raw LLM text | `{"pattern": valid_pattern, "rationale": str}` | `{"pattern": "", "rationale": ""}` |
| `parse_soften_response` | raw LLM text | `{"softened_text": str, "rationale": str}` | `{"softened_text": "", "rationale": ""}` |

The `raw_decode` scanning pattern is critical: LLM responses often have prose *before* or *after* the JSON blob, so `json.loads()` on the full string would fail. These parsers find the first `{` and decode from there.

### LLM Call Wrappers

Each is a thin "prompt → invoke → parse" trio:

- **`extract_smuggled_claim(belief_context, review_comment, ...)`** → `str`
- **`find_matching_premises(smuggled_claim, candidates, ...)`** → `dict`
- **`triage_belief(belief_context, review_comment, depth, flagged_ancestors, ...)`** → `dict`
- **`soften_belief(belief_text, antecedents_text, ...)`** → `dict`

All accept `model="claude"` and `timeout=300` as defaults, and all delegate to `invoke_model` from `reasons/llm.py`.

### Helper Functions

- **`_compute_depth(node_id, nodes, memo)`** — Memoized recursive depth calculation. A premise (no justifications) has depth 0. A derived belief has `max(antecedent_depths) + 1`. The `memo` dict is shared across calls within one `repair_beliefs` invocation to avoid recomputation.

- **`_format_antecedents(node, nodes)`** — Flattens all antecedent texts into a bulleted list for the `SOFTEN_PROMPT`.

- **`_do_search_and_link(...)`** — Extracts the full search-and-link pipeline into a reusable function so both `repair_beliefs` and `repair_smuggled_beliefs` can share it. Returns a 4-tuple: `(status, claim, matched_ids, rationale)`.

### Main Orchestrators

- **`repair_beliefs(review_results, nodes, ...)`** — The primary entry point. Filters to invalid beliefs, triages each via LLM, dispatches to the appropriate pattern, and returns a list of result dicts. Also aliased as `research_beliefs` (line ~340).

- **`repair_smuggled_beliefs(review_results, nodes, ...)`** — The legacy path. Skips triage entirely; assumes every invalid belief needs search-and-link. Largely superseded by `repair_beliefs` but kept for backward compatibility (imported by `reasons_lib/repair.py` and `reasons_lib/repair_premises.py`).

## 3. Patterns & Idioms

**Strategy dispatch via LLM triage.** The four repair patterns are not a Python-level strategy pattern with a class hierarchy — instead, the LLM *is* the dispatcher. `TRIAGE_PROMPT` encodes the decision logic in natural language, and `parse_triage_response` validates the output against `VALID_PATTERNS`. The `if/elif` chain in `repair_beliefs` (lines ~290–340) is the concrete dispatch.

**Dependency injection for testability.** `search_fn` is a parameter on both orchestrators. In production it defaults to `lambda query, **kw: api.search(query, include_out=True, **kw)`. In tests (see `tests/test_repair.py`) you inject a fake that returns canned JSON, avoiding real DB/LLM calls.

**Dry-run mode.** Every mutation (add_justification, supersede_with_text, retract_node) is guarded by `if not dry_run:`. In dry-run mode, the result dict still reports *what would happen*, but nothing is written.

**LLM-as-JSON-producer with tolerant parsing.** All four prompts ask for JSON, but the parsers use `raw_decode` scanning rather than `json.loads()`. This is a deliberate defensive choice: LLMs frequently emit `"Here's the result: {...}"` or trailing commentary.

**Memoized depth computation.** `_compute_depth` takes an optional `memo` dict. In `repair_beliefs`, a single `depth_memo = {}` is created and passed to every call, so the transitive closure of antecedents is only walked once per unique node.

**`api` imported lazily inside functions.** `from . import api` appears inside `_do_search_and_link`, `repair_beliefs`, and `repair_smuggled_beliefs` rather than at module top level. This avoids circular imports (since `api.py` imports from `repair.py` per the "Imported By" list).

## 4. Dependencies

### This file imports

| Dependency | What it's used for |
|---|---|
| `json` | Parsing LLM responses, deserializing search results |
| `sys` | `print(..., file=sys.stderr)` for progress logging |
| `.llm.invoke_model` | The actual LLM call (all four prompts go through this) |
| `.review.format_belief_for_review` | Renders a belief + its antecedents into a readable context string for prompts |
| `.api` (lazy) | `add_justification`, `supersede_with_text`, `retract_node`, `set_metadata`, `search` |

### This file is imported by

- **`reasons/api.py`** — the API layer calls `repair_beliefs` / `repair_smuggled_beliefs` as endpoints
- **`reasons_lib/repair.py`** and **`reasons_lib/repair_premises.py`** — the "lib" mirror copies (this repo appears to maintain parallel `reasons/` and `reasons_lib/` trees)
- **`tests/test_repair.py`**, **`tests/test_repair_premises.py`**, **`tests/test_research.py`** — test suites

## 5. Execution Flow

Here's the path through `repair_beliefs` for a single belief:

```
repair_beliefs(review_results, nodes, ...)
│
├─ Filter: invalid = [r for r in review_results if not r.valid]
├─ Build: invalid_ids set, depth_memo = {}
│
└─ For each invalid belief:
   │
   ├─ 1. LOOKUP: node = nodes[belief_id]
   │     belief_context = format_belief_for_review(belief_id, nodes)
   │     depth = _compute_depth(belief_id, nodes, depth_memo)
   │     flagged_ancestors = count of antecedents in invalid_ids
   │
   ├─ 2. TRIAGE (LLM call):
   │     triage_belief(belief_context, comment, depth, flagged_ancestors)
   │     → invoke_model(TRIAGE_PROMPT)
   │     → parse_triage_response → {"pattern": "...", "rationale": "..."}
   │
   └─ 3. DISPATCH by pattern:
       │
       ├─ search_and_link:
       │   ├─ extract_smuggled_claim (LLM) → claim string
       │   ├─ search_fn(claim) → search_results
       │   ├─ Filter candidates: exclude self, existing ants, OUT nodes, derived nodes
       │   ├─ find_matching_premises (LLM) → matched_ids
       │   └─ api.add_justification(belief_id, new_ants)
       │
       ├─ soften:
       │   ├─ _format_antecedents → antecedent text
       │   ├─ soften_belief (LLM) → softened_text
       │   └─ api.supersede_with_text(belief_id, softened) → new_id
       │
       ├─ abandon:
       │   └─ api.retract_node(belief_id, reason=...)
       │
       └─ research:
           └─ api.set_metadata(belief_id, "repair_research", rationale)
```

Each branch appends a result dict to `results` and the function returns the full list.

## 6. Invariants & Validation

1. **Candidate filtering (search-and-link path).** A node is only a valid link target if: it's not the belief itself, not already an antecedent, has `truth_value == "IN"`, and has **no justifications** (i.e., it's a premise, not a derived belief). This is enforced in both `_do_search_and_link` (lines ~220–230) and `repair_smuggled_beliefs` (lines ~380–390).

2. **`matched_ids` are validated against the actual candidate set.** `parse_match_response` filters `matched_ids` through `valid_ids = {c["id"] for c in candidates}`. If the LLM hallucinates an ID that wasn't in the candidate list, it's silently dropped.

3. **Triage pattern must be in `VALID_PATTERNS`.** `parse_triage_response` rejects any `pattern` value not in the four-element set. An empty pattern means triage failed, and the belief is skipped with `status: "triage_failed"`.

4. **Depth semantics.** Depth 0 = premise (no justifications). Depth N = derived belief whose deepest antecedent chain has length N. This is a structural invariant of the TMS graph.

5. **`repair_beliefs` is aliased as `research_beliefs`.** Any code importing `research_beliefs` gets the full four-pattern orchestrator, not a research-only subset.

## 7. Error Handling

The error-handling philosophy is **"never crash the batch; capture per-belief failures in the result dict."**

- **LLM call failures** (network error, timeout): `invoke_model` raises, which is caught by the outer `try/except Exception as exc` in the per-belief loop. The result dict gets `result["error"] = str(exc)` and the loop continues to the next belief.

- **Malformed LLM JSON**: The `raw_decode` scanning pattern means if *no* JSON object is found, the parser returns a safe default (empty strings, empty lists). For triage, this means `pattern == ""`, which hits the `if not pattern:` guard and produces `status: "triage_failed"`.

- **Search result parsing**: `json.loads(raw)` is wrapped in `try/except (json.JSONDecodeError, TypeError)`, falling back to `search_results = []`.

- **Belief not in network**: `nodes.get(belief_id)` returns `None` → `result["error"] = "belief not found in network"`, `continue`.

- **No candidates / no match**: These are *expected* outcomes, not errors. They produce distinct status codes (`"no_candidates"`, `"no_match"`, `"extraction_failed"`, `"soften_failed"`) so callers can differentiate "nothing to do" from "something broke."

- **Progress logging goes to `stderr`** via `print(..., file=sys.stderr)`, keeping `stdout` clean for result JSON.

---

## Topics to Explore

- [file] `reasons/review.py` — Defines `format_belief_for_review` and the review pipeline that *produces* the `review_results` this module consumes; understanding the review output contract is essential for writing correct prompts.
- [file] `reasons/api.py` — The API layer that imports and exposes `repair_beliefs`/`repair_smuggled_beliefs`; shows how the repair results are serialized and returned to clients, and reveals the circular-import constraint that forces lazy `from . import api`.
- [file] `tests/test_repair.py` — Demonstrates the `search_fn` injection pattern, dry-run assertions, and the expected shape of result dicts for each pattern; the fastest way to understand the behavioral contract.
- [function] `reasons/llm.py:invoke_model` — The single LLM call boundary all four prompts flow through; understanding its timeout/retry/error semantics tells you what "LLM call failure" actually looks like in production.
- [file] `reasons/repair_premises.py` — The sibling module for repairing *premise* nodes (as opposed to derived beliefs); shares parsing idioms and likely has overlapping candidate-filtering logic worth comparing.

## Beliefs

- `repair-beliefs-triages-via-llm` — `repair_beliefs` does not apply a fixed repair strategy; it calls `triage_belief` (an LLM prompt) to select one of four patterns, and only then dispatches.
- `search-and-link-requires-premises` — The search-and-link path exclusively links *premise* nodes (nodes with no justifications and `truth_value == "IN"`); derived beliefs are filtered out of candidates in both `_do_search_and_link` and `repair_smuggled_beliefs`.
- `matched-ids-filtered-against-candidates` — `parse_match_response` silently drops any LLM-returned ID not present in the `valid_ids` set, preventing the LLM from hallucinating a link to a nonexistent or irrelevant node.
- `dry-run-mutates-nothing` — When `dry_run=True`, no calls to `api.add_justification`, `api.supersede_with_text`, or `api.retract_node` are made; the result dicts report the intended action but the TMS graph is unchanged.
- `research-beliefs-is-alias-for-repair-beliefs` — `research_beliefs` is a direct alias (`research_beliefs = repair_beliefs`), not a separate code path; importing either name gives the full four-pattern orchestrator.
