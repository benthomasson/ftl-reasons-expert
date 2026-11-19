---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/review.py

# `reasons/review.py` — Semantic Review of Derived Beliefs

## Purpose

This module is the **semantic audits layer** over the Truth Maintenance System (TMS). The TMS engine (`verify.py`, `check_stale.py`, `derive.py`) already validates *structure* — that antecedents exist, justifications point to real nodes, and truth values propagate. What it cannot do is judge whether a derivation is *logically sound*. That is the job of this file: it feeds each derived belief plus its full justification chain (antecedent texts, outlists, labels) back to an LLM and asks three questions — is the conclusion **valid**, is the support **sufficient**, and is every antecedent **necessary**?

It's also the home of the shared `DEFEAT_REASON_TYPES` taxonomy and the single-shot `classify_defeat_reason` helper that later turns a rendered defeater into one of those canonical categories.

## Key Components

### Constants

- **`REVIEW_BATCH_SIZE = 20`** — Default beliefs per LLM call. Keeps prompts within context limits and limits blast radius of a single bad model response.
- **`REVIEW_PROMPT`** — A large templated prompt (`str.format`, note the `{{`/`}}` escapes for literal JSON braces). It:
  - Explains the TMS context to the model.
  - Defines the three evaluation axes (Valid / Sufficient / Necessary).
  - Specifies the **exact JSON array output shape** including structured defeater fields: `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`, `scope_findings`, `missing_property`, `defeat_reason_type`.
  - Contains rules: one object per belief *in the same order*, "valid if ANY justification is sound" (because justifications are alternative support paths), rigor caveats.
  - Requires that when `valid` is `false`, `scope_findings` and `missing_property` **must** be populated, and `defeat_reason_type` must be one of the enumerated types. This is what enables downstream construction of *auditable* structured defeaters in the graph.
- **`DEFEAT_REASON_TYPES`** — The canonical 7-value enumeration: `unsupported-conjunct`, `over-generalizes`, `false-causal-claim`, `internal-contradiction`, `circular-reasoning`, `missing-bridge`, `scope-mismatch`.
- **`CLASSIFY_DEFEAT_REASON_PROMPT`** — A locked-down single-purpose prompt that asks for exactly one type label given a defeater text and defeated belief text.

### Functions

**`classify_defeat_reason(defeater_text, defeated_text, model, timeout) -> str`**
- Renders `CLASSIFY_DEFEAT_REASON_PROMPT` and calls `invoke_model`.
- Normalizes with `.strip().lower()`.
- Resolution order: exact membership in `DEFEAT_REASON_TYPES`, then substring search, else `""`. So this is **tolerant** — it will happily extract a type label from chatty output.
- Returns `""` (falsy) on LLM failure or unrecognized output.

**`format_belief_for_review(node_id, nodes) -> str`**
- Renders one belief section: `### <id>`, `Claim: <text>`, then one block per justification.
- Emits `Justification i/N:` only when `len(justs) > 1`.
- Antecedents are rendered as bullet lines with `- <id>: <text>`; **missing antecedent nodes render as `(not found in network)`** rather than being dropped — deliberately keeping the ID visible to the model.
- `outlist` entries render as `Unless (must be OUT):` — this reflects TMS semantics where an outlist element must be OUT for the justification to hold.
- Returns `""` for unknown `node_id`.

**`parse_review_response(response) -> list[dict]`**
- Uses `json.JSONDecoder.raw_decode` starting at *each* `[` character until one parses cleanly. This handles preamble prose, multiple bracket-like artifacts, and trailing text.
- For every item in the parsed list: skip non-dicts or dicts without `id`; then **normalize** into a fixed-shape record with **permissive defaults**: `valid=True`, `sufficient=True`, `necessary=True`, `unnecessary_antecedents=[]`, `comment=""`, `scope_findings=[]`, `missing_property=""`, `defeat_reason_type=""`.
- Returns the first array that yields at least one valid item; otherwise `[]`.

