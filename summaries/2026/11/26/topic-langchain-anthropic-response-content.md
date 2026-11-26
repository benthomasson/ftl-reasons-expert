---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Investigate when LangChain `.content` is a plain string versus structured content blocks.

# When LangChain `.content` is a String vs. Structured Blocks

## TL;DR

The codebase **assumes** LangChain `.content` is always a plain `str`, but neither the code nor the tests verify this. The single LangChain call site (`reasons/llm.py:246`) returns `.content` without normalization, and every test mock sets `.content` to a string. There is no handling for structured (list-of-blocks) content anywhere in the LangChain path — which means the code will silently return a `list` if the provider ever returns one.

## Where `.content` is used

There are three distinct `.content` accessors in the codebase, and only one is LangChain:

| Location | Source | Type of `.content` |
|---|---|---|
| `reasons/llm.py:246` (`_invoke_api`) | **LangChain** `BaseMessage` | `str \| list[dict]` — **unchecked** |
| `reasons/llm.py:266` (`_invoke_openai`) | OpenAI Responses SDK (`item.content`) | `list[ResponseOutputContent]` — iterated with `hasattr(part, "text")` |
| `reasons/mcp_client.py:109` | MCP protocol `CallToolResult.content` | MCP content blocks |

Only the first is relevant to LangChain. The OpenAI path at line 266 is **not** LangChain — it's the OpenAI Python SDK's Responses API, and it already handles the structured case manually.

## What determines the shape

LangChain's `AIMessage.content` is typed `str | list[str | dict]` in modern versions (≥0.3, which `pyproject.toml` pins: `langchain-anthropic>=0.3`). The actual runtime shape depends on the provider and on what the model was asked to produce:

**Plain `str` — the common case**
- Simple text-in / text-out completion with no tools, no thinking, no citations.
- This is what `tests/test_llm.py:184, 200, 220, 322` mock, and what `_invoke_api`'s `-> str` return type promises.

**List of content blocks — the case the code doesn't handle**
- **Extended thinking** (Claude 3.7+ / Sonnet 4): the response contains `{"type": "thinking", ...}` blocks plus a `{"type": "text", "text": ...}` block.
- **Tool / function calls**: mixed `tool_use` and `text` blocks.
- **Citations** enabled on Anthropic.
- **Multimodal** prompt/response flows.
- **Streaming merges** or multi-block responses, where `langchain-anthropic` wraps accumulated chunks in a list even if the text content is trivial.

The `langchain-anthropic` 0.3+ adapters generally return `str` for a single text-only response and a `list[dict]` when thinking/tools/multimodal/citations produce multiple blocks — but the exact boundary is version- and model-dependent, which is why LangChain later added `AIMessage.content_blocks` as the normalized accessor.

## The actual risk in this code

`_invoke_api` (`reasons/llm.py:200-246`):

```python
def _invoke_api(prompt: str, model: str, timeout: int = 300) -> str:
    ...
    response = llm.invoke(prompt, config=config)
    ...
    return response.content       # line 246
```

- **Return annotation says `-> str`** but there is no `isinstance(response.content, str)` check and no `.content_blocks` normalization.
- If `ChatAnthropic` or `ChatAnthropicVertex` ever returns a `list` — e.g. because the model was invoked with thinking enabled, a citation-bearing response, a tool call, or a newer LangChain version that wraps text — this function returns the raw list up the stack. Callers then try to treat it as a string and fail in confusing ways downstream.
- There is **no test coverage** for the list case; `search_langchain_message_types` returned zero hits for `AIMessage` / `BaseMessage`, and `search_structured_content_terms` found no `content_blocks` handling anywhere in the repo.

Compare with `_invoke_openai` (`reasons/llm.py:249-269`), which **does** handle structured content:

```python
for item in response.output:
    if item.type == "message":
        for part in item.content:
            if hasattr(part, "text"):
                text += part.text
```

So the codebase already knows that one provider returns structured content blocks and normalizes it — but the LangChain path is not given the same treatment.

## What the observations can't tell us

- Whether the live providers actually return structured content here (no runtime capture).
- The exact installed `langchain-anthropic` / `langchain-google-vertexai` versions at run time (only the lower-bound pin is in `pyproject.toml`).
- Whether any caller enables tools, thinking, or citations on the `ChatAnthropic` / `ChatAnthropicVertex` instances. From `reasons/llm.py:212-230` they are constructed bare (`ChatAnthropic(model=name, timeout=...)` and `ChatAnthropicVertex(model_name=name, project=..., location=..., request_timeout=...)`), which reduces the chance of block-shaped responses today but does **not** guarantee it.
- Whether any downstream caller checks for a `list` return.

Results so far indicate a defensive gap, not an active bug — but it's the kind of gap that only surfaces when the model or version changes.

## Topics to Explore

- [file] `reasons/llm.py` — The `_invoke_api` return path should be normalized; worth refactoring with a `_normalize_content()` helper that handles `str`, `list[str]`, and `list[dict]` with `text` keys.
- [function] `reasons/llm.py:_invoke_api` — Add a parametrized test that mocks `.content` as a list of content blocks to lock down the contract.
- [function] `reasons/llm.py:_invoke_openai` — Existing reference implementation for manually extracting text from content blocks; the normalization logic it uses is what `_invoke_api` is missing.
- [general] `langchain-anthropic-content-shape` — Check the installed `langchain-anthropic` version and whether `AIMessage.content` returns `str` or `list[dict]` for the specific models used (Sonnet 4, etc.) by capturing a live response.
- [file] `pyproject.toml` — The pins `langchain-anthropic>=0.3` and `langchain-google-vertexai>=2.0` are open-ended; a version bump could change the `.content` shape without any code change here, which is the exact failure mode this investigation is about.

## Beliefs

- `_invoke_api-returns-content-unchecked` — `reasons/llm.py:246` returns `response.content` with no `isinstance` or block-normalization check despite the `-> str` annotation.
- `langchain-anthropic-is-the-only-langchain-provider` — `reasons/llm.py:21` and `pyproject.toml` show `ChatAnthropic` and `ChatAnthropicVertex` as the only LangChain message producers in the repo; the OpenAI path at `reasons/llm.py:249` uses the OpenAI SDK directly.
- `openai-path-handles-structured-content` — `_invoke_openai` at `reasons/llm.py:264-268` already iterates `item.content` blocks, unlike the LangChain path.
- `tests-only-exercise-string-content` — All four mock assignments (`tests/test_llm.py:184, 200, 220, 322`) set `.content` to a string; there is no test for `list`-shaped content.
- `no-content-blocks-references-exist` — The search for `content_blocks` / `structured content` / `content block` returned no hits in production code, so structured content normalization is unimplemented for LangChain.
- `api-providers-constructed-bare` — `reasons/llm.py:212` and `reasons/llm.py:226-230` build `ChatAnthropic` / `ChatAnthropicVertex` without tools, thinking, or citations enabled, which currently biases responses toward plain strings but does not constrain provider behavior.
