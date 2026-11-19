---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: propose_updates in reasons/propose_update.py

## `propose_updates`

### Purpose
`propose_updates` is the orchestration entry point for "review a set of TMS beliefs and let an LLM propose corrections/retractions for them." It exists so the caller doesn't need to know about batching, prompt construction, parsing, or per-batch fault tolerance. Its job is:

1. Pick the set of beliefs to audit (defaults to all `IN` beliefs).
2. Chunk them into LLM-sized batches.
3. Format each batch into the audit prompt.
4. Call the model, parse the structured proposals, and accumulate them.
5. Not die if one batch fails — log and continue.

It's the producer side of the review pipeline; the parsed proposals it returns are what `format_proposal_markdown` / `format_proposals_file` later render, and what the CLI's accept/retract commands consume.

### Contract
- **Preconditions (unenforced)**: `nodes` is a dict mapping `node_id -> node_dict`; each `node_dict` follows the shape read by `format_belief_for_update` (`text`, `truth_value`, `source`, `source_url`, `metadata`, `justifications`, `dependents`). `batch_size` must be a positive integer — this is **not** validated (see Error Handling). If `belief_ids` is `None`, the keys of `nodes` must be sortable.
- **Postconditions**: Returns a list of proposal dicts. It does not raise for per-batch LLM or parse failures; it degrades to partial (or empty) results. A batch's proposals appear in the returned list in the same order the batch was scheduled, but proposals *within* a batch are in the order the model emitted them.
- **Invariants**:
  - `belief_ids` ordering is preserved: chunked with `range(0, len, batch_size)`, so slices follow the input order (sorted by ID when `belief_ids is None`).
  - Every returned proposal has been through `parse_update_proposals`, so its keys/values are normalized (validated `action`, `failure_mode`, `basis`).

### Parameters
- **`nodes`** (`dict[str, dict]`): The full belief graph. Required. Only nodes in here are auditable; anything else referenced (antecedents, dependents) is rendered as `(not found)` by the formatter.
- **`belief_ids`** (`list[str] | None`): If `None`, defaults to `[nid for nid, node in sorted(nodes.items()) if node.get("truth_value") == "IN"]` — i.e. all currently-true beliefs, ordered by ID. If provided, it is filtered with `nid in nodes` (unknown IDs are silently dropped) but **not** filtered by `truth_value` — a caller can deliberately audit `OUT` or stale beliefs. Order is preserved.
- **`model`** (`str`, default `"claude"`): Forwarded verbatim to `invoke_model`.
- **`timeout`** (`int`, default `300`): Forwarded verbatim to `invoke_model` (seconds).
- **`batch_size`** (`int`, default `PROPOSE_UPDATE_BATCH_SIZE` = 10): Number of beliefs per LLM call. Assumed positive. `0` raises `ValueError` from `range`; a negative value makes `range(0, n, batch_size)` empty, so the function silently returns `[]` without making any calls.
- **`on_batch`** (`Callable[[list], None] | None`): Progress/streaming hook. Called with the **cumulative** `all_results` list after each successful batch — not the batch-local results. It is invoked even when that batch produced zero proposals (the list is simply unchanged), so callers must tolerate repeat invocations with identical content.

### Return Value
A `list[dict]`, each dict normalized by `parse_update_proposals` with keys:

| key | type | notes |
|---|---|---|
| `id` | `str` | required by parser; items without `id` are dropped |
| `action` | `"update"` \| `"retract"` | defaults to `"update"` on invalid values |
| `proposed_text` | `str \| None` | `None` for retractions or if model omitted it |
| `failure_mode` | `str` | must be one of `FAILURE_MODES`, else `""` |
| `basis` | `str` | must be one of `BASES`, else `"prior-knowledge"` |
| `evidence` | `str` | `""` if missing |
| `comment` | `str` | `""` if missing |

Return value is `[]` when: `belief_ids` (after filtering) is empty, all batches fail, or all batches return no proposals. **Callers must treat an empty result as "no proposals or no data" — the function cannot distinguish "model found nothing" from "every batch errored."**

### Algorithm (step by step)
1. **Resolve belief set.** If `belief_ids is None`, sort `nodes.items()` and keep nodes with `truth_value == "IN"`. Otherwise, filter the caller's list to IDs present in `nodes`.
2. **Short-circuit.** `if not belief_ids: return []`.
3. **Precompute progress.** `total_batches = ceil(len(belief_ids) / batch_size)`, used only for the stderr progress line.
4. **Per-batch loop** over `range(0, len(belief_ids), batch_size)`:
   a. Slice `batch = belief_ids[i:i+batch_size]`, compute 1-based `batch_num`.
   b. Emit progress line to `sys.stderr`.
   c. Build `beliefs_text` by joining `format_belief_for_update(nid, nodes)` for each ID with blank lines (double newline).
   d. Substitute into `PROPOSE_UPDATE_PROMPT.format(beliefs=beliefs_text)`. Note the template uses `{{`/`}}` for the literal JSON braces, so `.format` only touches the `{beliefs}` placeholder — braces inside `beliefs_text` are values and are not re-expanded.
   e. `invoke_model(prompt, model=model, timeout=timeout)`.
   f. `parse_update_proposals(response)` → list of normalized proposals.
   g. `all_results.extend(results)`.
   h. `if on_batch is not None: on_batch(all_results)` — cumulative.
