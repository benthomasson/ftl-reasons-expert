---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _resolve_config in reasons/mind_service.py

## Purpose

`_resolve_config` is a small internal configuration helper for the `agentic-mind-service` HTTP client. It merges explicit caller-supplied values with environment variables, validates that all required values are present, and normalizes the base service URL before returning the three values needed by the rest of the module.

It exists so callers do not have to duplicate the fallback/validation logic before calling functions like `fetch_export` and `push_belief`.

## Contract

### Preconditions

- Each parameter may be provided explicitly as a string or omitted/`None`.
- If a parameter is not provided, the corresponding environment variable is expected to supply it:
  - `MIND_SERVICE_URL` for `url`
  - `MIND_AGENT_ID` for `agent_id`
  - `MIND_API_KEY` for `api_key`

### Postconditions

- Returns a 3-tuple: `(url, agent_id, api_key)`.
- All three returned values passed a basic truthiness check before return.
- The returned `url` has all trailing `/` characters removed via `url.rstrip("/")`.
- `agent_id` and `api_key` are returned unchanged.
- If any required value is missing, raises `RuntimeError` instead of returning.

### Invariants

- Truthy explicit arguments take precedence over environment variables.
- Falsy explicit arguments, especially `None` and `""`, fall back to the environment variable.
- Missing fields are collected together before raising, so the caller sees all missing config at once.
- The missing-field order is deterministic: URL, agent ID, API key.

### Assumptions Not Enforced by the Type System

- The function assumes `url` is a `str` if truthy; a non-string truthy `url` will fail later at `url.rstrip("/")`.
- `agent_id` and `api_key` are only checked for truthiness, not format, whitespace, URL-safety, or header-safety.
- `url` is only checked for truthiness before `rstrip`. A URL consisting only of slashes, such as `"/"` or `"///"`, passes validation and is then normalized to an empty string.
- The URL is not validated for scheme, host, or general URL syntax.
- Environment variables are assumed to be string values, which is true for `os.environ`.

## Parameters

### `url: str | None = None`

Optional base URL for the mind service.

Resolution behavior:

- If `url` is truthy, it is used.
- If `url` is `None` or `""`, `MIND_SERVICE_URL` is read.
- If neither is present, the resolved value is `""`.

Normalization:

- After validation, trailing slashes are stripped with `rstrip("/")`.
- This strips all trailing slashes, not just one.
- Leading/trailing whitespace is not stripped.

Edge case:

- `url="/"` is truthy and passes the missing check, but `"/".rstrip("/")` becomes `""`, so the function can return an empty URL despite validation.

### `agent_id: str | None = None`

Optional agent identifier.

Resolution behavior:

- If `agent_id` is truthy, it is used.
- If `agent_id` is `None` or `""`, `MIND_AGENT_ID` is read.
- If neither is present, the resolved value is `""`.

No normalization is applied. The value is later interpolated directly into URL paths such as:

```python
f"{url}/api/agents/{agent_id}/export"
```

So the caller is responsible for providing a URL-safe agent ID if required.

### `api_key: str | None = None`

Optional API key.

Resolution behavior:

- If `api_key` is truthy, it is used.
- If `api_key` is `None` or `""`, `MIND_API_KEY` is read.
- If neither is present, the resolved value is `""`.

No validation or normalization is applied. The value is later used directly in the `Authorization: Bearer ...` header in sibling functions.

## Return Value

Returns:

```python
tuple[str, str, str]
```

as:

```python
(normalized_url, agent_id, api_key)
```

The returned tuple is ordered to match the first three parameters of `fetch_export` and `push_belief`, so callers can use it with tuple unpacking or argument unpacking where appropriate.

The caller must handle:

- `RuntimeError` when required configuration is missing.
- The possibility that `normalized_url` is empty if the original URL consisted only of slashes.
- The responsibility of not logging or otherwise exposing `api_key`.

## Algorithm

1. Resolve `url` using:
   ```python
   url = url or os.environ.get("MIND_SERVICE_URL", "")
   ```
