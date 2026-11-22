---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _invoke_api in reasons/llm.py

## Purpose

`_invoke_api` is an internal adapter that invokes an Anthropic-style model through LangChain for the `api:` and `vertex:` model prefixes. It exists to centralize:

- provider selection and dependency checks,
- Vertex project/region environment handling,
- optional Langfuse tracing,
- conversion of provider failures into the same error categories used by the CLI path,
- normalization of timeout failures to `subprocess.TimeoutExpired`.

It is called by `invoke_model` when the model string starts with `api:` or `vertex:`. It bypasses the CLI command resolution path entirely.

## Contract

### Preconditions

- `model` should be a string containing a colon, e.g. `api:claude-...` or `vertex:claude-...`.
- The prefix before the first colon must be exactly `api` or `vertex`.
- For `api:`, `langchain_anthropic` must be importable at module import time; otherwise `_HAS_LANGCHAIN_ANTHROPIC` is false.
- For `vertex:`, `langchain_google_vertexai.model_garden` must be importable at module import time; otherwise `_HAS_LANGCHAIN_VERTEX` is false.
- For `vertex:`, `GOOGLE_CLOUD_PROJECT` must be set and non-empty.
- Provider credentials are assumed to be configured in the environment or cloud SDK, e.g. Anthropic API key or Google application default credentials. This function does not validate them.
- The `llm.invoke(...)` result is assumed to be a LangChain message object with a `.content` attribute.
- `.content` is assumed to be a string because the function is annotated `-> str`, but this is not enforced at runtime.

### Postconditions

On success:

- Exactly one `llm.invoke(prompt, config=config)` call has been made.
- The return value is exactly `response.content`, with no stripping, string conversion, or post-processing.
- If a Langfuse handler is available, it is included in `config["callbacks"]`.

On failure:

- Dependency, configuration, constructor, and Langfuse initialization errors propagate before the `llm.invoke` try/except block.
- Exceptions raised by `llm.invoke` are either converted to `subprocess.TimeoutExpired` or wrapped in `RuntimeError`.
- Non-timeout `llm.invoke` failures retain the original exception as `__cause__`.

### Invariants / Assumptions Not Enforced by Types

- `model.split(":", 1)` assumes there is a colon. A model without a colon raises a generic unpack `ValueError`.
- An empty model name is allowed by the parsing logic, e.g. `api:` produces `name == ""`; the provider client is responsible for rejecting it.
- `timeout` is assumed to be convertible to `float`; `float(timeout)` happens before the invoke try/except.
- The function does not record token usage or cost. API Anthropic/Vertex calls are absent from `_record_cost`.
- Timeout detection is heuristic: it searches for `"timeout"` in the exception class name or exception string. It does not inspect specific timeout exception types.

## Parameters

| Parameter | Type | Meaning | Notes / Edge Cases |
|---|---|---|---|
| `prompt` | `str` | The user prompt passed directly to `llm.invoke`. | No templating, trimming, or message wrapping is performed. |
| `model` | `str` | Colon-delimited model identifier. | First split is on the first colon. `api:` uses `ChatAnthropic`; `vertex:` uses `ChatAnthropicVertex`. Unknown prefix raises `ValueError`. Missing colon raises `ValueError` from tuple unpacking. Extra colons become part of `name`. |
| `timeout` | `int`, default `300` | Timeout in seconds passed to the provider client. | Converted to `float(timeout)`. Passed as `timeout=` for `ChatAnthropic` and `request_timeout=` for `ChatAnthropicVertex`. Negative/zero values are not validated. Invalid values raise before the invoke try/except. |

## Return Value

Returns `response.content` from the LangChain response object.

Under normal LangChain Anthropic usage this is a string containing the assistant response. However:

- The function does not coerce the value with `str(...)`.
- Some LangChain versions/provider configurations can return structured content blocks as a list.
- If the response object lacks `.content`, an `AttributeError` is raised outside the invoke try/except.

Callers should therefore treat the return as the provider response content and be aware that the declared `str` return type is an assumption, not a runtime guarantee.

## Algorithm

1. Split `model` once on the first colon:
   ```python
   prefix, name = model.split(":", 1)
   ```

2. If `prefix == "api"`:
   - Check `_HAS_LANGCHAIN_ANTHROPIC`.
   - If unavailable, raise `ImportError` with an install hint for `ftl-reasons[api]`.
   - Instantiate:
     ```python
     ChatAnthropic(model=name, timeout=float(timeout))
     ```

3. If `prefix == "vertex"`:
   - Check `_HAS_LANGCHAIN_VERTEX`.
   - If unavailable, raise `ImportError` with an install hint for `ftl-reasons[vertex]`.
   - Read `GOOGLE_CLOUD_PROJECT`; if unset or empty, raise `RuntimeError`.
   - Read `GOOGLE_CLOUD_REGION`; default to `"us-east5"`.
   - Instantiate:
     ```python
     ChatAnthropicVertex(
         model_name=name,
         project=project,
         location=location,
         request_timeout=float(timeout),
     )
     ```

4. If `prefix` is anything else, raise:
   ```python
   ValueError(f"Unknown API prefix: {prefix}")
   ```

5. Build a LangChain config dict:
   - Start with `{}`.
   - Call `_get_langfuse_handler()`.
   - If a handler is returned, set `config["callbacks"] = [handler]`.

6. Invoke the model inside a try/except:
   ```python
   response = llm.invoke(prompt, config=config)
   ```

