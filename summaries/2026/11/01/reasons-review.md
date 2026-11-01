---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/review.py

# `reasons/review.py` — Semantic Belief Review

## Purpose

This file is the **semantic validation layer** of a Truth Maintenance System (TMS). The TMS elsewhere in this repo guarantees *structural* validity: antecedents exist, they're IN, justifications are well-formed. But a derivation can be structurally perfect and still be a non-sequitur. This module closes that gap by feeding each derived belief (with its full justification chain) to an LLM and asking: *does the conclusion actually follow from the premises?*

It owns three specific audit axes:
- **Valid** — is the inference logically sound?
- **Sufficient** — do the listed antecedents fully support the conclusion, or are hidden assumptions at play?
- **Necessary** — are all antecedents load-bearing, or is some redundant?

The output is structured verdicts (including classified failure modes like "over-generalizes" or "scope-mismatch") that downstream modules (`repair.py`, `review_justifications.py`, `api.py`) use to trigger defeaters, repairs, or user-facing reports.

## Key Components

### Constants

| Name | Value / Shape | Role |
|---|---|---|
| `REVIEW_BATCH_SIZE` | `20` | Max beliefs per LLM call. Keeps prompt size manageable and limits blast radius of a bad response. |
| `REVIEW_PROMPT` | Multi-line string with `{beliefs}` placeholder | The primary audit prompt. Instructs the LLM to return a JSON array and defines the full schema including `scope_findings`, `missing_property`, and `defeat_reason_type`. |
| `DEFEAT_REASON_TYPES` | 7-element `list[str]` | Closed vocabulary of logical failure modes. Used both as the LLM's allowed output set and as the validation set in `classify_defeat_reason`. |
| `CLASSIFY_DEFEAT_REASON_PROMPT` | String with `{defeater_text}` and `{defeated_text}` | A focused, single-purpose prompt for post-hoc classification when a defeat verdict already exists but lacks a structured reason type. |

### Functions

**`classify_defeat_reason(defeater_text, defeated_text, model, timeout) -> str`**
- **Contract**: Takes the human-readable defeater text and the defeated belief text. Returns exactly one string from `DEFEAT_REASON_TYPES`, or `""` on any failure (LLM error, unparseable response).
- **Implementation detail**: Does a two-pass match — first an exact `in` check on the stripped/lowered result, then a substring scan across the 7 types. This is defensive against the LLM returning something like `"scope-mismatch (antecedents only cover X)"`.

**`format_belief_for_review(node_id, nodes) -> str`**
- **Contract**: Takes a single belief ID and the full `nodes` dict (from `export_network()`). Returns a markdown-formatted block with the claim, every justification (antecedents + outlist + label).
- **Edge case**: If an antecedent or outlist ID is not present in `nodes`, it emits `"(not found in network)"` rather than raising. This lets the LLM reason about *missing* context as a potential insufficiency signal.
- Multiple justifications get a `Justification i/n:` header; a single justification does not.

**`parse_review_response(response) -> list[dict]`**
- **Contract**: Extracts the JSON array from arbitrary LLM output. Returns a list of normalized result dicts, or `[]` if no valid array is found.
- **Implementation detail**: Uses `json.JSONDecoder.raw_decode` scanning forward from every `[` character. This handles the common LLM pattern of preamble text ("Here are my findings:") before the array and trailing commentary after it. It also skips any array element that isn't a dict with an `"id"` key, and applies **defaults of `True`** for `valid`, `sufficient`, `necessary` when those keys are absent — a conservative bias toward "the derivation is fine" if the LLM omits a field.

**`review_beliefs(nodes, belief_ids=None, model="claude", timeout=300, batch_size=REVIEW_BATCH_SIZE, on_batch=None) -> list[dict]`**
- **Contract**: The main entry point. Filters to IN beliefs with non-empty justifications, batches them, calls the LLM, and returns a flat list of all review result dicts.
- **`on_batch` callback**: After each batch's results are accumulated, `on_batch(all_results)` is invoked. This lets callers (e.g., `repair.py`) react incrementally without waiting for the full review to complete — useful for long reviews where early defeats should trigger early repairs.
- **Filtering logic** (when `belief_ids` is `None`): requires `truth_value == "IN"` **and** a non-empty `justifications` list. This means OUT beliefs, IN beliefs with no justifications (axioms), and beliefs with `justifications: []` are all excluded.
- **When `belief_ids` is provided**: the filter is looser — it only checks membership in `nodes` and non-empty justifications. It does *not* require `truth_value == "IN"`, so you can review OUT or neutral beliefs if you explicitly ask.

