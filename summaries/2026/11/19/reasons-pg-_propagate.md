---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _propagate in reasons/pg.py

# `_propagate` in `reasons/pg.py`

## Purpose

`_propagate` performs incremental truth-value propagation through the dependency network after a node’s truth value (or retraction state) has already been changed.

It walks outward from `changed_id` to any nodes that reference it in a justification’s `antecedents` or `outlist`, re-evaluates those dependents using their justifications, updates their `truth_value` in PostgreSQL if needed, logs the change, and continues to their dependents. It is a BFS-style cascade used by mutation operations such as `assert_node`, `retract_node`, `add_justification`, `remove_justification`, and `challenge`.

It is **not** a full-network fixpoint recomputation. The separate `propagate()` method iterates over all derived nodes until stable. `_propagate` is an incremental optimization that assumes a single BFS pass is sufficient for the affected subgraph.

---

## Contract

### Preconditions

- `cur` is an open `psycopg` cursor in an active transaction.
- `changed_id` already exists in `rms_nodes` for `self.domain_id`, and its **new** truth value / metadata has already been written to the database before calling `_propagate`.
- All referenced node IDs in `ross_justifications.antecedents` and `outlist` are JSON arrays of strings.
- `metadata` is a JSON object. The code calls `meta.get(...)`, so it assumes `metadata` is not `None`.
- All nodes and justifications belong to `self.domain_id`.
- Truth values are exactly `"IN"` or `"OUT"`.
- The caller is responsible for committing or rolling back the transaction.

### Postconditions

- Reachable dependents may have `rms_nodes.truth_value` updated.
- Each such change inserts a row into `rms_propagation_log` via `self._log(cur, "propagate", dep_id, new_tv)`.
- No commit or rollback is performed.
- Returns two lists: node IDs that went `OUT` and node IDs that went `IN` during this cascade, **excluding `changed_id` itself**.
- Nodes marked with truthy `metadata["_retracted"]` are skipped and never updated.
- Premises (nodes with no justifications) are skipped and keep their current truth value.
- `updated_at`, `retracted_at`, metadata, and justifications are not modified by this method.

### Important assumptions / limitations

- The method assumes the initial node’s new truth value is already persisted. It does not recompute `changed_id`.
- `visited` is updated only when a node’s truth value changes. Once a node changes, it is never re-evaluated in this call. This assumes no node needs to change more than once during a single propagation call.
- Evaluation order within a batch is nondeterministic because `dep_ids` is a `set`. The code assumes correctness does not depend on the order in which dependents in the same BFS wave are evaluated.
- Missing antecedent nodes are treated as `"OUT"` and therefore fail the antecedent check.
- Missing outlist nodes are also treated as `"OUT"` and therefore satisfy the outlist check. This asymmetry is intentional/defensive but not enforced by the schema.
- Justification `type` (`SL` vs `CP`) is fetched but ignored. Both types are evaluated using the same antecedent/outlist rules.

---

## Parameters

| Parameter | Meaning |
|---|---|
| `self` | `PgApi` instance; provides `self.domain_id`, `_find_dependents`, and `_log`. |
| `cur` | Active `psycopg` cursor. Not type-checked. Must be in the same transaction/domain. |
| `changed_id` | String ID of the node whose change triggered propagation. Its new state must already be in the database. If it does not exist or has no dependents, the method may simply return empty lists. |

Edge cases:
- `changed_id` has no dependents → returns `([], [])`.
- `changed_id` is a premise → dependents are still processed; the premise itself is not re-evaluated.
- `changed_id` is retracted → its dependents are processed; the retracted node itself is skipped because it is in `visited` and would also be skipped by the `_retracted` check if encountered as a dependent.
- Cyclic dependencies → `visited` prevents infinite loops, but may also prevent legitimate re-evaluation.

---

## Return Value

Returns:

```python
(went_out, went_in)
```

- `went_out`: list of dependent node IDs that transitioned from `"IN"` to `"OUT"` during this call.
- `went_in`: list of dependent node IDs that transitioned from `"OUT"` to `"IN"` during this call.
- Does **not** include `changed_id`.
- Each changed node appears at most once because once its truth changes it is added to `visited` and will not be processed again in the same call.
- Order within a BFS wave is not deterministic because `dep_ids` is a set.
- If nothing changes, returns `([], [])`.

Callers typically combine the returned lists with the initial `changed_id` for user-facing results.

---

## Algorithm

1. Initialize:
   - `went_out = []`
   - `went_in = []`
   - `queue = deque([changed_id])`
   - `visited = {changed_id}`
   - `pid = self.domain_id`

