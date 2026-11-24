---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: detect_contradictions in reasons/contradictions.py

# `detect_contradictions`

## Purpose

`detect_contradictions(nodes, ...)` is the LLM-driven contradiction scanner for the belief network. It gathers a set of currently-held beliefs (nodes whose `truth_value` is `"IN"`) and sends them in batches to a language model with a prompt asking it to identify *nogoods* — sets of beliefs that cannot all be true simultaneously. It parses structured `### NOGOOD` blocks out of the response, validates claim IDs against the belief set it sent, optionally persists results incrementally, and returns the aggregated list of contradiction records.

It exists as the "flat/batch" strategy parallel to `detect_contradictions_semantic`, which pre-clusters beliefs by embedding similarity before checking.

## Contract

**Preconditions (informal, not enforced)**
- `nodes` is a mapping `node_id -> dict` where each value has at least `truth_value` and `text` keys.
- If `belief_ids` is provided, it can contain arbitrary IDs — non-existent IDs and non-`IN` IDs are silently filtered out.
- `batch_size` should be a positive integer; `batch_size <= 0` will fail at `range(0, len, batch_size)` (a `ValueError` for 0) or silently do nothing for negative values.
- `output_path`, if given, points somewhere `write_contradiction_plan` can write to.

**Postconditions**
- Returns a list of dicts, each shaped like `{"id", "claims", "analysis", "severity"}` (as produced by `parse_contradiction_response`), restricted to contradictions whose claims include ≥2 IDs from the batch set.
- The caller's `belief_ids` list (if provided) is **not** mutated — a fresh list is built before `random.shuffle`.
- If no `IN` beliefs survive filtering, returns `[]` immediately and performs **no** LLM calls.

**Invariants**
- `valid_ids` is a snapshot of the shuffled batch-id set; it is passed to the parser so hallucinated / cross-batch IDs are dropped.
- `header_written` is monotonic: once results have been flushed, it stays `True` so subsequent flushes append rather than re-write the header.

## Parameters

| Name | Meaning | Edge cases |
|---|---|---|
| `nodes` | Full network export dict | Empty dict ⇒ no IN beliefs ⇒ returns `[]` |
| `belief_ids` | Optional subset to check | Non-`IN` or missing IDs silently dropped; caller may pass unsorted / unsanitized list |
| `model` | LLM model identifier passed through to `invoke_model` | Model name is opaque to this function |
| `timeout` | Per-LLM-call timeout in seconds | Passed straight through; this function has no overall timeout |
| `batch_size` | Beliefs per LLM call (default `CONTRADICTION_BATCH_SIZE = 50`) | Must be > 0 |
| `output_path` | Optional plan file for incremental writes | Non-blocking: write failures are caught as batch failures |

## Return Value

Always a `list` (possibly empty). It contains all successfully parsed contradictions across **all** batches, including batches that eventually failed for other reasons? No — a failed batch contributes nothing to `all_results` because `all_results.extend(results)` only runs after `invoke_model` and `parse_contradiction_response` succeed. So the return value is a *best-effort partial result*: prior successful batches are present, failed batches are silently absent (only a stderr warning). The caller cannot distinguish "no contradictions found" from "all batches failed" purely from the return value.

## Algorithm

1. **Normalize belief set**
   - `belief_ids is None`: build sorted list of node IDs whose `truth_value == "IN"`.
   - Else: filter provided IDs to those present in `nodes` **and** marked `IN`.
2. **Early exit** if the resulting list is empty.
3. **Materialize and shuffle** (`list(...)` + `random.shuffle`) — randomizing batch composition so different runs surface different pairings.
4. **Prepare bookkeeping**: `valid_ids = set(belief_ids)`, `all_results = []`, `total_batches = ceil(len / batch_size)`, `header_written = False`.
5. **Batch loop** (`for i in range(0, len, batch_size)`):
   - Slice `belief_ids[i:i+batch_size]`.
   - Print `Checking batch n/total (...)` to `sys.stderr`.
   - Format beliefs via `format_beliefs_for_contradiction_check` (truncates texts >200 chars).
   - Inject into `CONTRADICTION_PROMPT`.
   - `try`:
     - `invoke_model(prompt, model, timeout)` → raw text.
     - `parse_contradiction_response(response, valid_ids=valid_ids)` → dicts.
     - Extend `all_results`.
     - `_flush_results(results, output_path, header_written)` → returns updated header flag.
   - `except Exception`: print WARN and continue to next batch.
6. **Return** `all_results`.

## Side Effects

