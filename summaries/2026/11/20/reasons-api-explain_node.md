---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: explain_node in reasons/api.py

## `explain_node` — `reasons/api.py`

### Purpose

`explain_node` is a read-only entry point into the Reason Maintenance System that answers the question *"why is this node currently `IN` or `OUT`?"* It is part of a family of standalone functional APIs in `api.py` (each one opens the DB, operates, closes) and is designed to be called from the CLI, LangGraph tools, or arbitrary scripts without managing a `Storage` lifecycle.

The function does three things:

1. Resolves a **backend** (SQLite via `_with_network` by default, or the PostgreSQL backend if `pg_conninfo` is supplied).
2. Performs **access-control** on the target node.
3. Delegates the actual "why" computation to `Network.explain` and wraps the result.

It exists alongside `trace_assumptions`, `trace_access_tags`, and `report_belief`, which all share the same pattern.

---

### Contract

**Preconditions (enforced):**
- `node_id` must currently exist in the network at `db_path` (or the PG domain) — otherwise `KeyError`.
- If `visible_to` is not `None`, the target node's `metadata["access_tags"]` must be a subset of `visible_to` — otherwise `PermissionError`.

**Preconditions (not enforced by types):**
- `net.nodes` must be a dict-like supporting `in` and `[]`.
- `Network.explain(node_id)` must return a JSON-serializable `list[dict]` — the caller serializes the outer dict.
- The PG backend (`PgApi`) must implement `explain_node(node_id=..., visible_to=...)` and return the same `{"steps": [...]}` shape.
- `node.metadata` must be a dict (or falsy container) — `_is_visible` calls `.get("access_tags", [])` on it.

**Postconditions:**
- On success, returns `{"steps": [...]}` — a fresh dict on each call.
- Never mutates the database or network (uses `write=False` under `_with_network`).
- Side effect: may populate `_network_cache[db_path]` in the SQLite path.

**Invariants assumed (not verified):**
- If `visible_to` is `None`, **no filtering** of any kind applies — everything is treated as visible.
- If `visible_to` is `[]` (empty list), only nodes with no `access_tags` are visible.
- The return of `net.explain` is **not** filtered by `visible_to` — so the returned `steps` may describe justifications/antecedents the caller is not authorized to see. The permission check is on the *target* node only.

---

### Parameters

| Name | Type | Meaning | Notes / Edge Cases |
|---|---|---|---|
| `node_id` | `str` | ID of the belief to explain | Must exist. Not namespaced automatically — caller must pass the exact key. |
| `visible_to` | `list[str] \| None` | The caller's access-tag identities | `None` = no access check (public mode). `[]` = only untagged nodes are visible. Non-empty = target's tags must all be in this set. |
| `db_path` | `str` | SQLite file path | Defaults to `"reasons.db"`. Ignored entirely when `pg_conninfo` is truthy. |
| `pg_conninfo` | `Any` | If truthy, route to the PG backend | Falsy values (`""`, `None`) fall through to SQLite. Only truthiness is tested — no validation of the string. |
| `domain_id` | `Any` | Tenant/domain identifier for PG | Only forwarded to `_pg_dispatch`; unused in the SQLite path. |

---

### Return Value

```python
{"steps": list[dict]}
```

The contents of `steps` come from `Network.explain(node_id)` — the schema of each step is defined there (typically justifications visited, antecedent truth values, outlist evaluations, and a final "why IN/OUT" verdict). The function itself does not reshape or sanitize these steps.

Callers must handle the possibility that the returned steps reference node IDs the caller doesn't have access to (see "Assumptions" above).

---

### Algorithm

```
1.  if pg_conninfo is truthy:
        return _pg_dispatch(pg_conninfo, domain_id, "explain_node",
                            node_id=node_id, visible_to=visible_to)
        # Opens PgApi(pg_conninfo, domain_id), calls .explain_node(...)
        # NOTE: db_path is intentionally not forwarded.

2.  with _with_network(db_path) as net:      # write=False (default)
        # _with_network checks os.path.getmtime(db_path) vs cached mtime.
        #   - Cache hit  → reuse in-memory Network, do NOT open Storage.
        #   - Cache miss → Storage(db_path).load() → network.
        # On exit: closes Storage if opened, refreshes cache entry.

    2a. if node_id not in net.nodes:
            raise KeyError(f"Node '{node_id}' not found")

    2b. if visible_to is not None and not _is_visible(net.nodes[node_id], visible_to):
            raise PermissionError(
                f"Node '{node_id}' requires access tags not in {visible_to}"
            )

    2c. steps = net.explain(node_id)          # core work, read-only
        return {"steps": steps}
```

`_is_visible` (called in 2b) applies the rule:
> A node is visible iff it has **no** `access_tags` **or** every tag is in `visible_to`.

So untagged nodes are effectively public. This is a deliberate security model — worth confirming when reviewing callers.

---

### Side Effects

- **Cache write (SQLite path only):** `_with_network.__exit__` updates `_network_cache[db_path] = (mtime, net)`. This is process-global mutable state; cache invalidation relies on the file's mtime changing.
- **Storage lifecycle:** On cache miss, a `Storage` object is created and closed before return. No writes occur (write path is gated on `write=True`, which is not passed here).
- **PostgreSQL path:** Opens a `PgApi` context manager (presumably holds a pool connection and releases it on exit). Does not touch the SQLite cache.
- **No logs emitted** by this function itself; exceptions propagate.

