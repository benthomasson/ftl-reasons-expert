---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How `JSONDecoder.raw_decode` differs from `json.loads` regarding trailing data and multiple values.

## How `JSONDecoder.raw_decode` differs from `json.loads`

The short version:

- **`json.loads(s)` expects the entire string to be exactly one JSON document** (ignoring leading/trailing whitespace). If there is any non-whitespace after the first JSON value, it raises `JSONDecodeError` with an “Extra data” message.
- **`JSONDecoder.raw_decode(s, idx)` decodes one JSON value starting at `idx`** and returns `(value, end_index)`. It does **not** require `end_index == len(s)`, so trailing text after a complete JSON value is ignored.
- `raw_decode` can be used to parse multiple concatenated values only by feeding the returned `end_index` back into another `raw_decode` call. The observed callers in this codebase **discard** that index with `items, _ = decoder.raw_decode(...)`, so they are not parsing multiple values; they are extracting one decodable JSON value from an offset.

### Comparison

| API | Input expectation | Trailing data | Return value | Multiple top-level values |
|---|---|---|---|---|
| `json.loads(s)` | Whole string is one JSON document | Raises `JSONDecodeError` (“Extra data”) after optional trailing whitespace | The decoded object | Not supported in one call |
| `JSONDecoder.raw_decode(s, idx)` | A JSON value begins at `idx` | Ignored after the first complete value | `(decoded_object, end_index)` | Supported only by calling again with `end_index` |

### What the codebase is doing

The repair/review/propose modules use `raw_decode` to tolerate LLM responses that contain JSON embedded in prose or Markdown. For example:

- `reasons/propose_update.py:167` says it “Uses json.JSONDecoder.raw_decode at each `[` position”; it creates `decoder = json.JSONDecoder()` at line 170 and calls `items, _ = decoder.raw_decode(response, i)` at line 175.
- `reasons/review.py:183` has the same “at each `[` position” comment, creates the decoder at line 186, and calls `items, _ = decoder.raw_decode(response, i)` at line 191.
- `reasons/repair.py:146` says it “Uses raw_decode to find JSON object in response text”; it creates the decoder at line 149 and calls `obj, _ = decoder.raw_decode(response, i)` at line 154, then again at lines 211 and 233.
- Similar patterns appear in `reasons/repair_premises.py:83/88`, `reasons/review_justifications.py:69/74`, and `reasons/review_premises.py:88/93`.

In those places, the caller is responsible for finding candidate start offsets—typically each `[` or `{`—and trying `raw_decode` from that offset. If the text before or after the JSON is prose, `raw_decode` can still succeed because it stops at the end of the first complete JSON value. `json.loads` would instead reject the whole string as having “Extra data.”

The ignored `_` end index is important: these callers are **not** extracting multiple JSON values sequentially. They are scanning candidate positions and taking the first valid value at that position. To parse multiple values with `raw_decode`, code would need to use the returned end index, skip whitespace, and call `raw_decode` again.

By contrast, `json.loads` is used where the input is expected to be a single exact JSON document, such as:

- `reasons/llm.py:129` in `_parse_cli_json`: `data = json.loads(output)` parses CLI JSON output.
- `reasons/api.py:2423`, `reasons/api.py:4026`, and many `reasons/pg.py` lines parse stored JSON columns or captured JSON substrings.
- `reasons/ask.py:122` parses a line as one JSON object.

Those uses assume a single JSON document and would fail on trailing non-whitespace data.

### Caveat about the observations

The snippets show decoder construction and `raw_decode` calls, but not the complete surrounding loops or exception handling. So the exact candidate-scanning and fallback behavior in `repair.py`, `review.py`, etc. is inferred from the comments and call sites; the full loops are not included in the provided observations.

## Topics to Explore

- [file] `reasons/repair.py` — Inspect the full candidate-offset loop around lines 146–154 and 206–233 to see how failed `raw_decode` attempts are handled.
- [function] `reasons/llm.py:_parse_cli_json` — Contrasts `json.loads` on CLI output with the `raw_decode` extraction strategy used for LLM response text.
- [file] `reasons/propose_update.py` — Shows the “try `raw_decode` at each `[` position” pattern for extracting arrays from prose-like responses.
- [file] `tests/test_import_json.py` — Demonstrates JSON round-tripping with `json.dumps`/`import_json`, where a single complete document is expected.
- [general] `llm-response-json-extraction` — Compare extraction and fallback strategies across `repair`, `review`, `propose_update`, and related modules.

## Beliefs

- `raw-decode-ignores-trailing-data` — `JSONDecoder.raw_decode` returns after the first complete JSON value and does not reject trailing text, unlike `json.loads`.
- `raw-decode-returns-end-index` — `raw_decode` returns `(value, end_index)`, but the observed callers discard the end index with `_`.
- `json-loads-enforces-single-document` — `json.loads` is used in `reasons/llm.py:129` and many `reasons/pg.py` lines where the input is expected to be exactly one JSON document.
- `llm-response-parsers-scan-candidate-offsets` — Repair, review, and propose modules instantiate `JSONDecoder` and call `raw_decode` at candidate `[` or `{` offsets to tolerate prose around JSON.
- `multiple-values-require-looping-end-index` — Parsing multiple concatenated JSON values with `raw_decode` requires reusing the returned end index; the observed code does not do this.
