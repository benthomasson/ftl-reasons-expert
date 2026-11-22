---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _repair_one in reasons/repair_premises.py

## `_repair_one`

### 1. Purpose

`_repair_one` is the per-premise worker driven by `repair_premises`. For a single premise that a prior review step flagged as inaccurate, it:

1. Looks up the premise's node metadata, its review comment, and the raw source document that supposedly supports it.
2. Formats a per-premise text block and injects it into the `REPAIR_PREMISES_PROMPT` template.
3. Calls the LLM, parses the (single-item) JSON array response, and returns the one repair result.
4. Degrades gracefully when the model returns unparseable output or when the LLM call fails.

Its real purpose is to be the **unit of work** that can be scheduled sequentially or across a `ThreadPoolExecutor`. It contains no shared mutable state, which is what makes the parallel path in `repair_premises` safe *from this function's perspective*.

### 2. Contract

**Preconditions** (not enforced by types, only by callers/tests):
- `node_id` MUST be a key in `nodes` — the code does `nodes[node_id]` with a bare subscript. A stray ID raises `KeyError` *before* the `try` block.
- `nodes[node_id]` is a mapping; `source` and `text` are optional.
- `source_contents` is a `dict` mapping source-path strings → content strings.
- `review_results` is a `dict` mapping node-id strings → review dicts with optional `error_type` / `comment`.
- `model` is a name understood by `invoke_model`; `timeout` is a number of seconds.

**Postconditions**:
- Returns a dict with keys `{"id", "action", "corrected_text", "rationale"}` — this shape is consistent across all three return paths.
- On a successful LLM round-trip producing parseable JSON, returns the **first** parsed item (`parsed[0]`), which is *expected* but not verified to have `id == node_id`.
- `action` is expected to be `"rewrite"` or `"retract"`, but is **not validated** by the parser (see §7).
- Never lets an exception escape from inside the `try` block. It *does* let exceptions escape from the preamble (before the `try`).
- Does not mutate its inputs.

### 3. Parameters

| Param | Type | Meaning / edge cases |
|---|---|---|
| `node_id` | `str` | Premise ID. Assumed present in `nodes`. If absent → `KeyError` before the try. |
| `nodes` | `dict[str, dict]` | Node metadata from `export_network()`. Direct subscript means this is a hard precondition. `source` defaults to `""` and `text` defaults to `""` downstream. |
| `source_contents` | `dict[str, str]` | Source-path → content. Looked up with `nodes[node_id].get("source", "")`. A node with no source ends up looking up the empty string `""`; if `""` isn't a key, the fallback `"(source not available)"` is used. **The entire source document is embedded in the prompt each call** — no truncation, so a big source inflates the prompt and cost. |
| `review_results` | `dict[str, dict]` | Review comment/error_type. `.get(node_id, {})` so missing entries degrade gracefully. |
| `model` | `str` | Forwarded to `invoke_model`. |
| `timeout` | `int`/`float` | Forwarded to `invoke_model`. Timeout failures become `action: "error"` results. |

### 4. Return Value

Always a four-key dict:

- **Success/parseable**: `parsed[0]` — normalizer ensures keys exist, but `id` may not match `node_id` if the model hallucinated a different id or returned extras (only the first is kept), and `action` may be an arbitrary string.
- **Unparseable response** (`parse_repair_response` returned `[]`):
  ```python
  {"id": node_id, "action": "retract", "corrected_text": None,
   "rationale": "Could not parse repair response"}
  ```
- **Exception inside try**: `{"id": node_id, "action": "error", "corrected_text": None, "rationale": str(e)}`.

Callers **must** handle:
- `action == "error"` (LLM failure / timeout / network).
- `action == "retract"` due to *parse failure*, which is not the same thing as the model genuinely deciding to retract.
- Possibly a non-`{"rewrite","retract"}` action if the model hallucinated one.

### 5. Algorithm

