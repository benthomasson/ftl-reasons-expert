---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: propose_update in reasons/mcp_server.py

# `propose_update` — MCP Tool for LLM-Driven Belief Review

## 1. Purpose

`propose_update` is an **MCP tool** (decorated with `@mcp.tool()`) that exposes the `reasons` library's LLM-driven belief review pipeline over the Model Context Protocol. It is the automated counterpart to the other proposal tools in the same file (`propose_retract`, `propose_supersede`, `propose_add`, `propose_nogood`), which accept explicit human/agent-supplied proposal arguments. Instead of the caller specifying *what* to propose, `propose_update` sends the current belief set to an LLM and lets the LLM decide which beliefs should be retracted or superseded. It then optionally persists those LLM-generated proposals into the `proposals` table for later review.

The function exists so that an agent (e.g., Claude Code) can trigger a semantic audit of the belief network with a single tool call, rather than enumerating each belief and crafting proposals by hand.

## 2. Contract

**Preconditions**
- A reachable SQLite database path must be resolvable via `_get_db()` (env var `REASONS_DB`, or `reasons.db` found by walking up from `cwd`, or `api.DEFAULT_DB`).
- The `mcp` package must be importable (enforced at module import time).
- If `belief_ids` is provided, the IDs must be valid keys in the beliefs table — otherwise `api.propose_update` raises `KeyError`.

**Postconditions**
- The function **always returns a `str`** containing a JSON document serialized with `indent=2`.
- On success, the JSON contains at minimum `reviewed` (int), `llm_proposals` (int), and `timestamp`.
- If `store=True` **and** at least one proposal was produced, the JSON additionally contains `stored`, `skipped`, and `stored_count`; the raw `proposals` array is **omitted**.
- If `store=False` **or** zero proposals were produced, the JSON contains `proposals` (array); `stored`/`skipped`/`stored_count` are **absent**.
- On a caught error, the JSON is exactly `{"error": "<message>"}`.
- All proposals that were persisted have `proposer=<proposer>` recorded on them (passed through `api.store_llm_proposals`).

**Invariants / assumptions not enforced by the type system**
- `result` (from `api.propose_update`) is a mapping with keys `"reviewed"`, `"proposals"`, and `"timestamp"`.
- `result["proposals"]` is a list (must support `len()`).
- `store_result` (from `api.store_llm_proposals`) has keys `"stored"`, `"skipped"`, `"count"`.
- The function never raises for the enumerated exception classes — callers do not need to wrap it in try/except for `KeyError`, `ValueError`, `RuntimeError`, or `OSError`.
- `belief_ids`, when non-None, contains IDs that exist in the DB.

## 3. Parameters

| Parameter | Type | Default | Meaning / Notes |
|---|---|---|---|
| `belief_ids` | `list[str] \| None` | `None` | Specific beliefs to evaluate. **Falsy normalization:** `belief_ids or None` means an empty list `[]` is silently converted to `None`, which the docs say means "all IN beliefs." So a caller cannot pass `[]` to mean "evaluate nothing." |
| `model` | `str` | `"claude"` | Passed straight to `api.propose_update`. Presumably a symbolic model selector resolved by the `reasons` library. |
| `stale_only` | `bool` | `False` | Passed through unchanged. Only evaluate beliefs already flagged stale (stale flag is a DB property, not inferred here). |
| `namespace` | `str` | `""` | Normalized via `namespace or None`. An empty string means "all namespaces" from the api's perspective, but the api receives `None`, not `""`. |
| `sample` | `int` | `0` | Normalized via `sample or None`. `0` (and any falsy int) becomes `None` = "all." Since negative ints are truthy, `sample=-1` would be passed through — behavior depends on `api.propose_update`. |
| `proposer` | `str` | `"llm"` | Only used when storing. Written onto each stored proposal row. |
| `store` | `bool` | `True` | Gate on whether to persist to the proposals table. Note that storing only happens if `store` is truthy **and** `result["proposals"]` is non-empty. |

## 4. Return Value

Always a **JSON-formatted string** (not a dict). Three possible shapes:

