---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _fts_query in reasons/api.py

# `_fts_query` — Explanation

```python
def _fts_query(conn, terms: list[str], limit: int = 200) -> list[str]:
    fts_query = " ".join(f'"{t}"' for t in terms)
    cursor = conn.execute(
        "SELECT id FROM nodes_fts WHERE nodes_fts MATCH ? ORDER BY rank LIMIT ?",
        (fts_query, limit),
    )
    return [row[0] for row in cursor.fetchall()]
```

## 1. Purpose

`_fts_query` is a thin, private wrapper around SQLite's FTS5 full-text index. It takes a list of pre-tokenized search terms, builds a safe FTS5 query string (every term wrapped in double quotes to force literal / phrase matching), executes it against the `nodes_fts` virtual table, and returns the matching node IDs ranked by FTS5 relevance.

It exists because the public search path (`search` → `_fts_search` → `_fts_query`) needs a single reusable "run these tokens against FTS5 and give me ranked IDs" primitive. Extracting it keeps the progressive-relaxation loop in `_fts_search` free of SQL-building noise.

## 2. Contract

**Preconditions (not enforced by types):**
- `conn` is an open, usable `sqlite3.Connection` bound to a database that contains an FTS5 virtual table named `nodes_fts` with an `id` column and FTS5 ranking support (`ORDER BY rank`).
- `terms` is a **non-empty** list of strings. The terms are already filtered and tokenized by the caller (`_fts_search` uses `re.findall(r'\w+', query)`, then removes `_STOP_WORDS` and single-character tokens).
- Each term contains only characters that survive the caller's `\w+` tokenizer — no embedded double quotes. (The f-string `f'"{t}"'` does **not** escape inner `"`; the caller's `\w+` discipline is what keeps this safe.)
- `limit` is a positive integer (SQLite silently accepts any int bindable value, including 0).

**Postconditions:**
- Returns a `list[str]` of node IDs, ordered from most relevant to least relevant (`ORDER BY rank` in FTS5 uses the default `bm25` ranking; ascending rank = best first).
- Result length is `≤ limit`.
- The connection is **not** closed or mutated — the caller owns its lifecycle.

**Invariants:**
- All phrases must be present in the document (FTS5 treats whitespace-separated quoted phrases as implicit AND).
- SQL injection is impossible for the terms because the FTS5 query string is passed as a bound parameter.

## 3. Parameters

