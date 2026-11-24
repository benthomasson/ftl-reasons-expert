---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: compact in reasons/api.py

# `compact` — Token-Budgeted Belief State Summary

## Purpose

`compact` produces a bounded, human/LLM-consumable *snapshot* of the belief network as a single string. It does not mutate the network; it just answers the question: *"Give me the most important IN beliefs I can afford to read within a token budget."*

Two output shapes are supported:
- **`format="default"`** — delegates to `reasons/compact.py:compact`, which renders beliefs (with dependency context) up to `budget`.
- **`format="names-only"`** — emits just node IDs, one per line, that fit a rough token budget.

This is used to feed an LLM context window with the state of a truth-maintenance network without exceeding cost/latency limits.

## Contract

**Preconditions**
- `db_path` exists and points to a valid RMS SQLite file (unless `pg_conninfo` is provided, in which case a PostgreSQL backend is used).
- `format` is *intended* to be `"default"` or `"names-only"`.

**Postconditions**
- Returns a `str`.
- The SQLite database file is not modified (no `write=True` on `_with_network`).
- In `names-only`, the sum of per-ID heuristic costs is ≤ `budget` (or the result is empty).

**Invariants**
- OUT nodes never appear when `include_out=False` (default).
- Nodes whose `metadata.access_tags` are not a subset of `visible_to` never appear when `visible_to` is provided.
- Any `Nogood` retained in the filtered subset must have *all* of its member node IDs present in the filtered subset.
- No validation is performed on `format` — an unrecognized value silently falls through to the default path.

## Parameters

| Param | Type | Default | Meaning |
|---|---|---|---|
| `budget` | `int` | `500` | Approximate token budget. In names-only this is a real ceiling (see cost heuristic); in default mode it is forwarded to `reasons.compact.compact`. |
| `truncate` | `bool` | `True` | Forwarded to the underlying `_compact`. **Ignored in `names-only` mode** — never referenced on that branch. |
| `visible_to` | `list[str] \| None` | `None` | Access-control tags. When provided, only nodes whose `access_tags` are all in this set (or that have none) are included. An empty list restricts to untagged nodes. |
| `include_out` | `bool` | `False` | When `False`, OUT beliefs are dropped before summary/listing. |
| `format` | `str` | `"default"` | `"names-only"` → bare IDs; anything else → default rendering path. |
| `db_path` | `str` | `"reasons.db"` | SQLite file target. |
| `pg_conninfo` | `Any` (truthy) | `None` | If set, calls dispatch to `PgApi` instead of SQLite. |
| `domain_id` | `Any` | `None` | Passed through to `PgApi` for PostgreSQL domain scoping. |

## Return Value

