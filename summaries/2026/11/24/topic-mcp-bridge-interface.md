---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The duck-typed contract (`list_tools`, `call_tool`, `get_instructions`) enforced only implicitly.

## The duck-typed bridge contract in `reasons/ask.py`

### What the contract is

`reasons/ask.py` treats any object passed via `mcp_servers=[...]` as a "bridge" if it exposes three methods. There is no base class, no `typing.Protocol`, and no `@runtime_checkable` declaration anywhere in the codebase (a codebase-wide search for `Protocol|runtime_checkable|duck[- ]typ` across the `hasattr/getattr` bucket returns 141 hits, but every visible hit is `getattr(args, ...)` in `reasons/cli.py` — none relate to the bridge contract). The three methods are:

| Method | Call site | Expected return shape |
|---|---|---|
| `list_tools()` | `reasons/ask.py:356`, `reasons/ask.py:487` | `list[dict]` where each dict has `name`, `description`, `input_schema` |
| `get_instructions()` | `reasons/ask.py:377` | `str` (may be empty) |
| `call_tool(name, arguments)` | `reasons/ask.py:539` | `str` (the tool's textual result) |

The shape of `list_tools()` entries is pinned only by what `_build_tools_section` consumes in the tests: `tests/test_ask_mcp.py:59-73` passes dicts with exactly those three keys and asserts the description and a property (`SELECT`) surface in the generated prompt. Nothing validates the keys at runtime.

### The concrete implementation

`reasons/mcp_client.py` defines `McpBridge` (`reasons/mcp_client.py:26`), which is the production implementation of the contract:

- `list_tools` (line 94) simply returns `self._tools`, populated once during `_session_lifecycle` at `reasons/mcp_client.py:77-85` from `session.list_tools()`. Each entry is built as `{"name": t.name, "description": t.description or "", "input_schema": t.inputSchema}`.
- `get_instructions` (line 97) returns `self._instructions`, set at line 74 from `result.instructions` only if `hasattr(result, "instructions")` — itself another duck-typed access, this time into the MCP SDK's `InitializeResult`.
- `call_tool(name, arguments)` (line 100) schedules `_call_tool_async` on the background event loop and blocks on `future.result(timeout=60)`. `_call_tool_async` returns a single `str` by joining `content.text` values (`reasons/mcp_client.py:106-113`), so `call_tool` always returns a string — never the raw SDK content list.

So `McpBridge` satisfies the contract but does not *declare* that it does, and `ask.py` never imports `McpBridge`. That decoupling is deliberate: `mcp_client.py` imports the `mcp` package in a guarded `try/except` (lines 9-14) and raises `ImportError` via `_require_mcp()` if it's missing, so `ask.py` can be used in environments where `mcp` isn't installed (see `pyproject.toml` — `mcp = ["mcp>=1.0"]` is an optional extra) as long as the caller supplies a bridge-shaped object.

### How it is "enforced"

It isn't — that's the point. Enforcement is entirely implicit:

1. **Attribute lookup at call time.** `bridge.list_tools()` on line 356 will raise `AttributeError` (or `TypeError` if it's not callable) the first time `_build_tools_section` runs. Same for `get_instructions()` at 377 and `call_tool(...)` at 539. There is no pre-flight check, no `hasattr`, no `isinstance`.
2. **The tests are the type checker.** `tests/test_ask_mcp.py:26-37` defines `FakeBridge` with nothing but the three methods, and every integration test in that file (`test_mcp_tool_dispatched`, `test_mcp_tool_error_handled`, `test_max_iterations_bumped_with_mcp`) passes it to `ask(..., mcp_servers=[bridge])` unchanged. If `FakeBridge` were missing a method these tests would fail — the tests are effectively the executable specification of the contract.
3. **Monkeypatching is legitimate.** `tests/test_ask_mcp.py:168-174` snapshots `bridge.call_tool` and replaces it with `tracking_call` to observe dispatch; `tests/test_ask_mcp.py:203` replaces it with a lambda that raises. Both rely on the bridge being a plain object with public attributes, which is the ethical consequence of duck typing: the contract is not a sealed interface. If `McpBridge` were an ABC with a `final` `call_tool`, both tests would break.
4. **Error isolation is separate from contract enforcement.** The `call_tool` failure test (`RuntimeError("connection lost")`) asserts that `ask()` still returns a useful answer — meaning the *runtime behavior* of a broken bridge is tolerated at the dispatch layer (`reasons/ask.py:539` callers catch the exception and record it in tool history), but a *structurally* missing method is not handled anywhere. So the contract is enforced only to the extent that callers crash on the first missing attribute.

### The boundary: what's in the contract vs. not

- **In the contract:** the three methods.
- **Not in the contract:** `connect`, `close`, `__enter__`, `__exit__`. `McpBridge` implements all of these (`reasons/mcp_client.py:39-130`), but `FakeBridge` implements none and every test still passes. So context-manager support is an implementation detail of `McpBridge`, not something `ask.py` can rely on.
- **Not enforced:** return value shape of `list_tools()` entries. If an implementation returned `{"name": ..., "parameters": ...}` (missing `input_schema`), the failure would only appear inside prompt construction, not at the boundary.

### What's missing from the observations

The full source of `reasons/ask.py` isn't in the snapshot — only four extracted lines (356, 377, 487, 539). So the exact signature of `ask()`, whether `mcp_servers` has a type annotation, and whether `_build_tools_section` defensively reads `.get("input_schema", {})` or indexes it directly are all unverified. If you need to know whether adding a `typing.Protocol` would be safe, read `reasons/ask.py` around lines 340-400 and 480-545 first.

## Topics to Explore

- [function] `reasons/ask.py:_build_tools_section` — The primary consumer of `list_tools()`; confirms whether the `name`/`description`/`input_schema` keys are read via `.get` or indexing, which determines how strict the return-value half of the contract actually is.
- [function] `reasons/mcp_client.py:McpBridge.call_tool` — The production implementation; shows the timeout (`future.result(timeout=60)`) and string-flattening behavior that a replacement bridge would need to match.
- [file] `reasons/mcp_server.py` — The server side that produces the tool metadata the bridge wraps; the source of `t.description` and `t.inputSchema` shapes.
- [general] `bridge-duck-typing-tradeoff` — Whether introducing a `typing.Protocol` (with `@runtime_checkable`) plus a `hasattr` pre-flight check at `ask()`'s entry would catch the `AttributeError` earlier without breaking the `FakeBridge` and monkeypatch tests.
- [function] `reasons/ask.py:ask` — The orchestration loop that iterates `mcp_servers` and dispatches `call_tool`; the only place where a bridge is actually wired into the reasoning flow.

## Beliefs

- `ask-py-requires-bridge-implements-three-methods` — `ask()` will call `list_tools`, `get_instructions`, and `call_tool` on every object passed in `mcp_servers`; no other bridge methods are invoked.
- `mcp-bridge-is-concrete-bridge-implementation` — `McpBridge` is the only production class in `reasons/mcp_client.py` implementing the three-method contract, and it is not imported by `reasons/ask.py`.
- `list-tools-returns-name-description-input-schema-dicts` — The entries returned by `list_tools()` are consumed as dicts with keys `name`, `description`, and `input_schema` (`reasons/mcp_client.py:78-85`, `tests/test_ask_mcp.py:56-73`).
- `call-tool-returns-single-string` — `McpBridge.call_tool` returns a `str` via `"\n".join(parts)`, never the raw MCP `content` list (`reasons/mcp_client.py:106-113`).
- `bridge-contract-is-confirmed-by-tests-not-by-typing` — There is no `Protocol`, ABC, or `runtime_checkable` for the bridge; `FakeBridge` in `tests/test_ask_mcp.py:26-37` is the executable specification of the contract.
- `missing-bridge-method-causes-attribute-error` — A bridge missing any of the three methods fails with `AttributeError` at the call site in `ask.py`, not at construction time.
- `bridge-dispatch-errors-are-caught-not-raised` — Exceptions raised inside `call_tool` are captured into tool history and do not abort the `ask()` loop (`tests/test_ask_mcp.py:181-207`).
