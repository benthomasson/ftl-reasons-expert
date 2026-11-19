---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How FastMCP derives tool schemas from docstrings and type hints, and what happens when a signature uses `list[str] | None` defaults.

# How FastMCP derives tool schemas from docstrings and type hints

## Where the derivation happens

All tool registration in this repo goes through one `FastMCP` instance:

- `reasons/mcp_server.py:16` — `from mcp.server.fastmcp import FastMCP`
- `reasons/mcp_server.py:25` — `mcp = FastMCP("reasons")`
- Ten `@mcp.tool()` decorators at lines 57, 74, 87, 100, 127, 145, 159, 175, 203, 222.

Each `@mcp.tool()` call wraps the decorated Python function. At registration time FastMCP inspects the function's `inspect.signature()` **and** its `__doc__` to build:

1. A **tool description** (from the docstring).
2. A **JSON input schema** (from the signature, type hints, and defaults).
3. A mapping from schema properties back to function parameters so calls can be dispatched.

The runtime dependency lives in `pyproject.toml` under the optional `mcp` extra (`mcp>=1.0`). When it isn't installed, `reasons/mcp_server.py` raises a targeted `ImportError` rather than failing obscurely.

## Type hints → JSON Schema

FastMCP (via pydantic-style conversion) maps Python annotations to JSON Schema properties:

| Python annotation                | JSON Schema property        | Notes                              |
|----------------------------------|-----------------------------|------------------------------------|
| `query: str` (no default)        | `{"type": "string"}`        | Required                           |
| `depth: int = 1`                 | `{"type": "integer", "default": 1}` | Optional                    |
| `include_out: bool = False`      | `{"type": "boolean", "default": false}` | Optional                |
| `node_ids: list[str]`            | `{"type": "array", "items": {"type": "string"}}` | Required        |
| `belief_ids: list[str] \| None = None` | nullable array           | Optional (see below)               |

You can see the contrast directly in the source:

- `reasons/mcp_server.py:175` — `nogood(node_ids: list[str])` has **no default**, so `node_ids` is required in the generated schema.
- `reasons/mcp_server.py:203` — `propose_update(belief_ids: list[str] | None = None, ...)` has a `None` default, so `belief_ids` is optional.

## Docstrings → tool + parameter descriptions

Every tool uses Google-style docstrings with an explicit `Args:` block. For example `search` at `reasons/mcp_server.py:57`:

```python
@mcp.tool()
def search(query: str, output_format: str = "markdown", depth: int = 1,
           include_out: bool = False, limit: int = 0, offset: int = 0) -> str:
    """Search beliefs by text with neighbor expansion.

    Args:
        query: Search terms (matches all terms in any order)
        output_format: Output format — "markdown", "json", "minimal", or "names-only"
        ...
    """
```

FastMCP's parser does two things:

1. The **first non-empty line** of the docstring becomes the tool's `description` field in the MCP manifest.
2. Each `name: text` entry under `Args:` becomes the `description` on the corresponding parameter inside `inputSchema.properties[name]`.

That's why every tool wraps a free-form summary in the docstring header and repeats parameter help under `Args:` — it's the only channel FastMCP uses to populate `description` values in the schema. (The `reasons/ask.py:361–368` snippet shows a *consumer* reading those fields back out: `pinfo.get("description", pname)` and `tool["description"].split("\n")[0]`.)

## What happens with `list[str] | None = None`

Take the one tool in `reasons/mcp_server.py` that uses that pattern, `propose_update` (line 203):

```python
@mcp.tool()
def propose_update(belief_ids: list[str] | None = None, model: str = "claude",
                   stale_only: bool = False, namespace: str = "",
                   sample: int = 0, proposer: str = "llm",
                   store: bool = True) -> str:
    ...
    result = api.propose_update(
        belief_ids=belief_ids or None,
        ...
    )
```

Three things follow from the signature:

1. **Schema shape.** `list[str] | None` is a `types.UnionType` of `list[str]` and `NoneType`. Pydantic v2 (which FastMCP uses under the hood) renders this as an `anyOf` with a `null` branch, roughly:

   ```json
   "belief_ids": {
     "anyOf": [
       {"type": "array", "items": {"type": "string"}},
       {"type": "null"}
     ],
     "default": null,
     "description": "Specific belief IDs to evaluate (default: all IN beliefs)"
   }
   ```

   Older FastMCP/pydantic combinations can emit `{"type": "array", "items": {"type": "string"}, "nullable": true}` or add `"default": null`. The exact dialect depends on the installed `mcp` and `pydantic` versions — that's the one detail we can't pin down from the observations alone (see below).

2. **Optionality.** Because `belief_ids` has a default of `None`, it is **not** added to the schema's `required` list. An MCP client can omit it entirely, or send JSON `null` — both dispatch to the Python function with `belief_ids=None`.

3. **Handler-side normalization.** FastMCP does *not* rewrite `None` into `[]`. The handler itself compensates: `belief_ids=belief_ids or None` (line ~230) collapses `[]` and `None` into a single sentinel before calling `api.propose_update`, matching the underlying API signature at `reasons/api.py:4178` (`belief_ids: list[str] | None = None`). This is the well-known pattern in this file — every `list[str] | None` parameter (there are 44 such signatures across `reasons/api.py`, found at lines 49, 288, 782, 870, 1032, 1054, 1079, 1550, 1814, 2342–2346, 2544, 2610, 2869, 2919, 3015, 3400, 3570, 3629, 3672, 3957, 4060–4067, 4178, 4231–4235, 4349, 4459, 4622) is documented as "default: all X" or similar — meaning `None` and empty list are deliberately treated the same.

## What is *not* observable from this snapshot

The actual `mcp.server.fastmcp` package source is not in the observations, so I can't show the exact code that walks `inspect.signature`. Likewise `tests/test_mcp_server.py` (up to line 300 shown) calls `mcp_server.search(...)`, `mcp_server.show(...)` etc. directly — it exercises handler bodies, **not** the generated JSON schema. Confirming the `anyOf` vs `nullable` output for `list[str] | None` requires calling `mcp.get_tools()` (or inspecting the MCP manifest) against the installed `mcp` version.

## Topics to Explore

- [file] `reasons/mcp_server.py` — The ten `@mcp.tool()`-decorated functions whose signatures and docstrings drive every generated schema; `propose_update` (line 203) is the canonical `list[str] | None` example.
- [function] `reasons/mcp_server.py:propose_update` — Compare its signature against the underlying `api.propose_update` (`reasons/api.py:4178`) to see how FastMCP's schema mirrors the API and where `None` vs `[]` is normalized.
- [file] `reasons/api.py` — Contains 44 `list[str] | None = None` signatures that define the convention the MCP layer is mirroring.
- [general] `fastmcp-schema-generation` — Inspect the live output of `mcp.get_tools()` to see the actual JSON Schema dialect (anyOf vs nullable) emitted for `belief_ids` under the installed `mcp`/pydantic versions.
- [file] `tests/test_mcp_server.py` — Shows that current tests bypass schema generation entirely; a follow-up test could assert on `mcp.get_tools()` output.

## Beliefs

- `tool-schemas-derived-from-signature-and-docstring` — FastMCP builds each tool's `inputSchema` from `inspect.signature()` (types + defaults) and `description` fields from the Google-style `Args:` block of the function's docstring.
- `propose-update-belief-ids-is-nullable-array` — The `belief_ids: list[str] | None = None` parameter on `propose_update` (`reasons/mcp_server.py:203`) serializes to a nullable array whose schema is not in `required`.
- `with-default-none-params-are-not-required` — Any parameter with a `None` default (e.g., `belief_ids`, `node_ids` on APIs, `visible_to` throughout `reasons/api.py`) is optional in the MCP schema; parameters without a default (e.g., `nogood(node_ids: list[str])`) are required.
- `mcp-tool-registration-requires-mcp-extra` — Importing `reasons/mcp_server.py` requires the optional `mcp>=1.0` extra declared in `pyproject.toml`; otherwise the module raises a targeted `ImportError` before any tool is registered.
- `nogood-requires-list-while-propose-update-optional-list` — `nogood(node_ids: list[str])` (line 175) lacks a default, so `node_ids` is required; `propose_update(belief_ids: list[str] | None = None)` (line 203) is the only tool in the file that makes the list parameter optional.