**`review_beliefs(nodes, belief_ids=None, model="claude", timeout=300, batch_size=REVIEW_BATCH_SIZE, on_batch=None) -> list[dict]`**
- **Target selection**:
  - `belief_ids is None` → all nodes with `truth_value == "IN"` **and** non-empty justifications, iterated in `sorted(nodes.items())` order.
  - Explicit list → filtered to IDs that exist in `nodes` and have justifications. Unknown or unjustified IDs are silently dropped.
- **Batching**: chunks `belief_ids` into `batch_size` (default 20) groups; prints progress to `stderr` per batch.
- For each batch: join `format_belief_for_review` outputs, `REVIEW_PROMPT.format(beliefs=...)`, call `invoke_model`, `parse_review_response`, extend `all_results`, and if `on_batch` is supplied, invoke it with the *cumulative* `all_results`.
- Returns concatenated results across all batches.

## Patterns

1. **Batch-and-callback**: Uniform batching (mirrors many other `reasons/*` modules like `import_beliefs.py`, `cluster.py`) with an `on_batch` hook so callers can stream progress or persist partial results.
2. **Tolerant JSON extraction**: `raw_decode` at each `[` is the project idiom for parsing LLM output that may contain prose framing. See also `reasons/propose_update.py`, `import_agent.py`.
3. **Schema normalization with permissive defaults**: `parse_review_response` guarantees every result dict has every key. Downstream code never needs `dict.get(...)` chains, and a dropped `valid` field silently means "passed" — safe-by-default.
4. **Prompt-template-as-contract**: `REVIEW_PROMPT` doubles as the schema spec; readers must keep the prompt and `DEFEAT_REASON_TYPES` in sync.
5. **Stringly-typed classification with fuzzy fallback**: exact-match first, substring second, `""` third — resilient to chatty models.

## Dependencies

**Imports**
- `json` — for `JSONDecoder.raw_decode` in `parse_review_response`.
- `sys` — for `stderr` diagnostics.
- `.llm.invoke_model` — the single LLM entry point. Its signature here is assumed to be `invoke_model(prompt, *, model, timeout) -> str`; any change to that contract propagates directly into this module.

**Imported by**
- `reasons/api.py` — likely exposes `review_beliefs` through the public Python API.
- `reasons/repair.py` — very likely consumes `defeat_reason_type` from review results to pick a repair strategy.
- `reasons/review_justifications.py` — sibling reviewer that may share the prompt/parser conventions.
- Mirrored `reasons_lib/*` copies and `tests/test_review.py`, `test_classify_defeaters.py`, `test_repair_premises.py`, `test_review_premises.py`, `test_review_justifications.py`.
- Because the module sits at the intersection of `api`, `repair`, and `review_justifications`, **the return schema from `parse_review_response` is effectively a public contract** for the repair subsystem.

## Flow

`review_beliefs` is the entry point; typical call is from `api.review` or `repair`:

```
nodes (dict id→node)
    │
    ├─ belief_ids=None → filter IN+nodes-with-justifications, sorted
    │  belief_ids=[...] → filter to existing nodes with justifications
    │
    ├─ if empty → return []
    │
    └─ for each chunk of batch_size:
         format_belief_for_review(nid, nodes)  → "### id\nClaim…\nAntecedents…"
            → join with "\n\n"
            → REVIEW_PROMPT.format(beliefs=…)
            → invoke_model(…, model, timeout)
            → parse_review_response(response)
                → raw_decode at each '['  (drop non-dicts & no-id items)
                → normalize to fixed-shape dicts with permissive defaults
            → all_results.extend(results)
            → on_batch(all_results)   (if provided)
       │
       └─ return all_results
```

The module does *no* mutation of `nodes`. It never persists anything; persistence and graph defeater-wiring are the responsibility of callers (`repair.py`, `api.py`).

## Invariants

