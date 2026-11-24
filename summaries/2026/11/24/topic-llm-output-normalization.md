---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How other call sites cope with chatty/nonconforming LLM output (compare with `parse_review_response`'s `raw_decode` strategy).

## Explanation

The dominant pattern in the observed LLM-response parsers is a **copy-pasted `raw_decode` scanner**: instead of calling `json.loads(response)` on the whole string, each parser walks through candidate JSON start positions and lets `json.JSONDecoder.raw_decode` decode from that position. That tolerates preamble, Markdown fences, and trailing commentary because parsing starts at the first `[` or `{` that yields a valid JSON value, and the returned end index is ignored.

### Parsers using the same `raw_decode` strategy as `parse_review_response`

`parse_review_response` is the reference example:

- `reasons/review.py:180` defines `parse_review_response`.
- `reasons/review.py:183` documents that it tries `json.JSONDecoder.raw_decode` at each `[` position.
- `reasons/review.py:191` calls `items, _ = decoder.raw_decode(response, i)`.

The same array-oriented scanner appears in several other call sites:

| File / function | Observed tactic |
|---|---|
| `reasons/propose_update.py:parse_update_proposals` line 164 | Scans `[` positions; line 167 documents `raw_decode`; line 175 decodes `items, _ = decoder.raw_decode(response, i)` |
| `reasons/repair_premises.py:parse_repair_response` line 81 | Array extraction via `raw_decode` at line 88 |
| `reasons/review_justifications.py:parse_justification_review` line 67 | Array extraction via `raw_decode` at line 74 |
| `reasons/review_premises.py:parse_premise_review_response` line 86 | Array extraction via `raw_decode` at line 93 |

The object-oriented analogue appears in `reasons/repair.py`:

- `parse_match_response` at line 143; line 146 says it uses `raw_decode` to find a JSON object, and line 154 calls `raw_decode`.
- `parse_triage_response` at line 204; line 211 calls `raw_decode`.
- `parse_soften_response` at line 226; line 233 calls `raw_decode`.

So most observed LLM-output parsers cope with chatty output by **locating a JSON array/object inside the response**, not by sanitizing the whole response first. This naturally handles text before/after the JSON and often handles fenced code blocks, because the backticks are simply skipped before the first `[` or `{`.

### Stricter or pre-extracted call sites

Other observed call sites use plain `json.loads`, but usually in contexts that may already be constrained:

- `reasons/llm.py:_parse_cli_json` line 123 ends with `data = json.loads(output)` at line 129. Based on the visible line alone, this is a strict whole-output parse; any preamble or fences would need to be removed before line 129, but those surrounding lines are not in the observations.
- `reasons/api.py:4026` uses `ids = json.loads(match.group())`, which suggests a regex-first extraction strategy before strict JSON parsing.
- `reasons/ask.py:122` uses `obj = json.loads(line)`, which fits line-delimited JSON parsing rather than parsing a free-form LLM response.
- Many `reasons/pg.py` `json.loads(...)` calls are database metadata/row decoding, not necessarily LLM-output parsing.

### What the `raw_decode` pattern does and does not solve

It solves:

- prose before the JSON,
- Markdown code fences around the JSON,
- trailing explanations after the JSON,
- multiple `[`/`{` characters that are not valid JSON starts, by trying each candidate.

It does **not** necessarily solve:

- malformed JSON syntax inside the payload, such as trailing commas or single quotes,
- choosing the intended JSON value when multiple valid JSON fragments are present,
- schema validation after decoding.

### Insufficient observations

The observations list `parse_contradiction_response` (`reasons/contradictions.py:70`), `parse_proposals` (`reasons/derive.py:377`), and `parse_verify_response` (`reasons/verify.py:90`), but do not show whether they use `raw_decode`, `json.loads`, regex extraction, or another helper. The actual error handling, fallback behavior, and schema validation for all parsers are also not visible. The search for LLM-output tests matching terms like `raw_decode`, `nonconforming`, `chatty`, `fenced`, and `preamble` returned no matches, so test coverage for these edge cases cannot be confirmed from the observations alone.

## Topics to Explore

- [function] `reasons/review.py:parse_review_response` — Read the full function to confirm the candidate loop, exception handling, and return value when no array is found.
- [function] `reasons/propose_update.py:parse_update_proposals` — Compare its array-scanning behavior and output schema with `parse_review_response`.
- [function] `reasons/repair.py:parse_match_response` — Inspect the object-oriented `raw_decode` variant and how `valid_ids` validation interacts with extraction.
- [function] `reasons/llm.py:_parse_cli_json` — Determine whether the direct `json.loads(output)` at line 129 has earlier stripping/fallback logic not shown in the observations.
- [file] `tests/test_review.py` — Inspect tests around line 159 for fenced, preamble, malformed, and empty-response coverage.

## Beliefs

- `review-parser-scans-array-brackets` — `parse_review_response` uses `raw_decode` at `[` positions to extract a JSON array from surrounding text (`reasons/review.py:183,191`).
- `multiple-review-parsers-duplicate-array-raw-decode` — The same `[`-scanning `raw_decode` pattern appears in `propose_update`, `repair_premises`, `review_justifications`, and `review_premises` parsers.
- `repair-object-parsers-scan-object-braces` — `parse_match_response`, `parse_triage_response`, and `parse_soften_response` use `raw_decode` for JSON objects in `reasons/repair.py`.
- `llm-cli-parser-uses-strict-json-loads` — `_parse_cli_json` calls `json.loads(output)` directly at `reasons/llm.py:129`, so it depends on pure JSON unless unobserved preprocessing exists.
- `api-regex-extraction-precedes-json-loads` — `reasons/api.py:4026` parses a regex match with `json.loads`, showing an extraction-then-strict-parse alternative to `raw_decode`.
