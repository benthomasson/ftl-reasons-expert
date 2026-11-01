---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/review_justifications.py

# `reasons/review_justifications.py`

## Purpose

This module implements a **read-only LLM-powered audit** of justification modes in the project's Truth Maintenance System (TMS). Specifically, it scans derived beliefs that have multi-antecedent SL (Simple Logic) justifications and asks an LLM to classify whether each justification is correctly typed as **ALL** (conjunctive — all antecedents required) or should actually be **ANY** (disjunctive — any single antecedent suffices), or a **MIXED** split.

The critical constraint: **it never mutates the database**. It produces a report of candidates for conversion. The actual conversion would happen downstream (likely in `propose_update.py` or `repair.py`).

This is one of several "review" modules in the project (`review.py`, `review_premises.py`, `review_justifications.py`), each targeting a different dimension of belief-graph health.

## Key Components

### Constants

| Name | Value | Role |
|------|-------|------|
| `REVIEW_BATCH_SIZE` | `15` | Default number of beliefs sent per LLM call. Chosen to fit within context windows while maximizing throughput. |

### `REVIEW_JUSTIFICATIONS_PROMPT` (template string)

The system prompt sent to the LLM. It:
- Defines the TMS context (ALL vs ANY vs MIXED semantics).
- Demands a strict JSON array response with fields: `id`, `classification`, `required_antecedents`, `independent_antecedents`, `comment`.
- Provides explicit rules for how to fill the two antecedent lists per classification.
- Ends with a `{beliefs}` placeholder that gets filled with formatted belief data.

Note the double-braces `{{ }}` around the JSON example — this is because the string is consumed by `str.format()`.

### `parse_justification_review(response) -> list[dict]`

**Contract:** Accepts a raw LLM response string. Returns a list of normalized result dicts, or `[]` if nothing parseable is found.

**Implementation detail:** It does *not* simply `json.loads(response)`. Instead, it iterates character-by-character looking for the first `[`, then uses `json.JSONDecoder().raw_decode()` from that offset. This is a resilience pattern — LLMs frequently wrap JSON in prose, code fences, or trailing commentary. The first valid JSON array wins.

Each item is validated: must be a dict with an `"id"` key. `classification` is uppercased (defaulting to `"ALL"` if not a string). Antecedent lists are validated as `list` instances, defaulting to `[]`.

### `_process_batch(batch, nodes, model, timeout) -> list[dict]`

Internal helper. Takes a list of belief IDs, formats them via `format_belief_for_review`, injects them into the prompt, calls `invoke_model`, and parses the result. Emits a `WARN` to stderr if the batch is non-empty but parsing yielded nothing.

### `review_justifications(...)` — **public entry point**

```python
def review_justifications(nodes, belief_ids=None, model="claude", timeout=300,
                          batch_size=REVIEW_BATCH_SIZE, min_antecedents=2,
                          on_batch=None, parallel=0):
```

