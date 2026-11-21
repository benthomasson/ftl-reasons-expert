---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: sync_agent in reasons/api.py

## `sync_agent`

### Purpose

`sync_agent` is the top-level functional API for **reconciling a remote agent's belief store into the local RMS database** ("remote wins"). Unlike `import_agent` (which appends/merges), sync is *authoritative*: it walks the incoming beliefs and, for each node already present under `agent_name:`, updates the text/justifications to match the remote file; adds nodes that are missing; and retracts nodes that exist locally but are absent from the remote file.

It exists so CLI, LangGraph tools, and scripts can invoke a full sync as a single function call without dealing with `Storage` lifecycle, argparse, or backend selection. It transparently dispatches to PostgreSQL (`PgApi`) when `pg_conninfo` is supplied, and otherwise operates on the SQLite-backed `Network` via `_with_network`.

### Contract

**Preconditions**
- `beliefs_file` must exist on disk. The function does not create directories or infer paths.
- If `nogoods_file` is provided (markdown path only), it must exist.
- `db_path` must point to a valid RMS database (or an initializable one) — the storage layer's `Storage(db_path)` will create/load as required.
- `agent_name` should be a bare namespace string; it becomes the `agent_name:` prefix on every imported node.

**Postconditions (on success)**
- Local namespace `agent_name:` matches the state described by `beliefs_file`:
  - New nodes added,
  - Existing nodes' text/justifications replaced,
  - Nodes not present in file are retracted (marked OUT via the namespace premise mechanism).
- A `sync_agent` result dict is returned with counts and the agent name.
- For SQLite, the write is committed atomically inside `_with_network.__exit__` — either the whole sync lands or nothing does.

**Invariants**
- The database is written only if the entire sync completes without raising (SQLite path). On exception, `_with_network.__exit__` sees `exc_type is not None` and **skips the save**, so partial state cannot be persisted.
- The remote file is authoritative — "remote wins" is the semantic contract.
- Nodes remain namespaced under `agent_name:`; no cross-namespace leakage occurs.

### Parameters

| Parameter | Type | Meaning | Notes |
|---|---|---|---|
| `agent_name` | `str` | Namespace prefix for all imported nodes (`<agent_name>:<id>`) | Becomes the `ns:active` premise's namespace. Should not contain `:` (would create nested prefixes unexpectedly). |
| `beliefs_file` | `str` | Path to a `beliefs.md` or `network.json` file | Dispatch key: `.json` suffix triggers the JSON path, anything else falls back to markdown. Suffix matching is **case-sensitive** (`X.JSON` → markdown path). |
| `nogoods_file` | `str \| None` | Optional separate nogoods markdown file | **Ignored on the JSON path.** Also ignored if empty string (falsy): the code checks `if nogoods_file:` so `""` triggers auto-discovery. |
| `only_in` | `bool` | If `True`, sync only beliefs whose truth value is IN (skip OUT beliefs) | Only affects markdown normalization (JSON normalizer also honors it, but "remote wins" reconciliation itself is truth-value-agnostic). |
| `db_path` | `str` | SQLite DB path | Default `reasons.db`, relative to CWD. Unused when `pg_conninfo` is set. |
| `pg_conninfo` | * | If truthy, use PostgreSQL backend | Empty string is falsy → uses SQLite. |
| `domain_id` | * | Companion to `pg_conninfo` for PG routing | Forwarded to `_pg_dispatch`. |

**Edge cases not enforced by the type system**
- No validation that `agent_name` doesn't contain `:`.
- `beliefs_path.suffix == ".json"` is exact — `.json5`, `.JSON`, `.json.gz` all go to the markdown branch.
- Empty `nogoods_file=""` silently activates auto-discovery.
- `pg_conninfo` uses truthiness, so `pg_conninfo=""` silently falls back to SQLite.

### Return Value

Returns a dict with at minimum `"agent": agent_name`. On the SQLite path it's whatever `_sync_agent` / `_sync_agent_json` return — per the docstring and the sibling `import_agent` shape, that typically includes keys like `"beliefs_added"`, `"beliefs_updated"`, `"beliefs_retracted"`, and `"nogoods"` counters. On the PG path it returns `PgApi.sync_agent(...)`'s dict, which may not have exactly the same keys.

Beyond `"agent"`, callers should **not** assume a specific schema — inspect the concrete return of `reasons.import_agent.sync_agent` for exact keys.

### Algorithm

1. **File resolution**
   - Build `Path(beliefs_file)` and raise `FileNotFoundError` if it doesn't exist.

2. **PostgreSQL branch** (if `pg_conninfo` truthy):
   - Lazy-import normalizers from `reasons.import_agent`.
   - If file is `.json`: parse JSON, extract `claims = _normalize_json(data, only_in)` and `nogoods = _normalize_nogoods_json(data)`. **`nogoods_file` is never consulted here.**
   - Else (markdown): read beliefs text; read `nogoods_file` if provided (existence checked); else auto-discover `beliefs_path.parent / "nogoods.md"` if present. Then `_normalize_markdown(..., only_in)` and `_normalize_nogoods_markdown(...)`.
   - Dispatch: `_pg_dispatch(pg_conninfo, domain_id, "sync_agent", agent_name=..., claims=..., nogoods=..., source_path=...)`.

