---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/contradictions.py

# `reasons/contradictions.py` — Explanation

## Purpose

This module owns **contradiction detection over a belief network**: it submits currently-held (`truth_value == "IN"`) beliefs to an LLM and asks it to surface *nogoods* — sets of beliefs that cannot all be true simultaneously. The output is a list of contradiction proposal dicts (`id`, `claims`, `analysis`, `severity`) that can be materialized into a "contradiction plan" file for downstream review/repair.

It provides two entry points:
- `detect_contradictions` — naive random batching.
- `detect_contradictions_semantic` — clusters beliefs by embedding similarity first, so topically related beliefs are analyzed together (better recall for contradictions that span similar concepts).

---

## Key Components

### Constants
- **`CONTRADICTION_BATCH_SIZE = 50`** — default max beliefs per LLM call. Used as the default `batch_size` in `detect_contradictions`, and as a hard-coded batch size inside `detect_contradictions_semantic`.
- **`CONTRADICTION_PROMPT`** — template string. Declares four contradiction classes (direct negation, incompatible properties, scope conflict, quantitative mismatch), mandates an exact output schema (`### NOGOOD <id>` with `Claims:`, `Analysis:`, `Severity:` bullet lines), enforces a minimum of 2 claims, and instructs the model to answer `No contradictions detected.` when nothing is found. `{beliefs}` is the only substitution slot.

### `format_beliefs_for_contradiction_check(belief_ids, nodes) -> str`
Renders beliefs as a newline-separated markdown list:
```
- `<id>`: <text>
```
- Skips any id missing from `nodes`.
- Truncates `text` to 200 chars (`197 + "..."`) so a batch of 50 beliefs doesn't blow up the context.
- Reads only `node["text"]` — the truth value is assumed to have been checked by the caller.

### `parse_contradiction_response(response, valid_ids=None) -> list[dict]`
Regex-based extraction of NOGOOD blocks.

- `_NOGOOD_PATTERN` matches `### NOGOOD <id>` followed by a body, terminated by the next `### NOGOOD` or end-of-text (`re.DOTALL`).
- Body is parsed line-by-line: each line is `strip()`ed and then `lstrip("- ")`, then matched case-insensitively against `claims:`, `analysis:`, `severity:` prefixes.
- `claims` is split on commas, each entry stripped and backtick-stripped.
- **When `valid_ids` is provided**, claims are filtered to the set; the LLM's invented ids are silently dropped.
- **Only blocks with ≥ 2 surviving claims** are appended — this is the hard "minimum 2 claims" invariant that mirrors the prompt instruction.

### `_flush_results(results, output_path, header_written) -> bool`
Incremental writer. If `output_path` is falsy or `results` is empty, it returns the input `header_written` unchanged. Otherwise it lazily imports `write_contradiction_plan` from `.api` and calls it with `append=header_written`, returning `True` — so the first flush writes the header, subsequent flushes append.

The lazy import **inside the function body** exists to avoid a circular import: `reasons/api.py` imports `contradictions`, so `contradictions` cannot import `api` at module load.

### `detect_contradictions(nodes, belief_ids=None, model="claude", timeout=300, batch_size=CONTRADICTION_BATCH_SIZE, output_path=None) -> list[dict]`
Naive batch detector.

### `detect_contradictions_semantic(nodes, belief_ids=None, model="claude", timeout=300, embedding_model=None, output_path=None) -> list[dict]`
Cluster-then-batch detector. Lazily imports `list_clusters` and `DEFAULT_MODEL` from `.cluster`. Note: it does **not** accept a `batch_size` parameter — inner batching always uses `CONTRADICTION_BATCH_SIZE` (50).

---

## Patterns

- **Lazy imports for circular-dependency breaking** — both `write_contradiction_plan` and `list_clusters` are imported inside function bodies rather than at module top.
- **Prompt-as-constant / structured output** — a rigid textual format ("### NOGOOD …") is requested so a simple regex parser can reliably extract structured records without a JSON/marshalling dependency. This trades strictness for simplicity.
- **Shuffle-then-batch** — `random.shuffle(belief_ids)` before slicing is deliberate: the LLM is a black box, so random partitioning gives variety across runs and avoids grouping beliefs alphabetically or by insertion order.
- **Best-effort / partial results** — a per-batch `try/except` that logs to stderr and continues means callers get whatever succeeded, not an all-or-nothing raise.
- **Streaming output with append-mode flags** — `header_written` is a monadic-ish flag threaded through `_flush_results`, making incremental writes idempotent for the header while the process runs.
- **Filter-by-truth-value at boundary** — both entry points reduce `belief_ids` to only nodes present in `nodes` **and** with `truth_value == "IN"` before doing any work.

---

## Dependencies

**Imports (this module → others):**
- `random`, `re`, `sys` — stdlib.
- `.llm.invoke_model` — top-level; the single LLM call site.
- `.api.write_contradiction_plan` — **lazy**, inside `_flush_results`.
- `.cluster.list_clusters`, `.cluster.DEFAULT_MODEL` — **lazy**, inside `detect_contradictions_semantic`.

**Imported by (others → this module):**
- `reasons/api.py` — likely re-exports or delegates to `detect_contradictions*`; also is the target of the lazy import.
- `reasons_lib/contradictions.py` — a mirrored copy in the duplicate `reasons_lib/` tree (verify divergence).
- `tests/test_contradictions.py` — direct tests.

