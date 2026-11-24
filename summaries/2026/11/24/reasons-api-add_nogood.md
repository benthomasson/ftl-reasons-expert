---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: add_nogood in reasons/api.py

## `add_nogood` — `reasons/api.py`

### Purpose

Records a *nogood* — a contradiction that says "these belief IDs cannot all be IN at the same time" (its semantics are "the truth value assertion set {n1…nk} is a contradiction that must be resolved"). The function delegates the actual work to the network layer — appending a `Nogood` to the log **and** performing dependency-directed backtracking to retract a premise so that the contradiction can no longer hold — while the wrapper just handles SQLite/PostgreSQL plumbing, reporting, and cache/save lifecycle. It exists so callers (CLI, `accept_proposal`, `apply_contradiction_plan`, `detect_contradictions(auto_apply=True)`) can record a contradiction through a single functional façade without managing the `Storage` lifecycle themselves.

### Contract

**Preconditions**
- `node_ids` is a list of node IDs; the intent is that at the caller's current belief state the passed IDs form the contradictory set (typically all IN, in which case culprit detection fires).
- The standard pattern is to invoke this through the `api` module (SQLite `_with_network` or PostgreSQL `_pg_dispatch`) — not by touching `Network` directly if the caller expects the change to be persisted.
- `pg_conninfo` and `db_path` are mutually exclusive: if `pg_conninfo` is truthy, `db_path` is ignored.

**Postconditions**
- A new `Nogood` with a generated `id` is appended to `net.nogoods` (the returned `ng` is literally `net.nogoods[-1]`).
- Whatever retraction is implied by `net.add_nogood` has propagated; the returned `changed` list is the set of node IDs whose truth values flipped.
- On clean exit the network is saved to storage and the in-memory cache is refreshed from the new mtime.
- If `pubsub.has_subscribers(db_path)` was True at `__enter__` and the write succeeds, `pubsub.compute_changes` + `publish` fire with the diff of truth values between `__enter__` and `__exit__`.

**Invariants assumed but not enforced by the type system**
- `net.add_nogood(node_ids)` **appends exactly one** `Nogood` to `net.nogoods` when it returns successfully; the function reads `net.nogoods[-1]` by index and would `IndexError` if that were not the case.
- `net.find_culprits(node_ids)` returns a list of dicts each carrying a `"premise"` key; the code does `culprits[0]["premise"]`.
- If an exception propagates from `net.add_nogood` (e.g. unknown node ID), `_with_network.__exit__` short-circuits the save (see `if write and exc_type is None`), so the database rolls back — that behavior is relied on here, but is not declared anywhere in this function.
- The PostgreSQL branch is assumed to return the same dict shape (`nogood_id`, `nodes`, `changed`, `backtracked_to`) — but this wrapper does no shape check.

### Parameters

| Param | Type | Meaning / Edge cases |
|---|---|---|
| `node_ids` | `list[str]` | The contradictory set. No length check — an empty list makes `all(...)` vacuously True, sending `[]` to `find_culprits` and `add_nogood`. Missing IDs are permitted by the function body (they only make `all_in` False), but whether `Network.add_nogood` accepts unknown IDs is a `Network`-level decision. Also note the *input list is passed by reference* to `add_nogood`; if the network mutates it in place, that mutation leaks to the caller. |
| `db_path` | `str` | SQLite file path; defaults to `reasons.db`. Ignored entirely when `pg_conninfo` is set. |
| `pg_conninfo` | arbitrary truthy | If set, routes to `PgApi.add_nogood` via `_pg_dispatch` and never touches SQLite. Backend choice. |
| `domain_id` | any | Tenant/domain hint passed through to the PostgreSQL path. Ignored locally. |

### Return Value

Always a `dict`:

```python
{
  "nogood_id":     str,            # ng.id of the appended Nogood
  "nodes":         list[str],      # ng.nodes — see note below
  "changed":       list[str],      # IDs whose truth_value changed
  "backtracked_to": str | None,    # premise ID chosen for retraction
}
```