**Parameters:**
- `nodes` — dict mapping `node_id → node_data` (the shape returned by `export_network()`). This is the in-memory graph.
- `belief_ids` — optional explicit filter; if `None`, auto-discovers all IN-truth-value nodes with multi-antecedent SL justifications.
- `model` — LLM model identifier (passed to `invoke_model`).
- `timeout` — per-call timeout in seconds (default 300).
- `batch_size` — beliefs per LLM call.
- `min_antecedents` — minimum antecedent count to consider (default 2; a single-antecedent justification can't be ALL vs ANY).
- `on_batch` — optional callback `fn(cumulative_results)` invoked after each batch (useful for streaming UIs or incremental persistence).
- `parallel` — if > 0, enables `ThreadPoolExecutor` with that many workers.

**Returns:** `list[dict]` — all parsed results concatenated in completion order.

### `_has_multi_antecedent_sl(node, min_antecedents) -> bool`

Predicate: checks whether *any* justification on the node has `type == "SL"` and `len(antecedents) >= min_antecedents`. Single-antecedent SL justifications are trivially unambiguous, so they're excluded.

## Patterns

1. **Read-only audit / report-only tool.** The module's docstring explicitly states "does not modify the database." It separates *diagnosis* (this file) from *remediation* (other modules). This is a deliberate architectural boundary.

2. **Batch + LLM fan-out.** Beliefs are chunked into batches of 15. Each batch is one LLM call. This bounds context length and allows progress tracking.

3. **Parallel execution via `ThreadPoolExecutor`.** When `parallel > 0`, batches are submitted concurrently. Note: this is thread-based (not process-based), which is fine because the bottleneck is I/O (waiting on the LLM API), not CPU.

4. **Defensive JSON parsing.** The `raw_decode` + character-scan approach in `parse_justification_review` is a common idiom for LLM output parsing — you never trust the LLM to return *pure* JSON.

5. **Callback-based progress.** `on_batch` lets callers stream results without the module owning any UI or persistence logic.

6. **Dual package layout.** The repo has both `reasons/` and `reasons_lib/` containing the same modules. `reasons/` appears to be the canonical source; `reasons_lib/` is likely a vendored or published copy (see `pyproject.toml` for packaging config). The module is imported by `reasons/api.py` using the relative import `from .review_justifications import ...`.

## Dependencies

### Upstream (this file imports)

| Import | Source | Used for |
|--------|--------|----------|
| `json` | stdlib | JSON parsing in `parse_justification_review` |
| `sys` | stdlib | `sys.stderr` for warning/progress output |
| `invoke_model` | `reasons/llm.py` | LLM API call wrapper (handles auth, retries, model routing) |
| `format_belief_for_review` | `reasons/review.py` | Serializes a node + its justifications into a human/LLM-readable text block |
| `ThreadPoolExecutor`, `as_completed` | `concurrent.futures` | Parallel batch processing (lazy import inside the `parallel > 0` branch) |

### Downstream (imports this file)

| Importer | Why |
|----------|-----|
| `reasons/api.py` | Exposes `review_justifications` as a programmatic API endpoint |
| `reasons_lib/review_justifications.py` | Mirror copy for the published library |
| `tests/test_review_justifications.py` | Unit/integration tests |

## Flow

```
Caller (api.py / cli.py)
  │
  ▼
review_justifications(nodes, ...)
  │
  ├── 1. Determine belief_ids
  │       Auto: filter nodes where truth_value=="IN" AND _has_multi_antecedent_sl(node, min_antecedents)
  │       Or:   filter caller-provided IDs through the same predicates
  │
  ├── 2. If no qualifying beliefs → return []
  │
  ├── 3. Split belief_ids into batches of `batch_size`
  │
  ├── 4. For each batch (sequential or parallel):
  │       │
  │       ▼
  │     _process_batch(batch, nodes, model, timeout)
  │       │
  │       ├── format_belief_for_review(nid, nodes)  ← per belief
  │       ├── REVIEW_JUSTIFICATIONS_PROMPT.format(beliefs=...)
  │       ├── invoke_model(prompt, model, timeout)  ← LLM call
  │       └── parse_justification_review(response)  ← extract JSON array
  │       │
  │       ▼
  │     results: list[dict]
  │
  ├── 5. Accumulate all_results, call on_batch(all_results) if set
  │
  └── 6. Return all_results
```

Data transformation: **in-memory node graph → formatted text prompt → LLM response (raw string) → parsed/validated list of dicts**.

## Invariants

- **Truth-value gate:** Only nodes with `truth_value == "IN"` are reviewed. OUT or UNJUSTIFIED nodes are skipped — their justifications are already inactive.
- **Minimum antecedent count:** A justification must have ≥ `min_antecedents` (default 2) SL antecedents to be considered. A 1-antecedent SL is neither ALL nor ANY — it's simply "requires this one premise."
- **Type filter:** Only `type == "SL"` justifications qualify. Other justification types (e.g., derived rules, imported beliefs) are out of scope.
- **Order preservation:** The prompt instructs the LLM to return results "in the same order as presented." The parser does not re-sort; it trusts order.
- **Classification vocabulary:** Must be one of `"ALL"`, `"ANY"`, `"MIXED"`. The parser uppercases whatever the LLM returns, so lowercase is tolerated.

## Error Handling

| Location | Strategy |
|----------|----------|
| `parse_justification_review` | Silently skips malformed items (no `"id"`, non-list antecedents). If *no* valid items are found, returns `[]`. Never raises. |
| `_process_batch` | If parsing yields `[]` for a non-empty batch, prints `WARN` to stderr. Does **not** raise — the batch is effectively lost. |
| `review_justifications` (sequential path) | Catches `Exception` per batch, prints `WARN`, continues to next batch. One failed batch doesn't abort the run. |
| `review_justifications` (parallel path) | Same: `future.result()` is wrapped in `try/except Exception`. Failed batches are logged and skipped. |
| `invoke_model` | Assumed to raise on timeout/auth failure (handled internally by `llm.py`). Propagates up to the per-batch `except` block. |

**Net effect:** This module is *degraded-tolerant*. A single LLM failure, a bad response, or a network blip produces a warning and a partial result set — never a crash. The caller is responsible for checking whether the result set is "complete enough."

## Topics to Explore

- [function] `reasons/review.py:format_belief_for_review` — This is the serialization function that turns a node's justifications into the text block fed to the LLM. Understanding what fields it includes (antecedent text, truth values, derivation history) tells you exactly what context the LLM sees.
- [function] `reasons/llm.py:invoke_model` — The LLM call wrapper. You'll want to see retry logic, timeout handling, and how `model` strings are mapped to API endpoints, since all review modules depend on its contract.
- [file] `reasons/propose_update.py` — The likely downstream consumer that would *actually* convert a justification from ALL to ANY (or vice versa) based on the audit results produced here. Understanding the full diagnose→propose→apply pipeline clarifies where this module sits.
- [file] `tests/test_review_justifications.py` — The test suite reveals the expected shapes of `nodes`, the exact prompt behavior, edge cases in parsing, and the parallel-mode contract.
- [general] `TMS-justification-types` — The broader taxonomy of justification types (SL, derived, imported, defeaters) in this TMS implementation. Knowing why SL is special and why ALL vs ANY matters for retraction propagation is essential context.

## Beliefs

- `review-justifications-is-read-only` — `review_justifications()` never writes to the database or mutates `nodes`; it returns a report list and the caller decides whether to act on it.
- `review-justifications-requires-IN-truth-value` — Only nodes with `truth_value == "IN"` are eligible for review; OUT and UNJUSTIFIED nodes are silently excluded regardless of their justification structure.
- `parse-justification-review-tolerates-prose-wrapper` — The parser scans for the first `[` character and uses `raw_decode`, meaning it correctly extracts JSON even when the LLM wraps it in markdown fences, preamble text, or trailing commentary.
- `review-justifications-batch-failure-is-non-fatal` — A single batch that raises or returns unparseable output produces a stderr warning and is skipped; remaining batches still execute and their results are returned.
- `min-antecedents-gates-review-eligibility` — A node with fewer than `min_antecedents` (default 2) SL antecedents is never included in the review set, even if its single-antecedent SL justification is arguably misclassified.
