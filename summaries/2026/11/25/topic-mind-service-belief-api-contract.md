---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Confirm server expectations for endpoint path, auth, payload fields, and response shape.

## Server expectations: what the observations actually show

**Short answer: there is no HTTP server layer in the observed code.** The four search patterns that would reveal endpoint paths, auth, payloads, and response shape all returned zero matches:

| Search | Pattern searched | Matches |
|---|---|---|
| HTTP auth | `Authorization\|Bearer\|API_KEY\|api_key\|auth\|token` | **0** |
| Routes/framework | `@app.get/post\|@router.*\|APIRouter\|FastAPI\|Flask\|add_api_route\|route(` | **0** |
| Response shaping | `response_model\|JSONResponse\|return \{\|status_code\|HTTPException` | **0** |

So we cannot confirm endpoint paths, auth expectation, request payload schema, or HTTP response shape from these observations — because the code that was read is a **Python function library**, not a web service.

### What is actually present

**`reasons/api.py` (5,886 lines, first 200 shown)** is explicitly a "Functional Python API" per its own docstring (lines 1–8):

> "This module provides standalone functions that any Python caller can use (CLI, LangGraph tools, scripts) without dealing with Storage lifecycle or argparse. Each function opens the database, operates, saves, and closes. All functions return dicts suitable for JSON serialization."

The observable contract is **Python function signatures + dict returns**, not HTTP:

- `init_db(db_path=DEFAULT_DB, force=False, project_name="", pg_conninfo=None, domain_id=None) -> dict` (~line 195) — returns a dict with at least a `"created"` key (per `tests/test_api.py::TestInitDb::test_creates_db`).
- Other API surfaces (`add_node`, `retract_node`, `assert_node`, `propagate`, `get_status`, `show_node`, `explain_node`, `add_nogood`, `get_belief_set`) are only visible via their call sites in `tests/test_api.py`, not their definitions in the 200 lines returned.
- Return shapes inferred from tests (not from server code): e.g. `add_node` returns `{"node_id", "truth_value", "type"}`, `retract_node` returns `{"changed": [...]}`, `show_node` returns `{"id", "text", "source", "justifications", "dependents"}`, `explain_node` returns `{"steps": [{"node", "reason"}, ...]}`, `add_nogood` returns `{"nogood_id", "nodes", "changed"}`, `get_status` returns `{"nodes", "total", "in_count"}`.

**`reasons_lib/api.py` (3 lines)** is only a backward-compat shim:
```python
from reasons.api import *  # noqa: F401,F403
```
No routes, no auth — just re-export.

**Access control that *does* exist is not HTTP auth.** `_is_visible(node, visible_to)` at `reasons/api.py` ~line 33 implements **tag-based visibility**, not authentication:

> "A node is visible if its access_tags are all contained in visible_to. Nodes with no access_tags are always visible."

`_filter_by_tags` (~line 69) provides tag-based filtering for `topic:`, `status:`, and prefixed `tag:` values. This is a data-access filter, not a server auth scheme (no tokens, no headers, no roles).

### Why the question cannot be answered yet

To confirm "server expectations for endpoint path, auth, payload fields, and response shape," these would need to exist in the codebase:

1. A web framework entry point (e.g., a `main.py`, `server.py`, `app.py`, or `routes/` package using FastAPI/Flask/Starlette).
2. Route decorators or an `APIRouter` with path strings, HTTP verbs, and path/query parameter declarations.
3. Pydantic request models (payload field names, types, required vs optional).
4. `response_model=` / `JSONResponse(...)` / `HTTPException(...)` to establish HTTP status semantics.
5. Auth middleware or dependency injection (`Depends(...)`, an `Authorization` header parser, an API-key gate).

None of these were in the observations. The only JSON-serializable contract visible is the **function-level dict contract** described in the docstring and asserted in `tests/test_api.py`.

---

## Topics to Explore

- [file] `reasons/api.py` — The remaining ~5,686 unobserved lines likely contain the actual signatures, docstrings, and return-dict keys for `add_node`, `show_node`, `explain_node`, etc.; that dict shape is the effective "payload/response schema" for callers even without HTTP.
- [file] `reasons/pg.py` — Referenced by `_pg_dispatch` (~line 180) as `PgApi(pg_conninfo, domain_id)`; its method signatures mirror the functional API, so if any RPC-style service exists it is likely here.
- [file] `reasons/pubsub.py` — Called from `_with_network.__exit__` via `pubsub.has_subscribers`, `compute_changes`, `publish`; this is the only observed producer/consumer boundary and could hint at event-driven "endpoints."
- [function] `reasons/api.py:_pg_dispatch` — Dispatches to `PgApi` by `method_name`, `domain_id`, `pg_conninfo`; this indirection is where any backend-specific contract (Postgres vs SQLite) would be defined.
- [general] `http-server-entrypoint` — Search the repo for `FastAPI(`, `Flask(`, `APIRouter(`, `uvicorn.run`, or a `Dockerfile`/`Procfile` to locate (or rule out) an actual HTTP server that the API is wrapped in.

## Beliefs

- `reasons-api-has-no-http-layer` — `reasons/api.py` is a functional Python library, not an HTTP server; the module docstring states it is meant for "CLI, LangGraph tools, scripts."
- `reasons-lib-api-is-a-shim` — `reasons_lib/api.py` contains only `from reasons.api import *` for backwards compatibility; it defines no behavior of its own.
- `no-auth-keyword-matches` — The codebase contains zero occurrences of `Authorization|Bearer|API_KEY|api_key|auth|token` in the searched files.
- `no-route-declarations` — The codebase contains zero FastAPI/Flask route decorators, `APIRouter`, or `add_api_route` calls in the searched files.
- `api-returns-json-serializable-dicts` — Every functional API in `reasons/api.py` returns a `dict` "suitable for JSON serialization" per the module docstring (lines 5–7), and tests assert specific keys (`node_id`, `truth_value`, `type`, `changed`, `steps`, etc.).
- `access-control-is-tag-based-not-token-based` — `_is_visible` (line ~33) enforces visibility via `access_tags` membership in a caller-supplied `visible_to` list, not via authenticated identity.
