---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: extract_tool_call in reasons/ask.py

# `extract_tool_call`

## Purpose

`extract_tool_call` parses a raw LLM response for a tool request. It exists because the prompt asks the model to respond with a single JSON line when it wants a tool:

```json
{"tool": "search_beliefs", "query": "..."}
```

The function returns that parsed JSON object if found, otherwise `None`. In practice, it is the decision point for the tool loop in `ask`: `None` means “the model answered directly,” while a returned dict means “the model wants a tool executed.”

It is intentionally simple and permissive: it scans lines, tries to JSON-decode lines that begin with `{`, and accepts the first object containing a top-level `"tool"` key.

## Contract

- **Precondition:** `text` is expected to be a `str` containing the raw LLM response.
- **Postcondition:** returns the first parsed JSON object from a stripped line that:
  1. starts with `{`,
  2. successfully decodes as JSON,
  3. contains a top-level `"tool"` key.
- Otherwise returns `None`.
- **Invariants:**
  - Does not mutate `text`.
  - Does not search for embedded JSON substrings; the JSON must occupy a whole stripped line.
  - Does not parse multi-line / pretty-printed JSON.
  - Does not validate the schema of the tool call.
  - Does not validate the tool name, argument names, or argument types.
  - If multiple matching JSON lines exist, the first one encountered wins.

Important implicit assumptions:

- The model obeys the “one JSON line, no other text” prompt convention.
- The JSON line is not prefixed by markdown bullets, prose, or code-fence markers.
- The JSON object is not pretty-printed across multiple lines.
- A final natural-language answer containing a standalone line like `{"tool": ...}` would be misclassified as a tool request.

## Parameters

### `text`

- Expected type: `str`.
- Meaning: raw text returned by the LLM.
- Edge cases:
  - Empty string returns `None`.
  - Whitespace-only string returns `None`.
  - `None` is not supported; calling `.split()` on it raises `AttributeError`.
  - `bytes` is not supported; line checks and JSON parsing assume text.

The type system does not enforce that `text` is a string, nor that it comes from an LLM response.

## Return Value

Returns either:

- a `dict` parsed from a JSON object, or
- `None`.

The returned dict is not guaranteed to be a valid tool call. It only guarantees that a top-level `"tool"` key exists. For example, these would all be returned:

```json
{"tool": null}
{"tool": "", "query": 123}
{"tool": "anything", "unexpected": true}
```

The caller is responsible for interpreting `"tool"` and validating any additional fields. In `ask`, the caller reads `tool_call.get("tool")`, then branches on known values such as `"search_beliefs"` or MCP tool names.

## Algorithm

Step-by-step:

1. Split the response text on newline characters:
   ```python
   text.split("\n")
   ```
2. For each line:
   - Strip leading and trailing whitespace.
   - Skip the line if it does not start with `{`.
3. Try to parse the stripped line as JSON:
   ```python
   obj = json.loads(line)
   ```
4. If parsing succeeds and `"tool" in obj`, return `obj`.
5. If parsing raises `json.JSONDecodeError`, skip that line and continue.
6. If no line produces a matching object, return `None`.

Notable consequences of the algorithm:

- Leading indentation before `{` is okay because of `.strip()`.
- Trailing whitespace after the JSON object is okay.
- Code-fence lines such as ` ```json ` are ignored, but the actual JSON line inside the fence can still be parsed if it starts with `{`.
- A line like `Here is the call: {"tool": ...}` is ignored because it does not start with `{`.
- A line with multiple JSON objects is invalid and is skipped.
- Multi-line JSON is not detected because each line is parsed independently.

## Side Effects

None.

The function is pure from the caller’s perspective:

- no file I/O,
- no network calls,
- no logging,
- no mutation of input,
- no global state changes.

It does allocate parsed JSON objects and may do CPU work proportional to response size.

## Error Handling

- `json.JSONDecodeError` is caught and ignored. Malformed JSON lines do not raise out of the function.
- The function returns `None` when no valid matching JSON line is found.
- It does not log or report skipped malformed lines.
- It does not catch non-JSON errors such as:
  - `text` being `None` → `AttributeError`,
  - `text` being bytes → likely `TypeError` during string/bytes operations,
  - unusual JSON parsing errors outside `JSONDecodeError`.

Because the function does not validate schema, it cannot produce a structured “invalid tool call” error. A dict with `"tool"` but missing required arguments is returned as-is; the caller may then behave unpredictably or use defaults.

## Usage Patterns

In `reasons/ask.py`, `extract_tool_call` is called inside the `ask` tool loop:

```python
tool_call = extract_tool_call(response)

if tool_call is None:
    return response.strip()
```

So:

- `None` means the LLM produced its final answer, and `ask` returns the stripped response.
- A returned dict means the model requested a tool.

The caller then does:

```python
tool_name = tool_call.get("tool")
```

and branches:

- `"search_beliefs"`: reads `query`, calls `api.search(...)`, appends tool history, updates belief context.
- Known MCP tool: passes all keys except `"tool"` as arguments to `bridge.call_tool(...)`.
- Unknown tool: returns the original response stripped, effectively treating it as a final answer.

Caller obligations:

- Pass raw model response text.
- Treat `None` as “no tool call.”
- Validate the returned dict before acting on it, since `extract_tool_call` does not.
- Be aware that an unrelated final-answer line containing `{"tool": ...}` will be interpreted as a tool request.

## Dependencies

- Python standard library: `json`.
- No internal module dependencies inside the function itself.
- In context, it is used by `ask`, which also depends on `invoke_model`, `api.search`, and MCP bridge tool execution.

## Topics to Explore

- [file] `reasons/ask.py` — Contains the prompts and tool-loop orchestration that consume `extract_tool_call`.
- [function] `reasons/ask.py:ask` — Shows exactly how extracted tool calls drive belief search, MCP execution, and final synthesis.
- [file] `tests/test_ask.py` — Likely covers parsing edge cases and expected tool-loop behavior.
- [file] `tests/test_ask_mcp.py` — Shows how extracted tool names are routed to MCP bridge tools.
- [general] `llm-tool-call-parsing` — Explore assumptions around JSON-line formatting, prompt compliance, and schema validation gaps.

## Beliefs

- `extract-tool-call-returns-none-without-valid-tool-json` — It returns `None` if no stripped line begins with `{` and parses as a JSON object containing a top-level `"tool"` key.
- `extract-tool-call-requires-single-line-json` — It scans individual stripped lines, so pretty-printed or multi-line JSON tool calls are not detected.
- `extract-tool-call-swallows-jsondecodeerror` — Malformed JSON lines are skipped rather than raising out of the function.
- `extract-tool-call-does-not-validate-schema` — It checks only for the presence of a `"tool"` key and does not validate the tool name, argument shape, or value types.
- `extract-tool-call-enables-ask-tool-loop` — A non-`None` result in `ask` causes the loop to execute a requested tool or treat unknown tool names as final responses.
