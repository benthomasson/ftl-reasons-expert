---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore whether agent IDs need URL encoding because `_resolve_config` does not sanitize or escape them.

# Agent ID URL Encoding

The current code **assumes agent IDs are safe URL path segments**. That assumption is visible in `reasons/mind_service.py`:

- `_resolve_config` is defined at `reasons/mind_service.py:9`.
- It returns `url.rstrip("/"), agent_id, api_key` at `reasons/mind_service.py:29`. It normalizes only the base URL and does **not** validate or escape `agent_id`.
- `fetch_export` builds the export endpoint with a raw f-string at `reasons/mind_service.py:34`:
  ```python
  endpoint = f"{url}/api/agents/{agent_id}/export"
  ```
- `push_belief` does the same for the beliefs endpoint at `reasons/mind_service.py:56`:
  ```python
  endpoint = f"{url}/api/agents/{agent_id}/beliefs"
  ```

So the real encoding gap is not only in `_resolve_config`; it is in the URL-construction functions. `import_api` and `export_api` pass `resolved_agent_id` into these functions (`reasons/api.py:2417-2419`, `reasons/api.py:2452-2453`, `reasons/api.py:2469`), but no encoding happens before interpolation.

The observations also show no `urlencode`, `quote`, `quote_plus`, or `urljoin` usage anywhere. Tests only exercise safe IDs such as `agent-1` and `a` (`tests/test_mind_service.py:40`, `:84`, `:122`). There are no observed tests for IDs containing `/`, `?`, `#`, spaces, unicode, control characters, or `..`.

## Practical conclusion

If `agent_id` can contain arbitrary user-controlled or externally supplied text, then **yes, it should be encoded as a URL path segment** before building the endpoint. Otherwise values like these change the URL structure:

- `agent_id="foo/bar"` → `/api/agents/foo/bar/export`
- `agent_id="foo?x=1"` → query string injection
- `agent_id="foo#bar"` → fragment injection
- `agent_id="../other"` → possible path traversal or unexpected routing, depending on server normalization
- spaces or unicode may produce invalid or inconsistently encoded URLs

The appropriate fix is path-segment encoding, not query encoding:

```python
from urllib.parse import quote

agent_segment = quote(agent_id, safe="")
endpoint = f"{url}/api/agents/{agent_segment}/export"
```

`quote(agent_id, safe="")` is preferable to `urlencode`/`quote_plus` here because `agent_id` is a path segment, not a query parameter. However, if the server decodes `%2F` back into `/` before route matching, then encoding alone may not be sufficient; the API contract should also forbid or explicitly support slashes in agent IDs.

If agent IDs are guaranteed by the server/API to match a safe charset such as `[A-Za-z0-9_-]+`, then encoding is not strictly required, but that constraint should be validated or documented rather than silently assumed. The missing observation is the agent-ID contract: allowed characters, whether IDs may already be percent-encoded, and how the server route parses `{agent_id}`.

## Topics to Explore

- [function] `reasons/mind_service.py:_resolve_config` — Decide whether this function should validate agent IDs or leave URL escaping to the endpoint builders.
- [function] `reasons/mind_service.py:fetch_export` — Add or verify path-segment encoding for `agent_id` in the export URL.
- [function] `reasons/mind_service.py:push_belief` — Apply the same encoding rule for the beliefs URL so both endpoints behave consistently.
- [general] `agent-id-api-contract` — Confirm the allowed agent-ID charset and whether the server decodes percent-encoded path segments.
- [file] `tests/test_mind_service.py` — Add regression tests for IDs containing `/`, `?`, `#`, spaces, unicode, and `..`.

## Beliefs

- `_resolve_config-does-not-escape-agent-id` — `_resolve_config` returns `agent_id` directly in its tuple after only applying `rstrip("/")` to the base URL.
- `agent-id-is-interpolated-as-path-segment` — `fetch_export` and `push_belief` insert raw `agent_id` values into `/api/agents/{agent_id}/...` URLs.
- `url-encoding-helpers-absent` — No observed code uses `urlencode`, `quote`, `quote_plus`, or `urljoin` in the relevant paths.
- `tests-cover-only-safe-agent-ids` — Existing observed URL assertions use safe IDs like `agent-1` and do not cover reserved or unsafe characters.
- `api-functions-pass-resolved-agent-id-to-url-builders` — `import_api` and `export_api` pass `resolved_agent_id` into `fetch_export`/`push_belief` without intermediate encoding.
