---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Robustness patterns for extracting JSON from free-form model output.

# Robustness Patterns for Extracting JSON from Free-Form Model Output

## Scope caveat

The symbol-search observations (`locate_extract_json_symbol`, `locate_parse_json_symbol`) both returned **0 definitions**, and `search_json_loads_in_reasons` / `search_fenced_or_extraction_patterns` both returned **0 matches**. That is misleading: the direct file read of `reasons/llm.py` shows `json.loads` and `json.JSONDecodeError` inside `_parse_cli_json`, and `reasons/ask.py` imports `json` and calls `json.loads` in `extract_tool_call`. So the search was likely scoped to a path that excludes these modules, or the pattern/quoting misfired. **Trust the file reads over the searches.** There is no helper literally named `extract_json` or `parse_json`; the real code is `extract_tool_call` (ask.py) and `_parse_cli_json` (llm.py). Exact line numbers weren't included in the returned slices, so anchors below are by function name.

## Two distinct problems being solved

### 1. Parsing a JSON *envelope* emitted by a CLI (`reasons/llm.py:_parse_cli_json`)

This is **not** free-form model text — it is the CLI's own `--output-format json` wrapper. The pattern is "parse or fall back, never raise":

```python
try:
    data = json.loads(output)
except (json.JSONDecodeError, ValueError):
    return output

if not isinstance(data, dict):
    return output
```

Robustness properties:

- **Never raises on malformed output.** Both `JSONDecodeError` and `ValueError` are caught (the former is a subclass of the latter, so this is defensive).
- **Shape check before key access.** `isinstance(data, dict)` rejects top-level arrays/scalars before any `.get()` call. `tests/test_llm.py::TestParseCliJson::test_json_array_returns_raw` locks this in.
- **Graceful degradation to raw string.** On any failure the original output is returned, so the caller still gets *something* rather than an exception.
- **Per-provider key extraction with defaults.** Gemini uses `data.get("response", output)`, Claude uses `data.get("result", output)`. Note the fallback default is the **entire raw envelope string**, not a sentinel — `test_json_missing_result_returns_raw` asserts this. That's a deliberate "do not lose data" choice, but it means downstream code can silently receive a JSON blob where a human answer was expected.
- **Cost accounting is coupled to successful parsing.** `_record_cost` is only reached inside the dict branch. `test_invalid_json_returns_raw` and `test_json_array_returns_raw` both assert `get_cost_summary()["calls"] == 0` — i.e. a parse failure also silently drops token/cost telemetry.

### 2. Extracting a *tool call* from free-form model text (`reasons/ask.py:extract_tool_call`)

This is the true free-form extraction path. The heuristic is "line-scan for the first line that parses as a JSON object with a `tool` key":

```python
for line in text.split("\n"):
    line = line.strip()
    if not line.startswith("{"):
        continue
    try:
        obj = json.loads(line)
        if "tool" in obj:
            return obj
    except json.JSONDecodeError:
        continue
return None
```

Robustness properties:

- **Prefilters on `{}` before attempting parse.** Prose lines, headings, bullet points, and closing code fences are skipped without paying the cost of an exception.
- **Per-line isolation.** One bad line cannot poison the whole response; parsing continues to the next line.
- **Returns `None` on total failure** rather than raising, letting the caller decide (fall back to treating the text as a final answer).
- **First-match wins.** Multiple tool calls in one response are silently truncated to the first.

### 3. Prompt-level discipline is the primary defense

Both parsers rely on the model cooperating. `reasons/ask.py::ASK_PROMPT` instructs:

> "If you need more information, respond with ONLY a single JSON line (no other text)."

and supplies canonical examples via `_DEFAULT_TOOLS_SECTION` (`{"tool": "search_beliefs", "query": "search terms"}`) and `_build_tools_section`, which renders MCP tools as one-line JSON examples. This is the cheapest mitigation: constrain the shape at generation time so extraction rarely needs to be clever.

### 4. Output normalization before extraction (`reasons/llm.py:invoke_model`)

The Ollama branch strips chain-of-thought prefixes before returning:

```python
# Fragile: ollama thinking markers may change across versions
if model.startswith("ollama:") and "Thinking...\n" in output:
    parts = output.split("...done thinking.\n", 1)
    if len(parts) == 2:
        output = parts[1]
```