2. Resolve `agent_id` using:
   ```python
   agent_id = agent_id or os.environ.get("MIND_AGENT_ID", "")
   ```
3. Resolve `api_key` using:
   ```python
   api_key = api_key or os.environ.get("MIND_API_KEY", "")
   ```
4. Build a `missing` list for each resolved value that is falsy.
5. If `missing` is non-empty, raise:
   ```python
   RuntimeError(f"Missing required config: {', '.join(missing)}")
   ```
6. Return:
   ```python
   return url.rstrip("/"), agent_id, api_key
   ```

Because `or` short-circuits, the corresponding environment variable is only read when the explicit argument is falsy.

## Side Effects

- Reads process environment variables via `os.environ.get`.
- Raises an exception when configuration is missing.
- Does not perform network I/O.
- Does not read or write files.
- Does not mutate global state.
- Local variables `url`, `agent_id`, and `api_key` are reassigned within the function.

## Error Handling

### Raises `RuntimeError`

If any resolved value is empty/falsy, raises `RuntimeError` with a message like:

```text
Missing required config: url (--url or MIND_SERVICE_URL), agent-id (--agent-id or MIND_AGENT_ID), api-key (--api-key or MIND_API_KEY)
```

Only the missing fields are included.

### Other Possible Exceptions

- `AttributeError` if `url` is truthy but not a string and has no `.rstrip` method.
- No explicit handling for invalid URL formats, invalid agent IDs, or invalid API keys.

The function does not swallow any exceptions.

## Usage Patterns

Typical usage is expected to be from a CLI or higher-level caller:

```python
url, agent_id, api_key = _resolve_config(
    url=args.url,
    agent_id=args.agent_id,
    api_key=args.api_key,
)
```

or with environment-only configuration:

```python
url, agent_id, api_key = _resolve_config()
```

The result can then be passed into sibling functions:

```python
export_json = fetch_export(url, agent_id, api_key)
```

or:

```python
push_belief(url, agent_id, api_key, node_id, text)
```

Caller obligations:

- Catch `RuntimeError` and present a useful configuration error.
- Avoid logging `api_key`.
- Ensure `agent_id` is acceptable for URL path interpolation.
- Be aware that `_resolve_config` normalizes trailing slashes on the URL but does nothing else.

## Dependencies

- `os` from the Python standard library.
- No external packages.
- No other functions in this module are called by `_resolve_config`.

The rest of the module depends on `_resolve_config` only indirectly as a configuration preparation helper for callers that then invoke `fetch_export` or `push_belief`.

## Topics to Explore

- [file] `tests/test_mind_service.py` — Check expected fallback behavior, missing-config error messages, and URL normalization edge cases.
- [function] `reasons/mind_service.py:fetch_export` — See how the resolved URL and agent ID are interpolated into the export endpoint and auth header.
- [function] `reasons/mind_service.py:push_belief` — See how resolved configuration feeds the POST payload and authentication.
- [general] `environment-variable-configuration` — Understand deployment expectations for `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY`.
- [general] `agent-id-url-escaping` — Explore whether agent IDs need URL encoding because `_resolve_config` does not sanitize or escape them.

## Beliefs

- `explicit-truthy-params-override-environment` — For each config value, a truthy explicit argument prevents reading the corresponding environment variable.
- `falsy-explicit-params-fallback-to-environment` — Passing `None` or `""` for a config parameter causes `_resolve_config` to read its environment variable, defaulting to an empty string.
- `missing-config-raises-runtime-error` — If any resolved value is empty, `_resolve_config` raises `RuntimeError` listing all missing fields in URL, agent ID, API key order.
- `url-trailing-slashes-are-stripped` — The returned URL has all trailing `/` characters removed via `rstrip("/")`, while `agent_id` and `api_key` are returned unchanged.
- `url-stripping-occurs-after-missing-check` — Because normalization happens after validation, a URL consisting only of slashes passes the missing check and is returned as an empty string.
