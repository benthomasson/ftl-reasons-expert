---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: `mcp_server.py` vs `mcp_client.py` is the clearest example of when to fail at import vs. at call; codify the rule before extracting a helper.

## The rule in one sentence

> **Fail at import when the module has import-time side effects that require the dependency (module-level instantiation, `@dep.tool` decorators, class bodies that subclass the dep); fail at call when the module is a widely-imported utility where the dep is additive and the symbols that need it are only touched on demand.**

`mcp_server.py` and `mcp_client.py` are the two halves of that rule living side-by-side in the same package, using the *same* optional extra (`mcp>=1.0`, `pyproject.toml:27`).

---

## Why `mcp_server.py` must fail at import

`reasons/mcp_server.py:16-21` does not swallow the `ImportError` — it re-raises with a targeted install hint:

```python
try:
    from mcp.server.fastmcp import FastMCP
except ImportError:
    raise ImportError(
        "The 'mcp' package is required for the MCP server. "
        "Install with: pip install 'ftl-reasons[mcp]'"
    )
```

The three structural facts that make import-time failure the only correct choice:

1. **Line 25: `mcp = FastMCP("reasons")` is a module-level binding.** If the guard didn't raise, this line would blow up with a `NameError: name 'FastMCP' is not defined`, not a useful `ImportError`. The guard exists to convert that into an actionable message *before* the module continues evaluating.
2. **Every tool is decorated at import scope.** From `search` (`mcp_server.py:~44`) through `accept_proposal` (end of file), each `@mcp.tool()` decorator is resolved while the module is being imported. Deferred failure is not possible — you cannot reach any of those 25+ functions without first passing through all the decorators.
3. **The only caller is the entry point.** `pyproject.toml:32`: `reasons-mcp = "reasons.mcp_server:main"`. There is no legitimate import path where "module loads without `mcp`" is a valid state — the module *is* the MCP server. `tests/test_mcp_server.py:8` confirms this by using `pytest.importorskip("mcp")`: when `mcp` is absent, the right outcome is that the whole module is unseen, which is exactly what import-time failure produces.

The `reasons_lib/mcp_server.py` re-export shim (`from reasons.mcp_server import *`) inherits this behavior, which is desirable: importing the shim without `mcp` should error loudly with the install hint.

---

## Why `mcp_client.py` must fail at call

`reasons/mcp_client.py:14-24` does not raise. It sets a flag and defers:

```python
try:
    from mcp import ClientSession, StdioServerParameters
    from mcp.client.stdio import stdio_client
    _mcp_available = True
except ImportError:
    _mcp_available = False


def _require_mcp():
    if not _mcp_available:
        raise ImportError(
            "mcp is required for MCP server support. "
            "Install it with: pip install 'mcp>=1.0'"
        )
```

The three structural facts that make call-time failure mandatory:

1. **The module is imported by `ask.py` on every `reasons ask` invocation.** The docstring at `mcp_client.py:1-4` explicitly says it exists "so ask.py can dispatch tool calls to any MCP server." MCP is an *additive* feature of `ask` — the user typing `reasons ask "why is X IN?"` with no MCP servers configured must not need the `mcp` package installed. If import failed here, it would break the entire `ask` command for anyone who didn't install `ftl-reasons[mcp]`.
2. **There are no module-level symbols that require `mcp`.** Unlike the server, `McpBridge` (`class McpBridge:`, line ~32) is a plain class — no `ClientSession`-derived base, no module-level session. The names imported in the `try` block are only used *inside* coroutines (`_session_lifecycle`, `_call_tool_async`). So the module body is fully evaluable without `mcp`.
3. **The first opportunity the caller has to *signal intent* is `__init__`.** `McpBridge.__init__` calls `_require_mcp()` (line ~47). That's the precise moment: "you constructed a bridge → you want MCP → therefore demand the dep." That granularity is testable: `tests/test_ask_mcp.py` uses a `FakeBridge` that never touches `_require_mcp`, so all ask-with-MCP logic can be tested without `mcp` installed.

The `reasons_lib/mcp_client.py` re-export shim also inherits this: importing the shim succeeds on any system, even one without `mcp`. You only pay when you instantiate a bridge.

---

## The rule codified

| Condition | Failure site | Example |
|---|---|---|
| Module-level instantiation of a dep object (`x = Dep()`) | **Import** | `mcp_server.py:25` `mcp = FastMCP("reasons")` |
| Dep-flavored decorators applied at module scope (`@dep.tool()`) | **Import** | all of `mcp_server.py` tool functions |
| Class body inherits from dep type (`class Foo(DepBase):`) | **Import** | (not present here) |
| Dep symbols only appear *inside* function bodies | **Call** | `mcp_client.py:_session_lifecycle`, `_call_tool_async` |
| Module is imported as a transitive dependency of core code (`ask.py`) | **Call** | `mcp_client.py` imported by `reasons.ask` |
| Module is only entered via a dedicated entry point (`reasons-mcp`) | **Import** | `pyproject.toml:32` |

