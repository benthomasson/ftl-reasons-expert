---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Compare the parsers named in the search results to see whether `raw_decode`-at-each-`[` is the common production extraction pattern.

## Summary

The search results support a **qualified yes**: `raw_decode`-at-each-`[` is a recurring production extraction pattern for LLM response parsing in the `reasons/` modules, but it is **not universal** across all parsers in the codebase.

Key evidence:

- `raw_decode` appears in **6 files**: `reasons/propose_update.py`, `reasons/repair.py`, `reasons/repair_premises.py`, `reasons/review.py`, `reasons/review_justifications.py`, and `reasons/review_premises.py`.
- There are **8 observed `raw_decode` code calls**, each paired nearby with a `json.JSONDecoder()` instantiation.
- Two files explicitly document the “at each `[`” behavior:
  - `reasons/propose_update.py:167-175` in `parse_update_proposals`
  - `reasons/review.py:183-191`
- Other production response parsers use regex/`finditer` instead, and some “parsers” are whole-JSON or structured-input parsers, not free-text extraction.

## Parser comparison

| Parser / location | Observed pattern | Notes |
|---|---|---|
| `reasons/propose_update.py:parse_update_proposals` line 164 | `raw_decode` at each `[` | Docstring line 167 explicitly says “Uses `json.JSONDecoder.raw_decode` at each `[` position”; decoder at line 170; call at line 175. |
| `reasons/review.py` around lines 183-191 | `raw_decode` at each `[` | Docstring line 183 says “Tries `json.JSONDecoder.raw_decode` at each `[` position”; decoder at line 186; call at line 191. |
| `reasons/repair.py:parse_match_response` line 143 | `raw_decode` to find JSON object | Docstring line 146 says “Uses `raw_decode` to find JSON object in response text”; decoder line 149; call line 154. |
| `reasons/repair.py:parse_triage_response` line 204 | `raw_decode` | Decoder line 206; call line 211. |
| `reasons/repair.py:parse_soften_response` line 226 | `raw_decode` | Decoder line 228; call line 233. |
| `reasons/repair.py:parse_extract_response` line 130 | Not observed using `raw_decode` | Included in parser definitions, but no `raw_decode` match near it. |
| `reasons/repair_premises.py` line 88 | `raw_decode` | Decoder at line 83; call at line 88. Function name not returned in the observed parser-definition subset. |
| `reasons/review_justifications.py` line 74 | `raw_decode` | Decoder at line 69; call at line 74. Function name not returned in the observed subset. |
| `reasons/review_premises.py` line 93 | `raw_decode` | Decoder at line 88; call at line 93. Function name not returned in the observed subset. |
| `reasons/contradictions.py:parse_contradiction_response` line 70 | Regex `finditer` | Uses `_NOGOOD_PATTERN.finditer(response)` at line 81; not `raw_decode`. |
| `reasons/derive.py:parse_proposals` line 377 | Regex `finditer` | Uses `new_pattern.finditer(response)` at line 395 and `old_pattern.finditer(response)` at line 420; not `raw_decode`. |
| `reasons/api.py` line 4024 | Regex bracket scanning | `re.finditer(r"\[.*?\]", response, re.DOTALL)`. No `raw_decode` matches appear in `api.py`. |
| `reasons/pg.py` line 1144 | Regex bracket scanning | Same `re.finditer(r"\[.*?\]", response, re.DOTALL)` pattern. No `raw_decode` matches appear in `pg.py`. |
| `reasons/api.py:parse_contradiction_plan` line 4867 | Unknown from returned snippet | Definition is listed, but body is not included. No `raw_decode` match in `api.py`, so it likely does not use this pattern, but the exact body is missing. |
| `reasons/api.py:parse_dedup_plan` line 5107 | Unknown from returned snippet | Same caveat as above. |
| `reasons/llm.py:_parse_cli_json` line 123 | Whole-output `json.loads` | Parses CLI wrapper output with `json.loads(output)`; not free-text `raw_decode` extraction. |
| `reasons/import_agent.py:_normalize_json` line 131 / `_normalize_nogoods_json` line 178 | Structured-data normalization | Operates on already-parsed JSON dicts, not LLM response text extraction. |
| `reasons/import_beliefs.py:parse_repos` / `parse_beliefs` / `parse_nogoods` lines 48/68/135 | Structured markdown/text parsing | Not the same problem as extracting JSON from arbitrary LLM output. |

## Conclusion

`raw_decode`-at-each-`[` is a **common production extraction pattern for LLM responses** in this codebase, especially in the `propose_update`, `review*`, and `repair*` modules. The repeated pairing of `json.JSONDecoder()` and `raw_decode`, plus explicit docstrings in `propose_update.py` and `review.py`, makes this a real production idiom rather than an isolated helper.

However, it is **not the only production extraction pattern**. Other response parsers use regex `finditer`:

- `reasons/contradictions.py:81`
- `reasons/derive.py:395` and `:420`
- `reasons/api.py:4024`
- `reasons/pg.py:1144`

So the most accurate statement is: **`raw_decode`-at-each-`[` is a common response-extraction pattern, but production parsing is mixed, with regex-based extraction and whole-JSON parsing also present.**

What is missing from the observations:

- The full loop bodies around `raw_decode` in `repair.py`, `repair_premises.py`, `review.py`, `review_justifications.py`, and `review_premises.py`, which would confirm whether every candidate is literally `[`.
- The bodies of `reasons/api.py:parse_contradiction_plan` and `parse_dedup_plan`.
- The full set of parser definitions: `json_parser_definitions` reports `total_count: 133` but only 30 were returned, so there may be additional parsers not compared here.

## Topics to Explore

- [file] `reasons/propose_update.py` — Inspect `parse_update_proposals` around lines 164-175; it is the clearest observed `raw_decode`-at-each-`[` example.
- [file] `reasons/repair.py` — Compare `parse_extract_response` (line 130), `parse_match_response` (143), `parse_triage_response` (204), and `parse_soften_response` (226) to see which repair parsers use `raw_decode`.
- [file] `reasons/review.py` — Read lines 183-191 to confirm the loop and candidate scanning behind the docstring “Tries ... at each `[` position”.
- [function] `reasons/contradictions.py:parse_contradiction_response` — Contrast its `_NOGOOD_PATTERN.finditer` extraction at line 81 with `raw_decode`.
- [file] `reasons/api.py` — Inspect regex bracket scanning at line 4024 and the definitions of `parse_contradiction_plan`/`parse_dedup_plan` at 4867/5107 to identify a separate parser family.

## Beliefs

- `raw-decode-used-in-six-response-parser-files` — `raw_decode` appears in `reasons/propose_update.py`, `reasons/repair.py`, `reasons/repair_premises.py`, `reasons/review.py`, `reasons/review_justifications.py`, and `reasons/review_premises.py`.
- `raw-decode-calls-paired-with-jsondecoder` — Every observed `raw_decode` code call has a nearby `json.JSONDecoder()` instantiation in the same parser cluster.
- `propose-update-explicitly-scans-each-open-bracket` — `reasons/propose_update.py:167-175` documents and implements `raw_decode` at each `[` position.
- `review-explicitly-scans-each-open-bracket` — `reasons/review.py:183-191` documents and implements trying `raw_decode` at each `[` position.
- `regex-finditer-parsers-coexist-with-raw-decode` — `reasons/contradictions.py:81`, `reasons/derive.py:395`, `reasons/derive.py:420`, `reasons/api.py:4024`, and `reasons/pg.py:1144` use regex `finditer` rather than `raw_decode`.
