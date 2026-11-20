---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: list_nodes in reasons/api.py

# `list_nodes` — Explanation

`reasons/api.py:list_nodes` is the read-side query API for the Reason Maintenance System. It is a **stateless, read-only, filter-heavy node-lister** that returns a JSON-serializable payload for tools, CLI, and script callers.

## 1. Purpose

`list_nodes` exists so callers can inspect the belief network without opening a `Storage` handle themselves: it opens (or reuses a cached) `Network` keyed by `db_path`, applies a large matrix of orthogonal filters (truth value, namespace, depth, tags, review status, visibility, justification label), optionally sorts and paginates, and returns `{"nodes": [...], "count": int}`.

It is the plural, filtered counterpart to `get_status` (aggregates + short summaries) and `show_node` (full detail for one node). It's also called internally from `build_wiki`, `report_gated`, `compact(format="names-only")`, etc.

## 2. Contract

**Preconditions**
- `db_path` must point to an existing (or auto-creatable) SQLite database. `_with_network` uses `Storage(db_path).load()` on cache miss.
- Caller must not pass unsupported filters when `pg_conninfo` is set (see §7).

**Postconditions**
- Returns a `dict` with exactly two keys: `nodes` (list of dicts) and `count` (int).
- `count` is the number of nodes that passed **all filters including `tag`**, but **before** `limit`/`offset` pagination. It is a "total matching" count, not a page size.
- `nodes` is bounded by `[offset, offset + limit)` when `limit` is not `None`, unbounded otherwise.
- The underlying `Network` and database are **not mutated**. No `pubsub` events fire.

**Invariants**
- `limit`, when not `None`, is coerced to `max(1, limit)` — a caller asking for 0 gets 1.
- `offset` is coerced to `max(0, offset)`.
- Iteration order of `net.nodes.items()` is by canonical ID; all user-facing sorting is stable on top of that.
- `depth` is computed and stored **only** when `min_depth`/`max_depth` were supplied **or** `sort == "depth"` at the sort step (see §5).

## 3. Parameters

| Param | Type / expected values | Notes |
|---|---|---|
| `status` | `"IN"`, `"OUT"`, or `None` | Plain string equality. Any other value yields zero matches (no error). |
| `premises_only` | `bool` | Keeps nodes with an empty `justifications` list. |
| `has_dependents` | `bool` | Keeps nodes with `len(node.dependents) > 0` (reverse-index set on `Network`). |
| `challenged` | `bool` | Keeps nodes with a truthy `metadata["challenges"]`. **Rejected in PG mode.** |
| `namespace` | `str` or `None` | Membership test `nid.startswith(f"{namespace}:")`. The caller must **not** include the trailing colon; `"ns"` works, `"ns:"` becomes `"ns::"`. |
| `min_depth`, `max_depth` | `int` or `None` | Filter by `_node_depth`; 0 = premise. Requires computing depth for every surviving candidate. **Rejected in PG mode.** |
| `visible_to` | `list[str]` or `None` | Access-tag subset check via `_is_visible`. `None` = no filtering. `[]` means "only untagged nodes visible." |
| `not_reviewed_since` | `int` (days) or `None` | Filters to nodes whose `reviewed_at` / `metadata["last_reviewed"]` is older than `now - timedelta(days=N)`. Premises are skipped. Invalid timestamps are silently kept. **Rejected in PG mode.** |
| `never_reviewed` | `bool` | Keeps derived nodes with no `reviewed_at` **and** no `metadata["last_reviewed"]`. Premises are skipped. **Rejected in PG mode.** |
| `by_impact` | `bool` | Sort by `dependent_count` descending, unless `sort` is `"created"`/`"updated"`/`"depth"`. **Rejected in PG mode.** |
| `sort` | `"created"`, `"updated"`, `"depth"`, `"impact"`, `"relevance"`(no-op) or `None` | Anything else is silently ignored. **Not forwarded to PG at all** — sorting is a no-op there even for `"created"`/`"updated"`. |
| `label` | `str` or `None` | Keeps nodes with at least one justification whose `label` matches. Premises never match. |
| `tag` | `dict[str, list[str]]` or `None` | Full tag filter; forwarded to `_filter_by_tags`. `{}` is falsy and is ignored. **Rejected in PG mode.** |
| `limit` | `int` or `None` | `None` = unlimited. Clamped to ≥ 1. |
| `offset` | `int` | Clamped to ≥ 0. |
| `db_path` | `str` | SQLite path; default `DEFAULT_DB`. |
| `pg_conninfo`, `domain_id` | PG dispatch | Route to `PgApi.list_nodes` if truthy. |

## 4. Return Value

