---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: How `FastMCP` converts the Python signature into `Tool.inputSchema` (in pydantic v2 this routes through `model_json_schema()`).

## How `FastMCP` converts a Python signature into `Tool.inputSchema`

### Short answer

The conversion does **not** live in this repository. `FastMCP` is imported from the third-party `mcp` package (`reasons/mcp_server.py`, lines ~9–14: `from mcp.server.fastmcp import FastMCP`), and that package is the code that walks the function signature, builds a pydantic model, and calls `model_json_schema()`. Everything in this repo is the *consumer side* of that contract.

### What the observations show

**1. `FastMCP` is instantiated and tools are registered via decorator**

`reasons/mcp_server.py`:

- Line ~22: `mcp = FastMCP("reasons")`
- Lines ~48, ~58, ~68, ~78, …: `@mcp.tool()` decorating functions like `search`, `show`, `explain`, `list_beliefs`, `add`, `retract`, etc.

Each decorated function has an explicit, fully-annotated signature, e.g.:

```python
@mcp.tool()
def search(query: str, output_format: str = "markdown", depth: int = 1,
           include_out: bool = False, limit: int = 0, offset: int = 0) -> str:
```

That typed signature is the *entire* input to schema generation — no `Field`, no explicit model, no manual JSON Schema is provided anywhere in `reasons/mcp_server.py`.

**2. The client normalizes `inputSchema` → `input_schema`**

- `reasons/mcp_client.py:82`: `"input_schema": t.inputSchema,` — this is reading the object attribute `Tool.inputSchema` that FastMCP produced.
- `reasons/ask.py:357`: `schema = tool.get("input_schema", {})` — the downstream consumer treats it as a plain JSON Schema dict.

**3. Tests pin the expected shape**

`tests/test_ask_mcp.py` fixtures assert the schema is normal JSON Schema shaped (`{"properties": {...}}`), e.g. line 49 (`"input_schema": {`), line 165 (`"properties": {"sql": {"description": "SQL", "type": "string"}}`), lines 63–74 (empty `properties` for no-arg tools). That is exactly the `{properties, required, type}` shape that `model_json_schema()` emits.

**4. `mcp` is an optional dependency**

`pyproject.toml`: `mcp = ["mcp>=1.0"]`. So the source that actually implements the signature→schema conversion is not vendored here — it lives in the installed `mcp` package.

### The route (conceptual, since the implementation isn't in these files)

For every `@mcp.tool()`-decorated function, FastMCP in the `mcp` package does roughly:

1. **Introspect** the function with `inspect.signature(fn)` — pull parameter names, annotations (`str`, `int`, `bool`, `list[str]`, …), and defaults (`"markdown"`, `1`, `False`, `0`, `""`).
2. **Describe** the tool's name (defaults to `fn.__name__`) and description (defaults to the docstring — e.g., the multi-line docstring on `search` in `reasons/mcp_server.py`).
3. **Build a pydantic v2 model** — typically via `pydantic.create_model(...)`, one field per parameter. `str`/`int`/`bool`/`list[str]` become pydantic `str`/`int`/`bool`/`List[str]`. A parameter with a default like `output_format: str = "markdown"` becomes a field with `default="markdown"`; a parameter with no default becomes `required`.
4. **Emit JSON Schema** from that model by calling **`SomeModel.model_json_schema()`** in pydantic v2. This produces `{"type": "object", "properties": {...}, "required": [...], "title": "..."}` — the same shape asserted in `tests/test_ask_mcp.py`.
5. **Assign the result** to the `Tool` returned to the MCP runtime as `Tool.inputSchema`.

The end-to-end chain is therefore:

```
@mcp.tool() def f(x: int, y: str = "a")
        │  inspect.signature + annotations + docstring
        ▼
pydantic create_model(…)
        │
        ▼
Model.model_json_schema()        ← pydantic v2 route
        │
        ▼
Tool.inputSchema                 ← MCP wire field (camelCase)
        │
        ▼
client: t.inputSchema → input_schema   (reasons/mcp_client.py:82)
        │
        ▼
schema = tool["input_schema"]           (reasons/ask.py:357)
```

### What's missing from the observations

The exact FastMCP implementation — the file(s) under `mcp/server/fastmcp/` in the installed package that call `inspect.signature`, `create_model`, and `model_json_schema()` — is **not** in the gathered excerpts. `find_schema_generation_calls` returned **zero matches** for `model_json_schema|json_schema|create_model|TypeAdapter` because those symbols are never referenced from `reasons/` or `tests/` — they live entirely inside the external `mcp` package. Also missing: the version-resolved location of `mcp` (only `mcp>=1.0` is declared) and whether FastMCP uses `TypeAdapter` vs `create_model` internally for scalars vs nested types.

To fully verify the claim, read the installed `mcp/server/fastmcp/tools/` (or `utilities/func_metadata.py` in typical FastMCP releases) at the pinned `mcp` version.

## Topics to Explore

- [file] `reasons/mcp_server.py` — The `@mcp.tool()`-decorated functions whose signatures drive every `inputSchema`; study how annotations, defaults, and the docstring block map onto JSON Schema fields and `description`.
- [general] `mcp-server-fastmcp-source` — The external source of `mcp.server.fastmcp.FastMCP` (typically `mcp/server/fastmcp/tools/base.py` and `utilities/func_metadata.py`) — this is where `inspect.signature`, pydantic model construction, and `model_json_schema()` actually run.
- [function] `reasons/mcp_client.py:82` — Where `Tool.inputSchema` is read from the MCP server and re-exposed as `input_schema`; the rename boundary between MCP wire names and internal dicts.
- [file] `reasons/ask.py` — The consumer at line 357 that reads `tool.get("input_schema", {})`; useful for understanding which parts of the generated schema (properties, required, description) actually affect model tool-calling.
- [general] `pydantic-v2-model_json_schema` — The pydantic v2 contract (`$defs`, `anyOf`, `required`, default emission) that determines the JSON Schema FastMCP ends up assigning to `Tool.inputSchema`.

## Beliefs

- `fastmcp-inputschema-produces-from-signature` — Every `Tool.inputSchema` in this server derives solely from the decorated function's `inspect`-observable signature (annotations + defaults + docstring); no `Field(...)` or hand-written schema exists in `reasons/mcp_server.py`.
- `fastmcp-is-third-party-dependency` — `FastMCP` is imported from the optional `mcp` package (`mcp>=1.0` in `pyproject.toml`), so the signature-to-schema conversion is not present in this repo.
- `mcp-server-registers-tools-via-decorator` — All tool schemas originate from `@mcp.tool()` decorators (e.g., `search`, `show`, `add`, `retract`) in `reasons/mcp_server.py`.
- `client-renames-inputSchema-to-input_schema` — `reasons/mcp_client.py:82` renames MCP's camelCase `t.inputSchema` to snake_case `input_schema`, which `reasons/ask.py:357` then consumes.
- `tests-assert-json-schema-shape` — `tests/test_ask_mcp.py` fixtures assert `input_schema` is a JSON Schema-shaped dict with `properties` and typed fields, matching pydantic v2's `model_json_schema()` output.
