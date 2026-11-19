---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Inspect `pyproject.toml` for the `[api]`, `[vertex]`, `[openai]` extras that gate every `_HAS_*` flag in this file.

## What `pyproject.toml` declares

In `pyproject.toml`, the extras are keys under `[project.optional-dependencies]` at line 14 — they are not literal TOML sections like `[api]`. The relevant declarations are:

| Extra key | `pyproject.toml` line | Installs | Corresponding flag in `reasons/llm.py` |
|---|---:|---|---|
| `api` | 20 | `langchain-anthropic>=0.3` | `_HAS_LANGCHAIN_ANTHROPIC` |
| `vertex` | 21 | `langchain-google-vertexai>=2.0` | `_HAS_LANGCHAIN_VERTEX` |
| `langfuse` | 22 | `langfuse>=2.0` | `_HAS_LANGFUSE` |
| `openai` | 23 | `openai>=1.0` | `_HAS_OPENAI` |
| `llm` | 24 | `langchain-anthropic`, `langchain-google-vertexai`, `langfuse` | sets the first three flags, but **not** `_HAS_OPENAI` |

So there are four `_HAS_*` flags, not three. The `langfuse` extra is also part of the runtime gating story.

## How the flags are set

`reasons/llm.py` does not read `pyproject.toml` at runtime. Instead, it uses optional imports in the block around lines 20–46:

- `_HAS_LANGCHAIN_ANTHROPIC` is set at `reasons/llm.py:22` / `:25` after trying `from langchain_anthropic import ChatAnthropic`.
- `_HAS_LANGCHAIN_VERTEX` is set at `reasons/llm.py:29` / `:32` after trying `from langchain_google_vertexai.model_garden import ChatAnthropicVertex`.
- `_HAS_LANGFUSE` is set at `reasons/llm.py:36` / `:39` after trying `from langfuse.langchain import CallbackHandler as LangfuseCallbackHandler`.
- `_HAS_OPENAI` is set at `reasons/llm.py:43` / `:46` after trying `import openai as _openai_mod`.

Each flag is therefore an **import-availability guard**:

```python
try:
    from langchain_anthropic import ChatAnthropic
    _HAS_LANGCHAIN_ANTHROPIC = True
except ImportError:
    ChatAnthropic = None
    _HAS_LANGCHAIN_ANTHROPIC = False
```

The extras control whether those imports are likely to succeed after installation. They do not directly assign the flags. For example, installing `ftl-reasons[api]` makes `langchain_anthropic` available, so the import at `reasons/llm.py:21` can succeed and set `_HAS_LANGCHAIN_ANTHROPIC = True`. Installing `ftl-reasons[llm]` covers anthropic, vertex, and langfuse, but not OpenAI.

## Where the flags gate behavior

The flags are consumed later in `reasons/llm.py`:

- `_HAS_LANGFUSE` is checked at `reasons/llm.py:162` inside `_get_langfuse_handler`; without the flag, the function returns `None`.
- `_HAS_LANGCHAIN_ANTHROPIC` is checked at `reasons/llm.py:207`.
- `_HAS_LANGCHAIN_VERTEX` is checked at `reasons/llm.py:214`.
- `_HAS_OPENAI` is checked at `reasons/llm.py:251`.

Tests exercise these paths by patching the module globals directly, e.g. `tests/test_llm.py:188`, `:204`, `:237`, `:304`, so they can test provider branches without depending on installed extras.

## Topics to Explore

- [file] `pyproject.toml` — Verify the optional dependency groups and confirm which extras bundle which provider packages.
- [file] `reasons/llm.py` — Inspect the full provider branches around lines 162, 207, 214, and 251 to see the runtime behavior when each flag is false.
- [file] `tests/test_llm.py` — See how tests patch `_HAS_*` flags to simulate installed and missing optional dependencies.
- [function] `reasons/llm.py:_get_langfuse_handler` — Explore how `_HAS_LANGFUSE` combines with `LANGFUSE_PUBLIC_KEY` to decide whether tracing is enabled.
- [general] `optional-dependency-runtime-matrix` — Map each install extra to the flags it can enable and the provider paths it unlocks.

## Beliefs

- `pyproject-extras-map-to-provider-packages` — The `api`, `vertex`, `langfuse`, and `openai` extras install `langchain-anthropic`, `langchain-google-vertexai`, `langfuse`, and `openai`, respectively.
- `llm-extra-excludes-openai` — The `llm` extra includes anthropic, vertex, and langfuse dependencies but does not include `openai`.
- `has-flags-are-import-guards` — Each `_HAS_*` flag is set `True` only by a successful optional import and `False` by `ImportError`.
- `pyproject-extras-enable-imports-not-direct-flag-assignment` — `pyproject.toml` extras affect `_HAS_*` values indirectly by making imports available, not by assigning the flags directly.
- `langfuse-flag-gates-callback-handler` — `_HAS_LANGFUSE` gates creation/use of `LangfuseCallbackHandler` in `_get_langfuse_handler` at `reasons/llm.py:162`.
