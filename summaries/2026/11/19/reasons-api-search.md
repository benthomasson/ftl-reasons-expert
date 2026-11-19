---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: search in reasons/api.py

# `search`

## Purpose

`search` is the main full-text search entry point for the Reason Maintenance System API. It searches belief nodes, applies access/truth/namespace/tag filters, optionally expands results to include nearby nodes in the dependency graph, and formats the result as a string for CLI, Python, or tool callers.

It supports three matching modes on SQLite:

1. Default FTS5 all-terms search, with substring fallback.
2. OR search when the query contains a literal `|`.
3. Regex search when `regex=True`.

It also dispatches to the PostgreSQL backend when `pg_conninfo` is provided.

---

## Contract

### Preconditions / assumptions

- `query` is a string. There is no validation that it is non-empty or well-formed beyond regex compilation when `regex=True`.
- On SQLite, `db_path` should point to a Reason Maintenance System SQLite database. If the FTS5 table is missing or broken, `_fts_search` swallows the SQLite error and the caller falls back to substring search.
- On PostgreSQL, `pg_conninfo` must be accepted by `PgApi`, and `PgApi.search` must accept `query`, `visible_to`, `format`, and `include_out`.
- The caller is responsible for choosing a valid `format`, `sort`, `limit`, and `offset`. The function does not strictly validate all of these.
- Neighbor expansion assumes `net.nodes[nid].justifications` and `net.nodes[nid].dependents` are the graph edges to traverse.
- If `include_out=True`, or if a stale FTS row produces an ID not present in `net.nodes`, some formatters can raise `KeyError` because they index `net.nodes[nid]` directly. In the default `include_out=False` path, missing IDs are filtered out.

### Postconditions / invariants

- On SQLite, the network is opened read-only through `_with_network(db_path)`. The function does not mutate nodes, justifications, metadata, or truth values.
- Filters are applied to matched IDs before pagination and before neighbor expansion.
- `limit` and `offset` constrain the matched IDs only. They do not cap the total number of returned nodes including neighbors.
- Neighbor expansion is breadth-first over `antecedents` and `dependents`, with a `visited` set to avoid cycles.
- `namespace` and `tag` filtering apply only to matched nodes, not to neighbor nodes.
- `include_out=False` filters OUT nodes from both matched results and final neighbors.
- `visible_to` filtering applies to matched results and final neighbors.
- If no matches remain after a filter, the function normally returns the literal string `"No results found."`.
- Unknown `format` values fall through to markdown formatting. Unknown `sort` values leave the existing order unchanged.
- On PostgreSQL, `depth != 1` and truthy `tag` values raise `NotImplementedError`. Other unsupported parameters such as `sort`, `namespace`, `limit`, `offset`, and `regex` are not forwarded to the PostgreSQL backend and are effectively ignored.

---

## Parameters

| Parameter | Meaning |
|---|---|
| `query` | Search text. By default interpreted as FTS5 all-terms search. A literal `|` switches to OR search. If `regex=True`, interpreted as a Python regex. |
| `visible_to` | Optional list of access tags granted to the caller. A node is visible only if all of its `metadata["access_tags"]` are contained in this set. Nodes with no access tags are always visible. |
| `db_path` | SQLite database path. Ignored when `pg_conninfo` is truthy. |
| `format` | Output format. Recognized values include `"markdown"`, `"json"`, `"minimal"`, `"compact"`, and `"names-only"`. Unknown values are treated as markdown. |
| `depth` | Number of BFS hops to expand around matched nodes. Default `1`. On PostgreSQL, only `depth == 1` is supported. On SQLite, negative or zero depth simply performs no expansion. |
| `include_out` | If false, OUT nodes are excluded from both matches and neighbors. If true, OUT nodes may appear. |
| `sort` | `"relevance"` leaves the current match order. `"newest"` and `"oldest"` sort matched IDs by `created_at` string. Other values are ignored and leave order unchanged. Only matched IDs are sorted, not neighbors. |
| `namespace` | Optional namespace prefix. Matches must have `nid.startswith(f"{namespace}:")`. Applied only to matched nodes. |
| `tag` | Optional tag filter mapping. Applied only to matched nodes via `_filter_by_tags`. For key `"tag"`, values are exact full tags. Other keys are prefixed as `key:value`. Filter groups are ANDed; values within a group are ORed. |
| `limit` | Maximum number of matched IDs after filters/sort/pagination. Not a cap on neighbors. `None` means no limit. |
| `offset` | Number of matched IDs to skip before expansion. Applies to matched IDs only. |
| `regex` | If true, match using Python regex on node ID and text. Emits an experimental warning to stderr. Invalid regex raises `ValueError`. Ignored by the PostgreSQL branch. |
| `pg_conninfo` | If truthy, dispatch to PostgreSQL via `_pg_dispatch`. |
| `domain_id` | PostgreSQL domain identifier, passed through only in the PostgreSQL branch. |

