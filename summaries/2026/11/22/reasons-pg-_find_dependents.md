---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _find_dependents in reasons/pg.py

## `_find_dependents` (in `reasons/pg.py`)

### 1. Purpose

Reverse-lookup of the dependency graph. Given one or more node IDs, it answers: *"Which justification rows reference any of these IDs, and therefore which nodes might change truth value if these IDs change state?"*

A dependent node is any `rms_justifications` row whose **antecedents** (`SL`/`CP` "depends on") *or* **outlist** (`unless`-style exception) contains one of the query IDs. Both columns matter: the truth of a justification is `IN` iff all antecedents are `IN` *and* all outlist nodes are `OUT`. So mutating **either** an antecedent or an outlist node can flip a dependent's status.

This is the core primitive used by:
- `_propagate` (BFS cascade after a truth-value change)
- `show_node` (display `dependents`)
- `_expand_neighbors` (search neighbor expansion)
- `_collect_what_if_results`, `_cascade_depth_pg` (labels for what-if reports)
- `_find_culprits_internal`, `_entrenchment`, `add_nogood` (scoring / backtracking)

### 2. Contract

- **Precondition:** `cur` is an open psycopg v3 cursor bound to the same connection as `self.conn`; the caller is responsible for transaction boundaries (no commit here). `node_ids` is an iterable of node-ID values, in practice `str`.
- **Postcondition:** returns a **new** `set` of `rms_justifications.node_id` strings (deduplicated across all queries), restricted to `domain_id = self.domain_id`. No writes, no commit, no logging.
- **Invariants:** deterministic given DB state; runs one query per `node_id`; never includes justification-row `id`s or `type` — only the owning `node_id`. Self-references are *not* filtered out: if a node references itself, it will be in the result.

### 3. Parameters

- `self`: `PgApi` instance; `self.domain_id` scopes queries. Must be a valid UUID string (as stored, `str`).
- `cur`: psycopg v3 cursor, **already open** (a `with self.conn.cursor() as cur:` block). Not created here.
- `node_ids`: iterable of node-ID strings. Empty or falsy → immediate early return `set()`. Duplicates are *not* pre-deduplicated (each duplicate triggers a redundant round-trip; the `DISTINCT` and the `set` still make the result correct). Non-`str` values (e.g. `int`, `None`) will serialize to JSON but won't match TEXT node IDs because JSONB containment compares *typed* JSON values — the result would silently exclude them.

### 4. Return Value

- `set[str]` of dependent `node_id` values. Empty set if `node_ids` is empty or nothing references them.
- Order is unspecified (set semantics).
- If a node references itself it is included; the caller may need to `- {source}` if it wants "other" dependents.

### 5. Algorithm

1. Short-circuit: `if not node_ids: return set()` — avoids touching the DB on empty input.
2. Cache `pid = self.domain_id`.
3. Initialize `dep_ids = set()`.
4. For each `nid`:
   - Serialize `[nid]` to JSON (`needle = json.dumps([nid])`).
   - Execute parameterized SQL:
     ```sql
     SELECT DISTINCT node_id FROM rms_justifications
     WHERE domain_id = %s
       AND (antecedents @> %s::jsonb OR outlist @> %s::jsonb)
     ```
     The `@>` operator is JSONB **containment**: an array like `["a","b"] @> '["a"]'` is true. This correctly handles substrings too, because it compares array elements, not strings.
   - Add every `row[0]` to `dep_ids`.
5. Return `dep_ids`.

The `DISTINCT` inside each query and the Python `set` across queries are redundant-but-harmless dedup: `DISTINCT` guards against a node appearing in a single justification row (impossible here since `node_id` is unique per row) and against the *same* node appearing via both antecedents and outlist of the *same* row (still one row → one output). The `set` guards against a dependent appearing across multiple `nid` iterations.

### 6. Side Effects

- Issues **N** SQL queries (one per element of `node_ids`) against the caller's cursor. This advances the cursor; subsequent caller code must not assume a fresh cursor position.
- Consumes a network round-trip per ID. For `_propagate`, this means a batch of B IDs takes B queries per BFS level — an O(B) query cost even though the batch could be answered with a single `WHERE antecedents @> ANY(...)`-style query.
- No writes, no commits, no logging, no in-memory state changes on `self`.

### 7. Error Handling

- **No try/except.** Errors propagate:
  - `psycopg` connection/cursor errors (e.g. cursor already consumed, connection closed) bubble up.
  - `json.dumps([nid])` raises `TypeError` if `nid` is not JSON-serializable (e.g. a `set`). For typical `str` inputs, this cannot happen.
  - DB errors (missing table, permission) raise.
