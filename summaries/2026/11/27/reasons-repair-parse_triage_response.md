---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: parse_triage_response in reasons/repair.py

# `parse_triage_response` in `reasons/repair.py`

## Purpose

`parse_triage_response` extracts a triage decision from an LLM response. The triage prompt asks the model to emit a JSON object like:

```json
{"pattern": "search_and_link" | "soften" | "abandon" | "research", "rationale": "..."}
```

LLM output is often not pure JSON — it may include prose, Markdown fences, leading explanations, or malformed JSON fragments. This function scans the response for the first JSON object whose `"pattern"` value is one of the valid triage patterns and returns a normalized `{"pattern", "rationale"}` dict.

It exists to make triage parsing tolerant of imperfect LLM formatting while still rejecting unknown or malformed pattern values.

## Contract

### Preconditions

- `response` is expected to be a `str` containing LLM output.
- `VALID_PATTERNS` must contain the allowed pattern strings. It currently is:

```python
VALID_PATTERNS = {"search_and_link", "soften", "abandon", "research"}
```

### Postconditions

On normal return, the function returns a new dict with exactly two keys:

```python
{
    "pattern": <one of VALID_PATTERNS or "">,
    "rationale": <whatever obj.get("rationale", "") returned or "">,
}
```

Invariants:

- `pattern` is either `""` or one of the strings in `VALID_PATTERNS`.
- Missing `rationale` defaults to `""`.
- Extra fields in the parsed JSON object are dropped.
- The function does not mutate `response`.
- The function has no I/O or external side effects.

### Non-enforced assumptions

The type system does not enforce:

- That `response` is actually a `str`. Passing `None` will fail at `enumerate(response)`.
- That `"pattern"` is a string. If it is a JSON list or dict, the membership test `pattern not in VALID_PATTERNS` can raise `TypeError` because those values are unhashable.
- That `"rationale"` is a string. It is returned as-is if present.
- That the first JSON object with a valid pattern is the intended result. The function scans every `{`, including nested objects, and returns the first object that passes validation.

## Parameters

### `response`

Expected type: `str`.

Interpreted as raw LLM output. It may contain:

- Pure JSON.
- JSON surrounded by prose or Markdown.
- Multiple JSON-like fragments.
- Malformed JSON.
- Empty string.

Edge cases:

- `""` returns `{"pattern": "", "rationale": ""}`.
- `None` is not handled and will likely raise `TypeError`.
- Non-string iterables are not meaningfully supported.
- If the response contains a valid JSON object with a valid pattern anywhere, that object may be returned.

## Return Value

Returns a dict:

```python
{"pattern": str, "rationale": Any}
```

More precisely:

- If a JSON object is found with `"pattern"` in `VALID_PATTERNS`, returns its `pattern` and `rationale`.
- If no such object is found, returns:

```python
{"pattern": "", "rationale": ""}
```

Callers must treat an empty `"pattern"` as a parse failure / triage failure. In `repair_beliefs`, this is handled as:

```python
if not pattern:
    result["status"] = "triage_failed"
```

The caller should not assume `rationale` is a string unless it separately validates it.

## Algorithm

Step by step:

1. Create a reusable `json.JSONDecoder`.
2. Iterate over every character in `response`, tracking its index.
3. Skip any character that is not `{`.
4. For each `{`, call:

   ```python
   decoder.raw_decode(response, i)
   ```

   This attempts to parse a JSON value starting exactly at position `i`.

5. If `raw_decode` raises `json.JSONDecodeError`, ignore that candidate and continue scanning.
6. If the parsed value is not a `dict`, ignore it and continue.
7. Read:

   ```python
   pattern = obj.get("pattern", "")
   ```

8. If `pattern` is not in `VALID_PATTERNS`, ignore this object and continue scanning.
9. On the first valid object, return:

   ```python
   {
       "pattern": pattern,
       "rationale": obj.get("rationale", ""),
   }
   ```

10. If the loop finishes without finding a valid object, return the empty-pattern fallback.

Important behavior: because the function scans every `{`, it can return a valid JSON object nested inside a larger JSON object or embedded in text. It does not require the parsed object to be the first or only JSON value in the response.

## Side Effects

None.

The function:

- Does not print.
- Does not write to disk.
- Does not modify `response`.
- Does not call the LLM or database.
- Only allocates a decoder and result dicts.

## Error Handling

The function intentionally swallows `json.JSONDecodeError` for candidate `{` positions:

```python
except json.JSONDecodeError:
    continue
```

This lets it skip malformed JSON fragments and keep scanning.

It does not catch other exceptions. Potentially unhandled cases include:

- `response=None` causing `TypeError` at `enumerate(response)`.
- A non-string response causing errors inside `raw_decode`.
- A parsed JSON object with `"pattern"` equal to a list or dict, causing `TypeError` at `pattern not in VALID_PATTERNS` because the value is unhashable.
- Unexpected runtime errors from malformed or pathological JSON.

In normal LLM-output scenarios, malformed JSON is handled gracefully by returning the empty fallback. The caller should still guard against unexpected exceptions if the response source is untrusted.

## Usage Patterns

Typical use is through `triage_belief`:

```python
response = invoke_model(prompt, model=model, timeout=timeout)
return parse_triage_response(response)
```

Then `repair_beliefs` branches on the returned pattern:

```python
triage = triage_belief(...)
pattern = triage.get("pattern", "")

if not pattern:
    result["status"] = "triage_failed"
elif pattern == "search_and_link":
    ...
elif pattern == "soften":
    ...
```

Caller obligations:

- Check for an empty `"pattern"` before using it.
- Treat `"pattern" == ""` as a parse/triage failure.
- Do not assume `"rationale"` is a non-empty string.
- Prefer wrapping calls in the normal orchestration path where exceptions are converted into per-belief error results.

## Dependencies

- Standard library `json`, specifically `json.JSONDecoder`, `raw_decode`, and `json.JSONDecodeError`.
- Module-level `VALID_PATTERNS`.
- Used by `triage_belief`.
- Indirectly exercised by `repair_beliefs`.
- Related parsers: `parse_match_response`, `parse_soften_response`, which follow similar raw-decode scanning patterns.

## Topics to Explore

- [file] `tests/test_repair.py` — Covers triage and repair behavior, including likely cases for valid and invalid LLM JSON responses.
- [function] `reasons/repair.py:triage_belief` — Shows how this parser is invoked after the LLM call and how its result feeds into repair orchestration.
- [function] `reasons/repair.py:parse_match_response` — Parallel JSON parser with different validation rules for matched premise IDs.
- [function] `reasons/repair.py:parse_soften_response` — Parallel parser for softened belief text; useful for comparing fallback behavior.
- [general] `llm-json-extraction-robustness` — Explores the tradeoffs of scanning for brace-delimited JSON with `raw_decode` instead of using stricter parsing or schema validation.

## Beliefs

- `parse-triage-response-returns-first-valid-pattern` — It returns the first JSON object whose `"pattern"` value is in `VALID_PATTERNS`, skipping earlier malformed or invalid-pattern objects.
- `parse-triage-response-defaults-empty-pattern` — When no valid triage JSON object is found, it returns `{"pattern": "", "rationale": ""}`.
- `parse-triage-response-swallows-json-decode-errors` — It catches `json.JSONDecodeError` for candidate `{` positions and continues scanning.
- `parse-triage-response-rejects-invalid-patterns` — It only accepts `search_and_link`, `soften`, `abandon`, or `research`; other pattern values are skipped.
- `parse-triage-response-does-not-validate-rationale-type` — It copies `rationale` via `obj.get("rationale", "")` without checking that the value is a string.