---

## Flow

### `detect_contradictions`
1. **Select candidates** — `belief_ids is None` ⇒ collect all IN ids sorted; otherwise filter the caller's list down to ids that exist in `nodes` and are still IN.
2. **Short-circuit** — empty candidate list returns `[]` with no LLM call.
3. `random.shuffle(belief_ids)`; snapshot `valid_ids = set(belief_ids)` so hallucinated ids in LLM output are filtered against the full set (not just the batch — a small looseness, since technically a hallucinated id from another batch would pass the filter).
4. Compute `total_batches = ceil(len / batch_size)`.
5. For each batch:
   - render prompt via `format_beliefs_for_contradiction_check` then `CONTRADICTION_PROMPT.format(...)`
   - `invoke_model(prompt, model, timeout)`
   - `parse_contradiction_response(response, valid_ids=valid_ids)`
   - extend `all_results`; call `_flush_results`
   - on exception: `WARN: batch N failed: …` to stderr, **continue**.
6. Return `all_results`.

### `detect_contradictions_semantic`
1. Same candidate filtering.
2. Build `beliefs = {id: text}` from `nodes`.
3. `list_clusters(beliefs, model_name=embedding_model or DEFAULT_MODEL)` ⇒ `result["clusters"]`, each cluster `{beliefs: [{id, ...}]}`.
4. For each cluster with ≥ 2 beliefs:
   - slice into chunks of `CONTRADICTION_BATCH_SIZE` (50), build prompt, invoke, parse, extend, flush.
   - on exception: `WARN: cluster C batch failed: …`, continue.
5. Return `all_results`.

Note the prompt and parser are **identical** across both paths — only the partitioning strategy differs.

---

## Invariants

- **Only IN beliefs are ever sent to the LLM.** Both entry points force this via the `truth_value == "IN"` filter during candidate selection.
- **Every emitted nogood has ≥ 2 claims.** Enforced in `parse_contradiction_response` (`if len(claims) >= 2`) and reinforced by the prompt text.
- **Claims must appear in `valid_ids`** (when non-`None`). The parser drops anything else — protects against LLM hallucination of ids.
- **Belief text is truncated to ≤ 200 chars** in prompt rendering.
- **The header of a plan file is written exactly once.** `_flush_results` returns `True` after the first successful write; every subsequent call passes `append=True`.
- **`detect_contradictions_semantic` uses the module-level 50-belief cap**, not a caller-supplied batch size — clusters cannot be given a different batch granularity.
- **Batch failures do not abort the run.** The caller always receives a list (possibly partial) — never an exception from `invoke_model`.

---

## Error Handling

- **Swallowed per-batch LLM errors** — any exception from `invoke_model` (or from `parse_contradiction_response` if it raised for some reason) is caught in the per-batch try/except and printed to stderr as `WARN: batch N failed: …` / `WARN: cluster C batch failed: …`. The loop continues.
- **Malformed model output is silently dropped** — `parse_contradiction_response` never raises. A response with no `### NOGOOD` blocks, or with blocks missing `Claims:`, simply yields no entries. Only structured blocks with ≥ 2 valid claims survive.
- **Empty candidates are a no-op** — both entry points return `[]` before invoking the LLM.
- **`_flush_results` is a no-op** when `output_path` is falsy or `results` is empty — so a caller can request detection without ever touching the filesystem.
- **No retry logic.** A transient LLM failure during a batch means that batch's potential contradictions are lost, with only a stderr warning as evidence.

---

## Topics to Explore

- [file] `reasons/cluster.py` — Defines `list_clusters` and `DEFAULT_MODEL`; determines the cluster structure the semantic path consumes (each belief appears in exactly one cluster?).
- [function] `reasons/llm.py:invoke_model` — The single LLM gateway; its error, timeout, and model-name semantics directly shape `contradictions.py`'s failure handling.
- [function] `reasons/api.py:write_contradiction_plan` — Determines the plan file format and exactly what `append=header_written` writes.
- [file] `reasons_lib/contradictions.py` — Confirm whether it is a byte-for-byte mirror or has diverged (both trees appear to ship the same modules).
- [file] `tests/test_contradictions.py` — Shows which parsing edge cases (empty response, hallucinated ids, single-claim nogoods) are regression-tested.

## Beliefs

- `contradictions-detects-only-in-beliefs` — Both `detect_contradictions` and `detect_contradictions_semantic` filter candidate beliefs to nodes whose `truth_value == "IN"` before any LLM call.
- `parse-contradiction-filters-hallucinated-ids` — `parse_contradiction_response` drops any claim not present in `valid_ids` and discards a nogood if fewer than 2 claims survive.
- `contradictions-requires-2-claims-per-nogood` — A nogood is only retained when its parsed `claims` list has length ≥ 2, mirroring the prompt's minimum-2 rule.
- `contradictions-swallows-batch-llm-failures` — `invoke_model` errors are caught per batch, logged as `WARN: …` to stderr, and do not abort the loop — the function always returns a list.
- `write-contradiction-plan-lazy-imported` — `write_contradiction_plan` is imported inside `_flush_results` rather than at module top to break the circular dependency with `reasons/api.py`.