---

## Return Value

Returns a `str` on the SQLite path. The PostgreSQL path returns whatever `PgApi.search(...)` returns, expected to also be a string.

The exact string depends on `format`:

- `"markdown"`: markdown sections for matched nodes plus a related-neighbors section.
- `"json"`: JSON array of matched and neighbor nodes.
- `"minimal"`: plain text of matched and neighbor nodes.
- `"compact"`: one line per node: `[STATUS] id — text`.
- `"names-only"`: bare IDs, one per line. This skips neighbor expansion entirely.
- Unknown format: markdown.

`"No results found."` is returned when no matches remain after matching or after one of the early filters.

There is a subtle edge case: if pagination empties `matched_ids` after filters, the early `"No results found."` checks have already passed. Then:

- `names-only` returns `"No results found."`
- `compact` returns `"No results found."`
- `json` returns `"[]"`
- `markdown` and `minimal` can return an empty string.

Callers should therefore treat both `"No results found."` and, for certain formats, an empty string as possible no-result outputs.

---

## Algorithm

### 1. PostgreSQL branch

If `pg_conninfo` is truthy:

- Reject `depth != 1` with `NotImplementedError`.
- Reject truthy `tag` with `NotImplementedError`.
- Dispatch to `PgApi.search` with only:
  - `query`
  - `visible_to`
  - `format`
  - `include_out`

Notably, `sort`, `namespace`, `limit`, `offset`, and `regex` are not passed to PostgreSQL.

### 2. Open the SQLite network

Use:

```python
with _with_network(db_path) as net:
```

This loads or reuses a cached `Network` keyed by database path and mtime. The call is read-only.

### 3. Determine matched IDs

Depending on the query mode:

#### Regex mode

If `regex=True`:

- Print an experimental warning to stderr.
- Call `_regex_search(query, net)`.
- This compiles the query with `re.IGNORECASE`, searches node IDs and node text, and returns matching IDs in sorted node order.
- Invalid regex raises `ValueError`.

#### OR mode

If `regex=False` and the query contains `|`:

- Call `_or_search(query, db_path, net)`.
- Split the query on literal `|`, strip terms, discard empty terms.
- For each term:
  - Try `_fts_search(term, db_path)`.
  - If no FTS matches, fall back to `_substring_search(term, net)`.
- Union all IDs and return them sorted lexicographically.
- This means OR results are not in FTS relevance order by default; default `sort="relevance"` leaves the lexicographic order intact.

#### Default FTS mode

Otherwise:

- Call `_fts_search(query, db_path)`.
- If that returns no IDs, fall back to `_substring_search(query, net)`.

`_fts_search` details:

- Tokenizes with `\w+`.
- Drops stop words and one-character terms when possible.
- Builds an FTS5 query by quoting each remaining term and joining with spaces, which FTS5 treats as an all-terms query.
- Orders by FTS rank and limits to 200 IDs per query.
- If there are more than two terms and no results, progressively relaxes by dropping terms, down to at least half the original term count, with a budget of 50 relaxation queries.
- Catches `sqlite3.OperationalError` and `sqlite3.DatabaseError` and returns `[]`, which triggers substring fallback.

`_substring_search` details:

- Lowercases the query.
- Matches if the query substring appears in the lowercase node ID or lowercase node text.
- Iterates over sorted nodes, so results are lexicographic.
- An empty query matches every node because `"" in text` is always true.

### 4. Early no-result check

If `matched_ids` is empty, return `"No results found."`.

### 5. Filter matches

Apply filters in this order:

1. **OUT filter**
   - If `include_out=False`, keep only IDs that exist in `net.nodes` and whose truth value is not `"OUT"`.
   - If all removed, return `"No results found."`.

2. **Namespace filter**
   - If `namespace` is set, keep IDs starting with `f"{namespace}:"`.
   - If all removed, return `"No results found."`.

3. **Access filter**
   - If `visible_to is not None`, keep IDs whose node exists and `_is_visible(node, visible_to)` is true.
   - If all removed, return `"No results found."`.

