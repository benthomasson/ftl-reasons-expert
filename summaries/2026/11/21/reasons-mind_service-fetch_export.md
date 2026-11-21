---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: fetch_export in reasons/mind_service.py

## `fetch_export`

### Purpose

`fetch_export` is a small HTTP client helper for the agentic-mind-service. It performs:

```text
GET {url}/api/agents/{agent_id}/export
```

with a Bearer token and returns the raw response body as a UTF-8 string. The docstring says this downloads the “full network JSON,” but the function itself does **not** parse or validate JSON. It is a transport-layer helper: compose URL, send authenticated GET, decode body, map two common HTTP failures into user-facing `RuntimeError`s.

### Contract

**Preconditions**

- `url` is expected to be a valid service base URL with a scheme, e.g. `https://mind.example.com`.
- `url` is expected to be the service root, typically normalized without a trailing slash by `_resolve_config`.
- `agent_id` is expected to be a non-empty, URL-path-safe identifier.
- `api_key` is expected to be a valid bearer token.
- The remote endpoint `/api/agents/{agent_id}/export` must exist and return the exported network representation.

**Postconditions**

- On success, returns the entire response body decoded with strict UTF-8.
- No JSON parsing, schema validation, or content-type checking is performed.
- The response body is fully read into memory.
- The HTTP response is closed via the `with` block.
- On HTTP 401, raises `RuntimeError` chained from the original `HTTPError`.
- On HTTP 404, raises `RuntimeError` chained from the original `HTTPError`.
- Other `HTTPError`s are re-raised unchanged.

**Invariants**

- Every request includes an `Authorization: Bearer <api_key>` header.
- The request method is GET because `Request` is created without body data or an explicit method.
- The timeout for `urlopen` is always 60 seconds.

### Parameters

- `url: str`
  - Base URL for the mind service.
  - Intended usage relies on `_resolve_config`, which strips trailing `/`.
  - If a trailing slash is passed directly, the endpoint becomes `//api/...`.
  - Empty, malformed, or non-HTTP URLs are not validated here.

- `agent_id: str`
  - Inserted directly into the URL path by f-string interpolation.
  - Not URL-encoded.
  - If it contains `/`, `?`, `#`, spaces, or other reserved characters, the path can change or become invalid.
  - Empty string is not rejected locally.

- `api_key: str`
  - Inserted into the `Authorization` header as `Bearer {api_key}`.
  - Not validated, stripped, or checked for newline/header-safety.
  - Empty key is not rejected locally; it will likely lead to a server 401.

### Return Value

Returns a `str` containing the full response body decoded as UTF-8. The caller receives raw JSON text, not a Python object. Callers that need structured data must call `json.loads(...)` or equivalent and handle parse failures themselves.

A successful empty body would return `""`. A successful but non-JSON body would also be returned as-is.

### Algorithm

1. Build the endpoint:
   ```python
   f"{url}/api/agents/{agent_id}/export"
   ```
2. Create a `urllib.request.Request` with:
   - the built endpoint
   - header `Authorization: Bearer {api_key}`
   - no body data, so the default method is GET.
3. Open the URL with `urlopen(req, timeout=60)`.
4. Enter the response context manager so the response is closed afterward.
5. Read the full response body bytes with `resp.read()`.
6. Decode the bytes using strict UTF-8 via `.decode("utf-8")`.
7. Return the decoded string.
8. If `urlopen` raises `HTTPError`:
   - code `401` -> raise `RuntimeError("Authentication failed. Check your MIND_API_KEY.") from e`
   - code `404` -> raise `RuntimeError(f"Agent not found: {agent_id}") from e`
   - any other code -> bare `raise`, re-raising the original `HTTPError`

### Side Effects

- Performs an outbound HTTP GET request.
- Sends the API key in the Authorization header.
- Reads the full remote response into memory.
- May leave server-side access logs.
- Does not mutate local state, write files, or cache anything.
- Does not retry.

### Error Handling

Handled errors:

- `HTTPError` with code `401`
  - Converted to `RuntimeError`.
  - The original exception is preserved as `__cause__` through `from e`.

- `HTTPError` with code `404`
  - Converted to `RuntimeError` with the agent ID in the message.
  - Also preserves the original exception as cause.

Not handled:

- Other `HTTPError` statuses, e.g. 400, 403, 500. These are re-raised as `HTTPError`.
- Network failures such as `URLError`, connection resets, DNS failures.
- Timeouts from `urlopen(..., timeout=60)`.
- Invalid URL construction errors.
- Invalid header values or invalid URL schemes.
- `UnicodeDecodeError` if the response body is not valid UTF-8.
- JSON errors are not possible inside this function because no JSON parsing occurs.

Assumption worth noting: a 404 is assumed to mean “agent not found.” It could also mean the endpoint path is wrong or the service route is unavailable.

### Usage Patterns

Typical usage:

```python
url, agent_id, api_key = _resolve_config(...)
raw = fetch_export(url, agent_id, api_key)
network = json.loads(raw)
```

Callers should:

- Catch `RuntimeError` when they want friendly messages for auth failure or missing agent.
- Catch `HTTPError`, `URLError`, timeout exceptions, and decode/JSON errors separately if they need robust handling.
- Avoid assuming the response is valid JSON.
- Avoid passing an `agent_id` containing unescaped reserved URL characters.
- Avoid passing a base URL with a trailing slash unless the server tolerates `//api/...`.

### Dependencies

- Standard library:
  - `urllib.error.HTTPError`
  - `urllib.request.Request`
  - `urllib.request.urlopen`
- External service endpoint:
  - `GET /api/agents/{agent_id}/export`
- Related module-level helper:
  - `_resolve_config` is not called directly by `fetch_export`, but callers likely use it to normalize `url` and resolve credentials.

### Assumptions Not Enforced by Types

- `url` is not just any string; it must be a valid base URL with a scheme.
- `url` should not contain a trailing slash if the double slash is not desired.
- `agent_id` must be URL-path-safe because it is interpolated without escaping.
- `agent_id` should be non-empty.
- `api_key` must be non-empty and safe for an HTTP header.
- The endpoint must return UTF-8 text.
- A 2xx response is assumed to be a successful export; content type and JSON validity are not checked.
- A 404 is assumed to mean the agent does not exist rather than a route/service problem.

## Topics to Explore

- [file] `tests/test_mind_service.py` — Verify expected behavior for success, 401, 404, and non-mapped HTTP failures.
- [function] `reasons/mind_service.py:_resolve_config` — See how URL normalization and credential resolution are intended to happen before `fetch_export`.
- [function] `reasons/mind_service.py:push_belief` — Contrast the write path’s JSON parsing and narrower error mapping.
- [general] `urllib-http-error-semantics` — Understand when `urlopen` raises `HTTPError` versus `URLError` or timeout-related exceptions.
- [general] `url-path-segment-escaping` — Explore how unescaped `agent_id` values affect URL construction and whether callers should quote path segments.

## Beliefs

- `fetch-export-returns-raw-utf8-string` — On success, `fetch_export` returns the response body decoded as UTF-8 and does not parse it as JSON.
- `fetch-export-maps-401-to-runtime-error` — An HTTP 401 raises `RuntimeError` with an authentication failure message chained from the original `HTTPError`.
- `fetch-export-maps-404-to-runtime-error` — An HTTP 404 raises `RuntimeError` identifying the missing agent and chains the original `HTTPError`.
- `fetch-export-propagates-other-http-errors` — Any `HTTPError` other than 401 or 404 is re-raised unchanged.
- `fetch-export-does-not-url-encode-agent-id` — `agent_id` is interpolated directly into the URL path without URL escaping.
