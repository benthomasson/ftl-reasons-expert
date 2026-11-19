---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: parse_review_response in reasons/review.py

## `parse_review_response` — Explanation

### Purpose

Parses the raw text returned by the LLM reviewer into a normalized list of review-result dictionaries. LLM responses are adversarial to `json.loads`: they may wrap the array in markdown code fences, prepend prose containing literal `[` characters (e.g., `[Note]`, citations), append commentary after the array, or emit multiple bracket groups. This function exists to extract *the* review array robustly, and to normalize each item into a stable schema regardless of which optional fields the model omitted.

### Contract

- **Precondition:** `response` is a `str` containing arbitrary text (potentially one or more JSON arrays plus prose).
- **Postcondition:** Returns a `list[dict]`. Each dict is guaranteed to contain the keys `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`, `scope_findings`, `missing_property`, `defeat_reason_type` (with defaults filled in). Returns `[]` if nothing usable was found.
- **Invariant:** Every returned element is a `dict` with an `"id"` key. No type checking is applied to the *values* — `valid` could be the string `"true"` if the model emitted that.
- **No exceptions escape** from parse failures; `json.JSONDecodeError` is caught per candidate position.

### Parameters

| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `response` | `str` | Raw model output to parse. | Empty string → returns `[]`. Non-str iterables technically iterate but `raw_decode(response, i)` requires a `str`, so passing bytes/`None` will fail (TypeError not caught). Multiple candidate arrays: first *non-empty* qualifying list wins. |

### Return Value

A list of sanitized dicts, each shaped:

```python
{
  "id": <whatever was in item["id"]>,           # required; unvalidated type
  "valid": item.get("valid", True),              # default True
  "sufficient": item.get("sufficient", True),    # default True
  "necessary": item.get("necessary", True),      # default True
  "unnecessary_antecedents": item.get(..., []),
  "comment": item.get("comment", ""),
  "scope_findings": item.get("scope_findings", []),
  "missing_property": item.get("missing_property", ""),
  "defeat_reason_type": item.get("defeat_reason_type", ""),
}
```

Returns `[]` if no candidate `[` produces a JSON list containing at least one dict with an `"id"`. Notably, an empty `[]` array in the response (or one whose items all fail the `id` filter) is **not** treated as success — the loop keeps scanning subsequent `[` positions.

### Algorithm

1. Construct a single `json.JSONDecoder`.
2. Scan `response` character-by-character with `enumerate`.
3. Skip any char that is not `[`.
4. At each `[`, invoke `decoder.raw_decode(response, i)`. This parses the *largest valid JSON value* starting exactly at index `i` and ignores anything after it (that's how trailing prose is tolerated). On `JSONDecodeError`, continue to the next `[`.
5. Defensive check: the decoded value must be a `list`. (Because we only attempt at `[`, this is virtually always true, but the check guards against exotic inputs and documents intent.)
6. Build a fresh `results` list: for each item, skip non-dicts and dicts missing `"id"`; otherwise emit a normalized dict.
7. If `results` is non-empty, return immediately (first-wins semantics).
8. If the loop exhausts all `[` positions without a qualifying non-empty list, return `[]`.

The "first wins" + "empty results doesn't count" combination is the key non-obvious behavior: prose like `[see section 3]` fails to decode and is skipped; a `[Note]` style prose bracket also fails; the real array decodes and (if it contains any valid dict) is returned.

### Side Effects

None. Pure function — no I/O, no mutation of arguments, no global state.

### Error Handling

- `json.JSONDecodeError` at each candidate `[` → caught, scan continues.
- Other exceptions (e.g., `TypeError` if `response` is not a `str` at `raw_decode`) → **not** caught, will propagate.
- Malformed individual items (wrong type, missing `id`) → silently dropped.
- Malformed individual *fields* (e.g., `valid` is a string, `scope_findings` is a dict) → passed through unmodified. Downstream consumers must tolerate this.
- Total failure to parse anything → returns `[]`. This is indistinguishable from "the model legitimately found zero results," so callers cannot tell parse failure from empty review set.

### Usage Patterns

Called only from `review_beliefs` (in the same file), once per batch, on the `invoke_model` response:

```python
response = invoke_model(prompt, model=model, timeout=timeout)
results = parse_review_response(response)
all_results.extend(results)
```

The caller treats `[]` as a no-op extension. Because the `try/except` around this block only catches raised exceptions, a silent parse failure produces no warning — a batch can appear to succeed with fewer results than beliefs sent. Callers that need strict cardinality (one result per belief) must check `len(results)` themselves.

### Dependencies

- Standard library only: `json.JSONDecoder.raw_decode`.
- No internal dependencies within `reasons/`. It sits *between* `invoke_model` (LLM I/O) and `review_beliefs` (orchestration).

### Type-System Gaps (assumptions not enforced)

- Assumes `response` is `str`; there is no `isinstance` guard.
- Assumes the model returned booleans for `valid`/`sufficient`/`necessary`; defaults are `True`, so a missing field silently becomes "believed valid".
- Assumes `scope_findings`, `unnecessary_antecedents` are lists and `comment`/`missing_property`/`defeat_reason_type` are strings — but nothing verifies the types of the extracted values.
- Assumes `item["id"]` is meaningful (string); any truthy-or-not value is accepted, including `None` or nested objects.
- Assumes the first qualifying non-empty array is the review array. If the response embeds a *different* valid JSON array (e.g., an example from the prompt) before the real one, that will be returned instead.

## Topics to Explore

- [function] `reasons/review.py:review_beliefs` — The orchestrator that calls this parser per batch; observe how it treats empty results and batch failures.
- [function] `reasons/review.py:format_belief_for_review` — How the belief/justification text that the model reviews is constructed (determines what fields the parser is expected to receive).
- [function] `reasons/llm.py:invoke_model` — The upstream I/O that produces the raw response string this function must tolerate.
- [file] `tests/test_review.py` — Existing tests for parsing behavior (prose brackets, code fences, malformed items, defaults).
- [general] `robust-llm-json-parsing` — Alternative strategies (regex extraction, `json5`, partial framing) compared to `raw_decode` scanning.

## Beliefs

- `parse-review-response-is-pure` — The function performs no I/O, mutates no arguments, and cannot raise on malformed JSON.
- `parse-review-response-returns-empty-list-on-failure` — Returns `[]` when no `[` position decodes to a list containing at least one dict with an `"id"` key.
- `parse-review-response-normalizes-schema` — Every emitted dict contains all nine canonical keys, with `True`/`[]`/`""` defaults when the model omitted them.
- `parse-review-response-filters-dicts-without-id` — Non-dict items and dicts missing `"id"` are silently dropped from the returned list.
- `parse-review-response-skips-empty-filtered-lists` — A successfully decoded but empty-or-fully-filtered list does not end the scan; later `[` positions continue to be tried.
