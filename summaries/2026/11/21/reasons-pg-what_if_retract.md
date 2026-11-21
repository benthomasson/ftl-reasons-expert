---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: what_if_retract in reasons/pg.py

# `what_if_retract` — PostgreSQL simulation of retracting a node

## Purpose

`what_if_retract` answers the question *"if I retracted `node_id`, which other beliefs would flip to `OUT`, and which (if any) would come back `IN`?"* — **without persisting anything**. It is the "dry run" analogue of `retract_node` plus `_propagate`, used for planning / UI preview before making an irreversible change.

The trick is that the simulation runs entirely inside an uncommitted transaction and is unconditionally rolled back, so the DB ends in exactly the state it started in.

## Contract

**Preconditions (enforced implicitly, not by types):**
- `self.conn` is a live psycopg connection whose `autocommit` is `False`. This is set in `__init__` when a conninfo string is passed, but *not verified* when an existing connection object is passed in. See "Assumptions" below.
- The connection has **no uncommitted work the caller cares about**. The trailing `rollback()` discards it.
- `self.domain_id` must be a valid UUID-parseable string (the DB column is `UUID`); otherwise psycopg raises on the first query.
- `node_id` exists in the current domain.

**Postconditions:**
- No rows are committed. All INSERTs/UPDATEs done during the simulation (including the `_propagate` logs to `rms_propagation_log`) are rolled back.
- The returned dict describes the state transition that *would* happen.

**Invariants:**
- The function never calls `commit()`. This is the defining property — the transaction is rolled back on *every* exit path, including the `already_out` early return and the `KeyError` path.
- `retracted` and `restored` are disjoint and sorted by `(depth, id)`.
- The node being retracted is **not** represented in `retracted`; only cascade-affected nodes are.

## Parameters

- **`node_id: str`** — the primary key of the node (`rms_nodes.id`) to simulate retracting. Combined with `self.domain_id` in the WHERE clauses. If missing from the domain, raises `KeyError` (not returned as error dict).

No other parameters. `self.domain_id` is used as the tenant/domain scope for every query.

## Return Value

Returns a `dict`:

```python
{
  "node_id": str,
  "already_out": bool,          # True iff the node was already OUT (no simulation ran)
  "retracted": [ {id, text, depth, dependents}, ... ],  # IN -> OUT in simulation
  "restored":  [ {id, text, depth, dependents}, ... ],  # OUT -> IN in simulation
  "total_affected": int,        # len(retracted) + len(restored), source NOT counted
}
```

Two branches:
1. **Node is `OUT` already** → returns `already_out=True`, empty lists, `total_affected=0`. No UPDATE, no propagation, but still hits the `finally` rollback (a harmless no-op-ish reset).
2. **Node is `IN`** → runs the simulation, returns `already_out=False` and populated lists.

Note that `total_affected` excludes the retracted node itself. Retracting a leaf node therefore returns `total_affected == 0` even though the node itself would flip to `OUT` if committed. Callers who want the source's transition must look at `node_id` themselves.

`KeyError` propagates (via the `raise`), so callers using this for a UI should `try/except KeyError`. No "error dict" pattern is used here.

## Algorithm

Step-by-step:

1. **Scope**: capture `pid = self.domain_id`.

2. **Enter `try/finally`**. `finally` unconditionally calls `self.conn.rollback()`. This is the whole simulation mechanism — every statement inside the `try` becomes disposable.

3. **Verify existence & early-out**:
   ```sql
   SELECT truth_value FROM rms_nodes WHERE id=%s AND domain_id=%s
   ```
   - No row → `raise KeyError`.
   - `row[0] == 'OUT'` → return `already_out=True` with empty lists (still hits rollback).

4. **Snapshot pre-state**:
   ```sql
   SELECT id, truth_value FROM rms_nodes WHERE domain_id=%s
   ```
   Builds `before: {id: truth_value}` across the entire domain. This snapshot is used later to diff actual transitions versus the queue's touched set.

