---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: trace_access_tags in reasons/api.py

# `trace_access_tags` — Walkthrough

## Purpose

`trace_access_tags` answers the question: **"What set of `access_tags` protects this node *and everything it transitively depends on*?"**

It walks backward through the justification graph from `node_id` (following SL antecedents, and likely the outlist too — see the underlying `Network.trace_access_tags`) and returns the union of every `access_tags` met along the way, formatted as a JSON-serializable dict.

It exists to support access-control auditing and export planning: a caller who can see `node_id` can discover *what additional tags would be required* to view or export the full dependency chain, without exposing the chain itself.

Unlike sibling APIs such as `trace_assumptions`, this function **does not filter** the returned tags by `visible_to` — it reports the complete union so the caller can reason about required access.

## Contract

### Preconditions
- `node_id` must exist in the target network.
- If `visible_to` is provided, the node's *own* `access_tags` must be a subset of it (the caller has permission to "see" the starting node).
- If `pg_conninfo` is set, a reachable PostgreSQL endpoint that supports `trace_access_tags` must be configured.

### Postconditions
- Returns `{"node_id": str, "access_tags": list[str]}`.
- `access_tags` is the union of every `metadata["access_tags"]` entry found along the backward dependency chain, including `node_id`'s own tags.
- No mutation of the network or database occurs (read-only path).

### Invariants
- `node_id` is echoed verbatim in the return dict.
- `visible_to` is enforced **only** at the starting node — the returned tags may contain tags not in `visible_to`, which is the whole point of the function.
- The visibility predicate is "all tags must be in the caller's set, or the node has no tags" — i.e. subset semantics, not intersection semantics (see `_is_visible`).

## Parameters

| Name | Type | Notes |
|---|---|---|
| `node_id` | `str` | Node identifier in the network. Must exist; `KeyError` otherwise. |
| `visible_to` | `list[str] \| None` | Caller's granted tag set. `None` disables the starting-node visibility check. An empty list is **not** the same as `None` — it means "only untagged nodes are visible." |
| `db_path` | `str` | SQLite database path. Defaults to `"reasons.db"`. Ignored when `pg_conninfo` is set. |
| `pg_conninfo` | anything truthy (usually a DSN/connection string) | If provided, the function delegates to the Postgres backend. The whole SQLite path is bypassed. |
| `domain_id` | any | Passed to `_pg_dispatch`; used by the Postgres backend to scope the domain. Ignored on SQLite. |

Edge cases:
- `visible_to=None` — no access check on the starting node.
- `visible_to=[]` — the starting node must have **no** `access_tags` to pass the check.
- `pg_conninfo` truthy while `db_path` also provided — `db_path` is silently ignored.
- Non-existent DB path — behavior depends on `Storage` (see "Assumptions" below).

## Return Value

```python
{
    "node_id": "<the input id, unchanged>",
    "access_tags": ["<tag1>", "<tag2>", ...],  # union along the chain
}
```

The caller must handle:
- An empty `access_tags` list — the entire chain is untagged.
- Tags in `access_tags` **not** in `visible_to` — these are exactly the tags the caller lacks to fully traverse the chain.
- Ordering is **unspecified** by this wrapper; the underlying `Network.trace_access_tags` defines any deterministic order (likely sorted, but not guaranteed here).

## Algorithm

1. **Postgres short-circuit.** If `pg_conninfo` is truthy, call
   `_pg_dispatch(pg_conninfo, domain_id, "trace_access_tags", node_id=..., visible_to=...)`.
   Inside `_pg_dispatch`, this opens a `PgApi(pg_conninfo, domain_id)` context and dispatches to `PgApi.trace_access_tags`. All other steps are skipped.
2. **Acquire network (read-only).** `_with_network(db_path)` — with `write=False` — returns either a cached `Network` (if `mtime` of the file matches the cache) or a fresh `Storage`-loaded one. Because `write=False`, `pubsub` subscribers are not consulted and nothing is saved on exit.
3. **Existence check.** If `node_id not in net.nodes`, raise `KeyError(f"Node '{node_id}' not found")`.
4. **Visibility check on the starting node.** If `visible_to is not None`, call `_is_visible(net.nodes[node_id], visible_to)`.
   - `_is_visible` reads `node.metadata.get("access_tags", [])`.
   - An empty list → visible (returns `True`).
   - Otherwise returns `all(t in set(visible_to) for t in tags)`.
   - On failure, raise `PermissionError(f"Node '{node_id}' requires access tags not in {visible_to}")`.
5. **Delegate the actual trace.** `tags = net.trace_access_tags(node_id)` — this is where the graph walk happens (in `reasons/network.py`, not shown here). The network-level method walks ancestors (and likely outlist nodes) and returns the union.
6. **Return** `{"node_id": node_id, "access_tags": tags}`.

## Side Effects