2. While `queue` is non-empty:
   - Drain the entire queue into `batch`. This processes one BFS wave at a time.
   - Compute:
     ```python
     dep_ids = self._find_dependents(cur, batch) - visited
     ```
     `_find_dependents` issues SQL over `rms_justifications` using JSONB containment:
     ```sql
     antecedents @> '["node_id"]' OR outlist @> '["node_id"]'
     ```
     It returns all nodes that reference any node in `batch`.
   - If `dep_ids` is empty, continue.
   - Fetch `id, truth_value, metadata` for all `dep_ids` from `rms_nodes`.
   - Fetch all justifications for `dep_ids` from `rms_justifications`.
     - Parse `antecedents` and `outlist` from JSON if they are returned as strings.
     - Build `justs_by_node` and `all_referenced` (union of all antecedents and outlist IDs).
   - Batch-fetch truth values for all `all_referenced` nodes into `truth_cache`.
   - For each `dep_id` in `dep_ids`:
     - Skip if no state was fetched.
     - Skip if `metadata.get("_retracted")` is truthy.
     - Skip if the node has no justifications (it is a premise).
     - Compute `new_tv`:
       - Start with `"OUT"`.
       - For each justification `(jtype, ants, outs)`:
         - `inlist_ok = all(truth_cache.get(a, "OUT") == "IN" for a in ants)`
         - `outlist_ok = all(truth_cache.get(o, "OUT") == "OUT" for o in outs)`
         - If both are true, set `new_tv = "IN"` and break.
     - If `old_tv != new_tv`:
       - Execute:
         ```sql
         UPDATE rms_nodes SET truth_value = %s
         WHERE id = %s AND domain_id = %s
         ```
       - Log with `self._log(cur, "propagate", dep_id, new_tv)`.
       - Update `truth_cache[dep_id] = new_tv` so later dependents in the same batch see the new value.
       - Add `dep_id` to `visited`.
       - Append `dep_id` to `queue` for the next BFS wave.
       - Append to `went_out` if `new_tv == "OUT"`, else to `went_in`.

3. Return `went_out, went_in`.

---

## Side Effects

- Executes many SQL `SELECT` queries:
  - One per node in each batch inside `_find_dependents`.
  - Batch fetches for dependent states, justifications, and referenced truth values.
- Executes SQL `UPDATE` on `rms_nodes` for each changed dependent.
- Inserts rows into `rms_propagation_log` via `_log`.
- Mutates in-memory structures: `queue`, `visited`, `truth_cache`, `went_out`, `went_in`.
- Does **not** commit or rollback.
- Does **not** update `updated_at`, `retracted_at`, metadata, or justification rows.

---

## Error Handling

- No explicit exception handling. Any `psycopg` error propagates to the caller; the caller’s transaction context is expected to roll back.
- `json.loads` may raise `json.JSONDecodeError` if `metadata`, `antecedents`, or `outlist` contain malformed JSON strings.
- `meta.get(...)` will raise `AttributeError` if `metadata` is `None`. The schema defaults metadata to `'{}'`, but the column is not `NOT NULL`, so this is an assumption.
- Missing referenced nodes are silently treated as `"OUT"` by `truth_cache.get(..., "OUT")`.
  - Missing antecedent → antecedent check fails.
  - Missing outlist node → outlist check passes.
- Invalid or missing `changed_id` does not raise a dedicated error; it may simply result in no dependents being found.
- Cyclic graphs terminate because `visited` prevents revisiting changed nodes, but the method does not attempt to resolve cycles to a fixpoint.

---

## Usage Patterns

`_propagate` is called internally after a mutation has already updated a node’s truth value:

- `retract_node`: sets node to `OUT`, marks `_retracted`, then calls `_propagate`.
- `assert_node`: sets node to `IN`, clears `_retracted`, then calls `_propagate`.
- `add_justification` / `remove_justification`: recompute node truth, update, then propagate.
- `what_if_retract` / `what_if_assert`: call propagation inside a transaction that is later rolled back.
- `add_nogood`: retracts a victim, then propagates.
- `challenge`, `convert_to_premise`, `supersede`, and other mutation paths.

Caller obligations:
- Ensure `changed_id`’s new state is already written.
- Commit or roll back the transaction.
- Remember that `changed_id` is not included in the return lists.
- For a guaranteed fixpoint over the whole network, use `propagate()` instead.

---

## Dependencies

- Internal:
  - `self._find_dependents(cur, node_ids)` — discovers dependents via JSONB containment.
  - `self._log(cur, action, target, value)` — writes propagation log rows.
  - `self.domain_id` — scopes all SQL.
- External:
  - `collections.deque`
  - `json`
  - `psycopg` cursor supplied by the caller.
- Database tables:
  - `rms_nodes`
  - `rms_justifications`
  - `rms_propagation_log`

---

## Topics to Explore

- [function] `reasons/pg.py:_find_dependents` — Shows exactly how dependents are discovered using JSONB containment and why multiple queries are issued per batch.
- [function] `reasons/pg.py:propagate` — Full fixpoint recomputation over all derived nodes; useful for contrasting with the incremental, visited-limited `_propagate`.
- [function] `reasons/pg.py:_compute_truth` — Single-node truth calculation with the same antecedent/outlist semantics; useful for verifying consistency with `_propagate`.
- [file] `tests/test_pg.py` — Likely exercises propagation cascades, retractions, and edge cases in the PostgreSQL backend.
- [general] `truth-maintenance-rms` — The broader RMS semantics of justifications, antecedents, outlists, nogoods, and backtracking that this propagation implements.

---

## Beliefs

- `_propagate-excludes-initial-node` — `_propagate` returns only dependents whose truth values changed, never includes the initial `changed_id` in `went_out` or `went_in`.
- `_propagate-skips-retracted-nodes` — Nodes with truthy `metadata["_retracted"]` are skipped and never updated by `_propagate`.
- `_propagate-treats-missing-outlist-as-out` — Missing outlist node IDs are treated as `"OUT"` and therefore satisfy the outlist condition.
- `_propagate-ignores-justification-type` — The justification `type` (`SL` vs `CP`) is fetched but not used; both are evaluated by the same antecedent/outlist rules.
- `_propagate-requires-preupdated-changed-id` — `_propagate` assumes the initial node’s new truth value is already persisted in the database before it is called.