## Patterns

1. **LLM-as-judge with structured output.** The prompt explicitly demands a JSON array and defines the schema inline. `parse_review_response` is a lenient extractor rather than a strict validator — it tolerates prose around the JSON and missing fields.

2. **Batch-and-callback processing.** `review_beliefs` iterates in fixed-size batches (default 20), calls the LLM per batch, and fires `on_batch` after each. This is a simple producer-consumer pattern without threads or async — the caller drives the loop.

3. **Defensive JSON parsing.** Rather than `json.loads(response)` (which would choke on any preamble), the code scans for `[` and tries `raw_decode` at each position. This is a pragmatic response to LLM output that is *mostly* correct but wrapped in explanatory text.

4. **Closed-vocabulary classification.** `DEFEAT_REASON_TYPES` is a fixed list of 7 strings. Both the review prompt and the classify prompt reference this exact list, and `classify_defeat_reason` validates against it. This keeps the downstream graph (defeater nodes) using a finite set of failure-mode tags rather than free-text.

5. **Mirror-module pattern.** The repo has `reasons/` and `reasons_lib/` as parallel packages. `reasons_lib/review.py` is a mirror of this file. Changes here must be mirrored there (or the duplication is intentional for distribution vs. monorepo use).

## Dependencies

### This file imports
- **`json`** — for `JSONDecoder.raw_decode` in the parser.
- **`sys`** — for `sys.stderr` progress/warning messages.
- **`.llm.invoke_model`** — the single LLM call primitive. This is the only external service dependency. All LLM interaction funnels through this one function, which abstracts model selection, auth, and transport.

### This file is imported by
- **`reasons/api.py`** — exposes review as an API endpoint.
- **`reasons/repair.py`** — consumes review verdicts to trigger repair actions (defeating invalid beliefs, removing unnecessary antecedents).
- **`reasons/review_justifications.py`** — a sibling module that reviews at the justification level; likely shares formatting and parsing utilities.
- **`reasons_lib/*`** — the mirror package.
- **`tests/test_review.py`, `tests/test_classify_defeaters.py`, `tests/test_repair_premises.py`, `tests/test_review_justifications.py`, `tests/test_review_premises.py`** — test coverage for the review flow and the defeat-reason classifier.

## Flow

```
review_beliefs(nodes, belief_ids?, ...)
│
├─ 1. FILTER
│     If belief_ids is None → pick all IN beliefs with ≥1 justification
│     If belief_ids given   → filter to those present in nodes with justifications
│     If empty → return []
│
├─ 2. BATCH LOOP  (step = batch_size, default 20)
│     │
│     ├─ format_belief_for_review(nid, nodes)   for each nid in batch
│     │     └─ builds "### id / Claim / Antecedents / Unless / Label" text
│     │
│     ├─ REVIEW_PROMPT.format(beliefs=beliefs_text)
│     │
│     ├─ invoke_model(prompt, model, timeout)   ← LLM call
│     │
│     ├─ parse_review_response(response)
│     │     └─ scans for '[' → raw_decode → normalizes each item
│     │
│     ├─ all_results.extend(results)
│     │
│     └─ on_batch(all_results)   (if callback provided)
│
└─ 3. RETURN all_results
```

Separate flow for **`classify_defeat_reason`** (used post-hoc, not in the batch loop):

```
classify_defeat_reason(defeater_text, defeated_text, model, timeout)
├─ CLASSIFY_DEFEAT_REASON_PROMPT.format(...)
├─ invoke_model(...)
├─ strip → lower → exact match in DEFEAT_REASON_TYPES?
├─ if not → substring scan across DEFEAT_REASON_TYPES
└─ return matched type or ""
```

## Invariants

