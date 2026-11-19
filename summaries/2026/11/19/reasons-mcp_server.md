---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/mcp_server.py

# `reasons/mcp_server.py`

## Purpose

This file is the **MCP (Model Context Protocol) server adapter** for the `reasons` project. It wraps the public API in `reasons.api` and exposes it as a set of tools discoverable by MCP-compatible clients (e.g., Claude Code) over **stdio**.

It owns exactly one responsibility: **translate LLM/client tool calls into `reasons.api` calls**, serialize the results as JSON strings, and return structured errors. All belief-network logic lives in `reasons.api` — this module is a thin, tiered facade plus a process entry point.

The module docstring documents the three deployment shapes:

- `reasons-mcp` (auto-discover `reasons.db`)
- `REASONS_DB=/path/to/reasons.db reasons-mcp`
- Registration with Claude Code via `claude mcp add reasons -- reasons-mcp`

## Key Components

### Module-level state

| Symbol | Purpose |
|---|---|
| `mcp = FastMCP("reasons")` | The server instance. All `@mcp.tool()` decorators register functions onto this object. The string `"reasons"` is the advertised server name. |
| `_db: str | None = None` | Lazily-cached resolved database path. Populated on first tool call, or eagerly by `main()`. | 

### DB resolution

- **`_find_db() -> str`** — Resolves the DB path in this precedence:
  1. `REASONS_DB` env var (if set and non-empty).
  2. Walk up from `os.getcwd()` looking for a `reasons.db`, stopping at filesystem root (`parent == d`).
  3. Fall back to `api.DEFAULT_DB`.
  The docstring explicitly notes this "matches CLI behavior" — the CLI and MCP server discover the DB identically so they see the same network.

- **`_get_db() -> str`** — Memoizes `_find_db()` into `_db`. Every tool calls `_get_db()`, so resolution happens at most once per process; a changed `REASONS_DB` after startup is *not* picked up.

### Tools (grouped by tier)

**Tier 1 — Core (read/write truth values):**
- `search(query, output_format, depth, include_out, limit, offset)`
- `show(node_id)`
- `explain(node_id)`
- `list_beliefs(status, premises_only, namespace, limit, offset, output_format)`
- `add(node_id, text, sl, unless, label)`
- `retract(node_id, reason)`
- `assert_belief(node_id)`

**Tier 2 — Reasoning (simulation / derivation):**
- `what_if(node_id, action, text, new_id)` — dispatches to `what_if_assert`, `what_if_supersede`, or `what_if_retract`
- `add_justification(node_id, sl, unless, label)`
- `nogood(node_ids)` — triggers TMS backtracking
- `trace(node_id)`
- `compact(budget, include_out, output_format)`

**Tier 3 — Proposals (deferred mutations):**
- `propose_update(...)` — LLM-driven review; also optionally calls `api.store_llm_proposals` (see flow below)
- `propose_retract`, `propose_supersede`, `propose_add`, `propose_nogood`
- `list_proposals`, `show_proposal`
- `accept_proposal`, `reject_proposal`, `withdraw_proposal`

**Tier 4 — Data management:**
- `status(output_format)`
- `list_gated(output_format)`
- `export_markdown()`
- `topics(limit)`

### Entry point

- **`main()`** — Resolves `_db` eagerly, then calls `mcp.run()` (blocking stdio loop). Used by the `reasons-mcp` console script and by `__main__`.

## Patterns

1. **Decorator-based tool registration.** Every public function is decorated with `@mcp.tool()`. FastMCP derives the tool name from the function name and the schema from the signature + docstring `Args:` block. Docstrings are the tool descriptions the LLM sees, so they are load-bearing, not documentation.

2. **Tiered facade.** The `# --- Tier N: ... ---` comments mirror the project's mental model: *core* → *reasoning* → *proposals* → *data*. A maintainer adding a tool can pick the appropriate tier immediately.

3. **Lazy singleton.** `_db` + `_get_db()` is a classic lazy-init pattern — avoids a filesystem walk at import time (important when the module is imported by tests that set `REASONS_DB` later).

4. **"Empty string → None" normalization.** Tools use `status=status or None`, `namespace=namespace or None`, `target_id=target_id or None`, `limit=limit or None`. The MCP arg parser produces the declared type's default (often `""`), so tools must coalesce falsy values to `None` before passing to `api`.

5. **"Return-and-never-raise" error convention.** Instead of allowing exceptions to bubble to FastMCP, mutating tools catch known exception types and return `json.dumps({"error": ...})`. This is a deliberate contract: an LLM client should see a structured error string, not a protocol-level tool failure.

6. **Optional dependency guard.** The `try/except ImportError` around `from mcp.server.fastmcp import FastMCP` re-raises with an install hint (`pip install 'ftl-reasons[mcp]'`), reserving the dependency for users who want the server.

## Dependencies

**Imports (upstream):**
- `json`, `os` — stdlib; JSON serialization and env/`cwd` resolution.
- `mcp.server.fastmcp.FastMCP` — optional runtime dep (extra `[mcp]`).
- `reasons.api` — the sole business-logic dependency. Every tool is a one-liner over an `api.*` call. This means `api` is the single seam a test needs to mock or a refactor needs to preserve.

**Imported by (downstream):**
- `reasons_lib/mcp_server.py` — appears to be a parallel/redistributed copy of the package tree; treat `reasons/mcp_server.py` as canonical.
- `tests/test_mcp_server.py` — the primary contract test for tool behavior.
- `tests/test_ask_mcp.py` — exercises the `search`/`ask`-style flows via MCP.

## Flow

