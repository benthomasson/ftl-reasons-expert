---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: propose_retract in reasons/mcp_server.py

# `propose_retract` — `reasons/mcp_server.py`

## Purpose

`propose_retract` is a **MCP tool wrapper** that exposes the belief-graph operation `api.propose_retraction` over the Model Context Protocol (MCP) stdio transport. It lets an MCP client (e.g. Claude Code, another LLM agent) record a **pending proposal** to retract a belief *without* changing the network's truth values.

Functionally, the wrapper is very thin — most behavior lives in `api.propose_retraction`:

- It resolves the DB path lazily via `_get_db()`.
- It forwards seven arguments plus `db_path` to the API layer.
- It serializes the API's response dict to a JSON string for the transport.
- It converts `KeyError` / `ValueError` raised by the API into a JSON `{"error": ...}` payload.

The decorator `@mcp.tool()` is what makes it callable over MCP; at import time this registers the function (and its docstring as the tool description, and its type hints as the JSON schema) with the `FastMCP` instance.

## Contract

**Preconditions**

- `target_id` names a node in the belief graph. If not, `api.propose_retraction` raises `KeyError` and the wrapper returns `{"error": ...}` — **no exception escapes**.
- The DB path resolved by `_get_db()` (via `REASONS_DB`, walk-up from cwd, or `api.DEFAULT_DB`) must be usable by the API layer.
- Argument values are semantically constrained but **not type-enforced** (see Assumptions).

**Postconditions**

- On success (no exception): the `proposals` table contains a new row with `status="pending"`, target `target_id`, the supplied metadata (`reason`, `failure_mode`, `basis`, `evidence`, `proposer`, `tags`), and a computed **cascade impact** snapshot. The belief's `IN`/`OUT` truth value in the belief graph is **unchanged**.
- The return value is *always* a JSON-encoded string; it is never `None`.

**Invariants (relative to the proposal lifecycle)**

- This function only creates proposals; it never accepts them. Truth values only change through `accept_proposal` → `api.accept_proposal`.
- Because proposals can go stale (target drifts before acceptance), a successful call here does not guarantee the change can later be applied; `accept_proposal` re-validates.

## Parameters

| Name | Type | Default | Meaning |
|---|---|---|---|
| `target_id` | `str` | *(required)* | Belief ID to propose retracting. Required by MCP schema. |
| `reason` | `str` | `""` | Human-readable justification for the retraction. |
| `failure_mode` | `str` | `""` | Categorical failure, e.g. `"outdated"`, `"incorrect"`. Free-form string at the wrapper level. |
| `basis` | `str` | `"prior-knowledge"` | Evidential basis. String by convention; API likely accepts a fixed set (`prior-knowledge`, `observation`, `inference`, …). |
| `evidence` | `str` | `""` | Supporting evidence text/URL. |
| `proposer` | `str` | `""` | Identity of the proposer. Empty means "anonymous" downstream. |
| `tags` | `list[str] \| None` | `None` | Access-control tags stored with the proposal. |

**Edge cases**

- `target_id` empty or unknown → `KeyError` from API → `{"error": "..."}` returned, nothing written.
- `tags=[]` vs `tags=None` may be treated differently by the API (empty list is a value, `None` is omission). The wrapper passes them through verbatim.
- `basis` / `failure_mode` not in the API's accepted set → `ValueError` from API → error JSON.
- `proposer=""` may be rejected or treated as anonymous depending on the API; the wrapper doesn't normalize.

## Return Value

- **Success path:** `json.dumps(result, indent=2)` where `result` is the dict returned by `api.propose_retraction`. The exact keys aren't visible in this file, but based on sibling APIs (`propose_update`, `propose_supersession`) it will include the proposal ID, the target, and cascade-impact information.
- **Failure path (caught):** `json.dumps({"error": str(e)})` — note this is **not** pretty-printed (`indent=2` omitted), a small inconsistency with the success path.

The caller (MCP client) receives a single text content block containing a JSON string and must `json.loads` / parse it. A malformed success payload that happens to include an `"error"` key from the API would be indistinguishable from a wrapper-level catch by shape alone.

## Algorithm

1. `@mcp.tool()` runs at import — registers `propose_retract` with `FastMCP`, deriving the JSON tool schema from the signature and docstring.
2. On invocation:
   1. Enter `try`.
   2. Evaluate `_get_db()` as the `db_path` keyword argument. On first call in the process (`_db is None`), this runs `_find_db()`: checks `REASONS_DB` env var, then walks parents of cwd looking for `reasons.db`, then falls back to `api.DEFAULT_DB`. The result is cached in module-global `_db`.
   3. Call `api.propose_retraction(target_id, reason=..., failure_mode=..., basis=..., evidence=..., proposer=..., tags=..., db_path=...)`.
   4. The API validates the target, computes the cascade impact of retracting it, and inserts a `pending` proposal row. Returns a dict.
   5. Serialize with `json.dumps(result, indent=2)` and return.
3. If `api.propose_retraction` raises `KeyError` or `ValueError`, jump to the `except` clause and return `json.dumps({"error": str(e)})`.
4. Any other exception type is **not** caught here and propagates to the MCP framework (which typically converts it into a protocol-level error response).

## Side Effects

