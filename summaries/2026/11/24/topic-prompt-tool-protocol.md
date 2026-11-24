---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How this text-based function-calling approach trades reliability for model-portability versus native tool APIs.

# How text-based function calling trades reliability for model-portability

## The design in one paragraph

Instead of passing a structured tool schema to a provider's native tool/function-calling API, `reasons/ask.py` embeds the tool description directly in the prompt (`ASK_PROMPT` / `_DEFAULT_TOOLS_SECTION`) and asks the model to reply with a single JSON line like `{"tool": "search_beliefs", "query": "search terms"}`. The caller then parses the response text itself with `extract_tool_call` (`reasons/ask.py:111`), inspects `tool_call.get("tool")` (`ask.py:519`), and builds an args dict with `{k: v for k, v in tool_call.items() if k != "tool"}` (`ask.py:536`). Results are re-injected into a follow-up prompt as text under `## Additional search results` and the model is asked again (`build_ask_prompt`, `ask.py:130`).

## Why this buys portability

`reasons/llm.py` supports a wide menu of backends through `MODEL_COMMANDS` and `resolve_model_cmd`: CLI subprocesses for `claude`, `gemini`, `claude:<model>`, `gemini:<model>`, `ollama:<model>`, plus LangChain adapters reached through `_invoke_api` for `api:<model>` and `vertex:<model>`. Every backend receives the *same* text prompt over stdin (CLI) or as a `HumanMessage` (LangChain). There is no per-provider tool-schema adapter:

- Claude CLI, Gemini CLI, `ollama run`, Vertex, and OpenAI-compatible `api:` endpoints all understand the same string prompt.
- MCP servers are wired in via `_build_tools_section(mcp_servers)` (`ask.py:350`) and `mcp_client.list_tools()` / `call_tool()` (`reasons/mcp_client.py:94`, `:100`), but the model still learns about them as *text* — the actual dispatch is a text JSON parse, not a native tool turn.
- Consequence: adding a new model provider does not require implementing tool-calling for it. The portability cost is paid once in the prompt template, not N times across providers.

## What it costs in reliability

