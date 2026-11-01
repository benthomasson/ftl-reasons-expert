---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/mind_service.py

# `reasons/mind_service.py`

## Purpose

This file is a **thin HTTP client** that bridges the `ftl-reasons` belief network to an external **agentic-mind-service** REST API. Its sole responsibility is to let a local reasoning agent sync its beliefs: **pull** a full network export (GET) or **push** individual beliefs (POST). It owns no local state, no storage, and no business logic beyond serializing/deserializing JSON and translating HTTP status codes into actionable `RuntimeError`s.

Think of it as the "dumb pipe" in a larger sync pipeline — `reasons/api.py` decides *what* to sync and *when*; this file handles *how* to talk to the remote service over HTTP.

## Key Components

### `_resolve_config(url, agent_id, api_key) -> tuple[str, str, str]`

**Contract:** Resolves the three required connection parameters using a strict precedence: explicit argument → environment variable → raise.

- **Returns:** `(url, agent_id, api_key)` — a 3-tuple of non-empty strings. `url` is right-stripped of trailing slashes.
- **Raises:** `RuntimeError` listing *all* missing values at once (not fail-fast on the first one).
- **Env vars read:** `MIND_SERVICE_URL`, `MIND_AGENT_ID`, `MIND_API_KEY`.
- **Not exported** (underscore prefix) — it's an internal helper. Callers in `api.py` will pass explicit values or rely on env vars.

### `fetch_export(url, agent_id, api_key) -> str`

**Contract:** Performs `GET {url}/api/agents/{agent_id}/export` and returns the raw response body as a UTF-8 string (expected to be a JSON document representing the full belief network).