---

### Error Handling

| Exception | Source | Recovery |
|---|---|---|
| `KeyError(f"Node '{node_id}' not found")` | Explicit existence check | Caller should `show_node`/`list_nodes` first or treat as a 404 |
| `PermissionError(...)` | `_is_visible` check | Caller must supply a `visible_to` superset of `node.metadata["access_tags"]` |
| `sqlite3.OperationalError`, `sqlite3.DatabaseError` | Opening `Storage` in `_with_network` | Corrupt/missing DB — propagate |
| `OSError` from `os.path.getmtime` | `_with_network.__init__` | Swallowed: `mtime = None`, cache is bypassed |
| `AttributeError` / `TypeError` | PG path — if `PgApi` lacks `explain_node` or signature drifts | Indicates backend contract drift, propagate |
| Arbitrary exception from `net.explain` | TMS algorithm | Propagate |

No errors are caught and converted to dicts — the caller must be prepared to handle every exception. This contrasts with, e.g., `lookup` / `search`, which return human-readable strings on empty results.

---

### Usage Patterns

```python
# CLI / script — plain SQLite
result = explain_node("belief-foo")
for step in result["steps"]:
    print(step)

# Multi-tenant API — restrict the caller
try:
    result = explain_node("belief-foo", visible_to=["team-a", "public"])
except PermissionError:
    return 403

# Server — route to Postgres
result = explain_node(
    "belief-foo",
    visible_to=requires_user.tags,
    pg_conninfo="postgresql://...",
    domain_id="tenant-1",
)
```

**Caller obligations:**
- Handle `KeyError` and `PermissionError` explicitly — they are the **only** "expected" failures but are *not* returned as dicts.
- Do **not** trust `visible_to` to have filtered `steps`. If `steps` can carry text/metadata of other nodes, downstream presentation must apply its own filter (see `report_belief`, which does partial filtering via `show_node` for premises).
- Do not rely on `db_path` when `pg_conninfo` is passed — it is silently ignored.

Internal caller in the same file:

```python
# report_belief()
explain_data = explain_node(node_id, **backend)
report = _report_belief(node_id, ..., explain_data["steps"], ...)
```

Here `backend` is `dict(db_path=..., pg_conninfo=..., domain_id=...)`, demonstrating that both backends share one call site.

---

### Dependencies

- `_with_network` (in-file) — load/cache/close orchestration.
- `_pg_dispatch` (in-file) — lazy-imports `reasons.pg.PgApi`.
- `_is_visible` (in-file) — access-tag subset predicate.
- `reasons.storage.Storage` — executes `load()` in the SQLite path.
- `reasons.network.Network` — provides `.nodes`, `.explain(node_id)`.
- `os.path.getmtime`, `sqlite3` (transitively via `Storage`).

---

### Assumptions Not Enforced by Types

1. **Cache correctness** — `_with_network` keys the cache on `db_path` only (not `pg_conninfo`, not process). Two concurrent writers with the same path but different file mtimes observed at different moments could theoretically race because `os.path.getmtime` + `load()` are not atomic.
2. **`visible_to` is not forwarded to `net.explain`** — the algorithm has no awareness of caller permissions. Any transitive data returned in `steps` is leaked to whoever receives the dict.
3. **`metadata["access_tags"]` is `list[str]`** — `_is_visible` uses `set(visible_to)` and membership; a dict/None/mixed types would fail silently or raise.
4. **PG backend parity** — the `{"steps": [...]}` shape and `KeyError`/`PermissionError` semantics are assumed but not enforced by the signature. Its own visibility check is entirely inside `PgApi.explain_node`.
5. **`Network.explain` cannot raise for OUT nodes** — the function assumes explaining an OUT node is a valid query, since it has no truth-value guard.

---

## Topics to Explore

- [function] `reasons/api.py:_is_visible` — The security predicate; its "no tags ⇒ public" rule shapes every access check in this module.
- [function] `reasons/api.py:_with_network` — Read/write cache semantics and mtime-based invalidation are the backbone of every functional API call.
- [function] `reasons/network.py:Network.explain` — Defines the actual shape of `steps`; read this to know what the caller receives.
- [file] `reasons/pg.py` — Where `PgApi.explain_node` lives; verify its visibility check and return schema match the SQLite path.
- [file] `tests/test_access_tags.py` — Exercises `visible_to` behaviour end-to-end including the empty-list case.

## Beliefs

- `explain-node-is-read-only` — `explain_node` never mutates the network or database; it uses `_with_network` with the default `write=False`.
- `explain-node-pg-dispatch-bypasses-sqlite` — A truthy `pg_conninfo` routes the call to `PgApi.explain_node` and completely ignores `db_path`.
- `explain-node-checks-target-node-visibility-only` — The `PermissionError` guard applies only to `node_id`; the returned `steps` are not filtered against `visible_to`.
- `explain-node-treats-untagged-nodes-as-public` — `_is_visible` returns `True` whenever `metadata["access_tags"]` is empty, so untagged nodes are visible to any caller, including `visible_to=[]`.
- `explain-node-raises-instead-of-returning-errors` — Missing nodes raise `KeyError` and access denials raise `PermissionError`; the function does not return an error dict like some sibling APIs (e.g. `search`, `lookup`).