- `nodes` comes from `ng.nodes`, i.e. **the stored nogood's node list, not the caller's input**. If `Network.add_nogood` normalizes/orders/dedupes (common for canonical nogood IDs, see `tests/test_nogood_id.py`), the caller-seen list may differ from what was passed.
- `backtracked_to` is `None` in **two** scenarios: (a) not all nodes were IN at call time, so `find_culprits` was skipped; (b) `find_culprits` returned an empty list. It is a *report-only* field — computed **before** retraction so it names the premise the backtracking picked, and it is not proof that the premise was actually retracted.
- `changed` may legitimately be empty if the nogood had no lifting consequences in the current state.
- Callers must not assume an exception on contradictory input — a successfully recorded nogood with `changed == []` is a valid outcome.

### Algorithm

1. **Backend dispatch.** If `pg_conninfo` is truthy, forward `{"node_ids": node_ids}` to `PgApi.add_nogood`; return its dict unchanged. No local reporting (`backtracked_to` etc.) is computed here — the PostgreSQL implementation is contractually expected to produce it.
2. **Open write context.** `with _with_network(db_path, write=True) as net:` — this always loads fresh from `Storage` (no cache hit when `write=True`), snapshots pre-mutation truth values if pubsub subscribers exist, and guarantees save + cache refresh on clean exit.
3. **Check "all in".** `all_in = all(nid in net.nodes and net.nodes[nid].truth_value == "IN" for nid in node_ids)`. This is both the culprit-detection gate and a way to avoid `KeyError` from `net.nodes[nid]` for unknown IDs.
4. **Culprit discovery.** If `all_in`, call `net.find_culprits(node_ids)` to get premises that could be retracted to break the contradiction; `backtracked_to` becomes `culprits[0]["premise"]` (the top-ranked one) or `None` if empty. This runs *before* mutation on purpose — after `net.add_nogood` the state is already consistent and culprits would be trivialized.
5. **Mutate.** `changed = net.add_nogood(node_ids)` — this is where the `Nogood` is appended and backtracking runs (retract premise → re-propagate/recompute cascade).
6. **Read back.** `ng = net.nogoods[-1]` — assumes the just-added nogood is now the tail of the list.
7. **Return** the reporting dict.
8. **Context exit** (only if no exception): `store.save(network)` (SQLite) → optional pubsub publish → cache set to new mtime.

### Side Effects

