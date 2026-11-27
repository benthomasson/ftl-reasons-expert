---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Missing fence stripping, balanced-brace scanning, and re-prompt-on-failure are the three highest-value additions.

## Why these three additions are the high-value gap

The phrase "Missing fence stripping, balanced-brace scanning, and re-prompt-on-failure are the three highest-value additions" is a prescription about where the current code leaks reliability: at the boundary between *model text* and *structured data*. Everything upstream of that boundary is already handled in `reasons/llm.py` — cost tracking, CLI/API dispatch, Langfuse tracing — but every path through that file returns the model's response as **raw text** with no normalization.

### 1. Every return path is raw text

`reasons/llm.py` has four places that hand generated text back to callers, and none of them touch the contents:

- `_parse_cli_json` (lines 122–153) parses only the CLI envelope `{"result": ..., "usage": ...}` and returns `data["result"]` verbatim at **line 147** (or `data["response"]` for Gemini at **line 136**).
- `_invoke_api` returns `response.content` verbatim at **line 245**.
- `_invoke_openai` concatenates `part.text` fragments and returns them at **line 270**.
- `invoke_model` returns the ollama stdout unmodified at **line 318**, and otherwise delegates to `_parse_cli_json` at **line 319**.

So if a prompt asks Claude for JSON and Claude answers with:

> Here is the result:
> ```json
> {"entries": [...]}
> ```

…that is exactly what `invoke_model` returns. Any downstream `json.loads(...)` call fails. `_parse_cli_json` even has a fallback at **line 130** that returns the raw stdout when the *envelope* fails to parse — which widens the leak rather than narrowing it.

### 2. The searches confirm the three helpers do not exist

Both lookups in the observations come back empty:

- `search_fence_terms_in_python` (`fence|balanced.?brace|re-?prompt|reprompt`) → **0 matches**
- `search_fence_terms_in_json` (`...|highest-value`) → **0 matches**

So there is no `_strip_fences`, no `_extract_json_object`, and no retry-on-parse-failure anywhere in the Python tree or the tracked JSON. The functionality described in the claim is, as stated, *missing*.

### 3. What each addition concretely buys you

| Addition | Problem it fixes | Natural insertion point |
|---|---|---|
| **Fence stripping** | Models wrap JSON in ```` ```json ... ``` ```` even when told not to. Downstream `json.loads` rejects it. | After the return value is extracted in `_parse_cli_json` (line 147/136), `_invoke_api` (line 245), and `_invoke_openai` (line 270) — ideally factored into a shared `_normalize_model_text`. |
| **Balanced-brace scanning** | Models prepend/append prose ("Here is the JSON:"). Naive `text.find('{')…text.rfind('}')` breaks on nested objects or braces inside strings. A depth counter that respects JSON string/escape state extracts the first complete top-level `{…}`. | Same normalization helper, applied only when the text isn't already valid JSON. |
| **Re-prompt-on-failure** | When extraction still fails, today the caller gets `None`/an exception. Instead, invoke the model again with a corrective prompt ("Your last response was not valid JSON; return only the JSON object, no prose, no fences") before giving up. | A thin wrapper around `invoke_model` (line 274) so all four backends benefit. |

The ordering matters: fence stripping is cheap and covers the majority of cases; balanced-brace scanning picks up prose-wrapped output; re-prompt handles the residue. Each layer only fires when the previous one left unparseable text, so the added latency/cost is proportional to how broken the model's output was.

### 4. The symptom is already visible in the pipeline report

`.forge/report.json` shows the `diary-update` pipeline running two steps:

- `propose-beliefs`: **`proposed: 260 from 17 entries`**
- `accept-beliefs`: **`accepted 0 beliefs`**, `total_beliefs: 201`

A 260 → 0 conversion is a near-total failure at the boundary between "model produced candidates" and "candidates were accepted." The most common cause of exactly that pattern is the accept step calling `json.loads` on text the propose step produced with fences or prose around it. `.forge/proposed-entries.json` (17 entry paths, 20 lines) is the proposed-entry manifest from the same run — it recorded the entries, but no acceptance occurred.

### 5. Caveat — what the observations do *not* show

The propose/accept implementation is **not in the observation set**. We only see `reasons/llm.py`, `reasons_lib/llm.py` (a 3-line shim re-exporting `reasons.llm`), the report, and the proposed-entries manifest. So we can infer the failure mode from the report's 260→0 shape and from `invoke_model`'s raw-text contract, but we cannot point to the exact line where the doomed `json.loads` call lives. Before writing the fix, locate the `propose-beliefs` / `accept-beliefs` step implementations and confirm they call `invoke_model` and then parse the return value directly.

## Topics to Explore

- [function] `reasons/llm.py:_parse_cli_json` — Only unwraps the CLI envelope; the `return text` on line 147 is the first place a normalization helper should be called.
- [function] `reasons/llm.py:invoke_model` — Single dispatch point for all four backends (lines 288–319); a re-prompt wrapper here reaches every caller for free.
- [file] `.forge/report.json` — Contains the `proposed: 260`, `accepted: 0` evidence that motivates the change; also useful as the before/after test fixture.
- [file] `.forge/proposed-entries.json` — The 17-entry manifest produced by the same run, confirming propose-side output survived even though accept-side did not.
- [general] `propose-beliefs-pipeline-stage` — The call site *not included in these observations* where model text is parsed; locate it to confirm the failure point before implementing fence stripping.

## Beliefs

- `invoke-model-returns-raw-model-text` — `invoke_model` returns the model's response unmodified across all four backends (lines 245, 270, 318, 319); it performs no fence stripping or JSON extraction.
- `parse-cli-json-unwraps-envelope-only` — `_parse_cli_json` parses only the CLI's outer `{result, usage}` object and returns `result` verbatim at line 147, so any fences or prose emitted by the model pass through unchanged.
- `parse-cli-json-falls-back-to-raw-output` — On `json.JSONDecodeError`, `_parse_cli_json` returns the unmodified stdout at line 130, bypassing even envelope parsing.
- `no-fence-or-brace-extraction-helper-exists` — Searches for `fence`, `balanced.?brace`, and `re-?prompt` across Python and JSON return zero matches; none of the three additions is implemented.
- `diary-update-pipeline-produces-zero-acceptance` — `.forge/report.json` records 260 proposed and 0 accepted beliefs in the same run, a symptom consistent with downstream JSON parsing failing on raw model output.