4. **Tag filter**
   - If `tag` is truthy, call `_filter_by_tags(matched_ids, tag, db_path)`.
   - If all removed, return `"No results found."`.

### 6. Sort matches

Only if `sort` is `"newest"` or `"oldest"`:

- Reverse order for `"newest"`.
- Sort by `net.nodes.get(nid).created_at` or `""` if missing.
- Missing nodes do not raise here because `getattr(None, "created_at", "")` yields an empty string.

For any other `sort` value, including `"relevance"`, the existing match order is preserved.

### 7. Paginate matches

Apply `offset` and `limit` to `matched_ids` before neighbor expansion.

This means:

- Skipped matches do not contribute neighbors.
- Limited matches only contribute neighbors.
- Neighbor count itself is not limited.

### 8. Names-only short-circuit

If `format == "names-only"`:

- Return `_format_names_only(matched_ids, set())`.
- This skips neighbor expansion entirely, even if `depth > 0`.
- Returns bare IDs newline-separated, or `"No results found."` if empty.

### 9. Neighbor expansion

Perform BFS for `depth` hops:

- Start with `frontier = set(matched_ids)`.
- Start `visited = set(matched_ids)`.
- For each hop:
  - For each node in the frontier:
    - Traverse all justification antecedents.
    - Traverse all `node.dependents`.
  - Add discovered IDs to `neighbor_ids` and the next frontier if they exist in `net.nodes`, are not already visited, and are not already matched.
  - Mark the next frontier as visited and continue.
- After expansion, subtract matched IDs from `neighbor_ids`.

Important behavior:

- Traversal is not pruned by OUT status or visibility. OUT or invisible intermediate nodes may be traversed to reach visible/included neighbors at greater depth.
- `namespace` and `tag` filters are not applied to neighbors.
- `visible_to` and `include_out` are applied only after expansion is complete.

### 10. Filter neighbors

- If `include_out=False`, remove OUT neighbors.
- If `visible_to is not None`, remove neighbors not visible to the caller.
- Neighbor IDs are guaranteed to exist in `net.nodes` by the expansion conditions, though the final visible check also rechecks existence.

### 11. Format and return

Dispatch based on `format`:

- `"json"` → `_format_json`
- `"minimal"` → `_format_minimal`
- `"compact"` → `_format_compact`
- Anything else, including `"markdown"` and unknown values → `_format_markdown`

Formatting behavior:

- Markdown: full sections for matches, plus sorted related-node bullets.
- JSON: array of objects; matched nodes have `"match": true`, neighbors have `"match": false` and `"relation": "neighbor"`.
- Minimal: just text lines.
- Compact: `[STATUS] id — text` lines.
- Names-only was already handled earlier.

---

## Side Effects

- Opens/loads the SQLite database via `_with_network`.
- Updates the in-memory `_network_cache` with `(mtime, network)` when the context exits.
- Opens additional temporary SQLite connections in `_fts_search` and `_filter_by_tags`.
- `regex=True` prints a warning to `stderr`: `"⚠️  Regex search is experimental and may be slow on large networks."`.
- Does not write to the SQLite database.
- Does not mutate the cached `Network` in the SQLite path.
- PostgreSQL path opens a `PgApi` context and delegates to `PgApi.search`.

---

## Error Handling

- `NotImplementedError`:
  - PostgreSQL + `depth != 1`.
  - PostgreSQL + truthy `tag`.

- `ValueError`:
  - Invalid regex, raised from `_regex_search`.

- SQLite errors:
  - `_fts_search` catches `sqlite3.OperationalError` and `sqlite3.DatabaseError` and returns `[]`, allowing substring fallback.
  - `_filter_by_tags` does not catch SQLite errors.
  - `_with_network` / `Storage` errors can propagate.

- `KeyError`:
  - Possible in formatting if `matched_ids` contains IDs not present in `net.nodes`.
  - This is most likely when `include_out=True` and FTS returns a stale ID, because the `include_out=False` filter checks `nid in net.nodes` but the `include_out=True` path does not.
  - `names-only` does not index `net.nodes`, so it can tolerate stale IDs.

- No exception for:
  - Unknown `format`; treated as markdown.
  - Unknown `sort`; original order retained.
  - Negative `depth`; no expansion because `range(depth)` is empty.
  - Negative `offset`; Python slice semantics apply.
  - Negative `limit`; Python slice semantics apply.
  - Empty query; usually matches all nodes through substring fallback, unless the query is an OR query with no terms.

---

## Usage Patterns

