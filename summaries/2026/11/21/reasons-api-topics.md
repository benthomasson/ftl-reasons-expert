---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: topics in reasons/api.py

# `topics` (`reasons/api.py:topics`)

## Purpose

`topics` provides a lightweight, read-only "word cloud" over the **node IDs** of a belief network. It counts how often each whitespace-free token appears across the IDs (splitting on `-`, `.`, `_`, `:`), filters out stop words and short tokens, and returns the top-N tokens by frequency. Its primary consumer today is `build_wiki` (`reasons/api.py` → `topics(limit=max_topics, db_path=db_path)`), which feeds the ranked topic list into `_assign_topics` to group beliefs into wiki pages. It exists so any caller (CLI, tools, LangGraph) can get a zero-config, no-LLM topical summary of a network using only ID conventions.

## Contract

**Preconditions**
- Either `pg_conninfo` is a valid connection string (PostgreSQL backend path), or `db_path` points to an existing SQLite database created by the RMS `Storage` class. (The function itself does not verify `db_path` existence; `Storage(db_path)` does.)
- `limit` is intended as a positive integer, but the type system permits any `int`.

**Postconditions**
- Returns a JSON-serializable `dict`.
- `result["topics"]` is a list of at most `limit` dicts, each `{"topic": str, "count": int}`.
- `result["total_nodes"]` equals `len(net.nodes)` — the **unfiltered** node count of the network.
- The database is **not mutated**.

**Invariants**
- Every token in `result["topics"]` has `len(token) >= 3` and is not in `_TOPIC_STOP_WORDS`.
- Topic counts are strictly non-increasing in order; ties are alphabetically ascending.
- Tokens preserve original casing — no lowercasing is performed before counting.

## Parameters

| Name | Type | Meaning / edge cases |
|---|---|---|
| `limit` | `int = 20` | Max number of topics returned. Applied as list slicing **after** ranking. `limit=0` → empty `topics`; `limit<0` → Python `[:-1]` semantics (returns all but last). Not clamped. |
| `db_path` | `str = "reasons.db"` | SQLite file path. Ignored when `pg_conninfo` is truthy. |
| `pg_conninfo` | `None` / connection string | If truthy, dispatch to `PgApi.topics(limit=limit)`. Only `limit` is forwarded; `db_path` is not. |
| `domain_id` | `None` / str | Forwarded to `_pg_dispatch` for PostgreSQL multi-tenancy. Ignored on the SQLite path. |

## Return Value

Always returns a dict of the shape:

```python
{
    "topics": [{"topic": "auth", "count": 42}, {"topic": "session", "count": 17}, ...],
    "total_nodes": 314,
}
```

Callers must handle:
- An **empty** `topics` list (e.g., when `limit <= 0`, when the network is empty, or when every token is a stop word / length ≤ 2).
- The fact that `total_nodes` **excludes nothing** — it is not the number of matched nodes; it's the entire node set. A network where only 3 nodes contribute is still reported with the total.
- Strings in `topics` are **raw tokens from IDs**, so `agent:belief-foo` yields tokens like `agent`, `belief`, `foo`, not `agent:belief-foo`.

## Algorithm

1. **Backend branch**: if `pg_conninfo` → `_pg_dispatch(pg_conninfo, domain_id, "topics", limit=limit)`. Exit.
2. **Open read-only context**: `with _with_network(db_path) as net:`. Because `write=False` (default), the context manager may serve a cached `Network` if the file mtime matches; otherwise it instantiates `Storage(db_path).load()`. No save is performed on exit.
3. **Accumulate word counts**: iterate every node ID (`net.nodes` dict keys are ID strings):
   - `re.split(r'[-._:]', nid)` — split on any of `-`, `.`, `_`, `:`. Inside the character class, `-` is the first char so it is a literal hyphen (not a range).
   - Skip empty tokens (`foo--bar` produces `["foo","","bar"]`).
   - Skip tokens where `len(word) <= 2`.
   - Skip tokens present in `_TOPIC_STOP_WORDS`.
   - Otherwise `word_counts[word] += 1`. Note: this is **per-token occurrence per node ID**, so `foo-foo` counts as `foo: 2`, not `foo: 1`.
4. **Rank**: `sorted(word_counts, key=lambda w: (-word_counts[w], w))` → descending count, ascending lexicographic tie-break. Then `[:limit]`.
5. **Shape response**: build the list of `{topic, count}` dicts in rank order, plus `total_nodes = len(net.nodes)`.

## Side Effects