The code comment itself flags the fragility. Tests `test_ollama_strips_thinking_output`, `test_ollama_no_thinking_markers_unchanged`, and `test_ollama_incomplete_thinking_unchanged` cover the three branches, and `test_claude_does_not_strip_thinking` confirms the normalization is Ollama-only. This is a robustness pattern with a known expiry date: marker strings will drift across model versions.

## Gaps worth naming

1. **No Markdown-fence handling anywhere.** The `search_fenced_or_extraction_patterns` search for ` ``` ` returned **0 matches** across the codebase. A response like ```` ```json\n{"tool": ...}\n``` ```` only works by accident: the fence lines are skipped by the `startswith("{")` filter, but any indentation or multi-line pretty-printed JSON inside the fence will fail every line individually.
2. **No multi-line / balanced-brace scanner.** `extract_tool_call` parses one line at a time; pretty-printed JSON is unparseable per-line.
3. **No `isinstance(obj, dict)` guard in `extract_tool_call`.** `"tool" in obj` is only safe for dicts:
   - `json.loads("42")` → `int` → `"tool" in 42` raises **`TypeError`**, which is *not* caught (`JSONDecodeError` only).
   - `json.loads('"tool"')` → `str` → `"tool" in "tool"` is `True` → the function returns a **string** despite its docstring promising a dict or `None`.
   Contrast this with `_parse_cli_json`, which *does* have the `isinstance(..., dict)` guard. The two extraction paths are not consistently hardened.
4. **No retry / re-prompt on extraction failure.** If `extract_tool_call` returns `None`, nothing in the visible slice re-asks the model with a stricter instruction. (The tool-loop portion of `ask.py` beyond line 400 wasn't returned, so this is unverified for the loop itself.)
5. **Leading-text JSON is missed.** `Sure! {"tool": "search_beliefs"}` fails because the stripped line does not start with `{`. Only line-leading JSON is recognized.

## Why this matters operationally

The design philosophy across both paths is **"parse best-effort, degrade to raw text, never raise."** That is the right default for LLM output, but it produces a specific failure mode: a malformed response does not crash the pipeline — it quietly turns into an answer that is actually a JSON envelope or a prose preamble. `tests/test_llm.py::TestParseCliJson` encodes this contract explicitly (`test_json_missing_result_returns_raw`, `test_invalid_json_returns_raw`), so any change here needs a corresponding test update.

## Topics to Explore

- [function] `reasons/ask.py:extract_tool_call` — The only hand-rolled free-form JSON extractor; add the missing `isinstance(obj, dict)` guard and decide on multi-line/fenced support.
- [function] `reasons/llm.py:_parse_cli_json` — The reference "parse-or-fallback" implementation; useful contrast for how shape checks and cost accounting interact.
- [file] `tests/test_llm.py` — `TestParseCliJson` is the executable spec for fallback behavior; extend it when hardening extraction.
- [file] `reasons/ask.py` — Read lines beyond the returned 400 to see how a `None` from `extract_tool_call` is handled in the tool loop (retry vs. treat as final answer).
- [general] `json-extraction-hardening` — Missing fence stripping, balanced-brace scanning, and re-prompt-on-failure are the three highest-value additions.

## Beliefs

- `extract-tool-call-requires-line-leading-brace` — `extract_tool_call` only attempts `json.loads` on lines whose first non-whitespace character is `{`, and returns the first parsed object containing a `"tool"` key.
- `extract-tool-call-swallows-json-errors` — `extract_tool_call` catches only `json.JSONDecodeError` and continues, so a line parsing to a non-iterable (e.g. `42`) raises an uncaught `TypeError`.
- `parse-cli-json-never-raises` — `_parse_cli_json` catches `JSONDecodeError`/`ValueError` and non-dict payloads, returning the raw output instead of propagating an exception.
- `parse-cli-json-fallback-skips-cost-accounting` — When `_parse_cli_json` falls back to raw output, `_record_cost` is never called, so `get_cost_summary()["calls"]` remains 0.
- `ollama-thinking-stripping-is-provider-scoped` — Only models with the `ollama:` prefix strip `Thinking...`/`...done thinking.` markers; Claude and Gemini outputs pass through unnormalized.