- **names-only**: newline-joined node IDs (sorted by ID via `list_nodes`'s internal `sorted(net.nodes.items())`). Empty result → `""` (not `None`).
- **default**: whatever `reasons.compact.compact(...)` returns — a formatted, budgeted summary string.
- **pg_conninfo + default**: the string returned by `PgApi.compact(...)`.

Callers must handle empty strings; there is no sentinel error for "nothing fit the budget."

## Algorithm

**Branch 1 — `format == "names-only"`:**

1. Derive `status_filter = None if include_out else "IN"` — i.e., only IN when OUT is excluded, otherwise no filter (returns both IN and OUT node IDs).
2. Call `list_nodes(...)` (works against either backend) to obtain `{nodes: [...]}`.
3. Iterate IDs in order; compute a heuristic per-ID cost:

   ```python
   cost = len(nid.split("-")) + 1
   ```

   For `"foo"` → `2`; for `"foo-bar-baz"` → `4`. The `+1` approximates a separator/newline token. This is **not** real tokenization.
4. Accumulate until `used + cost > budget`, then `break` — a strict greedy prefix. A large *early* ID can starve later, small IDs.
5. Return `"\n".join(lines)`.

**Branch 2 — PostgreSQL default path:**

```python
if pg_conninfo:
    return _pg_dispatch(pg_conninfo, domain_id, "compact",
                        budget=budget, truncate=truncate,
                        visible_to=visible_to, include_out=include_out)
```

Note `format` is *not* forwarded here — the PG method is only expected to implement the default format (names-only was already handled above).

**Branch 3 — SQLite default path:**

1. Import `reasons.compact.compact` as `_compact`.
2. Open the network read-only via `_with_network(db_path)`.
3. Decide whether filtering is required: `needs_filter = visible_to is not None or not include_out`.
4. If filtering is required:
   - Construct a *fresh* `Network`.
   - For each `(nid, node)` in `net.nodes.items()`, skip if `not include_out and node.truth_value == "OUT"`, and skip if `visible_to is not None and not _is_visible(node, visible_to)`.
   - Else assign `filtered.nodes[nid] = node` (reference copy — no deepcopy; safe because `_compact` is read-only).
   - Filter nogoods to those whose node IDs are *all* still present.
   - Call `_compact(filtered, budget=budget, truncate=truncate)`.
5. Else: `_compact(net, budget=budget, truncate=truncate)`.

## Side Effects

- **No DB writes.** `_with_network(db_path)` (without `write=True`) loads the network and may populate/refresh `_network_cache[db_path]` in process memory. The context manager's `__exit__` only calls `store.close()` and updates the mtime-keyed cache — never `store.save()`.
- **I/O:** SQLite file open/read (or PG connection via `_pg_dispatch` with `PgApi` context manager).
- **No logs, no stdout/stderr writes.**
- **No mutation of nodes/justifications.** The filtered `Network` shares node references with the cached network; this is only safe because the compact renderer is a pure read.

## Error Handling

- No `try/except` anywhere in the function.
- Possible propagations:
  - `FileNotFoundError` / `sqlite3.OperationalError` if `db_path` doesn't exist.
  - Errors from `PgApi` when `pg_conninfo` is set (connection failures, method not implemented, etc.).
  - Whatever `reasons.compact.compact` raises (e.g. internal formatting), or exceptions raised inside `list_nodes` for unsupported filters.
- **Silently tolerated:**
  - Unknown `format` values — treated as `"default"`.
  - An empty budget / budget smaller than the first ID — returns `""`.
- **Not tolerated:** missing DB, invalid access-tag metadata type, PG connection issues.

## Usage Patterns

- Typical call: `compact(db_path="reasons.db")` for a default IN-only, budgeted rendering.
- Cheap discovery: `compact(format="names-only", budget=200)` to obtain a set of IDs to feed into other tools (`show_node`, `compact` default, etc.).
- Access-scoped rendering: `compact(visible_to=["public"], include_out=False)`.
- Backend-agnostic: callers can pass `pg_conninfo`/`domain_id` and never touch `db_path`.
- **Caller obligations:**
  - Do not rely on `truncate` when `format="names-only"` — it is silently ignored.
  - Treat the output as opaque text; parse only one-ID-per-line in names-only mode.
  - Ensure `db_path` is initialized (`init_db`) or the PG backend implements the corresponding method.

## Dependencies

- `_with_network` (same module) — read-only cached network loader.
- `list_nodes` (same module) — used by names-only mode; also the source of `status`/`visible_to` filtering semantics.
- `_is_visible` (same module) — access-tag subset predicate.
- `Network` from `reasons.network` — used to build the filtered subset (fresh container, shared node references).
- `_pg_dispatch` (same module) — routes to `reasons.pg.PgApi`.
- `reasons.compact.compact` — the actual default-format renderer for SQLite.

## Assumptions not enforced by the type system

- `format` is one of two valid strings; any other value silently becomes `"default"`. No `ValueError` is raised.
- `_compact` is assumed pure/read-only; the filtered `Network` shares node objects with the cached network, so mutation would corrupt the cache.
- The names-only "cost" (`len(nid.split("-")) + 1`) is a proxy for tokens, not actual tokenization — real token count can differ significantly.
- In names-only mode, the greedy `break` assumes IDs are roughly interchangeable in importance; a large early ID blocks all later ones even if they'd fit.
- `pg_conninfo` implies the `PgApi` object implements both `compact` and `list_nodes` with compatible signatures; the type system cannot enforce this.
- `include_out=True` combined with `visible_to` still routes through the filter block. Because `needs_filter = visible_to is not None or not include_out`, any `visible_to` case enters the filter branch even when `include_out` is `True` — so access filtering and OUT inclusion are handled together.
- Nogood filtering preserves only "fully visible" contradictions; a partially-visible nogood (one hidden member) is dropped entirely rather than partially rendered.

## Topics to Explore

- [file] `reasons/compact.py` — Implements the default token-budgeted rendering that `compact` delegates to; understand how `budget` and `truncate` affect output shape.
- [function] `reasons/api.py:list_nodes` — Underpins names-only mode; also documents how status/visible_to filters behave across SQLite and PostgreSQL.
- [function] `reasons/api.py:_is_visible` — The access-tag subset predicate all filtering depends on; empty `visible_to` becomes "untagged only."
- [file] `reasons/pg.py` — Provides `PgApi.compact` and `PgApi.list_nodes`, the counterpart to the SQLite paths.
- [general] `token-budget-heuristics` — Compare the names-only cost formula against real tokenizers and `reasons.compact`'s budget accounting to see where budgets diverge.

## Beliefs

- `compact-names-only-ignores-truncate` — When `format="names-only"`, the `truncate` parameter is never referenced; only `budget` limits output.
- `compact-names-only-cost-is-heuristic` — Names-only per-ID cost equals `len(nid.split("-")) + 1`, an approximation rather than a real token count.
- `compact-excludes-out-by-default` — With `include_out=False` (the default), OUT nodes are excluded in every format, including `names-only` (via `status_filter="IN"`).
- `compact-filters-nogoods-transitively` — When filtering occurs, a `Nogood` is retained only if every node ID it references survives the filter.
- `compact-does-not-validate-format` — Any `format` value besides `"names-only"` silently takes the default code path; no validation error is raised.
- `compact-is-read-only` — The function opens the network without `write=True`, so it never persists mutations to SQLite; only the in-memory `_network_cache` may be updated.