- **Read-only** access to the SQLite DB. May populate / read `_network_cache` (keyed by `db_path`, mtime-tagged). Never writes to the DB itself.
- When `pg_conninfo` is truthy, opens a `PgApi` context and issues a remote call — external I/O.
- **No logging**, no stdout.

## Error Handling

Not swallowed — the caller sees:
- `FileNotFoundError` / `sqlite3.OperationalError` (or whatever `Storage(db_path)` raises) if the SQLite file is missing or unreadable.
- `AttributeError` if `PgApi` does not implement a `topics` method (the dispatch uses `getattr(pg, method_name)`).
- Any connection/exception raised by `PgApi(...)` on the PG path.
- `re.error` is not reachable — the pattern is a constant, valid char class.

There is **no special handling** for empty input: it simply returns `{"topics": [], "total_nodes": <n>}`.

## Usage Patterns

Typical call:

```python
from reasons.api import topics

result = topics(limit=10, db_path="reasons.db")
for t in result["topics"]:
    print(t["topic"], t["count"])
```

**Caller obligations**
- Treat `topics` as a heuristic. If node IDs do not follow `-`/`.`/`_`/`:` conventions, results will be one big blob per ID.
- Do not assume the tokens are normalized (case is preserved; pluralization is preserved).
- For PG, ensure the backend implements `topics` and accepts `limit`.

Known in-repo caller: `build_wiki(...)` → `topics(limit=max_topics, db_path=db_path)` → `_assign_topics(node_ids, topics_result["topics"])`. Also covered by `tests/test_topics.py` and `tests/test_build_wiki.py`.

## Dependencies

- `re` (stdlib) — tokenization.
- `_TOPIC_STOP_WORDS` — module-level frozenset-like stop-word list (defined immediately above the function).
- `_with_network` — the load/cache context manager (`reasons/api.py`).
- `_pg_dispatch` — PG backend bridge (`reasons/api.py`); pulls in `reasons.pg.PgApi`.
- `Storage` / `Network` (`reasons.storage`, `reasons.network`) — reached transitively through `_with_network` when the cache misses.

## Assumptions Not Enforced by Types

- **Node ID naming convention**: `-`, `.`, `_`, `:` are the token separators. Anything else (spaces, `/`, `,`) stays glued to the token.
- **`limit >= 0`**: negative `limit` silently produces "all but last N" via Python slice semantics — no `ValueError`.
- **Case consistency**: no lowercasing means `Foo` and `foo` bucket separately; if IDs are mostly lowercase this is fine.
- **`pg_conninfo` implies a valid `PgApi.topics(limit=...)`** exists — a missing method becomes an `AttributeError`, not a nice error.
- **`total_nodes` is a design choice**, not tied to the number of nodes that contributed a topic.
- **Token length semantics**: `> 2` means a 3-character token is kept; a 2-char token like `id` is dropped even though it may be meaningful.

---

## Topics to Explore

- [file] `reasons/api.py` — The `_TOPIC_STOP_WORDS` constant and `topics` caller `build_wiki` both live here; read the whole `topics`/wiki pipeline for context.
- [function] `reasons/api.py:_with_network` — The caching context manager that determines whether `topics` hits the file or a cached `Network`.
- [function] `reasons/api.py:_pg_dispatch` — How the PostgreSQL branch is wired, and where `PgApi.topics` must exist.
- [file] `tests/test_topics.py` — Concrete assertions on splitting, stop-word filtering, ranking, and low-`limit` behavior.
- [general] `postgresql-backend-parity` — Explore how the SQLite and PG paths stay consistent, especially where arguments like `db_path` are silently ignored.

## Beliefs

- `topics-splits-node-ids-on-hyphen-dot-underscore-colon` — Node IDs are tokenized with `re.split(r'[-._:]', nid)`, so those four characters are the only separators.
- `topics-keeps-only-tokens-length-3-or-more` — Tokens with fewer than 3 characters are excluded before counting.
- `topics-ranks-by-count-then-alphabetical` — Results are sorted by descending count, with ascending lexicographic order as tie-break.
- `topics-is-case-sensitive` — Tokens are compared and bucketed as-is; no lowercasing is applied.
- `topics-pg-conninfo-bypasses-sqlite` — When `pg_conninfo` is truthy, `db_path` is unused and the call is dispensed through `_pg_dispatch(..., "topics", limit=limit)`.
- `topics-total-nodes-is-unfiltered` — `total_nodes` returns `len(net.nodes)` for the whole network, not the number of nodes contributing tokens.
