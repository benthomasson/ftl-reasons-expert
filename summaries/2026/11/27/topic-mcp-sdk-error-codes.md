---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Enumerate `ErrorCode` values (JSON-RPC standard + MCP-specific) to confirm which ranges `FastMCP` actually emits.

# ErrorCode Enumeration in `reasons-mcp`

## Short answer

**This repository defines zero `ErrorCode` values, references none, and never raises an `McpError` / `JSONRPCError`.** The tool surface in `reasons/mcp_server.py` communicates *application-level* failures by returning a JSON string with an `"error"` key. All JSON-RPC/`ErrorCode`-range emission therefore lives inside the external `mcp` package that `FastMCP` wraps — not in this codebase.

The observations confirm this directly:

| Probe | Result | Meaning |
|---|---|---|
| `locate_errorcode_definition` | 0 definitions | No `ErrorCode` enum in `reasons/` |
| `find_all_errorcode_references` (`ErrorCode`) | 0 matches | Symbol never used |
| `find_mcp_error_raises` (`McpError`, `JSONRPCError`, `PARSE_ERROR`, `INVALID_PARAMS`, …) | 0 matches | No protocol-level error construction |
| `find_fastmcp_usage` | 2 matches (lines 16, 25) | Only the import and the server instance |

So there is no in-repo wire to "confirm which ranges `FastMCP` actually emits" — the mapping is one hundred percent delegated to `mcp`.

## Where `ErrorCode` actually lives

`pyproject.toml` line 17 declares the dependency:

```
mcp = ["mcp>=1.0"]
```

and `reasons/mcp_server.py:16` imports it lazily:

```python
try:
    from mcp.server.fastmcp import FastMCP
except ImportError:
    raise ImportError("The 'mcp' package is required for the MCP server...")
```

The `ErrorCode` enum (and every JSON-RPC code FastMCP can emit) is defined in the installed `mcp` package, somewhere like `mcp/types.py` (or `mcp/shared/exceptions.py` in newer SDK layouts). It is not vendored, not re-exported, and not referenced by any code path in `reasons/`.

## What ranges FastMCP can emit (external SDK behavior)

The `mcp` Python SDK (which `FastMCP` builds on) defines `ErrorCode` across two ranges. These are documented by the JSON-RPC 2.0 spec and the MCP spec — the reasons repo neither overrides nor narrows them:

**JSON-RPC 2.0 standard (−32700…−32603)**

| Value | Name | When FastMCP emits it |
|---|---|---|
| −32700 | `PARSE_ERROR` | Body is not valid JSON before the transport reaches the dispatcher |
| −32600 | `INVALID_REQUEST` | JSON-RPC envelope is malformed (missing `jsonrpc`/`method`/`id`) |
| −32601 | `METHOD_NOT_FOUND` | Unknown method in a `tools/list`/`tools/call`-adjacent frame |
| −32602 | `INVALID_PARAMS` | Request struct fails pydantic validation |
| −32603 | `INTERNAL_ERROR` | Unhandled exception in a protocol handler |

**Server / MCP-specific (−32000…−32099 and the cancel sentinel)**

| Value | Name | When FastMCP emits it |
|---|---|---|
| −32000 | `CONNECTION_CLOSED` | Transport closes mid-request |
| −32002 | `RESOURCE_NOT_FOUND` | MCP-specific: missing resource/URI |
| −32800 | `REQUEST_CANCELLED` | Peer sends `notifications/cancelled` |

**Crucial nuance:** FastMCP differentiates *tool* failures from *protocol* failures. A tool that raises an exception is not converted into a JSON-RPC error code — FastMCP returns a successful MCP result with `isError: true` and the exception message in `content`. The numeric codes above only fire for framing, dispatch, validation, and transport problems. That distinction is exactly why this repo returns error payloads instead of raising.

## What `reasons-mcp` actually emits

Scanning `reasons/mcp_server.py`, every tool wraps its body in `try/except` and returns a JSON *string*:

- `show` — `except KeyError: return json.dumps({"error": f"Node '{node_id}' not found"})`
- `explain` — same pattern
- `add` — `except (ValueError, KeyError) as e: return json.dumps({"error": str(e)})`
- `what_if` — `except KeyError:` and also an inline `{"error": "text is required for what_if supersede"}`
- `propose_update` — `except (KeyError, ValueError, RuntimeError, OSError) as e: return json.dumps({"error": str(e)})`
- `propose_retract`, `propose_supersede`, `propose_add`, `propose_nogood`, `show_proposal`, `accept_proposal` — same catch-and-stringify pattern

