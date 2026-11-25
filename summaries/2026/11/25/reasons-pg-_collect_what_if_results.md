---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: _collect_what_if_results in reasons/pg.py

## `_collect_what_if_results`

### Purpose

`_collect_what_if_results` is the reporting helper for the PostgreSQL what-if simulation methods. After `what_if_retract` or `what_if_assert` temporarily mutates a node’s truth value and runs `_propagate` inside an open transaction, this function converts the raw set of cascaded node IDs into two result lists:

- `retracted`: nodes that changed from `IN` to `OUT`
- `restored`: nodes that changed from `OUT` to `IN`

It enriches each result with the node’s current text, its cascade depth from the simulated root node, and its direct dependent count. It does not commit or roll back; the caller performs those lifecycle operations.

### Contract

Preconditions:

- `cur` is a live psycopg cursor in the same transaction as the simulation. Uncommitted truth-value changes must be visible to this cursor.
- `changed` is a re-iterable collection of node IDs, typically the `went_out + went_in` lists returned by `_propagate`.
- `before` is a mapping of node ID to prior truth value, usually captured before the simulation by selecting all nodes in the current domain.
- `source_id` is the node whose truth value was simulated. It is used as the starting point for cascade-depth calculation.
- `self.domain_id` is set and matches the domain used by the caller.

Postconditions:

- Returns a tuple `(retracted, restored)`.
- Each list contains dictionaries of the form:
  ```python
  {"id": nid, "text": text, "depth": depth, "dependents": dep_count}
  ```
- Nodes in `changed` that no longer exist in the current domain are silently skipped.
- Nodes whose `before` value equals their current value are skipped.
- Only exact transitions `IN -> OUT` and `OUT -> IN` are classified into `retracted` and `restored`.
- The function does not sort the result lists. Callers sort by `(depth, id)`.
- The function does not mutate the database, does not commit, does not rollback, and does not close the cursor.

Important invariant: the function trusts `changed` as a candidate set but rechecks current state against `before`. It does not assume every ID in `changed` actually changed, and it does not assume every ID is present in the domain.

### Parameters

- `cur`
  - A psycopg cursor.
  - Must be in the same transaction as the temporary simulation changes.
  - The function executes a `SELECT` against `rms_nodes` and calls helpers that issue more SQL.
  - Positional tuple-like rows are assumed because the code uses `r[0]`, `r[1]`, `r[2]`.

- `changed`
  - A collection of node IDs to inspect.
  - Typically a list produced by `went_out + went_in`.
  - Can be empty; then the function returns `([], [])` immediately.
  - Must be re-iterable. The code calls `list(changed)` for SQL, then later iterates over `changed` again. A one-shot generator would be consumed by `list(changed)` and produce no result entries.
  - Duplicates are not deduplicated. If `changed` contains the same ID twice, the same transition may appear twice in the output.

- `before`
  - A mapping from node ID to old truth value, expected values `"IN"` or `"OUT"`.
  - Usually a snapshot of all nodes in the domain before simulation.
  - If a node is missing from `before`, `old_tv` becomes `None`. That node will not be classified as retracted or restored because `None` matches neither transition, though the function may still compute depth and dependent count for it.

- `source_id`
  - The ID of the simulated root node.
  - Passed to `_cascade_depth_pg` to compute how many dependent hops separate each changed node from the root.
  - Not validated for existence. If it is invalid or the target is unreachable from it, `_cascade_depth_pg` returns `0`.

### Return Value

Returns:

```python
(retracted, restored)
```

Where each element is a list of dictionaries:

```python
{
    "id": nid,
    "text": text,
    "depth": depth,
    "dependents": dep_count,
}
```

- `retracted` contains entries where `old_tv == "IN"` and `new_tv == "OUT"`.
- `restored` contains entries where `old_tv == "OUT"` and `new_tv == "IN"`.
- `text` is the current text value from `rms_nodes` after the temporary simulation.
- `depth` is the shortest number of dependent edges from `source_id` to the node, as computed by `_cascade_depth_pg`.
- `dependents` is the number of direct dependent nodes, as returned by `_find_dependents(cur, [nid])`. This is a structural count, not a count of transitive dependents.

The function may return empty lists even when `changed` is non-empty, for example if all nodes are missing, unchanged, or missing from `before`.

### Algorithm

1. Initialize `pid = self.domain_id`, `retracted = []`, `restored = []`.
2. If `changed` is falsy, return the two empty lists immediately.
3. Query current state for all candidate IDs:
   ```sql
   SELECT id, text, truth_value
   FROM rms_nodes
   WHERE domain_id = %s AND id = ANY(%s)
   ```
   using `pid` and `list(changed)`.
4. Build `after_states`, mapping each found ID to `(text, truth_value)`.
5. Iterate over the original `changed` collection:
   - Skip IDs not present in `after_states`.
   - Extract current `text` and `new_tv`.
   - Look up `old_tv = before.get(nid)`.
   - If `old_tv == new_tv`, skip.
   - Compute `depth = self._cascade_depth_pg(cur, nid, source_id)`.
   - Compute `dep_count = len(self._find_dependents(cur, [nid]))`.
   - Build an `info` dictionary.
   - If `old_tv == "IN"` and `new_tv == "OUT"`, append `info` to `retracted`.
   - Else if `old_tv == "OUT"` and `new_tv == "IN"`, append `info` to `restored`.
6. Return `(retracted, restored)`.

