---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/mind_service.py

# `reasons/mind_service.py` — HTTP client for agentic-mind-service

## Purpose

This module is a **thin HTTP client** wrapping two operations against an external "agentic-mind-service": downloading an agent's full belief network (`fetch_export`) and pushing a single belief up to it (`push_belief`). It owns nothing else — no storage, no state, no domain logic. Its only responsibility is: given a base URL, agent ID, and API key, make an authenticated request, translate HTTP failures into Python-facing errors, and return decoded JSON.

It is the low-level transport that higher-level entry points (e.g. `reasons/api.py`) use when syncing beliefs with a remote mind-service.

## Key Components

### `_resolve_config(url, agent_id, api_key) -> tuple[str, str, str]`

A configuration resolver with a **cascade priority**: explicit argument → environment variable → missing.

| Field | Explicit arg | Env var |
|---|---|---|
| `url` | `url` param | `MIND_SERVICE_URL` |
| `agent_id` | `agent_id` param | `MIND_AGENT_ID` |
| `api_key` | `api_key` param | `MIND_API_KEY` |

- Returns `(url.rstrip("/"), agent_id, api_key)` — the URL is normalized so no double-slash ever appears when joined with `/api/...`.
- **Collects all missing fields** into a `missing` list, then raises a single `RuntimeError` describing every one (e.g. `"Missing required config: url (--url or MIND_SERVICE_URL), agent-id (...)"`). This is a fail-fast, tell-the-user-everything pattern.
- Underscore prefix marks it as module-private convention.

### `fetch_export(url, agent_id, api_key) -> str`

- `GET {url}/api/agents/{agent_id}/export` with header `Authorization: Bearer {api_key}`.
- 60-second timeout.
- Returns the response body **as a UTF-8 string** (expected to be full network JSON — the caller parses it, not this function).
- Special-cases `401` → `RuntimeError("Authentication failed. Check your MIND_API_KEY.")` and `404` → `RuntimeError(f"Agent not found: {agent_id}")`, both chained from the original via `from e`.

### `push_belief(url, agent_id, api_key, node_id, text, sl="", source="") -> dict`

- `POST {url}/api/agents/{agent_id}/beliefs`.
- Body is built conditionally: `{"node_id": ..., "text": ...}` plus `"sl"` and `"source"` **only if non-empty** — empty strings are omitted from the JSON entirely, not sent as `""`.
- 30-second timeout, `Content-Type: application/json`.
- Returns the parsed response via `json.loads(...)`.
- Same `401 → RuntimeError` translation; any other `HTTPError` propagates.

## Patterns

1. **Config cascade (params > env)** — the classic "argparse overrides environment" idiom. Keeps the function testable (pass literals) while supporting CLI-driven deployment via env.
2. **Stdlib-only HTTP client** — uses `urllib.request.Request`/`urlopen` rather than `requests`. Zero third-party dependency, ships everywhere Python does.
3. **Error translation layer** — raw `HTTPError` is converted into human-readable `RuntimeError`s only for well-understood status codes (401, 404), and left unmapped otherwise. This keeps callers free of `urllib.error` imports.
4. **Fail-fast aggregation** — `_resolve_config` gathers all missing config in one shot instead of raising on the first one.
5. **Conditional body composition** — optional payload fields are omitted rather than blanked, so the service receives a minimal, semantic body.
6. **Duplicate package layout** — `reasons/mind_service.py` and `reasons_lib/mind_service.py` exist as parallel copies; this file is the `reasons/` variant.

## Dependencies

**Imports (all stdlib):**
- `json` — encode request bodies, decode responses
- `os` — read `MIND_*` environment variables
- `urllib.error.HTTPError` — distinguish HTTP failures from transport failures
- `urllib.request.Request`, `urlopen` — build and dispatch requests

**Imported by:**
- `reasons/api.py` — the public API surface, presumably exposing CLI/command entry points
- `reasons_lib/mind_service.py` — the mirrored package version (itself importing this or vice versa)
- `tests/test_mind_service.py` — unit tests (likely patching `urlopen`)

## Flow