```python
{
    "nodes": [
        {
            "id": str,
            "text": str,
            "truth_value": "IN" | "OUT",
            "justification_count": int,
            "dependent_count": int,
            "challenges": list,           # metadata["challenges"] or []
            "last_reviewed": str | None,  # reviewed_at or metadata["last_reviewed"]
            "review_result": str | None,  # metadata["review_result"]
            "source_type": str,           # "" if absent
            "created_at": str,            # "" if falsy
            "updated_at": str,            # "" if falsy
            "depth": int | None,          # None unless depth computation was required
        },
        ...
    ],
    "count": int,  # total AFTER all filters, BEFORE pagination
}
```

The caller must handle:
- `depth` being `None` for the common (non-depth) code path.
- `count > len(nodes)` whenever `limit`/`offset` are used — this is intentional.
- Empty `result["nodes"]` with `count == 0` (filters are ANDed, no errors for "no match").

## 5. Algorithm

1. **Normalize inputs** — clamp `limit` (`max(1, ...)` if not `None`), `offset` (`max(0, ...)`).
2. **PG fast path** — if `pg_conninfo`, nail down which args are unsupported (`challenged`, `min_depth`, `max_depth`, `not_reviewed_since`, `never_reviewed`, `by_impact`, any truthy `tag`) and raise `NotImplementedError` listing them. Otherwise delegate to `PgApi.list_nodes`. Note: `sort` is **not** forwarded, so it has no effect in PG mode.
3. **SQLite path** — `with _with_network(db_path) as net:`. Read-only; the cached `Network` (keyed by path + mtime) may be reused.
4. **Setup** — `memo = {}` only if depth filtering requested (otherwise `None`); compute `cutoff` only if `not_reviewed_since` was given, as `datetime.now(timezone.utc) - timedelta(days=N)`.
5. **Single pass filter loop** — for each `(nid, node)` in sorted ID order, apply in order (any `continue` short-circuits later checks):
   - `namespace` prefix
   - `status` (string equality vs `node.truth_value`)
   - `premises_only` (empty justifications)
   - `has_dependents` (non-empty `.dependents` set)
   - `challenged` (truthy `metadata["challenges"]`)
   - `label` (any justification label equals the value)
   - `visible_to` (via `_is_visible`, which is a subset-of check on access tags)
   - `min_depth` / `max_depth` (via `_node_depth`, memoized)
   - `never_reviewed` — skips premises; skips if `reviewed_at` OR `metadata["last_reviewed"]` is set
   - `not_reviewed_since` — skips premises; parses `reviewed_at or metadata["last_reviewed"]`; if unparseable, **lets node through** (silent pass); skips if `dt >= cutoff`
   - Then computes `depth = _node_depth(...) if memo is not None else None` and appends the descriptor dict.