- **Only beliefs with non-empty `justifications` are reviewed.** A belief with `justifications: []` (an axiom or an unexplained assertion) is silently skipped, even if it's IN. This is by design: there's no derivation chain to audit.
- **The `belief_ids` override path does NOT enforce `truth_value == "IN"`.** If you pass explicit IDs, OUT beliefs will be reviewed too. The default path is stricter.
- **`parse_review_response` defaults missing `valid`/`sufficient`/`necessary` to `True`.** A belief the LLM didn't explicitly flag is considered fine. This is a fail-open bias.
- **`DEFEAT_REASON_TYPES` is a closed set of exactly 7 values.** Any code that matches on `defeat_reason_type` can do an exhaustive `if/elif` chain over these 7 (plus `""` for "unclassified").
- **Results preserve LLM ordering.** The parser does not re-sort. The prompt instructs "same order as presented," and the batch loop preserves the order of `belief_ids`.
- **`REVIEW_BATCH_SIZE` is a module-level constant, not a config file value.** Changing it requires editing this file (or passing `batch_size` explicitly to `review_beliefs`).

## Error Handling

The error strategy is **per-batch isolation with stderr logging**:

| Where | What happens | Effect |
|---|---|---|
| `review_beliefs` — `invoke_model` raises | Caught by `except Exception as e`, logged to `sys.stderr` as `"WARN: batch N failed: {e}"` | That batch's beliefs are silently skipped. Remaining batches still execute. `all_results` is missing those entries. |
| `review_beliefs` — `parse_review_response` returns `[]` | No exception; `extend([])` is a no-op | Same as a failed batch, but no warning is logged. The caller can't distinguish "LLM said nothing" from "LLM said 'all valid'" without checking result count. |
| `classify_defeat_reason` — `invoke_model` raises | Caught, logged to stderr, returns `""` | Downstream code treats `""` as "unclassified." No crash. |
| `classify_defeat_reason` — LLM returns garbage | Not in `DEFEAT_REASON_TYPES` and no substring match → returns `""` | Same graceful degradation. |
| `format_belief_for_review` — antecedent not in `nodes` | Emits `"(not found in network)"` in the text | The LLM sees the gap and may flag it as an insufficiency. No exception. |

Notably, **there is no retry logic**. A transient LLM timeout kills that batch permanently. If the caller needs resilience, they must wrap `review_beliefs` in their own retry loop.

The `on_batch` callback is called *before* the try/except's `except` block, so a callback that raises will propagate up through `review_beliefs` and interrupt the remaining batches. Callers should ensure `on_batch` is exception-safe.

## Topics to Explore

- [file] `reasons/repair.py` — Consumes review verdicts to defeat invalid beliefs and remove unnecessary antecedents; shows how `valid: false` + `defeat_reason_type` drives graph mutations
- [function] `reasons/llm.py:invoke_model` — The single LLM call primitive that both `review_beliefs` and `classify_defeat_reason` depend on; understand its timeout, model-routing, and error semantics
- [file] `reasons/review_justifications.py` — Sibling module that reviews at the justification (edge) level rather than the belief (node) level; likely shares or diverges from the patterns here
- [file] `tests/test_review.py` — Test fixtures showing realistic `nodes` dicts, mock LLM responses, and edge cases (empty justifications, missing antecedents, malformed JSON)
- [general] `reasons-vs-reasons-lib-duplication` — The `reasons/` and `reasons_lib/` packages are near-identical mirrors; understand whether they're kept in sync manually, via a build step, or by one importing the other

## Beliefs

- `review-beliefs-skips-axioms` — Beliefs with `justifications: []` (empty list) are excluded from review in the default path, even if their `truth_value` is `"IN"`; only beliefs with at least one justification are audited
- `parse-review-response-fails-open` — When the LLM omits `valid`, `sufficient`, or `necessary` from a result object, `parse_review_response` defaults all three to `True`, meaning an unflagged belief is treated as passing review
- `defeat-reason-types-is-closed-7-value-set` — `DEFEAT_REASON_TYPES` contains exactly 7 strings and is the sole valid vocabulary for the `defeat_reason_type` field; `classify_defeat_reason` returns `""` (not an exception) for any unmatchable input
- `review-beliefs-batch-isolation` — An exception in one batch's `invoke_model` call is caught and logged to stderr, allowing subsequent batches to proceed; the failed batch's beliefs are simply absent from the returned results
- `belief-ids-override-bypasses-in-filter` — When `belief_ids` is explicitly provided to `review_beliefs`, the filter checks only `nodes` membership and non-empty justifications, dropping the `truth_value == "IN"` requirement that the default path enforces
