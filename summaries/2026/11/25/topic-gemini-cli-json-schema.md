---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The `stats.models.<name>.tokens.input` / `.candidates` shape is defined by Gemini's CLI, not this repo; verify stability across versions.

## What this claim means

The claim is about `reasons/llm.py`, specifically `_parse_cli_json`. When the repo invokes Gemini, it does so via the CLI subprocess and then parses the JSON envelope that the CLI emits. The token counters `stats.models.<name>.tokens.input` and `stats.models.<name>.tokens.candidates` live inside that envelope — they are part of Gemini CLI's output contract, not a schema the repo defines, documents, or generates.

### Where the Gemini CLI is invoked

- `reasons/llm.py:51` — `MODEL_COMMANDS["gemini"] = ["gemini", "--skip-trust", "-o", "json", "-p", ""]`
- `reasons/llm.py:186-188` — `resolve_model_cmd("gemini:<model>")` builds `["gemini", "--skip-trust", "-m", <submodel>, "-o", "json", "-p", ""]`

The `-o json` flag is what causes the CLI to produce the structured envelope (with a `response` field, a `stats` field, etc.). The repo neither writes nor validates this envelope.

### Where the shape is consumed

`_parse_cli_json` in `reasons/llm.py` (roughly lines 124–148), for the Gemini branch:

- `reasons/llm.py:136` — `if model.startswith("gemini") or model.startswith("gemini:"):`
- `reasons/llm.py:141` — `for model_stats in stats.get("models", {}).values():`
- `reasons/llm.py:142` — `tokens = model_stats.get("tokens", {})`
- `reasons/llm.py:143` — `input_tokens += tokens.get("input", 0)`
- `reasons/llm.py:144` — `output_tokens += tokens.get("candidates", 0)`
- Then `_record_cost(model, input_tokens, output_tokens, 0.0)`

Concretely the assumed path is:

```
data["stats"]["models"][<model-name>]["tokens"]["input"]      -> input tokens
data["stats"]["models"][<model-name>]["tokens"]["candidates"] -> output tokens
```

The name `candidates` is Gemini's terminology for generated (candidate) tokens; the repo just sums it into `output_tokens`.

### How resilient this actually is

Every hop uses `.get(..., default)`: `stats.get("models", {})`, `model_stats.get("tokens", {})`, `tokens.get("input", 0)`, `tokens.get("candidates", 0)`. Combined with the outer `data.get("stats", {})`, a shape change (renamed key, moved stats block, dropped token counters) will **not raise**. It will silently produce `(0, 0)` for that call, so the cost summary at `reasons/llm.py:format_cost_summary` would under-report rather than error. That is a real verification hazard when relying on an external contract.

### What the repo actually pins

Nothing, with respect to Gemini CLI versions:

- `MODEL_COMMANDS` and `resolve_model_cmd` invoke the `gemini` binary from `PATH` without a version constraint.
- `find_cli_version_or_invocation` shows no `gemini --version` call anywhere in the repo. The only `--version` handler is `reasons/cli.py:3057`, and that reports **this package's** version (`ftl-reasons`), not Gemini CLI's.
- No CI/assertion pipeline compares the fixture shape against a real invocation of a known Gemini CLI release.

### What tests do and do not prove

The only repo artifacts that "encode" the shape are fixture-based:

- `tests/test_llm.py:371-384` (`test_gemini_json`) — feeds a fixture with `"gemini-2.5-flash": {"tokens": {"input": 80, "candidates": 40}}` (`tests/test_llm.py:377`) and asserts only `text == "Gemini says hi."`.
- `tests/test_llm.py:390-396` (`test_gemini_submodel_json`) — uses `"stats": {"models": {"flash": {"tokens": {"input": 10, "candidates": 5}}}}` at `tests/test_llm.py:393`.

Note the gap: these tests assert the parsed **response text**, not the extracted token totals. The token bookkeeping itself is tested separately via `_record_cost("gemini", 80, 40, 0.0)` at `tests/test_llm.py:447` and `:466`. So even within the tests, `_parse_cli_json`'s extraction of `input`/`candidates` is **not** asserted end-to-end. If Gemini renames those keys, the fixtures keep passing and only the running totals regress.

### What's missing to actually verify stability

The observations cannot confirm stability across CLI versions because:

1. The Gemini CLI's JSON schema/changelog is not in this repo — the observations only show the consumer side.
2. There is no `gemini --version` probe in the codebase to gate behavior on known-good versions.
3. There is no integration test that runs the real CLI and round-trips its JSON through `_parse_cli_json`.

So the correct reading of the claim is: **the schema is external and the repo is a best-effort consumer with silent fallbacks; verifying stability requires external evidence (CLI release notes/schema) and currently-unwritten integration checks.**

## Topics to Explore

- [function] `reasons/llm.py:_parse_cli_json` — Review the `.get()` fallbacks for silent schema drift; consider logging when `stats`/`tokens` are missing.
- [file] `tests/test_llm.py` — Add an end-to-end test asserting the token totals extracted from a Gemini JSON fixture (not just the response text).
- [general] `gemini-cli-contract` — Investigate Gemini CLI's released JSON schema and changelog across versions to confirm `stats.models.<name>.tokens.{input,candidates}` is still emitted.
- [general] `version-pinning` — Decide whether to probe `gemini --version` at invocation and warn/fail on unknown versions.
- [file] `reasons/cli.py` — The `--version` flag at line 3057 reports the `ftl-reasons` package version, not the model CLI versions, so it is not a substitute for Gemini CLI version checks.

## Beliefs

- `gemini-token-stats-shape-is-external` — The path `stats.models.<name>.tokens.input` / `.candidates` is produced by Gemini CLI output, and the repo only consumes it in `_parse_cli_json`.
- `parse-cli-json-has-silent-token-fallback` — Missing `stats`, `models`, `tokens`, `input`, or `candidates` keys all default to `0` / `{}` instead of raising.
- `repo-does-not-pin-gemini-cli-version` — No code path calls `gemini --version` or otherwise constrains the Gemini CLI version used.
- `gemini-candidates-maps-to-output-tokens` — `tokens["candidates"]` is summed into the tracker's `output_tokens` at `reasons/llm.py:144`.
- `gemini-json-tests-do-not-assert-token-totals` — `test_gemini_json` and `test_gemini_submodel_json` assert only the response text, leaving token extraction from `stats.models` untested end-to-end.
