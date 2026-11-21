---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: push_belief in reasons/mind_service.py

## Purpose

`push_belief` is a small HTTP client helper for the agentic-mind-service. It pushes one belief to an agent by sending a JSON `POST` request to:

```text
{url}/api/agents/{agent_id}/beliefs
```

It exists to encapsulate the request construction, bearer-token authentication, JSON serialization, timeout, response decoding, and the special handling of HTTP 401 responses.

## Contract

### Preconditions

- `url`, `agent_id`, and `api_key` must identify a reachable mind service and valid credentials.
- `node_id` and `text` are expected to be strings and are always included in the JSON body.
- `sl` and `source` are optional strings. They are included only if truthy.
- The service endpoint is expected to accept:

```http
POST /api/agents/{agent_id}/beliefs
Authorization: Bearer {api_key}
Content-Type: application/json
```

with a body shaped like:

```json
{
  "node_id": "...",
  "text": "...",
  "sl": "...",
  "source": "..."
}
```

where `sl` and `source` are omitted when empty/falsy.

### Postconditions

- On a successful HTTP response, returns the decoded and JSON-parsed response body.
- On HTTP 401, raises `RuntimeError("Authentication failed. Check your MIND_API_KEY.")`.
- On other `HTTPError`s, re-raises the original `HTTPError`.
- On network, timeout, decoding, or JSON parsing failures, lets the underlying exception propagate.

### Important invariants

- The request method is always `POST`.
- `node_id` and `text` are always present in the request body.
- `Authorization` is always sent as `Bearer {api_key}`.
- `Content-Type` is always `application/json`.
- No local state is mutated.
- No retry, logging, or config resolution is performed inside this function.

### Assumptions not enforced by types

- `url` is assumed to be a base URL without a trailing slash. Unlike `_resolve_config`, `push_belief` does not call `url.rstrip("/")`, so passing `"http://host/"` produces a double slash in the endpoint.
- `agent_id` is assumed to be safe as a raw path segment; it is not URL-encoded.
- `api_key` is assumed to be safe inside an HTTP header value.
- `node_id`, `text`, `sl`, and `source` are assumed to be JSON-serializable strings, but the function does not validate them.
- The response body is assumed to be valid UTF-8 JSON, and the type annotation assumes the parsed result is a `dict`.

## Parameters

- `url: str` — Base URL of the mind service. It is concatenated directly with `/api/agents/...`, so callers should normalize it without a trailing slash.
- `agent_id: str` — Agent identifier used as a raw path component in the endpoint. Not URL-encoded.
- `api_key: str` — Bearer token placed verbatim into the `Authorization` header.
- `node_id: str` — Identifier of the belief/node being pushed. Always included in the body.
- `text: str` — Belief text/content. Always included in the body.
- `sl: str = ""` — Optional field. Included in the JSON body only when truthy, e.g. non-empty string.
- `source: str = ""` — Optional field. Included in the JSON body only when truthy, e.g. non-empty string.

Edge cases:

- Passing `sl=""` or `source=""` omits the field entirely; callers cannot explicitly send an empty string for these fields through this function.
- Passing `None` for `node_id` or `text` is possible at runtime despite the type hint; `json.dumps` would encode it as JSON `null`.
- Passing a trailing slash in `url` creates a double slash before `/api/...`.
- Passing non-string values may still serialize if JSON-serializable, or raise `TypeError` if not.

## Return Value

The function returns:

```python
dict
```

per its annotation, but at runtime it returns whatever `json.loads(...)` produces. In practice the service is expected to return a JSON object, but the function does not enforce that. A successful response containing a JSON list, scalar, or `null` would violate the declared contract but would still be returned by `json.loads`.

The caller must handle:

- `RuntimeError` for authentication failure.
- `HTTPError` for non-401 non-2xx responses.
- `URLError`, socket timeouts, or other network exceptions.
- `JSONDecodeError` if the response is not valid JSON.
- `UnicodeDecodeError` if the response bytes are not valid UTF-8.

## Algorithm

1. Build the endpoint:

   ```python
   f"{url}/api/agents/{agent_id}/beliefs"
   ```

2. Create the request body with the required fields:

   ```python
   body = {"node_id": node_id, "text": text}
   ```

3. Conditionally add optional fields:

   ```python
   if sl:
       body["sl"] = sl
   if source:
       body["source"] = source
   ```

