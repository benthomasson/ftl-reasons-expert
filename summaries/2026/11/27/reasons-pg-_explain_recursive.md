---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: _explain_recursive in reasons/pg.py

# `_explain_recursive` (in `reasons/pg.py`)

## Purpose

`_explain_recursive` is the workhorse behind `PgApi.explain_node`. It walks the justification graph for a single node inside the current `domain_id`, and returns a flat, **pre-order DFS list of "explanation steps"**. Each step tells the caller *why* a node is `IN` or `OUT` in the current belief state:

- **Premises** (`IN`) → “premise”
- **Retracted premises** (`OUT`) → “retracted premise”
- **Derived `IN`** → the single justification that is currently valid, plus the antecedents of that justification
- **Derived `OUT`** → every justification is invalid; for each one, list which antecedents failed and which outlist nodes are currently violated

It is a private method (`_` prefix), takes a live DB cursor, and is meant to run entirely inside the caller’s transaction. The stated store already bakes truth values into `rms_nodes.truth_value`; this function **reads** those values rather than recomputing them (except for `_justification_valid_cached`, which does a fresh evaluation for `IN` nodes).

## Contract

**Preconditions**

- A live `cur` (psycopg cursor) in the same transaction/domain as `self.conn`.
- `node_id` — string, either exists in `rms_nodes` or the caller tolerates `[]`.
- `visible_to` — either `None` (no access filtering) or an iterable of access tags the caller is allowed to see.
- `visited` — a mutable `set` the caller owns. The method mutates it; passing a shared set is what gives cycle protection and the "each node explained at most once" invariant across the whole traversal.
- `self.domain_id` is set (constructor) and matches the DB rows.
- `self._is_visible` and `self._justification_valid_cached` exist on `self`.

**Postconditions**

- Returns a `list[dict]` of step entries in DFS pre-order. The first step (if any) concerns `node_id`; subsequent entries are recursively expanded antecedents of that first step, in antecedent-list order.
- Every `node_id` in the returned list is unique per call because `visited` is checked at entry. On a duplicate entry, the function short-circuits to `[]`.
- `visited` now contains every node that was visited — **even nodes that were returned `[]` because they were missing or invisible** (the branch returns after `visited.add(node_id)`).
- No database writes. No commits. Only `SELECT`s. `self._justification_valid_cached` also only `SELECT`s.

**Invariants the code assumes, not enforced**

- The persisted `truth_value` is (mostly) consistent with the justifications and antecedents. If an `IN` node has *no* justifications that currently validate, the `for` loop completes without `break` and **no step is emitted for that node** — silent fall-through.
- `antecedents`/`outlist` stored in JSONB are either `list[str]` or a JSON string that decodes to `list[str]`.
- A node has at most one "the" valid justification for the `IN` case, in the sense that first-match wins (`ORDER BY id`).

## Parameters

| Name | Type | Meaning | Edge cases |
|------|------|---------|-----------|
| `cur` | psycopg cursor | Live cursor bound to `self.domain_id`'s DB | Caller must not commit while traversing. |
| `node_id` | `str` | Belief ID to explain | Missing ID → `[]`. Premise → 1-step result. |
| `visible_to` | `None \| Iterable[str]` | Access tags allowed for the caller | `None` disables filtering. Empty set allows only nodes with empty `access_tags`. |
| `visited` | `set[str]` (mutable) | Cross-branch de-dup + cycle guard | Shared for the whole traversal; passed back to nested call. |

## Return Value

A `list[dict]`. Shape varies by node kind:

- **Premise / retracted premise** (no justifications):
  ```python
  {"node": id, "truth_value": tv, "reason": "premise" | "retracted premise"}
  ```

- **Valid `IN` derived node** (first valid justification only):
  ```python
  {"node": id, "truth_value": "IN",
   "reason": "SL justification valid" | "CP justification valid",
   "antecedents": [...], "label": <str>,
   "outlist": [...]}     # only present when non-empty
  ```
  followed by the recursive step lists of each antecedent.

