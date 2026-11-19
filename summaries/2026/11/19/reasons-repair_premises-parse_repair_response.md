---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: parse_repair_response in reasons/repair_premises.py

## Purpose

`parse_repair_response` extracts a JSON array of repair instructions from a free-form LLM response. It exists because the model is prompted to return “ONLY a JSON array,” but in practice responses may include prose, markdown fences, or malformed prefixes. The parser searches for a usable JSON array, normalizes each accepted item to the repair-result schema, and silently discards malformed items.

It does **not** perform semantic validation of repair actions; it only checks that each accepted item is a dict with an `"id"` key.

## Contract

### Preconditions

- `response` is expected to be a `str`.
- The caller expects the LLM response to eventually contain a JSON array whose items are objects with an `"id"` field.
- The caller generally expects one repair object per premise, but the parser allows multiple and returns all valid items from the first usable array.

### Postconditions

- Returns a `list`.
- Each returned element is a new dict with exactly these keys:
  - `"id"`
  - `"action"`
  - `"corrected_text"`
  - `"rationale"`
- `"id"` comes from `item["id"]`; no type validation is performed.
- `"action"` defaults to `"retract"` only when the key is missing. If present, it is copied unchanged, even if it is `None`, an invalid string, or another type.
- `"corrected_text"` defaults to `None` when missing or null.
- `"rationale"` defaults to `""` only when the key is missing. If present as `None`, the returned rationale is `None`.
- The returned list is non-empty when the function finds a decoded array containing at least one dict with an `"id"` key.
- If no usable array is found, returns `[]`.

### Invariants

- Does not mutate `response` or any decoded objects.
- Returns the first decoded array that yields at least one valid normalized item.
- Does not merge items from multiple arrays.
- Within an accepted array, invalid items are skipped rather than rejecting the whole array.
- It does not verify that `"id"` matches the premise being repaired.
- It does not validate `"action"` against `{"rewrite", "retract"}`.
- It does not require `"corrected_text"` when action is `"rewrite"`.

## Parameters

### `response`

Expected type: `str`.

The raw model output. It may contain markdown fences, explanatory text before or after the JSON, or malformed fragments. The parser scans for every `[` character and attempts to decode JSON starting at that position.

Edge cases:

- Empty string returns `[]`.
- A non-string value is not explicitly checked:
  - `None` or an integer will raise `TypeError` from `enumerate`.
  - `bytes` will iterate as integers; no character will equal `"["`, so it returns `[]`.
  - A list of one-character strings may be scanned like a string.
- If the response contains multiple arrays, the first array that yields at least one valid item wins.
- If the first valid array is empty or contains only invalid items, the scan continues to later `[` characters.
- Because it scans raw text, it can find arrays inside markdown code fences or prose. It can also find arrays embedded in string literals if they appear earlier than a usable outer array.

## Return Value

Returns a list of normalized repair result dictionaries.

Example shape:

```python
[
    {
        "id": "premise-1",
        "action": "rewrite",
        "corrected_text": "Corrected claim.",
        "rationale": "Original claim contradicted the source.",
    }
]
```

The caller must handle:

- `[]` meaning “no usable repair response was parsed.”
- Returned `"id"` values not matching the requested premise ID.
- Invalid or unexpected `"action"` values from the model.
- Non-string or missing semantic fields, because only key-presence/shape filtering is performed.

In the current file, `_repair_one` calls this function and uses only `parsed[0]` if the list is non-empty. It does not verify that `parsed[0]["id"] == node_id`.

## Algorithm

1. Create a reusable `json.JSONDecoder()`.
2. Iterate through `response` character by character with index `i`.
3. Ignore every character except `"["`.
4. For each `"["`, call:

   ```python
   items, _ = decoder.raw_decode(response, i)
   ```

   This attempts to decode any JSON value starting at index `i`. Since the character at `i` is `"["`, a successful decode should produce a JSON array.

5. If decoding raises `json.JSONDecodeError`, continue scanning at later `[` characters.
6. If the decoded value is not a list, continue. This is mostly defensive because decoding from `"["` should normally yield a list if it yields anything.
7. For each item in the decoded list:
   - Skip it unless it is a `dict`.
   - Skip it unless it contains an `"id"` key.
   - Otherwise append a normalized dict with `id`, `action`, `corrected_text`, and `rationale`.
8. If the normalized `results` list is non-empty, return it immediately.
9. If the decoded array was empty or all items were invalid, continue scanning for another `[` later in the response.
10. If no usable array is found, return `[]`.

Important detail: the parser returns the first array that produces at least one valid normalized item. It does not require the decoded array to be the first syntactically valid JSON array if that array has no usable items.

## Side Effects

None.

The function:

- Does not perform I/O.
- Does not log.
- Does not mutate input.
- Does not call external services.
- Only allocates a decoder and result dictionaries.

## Error Handling

- Catches only `json.JSONDecodeError` for each candidate `[` start. Such failures are swallowed and scanning continues.
- Does not catch other exceptions.
- Invalid item shapes are not treated as errors; they are silently skipped.
- A decoded array containing some invalid items and some valid items returns only the valid items.
- Invalid `"action"` values are not rejected.
- Missing `"id"` causes the item to be skipped entirely.
- Non-dict list entries are skipped.
- Non-string `response` values may raise or silently return `[]` depending on the type, because there is no explicit type check.

## Usage Patterns

The function is designed to wrap LLM output parsing:

```python
response = invoke_model(prompt, model=model, timeout=timeout)
parsed = parse_repair_response(response)
if parsed:
    return parsed[0]
return {
    "id": node_id,
    "action": "retract",
    "corrected_text": None,
    "rationale": "Could not parse repair response",
}
```

Caller obligations in the current code:

- `_repair_one` expects one item per response and uses `parsed[0]`.
- If the parser returns `[]`, `_repair_one` fabricates a retract fallback.
- If the parser returns an item with the wrong `"id"` or invalid `"action"`, `_repair_one` propagates it.
- Higher-level repair orchestration does not validate parser output further.

## Dependencies

This function depends only on the standard library `json` module, specifically `json.JSONDecoder` and `json.JSONDecodeError`.

The surrounding module also depends on:

- `concurrent.futures.ThreadPoolExecutor`, `as_completed`
- `reasons.llm.invoke_model`
- `sys`

But `parse_repair_response` itself only uses `json`.

## Topics to Explore

- [file] `tests/test_repair_premises.py` — Covers parsing behavior and repair workflow edge cases.
- [function] `reasons/repair_premises.py:_repair_one` — Shows how parsed results are consumed and how parse failure becomes a retract fallback.
- [function] `reasons/repair_premises.py:repair_premises` — Orchestrates sequential/parallel repairs and result callbacks.
- [general] `llm-json-response-parsing` — Robustness patterns for extracting JSON from free-form model output.
- [general] `premise-repair-actions` — Domain semantics of `rewrite` versus `retract` in the Truth Maintenance System.

## Beliefs

- `parse-repair-response-returns-normalized-dicts` — Every returned item contains exactly the keys `id`, `action`, `corrected_text`, and `rationale`.
- `parse-repair-response-skips-items-without-id` — Items that are not dictionaries or lack an `"id"` key are filtered out before returning results.
- `parse-repair-response-searches-all-bracket-candidates` — The parser scans every `[` and returns the first decoded array that yields at least one valid item.
- `parse-repair-response-does-not-validate-action` — Missing `action` defaults to `"retract"`, but any present action value is accepted unchanged.
- `repair-one-uses-first-parsed-item` — `_repair_one` treats any non-empty parse result as success and uses only `parsed[0]`.