```
caller
  │
  ├─ _resolve_config(...)         # validate, strip trailing slash
  │
  ├─ fetch_export(url, ...)       # GET  -> decode -> str
  │     Request(endpoint, headers={Authorization: Bearer ...})
  │     urlopen(req, timeout=60)
  │     resp.read().decode("utf-8")
  │
  └─ push_belief(url, ...)        # POST -> parse -> dict
        body = {node_id, text, [sl], [source]}
        data = json.dumps(body).encode("utf-8")
        Request(endpoint, data, headers, method="POST")
        urlopen(req, timeout=30)
        json.loads(resp.read().decode("utf-8"))
```

There is no retry, no batching, no pagination — one call, one response. Orchestration lives entirely with the caller (see `reasons/api.py`).

## Invariants

- `url`, `agent_id`, and `api_key` must all be non-empty after resolution; otherwise `_resolve_config` raises before any request is made.
- The base URL returned by `_resolve_config` **never ends in `/`** — callers can safely do `f"{url}/api/..."`.
- Every request carries `Authorization: Bearer {api_key}`.
- Every request has a finite timeout (60s for export, 30s for push) — no unbounded blocking.
- `push_belief` never sends `sl` or `source` when they are falsy strings. The server sees them absent rather than empty.
- `fetch_export` returns a raw string; the caller is responsible for `json.loads`. `push_belief` returns a parsed dict.
- HTTP 401 and 404 are always converted to `RuntimeError`; other status codes are propagated as `HTTPError`.

## Error Handling

| Scenario | Behavior |
|---|---|
| Missing config | `RuntimeError` from `_resolve_config`, listing all missing fields |
| 401 from either endpoint | `RuntimeError("Authentication failed. Check your MIND_API_KEY.")` — chained |
| 404 from `fetch_export` | `RuntimeError(f"Agent not found: {agent_id}")` — chained |
| 404 from `push_belief` | **Not** special-cased — propagates as `HTTPError` |
| Other 4xx/5xx | Propagates as `HTTPError` (only `from e`-chained for the handled cases) |
| Network / timeout / DNS | Propagates as whatever `urlopen` raises (`URLError`, `TimeoutError`, etc.) — **not** wrapped |
| Malformed JSON in response | Propagates `json.JSONDecodeError` |

Note the asymmetry: `push_belief` handles only 401, while `fetch_export` handles 401 and 404. A "not found" on `push_belief` will surface as a raw `HTTPError` — worth knowing if you're diagnosing a caller.

---

## Topics to Explore

- [file] `reasons/api.py` — See how these two functions are wired into CLI/API commands, and which callers rely on the `RuntimeError` contract vs. catching `HTTPError`.
- [file] `reasons/pg.py` — Understand the local persistence layer that these synced beliefs ultimately flow into or out of.
- [file] `tests/test_mind_service.py` — Learn whether `urlopen` is monkey-patched and which HTTP status codes are actually exercised.
- [general] `MIND_SERVICE-ENV-CONTRACT` — Document the `MIND_SERVICE_URL` / `MIND_AGENT_ID` / `MIND_API_KEY` variables and who sets them (CI, shell profile, docs).
- [file] `reasons_lib/mind_service.py` — Confirm whether the mirrored package diverges from this one — divergence could cause subtle bugs across import paths.

## Beliefs

- `mind-service-client-is-pure-stdlib` — The module imports only from `json`, `os`, and `urllib.*`; no third-party HTTP library is required.
- `config-resolution-prefers-explicit-params` — `_resolve_config` gives explicit function arguments precedence over `MIND_*` environment variables.
- `mind-service-missing-config-is-aggregated` — `_resolve_config` collects all missing fields and raises a single `RuntimeError` rather than failing on the first.
- `resolved-base-url-has-no-trailing-slash` — `_resolve_config` always strips trailing `/` from the URL before returning it.
- `push-belief-omits-empty-optional-fields` — `push_belief` excludes `sl` and `source` from the request body when they are empty strings.
- `fetch-export-maps-401-and-404-to-runtimeerror` — `fetch_export` converts HTTP 401 and 404 into `RuntimeError`, chained from the original `HTTPError`.
