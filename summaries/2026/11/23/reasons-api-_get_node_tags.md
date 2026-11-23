---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _get_node_tags in reasons/api.py

# `_get_node_tags` — Explanation

## 1. Purpose

`_get_node_tags` is a private read helper that bulk-loads tag rows from the SQLite `node_tags` table and reshapes them from flat `(node_id, tag)` rows into a grouped `{node_id: [tag, ...]}` mapping.

Its reason for existing is to let higher-level code (specifically `_filter_by_tags`, its only caller in this file) determine tag membership for a *set* of candidate nodes in a single SQL round-trip instead of issuing one `SELECT ... WHERE node_id = ?` per node. That matters in `search()` and `list_nodes()`, which may filter hundreds or thousands of IDs at once.

It is not a public API function: it doesn't dispatch to PostgreSQL, doesn't validate input, and doesn't return the JSON-friendly envelope dicts the public surface returns.

## 2. Contract

**Preconditions (not enforced):**
- `db_path` refers to an existing SQLite database that has a `node_tags` table with at least the columns `node_id` and `tag`.
- Every element of `node_ids` (when provided) is a `str` matching the exact `node_id` stored in the table — no namespace resolution, no normalization, no case folding happens here.
- The caller must not be relying on tag ordering (see §4).

**Postconditions:**
- Returns a `dict[str, list[str]]`.
- Every key present in the result maps to a **non-empty** list. Nodes with no rows in `node_tags` are simply absent — the function does not populate `{nid: []}` for the requested-but-untagged IDs.
- Values are appended in SQL result order, which is unspecified because the query has **no `ORDER BY`**.

**Invariants:**
- The database connection is always closed, including on early return and on exception, via the `finally` block.
- The function never writes to the database.
- Only literal SQL placeholders are interpolated from Python strings; node IDs are bound as parameters (see §7).

## 3. Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `db_path` | `str` | Filesystem path to the SQLite database opened directly with `sqlite3.connect`. | A nonexistent path in an *existing* directory gets **created as an empty DB** by `sqlite3.connect`, then the query fails with `no such table`. A path in a nonexistent directory raises `unable to open database file`. |
| `node_ids` | `list[str] \| None`, default `None` | Filter set for a `WHERE node_id IN (...)` clause. | `None` → load **all** tags. `[]` (empty list) → return `{}` immediately without querying. Non-empty list → filtered query. Duplicates are harmless (redundant `IN` entries). Very large lists can exceed SQLite's `SQLITE_MAX_VARIABLE_NUMBER` (historically 999, 32766 on modern builds) and raise `OperationalError` — this is **not** guarded. |

The `None` vs `[]` distinction is the load-bearing semantic: `None` means "no restriction, give me everything," while `[]` means "the caller has zero candidate nodes, so there is nothing to look up." Callers should not pass `[]` expecting "all." 

## 4. Return Value

A `dict[str, list[str]]`:

- On success: `{node_id: [tag1, tag2, ...]}` for every requested node that has ≥1 row.
- `{}` when `node_ids == []` (early return), or when the table is empty, or when no requested node has tags.
- **Tag order is not `ORDER BY tag`.** It reflects storage/scan order. Contrast with `show_node` and `export_markdown`/`export_network` in the same file, all of which explicitly add `ORDER BY tag` or `ORDER BY node_id, tag`. `_get_node_tags` deliberately omits that because its only consumer does set-membership tests (`v in node_tags`), where order is irrelevant.

**Caller obligation:** because untagged nodes are omitted, callers must use `.get(nid, [])` rather than `result[nid]`. `_filter_by_tags` does exactly this: `node_tags = all_tags.get(nid, [])`.

## 5. Algorithm

```
connect(db_path)                                  # opens/creates SQLite file
try:
    if node_ids is not None and len(node_ids) > 0:
        placeholders = ",".join("?" * len(node_ids))   # "?,?,?" for N=3
        rows = SELECT node_id, tag FROM node_tags WHERE node_id IN (placeholders)
               with node_ids as bind params
    elif node_ids is not None:                    # empty list case
        return {}
    else:
        rows = SELECT node_id, tag FROM node_tags # unfiltered
    result = {}
    for (nid, tag) in rows:
        result.setdefault(nid, []).append(tag)    # group + preserve row order
    return result
finally:
    conn.close()
```

The placeholder expression deserves a note: `"?" * n` produces a single string of `n` question marks (`"???"`); `",".join(...)` iterates that string character-by-character, yielding `"?,?,?"`. It works, but it's the kind of idiom that trips readers — the more common spelling is `",".join(["?"] * n)`.

The empty-list branch performs no query at all; it just falls into `finally` and closes the freshly opened connection.

## 6. Side Effects

- **Opens a new SQLite connection per call.** It does not share the `Storage`/`_with_network` connection, does not participate in the `_network_cache`, and does not see unflushed in-memory state from a `_with_network(..., write=True)` context that hasn't exited.
- **May create an empty database file** if `db_path` doesn't exist yet (behaviour of `sqlite3.connect`), immediately followed by a query failure. The file is left behind.
- **No writes**, no commits, no PRAGMAs, no pubsub events, no logging.
- Read-only I/O on the SQLite file.

## 7. Error Handling

No exceptions are caught or converted — only connection cleanup is guaranteed:

