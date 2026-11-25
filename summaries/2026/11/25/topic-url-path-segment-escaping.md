---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore how unescaped `agent_id` values affect URL construction and whether callers should quote path segments.

# Unescaped `agent_id` in URL Construction

## Where the URL is built

All `agent_id`-to-URL interpolation happens in **`reasons/mind_service.py`**:

- `fetch_export` — endpoint constructed at line 34:
  ```python
  endpoint = f"{url}/api/agents/{agent_id}/export"
  ```
- `post_belief` — endpoint constructed at line 56:
  ```python
  endpoint = f"{url}/api/agents/{agent_id}/beliefs"
  ```

`reasons/network.py`, `reasons/import_agent.py`, and `reasons/pg.py` also handle `agent_id`/`agent_name`, but only as node-ID and SQL-key strings (e.g. `f"{agent_name}:active"` in `import_agent._ensure_agent_nodes`); they never embed it in a URL. `reasons/hf.py` interpolates `repo_id` and `path_in_repo` into URLs with the same bare f-string style (lines ~71 and ~143), so it shares the same class of risk for a different path segment.

## Data flow into the URL

1. **CLI** — `reasons/cli.py:1029` and `reasons/cli.py:1047` pass `agent_id=args.agent_id` to `import_api` / `export_api`.
2. **API** — `reasons/api.py:2403` (`import_api`) and `reasons/api.py:2440` (`export_api`) accept `agent_id: str | None`, forward to `_resolve_config` (lines 2417–2418, 2452–2453), then hand the resolved value to `fetch_export` (line 2419) and the belief-push call (line 2469).
3. **Resolution** — `_resolve_config` (`mind_service.py:11`) falls back to `MIND_AGENT_ID` (line 16), asserts non-empty at line 22, and returns at line 29:
   ```python
   return url.rstrip("/"), agent_id, api_key
   ```
   Note that only `url` is sanitized (trailing slash). **`agent_id` is returned verbatim** — no `.strip()`, no `quote()`, no character-set validation.
4. **Interpolation** — `fetch_export` / `post_belief` drop the raw value straight into the path.

## Confirming the lack of quoting

The `find_url_quoting_helpers` search for `quote(|urlencode(|urllib.parse|quote_plus` returned **zero matches** across the tree. Every URL assembled from `agent_id` (and from `repo_id` in `hf.py`) is built by plain string formatting with no escaping layer.

## What unescaped values actually do

| `agent_id` | Constructed path | Observed effect |
|---|---|---|
| `foo bar` | `/api/agents/foo bar/export` | `urllib.request.Request` raises `InvalidURL` — spaces are not auto-encoded |
| `foo/bar` | `/api/agents/foo/bar/export` | Slash becomes an additional path segment; server 404s or routes to a different resource |
| `a?x=1` | `/api/agents/a?x=1/export` | `?` starts a query string; server sees path `/api/agents/a`, query `x=1/export` |
| `a#frag` | `.../agents/a#frag/export` | `#` starts a fragment; fragment is never sent; server sees `/api/agents/a` |
| `a%20b` | `.../agents/a%20b/export` | If a prior layer already percent-encoded, `%` is transmitted literally → double-encoding |
| id containing `\r\n` | — | `urllib` raises, but other HTTP libraries may interpret as header injection |

This is the classic "data crosses into a structural grammar without escaping" bug: the `agent_id` is user data, while `/{agent_id}/` is path syntax.

## Should callers quote?

No. The correct contract is:

- **`mind_service.fetch_export` and `mind_service.post_belief` quote internally** using `urllib.parse.quote(agent_id, safe="")` (note: `safe=""` — the default `safe="/"` would leave `/` unencoded and re-introduce the vulnerability). These are the functions that own URL construction, so they are the natural escaping boundary.
- **Not `api.py` / `cli.py`** — having every call site remember to `quote()` is a fragile contract that fails silently when someone forgets, and produces double-encoding if an id already arrives percent-encoded.
- **Not `_resolve_config`** — it is a resolution function (env lookup + non-empty check + `url.rstrip("/")`), not a serializer. Adding quoting there mixes concerns and would apply only to the callers that pass through it, which today is all of them but shouldn't be assumed forever.

