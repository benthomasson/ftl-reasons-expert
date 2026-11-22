---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Evaluate whether provider-native tool calls could be normalized behind `invoke_model` while preserving multi-backend support.

# Evaluating Provider-Native Tool Calls Behind `invoke_model`

## 1. What `invoke_model` is today

`reasons/llm.py:275` defines the single entry point:

```python
def invoke_model(prompt: str, model: str = "claude", timeout: int = 300) -> str:
```

Key properties visible in the observations:

- **String-only contract.** The function returns `str`. `tests/test_llm.py` asserts this throughout, and callers such as `reasons/ask.py:328` and `reasons/ask.py:336` immediately call `.strip()` on the return value.
- **Dispatch is model-prefix based**, inside the body of `invoke_model`:
  - `api:` / `vertex:` → `_invoke_api` (ChatAnthropic / ChatAnthropicVertex via LangChain)
  - `openai:` → `_invoke_openai` (OpenAI Responses API)
  - Everything else → `resolve_model_cmd` → `subprocess.run` with prompt on stdin
- **Cost tracking is a side channel** (`_record_cost`, `_parse_cli_json`, `_invoke_openai`). Structured data (tokens, cost) already flows out of provider responses but is deliberately flattened into a mutable global tracker, not surfaced on the return value.
- **Sedimentation.** 124 usages of `invoke_model` across `reasons/api.py`, `ask.py`, `contradictions.py`, `repair.py`, `repair_premises.py`, `propose_update.py`, `build_wiki.py`, `pg.py`, `report_belief.py`, `cli.py`. This is a wide fan-out, which matters for backward compatibility.

## 2. Where provider-native tool calls could actually come from

### API backends — yes, the capability is already inside `_invoke_api`

`_invoke_api` (in `reasons/llm.py`) constructs `ChatAnthropic(model=name, timeout=...)` or `ChatAnthropicVertex(...)` and calls:

```python
response = llm.invoke(prompt, config=config)
return response.content
```

LangChain's `ChatAnthropic` and `ChatAnthropicVertex` already normalize tool calls into `AIMessage.tool_calls` when (a) the model is bound with `.bind_tools([...])` and (b) the model emits tool calls. Today those fields are discarded — `response.content` is returned, nothing else. So for `api:` and `vertex:` the "normalization" is essentially free at the response side; the request side is where `tools` needs to be plumbed in.

### OpenAI backend — partially, but currently filtered out

`_invoke_openai` iterates:

```python
for item in response.output:
    if item.type == "message":
        for part in item.content:
            if hasattr(part, "text"):
                text += part.text
```

The Responses API emits `item.type == "function_call"` entries alongside `message` entries. Those are dropped today because the loop filters to `"message"`. Adding native tool support here means: (a) pass `tools=[...]` to `client.responses.create(...)`, and (b) collect `function_call` items into the normalized shape.

### CLI backends (`claude`, `gemini`, `ollama`) — not without emulation

`resolve_model_cmd` (in `reasons/llm.py`) only knows how to build:

- `claude -p [--model X] --output-format json`
- `gemini --skip-trust [-m X] -o json -p ""`
- `ollama run X`

No `--tools` / `--allowed-tools` style flag appears anywhere in the codebase, and `_parse_cli_json` only extracts `result`/`response` strings. Whatever tool behavior the CLIs offer, the current code does not invoke it. So for CLI backends, "native tool calls" is not exposed by the current subprocess contract — any normalized shape would have to be populated by the same text-parsing approach that exists in `ask.py` today.

## 3. What "normalization" already looks like in the codebase

The de facto normalization layer today is **prompt-based**, not provider-native:

- `reasons/ask.py:111` defines `extract_tool_call(text)`.
- `reasons/ask.py:514` calls it after `invoke_model`.
- `reasons/ask.py:519-536` reads `tool`, `query`, and generic args from the parsed dict.
- `tests/test_ask.py:42-73` and `tests/test_ask_mcp.py:123-125` test this parsing directly.

So a unified `ToolCall` shape already exists informally (`{"tool": ..., "query": ..., **args}`). Moving tool calls behind `invoke_model` is a serviceable idea — but it would be a **relocation** of the `extract_tool_call` layer, not a new capability, for CLI backends.

## 4. Key obstacles to normalizing behind `invoke_model`

1. **Signature/return-type change.** `-> str` is baked into 124 callers. Two of them (`ask.py:328`, `ask.py:336`) chain `.strip()` immediately; many others just concatenate or print the result. Silently switching to a `(text, tool_calls)` tuple or a dataclass breaks all of these. You need either:
   - a new function (`invoke_model_structured`, `invoke_model_with_tools`) or
   - an optional out-parameter / extra kwarg that defaults to off, i.e. an **additive** change.
