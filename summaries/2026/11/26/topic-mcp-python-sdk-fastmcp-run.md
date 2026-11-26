---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: External SDK source needed to verify `FastMCP.run()` default transport and exact JSON-RPC framing.

# Verifying `FastMCP.run()` default transport and JSON-RPC framing

## Short answer

**You cannot verify this from the codebase alone.** The behavior you're asking about — which transport `FastMCP.run()` uses when called with no arguments, and how individual JSON-RPC messages are framed on the wire — is defined inside the external `mcp` package, not in `reasons/`. Everything in this repo is a *consumer* of that SDK. To answer the question you must read the installed SDK source.

## What the codebase tells us (and doesn't)

### 1. `FastMCP` is imported from the external `mcp` package

`reasons/mcp_server.py:16`:

```python
from mcp.server.fastmcp import FastMCP
```

That's the only definition site the search found (`find_fastmcp_definitions_and_usages` — total 2 matches: the import at line 16 and the instantiation at line 25). No subclass, no wrapper, no override:

```python
mcp = FastMCP("reasons")   # reasons/mcp_server.py:25
```

So `mcp.run()` here is literally the SDK's `FastMCP.run` — whatever behavior that method has, that's what this process gets.

### 2. No transport is configured in this repo

`find_transport_references` returned **zero matches** across the repo. There is no call to `mcp.run(transport=...)` or `mcp.run("sse")` or `mcp.run("stdio")` anywhere the search spanned. Assuming the search covered the whole tree (it returned `[]` rather than an error), transport selection is entirely left to the SDK default.

### 3. The entry point hides the actual `run()` call

`pyproject.toml` registers:

```toml
[project.scripts]
reasons-mcp = "reasons.mcp_server:main"
```

But `read_mcp_server_implementation` only returned **lines 1–250 of 600** for `reasons/mcp_server.py`. The `def main()` function lives in the unreturned portion (lines ~250–600), and we have not seen it. So we can't confirm:
- whether `main()` calls `mcp.run()` with no args,
- whether it passes `transport="stdio"` explicitly,
- whether it wraps `mcp` in `uvicorn`/`sse_app` for a different transport.

This is a genuine gap in the observations.

### 4. The tests do not exercise the wire at all

`tests/test_mcp_server.py` calls the decorated tool functions directly, e.g.:

```python
result = mcp_server.show("premise-a")          # tests/test_mcp_server.py:78
result = json.loads(mcp_server.list_beliefs()) # tests/test_mcp_server.py:110
```

`@mcp.tool()` in FastMCP just registers the function and returns it unchanged, so the tests are hitting plain Python and never touch `mcp.run()`, JSON-RPC, or framing. There is no test asserting anything about transport.

### 5. The sibling client reveals what transport the *other* side expects

`reasons/mcp_client.py` is the client half of the same protocol:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
...
server_params = StdioServerParameters(command=parts[0], args=parts[1:])
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
```

(`reasons/mcp_client.py:10-11` and `McpBridge._session_lifecycle`.)

This tells you the client side targets **stdio** by default — but it does *not* tell you what `FastMCP.run()` chooses, because the client and server defaults are configured independently in the SDK. A user could call `mcp.run(transport="sse")` in `main()` and break this client; the fact that the client is stdio-only is suggestive evidence that the server default *is* stdio, but it isn't proof.

## Where the actual answer lives (external SDK)

To verify authoritatively, open these files in the installed `mcp` wheel (typically `.venv/lib/python3.11/site-packages/mcp/...`, version per `pyproject.toml`'s `mcp = ["mcp>=1.0"]`):

| Question | File |
|---|---|
| What does `FastMCP.run()` default to? | `mcp/server/fastmcp/server.py` — the `FastMCP` class, `run(self, transport=..., mount_path=...)` signature and body |
| What transports are supported? | `mcp/server/fastmcp/server.py` and `mcp/server/stdio.py` |
| How are stdio messages framed on the wire? | `mcp/server/stdio.py` — look for `readline()` / `json.loads(line)` / explicit `\n` writes, or `Content-Length:` header parsing |
| Client-side mirror | `mcp/client/stdio/__init__.py` — `stdio_client`, to confirm the sender and receiver agree on framing |

MCP's stdio transport (per the spec, and per the `mcp` PyPI package at `mcp>=1.0`) uses **newline-delimited JSON-RPC 2.0** — one JSON object per `\n`-terminated line, with no `Content-Length` header — rather than the `Content-Length:` header framing used by the LSP. But you should confirm this against the installed version, because:
- `mcp>=1.0` is a wide constraint and there is no lockfile pinning the exact version in `pyproject.toml`.
- The repo never asserts it anywhere, and none of the tests would catch a framing change on upgrade.

The two `application/json` hits (`reasons/hf.py:112`, `reasons/mind_service.py:69`) are unrelated — they're `Content-Type` headers on plain HTTP `requests.post` calls, not JSON-RPC framing code.

## What is missing from the observations

1. **`reasons/mcp_server.py` lines 250–600** — specifically `main()`, which is where `mcp.run()` is actually invoked. Without it, we cannot even rule out `mcp.run(transport="sse")`.
2. **The installed `mcp` package version and source** — not in the repo, not vendored, not pinned in `pyproject.toml`.
3. **Any integration test** — there is none; `tests/test_mcp_server.py` never starts the server process or performs an `initialize` handshake.

Until (1) and (2) are inspected, any statement of the form "`FastMCP.run()` defaults to stdio with newline-delimited JSON-RPC" is an assumption about the upstream library, not a fact this repo establishes.

## Topics to Explore

- [file] `reasons/mcp_server.py` — Read lines 250–600 to find `def main()` and whether it calls `mcp.run()` with a transport argument or relies on the SDK default.
- [file] `.venv/lib/python*/site-packages/mcp/server/fastmcp/server.py` — The actual `FastMCP.run()` definition; the default value of the `transport` parameter is the ground truth.
- [file] `.venv/lib/python*/site-packages/mcp/server/stdio.py` — The stdio transport implementation that reveals the framing (newline-delimited JSON vs. Content-Length headers).
- [function] `reasons/mcp_client.py:McpBridge._session_lifecycle` — The client-side counterpart; confirms the client expects stdio and initializes via `ClientSession.initialize()`.
- [general] `mcp-specification-stdio-transport` — The official MCP spec section on stdio framing, useful to cross-check the SDK against the standard.

## Beliefs

- `fastmcp-import-requires-mcp-package` — `reasons/mcp_server.py:16` imports `FastMCP` from `mcp.server.fastmcp`, so `FastMCP.run()` behavior is defined entirely by the external `mcp` package, not this repo.
- `mcp-server-entry-point-is-main` — `pyproject.toml` maps the `reasons-mcp` script to `reasons.mcp_server:main`, but `main()` is outside the returned 250-line excerpt and its `run()` call is not observable here.
- `no-transport-references-in-repo` — Searching the repo for `transport` returned zero matches, meaning transport selection is never configured explicitly and defaults entirely to the SDK.
- `tests-bypass-run-and-framing` — `tests/test_mcp_server.py` calls tool functions directly (e.g. `mcp_server.show` at line 78) and never invokes `mcp.run()` or exercises JSON-RPC framing.
- `client-uses-stdio-transport` — `reasons/mcp_client.py:10-11` imports `StdioServerParameters` and `stdio_client`, indicating the repo's own MCP *client* targets stdio (but not proving the server's `run()` default matches).