The same fix pattern applies to `hf.download_network` (`f"{HF_BASE}/{parsed_id}/resolve/main/network.json"`) and `hf.upload_file` (`path_in_repo` in the URL), which are equally unescaped.

## Test evidence: the bug is invisible today

`tests/test_mind_service.py` asserts the fully formed URL at line 84 and line 122:

```python
assert req.full_url == "http://localhost/api/agents/agent-1/export"   # line 84
assert req.full_url == "http://localhost/api/agents/agent-1/beliefs"  # line 122
```

Every `agent_id` used across the test module is `"agent-1"` or `"a"` — URLs-safe, alphanumeric, hyphen only. Lines 56, 165, 177, 199, 212, 233 all pass `agent_id="agent-1"` or `"a"`. No test ever exercises `/`, `?`, `#`, spaces, unicode, or `%`. So the current suite cannot catch the injection.

## What's missing from the observations

- The full body of `post_belief` (only signature lines 52–53 and endpoint line 56 shown).
- The full body of `_resolve_config` — we see lines 16, 22, 29 but not whether line 29 also strips the `agent_id`.
- What exactly `api.py:2469` is calling (the function name after `resolved_url, resolved_agent_id, resolved_api_key,` is cut off).
- Whether the server's `/api/agents/{agent_id}/...` route decodes `%2F` back to `/` or rejects it — a server-contract question that determines whether quoting alone is sufficient.

## Topics to Explore

- [file] `reasons/mind_service.py` — Read the full `fetch_export` and `post_belief` bodies to confirm these are the only two interpolation sites for `agent_id` and to see how error responses wrap the id (line 46).
- [function] `reasons/mind_service.py:_resolve_config` — Decide whether it should also strip whitespace / reject control characters in `agent_id` before it ever reaches URL construction, and confirm line 29's exact return expression.
- [file] `reasons/hf.py` — Same bare-f-string pattern for `repo_id` and `path_in_repo` in `download_network` and `upload_file`; the fix should be applied consistently.
- [file] `tests/test_mind_service.py` — Add cases with `agent_id="a/b"`, `"a?b"`, `"a b"`, `"ünïcode"` asserting the exact `req.full_url` after quoting.
- [general] `server-encoded-slash-routing` — Confirm whether the API server treats `%2F` in the path as an encoded `/` or as a path separator, since that determines whether `quote(agent_id, safe="")` is a complete defence.

## Beliefs

- `unescaped-agent-id-in-path` — `fetch_export` (mind_service.py:34) and `post_belief` (mind_service.py:56) interpolate `agent_id` directly into the URL path with no escaping.
- `no-url-escaping-helpers` — The codebase contains zero uses of `urllib.parse.quote`, `quote_plus`, or `urlencode`; `find_url_quoting_helpers` returned 0 matches.
- `resolve-config-does-not-quote-agent-id` — `_resolve_config` only checks `agent_id` is non-empty (line 22) and normalizes `url.rstrip("/")` (line 29); it never sanitizes `agent_id`.
- `agent-id-requires-quoting-at-url-layer` — `agent_id` must be percent-encoded before interpolation, otherwise `/`, `?`, `#`, spaces, and CRLF alter path structure, query, fragment, or headers respectively.
- `url-quoting-belongs-in-mind-service` — Quoting should occur inside `fetch_export` / `post_belief` (the URL-construction boundary), not in `cli.py` / `api.py` callers, to avoid double-encoding and forgotten call sites.
- `tests-use-only-url-safe-agent-ids` — Every `agent_id` in `tests/test_mind_service.py` is alphanumeric or hyphen-only; no test covers reserved characters, so the injection is undetected by CI.
