---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: parse_match_response in reasons/repair.py

# `parse_match_response` — Explanation

## Purpose

`parse_match_response` is a **defensive JSON extractor** for the output of the matching LLM call (`MATCH_PROMPT` → `invoke_model`). LLMs frequently wrap their answers in prose ("Here is the JSON: …", trailing notes, etc.), so the function scans the raw response text for the **first parseable JSON object** and returns a normalized `{"matched_ids": [...], "rationale": ...}` dict.

Its second job is a **safety filter**: it drops any `matched_ids` that are not in `valid_ids`, so a hallucinated premise ID can never leak into the repair pipeline (where it would otherwise be passed into `api.add_justification` as a real antecedent).

It is the parsing half of the `find_matching_premises` flow used by `_do_search_and_link` and `repair_smuggled_beliefs`.

## Contract

**Preconditions (assumed, mostly unenforced):**
- `response` is a string (or at least an iterable of single characters).
- `valid_ids` supports `in` (the caller passes a `set` of ID strings).
- The LLM's intended answer is a JSON **object** (dict) somewhere in the text.
- Elements of `obj["matched_ids"]` are hashable strings (implied, not validated — see Error Handling).

**Postconditions:**
- Always returns a `dict` with both keys `"matched_ids"` and `"rationale"`.
- `result["matched_ids"]` is a **list** and a **subset of `valid_ids`**, preserving the LLM's original order (duplicates preserved).
- `result["rationale"]` is whatever `obj.get("rationale", "")` produced — **not type-checked** (could be `None`, a number, a dict, etc., if the LLM misbehaves). Default is `""`.
- On total parse failure, returns `{"matched_ids": [], "rationale": ""}`.

**Invariant that *is* enforced:** every ID returned in `matched_ids` exists in `valid_ids`.

## Parameters

| Param | Type | Meaning / edge cases |
|---|---|---|
| `response` | `str` | Raw LLM text. Empty string, whitespace, or text without `{` → returns the empty default. Non-str (e.g. `None`) raises `TypeError` on iteration. |
| `valid_ids` | container supporting `in` (expected `set[str]`) | IDs of candidate premises. Because the caller uses a `set`, membership requires hashable elements — non-string, unhashable IDs in the response will crash (see below). Also accepts a list, but then filtering is O(n·m). |

## Return Value

A dict `{"matched_ids": list, "rationale": <from LLM or "">}`.

Callers must handle **empty `matched_ids`** — this is the signal for "no match" (used by `_do_search_and_link` to return status `"no_match"` and by `repair_smuggled_beliefs` similarly). Callers also effectively assume `rationale` is a string when embedding it in `api.set_metadata(...)` or `api.retract_node(reason=...)`; the function does not guarantee that.

## Algorithm

1. Build a `json.JSONDecoder`.
2. Walk the response character-by-character with its index `i`.
3. Skip any character that is not `{`.
4. At each `{`, call `decoder.raw_decode(response, i)` — this parses a JSON *value starting exactly at position `i`* and returns `(obj, end_index)`. The `end_index` is discarded, so anything after the object is ignored.
5. If `JSONDecodeError` is raised (the `{` isn't the start of valid JSON — e.g., it's inside prose or a malformed fragment), continue scanning for the next `{`.
6. If `obj` is not a `dict`, continue scanning (defensive; a value starting with `{` that parses successfully is virtually always a dict).
7. Read `matched = obj.get("matched_ids", [])`; if it is **not a list**, reset it to `[]` (so a bare string `"id-1"` is dropped rather than coerced).
8. `filtered = [mid for mid in matched if mid in valid_ids]` — order preserved, hallucinated IDs removed.
9. Return `{"matched_ids": filtered, "rationale": obj.get("rationale", "")}` immediately (first successful parse wins).
10. If the loop exhausts without success, return the empty default.

Notable behavior: **first-parseable-object wins.** If the LLM echoes a JSON example before its real answer, the example is what gets returned. In practice `MATCH_PROMPT` instructs "Respond with ONLY a JSON object…", minimizing this risk, but it is an assumption not enforced.