3. **SQLite JSON branch** (if suffix is `.json`):
   - Parse JSON.
   - Open writeable network cache: `with _with_network(db_path, write=True) as net`.
   - Delegate to `reasons.import_agent.sync_agent_json(net, agent_name=..., data=..., only_in=..., source_path=...)`.
   - On return, `_with_network.__exit__` calls `store.save(net)`, publishes pubsub events (if any subscribers), and updates the network mtime cache.

4. **SQLite markdown branch** (default):
   - Read beliefs text.
   - Resolve nogoods text: explicit file (existence checked) → else auto-discovered sibling `nogoods.md` → else `None`.
   - Open writeable network, delegate to `reasons.import_agent.sync_agent(net, agent_name=..., beliefs_text=..., nogoods_text=..., only_in=..., source_path=...)`, and rely on `_with_network` for commit.

The actual "remote wins" reconciliation (diffing, retraction of removed nodes, updating existing, adding new) lives entirely in `reasons/import_agent.py` — `sync_agent` here is a dispatcher + file reader + backend selector.

### Side Effects

- **Disk reads:** `beliefs_file` (and `nogoods_file` if used).
- **Disk writes (SQLite):** the DB file is rewritten via `Storage.save(net)` inside `_with_network.__exit__` when `exc_type is None`.
- **Network cache mutation:** on successful exit, `_network_cache[db_path]` is updated with the new `mtime` and `Network` instance.
- **Pub/Sub:** `_with_network.__exit__` computes diff events (before/after truth values) and calls `pubsub.publish(events, db_path)` when subscribers exist for that DB.
- **PG calls:** a remote DB session is opened via `PgApi(pg_conninfo, domain_id)` and closed by the context manager in `_pg_dispatch`.
- **Log module imports:** lazy imports from `.import_agent` on every call (not memoized at module level).

### Error Handling

- `FileNotFoundError` — raised explicitly for missing `beliefs_file` or explicit `nogoods_file`.
- `json.JSONDecodeError` — propagated from `json.loads` on malformed JSON.
- Unicode/`OSError` from `.read_text()` — propagated unchanged.
- `KeyError`, `ValueError`, `sqlite3.*` from downstream sync logic — propagated; because the exception escapes the `with` block, **no DB write occurs** for that call.
- `NotImplementedError` — from PG backend for features it doesn't support (none are exercised directly here, but `PgApi.sync_agent` could raise).
- No error *return* path — this function raises. Callers must wrap in try/except if they want to report failures as dicts.

### Usage Patterns

Typical call from a CLI or scheduled job:

```python
try:
    result = sync_agent(
        agent_name="researcher-a",
        beliefs_file="agents/researcher-a/beliefs.md",
        db_path="reasons.db",
    )
except FileNotFoundError as exc:
    log.error("missing input: %s", exc)
else:
    log.info("synced %s: %s", result["agent"], result)
```

Caller obligations:
- Ensure the *remote file is fresh* before calling — this function does not fetch; it only syncs a local path.
- Do not call concurrently on the same `db_path` from multiple threads without external locking; `_with_network` writes the whole DB on exit.
- Provide a `nogoods_file` only with markdown input — it's a no-op for JSON.
- Be aware that the sync is destructive in the namespace: any local node under `agent_name:` that is absent from the remote file will be retracted.

### Dependencies

- **Stdlib:** `json`, `pathlib.Path`, `os` (implicit via `_with_network`).
- **Local:** `reasons.api._with_network`, `reasons.api._pg_dispatch`, `reasons.api.DEFAULT_DB`, and lazily `reasons.import_agent.{sync_agent, sync_agent_json, _normalize_*}`.
- **Backends:** `reasons.pg.PgApi` (via `_pg_dispatch`), `reasons.storage.Storage` (via `_with_network`), `reasons.pubsub`.

## Topics to Explore

- [file] `reasons/import_agent.py` — Contains the actual "remote wins" reconciliation, diff/patch logic, AND the `_normalize_*` helpers used by both SQLite and PG paths.
- [function] `reasons/import_agent.py:sync_agent` — The SQLite markdown implementation this wrapper delegates to; defines the shape of the return dict.
- [function] `reasons/api.py:_with_network` — Atomic commit + pubsub + mtime cache semantics that give `sync_agent` its all-or-nothing guarantee.
- [file] `tests/test_sync_agent.py` — Concrete assertions about parameter edge cases (`only_in`, JSON vs markdown, nogoods auto-discovery) you can mirror against.
- [general] `pg-sqlite-parity` — The PG path forwards pre-normalized `claims`/`nogoods` while SQLite forwards raw text; differences in behavior here (e.g., `nogoods_file` being ignored for JSON) are a real parity gap worth auditing.

## Beliefs

- `sync-agent-requires-existing-beliefs-file` — Raises `FileNotFoundError` before any network work if `beliefs_file` does not exist on disk.
- `sync-agent-remote-wins-semantics` — On success, local `agent_name:` beliefs are reconciled to match the remote file: missing nodes added, present nodes updated, unreferenced local nodes retracted.
- `sync-agent-json-ignores-nogoods-file-parameter` — When `beliefs_file` ends in `.json`, the `nogoods_file` argument is never read; nogoods are extracted from the JSON payload only.
- `sync-agent-auto-discovers-sibling-nogoods-md` — On markdown paths, if `nogoods_file` is falsy, the function looks for `beliefs_path.parent / "nogoods.md"` and uses it if present.
- `sync-agent-atomic-write-via-with-network` — Because delegation occurs inside `with _with_network(db_path, write=True) as net`, any raised exception skips `store.save(net)` and leaves the SQLite DB unchanged.
