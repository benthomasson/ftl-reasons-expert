---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _or_search in reasons/api.py

# `_or_search` — Explanation

## Purpose

`_or_search` is the **OR/pipeline search primitive** used by the public `search()` function when the user's query contains a pipe character. Rather than treating a multi-word query as an AND of terms (which is what `_fts_search` does by default), it splits the query on `|` and performs an independent search per sub-query, then returns the **union** of all matches.

Example: `query="auth|token|oauth"` becomes three separate searches whose combined node IDs are returned, deduplicated.

Its job is narrow: it is a *search primitive* only — it does **not** filter OUT nodes, namespaces, access tags, paginate, or expand to neighbors. All of that is done by the caller (`search()`).

## Contract

**Preconditions**
- `query` is a `str`. If it contains `|`, callers route to this function (see `search()`: `elif "|" in query: matched_ids = _or_search(query, db_path, net)`).
- `db_path` points to a SQLite DB that may contain an FTS5 `nodes_fts` virtual table. If not, `_fts_search` degrades gracefully.
- `net` is a `Network` object (from `_with_network`) with a `.nodes` mapping of `node_id -> Node`, where each `Node` has `.id` and `.text`.

**Postconditions**
- Returns a `list[str]` of node IDs.
- The list is **sorted lexicographically** by node ID.
- No duplicates (guaranteed by the internal `set`).
- Returns `[]` if (a) no non-empty terms survived splitting, or (b) none of the terms matched anything.

**Invariants**
- Every term is searched **independently**; a term's result set is not affected by other terms.
- Substring fallback is **per-term**, not global (see "Error Handling" and "Usage Patterns" for implications).
- `_or_search` never mutates `net` and never writes to `db_path`.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `query` | `str` | Pipe-delimited OR query, e.g. `"foo|bar|baz"` | `""` or `" | "` yields `[]`. Whitespace around terms is stripped. Duplicate terms are effectively deduped by the result set. |
| `db_path` | `str` | Path to the RMS SQLite database; passed straight to `_fts_search`. | If the DB is missing, malformed, or lacks the FTS table, `_fts_search` swallows `OperationalError`/`DatabaseError` and returns `[]`, triggering fallback. |
| `net` | `Network` | In-memory network used only by `_substring_search`. | Assumed to be loaded from the same `db_path`; if not, the two result sets may not correspond. Must have `.nodes`; otherwise an `AttributeError` would propagate from the fallback. |

## Return Value

`sorted(all_results)` — a list of unique `node_id` strings in lexicographic order. Callers must be prepared to handle:

- An empty list (`[]`), which `search()` converts into `"No results found."` later.
- IDs returned by `_fts_search` that may no longer exist in `net.nodes` (if DB and cache drift) — the calling `search()` guards on this with `if nid in net.nodes`.
- Loss of FTS relevance ranking: `_fts_search` returns by `ORDER BY rank`, but the final `sorted(...)` discards that order.

## Algorithm

1. **Split & sanitize.** `query.split("|")` → strip each piece → drop empty strings. If the resulting `terms` list is empty, early-return `[]`.
2. **Accumulate into a set.** `all_results = set()` (implicit dedupe/union).
3. **Per-term search.**
   - Try FTS first: `_fts_search(term, db_path)`.
   - If that returns an empty list, fall back to `_substring_search(term, net)`.
   - `all_results.update(matched)`.
4. **Sort and return.** `sorted(all_results)` produces deterministic output.

Important subtleties of the loop body:

- **Per-term fallback, not global.** If a term has *any* FTS hit, the substring fallback is skipped for that term — even if the substring search would have found more. This mirrors the top level of `search()`, where `if not matched_ids: matched_ids = _substring_search(...)`.
- **`_fts_search` applies its own stop-word filtering + progressive relaxation** on the raw words inside each term. Because `_or_search` already split on `|`, a term like `"the auth flow"` will still be processed by `_fts_search` as multiple words, but a term like `"the"` alone may be dropped as a stop word (yielding `[]`), and then substring-search fallback kicks in.
- **FTS per-term limit of 200.** `_fts_query` uses `LIMIT 200` per term; the union is bounded by `200 × len(terms)` for FTS hits, plus unbound substring hits.

## Side Effects

- **Reads only.** No mutations to `net`, no writes to `db_path`.
- **New SQLite connections per term.** `_fts_search` opens and closes a fresh `sqlite3.connect(db_path)` for each term inside a `try/finally`. For a query like `"a|b|c|d|e"` this is five separate connections. Not a correctness issue, but a performance note worth flagging.
- **No logging.** Unlike `search()` which prints warnings for regex mode, `_or_search` is silent.

