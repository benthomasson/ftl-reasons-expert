---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: parse_justification_review in reasons/review_justifications.py

## `parse_justification_review`

### Purpose

This is a **best-effort, defense-in-depth parser** for LLM output. The LLM is instructed to reply with only a JSON array of review objects, but in practice it will often: wrap the JSON in prose ("Here is my analysis: ..."), include markdown fences (` ```json `), add trailing commentary, or occasionally return malformed/misshapen JSON. This function is designed to *find* the first usable JSON array embedded anywhere in the response and normalize its shape, rather than assuming the whole response is valid JSON.

It exists as a boundary layer between non-deterministic LLM output and the deterministic downstream consumer (`review_justifications`, `_process_batch`). It is the only place in this module responsible for shaping LLM data into the schema used by the caller.

### Contract

- **Precondition:** `response` is an iterable of single characters (in practice, a `str`). It need not be valid JSON; it need not even contain a JSON array.
- **Postcondition:** Returns a `list[dict]`. Every returned dict is guaranteed to have keys `id`, `classification`, `required_antecedents`, `independent_antecedents`, `comment` with the following shape guarantees:
  - `id`: whatever value was in `item["id"]` (no type enforcement — could be a non-string).
  - `classification`: an uppercased `str` (defaults to `"ALL"` if the field is not a string).
  - `required_antecedents` / `independent_antecedents`: `list` (defaults to `[]` if the field is not a list).
  - `comment`: `str` (defaults to `""`).
- **Invariants:** No exception escapes for malformed JSON. The function is pure (no I/O, no state mutation).

### Parameters

- **`response`** — the raw string returned by `invoke_model`. Expected to contain at least one JSON array literal somewhere. If it is not a `str`, `enumerate` will still iterate, but non-str iterables of chars won't match JSON grammar and `raw_decode` will raise `TypeError` (not caught — see Error Handling).

### Return Value

- A list of normalized review dicts, or `[]` if:
  - no `[` character exists in `response`;
  - every `[`-starting position fails `raw_decode`;
  - every successfully decoded array is empty, or contains only non-dict entries / dicts missing `"id"`.
- **Critical caller obligation:** `[]` is a sentinel for *both* "LLM returned an empty array" and "I could not extract anything". The caller (`_process_batch`) explicitly treats `[]` as a parse failure and logs a warning. There is no way to distinguish a legitimate "no items to report" from a parse failure at this layer.

### Algorithm

1. Instantiate a reusable `json.JSONDecoder()`.
2. Scan `response` character by character; skip everything that isn't `[`.
3. At each `[` (index `i`), call `decoder.raw_decode(response, i)`. `raw_decode` attempts to parse exactly one JSON value starting at `i` and returns `(value, end_index)`. The end index is discarded.
4. If decoding raises `json.JSONDecodeError`, continue scanning for the next `[`.
5. If the decoded value isn't a `list`, continue.
6. Otherwise build a fresh `results` list, iterating each element of the decoded list:
   - Skip anything that isn't a `dict` or that lacks `"id"`.
   - Extract `classification`, `required_antecedents`, `independent_antecedents`, `comment` with per-field normalization/fallback.
   - Append the normalized dict.
7. If `results` is non-empty, return it immediately.
8. If `results` is empty (empty array, or all entries were filtered out), continue scanning for the next `[`.
9. If the loop ends, return `[]`.

The "first non-empty valid array wins" behavior means the function will not merge multiple arrays and will not fail if a spurious `[` appears earlier in prose (as long as that spurious position doesn't itself decode to a non-empty list of dicts with `id`).

### Side Effects

None. Pure function; no I/O, no mutation of `response` or any module state. (I/O happens in the caller, `_process_batch`, via `invoke_model`.)

### Error Handling

- `json.JSONDecodeError` — caught per candidate `[` position; the scan continues.
- **Not caught:** `TypeError` if `response` is not iterator-of-characters in a way `json` accepts (e.g., not a `str`, or char `ch` isn't a string). `AttributeError` if `item.get("comment")` returns something whose `__or__` misbehaves for exotic types (rare). `cls.upper()` is guarded by `isinstance(cls, str)`, so no `AttributeError` from that branch.
- **Silently swallowed:** any semantic issues, such as an invalid `classification` value (e.g., `"FOO"` is not rejected — it's uppercased and passed through), non-string `id`, or extra fields. Schema validation is **not** performed here.

### Usage Patterns

Usually called via `_process_batch`:

```python
response = invoke_model(prompt, ...)
results = parse_justification_review(response)
if not results and batch:
    print("WARN: batch returned no parseable results", ...)
```

Callers should:
- Treat empty return as parse failure (the batch was expected to be non-empty).
- Not rely on `classification` being one of `{ALL, ANY, MIXED}` — the code does not enforce the prompt's rules.
- Optionally dedupe / reconcile by `id` themselves; this function preserves incoming order but does not guarantee uniqueness.

### Dependencies

- `json` (stdlib) — `JSONDecoder.raw_decode`.
- Indirect via module: `reasons.llm.invoke_model`, `reasons.review.format_belief_for_review` — used by the caller, not this function.

### Assumptions not enforced by the type system

- `response` is a `str` containing JSON (ideal but not required beyond "iterable of chars").
- `classification` is one of `"ALL"`, `"ANY"`, `"MIXED"` — **not validated**; any string is uppercased and passed through.
- `id` is a non-empty string that exists in `nodes` — **not validated**; the caller does not cross-check against the requested batch.
- `required_antecedents` / `independent_antecedents` contain valid antecedent ids — **not validated**; only the container type is checked.
- The first usable JSON array in the text corresponds to the actual review payload (heuristic; a model that emits a contradictory example array before the real one would fool it).

### Complexity note

Worst case is O(n·k) where `n` is response length and `k` is the number of `[` characters that trigger `raw_decode`, since each is an independent parse attempt. In realistic LLM responses with 1–3 brackets this is fine; it would degrade on adversarially bracketed text.

---

## Topics to Explore

- [function] `reasons/llm.py:invoke_model` — Where the raw `response` string comes from; how timeouts and model dispatch work.
- [function] `reasons/review_justifications.py:review_justifications` — The orchestrator that batches, parallelizes, and consumes `parse_justification_review` output.
- [file] `tests/test_review_justifications.py` — Existing coverage of parser edge cases (prose-wrapped JSON, missing `id`, unknown classifications).
- [function] `reasons/review.py:format_belief_for_review` — The prompt-side formatting that the LLM keys off when emitting `id`s.
- [general] `json-raw-decode-semantics` — How `JSONDecoder.raw_decode` differs from `json.loads` regarding trailing data and multiple values.

## Beliefs

- `parse-justification-review-is-best-effort` — The function never raises on malformed JSON; it returns `[]` on failure and callers must interpret that as a parse failure.
- `parse-justification-review-requires-id-field` — Any decoded list entry that is not a dict or lacks an `"id"` key is silently dropped from the output.
- `parse-justification-review-defaults-classification-to-all` — A missing or non-string `classification` value is coerced to `"ALL"`.
- `parse-justification-review-does-not-validate-classification-values` — Arbitrary alphabetic strings (e.g., `"FOO"`) survive as uppercase `classification` values; only `{ALL, ANY, MIXED}` appear in the prompt, not in enforcement.
- `parse-justification-review-normalizes-antecedent-lists` — Non-list values for `required_antecedents` / `independent_antecedents` are replaced with `[]`, and `comment` becomes `""` on falsy input.
- `parse-justification-review-scans-for-first-bracket` — It iterates character-by-character to find the first `[` whose `raw_decode` yields a non-empty list of dicts-with-id, discarding the trailing position from `raw_decode`.
