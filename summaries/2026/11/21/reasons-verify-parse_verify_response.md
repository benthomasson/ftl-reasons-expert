---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: parse_verify_response in reasons/verify.py

# `parse_verify_response`

## Purpose

`parse_verify_response` converts the raw text returned by an LLM into a normalized dictionary of verification verdicts. It exists because LLM responses are often not pure JSON: they may include prose, Markdown fences, or other text around the JSON object. The function extracts the JSON-like section, decodes it, and filters the decoded data down to entries that at least look like verification results.

It is used by `verify_beliefs` after `invoke_model(...)` returns the LLM response.

---

## Contract

### Preconditions

- `response` is expected to be a string or a falsy value such as `None` or `""`.
- If truthy, `response` is expected to contain a JSON object, possibly surrounded by non-JSON text.
- The decoded JSON is assumed to be a top-level object whose keys are belief IDs and whose values are verification-result objects.
- Each verification-result object is assumed to contain a string `"verdict"` field and optionally `"reason"` and `"quote"`.

### Postconditions

- Returns a `dict` mapping belief ID to a result dict of the form:
  ```python
  {
      "verdict": str,
      "reason": Any,
      "quote": Any,
  }
  ```
- `"verdict"` is always a string and is uppercased.
- `"reason"` defaults to `""` only when the key is absent. If `"reason"` exists but is `null`, the returned value is `None`.
- `"quote"` defaults to `None` when absent. If present, it is passed through unchanged.
- Entries are skipped if their value is not a dict or if `"verdict"` is missing or not a string.
- Parse failures return `{}`.
- The function does not guarantee that every original belief appears in the result.

### Invariants

- The function does not mutate `response`.
- The function performs no I/O.
- The returned verdict is never lowercased; it is always passed through `.upper()`.
- The function does not validate that the verdict is one of `CONFIRMED`, `STALE`, `PARTIAL`, or `INCONCLUSIVE`.
- The function does not validate that `reason` is a string.
- The function does not validate that `quote` is a verbatim quote from the source.

---

## Parameters

### `response`

Expected type: `str | None` in normal usage.

Behavior:

- Falsy values — `None`, `""`, `0`, `[]`, `{}`, etc. — immediately return `{}`.
- A truthy string is searched for a brace-delimited substring.
- A truthy non-string value can raise `TypeError` from `re.search(...)`, because that error is not caught.
- A `bytes` or `bytearray` response is likely incompatible with the string regex pattern and can also fail before the guarded `json.loads` call.

The function implicitly assumes the LLM returned text containing one JSON object.

---

## Return Value

Returns a `dict`.

Examples of possible outcomes:

- `None` → `{}`
- `""` → `{}`
- `"no json here"` → `{}`
- `'{"b1": {"verdict": "confirmed"}}'` →
  ```python
  {
      "b1": {
          "verdict": "CONFIRMED",
          "reason": "",
          "quote": None,
      }
  }
  ```
- `'{"b1": {"verdict": 123}}'` → `{}` because the verdict is not a string.
- `'{"b1": "not a dict"}'` → `{}` because the value is not a dict.

The caller must handle:

- An empty result dict, which may mean either no input response, no JSON found, invalid JSON, or no valid entries.
- Missing belief IDs.
- Verdict strings outside the expected four values.
- `reason` or `quote` values that are not strings.
- Possible exceptions for malformed top-level JSON types, such as a JSON array.

---

## Algorithm

1. If `response` is falsy, return `{}`.
2. Search `response` with:
   ```python
   re.search(r"\{.*\}", response, re.DOTALL)
   ```
   This is a greedy match from the first `{` to the last `}`. `DOTALL` makes `.` match newlines.
3. If no brace-delimited match is found, return `{}`.
4. Attempt to decode the matched substring with `json.loads`.
5. If decoding raises `json.JSONDecodeError` or `TypeError`, return `{}`.
6. Iterate over `data.items()`.
7. For each key/value pair:
   - Skip the entry if `v` is not a dict.
   - Skip the entry if `"verdict"` is not present.
   - Skip the entry if `v["verdict"]` is not a string.
   - Otherwise store:
     ```python
     results[k] = {
         "verdict": raw_verdict.upper(),
         "reason": v.get("reason", ""),
         "quote": v.get("quote"),
     }
     ```
