---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Design rationale for text-based vs native function calling, and how `extract_tool_call` can fail silently on pretty-printed JSON.

## Short answer

The code treats model tool calls as **plain text JSON**: `reasons/ask.py:111` defines `extract_tool_call(text)`, and `reasons/ask.py:514` calls it on the raw string returned by the model. The extracted object is expected to contain a `"tool"` key (`reasons/ask.py:519`), with all remaining keys treated as MCP arguments (`reasons/ask.py:536`).

The likely design rationale is backend uniformity: the LLM layer supports Claude/Gemini/Ollama CLIs and `api:`/`vertex:` LangChain wrappers, all surfaced as plain strings to the caller. A text protocol lets `ask.py` use one parsing path instead of provider-specific native tool-call APIs. The tradeoff is fragile parsing: because the only observed JSON parse is `json.loads(line)` at `reasons/ask.py:122`, multi-line pretty-printed JSON can be missed entirely, and callers then treat the miss as “no tool call” (`reasons/ask.py:516`, `:561`) rather than as a malformed tool call.

Caveat: the full `extract_tool_call` body, prompt builders, and any design notes are not in the observations. There are **no** grep hits for “text-based”, “native function”, or “tool calling” rationale. So the rationale below is an inference from the architecture, not a documented decision visible in the provided data.

## Design rationale: text-based vs native function calling

### Evidence for text-based calling

- `reasons/ask.py:111` defines `extract_tool_call(text)`.
- `reasons/ask.py:514` calls `extract_tool_call(response)`, where `response` is a model response string.
- `reasons/ask.py:519` expects the parsed object to have `"tool"`:
  ```python
  tool_name = tool_call.get("tool")
  ```
- `reasons/ask.py:536` turns every other key into arguments:
  ```python
  args = {k: v for k, v in tool_call.items() if k != "tool"}
  ```
- `reasons/ask.py:561` checks `if extract_tool_call(response):` again, implying the function returns `None`/falsy when no tool call is found.

This is a custom textual protocol, not a provider-native tool-call object. Native APIs usually return structured fields like `tool_calls`, `name`, and `arguments`; this code expects a JSON object embedded in text with a `"tool"` key.

### Why this likely won

The repository supports multiple execution backends:

- `tests/test_llm.py` exercises `resolve_model_cmd` and `invoke_model` for Claude CLI, Gemini CLI, and Ollama CLI commands.
- The same test file exercises `api:` and `vertex:` LangChain wrappers.
- `reasons_lib/llm.py:1` is just a compatibility shim re-exporting from `reasons.llm`.

In that setup, native function calling would require per-provider handling: Claude CLI, Gemini CLI, Ollama, Anthropic API, and Vertex may all expose tool calls differently or not at all through the CLI path. A text-based protocol gives one common contract:

1. Prompt the model to emit JSON with a `"tool"` key.
2. Receive a string from any backend.
3. Parse that string in `extract_tool_call`.
4. Dispatch to MCP via `reasons/mcp_client.py:100 call_tool`.

The MCP client is separate: `call_tool` executes a tool call; it does not provide model-native function calling.

### Tradeoffs

Text-based calling is simpler to port across backends and easier to inspect/replay, but it gives up:

- schema enforcement,
- reliable provider-side argument validation,
- clear separation between “no tool requested” and “tool JSON malformed”,
- robust parsing of formatting variations.

Native function calling would likely improve reliability, but it would complicate the multi-backend abstraction unless the code first normalized provider-native tool calls into one internal representation.

## How `extract_tool_call` can fail silently on pretty-printed JSON

The key observed parsing operation is:

```python
obj = json.loads(line)
```

at `reasons/ask.py:122`.

That strongly suggests `extract_tool_call` scans the response **line by line**. A compact tool call works:

```json
{"tool": "search", "query": "foo"}
```

A pretty-printed tool call does not work if every line is parsed independently:

```json
{
  "tool": "search",
  "query": "foo"
}
```

Here, `json.loads("{")` fails, `json.loads('  "tool": "search",')` fails, and `json.loads("}")` fails. Unless the function separately buffers lines into a full JSON object — which is not observed — no line contains a complete JSON document.

The silent part is the return contract. Callers check:

- `reasons/ask.py:516`: `if tool_call is None:`
- `reasons/ask.py:561`: `if extract_tool_call(response):`

So a parse failure and “the model did not request a tool” collapse into the same falsy/`None` result. If `extract_tool_call` catches `json.JSONDecodeError` internally and continues, a malformed or multi-line tool call looks exactly like a normal answer with no tool request. The ask loop then proceeds as if the model wanted no tool call.

The observed tests do not catch this regression. `tests/test_ask.py:42-73` includes `test_valid_tool_call`, `test_no_tool_call`, `test_first_tool_call_wins`, and empty-string handling, but the observed inputs are compact/single-line. There is no visible pretty-printed multi-line regression test.

## Topics to Explore

- [file] `reasons/ask.py` — Inspect the full `extract_tool_call` body, prompt builders, and ask loop to confirm line-by-line parsing and exception handling.
- [function] `reasons/ask.py:extract_tool_call` — Verify exactly what happens on `json.JSONDecodeError` and whether it can ever buffer multi-line JSON.
- [file] `reasons/llm.py` — Understand the backend abstraction and why `invoke_model` returns plain strings across CLI and API providers.
- [general] `native-tool-calling-adoption` — Evaluate whether provider-native tool calls could be normalized behind `invoke_model` while preserving multi-backend support.
- [file] `tests/test_ask.py` — Add a regression test for pretty-printed multi-line tool-call JSON and malformed JSON handling.

## Beliefs

- `extract-tool-call-parses-json-per-line` — `extract_tool_call` calls `json.loads(line)` at `reasons/ask.py:122`, so it likely only recognizes JSON that occupies a single line.
- `ask-treats-none-as-no-tool-call` — The ask loop checks `tool_call is None` at `reasons/ask.py:516` and `if extract_tool_call(response)` at `:561`, so a parse miss is indistinguishable from an absent tool call.
- `text-tool-schema-has-tool-key` — The extracted object’s `"tool"` key selects the tool at `reasons/ask.py:519`, and all remaining keys become MCP arguments at `:536`.
- `ask-consumes-text-not-native-tool-calls` — `reasons/ask.py:514` passes the raw model string to `extract_tool_call`, not a provider-native tool-call structure.
- `llm-invocation-returns-strings-across-backends` — `tests/test_llm.py` verifies `invoke_model` for Claude/Gemini/Ollama CLIs and `api:`/`vertex:` LangChain wrappers, all producing plain text responses.
