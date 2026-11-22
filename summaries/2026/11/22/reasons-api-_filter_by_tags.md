---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _filter_by_tags in reasons/api.py

# `_filter_by_tags` — Explanation

## Purpose

`_filter_by_tags` is an internal helper in `reasons/api.py` that narrows a list of node IDs down to those whose stored SQLite tags satisfy a set of filter criteria. It exists to support the `tag=` kwarg on the two public entry points that can return many nodes — `search()` and `list_nodes()` — without each of them re-implementing the same query-and-join logic. The function is never called directly by external callers; it is always invoked as a post-filter step inside a read-only `_with_network` block.

## Contract

**Preconditions (assumed, not enforced):**
- `db_path` points to a valid RMS SQLite database that has the `node_tags(node_id, tag)` table (created by `Storage`).
- Every element of `node_ids` is a string suitable as a SQLite bind parameter.
- `tag_filters` maps each filter key to a **non-empty** list of string values; an empty list is a caller bug that silently filters everything out (see Edge Cases).
- Tags stored in `node_tags` are expected to follow the `key:value` convention when prefix filters are intended to match them. There is no schema enforcement of this convention — tags are just opaque strings.

**Postconditions:**
- Returns a list of node IDs, each a member of `node_ids`, in the **same relative order** as the input.
- The result is a *subset*: `set(result) ⊆ set(node_ids)`.
- A node with **no tags** in `node_tags` will not match any non-empty filter (it takes the empty-list branch of `all_tags.get(nid, [])`).
- If `tag_filters` is falsy, the *same list object* passed in is returned (not a copy) — callers rely on reassignment, so this is invisible in practice but is a real aliasing behavior.

**Not guaranteed:**
- Determinism of the input node list. Result order follows input order; the function itself does not sort.
- Any relationship between `tag_filters` keys other than the reserved literal `"tag"`.

## Parameters