8. Return `results`.

Important detail: the regex is greedy and not JSON-aware. If the surrounding text contains additional braces, or if the response contains multiple JSON objects, the extracted span may be invalid JSON and the function will usually return `{}`.

---

## Side Effects

None.

The function:

- Does not perform file or network I/O.
- Does not mutate `response`.
- Does not mutate global state.
- Does not call external services.

It only uses regex matching, JSON decoding, and local dictionary construction.

---

## Error Handling

### Errors swallowed

- Falsy `response` → returns `{}`.
- No `{...}` match → returns `{}`.
- `json.JSONDecodeError` from invalid JSON → returns `{}`.
- `TypeError` from `json.loads(...)` → returns `{}`.

### Errors not swallowed

- `re.search(...)` can raise `TypeError` if `response` is truthy but not a string/bytes-compatible value.
- If the extracted JSON decodes to a non-dict top-level value, `data.items()` raises `AttributeError`. For example:
  ```python
  data = [1, 2, 3]
  data.items()  # AttributeError
  ```
- This function does not catch `AttributeError`.

So the actual error-handling boundary is narrower than it may first appear. It protects against bad JSON text, but not against every malformed LLM response shape.

---

## Assumptions Not Enforced by the Type System

- The decoded JSON is assumed to be a top-level object. A JSON array, string, or number will cause `AttributeError` at `data.items()`.
- The JSON keys are assumed to be belief IDs. The function does not verify them against the original belief set.
- The `"verdict"` value is assumed to be one of the four expected verdicts, but any string is accepted and uppercased.
- The `"reason"` value is assumed to be a string if present, but `None`, lists, or dicts are passed through.
- The `"quote"` value is assumed to represent a verbatim source quote, but no source comparison or validation occurs here.
- The response is assumed to contain at most one brace-delimited JSON object, or at least no extra braces outside the intended object.
- The function assumes `response` is a text string in normal use; truthy non-string inputs are not safely handled.

---

## Usage Patterns

Typical call site:

```python
response = invoke_model(prompt, model=model, timeout=timeout)
return parse_verify_response(response)
```

Callers should treat `{}` as a parse failure or an empty verification result. They should also be prepared for partial results: the LLM may omit some belief IDs, or this parser may skip entries with malformed verdicts.

Because the parser does not validate verdict values, callers may want to normalize or reject unexpected verdicts downstream if they require only the four documented values.

---

## Dependencies

- Standard library `json` for decoding.
- Standard library `re` for extracting the brace-delimited substring.
- Implicit dependency on the `VERIFY_PROMPT` response schema, which asks the LLM to return a JSON object mapping belief IDs to `{ "verdict", "reason", "quote" }`.
- No local module imports inside `parse_verify_response` itself.

---

## Topics to Explore

- [file] `reasons/verify.py` — Full verification pipeline including prompt construction, source reading, LLM invocation, and response parsing.
- [function] `reasons/verify.py:verify_beliefs` — Orchestrates formatting beliefs, calling the LLM, and passing the raw response into `parse_verify_response`.
- [function] `reasons/verify.py:read_source` — Handles source resolution and truncation before verification prompts are built.
- [file] `reasons/check_stale.py` — Provides `resolve_source_path`, which `read_source` depends on for resolving source files.
- [file] `tests/test_verify.py` — Pins expected behavior for valid responses, malformed JSON, markdown-wrapped JSON, and invalid verdict entries.

## Beliefs

- `parse-verify-response-returns-empty-dict-on-falsy-response` — A falsy `response` such as `None` or `""` immediately returns `{}` before any regex search.
- `parse-verify-response-extracts-greedy-brace-span` — It uses `r"\{.*\}"` with `DOTALL`, matching from the first `{` to the last `}`, not a non-greedy or JSON-aware extraction.
- `parse-verify-response-assumes-top-level-json-object` — If the extracted JSON decodes to a non-dict, `data.items()` raises `AttributeError`, which is not caught.
- `parse-verify-response-skips-entries-without-string-verdict` — Entries are omitted unless the value is a dict and its `"verdict"` field is a string.
- `parse-verify-response-uppercases-verdict-without-validation` — The parser uppercases any string verdict but does not check that it is one of `CONFIRMED`, `STALE`, `PARTIAL`, or `INCONCLUSIVE`.