| Name | Type | Meaning | Edge cases |
|------|------|---------|-----------|
| `conn` | `sqlite3.Connection` | Open connection to the RMS SQLite database. | Must have been created after schema setup registered the `nodes_fts` virtual table; otherwise `sqlite3.OperationalError`. |
| `terms` | `list[str]` | Pre-tokenized search terms. | `[]` produces `""`, which FTS5 rejects as a syntax error (`sqlite3.OperationalError`). The caller guards by returning early if empty. Terms with whitespace are treated as multi-word phrases. |
| `limit` | `int` (default `200`) | Upper bound on returned rows. | Negative numbers cause SQLite to behave as "no limit" (per SQLite's `LIMIT` semantics). `0` returns `[]`. |

## 4. Return Value

`list[str]` — the `id` values of matching nodes, ordered by FTS5 rank. Returns `[]` when no nodes match the query. On operational errors (missing table, malformed FTS5 query) this function raises — it does **not** swallow. The caller (`_fts_search`) is responsible for the `[]` fallback, and the caller's caller (`search`) falls back to `_substring_search` when the FTS path yields nothing.

## 5. Algorithm

1. **Build FTS5 query string**: `" ".join(f'"{t}"' for t in terms)`. Each term becomes a double-quoted phrase, and terms are separated by spaces. In FTS5 this is an AND of phrase queries — the doc must contain every quoted phrase.
2. **Execute**: `SELECT id FROM nodes_fts WHERE nodes_fts MATCH ? ORDER BY rank LIMIT ?` with `(fts_query, limit)` bound.
   - `nodes_fts MATCH ?` invokes the FTS5 match operator.
   - `ORDER BY rank` (a reserved FTS5 column) applies the table's ranking function — default `bm25()` — giving best matches first.
   - `LIMIT ?` bounds returned rows.
3. **Project**: `[row[0] for row in cursor.fetchall()]` extracts only the ID column.

Complexity is bounded by `LIMIT`; the underlying `MATCH` cost is FTS5-internal (inverted index scan, ~proportional to term frequency).

## 6. Side Effects

- Read-only SQL. No writes, no transactions, no locks beyond SQLite's normal read lock.
- Opens an implicit cursor (never explicitly `.close()`‑d — relies on GC; harmless since the caller closes the connection in `_fts_search`'s `finally`).
- No logging, no I/O beyond the DB.

## 7. Error Handling

This function has **no** try/except. It can raise:

- `sqlite3.OperationalError` — if `nodes_fts` doesn't exist (DB not migrated / FTS not enabled), or if `fts_query` is malformed (e.g., empty string from empty `terms`).
- `sqlite3.DatabaseError` — broader DB failure (corruption, closed connection, etc.).
- `sqlite3.ProgrammingError` — if `conn` is already closed.

All of these are caught upstream in `_fts_search`:

```python
except (sqlite3.OperationalError, sqlite3.DatabaseError):
    return []
```

So from `search()`'s perspective, FTS failures are non-fatal and trigger fallback behavior. If you call `_fts_query` directly (e.g., in a test), you must handle these yourself.

## 8. Usage Patterns

Only one caller (`_fts_search`):

```python
results = _fts_query(conn, terms)
if not results and len(terms) > 2:
    # progressive relaxation: try subsets of terms via combinations()
    for n in range(len(terms) - 1, min_terms - 1, -1):
        for combo in combinations(terms, n):
            results = _fts_query(conn, list(combo))
            if results:
                return results
```

And via `_or_search`, which splits the raw query on `|` and calls `_fts_search` per alternative, then unions the results.

**Caller obligations:**
- Open and eventually close the connection (typically `_fts_search`'s `try/finally`).
- Pass already-tokenized, stopword-filtered, non-empty `terms`.
- Treat `[]` as "no matches, consider fallback/relaxation" rather than an error.

## 9. Dependencies

- **`sqlite3`** (stdlib) — the driver.
- **FTS5 extension** — SQLite must be built with `SQLITE_ENABLE_FTS5` (standard on modern builds).
- **`nodes_fts` virtual table** — created by `Storage`'s schema setup (see `reasons/storage.py`). Not defined in this file.
- **`_STOP_WORDS` / `re.search(r'\w+', ...)`** — the caller in `_fts_search` does this filtering; `_fts_query` assumes it has already happened.
- The docstring on `_fts_search` mentions **porter stemming** — the `nodes_fts` table is configured with `tokenize='porter ...'`, so quoted phrases still go through stemming.

---

## Assumptions Not Enforced by Types

- `conn` is open and points at a DB with `nodes_fts` in place.
- `terms` are non-empty and free of `"` (only `\w+` characters).
- `limit` semantics follow SQLite (`0` → empty, negative → unbounded).
- The word "rank" in the SQL is FTS5's reserved ranking column — not a human-authored column named `rank`. If someone renames or replaces the FTS table with a non-FTS5 view lacking this magic column, the query silently changes meaning or errors.
- No FTS query operator injection is possible only because terms are `\w+` and quoted. If a future caller passes raw user text into `terms`, the double-quote wrapping is **not** sufficient protection against quotes embedded in tokens.

## Topics to Explore

- [function] `reasons/api.py:_fts_search` — Wraps `_fts_query` with tokenization, stopword filtering, and progressive-relaxation fallback; where the caller-side invariants are enforced.
- [function] `reasons/api.py:search` — The public entry point; shows the full fallback chain (FTS → substring → OR → regex) and how `_fts_query` results are filtered, sorted, paginated, and neighbor-expanded.
- [file] `reasons/storage.py` — Defines the `nodes_fts` FTS5 virtual table (tokenizer choice, columns, triggers for keeping it in sync). This is the schema `_fts_query` relies on.
- [general] `fts5-bm25-ranking` — Understand how `ORDER BY rank` maps to `bm25()` and why results are returned best-first.
- [general] `sqlite-fts5-query-syntax` — Learn how quoted phrases, implicit AND, and operators like `NEAR` / `OR` / `*` interact — critical for reasoning about whether quoting individual terms fully sanitizes the input.

## Beliefs

- `_fts-query-requires-nodes-fts-table` — The function fails with `sqlite3.OperationalError` if the `nodes_fts` FTS5 virtual table is missing, because the SQL names it directly in the `MATCH` clause.
- `_fts-query-implements-phrase-quoting` — Every term is wrapped in double quotes so FTS5 treats it as a literal phrase instead of an FTS query operator.
- `_fts-query-produces-rank-ordered-ids` — Returned IDs are sorted by FTS5's default ranking (bm25) via `ORDER BY rank`, so the most relevant match is first.
- `_fts-query-has-default-limit-200` — When the caller omits `limit`, at most 200 IDs are returned; `limit` is passed as a bound SQL parameter, not interpolated.
- `_fts-query-does-not-catch-sqlite-errors` — All SQLite exceptions propagate to the caller; only `_fts_search` converts them to `[]` as part of its fallback strategy.
