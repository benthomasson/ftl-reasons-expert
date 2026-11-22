---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: list_gated in reasons/api.py

# `list_gated` — Explanation

## Purpose

`list_gated` answers the question: *"Which OUT beliefs are currently held out by an active IN blocker?"*

In this RMS, a justification can carry an `outlist` of node IDs. A justification only holds when every node in its outlist is OUT. So when an outlist node is **IN**, it *gates* the justification — the belief goes OUT through the TMS's defeat mechanism (challenges, defeaters, supersession, nogoods, etc. ultimately gate via outlists).

This function surfaces that relationship inverted: instead of "node X is OUT", it produces "blocker B is gating nodes [X, Y, Z]". It exists primarily to feed `report_gated`, which turns gated beliefs into a problems/open-issues report, and to let callers see *why* otherwise-supported beliefs are stuck OUT.

## Contract

- **Precondition:** A reachable store — either a SQLite DB at `db_path` (or the default `reasons.db`) or a PostgreSQL instance reached via `pg_conninfo`/`domain_id`.
- **Postcondition:** A JSON-serializable dict describing blockers and the gated nodes each blocks. Nothing is written; no truth values are re-evaluated.
- **Invariants established:**
  - Only nodes with `truth_value == "OUT"` are considered as *gated* nodes.
  - Only nodes with `truth_value == "IN"` are considered *blockers*.
  - Nodes carrying `metadata["superseded_by"]` are excluded from the gated set (supersession is a deliberate removal, not an open gate).
  - `visible_to` filtering is applied to the **gated** node only (see caveats below).
- **Invariants NOT established (important):**
  - It does *not* verify that the gated node *would* be IN if the blocker were removed. It only asserts "this OUT node has a justification whose outlist contains an IN node." A node OUT for unrelated reasons (missing antecedents, nogood retraction) can still be reported.
  - It does *not* confirm the blocker's outlist entry is on the *specific* justification that could otherwise fire.

## Parameters

| Param | Type | Meaning / edge cases |
|---|---|---|
| `visible_to` | `list[str] \| None` | Access-tag filter. `None` → no filter. Otherwise a node is visible iff **all** its `access_tags` ⊆ `visible_to` (empty tags always visible). Applied to the *gated* OUT node; **not** applied to blocker nodes (see caveat). |
| `db_path` | `str` | SQLite path. Defaults to `DEFAULT_DB = "reasons.db"`. Ignored when `pg_conninfo` is set. |
| `pg_conninfo` | truthy/None | If set, all work is delegated to `PgApi` via `_pg_dispatch`; the SQLite branch never runs. Passing `visible_to` through to the PG backend, but **not** `domain_id` here (domain_id is baked into the `PgApi` constructor inside `_pg_dispatch`). |
| `domain_id` | any | Forwarded to `PgApi`; ignored on the SQLite path. |

## Return Value

```python
{
  "blockers": {
      "<blocker_id>": {
          "text": "<blocker node text>",
          "gated": [{"id": "<node_id>", "text": "<node text>"}, ...],
      },
      ...
  },
  "gated_count": int,   # sum of len(blockers[*]["gated"])
  "blocker_count": int, # number of distinct blocker IDs
}
```

- Always returns the same shape; empty state is `{"blockers": {}, "gated_count": 0, "blocker_count": 0}`.
- `gated_count` counts **(blocker, gated-node) associations**, not unique gated nodes. A single OUT node with two IN outlist blockers increments `gated_count` twice and appears in two `blockers` entries.
- Within one blocker, a gated node appears **at most once**, even if multiple justifications reference the same blocker (explicit dedupe by `id`).
- Ordering: `blockers` keys are in first-encounter order over nodes sorted by ID; `gated` entries follow sorted-node-id order (both are incidental, not contractual).

## Algorithm (SQLite path)

1. **Backend dispatch.** If `pg_conninfo` is set, return `_pg_dispatch(pg_conninfo, domain_id, "list_gated", visible_to=visible_to)`. Otherwise continue.
2. **Open network.** `with _with_network(db_path) as net:` — read-only context (`write=False`), so it may reuse the mtime-keyed cache and will not save anything.
3. **Scan every node, sorted by ID.** For each `(nid, node)`:
   - Skip unless `node.truth_value == "OUT"`.
   - Skip if `node.metadata.get("superseded_by")` is truthy.
   - Skip if `visible_to is not None` and `_is_visible(node, visible_to)` is false.
4. **Walk justifications → outlists.** For each justification `j` and each `outlist_id in j.outlist`:
   - Skip if `outlist_id` is not in `net.nodes` (dangling reference; silently ignored).
   - Skip if that node's `truth_value != "IN"` (inactive gate).
   - Lazily create `blockers[outlist_id] = {"text": out_node.text, "gated": []}`.
   - Append `{"id": nid, "text": node.text}` only if `nid` isn't already listed under this blocker.