4. Serialize the body to JSON and encode it as UTF-8:

   ```python
   data = json.dumps(body).encode("utf-8")
   ```

5. Build a `urllib.request.Request` with:
   - the endpoint URL,
   - POST data,
   - `Authorization: Bearer {api_key}`,
   - `Content-Type: application/json`,
   - `method="POST"`.

6. Send the request with a 30-second timeout:

   ```python
   with urlopen(req, timeout=30) as resp:
   ```

7. Read and decode the response body as UTF-8, then parse it as JSON:

   ```python
   return json.loads(resp.read().decode("utf-8"))
   ```

8. If an `HTTPError` occurs:
   - if `e.code == 401`, raise a `RuntimeError` with an authentication message and chain the original exception;
   - otherwise, re-raise the original `HTTPError`.

## Side Effects

- Performs an external HTTP `POST` request to the configured mind service.
- Presumably creates or updates a belief on the remote service, depending on server-side semantics.
- Sends credentials in the `Authorization` header.
- Uses network I/O and DNS resolution.
- Does not write files, mutate module state, read environment variables, log, cache, or retry.
- The response object is closed by the `with urlopen(...)` context manager.

## Error Handling

- `HTTPError` with status `401` is converted to `RuntimeError`:

  ```python
  raise RuntimeError(
      "Authentication failed. Check your MIND_API_KEY."
  ) from e
  ```

- Other `HTTPError` statuses are re-raised unchanged. For example, a 404 or 500 propagates as `HTTPError`.
- Unlike `fetch_export`, this function does not special-case 404.
- No retries are attempted.
- No generic `except Exception` is used, so unexpected failures propagate.
- JSON serialization errors from `json.dumps`, such as unserializable values, propagate.
- Invalid response JSON propagates as `json.JSONDecodeError`.
- Network failures and timeouts propagate from `urlopen`.

## Usage Patterns

Typical usage:

```python
response = push_belief(
    url="http://mind-service",
    agent_id="agent-123",
    api_key="secret",
    node_id="node-1",
    text="The sky is blue",
    sl="some-sl",
    source="observation",
)
```

Callers usually:

- resolve configuration first, potentially via `_resolve_config`, which normalizes the URL with `rstrip("/")`;
- handle `RuntimeError` for auth failures;
- handle `HTTPError`, timeout, and connection failures;
- assume the successful response is a JSON object if they rely on the `dict` annotation.

Because `push_belief` does not normalize `url`, callers should pass a base URL without a trailing slash. Because it does not retry, callers that need resilience should implement retry logic externally.

## Dependencies

Standard library only:

- `json` — request serialization and response parsing.
- `urllib.error.HTTPError` — HTTP status exception handling.
- `urllib.request.Request`, `urllib.request.urlopen` — HTTP request construction and execution.

The function also depends on the external agentic-mind-service API contract for the endpoint path, bearer auth, payload schema, and response shape.

## Topics to Explore

- [file] `tests/test_mind_service.py` — See how request payloads, headers, error mappings, and response parsing are asserted.
- [function] `reasons/mind_service.py:_resolve_config` — Shows config resolution and URL normalization that `push_belief` does not perform itself.
- [function] `reasons/mind_service.py:fetch_export` — Compare its error handling, especially its special handling of HTTP 404.
- [general] `mind-service-belief-api-contract` — Confirm server expectations for endpoint path, auth, payload fields, and response shape.
- [general] `urllib-http-error-semantics` — Understand which network and HTTP failures `urllib` raises and how they propagate.

## Beliefs

- `push-belief-posts-json-to-agent-beliefs-endpoint` — `push_belief` sends an HTTP POST to `{url}/api/agents/{agent_id}/beliefs` with a JSON body containing `node_id` and `text`.
- `push-belief-omits-empty-optional-fields` — `sl` and `source` are included in the request body only when truthy, so empty strings are omitted.
- `push-belief-maps-401-to-runtime-error` — A 401 HTTP response is converted into a `RuntimeError`; other `HTTPError` codes are re-raised.
- `push-belief-does-not-normalize-url` — Unlike `_resolve_config`, `push_belief` does not strip trailing slashes from `url`.
- `push-belief-returns-parsed-json` — The return value is `json.loads` of the response body, and the declared `dict` type is not enforced at runtime.