- Appends to `net.nogoods` (persisted to SQLite on exit).
- Mutates node truth values via backtracking/cascade.
- Writes to the SQLite database (schema/content under `Storage.save`).
- Updates the process-global `_network_cache[db_path]`.
- May publish pubsub events (if subscribers existed at entry).
- In the PostgreSQL branch: network round-trip + server-side mutation; no local DB or cache side effect.
- If `net.add_nogood` raises mid-mutation, `_with_network.__exit__` sees `exc_type is not None` and **skips** `store.save`, `pubsub.publish`, and the cache update — but the loaded in-memory `Network` object still carries dirty fields internally (harmless because it's discarded when the `with` block exits). No `Nogood` reaches the DB.

### Error Handling

- No explicit `try/except` inside this wrapper.
- `KeyError` / `NodeNotFound`-style errors surface from `net.add_nogood` if the network rejects unknown IDs, and from `culprits[0]["premise"]` if `find_culprits` returns culprit dicts without a `"premise"` key.
- `IndexError` would surface if `add_nogood` failed to append a `Nogood` but returned normally.
- PostgreSQL path: whatever `PgApi.add_nogood` raises propagates (its context manager handles rollback).
- Every raised exception prevents `Storage.save`; SQLite mutation is rolled back by omission, not by an explicit `rollback()`.
- No validation that `node_ids` is non-empty, no validation that IDs actually exist for the sake of a clean error, no validation that the nogood was *new* (calling twice with the same IDs may be idempotent inside `Network.add_nogood` or may duplicate — determined by `network.py`).

### Usage Patterns

Typical callers (from the same file):

- `accept_proposal` in the `"nogood"` action branch, after re-validating that all node IDs still exist.
- `apply_contradiction_plan`, when replaying a human-reviewed plan file; wraps each entry in `try/except` and reports per-entry errors.
- `detect_contradictions(auto_apply=True)`, where a contradiction is a tuple of claim IDs discovered by an LLM.
- CLI (`reasons nogood ...`).

Caller obligations:

- Inspect `backtracked_to` to know *why* a belief went OUT; `None` means "no culprit premise was picked" (the set was not fully IN, or was not reducible).
- Do not assume the input `node_ids` list is returned unchanged — read `nodes` from the result.
- Use the returned `nogood_id` for any follow-up query (e.g. `export_network` exposes nogoods by ID).
- Do not call this in a loop expecting per-call transactional isolation across many nogoods — each call opens its own `_with_network(write=True)` and re-loads from disk, so batching requires an outer transaction or a single call with multiple nogoods (not supported).

### Dependencies

Internal / sibling modules pulled in at call time:

- `_with_network` — the load/save/cache/pubsub context manager (defined in this file).
- `_pg_dispatch` — thin shim that opens `PgApi(pg_conninfo, domain_id)` and forwards to `add_nogood`.
- `Network.find_culprits`, `Network.add_nogood` — the real algorithm; `Network.nogoods` is the record list.
- `Storage` — via `_with_network.__init__` and `__exit__`.
- `pubsub.has_subscribers`, `pubsub.compute_changes`, `pubsub.publish` — via `_with_network`.
- `PgApi` — only when `pg_conninfo` is set.

### Assumptions not guarded by types

- That `node_ids` is not mutated in-place by `Network.add_nogood` in a way the caller cares about.
- That `net.add_nogood` returns the *same list type* as `changed` and appends exactly one `Nogood`.
- That `find_culprits` always returns a list (never `None`).
- That every culprit dict has a `"premise"` key.
- That `PgApi.add_nogood` reproduces the SQLite dict shape — no fallback, no shape check.
- That `culprits[0]` is the "best" candidate; no ranking criterion is asserted in the docstring.

## Topics to Explore

- [function] `reasons/network.py:Network.add_nogood` — the actual algorithm that appends the `Nogood`, computes the ID, and drives backtracking; see how "which premise to retract" is chosen and whether repeated nogoods are idempotent.
- [function] `reasons/network.py:Network.find_culprits` — how the "best" culprit is ranked and whether it always includes a `"premise"` key, since `add_nogood` indexes into that dict.
- [file] `tests/test_nogood_id.py` — verifies how `Nogood.id` is derived and whether two identical nogoods collide or dedupe; this determines whether repeated `add_nogood(ids)` calls are safe.
- [file] `tests/test_backtracking.py` — demonstrates the intended `changed` / `backtracked_to` behavior and the pubsub events emitted, and captures the expected rollback-on-exception semantics.
- [file] `tests/test_pg.py` — the parity contract between the SQLite and PostgreSQL branches (this function relies on it silently).

## Beliefs

- `add-nogood-requires-all-nodes-in-for-culprit-reporting` — `backtracked_to` is only ever populated when every input `node_id` already exists in the network and has truth value `"IN"`.
- `add-nogood-appends-exactly-one-nogood` — the function reads `net.nogoods[-1]` after `net.add_nogood`, so `Network.add_nogood` must append exactly one `Nogood` on every non-raising call.
- `add-nogood-returns-normalized-node-list` — the `nodes` field in the return value comes from `ng.nodes`, not from the caller-supplied `node_ids`, so the returned list may differ in order or content.
- `add-nogood-rolls-back-on-exception` — because `_with_network(write=True)` only calls `store.save()` when `exc_type is None`, any raise from `Network.add_nogood` discards the in-memory nogood and never persists it.
- `add-nogood-dispatches-to-postgres-when-pg-conninfo-set` — the presence of a truthy `pg_conninfo` bypasses SQLite and `_with_network` entirely, delegating to `PgApi.add_nogood` via `_pg_dispatch`.