5. **Apply the retraction**:
   ```sql
   UPDATE rms_nodes SET truth_value='OUT' WHERE id=%s AND domain_id=%s
   ```
   (Note: unlike `retract_node`, this does not touch metadata, timestamps, or `retracted_at`. It's purely a truth-value flip for the simulation.)

6. **Propagate**: call `self._propagate(cur, node_id)`, which returns `(went_out, went_in)`. Both are *lists of dependents that changed truth value* during the BFS (the source node itself is seeded into `visited` and therefore never appears in either list).

7. **Diff against snapshot**: `changed = went_out + went_in`. Call `self._collect_what_if_results(cur, changed, before, node_id)`:
   - For each `nid` in `changed`, re-fetch `(text, truth_value)` in the mutated transaction.
   - Compare to `before[nid]`; skip if unchanged (defensive — `_propagate` should already guarantee a change).
   - Compute `depth = self._cascade_depth_pg(cur, nid, node_id)` — a BFS from `node_id` through `_find_dependents` until it reaches `nid`, returning the hop count.
   - Compute `dependents = len(self._find_dependents(cur, [nid]))`.
   - Classify: `IN→OUT` ⇒ `retracted`; `OUT→IN` ⇒ `restored`.

8. **Exit `with cursor`** (closes cursor), then **`finally`**: `self.conn.rollback()` — all of the above is discarded.

9. **Sort** each list by `(depth, id)` so callers see cascade-affected nodes ordered by graph distance, then lexicographically within a depth.

10. **Return** the assembled dict.

## Side Effects

Within the (rolled-back) transaction during the call:

- `UPDATE rms_nodes SET truth_value='OUT'` on the target.
- Many `UPDATE rms_nodes SET truth_value=…` from `_propagate`.
- Many `INSERT INTO rms_propagation_log` via `_log("propagate", …)` inside `_propagate`.
- Read-only queries in `_collect_what_if_results`, `_cascade_depth_pg`, `_find_dependents`.

**Outside** the transaction: nothing persists. Notably:

- `self.conn.rollback()` closes the current transaction entirely, so any *pre-existing* uncommitted work on that connection (from a caller that pipelined statements before calling this) is silently discarded.
- The cursor is closed by the `with` context; the class instance and connection remain open.
- Takes row locks on `rms_nodes` rows it touches, for the duration of the simulation. Under concurrent writes this can block other writers, but since the transaction is short-lived (one function call) it's usually fine.

## Error Handling

- **`KeyError(f"Node '{node_id}' not found")`** — raised when the initial SELECT returns no row. Not swallowed. `finally` still rolls back.
- **`psycopg` errors** (connection errors, invalid `domain_id` UUID, constraint violations from `_propagate`, etc.) — propagate, but `finally` rollback runs first so no partial state persists.
- **All other exceptions from `_propagate` / `_collect_what_if_results`** — propagate identically.
- **No error dict**. Callers expecting an `{"error": ...}` pattern must instead catch exceptions. This diverges from some higher-level API conventions.

## Usage Patterns

Typical callers:

- A CLI command like `rms what-if retract <id>`.
- An MCP tool or higher-level API method that presents a preview before the user commits to a real `retract_node`.

Constants/caller obligations:

- Must call on the same `PgApi` instance whose `domain_id` scopes the target node.
- Caller must not rely on any pre-existing uncommitted writes surviving the call.
- Caller must catch `KeyError`.
- Caller should not interpret the returned lists as complete "state after retraction" — they only include nodes whose truth value changed (excluding the source).

Symmetric method `what_if_assert` performs the same simulation for `OUT→IN`, and has the identical shape and `finally`-rollback discipline.

## Dependencies

Internal:
- `PgApi._propagate` — BFS truth-value propagation through justifications.
- `PgApi._collect_what_if_results` — builds `[{id, text, depth, dependents}]`.
- `PgApi._cascade_depth_pg` — hop-distance BFS from source.
- `PgApi._find_dependents` — JSONB containment lookup against `rms_justifications.antecedents` / `.outlist`.

External:
- `psycopg` (v3) — connection/cursor and error types.
- Tables: `rms_nodes`, `rms_justifications`, `rms_propagation_log` (log writes rolled back).

## Assumptions not enforced by the type system

1. **`self.conn.autocommit is False`**. If a caller passes in an existing psycopg connection created with `autocommit=True`, `self.conn.rollback()` becomes effectively a no-op and the simulation's UPDATEs **persist**. The class only sets `autocommit=False` when it creates the connection from a conninfo string; it does not validate an externally-supplied one.
2. **`self.conn` is not in the middle of an implicit transaction with pending writes** at call time — the final rollback discards them.
3. **`node_id` string equals a valid row id** — no normalization/namespace-prefixing is attempted here (unlike `add_node`, `add_justification`, `retract_node`…). A caller working with namespaces must pre-resolve the fully-qualified id.
4. **Single-writer assumption for the duration of the call**: nothing prevents a concurrent transaction from changing related nodes between the snapshot SELECT and the UPDATE; READ COMMITTED would let this happen, potentially yielding a slightly stale `before` map versus the observed `after` states. In practice the deterministic diff logic tolerates this, but reported lists may not reflect a "clean" cascade.
5. **`changed` is not deduplicated** because `_propagate`'s `visited` set guarantees no node appears twice in `went_out + went_in`. If `_propagate` semantics ever change, this call site assumes uniqueness (it would otherwise double-count a node in the result lists).

## Topics to Explore

- [function] `reasons/pg.py:_propagate` — This is the BFS engine that produces `went_out`/`went_in`; understanding it is required to interpret `depth` and the cascade semantics.
- [function] `reasons/pg.py:_collect_what_if_results` — Where the diff against `before` and the depth/dependents metadata are assembled.
- [function] `reasons/pg.py:what_if_assert` — The symmetric dry-run; useful for checking whether the same rollback invariants are preserved on the assert side.
- [file] `tests/test_pg.py` — Shows the actual expected return shapes and edge cases (already-OUT, cascades, restorations) in tests.
- [general] `psycopg-autocommit-rollback` — Verify why passing an `autocommit=True` connection would silently persist the simulation, and how `PgApi.__init__` should be adjusted to defend against it.

## Beliefs

- `what-if-retract-never-commits` — The function always calls `self.conn.rollback()` in a `finally`, so no state from a `what_if_retract` call persists, even on success.
- `what-if-retract-excludes-source-node` — The retracted node itself is not in the `retracted` list; `changed` comes from `_propagate`, which seeds `node_id` into `visited` and only reports dependents.
- `what-if-retract-requires-autocommit-false` — If a caller supplies a psycopg connection with `autocommit=True`, the final `rollback()` is ineffective and the simulation's `UPDATE`s persist.
- `what-if-retract-raises-keyerror` — Unknown `node_id` raises `KeyError` rather than returning an error dict, unlike some other API methods.
- `what-if-retract-rollback-discards-pending-work` — The `finally` rollback also discards any caller work that was uncommitted on the same connection prior to this call.
- `what-if-retract-sorts-by-depth-then-id` — `retracted` and `restored` are sorted with `key=(depth, id)` before returning.