Note that the depth and dependent-count work happens before classification. Therefore, if a node exists in `after_states` and has `old_tv != new_tv` but is missing from `before`, the function still performs the extra helper queries even though it cannot classify the transition.

### Side Effects

- Executes only read queries: one batched `SELECT` plus helper queries from `_cascade_depth_pg` and `_find_dependents`.
- Advances the cursor state.
- Does not insert, update, delete, commit, or rollback.
- Does not log anything.
- Does not close the cursor.
- Does not mutate `before` or `retracted`/`restored` externally.
- If `changed` is a one-shot iterator, `list(changed)` consumes it, which is an observable side effect for the caller.

Because `_cascade_depth_pg` and `_find_dependents` are called per changed node, this helper can issue many SQL queries. For large cascades or large graphs this can be expensive.

### Error Handling

There is no explicit error handling.

- psycopg/database errors from `cur.execute` or helper queries propagate to the caller.
- If `before` is `None` or not mapping-like, `before.get(nid)` can raise `AttributeError`.
- If `changed` is not iterable, `list(changed)` or the later loop can raise `TypeError`.
- If `cur` is invalid or closed, cursor errors propagate.
- A missing `source_id` does not raise an error; it typically causes depth calculation to return `0`.
- Missing nodes in `after_states` are silently skipped.
- Missing old values in `before` are silently not classified.

The what-if callers use `try/finally` to always roll back. If this helper raises, the caller’s `finally` still rolls back, and the exception propagates.

### Usage Patterns

This helper is called only by the what-if simulation methods:

- `what_if_retract`
- `what_if_assert`

Typical call sequence:

1. Verify the target node exists and is not already in the requested state.
2. Snapshot all node truth values into `before`.
3. Temporarily update the target node’s truth value.
4. Run `_propagate(cur, node_id)`.
5. Build `changed = went_out + went_in`.
6. Call:
   ```python
   retracted, restored = self._collect_what_if_results(
       cur, changed, before, node_id,
   )
   ```
7. Roll back the transaction.
8. Sort results by `(depth, id)` and return them.

Important behavioral detail: the callers pass only propagation results in `changed`, not the original simulated node. Therefore the simulated node itself is not reported in `retracted` or `restored` unless it somehow appears in `went_out` or `went_in`. In normal propagation it does not, because `_propagate` marks the source as visited and does not return it.

So `total_affected` in the public what-if response counts cascaded nodes reported by this helper, not the directly simulated node.

### Dependencies

Internal dependencies:

- `self.domain_id`
- `self._cascade_depth_pg`
- `self._find_dependents`

Database dependencies:

- `rms_nodes`
- indirectly `rms_justifications`, because `_find_dependents` uses JSONB containment over antecedents and outlist arrays.

External dependency:

- `psycopg` cursor behavior, including parameterized arrays for `ANY(%s)`.

### Assumptions Not Enforced by the Type System

- `changed` must be re-iterable, not a generator. The function uses `list(changed)` and later iterates `changed` again.
- `changed` should not contain duplicates unless duplicate result entries are acceptable.
- IDs in `changed` should belong to `self.domain_id`; others are skipped.
- `before` should cover every relevant candidate ID. Missing IDs are not classified.
- The cursor should use positional tuple-like rows. The code indexes rows by integer position.
- The cursor must be in the same transaction as the temporary simulation changes. If changes were committed or rolled back before this call, the comparison is wrong.
- `source_id` is trusted to be the correct simulation root. It is not checked for existence.
- Truth values are assumed to be only `"IN"` or `"OUT"`, consistent with the schema check constraint.
- `depth` and `dependents` are structural metrics. They are not filtered by current truth value or `_retracted` metadata.
- A depth of `0` means either the target is the source itself or the target was not reachable from the source through `_find_dependents`. The function does not distinguish those cases.

## Topics to Explore

- [function] `reasons/pg.py:_cascade_depth_pg` — Computes the depth value returned by `_collect_what_if_results`; verify BFS semantics, cycle handling, and the meaning of unreachable depth `0`.
- [function] `reasons/pg.py:_find_dependents` — Defines structural dependents via JSONB containment and drives both `dependents` counts and cascade depth.
- [function] `reasons/pg.py:_propagate` — Produces the `went_out`/`went_in` lists passed as `changed`; confirm why the simulated root is excluded from those lists.
- [function] `reasons/pg.py:what_if_retract` — Shows the transaction lifecycle, `before` snapshot, rollback behavior, and final sorting around this helper.
- [file] `tests/test_pg.py` — Useful for checking expected what-if output shape and edge cases such as leaf retractions, cascades, and already-toggled nodes.

## Beliefs

- `collect-what-if-requires-reiterable-changed` — `_collect_what_if_results` calls `list(changed)` for SQL and then iterates over `changed` again, so a one-shot generator would yield no result entries.
- `collect-what-if-returns-empty-when-changed-falsy` — If `changed` is empty or falsy, the function returns `([], [])` without executing a query.
- `collect-what-if-ignores-missing-after-state` — Node IDs in `changed` that are absent from the domain-filtered `after_states` query are skipped.
- `collect-what-if-classifies-only-in-out-transitions` — It appends to `retracted` only for `IN -> OUT` and to `restored` only for `OUT -> IN`; other old/new combinations produce no entry.
- `collect-what-if-depth-is-structural` — Depth is computed from `source_id` over `_find_dependents`, and dependent counts are direct structural dependents, not transitive or truth-filtered dependents.