1. **Only justified derived beliefs are reviewed.** Both branches of `belief_ids` selection require `node.get("justifications")` non-empty. Beliefs without justifications are silently skipped — this prevents auditing *premise* beliefs through the derived-belief path (premises have their own reviewer in `review_premises.py`).
2. **Only `IN` beliefs are auto-selected.** When `belief_ids is None`, `truth_value == "IN"` is required. Passing an explicit list bypasses the truth-value filter but still requires justifications.
3. **Order is preserved.** Auto-selection iterates `sorted(nodes.items())`; explicit lists preserve caller order. The prompt contract also demands results "in the same order as presented" — but the parser does not re-order or re-key by position, so downstream must match by `id`.
4. **Every result dict has the same keys.** `parse_review_response` guarantees `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`, `scope_findings`, `missing_property`, `defeat_reason_type`. Missing model fields default to permissive (True/empty).
5. **`defeater_reason_type` values are constrained** to `DEFEAT_REASON_TYPES`. `classify_defeat_reason` will not emit anything else, but `parse_review_response` does **not** validate free-text `defeat_reason_type` from `REVIEW_PROMPT` output — callers must sanitize.
6. **`unnecessary_antecedents` is always a list** (default `[]`), even if the model omits it — callers can iterate without a guard.

## Error Handling

The overall posture is **degrade-and-continue**, not fail-fast:

- **`classify_defeat_reason`**: wraps `invoke_model` in `try/except Exception`. On failure, prints `WARN: classification failed: …` to `stderr` and returns `""`. Unrecognized model output also returns `""`.
- **`review_beliefs`**: each batch is wrapped in `try/except Exception`. On failure, prints `WARN: batch N failed: …` to `stderr` and continues to the next batch. **A failed batch is silently excluded** from `all_results` — the returned list may not cover every requested belief. Callers who need "every belief got a verdict" semantics must reconcile against `belief_ids`.
- **`parse_review_response`**: never raises. If no parseable array exists, returns `[]`. Malformed items are dropped one-by-one; a partially valid array still yields the surviving items.
- **`format_belief_for_review`**: returns `""` for unknown node IDs rather than raising. Note this can't happen in the batch path because selection already filters to existing nodes.

Consequence for the `repair` subsystem: a review that fails entirely still looks like "no problems found" (empty list), so repair code must distinguish "nothing to fix" from "review failed." The `stderr` warnings are the only signal.

---

## Topics to Explore

- [file] `reasons/repair.py` — How the `scope_findings` / `missing_property` / `defeat_reason_type` fields are consumed to construct structured defeaters and drive repairs
- [file] `reasons/llm.py` — The `invoke_model` contract (signature, return type, retries, timeouts) that this module depends on
- [file] `reasons/review_justifications.py` — Sibling reviewer; compare prompt/parser conventions and whether they share the `parse_review_response` idiom
- [function] `reasons/review.py:parse_review_response` — The `raw_decode`-scanning parser — verify behavior on prose-wrapped, truncated, and multi-array outputs
- [file] `tests/test_review.py` — Expected contracts: filtering behavior, batching, default-field normalization, and failure isolation
- [general] `truth-maintenance-structured-defeaters` — How scope/missing-property findings become auditable defeat nodes in the graph

## Beliefs

- `review-beliefs-requires-justifications` — `review_beliefs` only selects nodes whose `justifications` list is non-empty, whether IDs are auto-selected or caller-provided.
- `review-beliefs-batches-llm-calls` — The function splits work into chunks of `batch_size` (default 20) LLM invocations, one prompt per chunk.
- `parse-review-response-defaults-to-permissive` — Missing `valid`/`sufficient`/`necessary` fields in LLM JSON default to `True`.
- `review-batch-errors-are-swallowed` — Per-batch exceptions in `review_beliefs` are logged to `stderr` and the batch is skipped, not propagated.
- `review-produces-structured-defeaters` — When `valid` is `false`, results carry `scope_findings`, `missing_property`, and `defeat_reason_type` for downstream defeater construction.
- `classify-defeat-reason-maps-to-enum` — `classify_defeat_reason` resolves LLM output to a `DEFEAT_REASON_TYPES` value via exact-then-substring matching, otherwise returns `""`.