Two corollaries that matter for the eventual helper:

- **The same extra can be both.** `mcp>=1.0` (`pyproject.toml:27`) feeds both modules; the failure semantics differ because the *module shapes* differ. Don't infer the failure mode from the dependency — infer it from the module.
- **Error messages should still be consistent even when failure sites are not.** Today they aren't: server says `pip install 'ftl-reasons[mcp]'`, client says `pip install 'mcp>=1.0'`. That's a message-formatting inconsistency a helper *can* fix — but it must not touch the placement.

---

## What the helper can and cannot decide

This is why codifying the rule must precede extraction:

- **A helper cannot infer *when*.** Whether to raise at import or at call is determined by where the dependent symbol is first *evaluated* — `mcp_server.py` evaluates `FastMCP` at line 25, so the guard must precede line 25; `mcp_client.py` first evaluates `ClientSession` inside `_session_lifecycle`, so the guard belongs in `__init__`. No amount of introspection inside `require_optional()` can relocate that.
- **A helper can standardize *how*.** The two call sites differ only in (a) whether the check raises or flips a boolean, and (b) the install-hint string. A `require_optional(package, extra=...)` helper can encapsulate both if it supports two modes:
  - **probe mode** — return a boolean (used by `mcp_client.py:19` for `_mcp_available`),
  - **require mode** — raise `ImportError` with a canonical message (used by `_require_mcp()` and by `mcp_server.py:18`).
- **A helper must not swallow runtime `ImportError`s.** A lazy re-import inside `_session_lifecycle` that fails mid-call is a *different* failure (broken install, conflicting versions) and must not be caught by the availability probe. Only the initial top-of-module `try` belongs to the helper's contract.

The invariant to write in the helper's docstring:

> `require_optional()` does not change *where* you fail. It only changes what the error says. Call it at module top-level (before the first dependent symbol binding) for import-time failure; call it inside `__init__`/the first-use function for call-time failure.

---

## What's missing from the observations

- The actual `main()` function at the bottom of `mcp_server.py` (lines 500-600 were truncated) — it presumably calls `mcp.run()`, but I can't confirm whether it has additional guards.
- The top of `reasons/ask.py` showing *how* `mcp_client` is imported (deferred vs. top-level) — the docstring asserts the relationship, but the import line itself isn't in the observations.
- Other lazy-guard modules (`reasons/llm.py:23-44`, `reasons/cluster.py:17`, `reasons/pg.py:18`) — these are candidate consumers of the same helper and would reveal whether a third failure placement (e.g., "first non-trivial public call") exists in the codebase.

---

## Topics to Explore

- [file] `reasons/ask.py` — Confirm exactly where and how `mcp_client.McpBridge` is imported; the deferred-failure strategy for `mcp_client.py` depends entirely on this import being safe to execute without `mcp` installed.
- [function] `reasons/mcp_client.py:McpBridge.__init__` — The single call point for `_require_mcp()`; it defines the "first signal of intent" contract that a shared helper must preserve.
- [file] `reasons/llm.py` — Contains four separate `try/except ImportError` blocks (lines 23, 30, 37, 44) plus raises at 208, 215, 252; check whether these follow the import-vs-call rule or diverge, which would tell you whether the helper generalizes.
- [file] `reasons/pg.py` — Raises `ImportError` at line 92 inside a function while guarding at line 18; appears to be another call-time case, useful as a second example for the helper contract.
- [general] `optional-dependency-failure-matrix` — A written matrix of every optional extra (`mcp`, `pg`, `cluster`, `llm`, `vertex`, `langfuse`, `openai` in `pyproject.toml:10-21`) vs. where its guard fires, so the eventual helper doesn't accidentally homogenize heterogeneous cases.

---

## Beliefs

- `mcp-server-fails-at-import-because-of-module-level-fastmcp-instantiation` — `reasons/mcp_server.py` must raise at import time because line 25 binds `mcp = FastMCP("reasons")` at module scope before any deferred `try/except` could help.
- `mcp-client-defers-failure-to-support-ask-without-mcp` — `reasons/mcp_client.py` sets `_mcp_available = False` on ImportError and only raises via `_require_mcp()` from `McpBridge.__init__`, so that `reasons.ask` imports cleanly on systems without the `mcp` extra installed.
- `same-optional-extra-has-opposite-failure-semantics` — Both modules gate on the `mcp>=1.0` extra (`pyproject.toml:27`) but choose different failure sites because of module shape, not because of the dependency.
- `import-time-failure-precedes-first-dependent-symbol-binding` — In `reasons/mcp_server.py:16-21`, the `except ImportError: raise` block appears before line 25 and before all `@mcp.tool()` decorators, ensuring no partial module body executes.
- `reasons-lib-shims-inherit-distinct-failure-modes` — `reasons_lib/mcp_server.py` re-exports the import-time-failing module and `reasons_lib/mcp_client.py` re-exports the call-time-deferring module, so the shims preserve the same failure sites as their sources.