**A. Successful run, proposals stored** (`store=True` and `len(proposals) > 0`):
```json
{
  "reviewed": 12,
  "llm_proposals": 3,
  "timestamp": "...",
  "stored": 3,
  "skipped": 0,
  "stored_count": 3
}
```

**B. Successful run, not stored (or nothing to store)**:
```json
{
  "reviewed": 12,
  "llm_proposals": 0,
  "timestamp": "...",
  "proposals": []
}
```
or when `store=False` with proposals present:
```json
{
  "reviewed": 12,
  "llm_proposals": 3,
  "timestamp": "...",
  "proposals": [ ... raw proposal objects ... ]
}
```

**C. Caught error**:
```json
{"error": "Node 'foo' not found"}
```

The caller must:
- `json.loads` the string.
- Check for an `"error"` key before trusting the shape.
- Understand that `proposals` is only present when nothing was persisted; the "stored" path reports counts but **not** the raw proposals.
- Note `stored` and `stored_count` are both present on the stored path — `stored` comes from `store_result["stored"]` while `stored_count` is `store_result["count"]`. They may differ if `store_llm_proposals` semantics differ from what the naming implies (worth checking in `api.store_llm_proposals`).

## 5. Algorithm

1. **Normalize inputs** and delegate the actual LLM review to `api.propose_update(...)`:
   - `belief_ids or None` (empty list → `None`)
   - `namespace or None` (empty string → `None`)
   - `sample or None` (0 → `None`)
   - `db_path=_get_db()` (~lazy DB discovery with caching in module-level `_db`)
2. **Build a compact summary** into `output`:
   - `reviewed` = `result["reviewed"]`
   - `llm_proposals` = `len(result["proposals"])` (int count, not the array)
   - `timestamp` = `result["timestamp"]`
3. **Branch on persistence**:
   - If `store` is truthy AND `result["proposals"]` is non-empty: call `api.store_llm_proposals(result, proposer=proposer, db_path=_get_db())` and copy `stored`, `skipped`, and `count` (as `stored_count`) onto `output`. Note the full proposals are **not** re-attached to `output`.
   - Else: attach the raw `result["proposals"]` array to `output["proposals"]`.
4. **Serialize** with `json.dumps(output, indent=2)` and return.
5. **On exception**, fall into the `except (KeyError, ValueError, RuntimeError, OSError)` clause and return `json.dumps({"error": str(e)})`.

## 6. Side Effects

- **External LLM call**: `api.propose_update` is documented as sending beliefs to an LLM — this is the primary I/O. Any failure in that path surfaces as one of the caught exception types.
- **Database reads**: reads beliefs and their metadata.
- **Database writes**: if `store=True` and there are proposals, `api.store_llm_proposals` inserts rows into the `proposals` table. This is the only write the tool performs.
- **Module-global state**: `_get_db()` lazily sets the module-level `_db` global on first call. Subsequent calls reuse it (so a `REASONS_DB` change mid-process has no effect after first resolution).
- **No file writes**, no network outside the LLM call, no mutation of in-memory belief state.

## 7. Error Handling

**Caught and converted to `{"error": ...}`**:
- `KeyError` — e.g., invalid `belief_ids`, missing expected keys in `result`/`store_result` (`"reviewed"`, `"proposals"`, `"timestamp"`, `"stored"`, `"skipped"`, `"count"`).
- `ValueError` — typically bad argument shapes or ambiguous beliefs in the api layer.
- `RuntimeError` — engine/runtime failures (e.g., DB locked, LLM backend unavailable).
- `OSError` — file-system and DB-connection failures.

**Not caught (will propagate to the MCP framework as a tool error)**:
- `AttributeError` / `TypeError` from unexpected shapes (e.g., `result["proposals"]` being `None` would raise `TypeError` on `len()`, which is **not** caught).
- Any exception from the `mcp` runtime itself.
- `ImportError` is prevented at module load.

**Important subtlety:** `str(e)` is returned as the error message. If `e` is a `KeyError`, `str(e)` renders the quoted key (e.g., `"'reviewed'"`), which can be cryptic. There's no stack trace exposure — good for MCP clients, but limits debugging.