Typical calls:

```python
search("authentication flow", db_path="reasons.db")
search("auth|login", db_path="reasons.db", format="json")
search("test.*plan", db_path="reasons.db", regex=True)
search("security", db_path="reasons.db", namespace="agent-a", limit=10)
search("security", db_path="reasons.db", tag={"topic": ["security"]})
search("security", db_path="reasons.db", format="names-only")
```

Caller obligations:

- If using `format="json"`, parse the returned JSON string.
- If using PostgreSQL, avoid `depth != 1` and truthy `tag`.
- Remember that `limit` limits matched nodes only, not total output including neighbors.
- Do not expect neighbor nodes to respect `namespace` or `tag`.
- Be aware that `format="names-only"` ignores `depth`.
- If relying on visibility, pass `visible_to`; otherwise access filtering is skipped.
- If relying on OUT inclusion, pass `include_out=True`; otherwise OUT nodes are filtered.
- Check for `"No results found."` and, for some formats, possibly an empty string.

---

## Dependencies

Internal dependencies:

- `_with_network` — read-only database load/cache context.
- `_fts_search` — FTS5 search with progressive relaxation and fallback signal.
- `_substring_search` — substring fallback.
- `_or_search` — pipe-based OR search.
- `_regex_search` — regex search.
- `_warn_regex_experimental` — stderr warning.
- `_filter_by_tags` — tag filtering using the `node_tags` table.
- `_is_visible` — access-tag visibility check.
- `_format_markdown`, `_format_json`, `_format_minimal`, `_format_compact`, `_format_names_only`.
- `_pg_dispatch` and `PgApi` for PostgreSQL.
- `Network` objects with `nodes`, `justifications`, `dependents`, and `created_at`.

External / stdlib dependencies:

- `sqlite3`
- `re`
- `json`
- `sys`
- `itertools.combinations`
- SQLite FTS5 via the `nodes_fts` table
- Optional PostgreSQL backend

---

## Assumptions Not Enforced by Types

- `format` is not validated; unknown strings silently become markdown.
- `sort` is not validated; unknown strings silently leave match order unchanged.
- `limit` and `offset` are not validated as non-negative integers.
- `depth` is not validated as a non-negative integer on SQLite.
- `tag` values are assumed to be lists of strings, but this is not enforced.
- `visible_to` is assumed to be a list of strings when not `None`; passing a string would be interpreted as a set of characters by `_is_visible`.
- FTS result IDs are assumed to exist in `net.nodes` when `include_out=True` and no later existence-checking filter is applied.
- Neighbor expansion is assumed to be acceptable even when it traverses through OUT or invisible intermediate nodes.
- `namespace` and `tag` are assumed to describe only matched results, not related neighbors.
- PostgreSQL branch assumes `PgApi.search` ignores or does not require parameters that `search` does not forward.

---

## Topics to Explore

- [function] `reasons/api.py:_fts_search` — Explains FTS5 tokenization, stop-word removal, progressive relaxation, and the 200-result per-query cap.
- [function] `reasons/api.py:_filter_by_tags` — Details the AND/OR tag semantics and the exact `tag` vs prefixed-key behavior.
- [function] `reasons/api.py:_format_markdown` — Shows exactly how matched nodes and related neighbors are rendered and where missing-node assumptions matter.
- [function] `reasons/api.py:_with_network` — Covers the mtime-keyed network cache, read-only behavior, and database lifecycle.
- [function] `reasons/pg.py:PgApi.search` — Determines what PostgreSQL actually supports and which `search` parameters are silently ignored in the PG branch.

## Beliefs

- `search-paginates-before-neighbor-expansion` — `offset` and `limit` are applied to matched IDs before BFS neighbor expansion, so skipped matches never contribute neighbors.
- `search-neighbor-expansion-uses-antecedents-and-dependents` — Neighbor BFS traverses justification antecedents and `node.dependents`, with a visited set to avoid cycles.
- `search-namespace-and-tag-filters-do-not-apply-to-neighbors` — The `namespace` and `tag` parameters filter only matched IDs, not expanded neighbor IDs.
- `search-postgresql-branch-ignores-sort-namespace-limit-offset-regex` — The PostgreSQL dispatch only forwards `query`, `visible_to`, `format`, and `include_out`; the other parameters are not passed.
- `search-empty-query-falls-back-to-match-all-substring` — An empty non-OR query causes `_fts_search` to return `[]`, then `_substring_search("")` matches every node because `"" in text` is always true.