5. **Batch failure path**: any exception raised in steps (e)–(h) is caught, printed as `WARN: batch N failed: <exc>` to stderr, and the loop continues to the next batch.
6. Return `all_results`.

### Side Effects
- **Network/LLM**: one `invoke_model` call per batch (potentially slow, cost-bearing).
- **stderr writes**: a `Reviewing batch X/Y (N beliefs)...` line per batch, plus `WARN: ...` on failures. Nothing goes to stdout.
- **Callback invocation**: `on_batch(all_results)` after each successful batch; the list passed is the *live* `all_results` object, so a callback that mutates it mutates the return value.
- **No mutation of `nodes`**: the function and `format_belief_for_update` only read from it.
- **No file I/O** here (unlike the sibling `format_proposals_file`).

### Error Handling
- **Per batch, broadly swallowed**: the `try` wraps the model call, the parse, the extend, and `on_batch`. Exceptions from *any* of these — including a bug in the caller's `on_batch` callback — are caught by `except Exception` and reported as a batch failure. This means a callback exception looks like an LLM failure and is not re-raised. Subtle but deliberate consequence: a misbehaving `on_batch` cannot crash the run, but it also cannot surface its own failure.
- **Not caught**: errors before the loop (e.g. `sorted(nodes.items())` on non-sortable keys when `belief_ids is None`; `batch_size == 0` from `range`).
- **Silent data loss**: failed batches simply don't contribute proposals. There's no `errors` field on the return value, so callers using `on_batch` for progress are the only ones who can observe failures (by watching stderr or invocation counts).
- **No retry logic**: a transient LLM error drops the whole batch's beliefs for that run.

### Usage Patterns
Typical call from the CLI or a review command:

```python
proposals = propose_updates(nodes)                     # audit all IN beliefs
proposals = propose_updates(nodes, belief_ids=["b1"])  # targeted audit
proposals = propose_updates(nodes, on_batch=save_progress)  # stream cumulative results
```

Caller obligations:
- If you supply `belief_ids`, you're responsible for correctness — no `truth_value` check is applied.
- If you supply `on_batch`, tolerate cumulative (not per-batch) semantics and repeat calls with unchanged content.
- Treat the result as possibly partial: if correctness matters, correlate batch counts on stderr with `total_batches` to detect dropped batches.
- Proposals should be reviewed/validated before being applied; this function never mutates the graph. Acceptance is a separate step (`reasons update` / `reasons retract` commands referenced in `format_proposal_markdown`).

### Dependencies
- **Internal**: `.llm.invoke_model`; same-module `format_belief_for_update`, `parse_update_proposals`, `PROPOSE_UPDATE_PROMPT`, `FAILURE_MODES`, `BASES`, `PROPOSE_UPDATE_BATCH_SIZE`.
- **Stdlib**: `sys` (stderr), `json` (used in `parse_update_proposals`, which is relied upon transitively).
- **Implicit data contract**: node dict shape consumed by `format_belief_for_update` — the function has no validation for missing keys because that helper uses `.get()` with defaults.

### Assumptions not enforced by the type system
- `batch_size` is positive (zero raises, negative silently no-ops).
- Keys of `nodes` are mutually sortable (only relevant when `belief_ids is None`).
- `truth_value` is the string `"IN"` for live beliefs (TMS convention).
- `on_batch` accepts a single positional list argument and tolerates being called with the same cumulative list multiple times.
- Each node's `dependents` is a set or list (the formatter explicitly checks `isinstance(dependents, (set, list))` before iterating).
- The prompt template's `{beliefs}` placeholder exists in `PROPOSE_UPDATE_PROMPT`; substituting it doesn't re-expand braces inside belief text.

## Topics to Explore
- [function] `reasons/propose_update.py:parse_update_proposals` — How prose-wrapped JSON is recovered and how proposals are normalized/rejected
- [function] `reasons/propose_update.py:format_belief_for_update` — The exact node shape and justification/dependent rendering the LLM sees
- [function] `reasons/llm.py:invoke_model` — Retry, timeout, and error semantics the batch loop's fault tolerance depends on
- [file] `tests/test_propose_update.py` — Expected contracts for filtering, batching, and failure behavior
- [general] `on-batch-cumulative-callback-pattern` — Whether other modules use cumulative or per-batch callbacks, and how the CLI consumes them

## Beliefs
- `propose-updates-batches-beliefs` — `propose_updates` splits the resolved belief ID list into chunks of `batch_size` and issues exactly one `invoke_model` call per chunk.
- `propose-updates-defaults-to-in-nodes` — When `belief_ids is None`, it audits only nodes whose `truth_value` equals the string `"IN"`, sorted by node ID.
- `propose-updates-swallows-batch-errors` — Any exception inside the per-batch `try` block (including exceptions raised by the `on_batch` callback) is logged to stderr as `WARN: batch N failed` and never propagated.
- `propose-updates-on-batch-receives-cumulative-results` — `on_batch` is invoked once per successful batch with the running `all_results` list, even when that batch returned zero proposals.
- `propose-updates-filters-unknown-belief-ids` — Explicitly supplied `belief_ids` not present in `nodes` are silently dropped, and no `truth_value` filtering is applied to the survivors.