## 8. Usage Patterns

**Typical call from an MCP client** (Claude Code, etc.):
```json
{
  "tool": "reasons_propose_update",
  "arguments": {
    "namespace": "infra",
    "stale_only": true,
    "store": true,
    "proposer": "audit-agent-42"
  }
}
```

**Read-only preview** (do not persist):
```python
propose_update(belief_ids=["ansible-is-declarative"], store=False)
```
Caller is obligated to inspect the returned `proposals` array — the database is untouched in this mode.

**Programmatic batch audit**:
```python
out = propose_update(namespace="infra", sample=20, proposer="nightly-cron")
data = json.loads(out)
if "error" in data:
    alert(data["error"])
elif data["llm_proposals"] > 0:
    notify_reviewer(data["stored_count"])
```

**Caller obligations**:
- Do not assume `proposals` is present after `store=True` — use `list_proposals(status="pending")` to retrieve persisted rows.
- Do not assume the tool never fails — check for the `error` key.
- Do not pass an empty `belief_ids=[]` expecting `"no beliefs reviewed"`; it will be treated as "all IN beliefs."
- Trust the DB path resolution once per process; `REASONS_DB` changes after first call are ignored (see `_get_db`'s caching).

## 9. Dependencies

- `reasons.api.propose_update` — the actual LLM review entry point; shape of its return dict is a hard contract.
- `reasons.api.store_llm_proposals` — persistence; assumes the returned dict includes `stored`, `skipped`, `count`.
- `reasons.api.DEFAULT_DB` — fallback when `_find_db` fails.
- `reasons.mcp_server._get_db` / `_find_db` — lazy+cached DB path resolution (env var, cwd walk, fallback).
- `mcp.server.fastmcp.FastMCP` — provides the `@mcp.tool()` decorator that turns this into a JSON-typed MCP tool.
- `json` (stdlib) — serialization with `indent=2`.

## Topics to Explore

- [function] `reasons/mcp_server.py:_get_db` — Understand the caching and discovery semantics that this tool inherits implicitly; a mis-set `REASONS_DB` will silently follow the wrong file for the entire process.
- [file] `tests/test_propose_update.py` — Pins down the exact JSON shape contract (e.g., whether `stored_count` equals `stored`, and how `sample`/`namespace` normalizations are asserted).
- [function] `reasons/mcp_server.py:propose_retract` — The manual counterpart to `propose_update`; contrasting the two clarifies what the LLM-driven path automates.
- [general] `store-llm-proposals-shape` — The names `stored`, `skipped`, `count` being mapped to `stored`, `skipped`, `stored_count` suggests a subtle API mismatch worth verifying in the api layer.
- [file] `reasons/mcp_server.py` — The other proposal tools (`propose_supersede`, `propose_add`, `propose_nogood`) share the same JSON-in/JSON-out convention and error handling; useful for confirming the pattern is intentional.

## Beliefs

- `propose-update-returns-json-always` — `propose_update` always returns a `str` of JSON, including on caught errors, where it returns `{"error": ...}`.
- `propose-update-catches-key-value-runtime-oserr` — Only `KeyError`, `ValueError`, `RuntimeError`, and `OSError` are converted to error JSON; other exceptions (e.g., `TypeError` from `len(None)`) propagate to the MCP layer.
- `propose-update-empty-list-becomes-none` — `belief_ids=[]` is normalized by `belief_ids or None` to `None`, meaning "all IN beliefs," not "no beliefs."
- `propose-update-omits-proposals-when-stored` — When `store=True` and proposals exist, the raw `proposals` array is replaced by `stored`/`skipped`/`stored_count` counts in the output.
- `propose-update-requires-store-llm-proposals-shape` — Success of the `store=True` branch assumes `api.store_llm_proposals` returns a mapping with keys `"stored"`, `"skipped"`, and `"count"`; a missing key raises `KeyError` which is caught and turned into an error JSON.
- `propose-update-uses-lazy-cached-db` — The `db_path` is resolved by `_get_db()`, which caches the first resolution in a module-level `_db` global, so `REASONS_DB` changes after the first call are ignored.
