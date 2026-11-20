---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _fts_search in reasons/api.py

# `_fts_search` — FTS5 full-text search with progressive relaxation

## Purpose

Primary full-text search path for the `search()` API. It runs SQLite FTS5 `MATCH` queries against a `nodes_fts` virtual table, treating every surviving query term as a **required** term (implicit AND). If that strict query returns nothing and the query had more than two terms, it retries with progressively fewer terms (all combinations), bounded by a small budget. It exists so callers get *ranked* search results instead of the plain substring fallback (`_substring_search`), while remaining resilient to missing schemas, corrupt DBs, or empty queries.

## Contract

- **Preconditions** (not enforced):
  - `db_path` points to a SQLite file that has an FTS5 virtual table named `nodes_fts` with at least an `id` column and a valid FTS5 `rank` function.
  - That table has been populated by the storage layer (typically with a **porter** tokenizer, which is what the docstring's "porter stemming" refers to — this function does *not* set up stemming itself).
- **Postconditions**:
  - Returns `list[str]` of node IDs. All elements come from `nodes_fts.id`.
  - Results are ordered by FTS5 `rank` (bm25), capped at 200 per underlying query.
  - An empty list is returned for: no terms after filtering, DB open/query errors, or a legitimately empty result set — **all indistinguishable to the caller**.
- **Invariants**:
  - The function never raises on SQLite errors. All failures collapse to `[]`.
  - The connection is always closed (via inner `try/finally`) when it was successfully opened.
  - At most `1 + 50 = 51` FTS `MATCH` statements are issued per invocation.

## Parameters

- `query: str` — Raw user text. Tokenized with `re.findall(r'\w+', query)`, so punctuation, quotes, and FTS5 operators are stripped before they reach FTS. Unicode word characters *are* preserved.
- `db_path: str` — Path handed straight to `sqlite3.connect`. Not validated; a nonexistent path creates an empty DB and the subsequent FTS query fails with `OperationalError`, which is swallowed.

Edge cases:
- Empty or punctuation-only query → `[]`.
- Single-character query `"a"` → `[]` (both filters require `len > 1`).
- Query made entirely of stop words (`"the is a"`) → fallback keeps `["the", "is"]` (stop-word filter is bypassed on fallback), then FTS queries against those literal tokens.

## Return Value

- Non-empty `list[str]`: up to 200 IDs from the winning FTS query, in `rank` order.
- Empty `list[str]`: "no terms," "no matches," or "error." The caller cannot distinguish these and must treat them uniformly as **no FTS match**, which is what `search()` does — it falls back to `_substring_search` and `_or_search` does the same for each OR branch.

## Algorithm

1. Open a SQLite connection to `db_path` inside the outer `try`.
2. Tokenize the query with `re.findall(r'\w+', query)`.
3. Apply the primary filter: keep terms whose lowercase form is not in `_STOP_WORDS` **and** whose length exceeds 1.
4. If that yields nothing, fall back to "non-stopword filter ignored": keep terms with `len > 1` only.
5. If still nothing, return `[]`.
6. Run `_fts_query(conn, terms)` — this joins terms as `"t1" "t2" ...` (FTS5 AND) and does `ORDER BY rank LIMIT 200`.
7. If results are non-empty, return them (fast path, 1 query).
8. If empty **and** `len(terms) > 2`:
   - `min_terms = max(1, len(terms) // 2)` (for 3 terms → 1, for 4 → 2, for 6 → 3, …).
   - Loop `n` from `len(terms)-1` down to `min_terms` (inclusive).
   - For each `n`, iterate `itertools.combinations(terms, n)`:
     - Decrement the shared `_MAX_RELAXATION_QUERIES = 50` budget **before** running the query.
     - If budget drops below 0, return `results` (which is always `[]` at that point because non-empty results would have already returned).
     - Run `_fts_query` with the current combo. If non-empty, return immediately.
9. Return `results` (empty list if the whole relaxation fell through).

Subtle behaviors worth noting:

- **Combination order** preserves the term order from the raw query, so relaxation drops the *last* term first — a deliberate heuristic that favors earlier terms.
- **Early-return on first hit** means the first combination that returns anything wins; the function does *not* try all relaxation levels and pick the highest-ranked. Ranking is only applied *within* a single FTS query.
- Because `budget` is decremented before the query, the budget-exceeded branch always returns `[]` (a `_fts_query` that returned data would have short-circuited earlier).
- The 200-row `LIMIT` is per FTS query, not per relaxation step.

## Side Effects

- Opens a read-only-in-practice SQLite connection (no writes, no transactions beyond the implicit read).
- Reads the `nodes_fts` virtual table.
- Closes the connection deterministically via `finally`.
- Emits no logs, prints, or telemetry. Errors are silently discarded.

## Error Handling

- `sqlite3.OperationalError` and `sqlite3.DatabaseError` (both raised by `sqlite3.connect` on invalid paths, by missing tables, or by corrupt files) are caught at the outer try and mapped to `[]`.
- `conn.close()` is in an inner `finally`, and any exception from it propagates to the outer `except` and is likewise mapped to `[]`.
- Programming errors (`TypeError`, `AttributeError`, etc.) are **not** caught — those indicate real bugs and would propagate.
- FTS5 syntax errors from `_fts_query` cannot be triggered today because all tokens are `\w+` and are double-quoted, but the `OperationalError` catch would cover them if they ever were.

## Usage Patterns

- Primary call site: `search()` in the same file, which treats `[]` as "try substring fallback."
- Secondary call site: `_or_search`, which loops each pipe-separated subterm through `_fts_search` with its own substring fallback per branch.
- Callers must not attempt to distinguish "no match" from "error" — the contract intentionally collapses them.
- Callers must apply their own filtering afterwards (truth value, namespace, access tags, tag filters, pagination, neighbor expansion) — `_fts_search` does none of that.

## Dependencies

- **Stdlib**: `sqlite3`, `re`, `itertools.combinations`.
- **Same module**: `_fts_query` (helper that issues the `MATCH`), `_STOP_WORDS` (module-level frozenset).
- **Implicit (schema)**: a working FTS5 virtual table `nodes_fts` in the target DB, typically created by the storage layer with the porter tokenizer — see `reasons.storage.Storage` / its schema definitions.

## Assumptions not enforced by the type system

- `nodes_fts` exists and is populated and synchronized with `nodes`.
- `rank` is a valid FTS5 sort key (it is by default, but a custom schema could rename or shadow it).
- `db_path` is a SQLite file, not a directory, URL, or in-memory `":memory:"` handle the caller somehow expects to persist across calls.
- Callers do not need to distinguish "no match" from "error."
- The docstring's "porter stemming" is inherited from the schema, *not* applied here — this function assumes it and does not verify it.

---

## Topics to Explore

- [file] `reasons/storage.py` — Defines the `nodes_fts` FTS5 virtual table (likely with `tokenize = 'porter'`); this is where the "porter stemming" claim in the docstring actually originates.
- [function] `reasons/api.py:_fts_query` — The helper that builds `"t1" "t2"` and runs `MATCH ... ORDER BY rank LIMIT 200`; understanding the implicit AND semantics is essential for interpreting `_fts_search` results.
- [function] `reasons/api.py:search` — The primary caller; shows how `[]` from `_fts_search` triggers `_substring_search` fallback and how neighbor expansion and pagination are layered on top.
- [function] `reasons/api.py:_or_search` — Handles `foo|bar` queries by running each subterm through `_fts_search` with per-branch fallback; useful for understanding how OR composition interacts with relaxation.
- [function] `reasons/api.py:_STOP_WORDS` — The exact stop-word list feeding step 3; changing this changes the relaxation behavior because it determines which terms survive the primary filter.

## Beliefs

- `fts-search-swallows-sqlite-errors` — `_fts_search` catches `sqlite3.OperationalError` and `sqlite3.DatabaseError` and returns `[]`, so callers never see DB failures as exceptions.
- `fts-search-uses-and-semantics` — `_fts_query` joins terms with spaces inside FTS5, producing implicit AND matching; every term must appear for the first (non-relaxed) query to return anything.
- `fts-search-relaxation-budget-is-global` — The 50-query budget is shared across all relaxation levels, so wider term sets exhaust it before reaching lower `n` values.
- `fts-search-returns-empty-on-budget-exhaustion` — When the relaxation budget runs out, `_fts_search` returns `[]`, never a partial or best-effort result.
- `fts-search-filters-single-char-terms` — Terms of length ≤ 1 are discarded in both the primary filter and the stop-word fallback, so single-character queries always return `[]` regardless of the FTS index contents.