- **`OUT` derived node** (once per justification):
  ```python
  {"node": id, "truth_value": "OUT",
   "reason": "SL justification invalid" | "CP justification invalid",
   "failed_antecedents": [...],
   "label": <str>,
   "violated_outlist": [...]}     # only present when non-empty
  ```
  No recursion on the OUT branch.

- **Missing / invisible node**: `[]` (contributes nothing).

Callers (`explain_node`) wrap this as `{"steps": steps}`. There is **no** step type for “found no valid justification but `truth_value == 'IN'`”, so that state silently disappears from the trace.

## Algorithm

```
1.  if node_id in visited: return []
2.  visited.add(node_id)
3.  SELECT truth_value, metadata FROM rms_nodes
      WHERE id = %s AND domain_id = self.domain_id
4.  if no row: return []
5.  if metadata is a str: json.loads it
6.  if visible_to is not None and not self._is_visible(metadata, visible_to): return []
7.  SELECT type, antecedents, outlist, label
      FROM rms_justifications
      WHERE node_id = %s AND domain_id = %s
      ORDER BY id
8.  if no justifications:
        append {node, tv, "premise"/"retracted premise"} ; return
9.  if tv == "IN":
        for each justification (in id order):
            decode JSON if str
            if self._justification_valid_cached(cur, ants, outs):
                emit IN step (with optional outlist)
                for ant in ants:
                    steps.extend(self._explain_recursive(cur, ant, visible_to, visited))
                break            # <-- only first valid justification
10. else (tv == "OUT"):
        for each justification:
            decode JSON
            batch SELECT truth_value for set(ants) | set(outs)
            failed   = [a for a in ants if cache.get(a, "OUT") == "OUT"]
            violated = [o for o in outs if cache.get(o, "OUT") == "IN"]
            emit OUT step with failed/violated
        # no recursion into antecedents
11. return steps
```

Three non-obvious behaviors:

- **DFS pre-order, single pass.** Because `visited` is shared, a node reachable as an antecedent of two different IN branches will only be expanded once. The second reference still appears in the parent step’s `antecedents` list, but there is no corresponding sub-tree — the reader has to treat `visited`-suppressed nodes as "already shown above".
- **Only one valid justification is followed for `IN` nodes.** Any other valid justifications in the DB are silently omitted from the trace. `ORDER BY id` picks a deterministic winner.
- **`OUT` branch is a shallow fan-out.** All justifications are reported, but the graph is *not* walked at all — a `OUT` explanation shows failed antecedents by *name*, not by recursive breakdown.

## Side Effects

- Reads only (`SELECT`) — no `UPDATE`, `INSERT`, or `COMMIT`.
- **Mutates `visited`** (adds `node_id`). This is the shared side channel that makes the whole recursion correct; callers must pass a fresh `set()` per top-level `explain_node` call.
- Calls `self._is_visible` (pure) and `self._justification_valid_cached`, which issues another `SELECT` for every justification considered on the `IN` path.
- Combined effect: for an `IN` node this technically makes *two* truth fetches per candidate justification (one for validity, one inside `_justification_valid_cached` for the batch) — acceptable for explanation-sized traversals.

## Error Handling

- **No row for `node_id`** → returns `[]`. Not an exception.
- **Node not visible to `visible_to`** → returns `[]`. **Note**: the parent step that referenced this antecedent still lists it in `antecedents`, so the tree is *partially* filtered — the reference leaks as a bare ID, but its expansion is suppressed. This is arguably a bug or at least a leak of node IDs across access boundaries; callers with strict access requirements should post-filter.
- **Bad JSON in a JSONB column** → `json.loads` would raise `json.JSONDecodeError`. Unlikely in practice (JSONB columns are always valid JSON), but the string-coercion check assumes the driver offers either a `list` or a `str`, not `bytes`.
- **DB errors** (connection loss, permission) propagate as psycopg exceptions. Nothing is swallowed.
- **Silent fall-through** for `IN` node with no valid justification: emits no step for that node and does not recurse. No exception, no warning.

