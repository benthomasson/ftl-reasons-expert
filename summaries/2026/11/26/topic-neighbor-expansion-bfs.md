---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The `depth`-driven BFS in `search()` that determines which IDs end up in `neighbor_ids`.

## How `depth` shapes `neighbor_ids` in `search()`

There are two `search()` definitions in the observed code:

- `reasons/api.py:3015` — the API implementation with `depth: int = 1`
- `reasons/mcp_server.py:58` — an MCP wrapper also exposing `depth: int = 1`

The depth-driven neighbor expansion described here is in the API implementation, `reasons/api.py:search`. The parameter is documented at `reasons/api.py:3041` as:

> “number of hops to expand along justification chains (default: 1)”

At a high level, the observed flow is:

1. `search()` first computes the direct query matches, referred to as `matched_ids`. The exact matching code is not included in the observations.
2. It initializes an empty neighbor set:
   - `neighbor_ids = set()` at `reasons/api.py:3123`
3. It runs one expansion pass per hop:
   - `for _ in range(depth):` at `reasons/api.py:3126`
4. Inside each expansion pass, the visible code adds IDs reached through two directions:
   - `neighbor_ids.add(ant_id)` at `reasons/api.py:3135` — antecedent IDs reached by walking justification chains backward.
   - `neighbor_ids.add(dep_id)` at `reasons/api.py:3139` — dependent IDs reached through the reverse dependency index.
5. Because `neighbor_ids` is a `set`, duplicate IDs found through multiple paths or multiple hops are deduplicated.
6. After expansion, direct matches are removed from the neighbor set:
   - `neighbor_ids -= set(matched_ids)` at `reasons/api.py:3145`
   
   This makes the final `neighbor_ids` disjoint from `matched_ids`.
7. Additional filtering is applied at `reasons/api.py:3149` and `reasons/api.py:3154-3155`. The exact predicates are not visible in the observations.
8. The resulting `neighbor_ids` set is passed to each output formatter:
   - `_format_json(...)` at `reasons/api.py:3160`
   - `_format_minimal(...)` at `reasons/api.py:3162`
   - `_format_compact(...)` at `reasons/api.py:3164`
   - `_format_markdown(...)` at `reasons/api.py:3166`

So `depth=1` gives one hop of context around each match, `depth=2` gives up to two hops, and `depth=0` would leave `neighbor_ids` empty because the loop body never runs. Note that the PostgreSQL path explicitly rejects non-default depth:

- `if depth != 1: raise NotImplementedError("depth is not supported with PostgreSQL")` at `reasons/api.py:3052-3053`

That means depth-driven neighbor expansion beyond one hop is only supported on the non-PostgreSQL path.

### What the observations do not show

The observations contain line markers around the loop, but not the full loop body from `reasons/api.py:3126` through `3144`. Therefore we cannot verify:

- the exact frontier/visited bookkeeping used for BFS,
- whether traversal starts from `matched_ids` or another set,
- whether all justifications are followed or only active ones,
- the exact filtering conditions at `reasons/api.py:3149` and `3154-3155`.

The visible evidence is enough to confirm that `depth` controls repeated expansion passes and that antecedent/dependent IDs are accumulated into `neighbor_ids`, but not enough to reconstruct the exact BFS implementation line-by-line.

## Topics to Explore

- [file] `reasons/api.py` — Read the full `search()` body around lines 3120–3168 to see the actual frontier, visited-set logic, and neighbor filters.
- [function] `reasons/api.py:search` — The main entry point that owns `depth` semantics and `neighbor_ids` construction.
- [function] `reasons/mcp_server.py:search` — Check how the MCP wrapper passes or transforms `depth`.
- [file] `reasons/network.py` — Understand the justification/antecedent and `dependents` reverse-index structures that the BFS walks.
- [general] `depth-semantics` — Explore behavior for `depth=0`, `depth=1`, `depth>1`, and the PostgreSQL `depth != 1` limitation.

## Beliefs

- `search-depth-controls-neighbor-hops` — `reasons/api.py:search` expands neighbor IDs once per unit of `depth` via `for _ in range(depth)` at line 3126.
- `neighbor-ids-are-disjoint-from-matched-ids` — The function subtracts `matched_ids` from `neighbor_ids` at line 3145 before formatting output.
- `neighbor-expansion-follows-antecedents-and-dependents` — The visible additions are `ant_id` at line 3135 and `dep_id` at line 3139, indicating traversal through both justification antecedents and dependents.
- `postgres-search-rejects-non-default-depth` — `reasons/api.py:3052-3053` raises `NotImplementedError` when `depth != 1` on the PostgreSQL path.
- `neighbor-ids-feed-all-output-formatters` — `neighbor_ids` is passed to JSON, minimal, compact, and markdown formatters at `reasons/api.py:3160-3166`.