6. **Tag filter** — if `tag` is truthy, resolve IDs against `node_tags` via `_filter_by_tags` (opens a **fresh** SQLite connection to `db_path`, bypassing the in-memory `Network`), then keep only matching nodes.
7. **Sort** — priority order:
   - `"created"` → descending by `created_at` (raw ISO string compare; `""` sorts last in descending because it's lexically smallest, then reversed → first, then stable).
   - `"updated"` → descending by `updated_at`.
   - `"depth"` → **if `memo is None`**, allocate `memo = {}` and (re)compute `depth` for every surviving node — otherwise reuse the already-populated `depth` field. Sort ascending by `depth or 0`.
   - `by_impact or sort == "impact"` → descending by `dependent_count`.
   - `by_impact` loses if `sort` is one of the above.
8. **Paginate** — `total = len(nodes)` (this is the final `count`), then `nodes = nodes[offset : offset + limit]` if `limit` is not None.
9. Return `{"nodes": nodes, "count": total}`.

## 6. Side Effects

- **Reads**: opens `Storage(db_path).load()` on cache miss; updates the module-level `_network_cache` on exit; opens a separate raw `sqlite3` connection if `tag` was supplied (`_filter_by_tags`).
- **No writes**, no pubsub events, no LLM calls, no logging.
- **Memory**: `memo` dict for depth computation, and a full in-memory `nodes` list before pagination. For a very large DB, `limit` does **not** short-circuit the loop — it always builds the full filtered list.

## 7. Error Handling

| Condition | Behavior |
|---|---|
| Unsupported PG + specific filter | `NotImplementedError(f"{names} not supported with PostgreSQL")` |
| `db_path` missing | Propagates `sqlite3.OperationalError` / `FileNotFoundError` / `OSError` from `Storage`. Not caught. |
| Invalid `status` string | Silently filters everything out; no error |
| Unknown `sort` value | Silently ignored |
| Bad `reviewed_at` ISO string | Caught `ValueError` → node is **kept** in the result set |
| Cycle in justifications | `_node_depth` guards via `memo[nid] = 0` before recursion |
| `namespace` with trailing `:` | Not detected — becomes `"ns::"`, matches nothing |

## 8. Usage Patterns

Typical calls:

```python
# List all IN beliefs in a namespace
list_nodes(status="IN", namespace="alice", db_path="reasons.db")

# Page through derived beliefs by cascade impact
list_nodes(premises_only=False, has_dependents=True,
           by_impact=True, limit=20, offset=40, db_path="reasons.db")

# Find derived beliefs that need a review pass
list_nodes(never_reviewed=True, min_depth=1, db_path="reasons.db")

# Access-controlled view for a tenant
list_nodes(visible_to=["public", "acme"], status="IN", db_path="reasons.db")
```

Caller obligations:
- Use `count` (not `len(nodes)`) for total-match logic when paginating.
- Do **not** rely on `depth` being populated unless you asked for depth filtering or `sort="depth"`.
- Do **not** pass `not_reviewed_since`/`never_reviewed`/`challenged`/`tag`/`min_depth`/`max_depth`/`by_impact` with `pg_conninfo` — it will raise.
- Understand `namespace="foo"` means node IDs starting with `"foo:"`, not `namespace` metadata matching.

## 9. Dependencies

- `reasons.storage.Storage` and `reasons.network.Network` — node model, `justifications`, `dependents`, `metadata`.
- `reasons.api._with_network` — read-only load/save + memoized `_network_cache`.
- `reasons.api._is_visible` — access-tag filtering.
- `reasons.api._filter_by_tags` + `_get_node_tags` — direct `sqlite3` reads on `node_tags`.
- `reasons.api._node_depth` — depth computation with cycle guard.
- `reasons.api._pg_dispatch` + `reasons.pg.PgApi` — PG backend (subset of filters).
- `datetime` (`timezone`, `timedelta`) — review-cutoff calculation.

## Assumptions not enforced by the type system

- `node.metadata` is a `dict` (mid-loop `.get` calls will raise `AttributeError` otherwise).
- `node.reviewed_at`, `.created_at`, `.updated_at`, `.justifications`, `.dependents`, `.truth_value`, `.text`, `.id` all exist on the object — the function never uses `getattr` fallbacks.
- `metadata["challenges"]` is a list-like that is truth-testable in the intended way (any truthy value triggers the filter).
- ISO timestamps parse with `datetime.fromisoformat` (a legacy format with `Z` suffix would raise `ValueError` on older Python and be silently kept).
- `namespace=""` (empty string) is a no-op because `if namespace:` is falsy — not a "match un-namespaced only" filter (contrast with `review_beliefs`, which treats `namespace=""` distinctly).
- `pg_conninfo` truthy implies PG backend fully wired (`reasons.pg.PgApi.list_nodes` exists and speaks the same keyword set minus the unsupported ones).

## Topics to Explore

- [function] `reasons/api.py:_node_depth` — Recursive depth computation with cycle guard and memoization; used by `min_depth`/`max_depth` and `sort="depth"`.
- [function] `reasons/api.py:_filter_by_tags` — How prefix vs exact tag matching works, and why it opens its own sqlite connection.
- [function] `reasons/api.py:_with_network` — Cache key (path + mtime), when writes evict, why read-only calls can share a `Network`.
- [file] `reasons/pg.py` — The PostgreSQL implementation of `list_nodes`: which filters survive the dispatch and how `sort` is silently dropped.
- [general] `access-tag-visibility-model` — `_is_visible` semantics (`all()` subset check), and where else (search/compact/export) it gates node visibility.

## Beliefs

- `list-nodes-count-is-pagination-invariant` — The returned `count` reflects the post-filter total before `limit`/`offset` are applied, so `count >= len(nodes)` whenever pagination is used.
- `list-nodes-limit-clamped-minimum-one` — A non-`None` `limit` is coerced to `max(1, limit)`; `limit=None` means unlimited, and `limit=0` yields exactly one result.
- `list-nodes-depth-populated-only-when-requested` — The `depth` field is `None` in the common path and is populated only when `min_depth`/`max_depth` are set or `sort == "depth"` triggers an on-demand recompute.
- `not-reviewed-filters-require-justifications` — Both `not_reviewed_since` and `never_reviewed` silently skip premise nodes (those with empty `justifications`), so they never target premises.
- `pg-list-nodes-rejects-depth-tag-review-and-impact-filters` — The PostgreSQL branch raises `NotImplementedError` for `challenged`, `min_depth`, `max_depth`, `not_reviewed_since`, `never_reviewed`, `by_impact`, and any truthy `tag`.
- `pg-list-nodes-ignores-sort-parameter` — `sort` is not forwarded to `_pg_dispatch`, so any valid `sort` value (including `"created"`/`"updated"`) is a silent no-op on the PostgreSQL backend.