## Error Handling

- **Swallowed at the FTS layer.** `_fts_search` catches `sqlite3.OperationalError` and `sqlite3.DatabaseError` and returns `[]`. This includes malformed FTS syntax (e.g., a term containing a `"` that would break the quoted MATCH expression) — the caller transparently falls back to substring search.
- **Not caught at `_or_search` level.** Anything else raised inside `_substring_search` (e.g., `AttributeError` if `net` is malformed) or the `sorted()` call would propagate.
- **`ValueError`** is *not* raised here for malformed queries — unlike `_regex_search`, which raises `ValueError` on `re.error`. This is intentional: bad FTS terms silently degrade to substring search.

## Usage Patterns

- **Only caller today:** `search()` in the same module, dispatched via:
  ```python
  if regex: ...
  elif "|" in query:
      matched_ids = _or_search(query, db_path, net)
  else:
      matched_ids = _fts_search(query, db_path)
      if not matched_ids:
          matched_ids = _substring_search(query, net)
  ```
- **Called inside `_with_network(db_path)`** so `net` and `db_path` are consistent for the duration of the call.
- **Caller obligations:**
  - Filter OUT nodes, apply `visible_to`, `namespace`, `tag`, `include_out` logic after this returns.
  - Note that `include_out=False` filtering and access filtering happen *after* `_or_search`, so it may return IDs the caller then discards.
  - Check for `nid in net.nodes` before dereferencing (search does this).

## Dependencies

- `_fts_search` (same module) — FTS5-backed search; opens SQLite, uses `_fts_query`, `_STOP_WORDS`, and `itertools.combinations` for progressive relaxation.
- `_substring_search` (same module) — pure in-memory scan of `net.nodes` matching on lowercased id/text.
- Standard library: nothing direct, but the transitive dependency is `sqlite3` (via `_fts_search`).
- Caller context: `Network` object from `.network` (via `_with_network`).

## Assumptions Not Enforced by Types

1. **`net` corresponds to `db_path`.** The type checker cannot verify that the in-memory network was loaded from the same SQLite file. If the cache is stale (e.g., DB modified outside the process), FTS and substring results may disagree.
2. **`net.nodes` is a `dict`-like of node objects with `.id` and `.text`.** `_substring_search` assumes `nid.lower()` and `node.text.lower()` exist.
3. **`db_path` points to a readable SQLite DB.** Not checked here; `_fts_search` silently treats unreadable DBs as "no FTS results."
4. **FTS5 extension is available.** If not, every term falls through to substring fallback — a silent performance cliff. There is no explicit "FTS unavailable" diagnostic.

## Topics to Explore

- [file] `reasons/api.py` — See how `_or_search` plugs into the surrounding `search()` filter/sort/paginate/neighbor-expansion pipeline.
- [function] `reasons/api.py:_fts_search` — Understand FTS5 stop-word filtering, progressive relaxation, and why malformed queries degrade to `[]`.
- [function] `reasons/api.py:_substring_search` — Confirm the fallback scans `id` and `text` case-insensitively and returns sorted traversal order.
- [general] `fts5-rank-ordering` — Explore why the final `sorted(all_results)` discards the relevance order returned by `ORDER BY rank`, and whether callers (`sort="relevance"`) are affected.
- [file] `tests/test_api.py` — Check existing tests for expected OR-search behavior and whether dedup/sort semantics are asserted.

## Beliefs

- `or-search-implements-union-semantics` — `_or_search` implements OR/pipeline union search: each pipe-separated term is searched independently and the results are unioned.
- `or-search-falls-back-per-term-not-globally` — The substring fallback fires per-term when `_fts_search` returns `[]`, so any single FTS hit for a term suppresses substring fallback for that term.
- `or-search-sorts-by-id-not-rank` — The return value is `sorted(...)` lexicographically by node ID, discarding the FTS rank order produced by `_fts_query`.
- `or-search-requires-consistent-db-and-net` — The function queries `db_path` (FTS) and `net` (substring) independently and does not reconcile them, assuming they represent the same underlying state.
- `or-search-empty-terms-return-empty-list` — A query consisting only of pipes and/or whitespace returns `[]` without touching the database or network.
- `or-search-dedupes-via-set` — Duplicate IDs across terms are eliminated by accumulating into a `set` before the final `sorted` call.