- **Randomness**: calls `random.shuffle` (module-level RNG; affects global RNG state).
- **stderr writes**: progress line per batch and warning line per failure.
- **Network I/O**: one `invoke_model` call per batch — potentially many expensive LLM calls.
- **File I/O**: when `output_path` is set, appends via `reasons.api.write_contradiction_plan(results, path, append=header_written)` — the first successful flush writes the header, later ones append.
- No mutation of caller-provided `nodes` or `belief_ids`.

## Error Handling

- **Per-batch exception swallowing**: any exception from `invoke_model`, `parse_contradiction_response`, or `_flush_results` is caught by the outer `try/except Exception` and turned into a `WARN: batch N failed: ...` stderr line. The overall call therefore **never raises** for normal runtime failures — callers must accept the "partial results" semantics.
- **Programmer errors are also swallowed**: e.g., an unexpected shape in `nodes` values would `AttributeError` inside the try and be reported as a batch failure rather than crashing — masking bugs.
- **Zero `batch_size`** raises `ValueError` from `range()` *outside* the try, so it does propagate. Negative `batch_size` similarly propagates or silently does nothing.
- **Write failures in `_flush_results`** are treated like batch failures — the parsed results remain in `all_results` but the file is not updated for that batch.

## Usage Patterns

Typically called from a CLI/audit entry point after `export_network()` produces a `nodes` dict:

```python
nodes = export_network(...)
contradictions = detect_contradictions(
    nodes, model="claude", output_path="plan.md"
)
```

Caller obligations:
- Pass a well-formed `nodes` mapping (dicts with `truth_value`/`text`).
- Treat the return as best-effort; if you require completeness, check stderr / log for `WARN` lines or compare `total_batches` to successful flush count (not exposed).
- When using `output_path`, understand that the file is created/appended incrementally and may be incomplete if batches fail.

## Dependencies

- Standard library: `random`, `sys` (used here), plus `re` and `sys` at module level.
- `reasons.llm.invoke_model` — external LLM call.
- `reasons.api.write_contradiction_plan` — imported lazily inside `_flush_results`.
- Module-level: `CONTRADICTION_BATCH_SIZE`, `CONTRADICTION_PROMPT`, `format_beliefs_for_contradiction_check`, `parse_contradiction_response`, `_flush_results`.

## Assumptions Not Enforced by the Type System

- Every value in `nodes` is a `dict` exposing `.get("truth_value")` and `.get("text")`.
- Each belief has a stable string ID usable both as a dict key and in the LLM prompt / parse step.
- The LLM actually honors the `### NOGOOD` output contract; parsing silently yields nothing otherwise.
- `CONTRADICTION_PROMPT` contains a literal `{beliefs}` placeholder (obvious from the `.format` call, but untyped).
- Batch ordering is irrelevant to correctness (relies on `random.shuffle`, but the initial `sorted(...)` in the `None` branch is redundant since shuffling follows — it only affects the shuffle's deterministic seed consumption order).

## Topics to Explore

- [function] `reasons/contradictions.py:parse_contradiction_response` — How the `### NOGOOD` blocks are regex-parsed, and how `valid_ids` filtering drops hallucinated claim IDs.
- [function] `reasons/contradictions.py:detect_contradictions_semantic` — The clustering-based alternative that groups similar beliefs before batching; compare completeness vs. cost.
- [function] `reasons/api.py:write_contradiction_plan` — The incremental-write target; determines header logic (`append=header_written`) and file format.
- [file] `reasons/llm.py` — Where `invoke_model` lives; timeout, retries, and provider-specific behavior.
- [file] `tests/test_contradictions.py` — Existing test coverage for prompt formatting, parsing edge cases, and batch-failure semantics.

## Beliefs

- `detect-contradictions-requires-in-truth-values` — Only nodes with `truth_value == "IN"` are eligible; the function filters both the `None` and explicit-`belief_ids` paths.
- `detect-contradictions-prevents-caller-list-mutation` — A fresh list is built before `random.shuffle`, so the caller's `belief_ids` argument is never mutated.
- `detect-contradictions-produces-partial-results-on-batch-failure` — Any exception during a batch is caught and only printed to stderr; the return value contains prior successful batches only.
- `detect-contradictions-enables-cross-batch-claim-filtering` — `valid_ids` is the full shuffled ID set (not per-batch) and is passed to the parser, so cross-batch claims survive while out-of-set claims are dropped.
- `detect-contradictions-implements-flat-batch-strategy` — It is the non-clustering counterpart to `detect_contradictions_semantic`, distinguished by random batching rather than embedding similarity.
