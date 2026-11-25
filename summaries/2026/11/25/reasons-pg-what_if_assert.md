---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: what_if_assert in reasons/pg.py

# `what_if_assert` — What-If Assertion Simulation

## Purpose

`what_if_assert` answers the question *"if I asserted (set to `IN`) this node's truth value, which other nodes in the belief network would be retracted or restored as a consequence — and how deep in the dependency graph would each of those changes be?"*

It exists so callers can preview the ripple effects of an assertion **without persisting** the mutation. The technique is transaction-level simulation: the update and full propagation are actually executed against the database inside an uncommitted transaction, the resulting diff is captured, and then everything is rolled back.

## Contract

**Preconditions**
- `self.domain_id` is set and corresponds to a `rms_nodes` partition.
- `self.conn` is a live psycopg connection created with `autocommit=False` (the constructor sets this — see `__init__`).
- The caller is not relying on this connection's current transaction state surviving the call.
- `node_id` identifies a node in `self.domain_id` (a `KeyError` if not).

**Postconditions**
- The database is byte-for-byte identical to before the call (assuming no concurrent writers racing on this transaction).
- Returns a structured diff of the simulated cascade.
- The connection's transaction is **always** rolled back — including on the `already_in` early return — so any *other* pending uncommitted work on the same connection is discarded.

**Invariants**
- `total_affected == len(retracted) + len(restored)`.
- Every entry in `retracted` had `before[id] == "IN"` and `after[id] == "OUT"` at collection time.
- Every entry in `restored` had `before[id] == "OUT"` and `after[id] == "IN"`.
- Both lists are sorted ascending by `(depth, id)`.

## Parameters

- `node_id: str` — The node ID to be asserted. Must exist in the current domain. The ID is compared against `rms_nodes.id` (TEXT) with the domain id as a composite filter (`PRIMARY KEY (id, domain_id)`).

Edge cases:
- `node_id` already `IN` → short-circuits with `already_in: True`.
- `node_id` not found → `KeyError` (not an error dict).
- Empty-string `node_id` or an ID from a different domain → treated as not-found (`KeyError`).

## Return Value

Two possible dict shapes:

**Early return (already `IN`):**
```python
{
    "node_id": node_id,
    "already_in": True,
    "retracted": [],
    "restored": [],
    "total_affected": 0,
}
```

**Normal return:**
```python
{
    "node_id": node_id,
    "already_in": False,
    "retracted": [ {...}, ... ],   # IN -> OUT during simulation
    "restored":  [ {...}, ... ],   # OUT -> IN during simulation
    "total_affected": int,          # len(retracted) + len(restored)
}
```

Each element inside `retracted` / `restored` comes from `_collect_what_if_results`:
```python
{
    "id": str,
    "text": str,
    "depth": int,          # shortest from source_id via dependents; 0 if unreachable
    "dependents": int,     # size of _find_dependents(nid)
}
```

Callers must handle: `KeyError` for missing nodes, and the fact that the *actual network was not mutated* — any caller that assumed mutation was performed must apply it separately with `assert_node`.

## Algorithm (step-by-step)

1. `pid = self.domain_id` — cache the domain for the query clauses.
2. Open a cursor inside a `try`.
3. `SELECT truth_value FROM rms_nodes WHERE id=… AND domain_id=…`.
   - No row → `raise KeyError`.
   - Row is `"IN"` → return the early dict. `finally` still fires, executing `conn.rollback()` (harmless: nothing changed on this path, but it **will** discard any other pending work on the connection).
4. Snapshot entire domain: `SELECT id, truth_value FROM rms_nodes WHERE domain_id=…` → `{node_id: truth_value}` called `before`. O(N) in domain size.
5. `UPDATE rms_nodes SET truth_value='IN'` for the target.
6. `went_out, went_in = self._propagate(cur, node_id)` — BFS through dependents, re-evaluating justification validity and flipping `truth_value` as needed. Returns two lists of node IDs that changed OUT and IN respectively. `_propagate` also writes log entries via `_log`.
7. `changed = went_out + went_in`.
8. `retracted, restored = self._collect_what_if_results(cur, changed, before, node_id)`:
   - Reads post-propagation state for `changed` IDs.
   - Filters out nodes where `before[id] == after[id]` (no net change).
   - For each remaining node, computes BFS depth from `source_id` via `_cascade_depth_pg` and dependent-count via `_find_dependents`.
   - Splits into `retracted` (`IN→OUT`) and `restored` (`OUT→IN`).
9. `finally: self.conn.rollback()` — **always** undo the update, all propagation `UPDATE`s, and all log inserts.
10. Sort each list by `(depth, id)` for deterministic output.
11. Return the result dict.