2. **Request-side schema diverges per backend.** Anthropic takes `tools=[{"name", "description", "input_schema"}]`; OpenAI Responses takes `tools=[{"type": "function", "name", ...}]`; LangChain's `.bind_tools` accepts a provider-agnostic shape that it translates. There is no single wire format, so "normalize behind `invoke_model`" really means "normalize at the LangChain/OpenAI SDK boundary, plus emulation at the CLI boundary."
3. **No tool definitions exist in `llm.py` today.** The tools are described in prompts built in `ask.py` (visible in the `build_*_prompt` imports at `tests/test_ask.py:12`). Whichever new entry point is chosen has to receive tool schemas from the caller; nothing in `llm.py` currently knows what a "tool" is.
4. **Error semantics differ.** `invoke_model` already raises three distinct exception types (`FileNotFoundError`, `ImportError`, `RuntimeError`, `subprocess.TimeoutExpired`). Structured tool-call failures (malformed args, unknown tool name) would need to be folded into this taxonomy consistently across backends.
5. **Cost accounting must stay intact.** `_record_cost` is invoked from both `_parse_cli_json` and `_invoke_openai`. Any refactor of those needs to keep token/cost accumulation behavior tested by `TestParseCliJson`.

## 5. Multi-backend preservation: possible, with a tiered model

The clean design that preserves all backends is:

```text
invoke_model / invoke_model_structured
        │
        ├── api: / vertex:   → LangChain bind_tools → AIMessage.tool_calls   (native)
        ├── openai:          → Responses API tools= → function_call items     (native)
        └── claude/gemini/ollama (CLI)
                             → prompt-injected tools → parse text via
                               extract_tool_call-like logic                    (emulated)
```

The normalized result — e.g. `ModelResult(text: str, tool_calls: list[ToolCall], raw: Any)` — has to tolerate "no native tool calls produced" and "native tool calls produced." Push that decision down to each backend without leaking backend types upward. This is exactly what `.bind_tools` already does for chat-style APIs, so the API paths are the easy part; the CLI paths simply return `tool_calls=[]` unless the caller explicitly opts into prompt-based emulation.

## 6. Verdict

**Feasible, but only additively and per-backend.** Normalizing provider-native tool calls behind `invoke_model` is technically sound for `api:`, `vertex:`, and `openai:` models — those SDKs already expose structured tool calls and the current code is just discarding them. For `claude`/`gemini`/`ollama` CLI backends, no native interface exists in the current command construction, so tool calls would remain paraphrased into prompts and recovered by string parsing (what `extract_tool_call` does today).

Because `invoke_model` returns `str` and has 124 direct call sites, the least disruptive path is a new sibling function (e.g. `invoke_model_structured`) that returns a dataclass, keeping `invoke_model` a thin wrapper for existing callers. Do **not** change `invoke_model`'s return type in place.

What the observations **do not** show, and would change the conclusion:
- The body of `ask.py` lines around 453-561, i.e. how tools are described to the model in the prompt and how `extract_tool_call` output is consumed.
- Any `--tools`/`--mcp` flags the `claude` and `gemini` CLIs might accept (the search pattern `tools=|tool_calls` only hits Python files, not CLI invocations).
- Whether `openai:` uses the same tool schema as `ask.py` or an unrelated internal shape.

## Topics to Explore

- [file] `reasons/ask.py` — Where prompt-based tool calling is currently implemented end to end; the closest analogue to the proposed normalization layer.
- [function] `reasons/ask.py:extract_tool_call` — The existing de facto normalizer; its token/arg shape (`{"tool", "query", ...}`) is what "normalized tool calls" already mean in this codebase.
- [function] `reasons/llm.py:_invoke_api` — The natural integration point for LangChain `.bind_tools()` and `AIMessage.tool_calls`; currently returns `response.content` only.
- [function] `reasons/llm.py:_invoke_openai` — Needs to grow `function_call` item handling in the Responses API loop to surface native tool calls for `openai:`.
- [general] `cli-tool-support` — Investigate whether `claude` and `gemini` CLIs expose `--tools`/`--mcp` flags; the observations show none, which determines whether CLI backends can ever be native or must remain emulated.

## Beliefs

- `invoke-model-returns-only-str` — `invoke_model` in `reasons/llm.py:275` returns `str`, and callers like `reasons/ask.py:328`/`ask.py:336` chain `.strip()` on that return, so a non-`str` return would break call sites.
- `api-backends-discard-tool-calls` — `_invoke_api` calls `llm.invoke(prompt, config=...)` and returns only `response.content`, so any LangChain `AIMessage.tool_calls` are dropped.
- `openai-backend-filters-non-text-items` — `_invoke_openai` only appends text from `item.type == "message"` items, so Responses-API `function_call` items never reach the caller.
- `tool-calls-are-text-parsed-today` — The current tool-calling path is prompt-based: `ask.py` builds a prompt that describes tools and recovers calls via `extract_tool_call` (`reasons/ask.py:111`), not via provider APIs.
- `cli-backends-lack-native-tool-surface` — `resolve_model_cmd` and the CLI branch of `invoke_model` pass only a prompt on stdin and parse stdout, with no CLI flag in the codebase that selects native tools.
- `openai-models-bypass-resolve-model-cmd` — `resolve_model_cmd` raises `ValueError` for `openai:` models (and `api:`/`vertex:` are short-circuited before reaching it), so the OpenAI path is dispatched separately in `invoke_model`.