- `sqlite3.OperationalError: no such table: node_tags` — caller passed an uninitialized DB (or a freshly created empty one).
- `sqlite3.OperationalError: unable to open database file` — bad directory or permission problem.
- `sqlite3.InterfaceError` — `node_ids` contained a value SQLite can't bind (e.g. a `dict`).
- `sqlite3.too many SQL variables` — `len(node_ids)` exceeds the SQLite compile-time limit.
- `TypeError` from `len(node_ids)` / iteration if `node_ids` isn't actually list-like.

**Injection safety:** the only Python value interpolated into SQL is `placeholders`, built from `"?"` characters, so the *count* of placeholders is safe. Actual node IDs go through the parameter list. This is correct — but subtle, because an f-string is used at all; reviewers need to verify the f-string can only contain `?` and `,`.

## 8. Usage Patterns

The sole in-file consumer is `_filter_by_tags`:

```python
all_tags = _get_node_tags(db_path, node_ids)
...
node_tags = all_tags.get(nid, [])
if filter_key == "tag":
    if not any(v in node_tags for v in filter_values): ...
else:
    prefixed = [f"{filter_key}:{v}" for v in filter_values]
    if not any(p in node_tags for p in prefixed): ...
```

`_filter_by_tags` in turn is called by:
- `search(..., tag=...)` — after FTS/substring matching, namespace filtering, and access filtering have narrowed `matched_ids`, so the list passed to `_get_node_tags` is usually small.
- `list_nodes(..., tag=...)` — after all other predicates have been applied to the in-memory node dict.

Caller obligations:
1. Pass the same `db_path` used to open the network, otherwise filtering silently misses tags.
2. Treat the result as sparse (`get(..., [])`).
3. Don't rely on tag order or assume a stable order across calls.
4. Don't call it inside a hot loop for single nodes — for one node, `get_tags`/`show_node` are the idiomatic entry points (though they do extra existence checking and `ORDER BY tag`).

Note the tag-keying convention asserted by the callers: stored tags look like `"topic:networking"` or `"status:needs-review"` — the `prefix:value` form is what `_filter_by_tags` reconstructs. `_get_node_tags` is completely agnostic to that convention; it returns raw stored strings.

## 9. Dependencies

- **`sqlite3`** (stdlib) — direct connection, no ORM, no `Storage` wrapper.
- **SQLite schema** — the `node_tags(node_id, tag)` table, defined in `reasons/storage.py` (via `Storage`/`PROPOSALS_SCHEMA` context).
- **Sibling helpers** in `reasons/api.py` — `_filter_by_tags` (the only caller), and by convention `add_tags`, `remove_tags`, `get_tags`, `show_node`, `export_network`, `export_markdown`, which all touch the same table with different query shapes (`ORDER BY tag`, existence checks, per-node selection).

## Assumptions the type system does not enforce

- **Schema existence and column names.** `node_tags` may not exist; the function will raise, not degrade.
- **ID equality semantics.** Strings must match byte-for-byte; `node_id="foo"` will not match `"ns:foo"` or `"Foo"`.
- **`None` ≠ `[]`.** Nothing in the signature documents that an empty list short-circuits to `{}` while `None` means "all."
- **Sparsity of the result.** The annotation `dict[str, list[str]]` implies nothing about completeness; callers must `get` with a default.
- **Tag ordering.** Nothing expresses that there is no `ORDER BY`.
- **Value size limit.** No guard or chunking for very large `node_ids` lists against `SQLITE_MAX_VARIABLE_NUMBER`.
- **Connection lifecycle.** The function assumes it can open an independent connection; it doesn't accept an existing `conn` and doesn't coordinate with `_with_network` write contexts.

---

## Topics to Explore

- [function] `reasons/api.py:_filter_by_tags` — The only caller; shows how the sparse dict and the `prefix:value` tag convention are consumed.
- [file] `reasons/storage.py` — Where the `node_tags` table is declared and how tags are persisted/loaded alongside nodes.
- [function] `reasons/api.py:add_tags` — The write-side counterpart; note it performs an explicit node-existence check that `_get_node_tags` does not.
- [function] `reasons/api.py:export_network` — An alternative bulk tag loader that uses `ORDER BY node_id, tag`; useful for contrasting ordering guarantees.
- [general] `sqlite-variable-limit` — Why feeding a large `node_ids` list into an `IN (...)` clause can raise, and what chunking strategies exist.

## Beliefs

- `_get-node-tags-returns-empty-dict-for-empty-list` — Passing `node_ids=[]` returns `{}` without executing any SQL query; only `None` means "load all tags."
- `_get-node-tags-manages-its-own-connection` — The function opens a dedicated `sqlite3` connection and closes it in `finally`, independent of `_with_network` and the `_network_cache`.
- `_get-node-tags-propagates-sqlite-errors` — No exception is caught or converted; `OperationalError` from a missing `node_tags` table reaches the caller, with only connection cleanup guaranteed.
- `_get-node-tags-uses-parameterized-in-clause` — Only `?` placeholders are built via f-string; node IDs are bound parameters, so the query is not injectable.
- `_get-node-tags-omits-untagged-nodes` — Nodes with zero rows in `node_tags` never appear as keys, forcing callers to use `result.get(nid, [])`.
- `_get-node-tags-has-no-tag-order-guarantee` — Unlike `show_node` and `export_network`, the query lacks `ORDER BY tag`, so per-node tag order is unspecified.