5. **Aggregate.** `gated_count = sum(len(b["gated"]) ...)`; `blocker_count = len(blockers)`.
6. **Return** the dict.

## Side Effects

- **None that persist.** No writes, no truth-value recomputation, no metadata mutation — the network is loaded (or reused from cache) read-only.
- **Cache:** may populate/update `_network_cache[db_path]` as a side effect of `_with_network.__exit__` (read-only branch stores the mtime + network).
- **I/O:** opens SQLite (via `Storage`) or a Postgres connection. `PgApi` runs inside a `with` block (`_pg_dispatch`).
- **No logging, no pubsub** in this function.

## Error Handling

- **No explicit `raise` or error dict.** All filtering is by `continue`.
- **Dangling outlist IDs** are silently dropped.
- **Outlist nodes that exist but are OUT** are silently dropped (treated as inactive gates).
- Exceptions can come from below:
  - `_with_network` → `Storage(db_path).load()` for a missing/corrupt DB will raise (e.g., `sqlite3.OperationalError`).
  - `_pg_dispatch` → `PgApi(...)` construction and method call may raise (connection, schema, etc.).
  - Attribute errors if a node/justification is malformed (broken `metadata`, missing `outlist`).
- Callers get none of the "soft" failures surfaced — if you need to know a gated node exists but the blocker was filtered, this function will not tell you.

## Usage Patterns

- Typically called from CLI/reporting code (`report_gated` calls `list_gated(**vis, **backend)` and then enriches with `show_node` info for each blocker).
- Caller obligations:
  - Treat empty `blockers` as the normal "everything is fine" path.
  - Don't assume the reported blocker is the *only* reason the node is OUT (see Contract caveats).
  - Don't assume `blockers[bid]["text"]` reflects a node the caller is authorized to read — see the visibility caveat.

## Dependencies

Internal:
- `_with_network` — cache-aware read/write context manager.
- `_is_visible` — access-tag subset check (`node.metadata["access_tags"]`).
- `_pg_dispatch` — Postgres backend router (`reasons.pg.PgApi`).
- `Network`/`Node`/`Justification` (from `reasons`), `Storage`, `DEFAULT_DB`.

External:
- `sqlite3` (indirectly, through `Storage`).
- PostgreSQL driver (indirectly, via `PgApi`).

## Caveats the type system doesn't enforce

1. **Visibility is asymmetric.** `visible_to` is checked only on the OUT node. If a gated node is visible but its blocker is tagged with `access_tags` the caller lacks, the blocker's **ID and text are still returned**. This is a potential access-tag leak of blocker data — callers relying on `visible_to` for confidentiality should filter `blockers` themselves (e.g., with `_is_visible(net.nodes[bid], visible_to)`).
2. **"Gated" ≠ "would be IN otherwise."** The check is purely `OUT node whose justification has an IN outlist entry`. It does not verify antecedent satisfaction or that this is the operative justification.
3. **Metadata assumed dict-shaped.** `node.metadata.get(...)` assumes `metadata` supports `.get` — a `None` or non-dict value would raise `AttributeError` at the `.get` call inside the check.
4. **`outlist` assumed to be strings of existing-or-missing node IDs.** No validation; a self-referential outlist entry (`outlist_id == nid`) would still be treated literally, and a blocker can't be `nid` because the blocker is required to be IN while `nid` is OUT.

## Topics to Explore

- [function] `reasons/api.py:_is_visible` — Exact subset semantics for `access_tags` and why empty tags are always visible.
- [function] `reasons/api.py:_with_network` — How read-only calls reuse the network cache and when the DB is reloaded.
- [file] `reasons/pg.py` — The `PgApi.list_gated` implementation to compare SQL semantics with the Python loop.
- [function] `reasons/api.py:report_gated` — The primary consumer that turns these blockers into a problems report.
- [general] `outlist-defeat-semantics` — How `Justification.outlist` nodes gate a belief and interact with recompute, challenges, and defeaters.

## Beliefs

- `list-gated-excludes-superseded-out-nodes` — Any OUT node with a truthy `metadata["superseded_by"]` is skipped before its justifications are inspected.
- `list-gated-filters-gated-nodes-by-visible-to` — The `_is_visible` check is applied to the OUT (gated) node, so nodes failing the tag subset are omitted entirely.
- `list-gated-omits-blocker-visibility-check` — Blocker IDs and `text` are included in the result without any `visible_to` filtering on the blocker node, so caller-facing access control can be bypassed for blocker details.
- `list-gated-requires-in-blocker` — An `outlist` entry only counts as a gate if the referenced node exists in the network and currently has `truth_value == "IN"`.
- `list-gated-delegates-to-postgres-when-pg-conninfo-set` — When `pg_conninfo` is truthy, the function calls `PgApi.list_gated` via `_pg_dispatch` and never executes the SQLite scan.
