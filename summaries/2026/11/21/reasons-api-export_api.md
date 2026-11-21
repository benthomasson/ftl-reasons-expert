---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: export_api in reasons/api.py

## `export_api` — Export local beliefs to agentic-mind-service

### 1. Purpose

`export_api` is the **outbound push client** for the "Reason Maintenance System" (RMS). It loads every node in the local belief network and POSTs them one-by-one to a remote agentic-mind-service via `POST /beliefs`. It exists so a local RMS database can be mirrored / published to a remote service without the caller needing to know about storage, configuration, or HTTP details. It is the counterpart of `import_api` (which pulls via `GET /export`).

The docstring emphasizes **idempotency on `node_id`** — but that contract lives on the *server* side. This function does nothing to deduplicate; it just re-pushes, relying on the remote to overwrite rather than duplicate.

---

### 2. Contract

**Preconditions**
- The local storage backend (`db_path` for SQLite or `pg_conninfo` for PG) must be reachable; otherwise `export_network` raises before any push.
- `_resolve_config(url, agent_id, api_key)` must be able to resolve all three required values (falling back to environment / defaults). If it cannot, it raises before any push.
- The remote service must expose `POST /beliefs` with an idempotent `node_id`-keyed upsert, and accept `node_id`, `text`, `sl`, `source`.

**Postconditions**
- Every node the local network contains has been attempted. Return value reports `{"nodes_exported", "errors"}`.
- Invariant maintained across the loop: `nodes_exported + errors == len(data["nodes"])` — every iteration hits exactly one of the two branches.

**Invariants the caller should rely on**
- Nodes are iterated in the deterministic order produced by `export_network` (sorted by `node_id`).
- The result never raises for individual node failures.
- Nodes with `truth_value == "OUT"` are **also exported** — the remote receives the full graph, not merely the current belief set.

---

### 3. Parameters

| Param | Type | Notes |
|---|---|---|
| `url` | `str \| None` | Mind-service base URL. `None` → resolved by `_resolve_config` (env/default). |
| `agent_id` | `str \| None` | Agent identity for auth. `None` → resolved by `_resolve_config`. |
| `api_key` | `str \| None` | API key. `None` → resolved by `_resolve_config`. |
| `db_path` | `str` | SQLite database path. Defaults to `DEFAULT_DB = "reasons.db"`. Ignored if `pg_conninfo` is supplied. |
| `pg_conninfo` | `Any \| None` | If truthy, switches the read backend to PostgreSQL. |
| `domain_id` | `Any \| None` | PG domain scoping; only used when `pg_conninfo` is supplied. |

There is **no `visible_to` filter** — every node in the network is pushed regardless of access tags. There is **no `dry_run`**, no partial-selection parameter, no batching, and no concurrency control.

---

### 4. Return Value

Always returns a plain dict:

```python
{"nodes_exported": int, "errors": int}
```

- `nodes_exported` — count of `push_belief` calls that returned without raising.
- `errors` — count that raised any `Exception`.

**What the caller must handle**
- There is **no per-node error detail**: no node IDs or exception messages are surfaced — those only go to `logger.warning`.
- `errors > 0` does **not** mean the export failed; it means the export partially succeeded. The caller must decide whether to retry (safe because of server-side idempotency) or accept partial state.
- A complete failure (`_resolve_config` or `export_network` raising) is signalled by an **exception**, not the return dict. The caller must wrap this in a try/except.

---

### 5. Algorithm

```
1. Lazy import _resolve_config, push_belief from .mind_service
      (deferred to keep the module import light and avoid circular deps).

2. resolved_url, resolved_agent_id, resolved_api_key =
       _resolve_config(url, agent_id, api_key)
      -> falls back to env vars / defaults; may raise if unresolvable.

3. data = export_network(db_path=db_path,
                         pg_conninfo=pg_conninfo,
                         domain_id=domain_id)
      -> dict {meta, nodes{id: {...}}, nogoods, repos}
      -> uses _with_network cache; no visible_to filter.

4. For each (nid, ndata) in data["nodes"]:

   a. Flatten SL antecedents:
        sl_parts = []
        for j in ndata["justifications"]:
            if j.type == "SL" and j.antecedents:
                sl_parts.extend(j.antecedents)
        sl = ",".join(sl_parts) if sl_parts else ""

   b. push_belief(
          resolved_url, resolved_agent_id, resolved_api_key,
          node_id=nid,
          text=ndata.get("text", ""),
          sl=sl,
          source=ndata.get("source", ""),
      )
        on success: exported += 1
        on Exception: logger.warning(...); errors += 1

5. return {"nodes_exported": exported, "errors": errors}
```

**Key structural facts of the algorithm**

- **Lossy SL flattening.** If a node has two SL justifications `{A,B}` and `{C,D}`, the export string becomes `"A,B,C,D"`. In TMS semantics those two justifications are *disjunctive* (node is IN if either holds), but the flattened string is *conjunctive* — the remote will only mark the node IN when **all four** are IN. This silently changes meaning.
- **Outlists are discarded.** No `unless` info is sent. Any `UNLESS` gate on the local side will not exist remotely.
- **Any `CP` justification is silently dropped.** A node supported only by a CP justification is exported with `sl=""` — on the remote it will appear as a **premise** (no antecedent).
- **Duplicate antecedents are not removed** across justifications (an antecedent cited by two SL justifications appears twice in the string).
- **No batching / concurrency.** N sequential HTTP round-trips, one per node. Slow on large networks.
- **No transactional rollback.** A partial export leaves the remote in an intermediate state; the caller must rely on idempotency for a retry.