| Parameter | Type | Meaning |
|---|---|---|
| `node_ids` | `list[str]` | Candidate node IDs. May be empty. May contain IDs not present in the DB (they simply won't match unless they have no tags and `tag_filters` is empty). |
| `tag_filters` | `dict[str, list[str]]` | ANDed filter groups; within each group values are ORed. The key `"tag"` is special (exact-tag match); any other key triggers auto-prefixing. |
| `db_path` | `str` | Path to SQLite DB. Opening a nonexistent file silently creates an empty DB, but querying a missing table raises `sqlite3.OperationalError`. |

**Edge cases:**
- `tag_filters == {}` or `None` → returns `node_ids` **by reference**, and no DB access occurs.
- `node_ids == []` → `_get_node_tags` short-circuits to `{}`, and the loop body never executes, so `[]` is returned.
- A filter value list of `[]` under any key (e.g. `{"topic": []}`) makes `any(...)` return `False`, so **no node can match** — this is almost certainly a caller bug, but the function does not raise.
- If the reserved key `"tag"` is combined with other keys, both must be satisfied (AND), which is usually fine but makes `"tag"` and prefix keys cohabitable.
- Filter values are interpolated into an f-string (`f"{filter_key}:{v}"`), so a non-string value would be silently coerced, not rejected.

## Return Value

- `list[str]` — filtered node IDs preserving the input order.
- If `tag_filters` is falsy, the exact same list object is returned (aliasing hazard if the caller mutates it).
- If the DB has no rows for a given node ID, that node's tags are `[]` and it fails any non-trivial filter.
- Never returns `None`; never raises for "no matches" — an empty list is a valid, meaningful return.

## Algorithm

1. **Short-circuit empty filters.**
   ```python
   if not tag_filters:
       return node_ids
   ```
   Skips the DB call entirely when no filter is requested.

2. **Bulk-load tags.**
   ```python
   all_tags = _get_node_tags(db_path, node_ids)
   ```
   Opens a fresh SQLite connection, issues one `SELECT node_id, tag FROM node_tags WHERE node_id IN (...)` (with `?` placeholders for each ID), and builds `{node_id: [tag, ...]}`. If `node_ids` is empty, that helper returns `{}` without a query.

3. **Iterate candidates in order.**
   For each `nid`:
   - Look up `node_tags = all_tags.get(nid, [])`.
   - Assume `match = True` and break early on the first failing group.

4. **Evaluate each filter group (AND).**
   ```python
   for filter_key, filter_values in tag_filters.items():
       if filter_key == "tag":
           if not any(v in node_tags for v in filter_values):
               match = False; break
       else:
           prefixed = [f"{filter_key}:{v}" for v in filter_values]
           if not any(p in node_tags for p in prefixed):
               match = False; break
   ```
   - `"tag"` key: raw list-membership against the node's tag strings. Despite the docstring's "exact match" language, this is ordinary Python `in` on a `list[str]` — it's exact string equality, not prefix matching.
   - Any other key: the prefix `"{key}:"` is prepended to each value, then prefix-expanded values are checked by the same `in` operation. So the docstring's "contains match" means *"the auto-prefixed string is contained in the tag list"* — still exact tag equality, just with the prefix added first. It is **not** substring matching.

   Both branches, therefore, are the same operation (`value in node_tags`) — the only difference is whether the caller-supplied string is used verbatim or auto-prefixed. The `any(...)` gives OR semantics within a group.

5. **Collect matches.**
   ```python
   if match:
       result.append(nid)
   ```

6. **Return** the filtered list.

Because groups are ANDed and the loop breaks on the first miss, per-node cost is O(groups × values × tag-count-per-node) worst case — tiny, since node tag lists are small.

## Side Effects

- **I/O:** Opens and closes one read-only SQLite connection per call (inside `_get_node_tags`). Two connections can be live simultaneously when called from inside `search()`/`list_nodes()` (which already hold their own connection via `_with_network`). SQLite allows this.
- **No mutations:** No writes, no `_network_cache` interaction, no pubsub emissions. This is intentionally cache-bypassing: `add_tags`/`remove_tags` write directly to `node_tags`, and this function always reads the current DB state regardless of what's in `_network_cache`.
- **Aliasing leak:** The early-return aliases the caller's list. Callers here reassign (`matched_ids = _filter_by_tags(...)`), so it's benign, but this would surprise a future caller who expects a copy.

## Error Handling

- **Propagates** `sqlite3.OperationalError` / `sqlite3.DatabaseError` if the `node_tags` table is missing or `db_path` is invalid. Nothing is caught.
- **Propagates** `TypeError` if a filter value is not string-coercible into an f-string (unlikely).
- **Silently swallows** the "empty filter-values" edge case: `any([])` is `False`, so a group with no values excludes every node. The function does not raise or log — it simply returns `[]`. Callers should validate that each value list is non-empty before invoking.
- **Silently swallows** the "ID not in DB" case: an unknown node ID just has no tags and fails.
- **No injection risk:** the SQL uses `?` placeholders for each ID; only the number of placeholders is dynamic.

## Usage Patterns

Two call sites, both inside a read-only `_with_network` context:

- `search(...)` — filters `matched_ids` **after** truth-value, namespace, and access filters, but **before** sorting and pagination, so tag-filtered results are counted correctly for `limit`/`offset`. If the result is empty, `search` returns `"No results found."`.
- `list_nodes(...)` — filters the already-materialized `nodes` list at the end of the row-building loop, but **before** `total = len(nodes)` and pagination, so `count` reflects tag filtering.

Typical caller shape:
```python
matched_ids = _filter_by_tags(matched_ids, {"topic": ["networking"]}, db_path)
```

Caller obligations:
1. Ensure the `node_tags` table exists (i.e. a real `Storage`-initialized DB).
2. Do not pass empty `filter_values` lists unless you actually want zero results.
3. Do not assume prefix filters do substring matching — the tag string itself must literally be `key:value` in `node_tags`.
4. Don't mutate the returned list if you passed an empty `tag_filters` and care about the input list.

The reserved `"tag"` key means a user cannot use `"tag"` as an ordinary *prefix* key — e.g., to match tags of the form `tag:foo`, they must use `{"tag": ["tag:foo"]}` (the raw form) rather than `{"tag": ["foo"]}` (which will match `"tag:foo"` only because of the exact-equality coincidence — and also match nothing else).

## Dependencies

- `_get_node_tags` (module-local) — bulk-loader with its own SQLite connection; handles the empty-list short-circuit.
- `sqlite3` — standard library.
- Indirectly, the `node_tags` table schema owned by `reasons/storage.py` (created via `Storage`).
- No dependency on `_with_network`, `Network`, or `_network_cache` — tags are deliberately read from disk, decoupled from the cached in-memory network.

## Topics to Explore

- [function] `reasons/api.py:_get_node_tags` — The bulk tag loader this function delegates to; owns the `IN (...)` placeholder construction and the empty-list short-circuit.
- [function] `reasons/api.py:search` — The primary consumer; shows where `_filter_by_tags` sits relative to FTS, neighbor expansion, and pagination.
- [function] `reasons/api.py:list_nodes` — The other consumer; shows how it integrates with the `count`-before-paginate contract.
- [function] `reasons/api.py:add_tags` — The writer side of the `node_tags` table; useful for understanding the tag format the filter assumes.
- [file] `tests/test_api.py` — Existing coverage; check whether empty `filter_values` and the `"tag"`-vs-prefix distinction are actually exercised.

## Beliefs

- `empty-tag-filters-short-circuits-before-db-access` — When `tag_filters` is falsy, `_filter_by_tags` returns the input list without calling `_get_node_tags`; no SQLite connection is opened.
- `tag-filter-key-uses-exact-membership-not-substring` — The `"tag"` filter key performs Python `in` over the node's tag list, which is exact string equality, not substring or prefix matching.
- `prefix-keys-auto-prefix-with-colon` — Every non-`"tag"` filter key is expanded to `"{key}:{value}"` before the membership check, so matching requires tags stored literally as `key:value`.
- `filter-groups-are-anded-within-or` — A node must satisfy every key in `tag_filters`; within one key, matching any value suffices (`any()` on a generator).
- `empty-filter-values-exclude-all-nodes` — A filter value list of `[]` makes `any(...)` return `False`, so the corresponding group fails for every node; the function does not validate against this.
