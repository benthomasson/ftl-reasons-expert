---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: trace_assumptions in reasons/api.py

## Overview

`trace_assumptions` is a thin, functional wrapper around `Network.trace_assumptions` that (a) resolves a node ID to its leaf premises in a belief network, (b) enforces access-tag visibility at the entry point, and (c) optionally filters out premises the caller is not allowed to see. It is part of the "functional API" layer that opens/loads a database, delegates to a domain object, then closes — so callers don't manage `Storage` lifecycle.

## Purpose

Given a node in a Truth Maintenance System network, find every *premise* (a node with no justifications — a leaf the belief bottoms out in). It exists so higher-level tools (e.g. `report_belief`, `verify_belief`, CLI) can enumerate "what does this belief ultimately rest on?" without touching the `Network` object, and it centralizes the access-control check that callers would otherwise have to reimplement.

## Contract

**Preconditions**
- `node_id` must exist in the network. Otherwise → `KeyError`.
- If `visible_to` is not `None`, the *target* node's `metadata["access_tags"]` must be a subset of `visible_to`. Otherwise → `PermissionError`.
- `db_path` must point to a valid SQLite DB (or `pg_conninfo` must be supplied to bypass it).

**Postconditions**
- Returns `{"node_id": str, "premises": list[str]}`.
- No DB mutation; network loaded read-only (cache-eligible).

**Invariants / assumptions not enforced by the type system**
- `net.trace_assumptions(node_id)` returns a list of node ID strings. Some may reference nodes that *do not exist* in `net.nodes` (dangling refs); the wrapper treats these defensively (filters them out).
- `node.metadata` is always a dict (used lazily via `.get` in `_is_visible`).
- The PG backend's `trace_assumptions` method is assumed to accept exactly `node_id` and `visible_to` kwargs and return the same dict shape.
- `visible_to=None` means **no access control at all**; `visible_to=[]` is not the same — it only lets through nodes with no access_tags.

## Parameters

| Parameter | Type | Notes |
|---|---|---|
| `node_id` | `str` | Must exist. Namespaced IDs (`ns:foo`) are fine — no namespace resolution occurs here. |
| `visible_to` | `list[str] | None` | If `None`, both the target check and the premise filter are skipped entirely. Empty list is treated as "only empty-tag nodes visible". |
| `db_path` | `str` | SQLite file path. Defaults to `DEFAULT_DB` (`"reasons.db"`). Ignored when `pg_conninfo` is truthy. |
| `pg_conninfo` | any (likely str/dict) | If truthy, dispatches to the PostgreSQL backend via `_pg_dispatch`; `db_path` is then unused. |
| `domain_id` | any | Forwarded to the PG backend only; ignored in SQLite path. |

## Return Value

```python
{"node_id": "<the same id passed in>", "premises": ["id1", "id2", ...]}
```

- If the target is itself a premise, `premises` is `[]` (no ancestors to walk).
- If all premises are invisible to `visible_to`, returns `{"node_id": ..., "premises": []}` — a *partial* result with no error to signal that filtering occurred.
- Order of `premises` is whatever `Network.trace_assumptions` returns (typically traversal order); no sort or dedup is done here.

## Algorithm

1. If `pg_conninfo` is truthy, delegate: `_pg_dispatch(pg_conninfo, domain_id, "trace_assumptions", node_id=..., visible_to=...)`. Return verbatim.
2. Otherwise enter `_with_network(db_path)` (a context manager that loads from the mtime-keyed cache when possible, else opens `Storage` and loads a `Network`).
3. Assert `node_id in net.nodes`; else `KeyError`.
4. If `visible_to is not None`, run `_is_visible(net.nodes[node_id], visible_to)`. If false, raise `PermissionError`. This is a *gate* — you cannot even query invisible nodes.
5. Call `net.trace_assumptions(node_id)` — the actual backward traversal lives in `Network`.
6. If `visible_to is not None`, post-filter the list: keep a premise only if it *exists in the network* **and** passes `_is_visible`. Dangling IDs are silently dropped; invisible IDs are silently dropped.
7. Wrap and return the dict.