1. **Fragile parsing surface.** `extract_tool_call` only considers lines where `line.strip().startswith("{")` and `json.loads(line)` succeeds. Multi-line JSON, markdown fences (` ```json `), leading prose, or JSON with a trailing comma all fail silently. `test_json_without_tool_key` (`tests/test_ask.py:52`) shows that valid JSON without a `"tool"` key is discarded — the model gets a plain-answer path instead of a call.

2. **"First tool call wins" ambiguity.** `test_first_tool_call_wins` (`tests/test_ask.py:62`) documents that if the model emits several JSON lines, only the first is executed. There is no parallel tool invocation, no round-trip per call — unlike native APIs where `tool_calls` is an ordered list the runtime can execute natively.

3. **Schema is advisory, not enforced.** `args = {k: v for k, v in tool_call.items() if k != "tool"}` (`ask.py:536`) accepts any keys the model emits. There is no JSON Schema, no type coercion, no required-field validation on the calling side beyond `tool_call.get("query", "")` (`ask.py:522`). Native tool APIs reject malformed arguments at the API boundary; here malformed arguments reach application code.

4. **Unknown-tool behavior is lenient.** `test_unknown_tool_returns_response` (`tests/test_ask.py:311`) suggests the model's original text is surfaced as the answer when the named tool is unrecognized — a graceful degradation, but also a silent one: the caller cannot distinguish "no tool needed" from "tool requested but unavailable" without inspecting logs.

5. **Extra synthesis loops.** `test_final_tool_call_triggers_extra_synthesis` (`tests/test_ask.py:275`) and `test_extra_synthesis_tool_call_returns_beliefs` (`tests/test_ask.py:338`) exist because the final prompt has no tool definition (`test_final_prompt_has_no_tool_definition`, `tests/test_ask.py:103`), yet the model may still emit one. That triggers an extra model invocation — extra latency and cost, driven purely by text-parsing heuristics.

6. **No native usage metadata for the tool turn.** `_parse_cli_json` (`reasons/llm.py:129`) parses CLI JSON only for *cost/token accounting* (`_record_cost`); it does not extract a `tool_use` block, because there isn't one. Tool call provenance lives in prompt history, not in a structured transcript.

7. **Defensive test suite confirms fragility.** Tests like `test_timeout_after_successful_tool_search_returns_beliefs` (`tests/test_ask.py:209`), `test_sources_survive_tool_call` (`tests/test_ask.py:372`), and `test_no_tool_history` (`tests/test_ask.py:84`) guard against text-mode failure modes that native APIs make structurally impossible: mismatched turn pairs, degraded partial results, provenance loss across turns.

## What native tool APIs would give you (and this code gives up)

- Provider-enforced argument schemas and typed errors at the API boundary.
- Ordered, multi-call `tool_calls` with parallel execution.
- Structural call/result turn token pairs, so the model can't be "confused" by a call/result mismatch.
- Streaming and stop-reason signals (`tool_use` stop reason) so the loop terminates deterministically rather than by re-parsing text.
- Rich usage attribution per call.

## How the code compensates

- Prompts explicitly warn the model ("respond with ONLY a single JSON line…") and separate phases: `build_ask_prompt` includes the tools section, `build_final_prompt` (`ask.py:156`) does not, and `build_simple_prompt` (`ask.py:472` region) has no tool definition at all — three distinct surface areas that tests cover individually.
- Redundant extraction: the loop calls `extract_tool_call(response)` again at `ask.py:561` for the final pass, and `_strip_belief_metadata` normalizes context regardless.
- Selective opt-out: `natural=True` swaps `_CITE_RULE` for `_CITE_RULE_NATURAL`, decoupling citation format from the tool loop.
- MCP is treated as an external tool but its descriptions are still rendered through `_build_tools_section`, so the same text protocol covers both built-in and external tools.

**Bottom line:** the code is deliberately provider-agnostic at the cost of moving tool-call semantics from the API layer into prompt engineering plus a line-based JSON parser (`ask.py:111`) with a "first match wins" rule. Reliability is recovered through prompt discipline, phase-specific templates, and defensive tests — not through typed contracts.

## Topics to Explore

- [file] `reasons/ask.py` — Full tool loop, including the retry/extra-synthesis path around line 561 and `_build_tools_section` at line 350
- [function] `reasons/ask.py:extract_tool_call` — The line-scanner that defines the entire parsing contract (startswith `{`, valid per-line JSON, requires `"tool"` key)
- [file] `tests/test_ask.py` — Failing-mode catalogue for text tool calls: `test_first_tool_call_wins`, `test_unknown_tool_returns_response`, `test_final_tool_call_triggers_extra_synthesis`
- [function] `reasons/llm.py:resolve_model_cmd` — Enumerates the provider matrix that motivates the text-based approach
- [general] `mcp-tool-integration` — How MCP `list_tools`/`call_tool` are serialized into prompt text rather than exposed as native tool schemas

## Beliefs

- `extract-tool-call-requires-line-starting-brace` — `extract_tool_call` only parses lines whose stripped form begins with `{`, so multi-line or fenced JSON tool calls are silently ignored.
- `text-tool-protocol-enables-provider-portability` — Because `MODEL_COMMANDS` covers CLI (`claude`, `gemini`, `ollama`) and LangChain providers (`api:`, `vertex:`), the text JSON protocol is the only path that works uniformly across all of them.
- `first-json-tool-line-wins` — If the model emits multiple tool-call JSON lines, only the first is executed, per `test_first_tool_call_wins`.
- `tool-args-are-unvalidated-passthroughs` — Args are built by excluding the `"tool"` key, and no schema validation or type checking occurs on the caller side (`ask.py:536`).
- `final-prompt-excludes-tool-definition` — `build_final_prompt` deliberately omits the tools section, so an emitted tool call in that phase triggers `test_final_tool_call_triggers_extra_synthesis` behavior.
