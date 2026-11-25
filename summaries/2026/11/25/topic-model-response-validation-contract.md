---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Whether there is any test coverage asserting that malformed model output is rejected/coerced consistently across modules

## Short answer

No — not in the sense of a cross-module guarantee that malformed model output is consistently rejected or coerced.

There **is** narrow local test coverage for the CLI-JSON parsing helper `_parse_cli_json` in `reasons/llm.py`, exercised from `tests/test_llm.py`. But those tests assert **permissive fallback to raw output**, not rejection or schema validation. They also only cover the CLI transport/envelope JSON, not malformed semantic model output returned in fields like `result` or `response`.

Also, the summary grep results appear inconsistent with the read files: for example, `tests/test_llm.py` clearly contains `test_invalid_json_returns_raw` and the string `"not json at all"`, yet the relevant grep summaries report zero matches. I would trust the file contents over those empty match lists.

## What coverage exists

In `reasons/llm.py` (`0–327` lines returned), `_parse_cli_json`:

- catches `json.JSONDecodeError`/`ValueError` and returns the raw output instead of raising;
- returns raw output for non-dict JSON payloads;
- for Claude-style dict payloads, falls back to the raw output when `result` is missing, but still records a zero-token/zero-cost call.

In `tests/test_llm.py` (`0–450` of `470` lines returned), `TestParseCliJson` includes:

- `test_invalid_json_returns_raw` — asserts invalid JSON returns raw text and records no cost.
- `test_json_missing_result_returns_raw` — asserts a dict missing `result` returns raw JSON, but does **not** assert the cost-tracker side effect.
- `test_json_array_returns_raw` — asserts a non-dict JSON array returns raw text and records no cost.
- Several happy-path Claude/Gemini token/cost parsing tests.

These are useful local tests for `_parse_cli_json`, but they do **not** establish consistent malformed-output handling across modules. They test fallback behavior in one LLM invocation helper, not schema validation, normalization, or rejection by downstream consumers.

## What is missing

- No evidence of shared schema/validation/coercion tests across modules.
- No tests that malformed **model content** inside a valid CLI JSON envelope is rejected. `_parse_cli_json` only parses the CLI envelope; if `result` is arbitrary garbage, it is returned as-is.
- No tests for other modules consuming `invoke_model(...)` output and parsing/validating it.
- No test asserts consistency of behavior between CLI providers, API providers, and downstream structured-output consumers.
- The `test_json_missing_result_returns_raw` test leaves an important side effect unchecked: `_parse_cli_json` still records a zero-cost call for that malformed payload.

## Topics to Explore

- [file] `reasons/llm.py` — Inspect `_parse_cli_json` fallback paths, especially missing `result`/`response`, malformed `usage`, and malformed `stats`.
- [file] `tests/test_llm.py` — Review `TestParseCliJson` and the 20 unreturned lines; verify whether any additional malformed-output assertions exist.
- [function] `reasons/llm.py:_parse_cli_json` — Confirm whether missing `result` still records a zero-cost call and whether that behavior is intentional.
- [general] `downstream-llm-output-parsers` — Search for other modules that consume LLM output and parse it into schemas/JSON; the observations do not include those modules.
- [repo] `tests` — Re-run the invalid/malformed/coerce/schema searches because the provided zero-match summaries conflict with visible tests.

## Beliefs

- `malformed-cli-json-falls-back-to-raw` — `_parse_cli_json` catches `json.JSONDecodeError`/`ValueError` and returns the original output instead of raising.
- `non-dict-cli-json-falls-back-to-raw` — `_parse_cli_json` returns raw output for JSON arrays/scalars and does not record cost for non-dict payloads.
- `missing-result-still-records-zero-cost-call` — For a Claude dict missing `result`, `_parse_cli_json` returns raw JSON and still calls `_record_cost` with zero tokens/cost.
- `llm-tests-cover-parser-fallback-not-schema-validation` — `tests/test_llm.py` includes malformed CLI JSON fallback tests, but no schema-validation or normalization tests outside that helper.
- `cross-module-consistency-unverified` — The observations do not include downstream module tests or shared validation helpers, so consistent rejection/coercion across modules is not established.