1. `review = review_results.get(node_id, {})` — tolerate missing reviews.
2. `source = nodes[node_id].get("source", "")` — **hard index**, raises `KeyError` if missing.
3. `source_text = source_contents.get(source, "(source not available)")` — fallback if the source path is unknown or empty.
4. `prompt_text = format_premise_for_repair(...)` — builds "### id / Original claim / Error type / Review comment / Source ref / fenced source body". (Inside here, the "node not found" branch is effectively dead because step 2 already guaranteed existence.)
5. `prompt = REPAIR_PREMISES_PROMPT.format(premises=prompt_text)` — template substitution. Note: this happens **outside** the try, so a `KeyError`/`IndexError` from bad braces in the template would propagate (currently the only placeholder is `{premises}` and braces inside JSON are doubled, so it's fine).
6. **try**: `invoke_model(...)` → `parse_repair_response(...)` (character-scan for the first decodable JSON array, filters to dicts with an `id`).
7. If parsed list is non-empty, return `parsed[0]` immediately — the single-premise prompt should yield exactly one object; extras are dropped.
8. If parsing yields nothing, synthesize a **retract** result with a fixed rationale.
9. On any exception in steps 6–8, synthesize an **error** result whose rationale is `str(e)`.

### 6. Side Effects

- One network/LLM call through `invoke_model` (the only real external effect).
- No file I/O, no writes to `nodes`, `source_contents`, `review_results`, or global state.
- No threads spawned here; the caller (`repair_premises`) may run this concurrently. Correctness under concurrency therefore depends on `invoke_model` being thread-safe (or performing an independent request per call).

### 7. Error Handling

- The `try/except Exception` is **broad** and swallows *all* exceptions raised inside it (network, timeout, JSON parsing, model client errors), converting them into `{"action": "error", ...}`.
- **Gaps / assumptions not enforced**:
  - `nodes[node_id]` at the top is **outside** the try — a `KeyError` (or wrong-typed entry that raises) propagates. In the parallel path this surfaces via `future.result()`, crashing `repair_premises`; in the sequential path it aborts the whole run.
  - `REPAIR_PREMISES_PROMPT.format(...)` is outside the try — a malformed template would propagate similarly.
  - `parse_repair_response` does **not** validate `action` against `{"rewrite","retract"}`, so a hallucinated action such as `"delete"` passes through unchanged.
  - `parse_repair_response` does **not** verify `parsed[0]["id"] == node_id`, so an id mismatch is silently accepted (the caller gets a result keyed by the wrong id).
  - `str(e)` in the rationale may leak internal details (e.g., HTTP statuses, hostnames) into downstream artifacts.

### 8. Usage Patterns

- Only invoked from `repair_premises`:
  - **Sequential**: `pid in premise_ids` loop, appending each result.
  - **Parallel**: `executor.submit(_repair_one, pid, ...)`; results collected via `as_completed`, so `all_results` is in **completion order, not `premise_ids` order**. `on_result(all_results)` fires after every completed future with the *accumulating* list — callers needing determinism must key by `id`.
- Because of the `nodes[node_id]` hard precondition, callers should ensure `premise_ids ⊂ nodes.keys()` before fan-out, or wrap the call to handle `KeyError`.
- The prompt strategy embeds the full source document once per premise; assume `repair_premises` may repeat the same source across many nodes, so cost scales with premise count, not distinct sources.

### 9. Dependencies

- Internal: `format_premise_for_repair`, `parse_repair_response`, `REPAIR_PREMISES_PROMPT` (all same module).
- Internal: `reasons.llm.invoke_model` — the sole external boundary; its return must be a JSON-ish `str`.
- Stdlib: `json` (via the parser), and imported indirectly `concurrent.futures`, `sys` are used by the caller, not this function.

## Topics to Explore

- [file] `reasons/repair_premises.py` — The full module: how `_repair_one` is fanned out and how `repair_premises` aggregates results.
- [function] `reasons/repair_premises.py:parse_repair_response` — The parser's leniency (first `[`, no action/id validation) that `_repair_one` relies on.
- [function] `reasons/repair_premises.py:format_premise_for_repair` — Which fields are rendered and what the per-premise context block actually looks like.
- [file] `tests/test_repair_premises.py` — Asserted behavior for `_repair_one` (error/retract fallbacks, node-missing behavior, parallelism order).
- [general] `llm-concurrency-and-timeout` — Whether `invoke_model` is thread-safe and how timeouts surface as the `error` action.

## Beliefs

- `_repair_one-requires-node-in-nodes` — `nodes[node_id]` is accessed with a bare subscript before the `try`, so a missing node ID raises `KeyError` instead of producing an `error` result.
- `_repair_one-swallows-only-try-block-exceptions` — Any exception inside the `try` maps to `action: "error"` with `rationale = str(e)`; exceptions in the preamble propagate.
- `_repair_one-returns-first-parsed-item` — Only `parsed[0]` is returned, and its `id` is never checked against `node_id`.
- `parse-failure-maps-to-retract` — When `parse_repair_response` yields an empty list, `_repair_one` returns `action: "retract"` with a fixed rationale, conflating parse failure with source-unsupported retraction.
- `action-values-not-validated` — `parse_repair_response` uses `item.get("action", "retract")` without restricting to `{"rewrite","retract"}`, so an arbitrary model-supplied action can reach callers.
- `source-lookup-is-by-path` — Source text is fetched via `nodes[node_id].get("source", "")` into `source_contents` with the literal fallback `"(source not available)"`; the full document is embedded per premise.
- `parallel-result-order-is-completion-order` — `repair_premises` appends `_repair_one` results via `as_completed`, so `all_results` ordering differs from `premise_ids` under `parallel > 0`.