7. On any `Exception` from `llm.invoke`:
   - Get the exception class name.
   - If either the class name or the stringified exception contains `"timeout"` case-insensitively, raise:
     ```python
     subprocess.TimeoutExpired(model, timeout)
     ```
     chained from the original exception.
   - Otherwise raise:
     ```python
     RuntimeError(f"{model} failed: {exc}")
     ```
     chained from the original exception.

8. Return `response.content`.

## Side Effects

- Performs a network call through LangChain to Anthropic or Vertex AI.
- Instantiates a new provider client on each call; the client is not cached.
- Reads environment variables:
  - `GOOGLE_CLOUD_PROJECT` for Vertex,
  - `GOOGLE_CLOUD_REGION` for Vertex,
  - indirectly `LANGFUSE_PUBLIC_KEY` via `_get_langfuse_handler`.
- May initialize and cache the global Langfuse handler via `_get_langfuse_handler`.
- May emit tracing data to Langfuse if configured.
- Does **not** update token/cost tracking via `_record_cost`.
- Does not mutate the prompt or model strings.
- Does not write files directly, though provider SDKs may.

## Error Handling

| Exception | Source / Condition |
|---|---|
| `ImportError` | Missing `langchain-anthropic` for `api:` or missing `langchain-google-vertexai` for `vertex:`. Raised before the invoke try/except. |
| `RuntimeError` | Missing/empty `GOOGLE_CLOUD_PROJECT` for `vertex:`. Also used to wrap non-timeout failures from `llm.invoke`. |
| `ValueError` | Unknown prefix; missing colon causing unpack failure; invalid `timeout` conversion; provider constructor validation failures. |
| `TypeError` | `float(timeout)` on unsupported types, e.g. `None` or arbitrary objects. |
| `subprocess.TimeoutExpired` | Raised when `llm.invoke` raises an exception whose class name or message contains `"timeout"` case-insensitively. This is a normalization choice, not necessarily a real subprocess timeout. |
| Langfuse initialization errors | Propagate from `_get_langfuse_handler()` before the invoke try/except. |
| `AttributeError` | If `llm.invoke` returns an object without `.content`. This occurs after the try/except and is not wrapped. |
| Other provider exceptions | If raised during `llm.invoke` and not classified as timeout, they are wrapped in `RuntimeError`. |

The timeout classification is heuristic and can misclassify errors. For example, a non-timeout error whose message mentions “timeout configuration” would be converted to `subprocess.TimeoutExpired`. Conversely, a timeout exception that does not contain the substring `"timeout"` would not be normalized.

## Usage Patterns

Typical call path:

```python
invoke_model(prompt, model="api:claude-3-5-sonnet-latest", timeout=300)
invoke_model(prompt, model="vertex:claude-3-5-sonnet@20240620", timeout=300)
```

Inside `invoke_model`:

```python
if model.startswith("api:") or model.startswith("vertex:"):
    return _invoke_api(prompt, model, timeout)
```

Caller obligations:

- Ensure provider dependencies are installed if using `api:` or `vertex:`.
- Ensure `GOOGLE_CLOUD_PROJECT` is set for `vertex:`.
- Ensure provider credentials are available through the environment or cloud SDK.
- Handle `ImportError`, `RuntimeError`, `ValueError`, and `subprocess.TimeoutExpired`.
- Do not assume these API calls appear in `get_cost_summary()` token/cost totals. Only elapsed time is updated by `invoke_model`’s outer `finally` block.

## Dependencies

- Standard library:
  - `os` for environment variables,
  - `subprocess` for `subprocess.TimeoutExpired`.
- Optional external:
  - `langchain_anthropic.ChatAnthropic`,
  - `langchain_google_vertexai.model_garden.ChatAnthropicVertex`,
  - `langfuse.langchain.CallbackHandler` via `_get_langfuse_handler`.
- Internal:
  - `_get_langfuse_handler`,
  - `_HAS_LANGCHAIN_ANTHROPIC`,
  - `_HAS_LANGCHAIN_VERTEX`,
  - called from `invoke_model`.
- Provider-side dependencies:
  - Anthropic API credentials for `api:`,
  - Google Cloud project/region and credentials for `vertex:`.

## Topics to Explore

- [file] `tests/test_llm.py` — Explore the tests that exercise this adapter, especially dependency flags, Langfuse callbacks, and timeout mapping.
- [function] `reasons/llm.py:_get_langfuse_handler` — Understand how the optional Langfuse handler is initialized, cached, and gated by environment variables.
- [function] `reasons/llm.py:invoke_model` — See how `_invoke_api` is selected and how outer timing and cost tracking interact with this function.
- [general] `langchain-anthropic-response-content` — Investigate when LangChain `.content` is a plain string versus structured content blocks.
- [general] `api-cost-tracking` — Understand why `api:` and `vertex:` calls are not included in `_record_cost` and what that means for `format_cost_summary()`.

## Beliefs

- `api-prefix-requires-langchain-anthropic` — `api:` models raise `ImportError` unless `_HAS_LANGCHAIN_ANTHROPIC` is true.
- `vertex-prefix-requires-google-cloud-project` — `vertex:` models raise `RuntimeError` when `GOOGLE_CLOUD_PROJECT` is unset or empty.
- `_invoke_api-does-not-record-token-costs` — This function never calls `_record_cost`; token/cost tracking is only performed by CLI JSON parsing and `_invoke_openai`.
- `timeout-classification-is-heuristic` — Timeout detection uses a case-insensitive substring search for `"timeout"` in the exception class name or message, not exception type inspection.
- `return-value-is-not-coerced` — The function returns `response.content` directly and assumes it is a string, with no `str()` conversion or validation.
