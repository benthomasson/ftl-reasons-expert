---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/mcp_server.py

# `reasons/mcp_server.py`

## 1. Purpose

This file is the **MCP (Model Context Protocol) server** for the ftl-reasons belief network. Its sole job is to act as a thin, stdio-transported RPC layer that exposes the domain operations in `reasons/api.py` as named tools that an LLM agent (specifically Claude Code) can invoke. It owns:

- The wire protocol contract (function signatures, parameter types, return string formats) between the MCP client and the belief-network API.
- Database path discovery and lazy resolution.
- Error serialization (converting Python exceptions into `{"error": "..."}` JSON strings so the LLM gets a recoverable message rather than a crash).

It contains **zero domain logic**. Every tool body is a one-or-two-line delegation into `api.*`, with a try/except wrapper.

## 2. Key Components

### Server instance & DB resolution

| Component | Contract |
|---|---|
| `mcp = FastMCP("reasons")` | Singleton server; all `@mcp.tool()` decorators register handlers on it. |
| `_db: str \| None` | Module-level cache. `None` until first resolution. |
| `_find_db() -> str` | Resolution order: `REASONS_DB` env var → walk up from `cwd` looking for `reasons.db` → `api.DEFAULT_DB`. Mirrors the CLI's discovery logic. |
| `_get_db() -> str` | Lazy singleton. Calls `_find_db()` once, caches in `_db`. Every tool calls this, so the path is resolved at most once per process. |

### Tools (organized in 4 tiers)

**Tier 1 — Core (read/write the network directly):**

- `search(query, output_format, depth, include_out, limit, offset)` → `str`
- `show(node_id)` → `str` (JSON)
- `explain(node_id)` → `str` (JSON)
- `list_beliefs(status, premises_only, namespace, limit, offset, output_format)` → `str`
- `add(node_id, text, sl, unless, label)` → `str` (JSON)
- `retract(node_id, reason)` → `str` (JSON)
- `assert_belief(node_id)` → `str` (JSON)

**Tier 2 — Reasoning (simulate / introspect):**

- `what_if(node_id, action, text, new_id)` → dispatches to `api.what_if_retract/assert/supersede` based on the `action` string.
- `add_justification(node_id, sl, unless, label)` → adds a second justification to an existing node.
- `nogood(node_ids)` → records a contradiction, triggers backtracking.
- `trace(node_id)` → walks to root premises.
- `compact(budget, include_out, output_format)` → token-budgeted network summary.

**Tier 3 — Proposals (staged changes with review workflow):**

- `propose_update(...)` → LLM-driven review + optional store.
- `propose_retract(target_id, ...)` → staged retraction.
- `propose_supersede(old_id, new_text, ...)` → staged replacement.
- `propose_add(node_id, text, ...)` → staged new-belief.
- `propose_nogood(node_ids, ...)` → staged contradiction.
- `list_proposals(...)`, `show_proposal(...)`, `accept_proposal(...)`, `reject_proposal(...)`, `withdraw_proposal(...)` → proposal lifecycle.

**Tier 4 — Data management:**

- `status(output_format)` → network overview.
- `list_gated(output_format)` → OUT beliefs blocked by outlist gates.
- `export_markdown()` → full `beliefs.md` dump.
- `topics(limit)` → topic clusters from IDs.

### Entry point

```python
def main():
    global _db
    _db = _find_db()   # eager resolve so the first tool call doesn't pay discovery cost
    mcp.run()          # starts the stdio loop; blocks until client disconnects
```

## 3. Patterns

- **Thin facade / proxy over a domain API.** Every tool is `return api.something(args..., db_path=_get_db())`. The MCP layer adds no business logic; it only serializes errors and adapts parameter conventions (e.g. `limit: int = 0` → `limit or None` so 0 means "no limit").

- **Lazy singleton for configuration.** `_db` starts as `None`; `_get_db()` does the one-time discovery. `main()` can also set it eagerly before `mcp.run()`.

- **Tiered API surface.** The `# --- Tier N ---` comments partition tools by blast radius: Tier 1 mutates immediately, Tier 2 simulates or introspects, Tier 3 stages changes for review, Tier 4 is read-only reporting. This is a *soft* convention for the LLM consumer, not an enforcement mechanism.

- **String-only returns.** MCP tool handlers must return a string. So every tool either returns a JSON-dumped string or a markdown string. There is no structured return type at the MCP boundary.

- **Error-as-data.** Exceptions are caught and converted to `{"error": "..."}` JSON strings. The LLM sees a JSON object it can parse and react to, rather than the MCP connection dying.

- **Falsy-to-None idiom.** `limit or None`, `status or None`, `namespace or None` — the MCP layer uses `0`/`""` as "unspecified" sentinels (natural for LLM-generated JSON) and converts them to `None` for the internal API which uses `None` to mean "no filter / no limit."

## 4. Dependencies

**Imports (what this file needs):**

| Dependency | Role |
|---|---|
| `mcp.server.fastmcp.FastMCP` | MCP server framework; provides `@mcp.tool()` decorator and `mcp.run()`. Gated behind a try/except that raises a helpful `ImportError` pointing to `pip install 'ftl-reasons[mcp]'`. |
| `reasons.api` | The entire domain API. Every tool delegates here. |
| `json`, `os` | Serialization and path walking. |

**Imported by (what needs this file):**

| Importer | Why |
|---|---|
| `reasons_lib/mcp_server.py` | The `reasons_lib` package mirrors the `reasons` package (likely a vendored/legacy copy). It either re-exports or duplicates this module. |
| `tests/test_mcp_server.py` | Unit/integration tests that exercise the tool functions. |