- **Reads** the SQLite database (or Postgres). No writes.
- **Populates / reads** the module-level `_network_cache[db_path]` — subsequent read-only calls benefit from the cached `Network` object until the DB file's mtime changes.
- **Opens and closes** a `sqlite3` connection inside `Storage` (via `_with_network`).
- **Does not** trigger `pubsub` events (write is disabled).
- **Does not** mutate `net.nodes` or any `Node` objects.

## Error Handling

| Condition | Behavior |
|---|---|
| `node_id` missing | Raises `KeyError(f"Node '{node_id}' not found")`. |
| `visible_to` given, node's own tags exceed it | Raises `PermissionError`. |
| DB file missing / unreadable | Propagates whatever `Storage(db_path)` raises (likely `FileNotFoundError` or an `sqlite3` error). Not caught. |
| Postgres dispatch failure | Propagates from `PgApi.__enter__` / `PgApi.trace_access_tags`. |
| Underlying `net.trace_access_tags` raises | Propagates unaltered. |

There are **no swallowed errors** in the SQLite path. The function is intentionally "loud" — every runtime failure surfaces to the caller.

## Usage Patterns

Typical callers:
1. **CLI auditing** — `reasons trace-tags <id>` prints the union for a human to review.
2. **Export gating** — before calling `export_network(visible_to=X)`, check whether `X` covers the full ancestry of a belief; if `trace_access_tags` returns tags outside `X`, the caller knows the export would be incomplete.
3. **Access bundle calculation** — given a belief a user wants to see, the union tells the auth team exactly which additional tags would be required.

Caller obligations:
- Catch `KeyError` for unknown nodes.
- Catch `PermissionError` when passing `visible_to` they lack — the start node is invisible to them.
- Treat `access_tags` as potentially containing tags **not** in `visible_to` (this is expected, not a bug).

## Dependencies

- `_with_network` (same module) — context manager that loads / caches the `Network` from `Storage`.
- `_is_visible` (same module) — subset check against `visible_to`.
- `_pg_dispatch` (same module) — thin wrapper that imports `PgApi` lazily and dispatches.
- `Network.trace_access_tags` (in `reasons/network.py`) — the real graph walk that produces the union.
- `Storage` (in `reasons/storage.py`) — SQLite persistence.
- `PgApi` (in `reasons/pg.py`, imported lazily) — Postgres backend.

## Assumptions Not Enforced by Types

- `node.metadata` is assumed to be a `dict` (or at least `.get`-able and truthy for `.get("access_tags", [])` to run). `Node.metadata` defaults to `{}` in the codebase, but the type isn't checked here.
- `net.trace_access_tags` is assumed to return `list[str]` (or something JSON-serializable as one). No `typing` guard.
- `visible_to` is assumed to be a list of strings — a list containing hashables that can be tested for membership against tag strings. Passing e.g. `[1, 2]` would silently produce `False` for all tag comparisons, causing `PermissionError`.
- `pg_conninfo` is assumed to be a valid connection string for `PgApi`; the function itself performs no validation.
- `Storage(db_path)` on a missing file — behavior depends on `Storage.__init__` (likely creates the DB in some code paths but not guaranteed for read-only use). Not validated here.

## Topics to Explore

- [function] `reasons/network.py:trace_access_tags` — The actual graph walk; defines ancestor/outlist traversal semantics and tag deduplication order.
- [function] `reasons/api.py:_is_visible` — Subset-based visibility predicate reused by `show_node`, `search`, `list_nodes`, and others; the single source of truth for access checks.
- [function] `reasons/api.py:trace_assumptions` — Sibling function with a *different* policy: it filters returned premises by `_is_visible`, contrasting with `trace_access_tags`'s unfiltered union.
- [file] `reasons/pg.py` — Postgres `PgApi` implementation of `trace_access_tags`; defines whether the pg backend matches SQLite semantics.
- [file] `tests/test_access_tags.py` — Expected behaviors, edge cases, and pg parity tests for this symbol.

## Beliefs

- `trace-access-tags-only-checks-start-node` — The `visible_to` check applies only to `node_id`; ancestor tags are not filtered from the returned union.
- `trace-access-tags-returns-unfiltered-union` — The returned `access_tags` may contain tags not present in `visible_to`, intentionally revealing the full protection set along the dependency chain.
- `trace-access-tags-is-read-only` — The function uses `_with_network(db_path)` (write=False) and performs no mutations to the network or database.
- `trace-access-tags-dispatches-to-pg` — When `pg_conninfo` is truthy, the entire SQLite code path is bypassed and `_pg_dispatch` routes to `PgApi.trace_access_tags`.
- `is-visible-requires-subset` — `_is_visible` uses "all tags ∈ visible_to" subset semantics (or no tags) — a node with any tag outside `visible_to` is invisible, not merely filtered.
- `access-tags-persist-in-metadata` — Access tags live in `node.metadata["access_tags"]` (set by `add_node(access_tags=...)`), not in the `node_tags` SQLite table used by `_get_node_tags`.
