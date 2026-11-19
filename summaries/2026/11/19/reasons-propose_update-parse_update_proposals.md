---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: parse_update_proposals in reasons/propose_update.py

## `parse_update_proposals` — Explanation

### Purpose

This function is the "defensive parsing" step between an LLM's free-form text output and the rest of the update-proposal pipeline. `PROPOSE_UPDATE_PROMPT` asks the model to return *only* a JSON array, but in practice LLM responses often include prose, markdown fences, or example arrays echoed from the prompt. `parse_update_proposals` scans the raw response, finds a decodable JSON array, sanitizes each element into a known schema, and returns a clean list of proposal dicts. The output feeds `format_proposal_markdown` / `format_proposals_file`, which assume a strict key shape.

### Contract

**Preconditions**
- `response` should be `str`. Nothing enforces this at runtime; non-iterable or non-stringable inputs will raise.
- Module-level `FAILURE_MODES` and `BASES` must be defined (they are, as sets).

**Postconditions on a valid result**
- Returns a **non-empty** `list[dict]`, where each dict has exactly the keys: `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`.
- `action ∈ {"update", "retract"}`.
- `failure_mode ∈ FAILURE_MODES ∪ {""}`.
- `basis ∈ BASES`.
- `evidence` and `comment` are `str` (default `""`) — but **only if the LLM supplied a string**; if the LLM sent a different type, it's passed through verbatim.
- `proposed_text` may be `None` (retraction) or any value the LLM supplied.

**Postconditions on failure**
- Returns `[]` — perfectly valid "no proposals" signal to the caller.

**Invariants**
- Never raises `json.JSONDecodeError` — that exception is explicitly swallowed.
- Doesn't mutate `response` or any module-level state.

### Parameters

| Name | Meaning | Expected type | Edge cases |
|------|---------|---------------|------------|
| `response` | Raw text returned by the LLM | `str` | Empty string → returns `[]`. Contains `[` in prose, in strings, or as `[]` empty array → those are skipped until a non-empty parse yields results. A non-`str` (e.g. `None`) → `enumerate` raises `TypeError`. |

### Return Value

`list[dict]`. The function tries each `[` in order and **returns at the first `[` that yields at least one valid item**. It does *not* merge multiple arrays and does *not* attempt to pick the "best" one. If no `[` produces a non-empty `results` list, it returns `[]`.

Caller contract (informal): the caller (`propose_updates`) uses `all_results.extend(results)`, so `[]` is treated as "this batch produced no proposals", which is indistinguishable from "the LLM returned garbage". That's intentional — failures are soft.

### Algorithm

1. Build a `json.JSONDecoder` once.
2. Scan `response` character by character. Skip anything that isn't `[`.
3. At each `[` index `i`, call `decoder.raw_decode(response, i)`. `raw_decode` attempts to parse JSON starting exactly at `i` and returns `(obj, end_index)`; it tolerates trailing content (that's why it's used instead of `json.loads`).
4. If parsing raises `JSONDecodeError` (unbalanced brackets, `[` inside prose, string content, etc.), `continue` to the next `[`.
5. If parsing succeeds but the decoded object isn't a `list`, `continue`.
6. Iterate the list. For each element:
   - Skip unless it's a `dict` **with an `"id"` key**. (Everything else — lists, scalars, dicts missing `id` — is silently dropped.)
   - **Normalize `action`**: default `"update"`; any value outside `{"update","retract"}` is coerced to `"update"`.
   - **Normalize `failure_mode`**: default `""`; any value not in `FAILURE_MODES` is coerced to `""`.
   - **Normalize `basis`**: default `"prior-knowledge"`; any value not in `BASES` is coerced to `"prior-knowledge"`.
   - Append a **fresh** dict with the seven canonical keys, pulling the rest via `.get(..., "")` / `.get("proposed_text")`.
7. If `results` is non-empty, **return immediately**. If it's empty (e.g., the `[` was `[]` or all items were invalid), fall through and continue scanning to the next `[`.
8. If the loop exhausts, return `[]`.

Notable behavior: `raw_decode` is not line-oriented. It will attempt positions inside JSON strings too, but `raw_decode` will fail there because the char is `[` inside a quoted string — the parser stops at the opening quote of the enclosing string, so those attempts are usually harmless.

