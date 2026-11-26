---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: _invoke_openai in reasons/llm.py

## `_invoke_openai`

### 1. Purpose
`_invoke_openai` is the OpenAI-specific backend for the public `invoke_model()` dispatcher. It exists so models named with the `openai:` prefix can be called directly through the official OpenAI Python SDK’s Responses API, without going through CLI subprocesses or LangChain adapters.

It is intentionally private and is called from `invoke_model()` when `model.startswith("openai:")`.

### 2. Contract
**Preconditions**
- The `openai` package must be importable; otherwise the function raises `ImportError`.
- `model` must contain a colon and should be formatted as `openai:<model-name>`. The public caller enforces the `openai:` prefix, but this function itself only does `split(":", 1)[1]`.
- Valid OpenAI credentials/configuration must be available to `_openai_mod.OpenAI()`.
- `prompt` must be acceptable as the `input` value for `client.responses.create`.
- The installed OpenAI SDK must support the Responses API and the expected response object shape.

**Postconditions**
- On success, returns a string assembled from message text parts in the OpenAI response.
- On success, records one call and token counts through `_record_cost`, with `cost_usd` hard-coded to `0.0`.
- The recorded model key is the original full string, e.g. `"openai:gpt-4o"`, not the stripped OpenAI model name.
- No Langfuse callback is used, unlike `_invoke_api`.

**Invariants**
- It does not mutate the input prompt or model string.
- It performs a synchronous, non-streaming API call.
- It updates the global cost tracker via `_record_cost`, which is lock-protected.
- It does not compute real USD cost; token accounting is recorded but dollar cost is always zero.

### 3. Parameters
- `prompt: str` — Sent directly as `input=prompt` to the Responses API. It is not rewrapped into a chat message list here.
- `model: str` — Expected to be `openai:<model-name>`. The function strips the first colon-delimited segment and passes the remainder to OpenAI. For example:
  - `"openai:gpt-4o"` → `"gpt-4o"`
  - `"openai:gpt-4.1-mini"` → `"gpt-4.1-mini"`
  Directly calling with `"gpt-4o"` raises `IndexError` because there is no colon.
- `timeout: int = 300` — Forwarded to the OpenAI SDK request as `timeout=timeout`. It is not enforced by this function itself; the SDK/HTTP layer handles it.

### 4. Return Value
Returns a single string containing concatenated `.text` values from `response.output` items whose `type == "message"`, across their content parts that have a `text` attribute. It does not insert separators between parts.

If the response contains no message text parts, it returns `""`. Callers must handle empty-string responses. Tool calls, reasoning items, refusals, or other non-message/non-text output parts are silently ignored.

### 5. Algorithm
1. Check `_HAS_OPENAI`. If false, raise `ImportError` with installation instructions.
2. Strip the `openai:` prefix:
   ```python
   openai_model = model.split(":", 1)[1]
   ```
3. Construct a new OpenAI client:
   ```python
   client = _openai_mod.OpenAI()
   ```
4. Call the Responses API synchronously:
   ```python
   response = client.responses.create(
       model=openai_model,
       input=prompt,
       timeout=timeout,
   )
   ```
5. Initialize `text = ""`.
6. Iterate over `response.output`.
7. For each output item with `item.type == "message"`, iterate over `item.content`.
8. Append `part.text` for each content part that has a `text` attribute.
9. Extract usage:
   ```python
   input_tokens = response.usage.input_tokens if response.usage else 0
   output_tokens = response.usage.output_tokens if response.usage else 0
   ```
10. Record token usage and a zero dollar cost:
    ```python
    _record_cost(model, input_tokens, output_tokens, 0.0)
    ```
11. Return the accumulated text.

### 6. Side Effects
- Performs network I/O to the OpenAI API.
- Reads OpenAI configuration/credentials from the environment or SDK-managed configuration when `OpenAI()` is instantiated.
- Mutates the module-level `_cost_tracker` through `_record_cost`:
  - increments global call count,
  - increments global input/output token counts,
  - creates/increments per-model stats under the full model string.
- Creates a new OpenAI client per call. The client is not explicitly closed in this function.

It does **not** record elapsed time itself. `invoke_model()` records elapsed time in its `finally` block, including per-model elapsed time if `_record_cost` created the model entry.

### 7. Error Handling
- Raises `ImportError` if the OpenAI package is not installed.
- Raises `IndexError` if `model` has no colon and this function is called directly.
- Propagates OpenAI SDK exceptions such as authentication errors, bad request errors, connection errors, rate-limit errors, and `APITimeoutError`. This function does not catch or translate them.
- Note a documentation mismatch: `invoke_model()` documents `subprocess.TimeoutExpired` on timeout, but `_invoke_openai` does not translate OpenAI timeout errors into `subprocess.TimeoutExpired`.
- May raise `AttributeError` or `TypeError` if the OpenAI SDK response shape differs from expectations, e.g. missing `response.output`, malformed content parts, or non-string `.text`.
- If the API call succeeds but text extraction fails before `_record_cost`, the usage is not recorded.
- If the API succeeds and extraction completes, it records the call even if the extracted text is empty.

### 8. Usage Patterns
Typical usage is indirect through the public dispatcher:

```python
from reasons.llm import invoke_model

text = invoke_model(
    "Summarize this document...",
    model="openai:gpt-4o",
    timeout=120,
)
```

Caller obligations:
- Install the OpenAI extra/dependency.
- Configure OpenAI credentials.
- Use a valid `openai:<model-name>` string.
- Handle OpenAI SDK exceptions and empty response strings.
- Understand that token usage is tracked but USD cost is recorded as `0.0`.

### 9. Dependencies
- Internal:
  - `_HAS_OPENAI` and `_openai_mod` from the optional OpenAI import.
  - `_record_cost` for cost/token aggregation.
- External:
  - `openai` Python package.
  - OpenAI Responses API.
  - OpenAI credentials/configuration available to `OpenAI()`.
- SDK-version assumptions:
  - The installed SDK must expose `OpenAI().responses.create`.
  - The response must expose `output`, `output[].type`, `output[].content`, and `usage.input_tokens` / `usage.output_tokens`.

## Topics to Explore
- [file] `reasons/llm.py` — See how `_invoke_openai` fits into the broader provider routing and cost-tracking design.
- [function] `reasons/llm.py:invoke_model` — Understand the public dispatch logic, timeout documentation, and elapsed-time accounting around OpenAI calls.
- [function] `reasons/llm.py:_record_cost` — Inspect how per-model token and call statistics are accumulated under the full model name.
- [general] `openai-responses-api` — Explore the Responses API output structure, content-part types, and usage fields assumed by this code.
- [file] `tests/test_llm.py` — Review existing tests/mocks for provider routing and whether OpenAI response extraction and cost recording are covered.

## Beliefs
- `invoke-openai-requires-openai-package` — `_invoke_openai` raises `ImportError` before calling OpenAI when `_HAS_OPENAI` is false.
- `openai-model-prefix-is-stripped-before-api-call` — A model like `openai:gpt-4o` passes only `gpt-4o` to `client.responses.create`.
- `openai-cost-tracking-records-zero-usd` — `_record_cost` is called with token counts and `cost_usd=0.0`, so OpenAI calls never contribute a nonzero dollar cost.
- `openai-response-text-extraction-only-includes-message-text-parts` — The return value is built only from `response.output` message items whose content parts have a `text` attribute.
- `openai-timeout-errors-propagate-unwrapped` — OpenAI SDK timeout exceptions are not caught or converted to `subprocess.TimeoutExpired` by this function.