Not a single `raise McpError(...)`, `raise JSONRPCError(...)`, or `raise ErrorData(...)` appears. The result: a missing node, a bad argument, or an LLM/OS failure all look like a *successful* MCP tool call whose text happens to contain `{"error": ...}`.

## What the tests confirm (and what they don't)

`tests/test_mcp_server.py` mirrors this behavior. Every error-path test asserts only presence of the key:

- `test_show_missing_returns_error` — `assert "error" in result`
- `test_retract_missing_returns_error` — `assert "error" in result`
- `test_propose_retract_not_found` — `assert "error" in result`
- `test_propose_retract_already_out` — `assert "error" in result`
- `test_what_if_supersede_missing_text` — `assert "error" in result`
- `test_propose_nogood_too_few` — `assert "error" in result`

Zero assertions on `code`, `-32601`, `-32700`, or any numeric literal. And the tests exercise the tools in-process (`mcp_server.show(...)`), not over the stdio/HTTP wire, so they never touch FastMCP's JSON-RPC layer at all.

## What's missing to *definitively* pin the emitted ranges

To answer the original question end-to-end you would still need:

1. **The installed `mcp` version's source** — `pip show mcp` then read `mcp/types.py` / `mcp/shared/exceptions.py`. That is the only authority on which `ErrorCode` members exist in your pinned `mcp>=1.0`.
2. **A protocol-level test** — spawn `reasons-mcp` over stdio (the entry point is `[project.scripts] reasons-mcp = "reasons.mcp_server:main"`, pyproject.toml line 27) and send malformed JSON, an unknown method, and an invalid `tools/call` payload. That would prove which FastMCP frames actually produce −32700 / −32601 / −32602.
3. **A grep for `isError`** — none of the observed tools set it; confirming whether FastMCP auto-sets `isError=true` when a tool returns `{"error": …}` is a behavior question that only a wire-level test or an SDK source read can answer.

## Topics to Explore

- [file] `reasons/mcp_server.py` — Every `try/except` block that maps `KeyError`/`ValueError`/`RuntimeError`/`OSError` to `{"error": ...}` payloads; a good before/after if you decide to move to `ToolError` or real JSON-RPC codes.
- [function] `reasons/mcp_server.py:main` — The `reasons-mcp` entry point that FastMCP bootstraps onto the stdio transport; where a protocol-level test would attach.
- [general] `mcp-sdk-errorcode-source` — Read `mcp/types.py` (or `mcp/shared/exceptions.py`) in your installed `mcp` version to lock in the exact enum members and ranges.
- [general] `protocol-level-tests` — No current test drives the JSON-RPC wire; a malformed-JSON / unknown-method / bad-params test set is what would empirically confirm FastMCP's emission range.
- [repo] `modelcontextprotocol/python-sdk` — Upstream repo where `ErrorCode` and FastMCP's error mapping are actually defined, not vendored here.

## Beliefs

- `errorcode-absent-from-repo` — The symbol `ErrorCode` is neither defined nor referenced anywhere in `reasons/` (0 definitions, 0 references per source probes).
- `mcp-is-optional-dependency` — The `mcp` package is declared under `[project.optional-dependencies].mcp = ["mcp>=1.0"]` in `pyproject.toml` and imported inside a `try/except ImportError` guard at `reasons/mcp_server.py:16`.
- `fastmcp-references-limited-to-two-lines` — The only `FastMCP` usages are the import at `reasons/mcp_server.py:16` and the instance `mcp = FastMCP("reasons")` at line 25.
- `tool-errors-return-json-payload` — Every error path in `reasons/mcp_server.py` returns `json.dumps({"error": ...})` instead of raising `McpError` or a JSON-RPC exception.
- `mcp-tests-assert-only-error-key` — All error-path assertions in `tests/test_mcp_server.py` check `"error" in result` and never a specific numeric `code`.
- `mcp-server-uses-stdio-entrypoint` — `reasons-mcp` is registered as `reasons.mcp_server:main` in `[project.scripts]`, so `FastMCP` runs over stdio by default.