- Empty/falsy `node_ids` are swallowed normally into `set()` — the caller does **not** need to special-case empty input.
- `None` is not explicitly rejected; `json.dumps([None])` is `'[null]'`, which simply matches nothing (TEXT `node_id` never equals JSON `null`). Silent no-op rather than an error.

### 8. Usage Patterns

All call sites pass an **active cursor** and never commit inside this method; the surrounding method manages transaction lifetime:

- `_propagate`: `dep_ids = self._find_dependents(cur, batch) - visited` — hot path during cascade; batch is the current BFS frontier.
- `show_node`: `sorted(self._find_dependents(cur, [node_id]))` — display.
- `_expand_neighbors`: `neighbor_ids.update(self._find_dependents(cur, matched_ids))` — search neighbor expansion, batch of search hits.
- `_cascade_depth_pg`, `_collect_what_if_results`: counts of dependents for what-if report entries.
- `_find_culprits_internal`, `_entrenchment`, `add_nogood`: degree-centrality-ish scoring.

Caller obligations:
- Pass IDs that actually exist if you want meaningful results — the method does **no** existence check.
- Pass `str` IDs; other JSON types will silently match nothing.
- Expect O(|node_ids|) round-trips; batch callers who care about latency should pass already-deduplicated lists.
- Do not assume the cursor is untouched afterward.

### 9. Dependencies

- stdlib: `json` (`json.dumps`).
- database schema from `SCHEMA`:
  - `rms_justifications(node_id, domain_id, antecedents JSONB, outlist JSONB)` — column names are hard-coded.
- performance depends on `INDEXES`:
  - `idx_rms_justifications_antecedents` (GIN on `antecedents`)
  - `idx_rms_justifications_outlist` (GIN on `outlist`)
  - PostgreSQL must plan the `OR` across two GIN indexes (typically a Bitmap OR); without these indexes this becomes a full scan per query.
- psycopg v3 cursor (returns `tuple` rows here, not `dict_row` — the code uses positional `row[0]`).

### 10. Assumptions Not Enforced by the Type System

- `node_ids` items are **strings**; ints/None are accepted but silently yield no matches (JSONB `@>` is value-typed).
- `node_ids` values are **actual node IDs**; no FK check or join. Unknown strings simply return no rows.
- `antecedents`/`outlist` are JSONB **arrays of strings**; if a future migration stored them as comma-separated strings, `@>` would break silently.
- `cur` is on the **same connection/domain transaction** as every prior call in the calling method (e.g. mutations in `_propagate` are visible to subsequent `_find_dependents` calls because they're in the same transaction).
- The caller is **not** relying on cursor position — this method consumes `cur.fetchall()`.

---

## Topics to Explore

- [function] `reasons/pg.py:_propagate` — The main caller; BFS cascade that treats `_find_dependents` output as the frontier and reuses the same cursor.
- [function] `reasons/pg.py:_justification_valid_cached` — Explains *why* outlist nodes count as dependencies (the truth condition uses both antecedents and outlist).
- [file] `reasons/pg.py` — The full file to see how `SCHEMA`, `INDEXES`, and cursor management interact with read-only helpers like this one.
- [general] `jsonb-containment-queries` — Understanding PostgreSQL's `@>` semantics on JSONB arrays is essential for reasoning about correctness of `needle` matching.
- [general] `gin-index-or-optimization` — Whether the `antecedents @> ... OR outlist @> ...` query actually uses both GIN indexes is a performance-relevant question worth benchmarking.

## Beliefs

- `_find_dependents-uses-jsonb-containment` — The query relies on PostgreSQL's `@>` JSONB containment to match `["nid"]` against antecedents and outlist arrays.
- `outlist-nodes-are-dependents` — Both antecedents and outlist entries count as dependencies because either can change a node's truth value.
- `_find_dependents-performs-query-per-id` — The method issues one SQL round-trip per element of `node_ids`, not a batched `ANY(...)` query.
- `_find_dependents-requires-active-cursor` — Callers must supply an open cursor from the same connection; the method never creates, commits, or rolls back the transaction.
- `_find_dependents-empty-input-returns-empty-set` — A falsy `node_ids` returns `set()` without touching the DB; the guard is not an exception path.
- `non-string-node-ids-match-nothing` — Passing an `int` or `None` in `node_ids` serializes to a JSON number/null and will not match TEXT node IDs, silently producing fewer results.