---

### 6. Side Effects

- **Network I/O:** one `POST /beliefs` per node to the mind-service (N requests total).
- **Remote state mutation:** creates/updates nodes on the remote — the intended effect.
- **Local read only:** `export_network` uses `_with_network(db_path)` in read mode, so the local DB is not written. However, `_with_network` *does* populate `_network_cache` on read (mtime-keyed).
- **Logging:** one `logger.warning(...)` per failed node (message includes the node ID and exception text).
- **No filesystem writes** by this function itself.

---

### 7. Error Handling

| Source | Behavior |
|---|---|
| `_resolve_config` | Raises — propagates to caller (no local try/except). |
| `export_network` | Raises on DB/PG errors — propagates to caller. |
| `push_belief` (per node) | `except Exception` — logged, `errors++`, loop continues. |
| Any other bug in the loop | Also caught by the broad `except Exception` and recorded as a node error. |

The broad `except Exception` is intentional resilience but also a **gotcha**: a programming error (e.g., `AttributeError` because `export_network` shape changed) would be silently counted as a node-level error and the caller would see only `errors += 1`. Only the log line tells the story.

The function will **not** fail on the caller's behalf; if every remote push fails (bad API key, remote down), the caller gets back `{"nodes_exported": 0, "errors": N}` and must decide whether that constitutes failure.

---

### 8. Usage Patterns

Typical call:

```python
from reasons.api import export_api

result = export_api(db_path="reasons.db")  # uses env for URL/agent/key
if result["errors"]:
    logger.warning("Export finished with %d errors", result["errors"])
    # safe to retry — server-side idempotency means re-push is a no-op
```

- **CLI publishing:** the "push to mind-service" subcommand.
- **After local edits:** call once per batch of local changes; retry is safe.
- **PG-backed deployment:** pass `pg_conninfo=...` and `domain_id=...`; `db_path` is ignored.

**Caller obligations**
- Treat the whole call as potentially partial *and* potentially raising (config/backend errors).
- Do not rely on this function for reproducible/graph-faithful remote state if the local graph has multiple SL justifications, outlists, or CP justifications — those aspects are lossy.
- For large networks, expect latency proportional to node count (N uncoupled HTTP calls).

---

### 9. Dependencies

- `.mind_service._resolve_config` — resolves/hardens URL+agent+key (env fallback).
- `.mind_service.push_belief` — the actual HTTP client POST.
- `.network.export_network` (defined in the same file) — reads the network via `_with_network` cache / Storage / PG dispatch and builds the `{meta, nodes, nogoods, repos}` payload.
- `logger` (module-level), `DEFAULT_DB`.

---

### Assumptions not enforced by the type system

1. `export_network` returns nodes with keys `justifications` (list of dicts with `type`, `antecedents`), `text`, and `source` — `.get(...)` provides defaults, so a shape change wouldn't raise immediately; it would silently export empty strings.
2. The remote's `POST /beliefs` is idempotent keyed by `node_id` — nothing in the client enforces or verifies this.
3. `push_belief`'s keyword arguments match `node_id`, `text`, `sl`, `source` — a signature change would be caught at runtime as an `Exception` and hidden behind `errors++`.
4. The remote's `sl=""` implies "premise" — required for CP-only and premise nodes to retain their meaning remotely.

---

## Topics to Explore

- [file] `reasons/mind_service.py` — Where `_resolve_config` (env/default fallback rules) and `push_belief` (HTTP contract for `POST /beliefs`) are implemented; check how idempotency is on the server.
- [function] `reasons/api.py:export_network` — The source dict consumed here; understand the exact shape of `nodes[*].justifications`, and why `visible_to` is not passed.
- [function] `reasons/api.py:import_api` — The inverse direction (pull via `GET /export`), useful for verifying round-trip faithfulness (or lack thereof) for outlists/CP.
- [file] `tests/test_mind_service.py` — Covers the client/server contract; watch for what it asserts about idempotency and per-node errors.
- [general] `lossy-network-transfer` — Meta-question: why SL flattening/outlist drop is semantically significant in a TMS, and whether a wire format that preserves full justification structure would be preferable.

## Beliefs

- `export-api-swallows-per-node-errors` — Any exception raised by `push_belief` for one node is caught, logged at WARNING, and counted in `errors`; the function does not raise on individual node failures.
- `export-api-requires-server-idempotency` — Idempotency on `node_id` is a server-side contract of `POST /beliefs`; the client does not deduplicate and relies on that server behavior.
- `export-api-produces-flattened-sl` — Multiple SL justifications per node are merged into a single comma-separated antecedent string, collapsing disjunctive justification structure into a conjunction.
- `export-api-drops-outlists` — Outlist (`UNLESS`) conditions on SL justifications and all CP justifications are not transmitted, so nodes with such structure arrive at the remote as premises or plain SL nodes.
- `export-api-returns-no-error-details` — The return dict exposes only counts (`nodes_exported`, `errors`); per-node IDs and exception messages go only to the module logger.