## Side Effects

None. Pure function: no I/O, no mutation of arguments, no global state. (It can raise `TypeError` — see below — but that is a return-path property, not a side effect.)

## Error Handling

- **Swallowed:** `json.JSONDecodeError` (skips to next `{`), non-`dict` parse results (skipped), non-list `matched_ids` (reset to `[]`).
- **Not swallowed — potential `TypeError`:** Because the caller passes `valid_ids` as a `set`, the expression `mid in valid_ids` raises `TypeError: unhashable type` if any element of `matched_ids` is itself a list or dict (both legal in JSON arrays, e.g. `{"matched_ids": [[1,2]]}`). This propagates out of `parse_match_response` and up through `find_matching_premises` → the `try/except Exception` around each belief in `repair_beliefs`/`repair_smuggled_beliefs` catches it there (status `"error"`).
- **Not swallowed — `TypeError` on bad `response`:** if `response` is not iterable of chars.
- **No validation of `rationale` type** — a non-string rationale flows through unchanged.

## Usage Patterns

- Single caller: `find_matching_premises(smuggled_claim, candidates, ...)`, which computes `valid_ids = {c["id"] for c in candidates}` and returns the parsed dict directly.
- Consumers downstream: `_do_search_and_link` and `repair_smuggled_beliefs` check `matched_ids` truthiness to decide `"no_match"` vs `"linked"`/`"repaired"`, and feed the IDs into `api.add_justification`.
- **Caller obligation:** an empty `matched_ids` list means "fail/no-op", not an error — callers must branch on it. Callers should also treat `rationale` as potentially non-string when interpolating into API calls.

## Dependencies

- `json` (stdlib) — specifically `json.JSONDecoder.raw_decode`.
- No imports from the rest of the codebase; this function is self-contained within `reasons/repair.py`. It is exercised indirectly via `find_matching_premises` in `tests/test_repair.py` and `tests/test_repair_premises.py`.

### Assumptions the type system does not enforce
- `response` is a `str`; passing `None`/bytes fails at iteration.
- `valid_ids` is a set of **hashable** IDs; unhashable `matched_ids` entries crash the filter.
- The first valid JSON object in the text is the model's *answer*, not an echo/example.
- `rationale` is a string; it may be `null`, number, or object.
- `matched_ids` entries are strings, not arbitrary JSON values.

## Topics to Explore

- [file] `reasons/repair.py` — Full repair orchestration; see how `parse_match_response` feeds `_do_search_and_link` and `repair_smuggled_beliefs`.
- [function] `reasons/repair.py:find_matching_premises` — The sole caller; shows how `valid_ids` is constructed from candidates.
- [function] `reasons/repair.py:parse_triage_response` — Sibling parser using the same `raw_decode` scanning idiom, but with pattern validation instead of ID filtering.
- [general] `llm-json-extraction-pattern` — The `for i, ch … raw_decode` scan used across all parsers in this module; consider hardening against unhashable IDs and echoed examples.
- [file] `tests/test_repair_premises.py` — Edge cases covered for parsing/filtering of matched premise IDs.

## Beliefs

- `parse-match-response-filters-to-valid-ids` — Returned `matched_ids` is always a subset of `valid_ids`; hallucinated IDs are dropped.
- `parse-match-response-returns-first-parseable-object` — It returns on the first `{`-anchored JSON dict that parses, ignoring all trailing text and earlier non-JSON fragments.
- `parse-match-response-requires-hashable-ids` — Because callers pass a `set`, unhashable entries in `matched_ids` raise `TypeError` rather than being filtered out.
- `parse-match-response-defaults-empty-result` — Any fully unparseable response yields `{"matched_ids": [], "rationale": ""}`, which callers treat as "no match".
- `parse-match-response-resets-non-list-matched-ids` — A non-list `matched_ids` (e.g., a single string) is discarded to `[]` rather than coerced.
- `parse-match-response-does-not-validate-rationale` — `rationale` is passed through verbatim from the JSON object, including non-string values.