The context manager's `__exit__` updates `_network_cache[db_path]` with the current file mtime, so repeated read-only calls are memoized until the file changes.

## Side Effects

- **Reads** the SQLite DB (or hits PostgreSQL).
- **Mutates the module-global cache** `_network_cache[db_path] = (mtime, net)` on exit — a shared-cache side effect even for a "pure-looking" read.
- On the PG path, opens and closes a `PgApi` connection.
- No writes, no logging, no pubsub.

## Error Handling

| Exception | When |
|---|---|
| `KeyError` | Target node not in `net.nodes`. Message includes the id. |
| `PermissionError` | Target node exists but its access_tags aren't a subset of `visible_to`. |
| (PG errors) | Any error from `PgApi.trace_assumptions` propagates unchanged. |

Notably **swallowed**:
- Premises that don't exist in the network (`p in net.nodes` check).
- Premises whose access_tags fail `_is_visible`.

This is a silent, non-signaled filter — the caller cannot distinguish "node has 0 premises" from "all premises were filtered out by permissions". That asymmetry (raise on target, silently drop premises) is a deliberate-but-subtle policy choice.

## Usage Patterns

Two internal callers, both pass `**backend` (`db_path`, `pg_conninfo`, `domain_id`) and leave `visible_to` unset:

- `report_belief(...)` — traces premises, fetches each one's details, and passes them to the report generator.
- `verify_belief(node_id, trace=True, ...)` — only traces when the target has justifications; each premise is then verified against its source.

Neither caller passes `visible_to`, so **in practice the access-control path is currently unused inside the codebase** — it exists for CLI / external API callers. A caller must remember: if `visible_to=None`, no check happens, so access control is opt-in.

## Dependencies

- `_pg_dispatch` (same module) → `reasons.pg.PgApi`.
- `_with_network` (same module) → `reasons.storage.Storage`, `reasons.network.Network`, `_network_cache`.
- `_is_visible` (same module) — reads `node.metadata.get("access_tags", [])`.
- `Network.trace_assumptions` (in `reasons/network.py`) — does the real graph walk.
- `PgApi.trace_assumptions` (in `reasons/pg.py`) — parallel backend.

## Topics to Explore

- [function] `reasons/network.py:Network.trace_assumptions` — The actual backward traversal; whether it dedups, handles cycles, and includes the target itself.
- [function] `reasons/api.py:_with_network` — Cache invalidation via mtime; implications for concurrent writers.
- [file] `reasons/pg.py` — Verify the PG backend matches the SQLite semantics for visibility filtering and dangling premises.
- [file] `tests/test_access_tags.py` — How the "silently filter premises" and "raise on target" policies are exercised.
- [general] `access-tag-propagation` — Why the wrapper re-checks visibility on premises instead of trusting `trace_access_tags`-style union.

## Beliefs

- `trace-assumptions-requires-node-existence` — The function raises `KeyError` if `node_id` is not present in `net.nodes`; it never returns an empty-premises result for a missing node.
- `visible-to-none-disables-access-checks` — Passing `visible_to=None` skips both the target visibility check and the premise filter, so no `PermissionError` can be raised on that path.
- `invisible-premises-are-silently-filtered` — Premises whose access_tags aren't a subset of `visible_to` are dropped from the returned list with no error or signal to the caller.
- `trace-assumptions-filters-dangling-premises` — The list comprehension's `p in net.nodes` guard discards premise IDs that reference nodes not present in the network.
- `trace-assumptions-dispatches-to-pg` — A truthy `pg_conninfo` bypasses the SQLite code path entirely and delegates to `PgApi.trace_assumptions` with only `node_id` and `visible_to`.
- `trace-assumptions-is-cache-mutating-read` — Although read-only with respect to the DB, the function populates `_network_cache[db_path]` as a side effect of `_with_network`'s `__exit__`.
