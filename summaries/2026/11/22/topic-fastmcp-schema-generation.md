---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Inspect the live output of `mcp.get_tools()` to see the actual JSON Schema dialect (anyOf vs nullable) emitted for `belief_ids` under the installed `mcp`/pydantic versions.

## What the observations actually show

The captured data establishes the *source* of the schema but not its runtime output:

- `reasons/mcp_server.py` imports `FastMCP` via `from mcp.server.fastmcp import FastMCP` and constructs a module-level `mcp = FastMCP("reasons")` instance. The excerpt shown ends mid-file at line 320, having just started `propose_retract(...)`, but earlier in the same excerpt is the relevant decorator:
  - `@mcp.tool()` on `def propose_update(belief_ids: list[str] | None = None, ...)` — this is the parameter whose dialect is in question.
  - `propose_retract(target_id: str, ..., tags: list[str] | None = None)` uses the identical `list[str] | None` pattern and would be subject to the same serialization.
- The underlying API signatures are duplicated in `reasons/api.py` at lines **4060, 4178, 4231, 4349, 4459, 4622**, all using `belief_ids: list[str] | None = None` (PEP 604 union with `None`).
- `pyproject.toml` declares the optional dependency as `mcp = ["mcp>=1.0"]` — an open-ended lower bound with no upper cap, and **no direct pin on `pydantic`** (it arrives transitively through `mcp`). There is no lockfile in the observations.
- `locate_get_tools` returned `count: 0`. That is expected: `get_tools` is not defined anywhere in `reasons/`; it is a method on the `FastMCP` *instance*, so it lives in the `mcp` package, not this repo.
- `find_schema_dialect_mentions` and `inspect_mcp_schema_tests` both returned **0 matches** for `anyOf|nullable`. So the repo never asserts or overrides the schema dialect — it is whatever FastMCP/pydantic emit.

## What's missing to answer the question directly

The observations do **not** contain the actual `mcp.get_tools()` output, nor the resolved `mcp`/`pydantic` versions. To actually observe the dialect you would need:

1. The `Tool.inputSchema` (or `.parameters` / `.input_schema`, depending on version) payload for `propose_update`.
2. `pip show mcp pydantic` (or `pip freeze`) from the installed environment.
3. Which attribute the installed `FastMCP` version stores the JSON Schema on — the field name has churned across `mcp` releases.

## How to inspect it live

Run this snippet inside the environment that has `ftl-reasons[mcp]` installed:

```python
import json, asyncio
from reasons.mcp_server import mcp

# FastMCP exposes get_tools() in recent versions; older versions expose async list_tools().
try:
    tools = mcp.get_tools()                          # {name: Tool}
except AttributeError:
    tools = {t.name: t for t in asyncio.run(mcp.list_tools())}

tool = tools["propose_update"]
schema = (
    getattr(tool, "inputSchema", None)
    or getattr(tool, "parameters", None)
    or getattr(tool, "input_schema", None)
)
print(json.dumps(schema["properties"]["belief_ids"], indent=2))
```

Interpretation:

- `{"anyOf": [{"type":"array","items":{"type":"string"}}, {"type":"null"}], "default": null}` → **pydantic-v2 native JSON Schema (anyOf dialect)**. This is the expected result for `mcp>=1.0`, which pulls in pydantic v2.
- `{"type": ["array","null"], "items": {"type":"string"}, ...}` → JSON Schema 2020-12 tuple-form `type` list; rare from pydantic v2 but possible if a caller rewrites the schema.
- `{"type":"array","items":{"type":"string"}, "nullable": true, ...}` → **OpenAPI 3.0 nullable dialect**. This is only produced if FastMCP (or a caller) downgrades/normalizes the schema for an OpenAPI 3.0 consumer, or under a much older pydantic.

## Expected answer (to be confirmed live)

Because `mcp>=1.0` transitively installs pydantic v2, `list[str] | None` should serialize to **anyOf with a `null` branch**, not the OpenAPI `nullable: true` extension — for both `propose_update.belief_ids` and `propose_retract.tags`. But since `pyproject.toml` only caps the lower bound at `mcp>=1.0`, the actual dialect is a function of resolution time, which is precisely why the live call is the authoritative check.

## Topics to Explore

- [file] `reasons/mcp_server.py` — Every `@mcp.tool()` in the project is defined here; this is the single source of all MCP-exposed JSON Schemas.
- [function] `reasons/mcp_server.py:propose_update` — Tool whose `belief_ids: list[str] | None` parameter is the subject of the dialect question.
- [general] `mcp-pydantic-version-resolution` — The resolved `mcp`/`pydantic` pair determines whether the schema is anyOf or nullable; pin or inspect this before relying on the schema.
- [general] `fastmcp-tool-schema-serialization` — How `FastMCP` converts the Python signature into `Tool.inputSchema` (in pydantic v2 this routes through `model_json_schema()`).
- [file] `reasons/api.py` — Underlying signatures at lines 4060, 4178, 4231, 4349, 4459, 4622 mirror the MCP-exposed parameters and are the ultimate contract being re-typed.

## Beliefs

- `mcp-server-defines-propose-update` — `reasons/mcp_server.py` registers `propose_update` via `@mcp.tool()` with `belief_ids: list[str] | None = None`.
- `mcp-get-tools-is-instance-method-not-repo-symbol` — `mcp.get_tools()` resolves to a method on the module-level `FastMCP("reasons")` instance; no `get_tools` function exists in `reasons/`.
- `mcp-dependency-is-unpinned-beyond-lower-bound` — `pyproject.toml` declares `mcp>=1.0` with no upper bound and no direct pydantic pin, so the emitted schema dialect follows transitive resolution.
- `schema-dialect-is-unasserted-in-tests` — No test or source file references `anyOf` or `nullable`, so the dialect is entirely determined by FastMCP + pydantic.
- `propose-retract-tags-shares-belief-ids-dialect` — `propose_retract(tags: list[str] | None = None)` uses the same union-optional pattern and will serialize identically to `belief_ids`.