- **Database write (only on success):** inserts one row into the `proposals` table — a new pending proposal with a cascade-impact snapshot.
- **Database read:** reads the belief graph to validate `target_id` and compute cascade.
- **Module-global mutation (once):** `_db` is set on first `_get_db()` call.
- **Filesystem I/O (once):** `_find_db()` stats files walking up from cwd.
- **No truth-value mutation:** the belief network's `IN`/`OUT` states are untouched.
- **No stdout/stderr writes** (MCP uses its own transport; logging would interfere).

## Error Handling

- **Caught and converted to JSON:** `KeyError` (unknown `target_id`, unknown antecedent, etc.), `ValueError` (invalid `basis` / `failure_mode` / arg combination).
- **Not caught (will surface as MCP error / Python exception):**
  - `sqlite3.Error` / `OSError` from DB access (locked DB, missing file, disk full).
  - `RuntimeError` from the API layer.
  - `TypeError` if the MCP client sends an unexpected type despite schema enforcement.
  - JSON serialization errors from `json.dumps(result)` if the API returns non-serializable objects.
- **Deliberate inconsistency:** `propose_update` in this same file catches `(KeyError, ValueError, RuntimeError, OSError)`; `propose_retract` narrows to `(KeyError, ValueError)`. Callers should not assume symmetry.

## Usage Patterns

Typical MCP flow (e.g. from Claude Code):

1. Agent calls `propose_retract(target_id="ansible-is-declarative", reason="...", failure_mode="outdated", basis="observation", evidence="...", proposer="claude")`.
2. Wrapper returns JSON including a `proposal_id`.
3. Agent/user calls `show_proposal(proposal_id)` to inspect cascade impact.
4. Later, someone calls `accept_proposal(proposal_id)` (which re-validates against drift) or `reject_proposal` / `withdraw_proposal`.

**Caller obligations**

- Parse the JSON and check for an `"error"` key even though the transport signals success (`isError=false`) — this function reports application errors as JSON, not protocol errors.
- Do not assume that a successful proposal will be accepted; drift can mark it stale.
- Treat `target_id`, `basis`, and `failure_mode` as enum-like values even though they are typed as `str`.

## Dependencies

- `reasons.api` — `api.propose_retraction` (core logic), `api.DEFAULT_DB`.
- `mcp.server.fastmcp.FastMCP` — provides `mcp` instance and `@mcp.tool()` registration. Missing dependency raises `ImportError` at module import (see the file header).
- `json` — serialization.
- Transitive (via `_get_db`): `os`, `os.environ`, `os.getcwd`, `os.path`.
- Test coverage: `tests/test_mcp_server.py` (likely invokes the tool as a plain function with a temp DB), `tests/test_api.py` (underlying `propose_retraction`).

## Assumptions Not Enforced by the Type System

- **`target_id` existence:** typed as `str`; validity only checked at runtime inside the API (`KeyError`).
- **Enumerated string values:** `basis` and `failure_mode` are effectively enums but typed as `str`; no `Literal[...]` constraint, so typos pass the call and fail inside the API.
- **`tags` item semantics:** typed `list[str]`; the meaning of tags (access control scope? audience? filters?) is convention only.
- **`proposer` identity model:** `""` default — how empty is interpreted (anonymous, reject, error) is API territory.
- **`_get_db()` result validity:** may return a path that doesn't exist yet or `api.DEFAULT_DB`; the wrapper assumes the API will create or fail deterministically.
- **JSON-serializability of the API return:** assumed; nothing verifies the `result` dict is JSON-safe.
- **The exceptions caught are sufficient:** `sqlite3.Error` / `RuntimeError` from the API will not be converted to a friendly error payload, contrary to what test authors may expect from the sibling `propose_update`.

## Topics to Explore

- [function] `reasons/api.py:propose_retraction` — The real logic: cascade impact computation, proposal row shape, and what validations actually raise `KeyError` vs `ValueError`.
- [function] `reasons/api.py:accept_proposal` — Drift re-validation path that determines whether a pending retraction can actually flip truth values.
- [file] `reasons/mcp_server.py` — The complete tool surface: note the inconsistent exception sets across `propose_*` wrappers and the `ident=2` vs compact error JSON difference.
- [file] `tests/test_mcp_server.py` — Shows how tools are exercised directly (bypassing the MCP transport) and what JSON shapes are asserted.
- [general] `mcp-tool-schema-derivation` — How `FastMCP` derives the client-facing JSON schema from annotations like `list[str] | None` and defaults, which governs what an LLM client can actually send.

## Beliefs

- `propose-retract-preserves-belief-truth-values` — The function only writes a pending proposal row; it never flips a belief's `IN`/`OUT` status.
- `propose-retract-returns-json-string-always` — Every code path returns a `str` of JSON; error cases return `{"error": "..."}` and never propagate `KeyError`/`ValueError`.
- `propose-retract-catches-narrower-exceptions-than-propose-update` — `propose_retract` catches only `(KeyError, ValueError)` while `propose_update` in the same file also catches `RuntimeError` and `OSError`.
- `propose-retract-requires-existing-target-id` — An unknown `target_id` raises `KeyError` inside `api.propose_retraction`; the wrapper does not pre-validate and converts it to an error JSON payload.
- `propose-retract-depends-on-mcp-tool-decorator` — The `@mcp.tool()` decorator is what registers the function on the `FastMCP` instance at import time; without it, the function would not be reachable over the MCP stdio transport.
