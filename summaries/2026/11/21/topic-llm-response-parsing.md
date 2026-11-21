---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Explore how the production parser locates JSON arrays, strips fences, and normalizes proposal fields.

## Production parser: `parse_update_proposals`

The relevant production parser is `reasons/propose_update.py:parse_update_proposals` (`reasons/propose_update.py:164`). It is called by `propose_updates` after each LLM batch at `reasons/propose_update.py:324`. `reasons_lib/propose_update.py` is only a backwards-compatibility re-export shim.

### 1. How it locates JSON arrays

The parser does **not** use a regex or a separate “find the JSON block” helper. It scans the response character by character:

```python
decoder = json.JSONDecoder()
for i, ch in enumerate(response):
    if ch != "[":
        continue
    try:
        items, _ = decoder.raw_decode(response, i)
    except json.JSONDecodeError:
        continue
    if not isinstance(items, list):
        continue
```

So it:

1. Tries every `[` position as a possible JSON array start.
2. Calls `json.JSONDecoder.raw_decode(response, i)` from that exact index.
3. Ignores the decoded end index (`_`), which is why trailing prose after the array is harmless.
4. Continues on `JSONDecodeError` or non-list JSON values.
5. Keeps scanning if a decoded list contains no valid proposal dictionaries.

This also explains an important behavioral detail: the function returns the **first non-empty normalized result**. An empty `[]` or an array containing only invalid items does not stop the scan early; the loop continues looking for another candidate `[`. If nothing valid is found, it returns `[]`.

### 2. How it handles fenced JSON

There is **no explicit Markdown fence stripping** in `parse_update_proposals`. The docstring at `reasons/propose_update.py:165-168` says it uses `raw_decode` at each `[` to handle prose around the JSON array.

Fenced responses such as:

```markdown
```json
[
  {"id": "x", ...}
]
```
```

work because the parser finds the `[` inside the fence and `raw_decode` parses from there. The leading ` ```json ` is just preceding prose; the trailing ` ``` ` is just trailing data ignored by `raw_decode`.

This is covered by `tests/test_propose_update.py::TestParseUpdateProposals::test_valid_json`, which passes a response with prose, a ` ```json ` fence, and a trailing fence, and by `test_prose_around_json`, which verifies prose before and after the array.

### 3. How it normalizes proposal fields

Normalization is inline in `parse_update_proposals`; there is no separate `normalize_proposals` function in this module.

For each decoded list item:

| Field | Normalization rule |
|---|---|
| raw item | Must be a `dict` and must contain `"id"`; otherwise skipped |
| `id` | Preserved as `item["id"]` |
| `action` | Defaults to `"update"`; if not `"update"` or `"retract"`, forced to `"update"` |
| `proposed_text` | `item.get("proposed_text")`, so missing becomes `None` |
| `failure_mode` | Defaults to `""`; if not in module-level `FAILURE_MODES`, becomes `""` |
| `basis` | Defaults to `"prior-knowledge"`; if not in module-level `BASES`, becomes `"prior-knowledge"` |
| `evidence` | Missing becomes `""` |
| `comment` | Missing becomes `""` |

The returned dictionaries therefore always contain exactly these normalized keys:

```python
{
    "id": ...,
    "action": ...,
    "proposed_text": ...,
    "failure_mode": ...,
    "basis": ...,
    "evidence": ...,
    "comment": ...,
}
```

Extra LLM fields are dropped. The parser also does **not** enforce semantic prompt rules such as “update proposed_text must differ from current text” or “retract must set proposed_text to null”; it only normalizes the taxonomy fields and defaults.

The tests confirm the defaulting behavior: `test_invalid_basis_defaults`, `test_invalid_failure_mode_defaults`, and `test_invalid_action_defaults` in `tests/test_propose_update.py`.

### Scope note

The observations show the full implementation of `reasons/propose_update.py`, so the update-proposal parser behavior above is directly supported. Other modules such as `reasons/review.py` and `reasons/repair_premises.py` also appear to contain JSON-array parsers, but their implementations were not included in the observations, so their fence-handling details cannot be confirmed here.

## Topics to Explore

- [file] `reasons/derive.py` — Contains `parse_proposals` at line 377, a different proposal parser using `### DERIVE / ### GATE` text format rather than JSON arrays.
- [function] `reasons/propose_update.py:propose_updates` — Shows batching, LLM invocation, and how `parse_update_proposals` is called at line 324 with per-batch error handling.
- [file] `reasons/review.py` — Has another JSON-array extraction path around line 181 with docstring language about prose brackets and trailing text.
- [file] `tests/test_propose_update.py` — Codifies fenced JSON, prose-around-JSON, malformed JSON, missing IDs, and invalid taxonomy defaults.
- [general] `llm-json-extraction-pattern` — Compare the parsers named in the search results to see whether `raw_decode`-at-each-`[` is the common production extraction pattern.

## Beliefs

- `parse-update-proposals-scans-open-brackets` — `parse_update_proposals` iterates over every character and attempts `JSONDecoder.raw_decode` only at positions where the character is `[`.
- `parse-update-proposals-does-not-strip-fences` — It tolerates Markdown code fences by decoding the JSON array inside them, not by removing the fence markers.
- `parse-update-proposals-requires-id` — List items that are not dictionaries or that lack an `id` key are skipped during normalization.
- `parse-update-proposals-defaults-taxonomy-fields` — Invalid or missing `action`, `failure_mode`, and `basis` values normalize to `"update"`, `""`, and `"prior-knowledge"`, respectively.
- `parse-update-proposals-returns-first-nonempty-array` — The parser continues past decoded lists that produce no valid proposal dictionaries and returns the first non-empty normalized result.