- **Timeout:** 60 seconds (longer, since it's a full export — could be large).
- **Auth header:** `Authorization: Bearer {api_key}`.
- **Raises:** `RuntimeError` on 401 (auth) or 404 (agent not found); re-raises other `HTTPError`s untouched.

### `push_belief(url, agent_id, api_key, node_id, text, sl="", source="") -> dict`

**Contract:** Performs `POST {url}/api/agents/{agent_id}/beliefs` with a JSON body and returns the parsed JSON response as a `dict`.

- **Body schema:** `{"node_id": ..., "text": ...}` always; `sl` and `source` are included *only if* non-empty (avoids sending `""` fields).
- **Timeout:** 30 seconds (shorter, since it's a single-record write).
- **Auth header:** same Bearer token pattern.
- **Raises:** `RuntimeError` on 401; re-raises other `HTTPError`s.
- **Notable:** 404 is *not* special-cased here (unlike `fetch_export`). A 404 on push would bubble up as a raw `HTTPError`.

## Patterns

| Pattern | Where | Notes |
|---|---|---|
| **Config resolution ladder** | `_resolve_config` | `param → env → error`. Standard 12-factor "config via env" idiom. Collects *all* missing keys before raising, so a misconfigured deployment surfaces every problem at once. |
| **Bearer-token auth** | both HTTP functions | Inline `Authorization` header; no separate auth middleware. Kept deliberately simple since it's a 2-endpoint client. |
| **stdlib-only HTTP** | `urllib.request` | No `requests` dependency. This is a deliberate choice to keep the `reasons` package lightweight — the client is small enough that the stdlib is sufficient. |
| **Narrow exception mapping** | `except HTTPError` blocks | Only 401 and (in `fetch_export`) 404 are translated. Everything else propagates as `HTTPError`, preserving the raw status/headers for the caller to inspect. |
| **Duplicate module** | `reasons/` vs `reasons_lib/` | `reasons_lib/mind_service.py` is a parallel copy. The repo appears to maintain two package trees (likely a build/packaging split). Changes must be kept in sync. |

## Dependencies

### Imports (this file → other modules)

| Import | Why |
|---|---|
| `json` | Serialize push body, deserialize push response |
| `os` | Read `MIND_SERVICE_URL`, `MIND_AGENT_ID`, `MIND_API_KEY` |
| `urllib.error.HTTPError` | Catch and classify HTTP failure codes |
| `urllib.request.Request, urlopen` | Build and execute HTTP requests |

Notably, it imports **nothing** from other `reasons` modules. It's fully self-contained.

### Imported by (other modules → this file)

| Consumer | Likely usage |
|---|---|
| `reasons/api.py` | The main API layer calls `fetch_export` / `push_belief` when a user runs sync commands |
| `reasons_lib/mind_service.py` | Mirror copy in the `reasons_lib` package (build artifact or parallel distribution) |
| `tests/test_mind_service.py` | Unit/integration tests for config resolution and HTTP interactions (likely with `urlopen` mocked) |

## Flow

```
Caller (api.py)
│
├─► _resolve_config(url?, agent_id?, api_key?)
│     │  1. Try each arg; fall back to os.environ
│     │  2. Collect missing into a list
│     │  3. If any missing → raise RuntimeError("Missing required config: ...")
│     │  4. Return (url.rstrip("/"), agent_id, api_key)
│
├─► fetch_export(url, agent_id, api_key)
│     │  1. Build URL: {url}/api/agents/{agent_id}/export
│     │  2. Request with Bearer header
│     │  3. urlopen(timeout=60)
│     │  4. resp.read().decode("utf-8") → return raw string
│     │  5. On HTTPError: 401→RuntimeError, 404→RuntimeError, else re-raise
│
└─► push_belief(url, agent_id, api_key, node_id, text, sl?, source?)
      │  1. Build JSON body (conditionally include sl/source)
      │  2. Request POST with Bearer + Content-Type headers
      │  3. urlopen(timeout=30)
      │  4. json.loads(resp.read().decode("utf-8")) → return dict
      │  5. On HTTPError: 401→RuntimeError, else re-raise
```

There is no retry logic, no connection pooling, and no async. Each call opens a fresh TCP connection, performs one request, and closes.

## Invariants

1. **Config completeness is enforced at the boundary.** `_resolve_config` guarantees that every downstream function receives three non-empty strings. No function in this module should ever see an empty `url`, `agent_id`, or `api_key`.
2. **`url` is always slash-normalized.** `rstrip("/")` prevents double-slash bugs in URL construction (`{url}/api/...`).
3. **`sl` and `source` are optional in the push body.** They are omitted from the JSON payload entirely when empty, rather than sent as `""`. The server must treat their *absence* as "not provided," not as an empty string.
4. **The export response is returned as a raw string, not parsed JSON.** The caller (`api.py`) is responsible for `json.loads()`. This keeps `fetch_export` format-agnostic and makes it easy to swap to a different content type later.
5. **`node_id` is always present in the push body.** Unlike `sl`/`source`, there is no conditional — it's a required field of the API contract.

## Error Handling

The error strategy is **selective translation with pass-through**:

| HTTP Status | `fetch_export` | `push_belief` |
|---|---|---|
| 401 | `RuntimeError("Authentication failed. Check your MIND_API_KEY.")` | Same |
| 404 | `RuntimeError(f"Agent not found: {agent_id}")` | **Not caught** — propagates as raw `HTTPError` |
| 403, 500, 503, etc. | Re-raised as `HTTPError` | Re-raised as `HTTPError` |
| Network timeout (`URLError`) | Not caught — propagates | Not caught — propagates |

**Practical implications for the caller:**

- Callers in `api.py` must catch `RuntimeError` for the "expected" failure modes (bad key, unknown agent) and `HTTPError` / `URLError` for "unexpected" ones.
- The asymmetric 404 handling (caught in export, not in push) is worth noting. If the agent ID is wrong, a push will produce a raw `HTTPError(404)` rather than a friendly message. This may be intentional (a 404 on POST is less common since the endpoint is the same) or an oversight.
- There is **no retry** logic. A transient 503 or network blip will immediately surface to the caller. If you need resilience, that belongs in `api.py` or a wrapper.
- All `RuntimeError`s are raised with `from e`, preserving the original `HTTPError` in the `__cause__` chain for debugging.

---

## Topics to Explore

- [file] `reasons/api.py` — The primary consumer; shows how `fetch_export` and `push_belief` are wired into user-facing CLI commands and what post-processing (JSON parsing, storage) happens on the result
- [file] `reasons_lib/mind_service.py` — The parallel copy in the `reasons_lib` package; confirm it's byte-identical and understand the build/packaging mechanism that keeps them in sync
- [file] `tests/test_mind_service.py` — Reveals the mocking strategy for `urlopen`, the expected env-var interactions, and edge cases around `_resolve_config`
- [file] `reasons/storage.py` — Where the exported JSON actually gets persisted; understanding this clarifies why `fetch_export` returns a raw string rather than a parsed object
- [general] `agentic-mind-service-api-contract` — The remote REST API this client targets; the server-side spec would clarify the full belief schema, pagination (if any), and the exact meaning of `sl` and `source` fields

## Beliefs

- `mind-service-uses-stdlib-only` — `mind_service.py` depends exclusively on Python stdlib (`urllib`, `json`, `os`) for HTTP; it has zero third-party HTTP dependencies, making it safe to ship in minimal environments
- `fetch-export-returns-raw-string` — `fetch_export` returns the response body as an unparsed UTF-8 string, delegating JSON deserialization to the caller (`api.py`), which enables future content-type changes without modifying this module
- `push-belief-omits-empty-optionals` — `push_belief` conditionally excludes `sl` and `source` from the JSON body when they are empty strings, so the server must distinguish "field absent" from "field present but empty"
- `config-resolution-collects-all-missing` — `_resolve_config` accumulates *all* missing config values into a single `RuntimeError` message rather than failing on the first absent variable, so a misconfigured deployment reports every gap in one shot
- `reasons-mind-service-duplicated-in-reasons-lib` — `reasons/mind_service.py` and `reasons_lib/mind_service.py` are parallel copies that must be kept in sync; a change to one without the other will cause behavioral divergence between the two package trees