## Usage Patterns

- Called only from `PgApi.explain_node`:
  ```python
  def explain_node(self, node_id, visible_to=None):
      with self.conn.cursor() as cur:
          steps = self._explain_recursive(cur, node_id, visible_to, set())
      return {"steps": steps}
  ```
  Note: `explain_node` **does not commit** — it opens a cursor in a `with` block; psycopg closes the cursor at block exit but the implicit transaction of reads usually stays open unless the caller commits/rolls back. The read-only nature means rollback is safe.
- Also imported/invoked by tests (`tests/test_pg.py`, `tests/test_pg_dispatch.py`) via `explain_node`, not directly.
- Caller obligations:
  - Pass a **fresh `visited` set** per call.
  - Do not rely on the returned list being a strict proof tree — it is a partial DFS with duplicate suppression.
  - Post-filter on `visible_to` if the caller needs antecedents not to leak IDs from inaccessible nodes.
  - Treat missing steps for `IN` nodes with justifications as a signal that stored `truth_value` disagrees with recomputed truth — hook to `propagate()` / `_compute_truth` if consistency matters.

## Dependencies

**Internal (same file / class):**
- `self.domain_id`, `self.conn`
- `self._is_visible` — access_tags ⊆ visible_to check
- `self._justification_valid_cached` — atomically checks `all(ant IN)` and `all(out OUT)` by batch `SELECT`ing `rms_nodes.truth_value`. **Despite the name, there is no cache**; it’s a one-shot batch query.

**DB schema (defined in `SCHEMA`):**
- `rms_nodes(id, domain_id, truth_value, metadata)` — PK `(id, domain_id)`.
- `rms_justifications(id, node_id, domain_id, type, antecedents, outlist, label)` — ordered by `id` for determinism.

**External:**
- `json` (module-level).
- psycopg cursor interface (`execute`, `fetchone`, `fetchall`, `ANY(...)` array param).

**Companion routines the reader should recognize:**
- `_compute_truth` — the canonical recompute; `explain_node` prefers stored `truth_value` here, so they can diverge if the DB is stale.
- `_propagate` — the cascade that keeps `truth_value` in sync; explanation is a read-only view of the result of that cascade.
- `_trace_assumptions_recursive` — same traversal pattern, but returns premises only.

## Topics to Explore

- [function] `reasons/pg.py:_justification_valid_cached` — Semantics of “SL/CP valid” come from this; compare to `_compute_truth` to see how they can disagree.
- [function] `reasons/pg.py:explain_node` — The public entry point; note the missing `commit/rollback` handling and how `visible_to` is forwarded.
- [function] `reasons/pg.py:_propagate` — The upstream cascade that produced the stored `truth_value` this method reads.
- [file] `tests/test_pg.py` — Exercises the explanation tree against the async/dispatch path; useful for verifying the DFS shape.
- [general] `access-tag-inheritance` — `_is_visible` + `_inherit_access_tags` determine whether `[]` is returned for a node, which silently prunes the explanation tree.

## Beliefs

- `explain-recursive-is-scoped-to-self-domain-id` — Every `rms_nodes`/`rms_justifications` query filters by `self.domain_id`, preventing cross-tenant leakage at the SQL layer.
- `explain-recursive-uses-shared-visited-set-to-avoid-infinite-recursion` — Cycle protection and “explain each node once” are implemented via the caller-owned `visited` set, not a depth limit.
- `explain-recursive-recurses-only-for-IN-nodes` — Antecedents of `OUT` nodes are reported as `failed_antecedents` but not expanded.
- `explain-recursive-emits-one-step-per-node` — Because of `visited`, each node contributes at most one step to the whole result list per top-level call.
- `explain-recursive-returns-empty-list-for-invisible-nodes` — When `visible_to` is provided and the node’s access tags are not a subset, the function returns `[]` — the reference from a parent step is not scrubbed.
- `explain-recursive-has-no-writes` — It issues only `SELECT`s; the caller’s transaction is not mutated by calling it.