The `reasons-mcp` console script (defined in `pyproject.toml`) points at `reasons.mcp_server:main`.

## 5. Flow

```
Client (Claude Code)
    │  stdio JSON-RPC
    ▼
mcp.run()  ← FastMCP stdio loop
    │
    │  dispatch by tool name
    ▼
@mcp.tool() handler (e.g. search, add, accept_proposal)
    │
    │  1. _get_db() → resolved path (cached after first call)
    │  2. Call api.<method>(..., db_path=_get_db())
    │  3. Wrap result: json.dumps(result, indent=2) or raw string
    │  4. On exception: json.dumps({"error": str(e)})
    ▼
String response → back over stdio → LLM reads it
```

There is no concurrency concern in the handler itself — FastMCP's stdio loop is single-threaded per session. The `_db` global is set once in `main()` (or lazily on first tool call) and never mutated afterward, so there's no race.

## 6. Invariants

- **All tool return types are `str`.** No tool returns a dict, list, or raises to the MCP framework. This is required by the `FastMCP` tool contract.
- **`limit=0` means "no limit."** This is a non-obvious convention. The internal `api` functions treat `None` as "no limit," so the MCP layer translates `0 → None` via `limit or None`. If you change this, you must update both the docstring and the conversion.
- **`""` (empty string) means "no filter"** for optional string params (`status`, `namespace`, `reason`, `proposer`, etc.). Again translated to `None` before calling `api`.
- **Error JSON always has exactly one key: `"error"`.** The LLM can check `if "error" in parsed_json` to detect failure. No `status` field, no `code` field.
- **`node_id` is the universal identifier.** Every tool that targets a specific belief takes a `node_id` (or `target_id` / `old_id` in proposals). It is a slug-like string, not an integer PK.
- **`db_path` is always passed to `api.*`.** There is no module-level default in `api` that the MCP server relies on; the path is always threaded through explicitly.

## 7. Error Handling

The pattern is uniform across all ~25 tools:

```python
try:
    result = api.something(...)
    return json.dumps(result, indent=2)
except (KeyError, ValueError) as e:
    return json.dumps({"error": str(e)})
```

| Exception type | Meaning | Example |
|---|---|---|
| `KeyError` | Node/proposal not found | `show("nonexistent")` |
| `ValueError` | Invalid argument or state | `add` with duplicate ID, `propose_supersede` with empty `new_text` |
| `RuntimeError`, `OSError` | LLM call failure, file I/O | `propose_update` when the LLM is unreachable |
| `ImportError` | Missing `mcp` package | Raised at module import time, not per-call |

**Notable:** `compact`, `search`, `list_beliefs`, `status`, `list_gated`, `export_markdown`, and `topics` have **no try/except**. They rely on the `api` functions not raising (or they accept that an unhandled exception will propagate to the MCP framework, which will surface it as a protocol-level error to the client). This is a slight inconsistency — if `api.compact()` ever raises, the LLM gets a raw MCP error instead of a friendly JSON object.

**`what_if`** is the only tool that does **conditional dispatch** inside the try block (switching on `action` string to pick which `api.what_if_*` to call), and it has its own validation (`text` required for `supersede`).

---

## Topics to Explore

- [file] `reasons/api.py` — The entire domain API this file delegates to. Understanding `api.search`, `api.add_node`, `api.accept_proposal`, etc. is prerequisite to understanding what each MCP tool actually does.
- [file] `reasons/mcp_client.py` — The client-side counterpart. Shows how an LLM agent *calls* these tools programmatically, which clarifies the wire contract from the other side.
- [file] `tests/test_mcp_server.py` — The test suite for this module. Reveals expected parameter edge cases (limit=0, empty strings, missing nodes) and the exact JSON shapes returned.
- [general] `mcp-protocol-spec` — The MCP (Model Context Protocol) specification defines how `FastMCP` serializes tool schemas, handles stdio framing, and reports errors. Knowing the spec explains why all returns must be strings and why the `@mcp.tool()` decorator generates a JSON schema from type hints.
- [file] `reasons/propose_update.py` — The LLM-driven review pipeline that `propose_update` triggers. Shows the non-trivial logic (sampling, model calls, proposal construction) that the MCP layer simply proxies.

---

## Beliefs

- `mcp-server-returns-only-strings` — Every `@mcp.tool()` handler in `mcp_server.py` returns a `str` (JSON-dumped or markdown); no tool raises an unhandled exception to the MCP framework, except the Tier-4 read tools (`compact`, `search`, `status`, `list_gated`, `export_markdown`, `topics`) which lack try/except guards.
- `mcp-server-is-thin-proxy-over-api` — `mcp_server.py` contains no belief-network logic; every tool body is a direct call into `reasons.api` with `db_path` threaded explicitly. All domain invariants (cascade propagation, nogood backtracking, drift re-validation) live in `api.py`.
- `db-path-resolved-once-per-process` — `_db` is a module-level global set either eagerly in `main()` or lazily on first `_get_db()` call; it is never re-resolved, so changing `REASONS_DB` at runtime has no effect on a running server.
- `limit-zero-means-unlimited` — The MCP layer uses `0` as the "no limit" sentinel for `limit` parameters (matching LLM-friendly integer defaults) and translates it to `None` before calling `api`, which uses `None` as its no-limit sentinel.
- `error-json-has-single-error-key` — All caught exceptions are serialized as `{"error": "<message>"}` — a single-key JSON object — giving the LLM consumer a uniform, parseable failure contract distinct from success responses.
