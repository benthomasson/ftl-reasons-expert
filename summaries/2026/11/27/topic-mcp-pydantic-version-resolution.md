---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: The resolved `mcp`/`pydantic` pair determines whether the schema is anyOf or nullable; pin or inspect this before relying on the schema.

# Schema shape depends on the resolved `mcp`/`pydantic` pair

## The claim, unpacked

Optional fields on MCP tools can be represented two ways in the emitted JSON Schema:

- **`anyOf` form** — JSON Schema 2020-12 / OpenAPI 3.1 style:
  `"anyOf": [{"type": "array", "items": {"type": "string"}}, {"type": "null"}]`
- **`nullable` form** — OpenAPI 3.0 style:
  `"type": "array", ..., "nullable": true`

Which one you get is decided by the `mcp` library (which serializes FastMCP tool signatures to `inputSchema`) and the `pydantic` version it resolves against (which produces `model_json_schema()` / `.schema()` in a version-specific dialect). Neither of these dialects is stable across the `mcp` 1.x line or across pydantic v1→v2.

The observation confirms **the codebase itself does not choose**. Both greps return zero matches:

- `find_anyof_schema_logic`: `total_count: 0`
- `find_nullable_schema_logic`: `total_count: 0`

There is no normalization, no dialect coercion, and no assertion on the shape. The only schema touchpoint is the **consumer**:

- `reasons/mcp_client.py:82` — `"input_schema": t.inputSchema,`

That line takes whatever `mcp` handed back and stores it. So the schema contract you actually get at runtime is whatever the resolved `(mcp, pydantic)` pair emits.

## Where the schema is produced (implicitly)

`reasons/mcp_server.py` builds `mcp = FastMCP("reasons")` and declares every tool via `@mcp.tool()`. There is no direct call to `model_json_schema` or `schema_json` in the codebase — the sole `model_json_schema|schema_json|inputSchema|outputSchema` match is the client's read at line 82. Schema generation is entirely delegated to FastMCP.

A concrete example that forces the fork is `propose_update` in `reasons/mcp_server.py`:

```python
@mcp.tool()
def propose_update(belief_ids: list[str] | None = None, model: str = "claude",
                   ...
```

`list[str] | None` is exactly the type that pydantic renders as `anyOf: [..., {"type": "null"}]` under v2 defaults, and that a v1-style generator would render as `{"type": "array", ..., "nullable": true}`. Most other tool params use plain defaults (`str = ""`, `int = 0`) which are not optional-typed and won't hit the fork, but `propose_update` and any future `X | None` params will.

## Why the pin matters

`pyproject.toml` (35 lines total, `lines_returned: 34`) declares:

```toml
mcp = ["mcp>=1.0"]
```

That is the entire constraint. There is **no upper bound on `mcp`** and **no explicit `pydantic` dependency at all** — `pydantic` is pulled in transitively by `mcp`. Consequences:

- `pip install 'ftl-reasons[mcp]'` today may resolve to a newer `mcp` and a newer `pydantic` than what was tested.
- Because `reasons/mcp_client.py:82` blindly stores `t.inputSchema`, any downstream code that inspects `input_schema` will see the ambient dialect, not a stable one.
- No test pins the shape. `tests/test_mcp_server.py:8` uses `pytest.importorskip("mcp")`, so the entire MCP test module silently skips when `mcp` isn't installed. And nothing in the visible tests exercises `propose_update`'s schema or `inputSchema` — the tests exercise tool *behavior*, not schema *shape*.

## What to do

Before relying on the `inputSchema` returned by `reasons/mcp_client.py:82`:

1. **Inspect at runtime.** Resolve the actual versions in the environment (`pip show mcp pydantic`) and dump `propose_update`'s `inputSchema` once. Check whether it contains `anyOf` or `nullable`.
2. **Pin or narrow.** Convert `mcp = ["mcp>=1.0"]` to something like `mcp = ["mcp>=1.0,<2"]` and add an explicit `pydantic` bound (ideally the v2 line) so the dialect is reproducible across installs.
3. **If you cannot pin** (e.g., library used across environments), add an explicit normalizer in `reasons/mcp_client.py` before that schema is stored — collapse `anyOf: [X, {"type":"null"}]` and `{"nullable": true, ...}` into one canonical internal form.

## What's missing from the observations

- The full contents of `reasons/mcp_client.py` beyond line 82 were not surfaced — I cannot see whether any downstream code assumes a particular dialect.
- The resolved `mcp`/`pydantic` versions in the environment are not in the observations; only the declared constraint.
- `reasons/mcp_server.py` was truncated at line 300 of 600; additional `X | None` tool params may exist past that cut.
- No test was observed that inspects `inputSchema`, so the shape is effectively untested.

## Topics to Explore

- [file] `reasons/mcp_client.py` — Read past line 82 to see whether any consumer assumes `anyOf` or `nullable` shape for `input_schema`
- [file] `pyproject.toml` — The `mcp>=1.0` constraint on line 18 is the only pin; tightening it (or adding a pydantic bound) fixes the dialect
- [function] `reasons/mcp_server.py:propose_update` — The only observed tool parameter typed `list[str] | None`, which is the canonical trigger for the anyOf-vs-nullable fork
- [general] `mcp-version-resolution` — Resolve the actual installed `mcp`/`pydantic` pair in the target environment and dump one `inputSchema` to see the empirically emitted dialect
- [file] `tests/test_mcp_server.py` — Uses `pytest.importorskip("mcp")` at line 8; add a schema-shape test here to lock the contract

## Beliefs

- `mcp-pin-is-a-minimum-only` — `pyproject.toml` declares `mcp = ["mcp>=1.0"]` with no upper bound, so the resolved `mcp` version is unbounded across installs.
- `pydantic-is-transitive-not-declared` — `pydantic` is not listed in `[project.optional-dependencies]`; it arrives only as a transitive dependency of `mcp`.
- `client-consumes-inputschema-verbatim` — `reasons/mcp_client.py:82` stores `t.inputSchema` without any dialect normalization.
- `no-schema-dialect-handling-exists` — Repository-wide greps for `anyOf` and `nullable` return zero matches, so neither schema form is normalized or asserted on.
- `propose-update-has-optional-list-param` — `propose_update(belief_ids: list[str] | None = None, ...)` in `reasons/mcp_server.py` is the observed parameter whose schema will differ between anyOf and nullable dialects.
- `mcp-tests-skip-without-mcp-installed` — `tests/test_mcp_server.py:8` uses `pytest.importorskip("mcp")`, so schema-dependent tests do not run when `mcp` is absent.
