---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: what_if_supersede in reasons/api.py

# `what_if_supersede` — Explanation

## Purpose

`what_if_supersede` is a **read-only simulator** for the supersede operation. It answers: *"If I created a successor node with this new text and superseded `old_id` with it, what would the truth-value ripple look like?"* — without touching the database.

It mirrors the write operation `supersede_with_text` (same auto-id generation, same metadata copy, same `net.supersede(...)` call), but wraps everything in a `write=False` network context and a `deepcopy` so the cached network is never mutated. The return payload is designed for proposal previews (see `propose_supersession`, which stores `what_if_supersede`'s output as `impact_json`).

## Contract

**Preconditions**
- `old_id` must exist in the network. Otherwise `KeyError`.
- `new_id` (if provided) is assumed to *not* exist — see "Assumptions" below; this is **not enforced**.
- The database at `db_path` must be readable (schema initialized).

**Postconditions**
- No rows are written, no mtime changes, and the module-level `_network_cache` entry is left pointing at an unmutated `Network`.
- Returns a JSON-serializable dict (all node ids are strings, all lists of dicts contain only primitives).
- `total_affected == len(retracted) + len(restored)`.
- `old_id` is **never** present in `retracted` or `restored` (explicitly skipped) — the simulation quantifies cascade effects *caused by* the supersede, not the supersede itself.
- `retracted` ∪ `restored` are disjoint (they test mutually exclusive truth-value transitions).

**Invariants preserved by the code**
- All affected node ids existed *before* the change (`for nid in before` — the newly-added `new_id` is never reported as an effect).
- Each entry is sorted by `(depth, id)` ascending.

## Parameters

| Name | Type | Meaning |
|---|---|---|
| `old_id` | `str` | The node to supersede. Must be a key in `net.nodes`. |
| `new_text` | `str` | Text of the newly-created successor node. |
| `new_id` | `str \| None` | Optional successor id. If falsy, auto-generates `{old_id}-v2`, `-v3`, … skipping existing ids. |
| `db_path` | `str` | SQLite path. Ignored when `pg_conninfo` is set. |
| `pg_conninfo` | optional | If truthy, dispatches to `PgApi.what_if_supersede` and skips all local logic. |
| `domain_id` | optional | Passed through to the PostgreSQL dispatcher. |

Edge cases:
- `old_id` missing → `KeyError(f"Node '{old_id}' not found")`.
- Old node already `OUT` → returns `already_out: True` and `new_id` **exactly as passed** (may be `None`). Callers rendering the response must tolerate `new_id is None`.
- `new_id` collides with an existing node → **not guarded**. The auto-generation loop only protects the *auto-generated* path. Callers passing an explicit `new_id` are responsible for uniqueness (`net.add_node` semantics will decide what happens — typically overwrite or error).

## Return Value

```python
{
  "old_id": str,
  "new_id": str | None,     # None only in the already_out short-circuit
  "already_out": bool,
  "retracted": [{"id": str, "text": str, "depth": int, "dependents": int}, ...],
  "restored":  [{"id": str, "text": str, "depth": int, "dependents": int}, ...],
  "total_affected": int,
}
```

- **`retracted`** — nodes that transitioned `IN → OUT` because `old_id` went OUT via the supersede outlist.
- **`restored`** — nodes that transitioned `OUT → IN` (typically `old_id` being IN was gating them via an outlist).
- **`depth`** — shortest BFS distance from `old_id` along `dependents` edges via `_cascade_depth`. Returns `0` when no path exists (common for restored/gated nodes, since they are often *blocked by* rather than *derived from* `old_id`).
- **`dependents`** — `len(node.dependents)` in the post-supersede graph (i.e. includes any edge added by supersede).

## Algorithm

1. If `pg_conninfo` is set → `_pg_dispatch(...)` and return. All local logic is bypassed.
2. Open the network **read-only** (`_with_network(db_path, write=False)`), which may reuse the mtime-keyed cache.
3. Check `old_id in net.nodes`; raise `KeyError` if not.
4. If `old_node.truth_value == "OUT"` → short-circuit with `already_out: True`. (Justified: superseding an already-OUT node cannot change any truth values, because the outlist violation is already effective.)
5. **Deep copy** the network. This is critical — subsequent mutations propagate through `dependents` sets and `justifications` lists, and the context manager will cache this object at `__exit__`.
6. If `new_id` is falsy, auto-generate `{old_id}-v2`, testing `in net.nodes` against the copy and incrementing `suffix` until a free id is found.
7. Snapshot `before = {nid: n.truth_value}` from the copy — includes every pre-existing node.
8. Re-fetch `old_node` from the copy (the earlier reference points to the original).
9. Build `metadata`: only `access_tags` is preserved. Other metadata (`source_type`, `example`, `accepted_pr`, etc.) is **dropped**.
10. `net.add_node(id=new_id, text=new_text, source=..., source_url=..., metadata=...)` — **no `justifications` argument**, so the new node is inserted as a **premise** (IN by default). This matches `supersede_with_text`'s behavior.
11. `net.supersede(old_id, new_id)` — non-transitive: only `old_id` is directly impacted. Ancestors in the supersession chain are *not* retracted (unlike `supersede(..., transitive=True)`).
12. Iterate `for nid in before` (excluding `old_id`): classify each truth-value transition into `retracted` (`IN→OUT`) or `restored` (`OUT→IN`), capturing `_cascade_depth(net, nid, old_id)` and `len(n.dependents)`.
13. Sort both lists by `(depth, id)`.
14. Return the payload.

## Side Effects

- **Database**: none. `write=False` prevents `store.save()`.
- **`_network_cache`**: on `__exit__`, the context manager writes `(mtime, self.network)` back to the cache. Because the mutations happened on the local `net = copy.deepcopy(net)` binding, `self.network` (the cached object) remains unmutated. In the fresh-load path, the freshly loaded network is cached. In the cache-hit path, the cached object is re-cached — no harm.
- **Logging / pubsub**: `pubsub.has_subscribers` is only invoked when `write=True`; this function triggers neither `compute_changes` nor `publish`.
- **`_cascade_depth`**: allocates temporary `deque` state per call; purely functional on the network.

## Error Handling

| Situation | Behavior |
|---|---|
| `old_id` missing | `KeyError(f"Node '{old_id}' not found")` — propagates. |
| `pg_conninfo` set but PG backend raises | Propagates from `PgApi`. **No local validation** happens on this path — e.g., a missing `old_id` becomes whatever the PG implementation does. |
| `new_id` collides | **Not handled** — the code will attempt `net.add_node` and let the underlying behavior decide. Callers passing explicit ids must pre-check (`supersede_with_text` and `propose_supersession` differ here — the proposal path explicitly raises `ValueError`). |
| Copy / DB load errors | Propagate. |
| Database missing schema | `Storage(...).load()` may raise `sqlite3.OperationalError` — propagates. |

There is **no try/except anywhere** — this function is "let it raise" style.

## Assumptions Not Enforced by Types

1. `new_id` (when supplied) does not collide with an existing node — no assertion, unlike `propose_supersession`.
2. `old_id` refers to an `IN` node when we want a non-trivial simulation; the code handles `OUT` explicitly, but callers comparing to `supersede_with_text` should note that in the real write path, `supersede_with_text` **does not** guard against already-OUT either — the difference is only in the reports.
3. `net.add_node` accepts `metadata=None` or `metadata={...}` — but this function passes `{}` as a bare dict even when the old node has no tags.
4. `net.supersede(old_id, new_id)` is **non-transitive** here. Callers using `transitive=True` on the real path will get a different result from this preview.
5. `_cascade_depth` returns `0` for any affected node not reachable via `dependents` from `old_id` — restored nodes are frequently depth 0, so sort-ordering puts them first among equals.
6. The new node is inserted as a **premise**, so if the old node was derived (had justifications), the successor loses that derivation structure. This mirrors `supersede_with_text` but is a modeling decision worth flagging.

## Usage Patterns

Called by:

- **`propose_supersession`** to populate `impact_json` when creating a pending supersession proposal. The stored `impact` field is later surfaced by `show_proposal` and used for human review.
- Scripts / CLI (`reasons what-if-supersede ...`) for interactive queries before committing.
- CLI parity: `supersede_with_text` (the write version) uses the same helper structure, so `what_if_supersede` is a preview of that commit.

Callers typically:
- Render `already_out` and short-circuit UI paths when `True`.
- Render `retracted`/`restored` as a list sorted by depth.
- Guard `new_id is None` in the `already_out` branch.

## Dependencies

- `_with_network` — read-only context manager with mtime cache.
- `_pg_dispatch` — PG backend dispatch.
- `_cascade_depth` — BFS depth along `dependents` edges.
- `Network.add_node`, `Network.supersede` — mutate the deep-copied network.
- Standard library: `copy.deepcopy`.
- `DEFAULT_DB` for default `db_path`.

## Topics to Explore

- [function] `reasons/network.py:Network.supersede` — The outlist mechanism this simulation drives; understanding it explains why the old node goes OUT and why gated nodes come back IN.
- [function] `reasons/api.py:supersede_with_text` — The write-path counterpart; comparing the two highlights exactly which state survives (`access_tags` only, no derivation) and where `new_id` collision handling differs.
- [function] `reasons/api.py:_with_network` — Cache semantics and the mtime guard; explains why `what_if_supersede` must `deepcopy` and why the DB never changes here.
- [function] `reasons/api.py:propose_supersession` — The primary consumer; shows how the impact report is stored and re-surfaced via `show_proposal`.
- [function] `reasons/api.py:_cascade_depth` — Depth derivation and the `0`-when-unreachable fallback that explains restored-node sort order.

## Beliefs

- `what-if-supersede-is-read-only` — Opens the network with `write=False` and never calls `store.save`, so the database is unchanged.
- `what-if-supersede-deep-copies-network` — Performs all mutations on `copy.deepcopy(net)` so the module-level `_network_cache` object stays pristine.
- `what-if-supersede-adds-new-node-as-premise` — `net.add_node` is called without `justifications`, so the successor becomes a premise regardless of `old_id`'s derivation.
- `what-if-supersede-preserves-access-tags-only` — Only `access_tags` from the old node's metadata is carried over; other metadata keys (`source_type`, `example`, `accepted_pr`) are dropped.
- `what-if-supersede-is-non-transitive` — Delegates to `net.supersede(old_id, new_id)` without `transitive=True`, so supersession-chain ancestors are not retracted in the simulation.
- `what-if-supersede-already-out-preserves-new-id` — The `already_out` short-circuit returns `new_id` as the caller passed it, which may be `None`; no auto-generation occurs on that path.
- `what-if-supersede-does-not-guard-new-id-collision` — Unlike `propose_supersession`, passing an existing `new_id` is not rejected by `what_if_supersede`; uniqueness is an unenforced caller obligation.