### Side Effects

- **None.** Pure function; no I/O, no mutation of arguments or module state. The only implicit "external" is the module-level `FAILURE_MODES` and `BASES` sets.

### Error Handling

- **Swallowed**: `json.JSONDecodeError` from `raw_decode` at each `[` attempt.
- **Propagated**: any other exception. A `TypeError` from `enumerate(response)` if `response` is `None`/non-iterable; a `TypeError` from `raw_decode` if `response` is not a `str` (e.g. bytes).
- **Logical degradation**: an item still passes through if the LLM supplied wrong-typed `id`, `evidence`, `comment` — there's no type check on those fields. Downstream `format_proposal_markdown` assumes `evidence` and `comment` are strings when interpolating.

### Usage Patterns

Called once per batch inside `propose_updates`:

```python
response = invoke_model(prompt, model=model, timeout=timeout)
results = parse_update_proposals(response)
all_results.extend(results)
```

Typical obligations for callers:
- Treat `[]` as "no proposals" — do not assume it means "malformed response" (though it may).
- Do not assume `id` is a valid node key — downstream code (e.g. `format_proposal_markdown`) does `nodes.get(nid, {})` so unknown IDs degrade gracefully, but a caller planning to write back to the graph should validate existence.
- Do not rely on `proposed_text` being a string — retractions set it to `None` by convention.

### Dependencies

- `json` (stdlib) — specifically `JSONDecoder.raw_decode`.
- Module-level constants `FAILURE_MODES` and `BASES` (must stay in sync with the prompt taxonomy).
- Indirect: `propose_updates` (which calls `invoke_model` from `reasons.llm`) provides the raw response this function parses.

### Assumptions Not Enforced by Types

- **`response` is a `str`** — no `isinstance` check, no graceful failure.
- **First non-empty array wins** — if the LLM echoes an example array from the prompt (which the prompt *does* include), it could win over the real answer. Mitigation relies on prompt engineering and LLM behavior, not code.
- **Item values are strings where expected** — `evidence`, `comment`, `id` aren't type-checked; a dict or number will slip through and may format oddly downstream.
- **`id` refers to an existing belief** — the parser has no access to `nodes`, so it cannot filter IDs. Downstream code is expected to handle unknown IDs.
- **Items remain dicts** — the parser doesn't deep-copy or restrict payload keys; if the LLM includes extra fields like `"confidence"`, they're silently dropped (only the seven canonical keys survive).

## Topics to Explore

- [function] `reasons/propose_update.py:propose_updates` — The caller that batches beliefs, builds the prompt, and merges `parse_update_proposals` output; shows how parse failures degrade.
- [function] `reasons/propose_update.py:format_belief_for_update` — Serializes each belief; understanding this helps explain what the LLM actually sees and why arrays might appear echoed in the response.
- [function] `reasons/llm.py:invoke_model` — Determines the format/shape of `response`; verify whether it strips markdown fences before returning.
- [file] `tests/test_propose_update.py` — Documents the accepted edge cases (empty array, prose-wrapped JSON, invalid taxonomy values).
- [file] `reasons/propose_update.py:PROPOSE_UPDATE_PROMPT` — The taxonomy contract (`FAILURE_MODES`, `BASES`) originates here; drift between prompt and code would cause silent normalization to defaults.

## Beliefs

- `parse-update-proposals-uses-raw-decode` — `parse_update_proposals` uses `json.JSONDecoder.raw_decode` at every `[` character to extract a JSON array from prose around it.
- `parse-update-proposals-swallows-json-errors` — `JSONDecodeError` is caught and the scan continues; the function never propagates JSON parse errors.
- `parse-update-proposals-returns-first-nonempty` — The function returns at the first `[` whose decoded list yields at least one valid item; later arrays in the same response are ignored.
- `parse-update-proposals-requires-id` — Items that are not dicts or that lack an `"id"` key are silently dropped.
- `parse-update-proposals-normalizes-taxonomy` — Invalid `action`, `failure_mode`, and `basis` values are coerced to `"update"`, `""`, and `"prior-knowledge"` respectively.
- `parse-update-proposals-may-raise-on-non-str` — Passing `None` or a non-string `response` raises `TypeError` from `enumerate`/`raw_decode` rather than returning `[]`.