**Startup:**
```
import module
  → import FastMCP (fail fast if [mcp] extra missing)
  → mcp = FastMCP("reasons")
  → @mcp.tool() decorators run, registering all tool functions
  → (if __main__) main()
      → _db = _find_db()            # eager resolution
      → mcp.run()                    # blocking stdio loop
```

**Per tool call:**
```
client → MCP JSON-RPC over stdio
  → FastMCP validates args against function signature
  → tool body runs:
      _get_db()                      # cached after first call
      api.<fn>(..., db_path=...)
      return json.dumps(result[, indent=2])
  → FastMCP wraps response as MCP tool result
```

**`propose_update` branch (the one non-trivial tool):**
1. Call `api.propose_update(...)` → returns `{reviewed, proposals, timestamp}`.
2. Build a summary `{reviewed, llm_proposals, timestamp}`.
3. If `store` is true and proposals exist: call `api.store_llm_proposals(result, proposer=..., db_path=...)` and enrich output with `stored`, `skipped`, `stored_count`.
4. Otherwise, return the raw proposals inline under `"proposals"`.

This is the only tool that *chains two* API calls and conditionally reshapes output. Anything added here should follow the same idiom: keep `db_path` forwarding and JSON wrapping consistent.

## Invariants

- **`_db` resolves once.** After the first `_get_db()` call (or `main()`), changing `REASONS_DB` or the working directory does not change the DB used by subsequent calls in the same process. Tests must set `REASONS_DB` *before* the first call or use the `_db` global directly.
- **Explicit `REASONS_DB` wins.** Even if a `reasons.db` exists higher up, an env-set path overrides it.
- **`_find_db` matches `reasons.cli`.** If the CLI's discovery algorithm changes, this function must change too; otherwise `reasons` and `reasons-mcp` can drift onto different databases.
- **No mutation outside `api`.** Every tool is `return api.X(..., db_path=_get_db())` (plus JSON/format wrapping). Direct DB access here would violate the facade boundary.
- **`output_format` enums drive formatting, not the API.** For `search` and `compact`, `output_format` is forwarded into `api` (they own markdown/minimal formatting). For `list_beliefs`, `status`, and `list_gated`, `"names-only"` is handled *locally* — the API always returns the JSON structure and this module strips it down.
- **`what_if` requires `text` for `supersede`.** Enforced inside the tool (`if not text: return error`) — it is a client-facing validation the API does not necessarily repeat.

## Error Handling

Error handling is deliberately **uneven per tool**, and that unevenness is a contract worth knowing:

| Tool group | Caught exceptions | Behavior |
|---|---|---|
| `show`, `explain`, `retract`, `assert_belief`, `trace` | `KeyError` | `{"error": "Node '<id>' not found"}` |
| `what_if` | `KeyError` | `{"error": "Node '<id>' not found"}` |
| `add`, `add_justification`, `nogood`, `propose_retract`, `propose_supersede`, `propose_add`, `propose_nogood`, `accept_proposal`, `reject_proposal`, `withdraw_proposal` | `KeyError, ValueError` | `{"error": str(e)}` |
| `propose_update` | `KeyError, ValueError, RuntimeError, OSError` | `{"error": str(e)}` |
| `search`, `list_beliefs`, `compact`, `list_proposals`, `status`, `list_gated`, `export_markdown`, `topics` | **none** | Exceptions propagate to FastMCP as tool-level failures |

Notes:
- `nogood` uses `except KeyError as e` and returns `str(e)` (not the "not found" templated message) — small inconsistency with neighbors.
- Read-only tools intentionally let exceptions escape: they are expected to fail only on infrastructure problems (missing DB file, bad connection), where a protocol-level error is preferable to a JSON blob.
- The JSON-error convention **does not distinguish "not found" from "invalid"** to an LLM beyond the message text; callers must parse `error`.
- `main()` does not guard `_find_db()` — if no `reasons.db` is present anywhere up the tree, the server starts against `api.DEFAULT_DB` rather than failing fast.

## Topics to Explore

- [file] `reasons/api.py` — The real contract behind every tool; verify `search` vs `list_nodes` result shapes, and which `format` values each function accepts.
- [function] `reasons/cli.py:_find_db` — Confirm the "matches CLI behavior" claim in `_find_db`; any divergence creates silent DB drift between `reasons` and `reasons-mcp`.
- [file] `tests/test_mcp_server.py` — Shows exactly which tools are tested, how MCP responses are captured, and whether `_db`/env handling is reset between tests.
- [general] `mcp-fastmcp-tool-schema` — How FastMCP derives tool schemas from docstrings and type hints, and what happens when a signature uses `list[str] | None` defaults.
- [file] `reasons/propose_update.py` — The two-step `propose_update` → `store_llm_proposals` flow and the drift-validation logic that `accept_proposal` relies on.

## Beliefs

- `mcp-server-import-requires-mcp-extra` — Importing `reasons.mcp_server` raises `ImportError` if the `mcp` package is missing, directing users to install `ftl-reasons[mcp]`.
- `mcp-server-delegates-all-mutations-to-api` — Every tool body calls a `reasons.api.*` function; no tool accesses the database directly.
- `reasons-db-env-overrides-walk-up-discovery` — `_find_db` returns `REASONS_DB` if set, otherwise walks up from cwd for `reasons.db`, otherwise falls back to `api.DEFAULT_DB`.
- `mcp-read-tools-propagate-exceptions` — `search`, `list_beliefs`, `compact`, `list_proposals`, `status`, `list_gated`, `export_markdown`, and `topics` have no try/except and let exceptions surface to FastMCP, unlike mutating tools which return `{"error": ...}`.
- `mcp-names-only-handled-locally-for-lists` — `list_beliefs`, `status`, and `list_gated` implement `output_format="names-only"` by post-processing API results, whereas `search` and `compact` forward `output_format` into the API.
