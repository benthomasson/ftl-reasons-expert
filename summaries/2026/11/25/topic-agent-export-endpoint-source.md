---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Search the repo for a web/router module (FastAPI/Flask/ASGI) that actually registers `/api/agents/{id}/export` and `/beliefs`; the current searches found none

# Where Are the `/api/agents/{id}/export` and `/beliefs` Routes?

## Short answer

**They are not registered anywhere in this repo.** The searches are correct — there is no FastAPI/Flask/ASGI/Starlette router, and no `@app.get(...)`-style decorator. The two paths appear only as **URLs constructed by an HTTP *client*** in `reasons/mind_service.py`, not as route definitions served by this codebase.

## Evidence

### 1. The paths are client-side constructions, not route registrations

In `reasons/mind_service.py`:

- **Lines 33–34**:
  ```python
  """GET /api/agents/{agent_id}/export — download full network JSON."""
  endpoint = f"{url}/api/agents/{agent_id}/export"
  ```
- **Lines 55–56**:
  ```python
  """POST /api/agents/{agent_id}/beliefs — push one belief."""
  endpoint = f"{url}/api/agents/{agent_id}/beliefs"
  ```

The `"""..."""` strings are *docstrings describing the HTTP call being made*, and `endpoint = f"{url}/..."` builds a URL against a base `url` supplied from outside (config/argument). Nothing here *listens* on those paths.

`tests/test_mind_service.py:84` and `:122` only assert the **client assembled the right URL**:
```python
assert req.full_url == "http://localhost/api/agents/agent-1/export"
assert req.full_url == "http://localhost/api/agents/agent-1/beliefs"
```
That is a client contract test — it verifies the client emits the expected request, not that a server registers a handler.

### 2. No web framework is used anywhere

- `find_web_framework_usage` (`FastAPI|Flask|APIRouter|Blueprint|Starlette|ASGI`) → **0 matches**.
- `find_route_decorators` (`@(app|router|api)\.(get|post|put|delete|patch|route)`) → **0 matches**.

Even though `api` appears all over `reasons/api.py`, it's a Python *module* named `api`, not a router object being decorated.

### 3. `reasons/api.py` is a functional library API, not an HTTP API

The module docstring at the top of `reasons/api.py` states the intent:

> "This module provides standalone functions that any Python caller can use (CLI, LangGraph tools, scripts) without dealing with Storage lifecycle or argparse. Each function opens the database, operates, saves, and closes. All functions return dicts suitable for JSON serialization."

That is a hand-rolled, function-per-operation API:
- `init_db`, `add_node`, `retract_node`, `assert_node`, `propagate`, `get_status`, `show_node`, `explain_node`, `export_network`, `import_beliefs`, `import_agent_beliefs`, …
- `reasons/api.py:1888` `def import_beliefs(...)`
- `reasons/api.py:1926` `def import_agent_beliefs(...)`, which at line 1952 accepts `.json` "network.json (JSON export)" — i.e., the exact payload the `/export` endpoint would return.

`reasons_lib/api.py` is just `from reasons.api import *` — again, no web layer.

### 4. The `beliefs` grep hit 1133 times — none are routes

`find_beliefs_route_references` returned 1,133 matches for the literal string `beliefs` in `reasons/api.py` and elsewhere. Every one is **domain vocabulary** (namespace beliefs, `import_beliefs`, `get_belief_set`, belief-set iteration, "Agent 'X' beliefs are trusted" premise text, `total_beliefs` / `in_beliefs` counters), never a `@router.post("/beliefs")` decorator.

## Interpretation

There are two separable concerns:

| Concern | Where it lives |
| --- | --- |
| **Client** that calls `/api/agents/{id}/export` and `/beliefs` | `reasons/mind_service.py` (lines 33–34, 55–56) |
| **Server** that registers those routes | **Not present in this repo** |
| **Domain logic** an external server would wrap | `reasons/api.py` — e.g., `export_network`, `import_beliefs` (line 1888), `import_agent_beliefs` (line 1926) |

So the picture is: `mind_service.py` is a **client** to *some* web service (base `url` is injected), and the domain functions in `reasons/api.py` are what that external service most plausibly calls under the hood for `/export` (→ `export_network`) and `/beliefs` (→ `add_node` / import routines). The actual FastAPI/Flask host is **not in this snapshot**.

## What the observations do *not* tell us (and what to check next)

- The concrete value/argument sanity of `url` in `mind_service.py` — where is it configured? (env var? CLI arg? caller?)
- Whether the server lives in a **sibling package**, a **deployment repo**, or a **Docker build context** — none is visible here.
- Whether `pyproject.toml`/`requirements.txt` lists `fastapi`, `flask`, `starlette`, or a similar dependency (zero matches on the pattern search suggests no, but the manifest itself wasn't read).
- Whether `mind_service.py` targets a third-party "mind service" product that is deliberately separate from this engine.

## Topics to Explore

- [file] `reasons/mind_service.py` — The client that constructs both target URLs; look at how the `url` base is supplied and which class/function owns lines 33–56.
- [function] `reasons/api.py:export_network` — The domain function whose return value most closely matches the documented body of `GET /api/agents/{id}/export`.
- [function] `reasons/api.py:import_agent_beliefs` — Accepts `beliefs.md` or `network.json`; this is the server-side handler body for `POST .../beliefs` in waiting.
- [file] `tests/test_mind_service.py` — Confirms the contract is client-side only (URL construction, not routing) at lines 84 and 122.
- [general] `mind-service-server-location` — The actual FastAPI/Flask app that hosts these routes is not in this snapshot; search sibling packages, deployment configs, and compose files.

## Beliefs

- `no-web-framework-in-repo` — The codebase contains zero references to FastAPI, Flask, Starlette, ASGI, APIRouter, or Blueprint.
- `mind-service-is-a-client` — `reasons/mind_service.py` only *constructs* `/api/agents/{id}/export` and `/api/agents/{id}/beliefs` URLs; it does not register them.
- `reasons-api-is-functional-not-http` — `reasons/api.py` exposes plain Python functions returning dicts; its docstring explicitly targets CLI, LangGraph tools, and scripts, not HTTP.
- `reasons-lib-api-re-exports-reasons-api` — `reasons_lib/api.py` is a single wildcard re-export of `reasons.api`.
- `beliefs-grep-hits-are-domain-vocabulary` — All 1,133 `beliefs` occurrences are namespace/domain logic; no route decorator matches the pattern.
- `export-payload-format-is-shared` — `reasons/api.py:export_network` produces the JSON that `import_agent_beliefs` (line ~1926) accepts at line 1952 as `network.json`, matching the documented `/export` response.