Note: the source node itself is *not* included in `retracted`/`restored` because it's not in `went_out`/`went_in` (the caller already knows it was the trigger).

## Side Effects

- **Database (transient):** an `UPDATE` on the target, N `UPDATE`s from `_propagate`, and corresponding `INSERT`s into `rms_propagation_log` — all discarded by `rollback()`.
- **Locks:** row locks acquired by those `UPDATE`s are held until the rollback executes. Under concurrent writers this can cause blocking for the duration of the simulation. Long dependency chains widen the window.
- **Connection state:** any unrelated uncommitted work on `self.conn` is rolled back by the `finally`, regardless of whether the early-return path was taken.
- **No I/O or stat changes** that survive the call.

## Error Handling

- `KeyError(f"Node '{node_id}' not found")` when the SELECT returns no row.
- Any exception inside the `with` block (e.g., psycopg `OperationalError`, `ProgrammingError`) propagates after `finally` runs the rollback — the transaction is not leaked.
- The early-return path *also* runs `finally`, so `conn.rollback()` executes even when the function is a no-op. It is silently tolerated if there's nothing to roll back.
- No `except` catches — all errors bubble up. Callers must catch `KeyError` and any DB-level exceptions themselves.

## Usage Patterns

Typical usage:
```python
with PgApi(conninfo, domain_id) as api:
    preview = api.what_if_assert("belief:42")
    if preview["total_affected"] > 0:
        for node in preview["retracted"]:
            print(node["id"], "would go OUT at depth", node["depth"])
        # user confirms
        api.assert_node("belief:42")
```

Caller obligations:
- Do **not** rely on the connection holding an open transaction across this call — the method owns the transaction boundary and rolls back unconditionally.
- Do not open it inside a larger transaction that shouldn't be destroyed; use a dedicated `PgApi`/connection for simulations if you need other pending work isolated.
- Serialize with other writers if you must observe a consistent snapshot of the domain (the `before` snapshot is read at a single point, but concurrent commits between step 4 and step 8 could produce surprising diffs; PostgreSQL `READ COMMITTED` is the default here).

It is the mirror image of `what_if_retract` — same transactional pattern, opposite target value and opposite truth-value direction.

## Dependencies

- **psycopg** (v3) — cursor, parameterized queries, rollback.
- **`self._propagate`** — BFS propagation engine (same file).
- **`self._collect_what_if_results`** — assembles the diff (same file).
- **`self._cascade_depth_pg`** — BFS from source to target through `_find_dependents`.
- **`self._find_dependents`** — JSONB containment queries against `rms_justifications`.
- **`self._log`** — writes to `rms_propagation_log` inside the same (eventually-rolled-back) transaction.
- **Schema tables** — `rms_nodes`, `rms_justifications`, `rms_propagation_log`.

## Topics to Explore

- [function] `reasons/pg.py:_propagate` — BFS propagation that determines exactly which nodes flip during the simulation, with its own `_retracted` and truth-table logic
- [function] `reasons/pg.py:_collect_what_if_results` — Formatting/aggregation used by both `what_if_retract` and `what_if_assert`, including the depth and dependent-count computation
- [file] `tests/test_pg.py` — Verifies what-if behavior against a real PostgreSQL instance; the best way to see contract expectations in action
- [function] `reasons/pg.py:what_if_retract` — The symmetric twin; side-by-side it reveals the extraction pattern (target value, "already" flag, direction)
- [general] `transaction-as-simulation` — Pattern of using uncommitted DB transactions + rollback for what-if analysis and its concurrency implications (locking, snapshot isolation, interaction with caller transactions)

## Beliefs

- `what-if-assert-uses-rollback` — `what_if_assert` executes a real `UPDATE` and full propagation inside an uncommitted transaction, then unconditionally calls `self.conn.rollback()` so no state is persisted.
- `what-if-assert-requires-autocommit-false` — The simulation only works because `PgApi.__init__` creates the connection with `autocommit=False`; with autocommit on, changes would persist and the rollback would be a no-op.
- `what-if-assert-early-return-still-rolls-back` — The `already_in` early return path still triggers the `finally` block's `rollback()`, discarding any unrelated uncommitted work on the connection.
- `what-if-assert-raises-keyerror-for-missing-node` — A missing `node_id` raises `KeyError`, not a result dict with an error field.
- `what-if-assert-produces-depth-sorted-diff` — Every entry in `retracted`/`restored` carries `depth` and `dependents`, and the lists are sorted by `(depth, id)`.
- `what-if-assert-excludes-source-node` — The triggering node itself is not included in `retracted` or `restored`; only downstream/upstream dependents that changed are listed.
