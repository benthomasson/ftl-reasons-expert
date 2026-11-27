---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: find_culprits in reasons/pg.py

# `find_culprits` — Explanation

## 1. Purpose

`find_culprits` is the **public read-only entry point** for "dependency-directed backtracking" in the PostgreSQL-backed belief network. Given a *nogood* — a set of node IDs that are believed to be mutually incompatible — it determines which **premises** (undifferentiated, non-derived nodes) in the network, if retracted, would break the derivation chain leading to some of those nodes and therefore potentially resolve the contradiction.

The function is a thin wrapper: it opens a cursor, delegates to `_find_culprits_internal`, and packages the result in a `{"culprits": [...]}` envelope so it can be serialized directly by the CLI/API layer. The actual reasoning lives in `_find_culprits_internal`, `_trace_assumptions_recursive`, and `_entrenchment`.

## 2. Contract

**Preconditions**
- `self.conn` is a live psycopg v3 connection configured with `self.domain_id`.
- The `rms_nodes` and `rms_justifications` tables have been initialized for that domain (`init_db` was called).
- `node_ids` is an arbitrary iterable of node-ID strings (this matters — see below).

**Postconditions**
- The returned dict contains exactly one key, `"culprits"`, whose value is a list of candidate dicts.
- Candidates are only drawn from *premises reachable from IN nodes listed in `node_ids`*. Nodes that are OUT are skipped entirely.
- Candidates are sorted by `entrenchment` in **ascending** order — weakest (lowest-entrenchment) premises first, which is the preferred retraction order for a truth-maintenance system.

**Invariants**
- The function performs **no writes and no commits**. It leaves the caller's transaction in whatever state it was in (initial, active, or pending).
- Each candidate's `premise` is a node ID that has **no justifications** (i.e., is a premise) — this is enforced indirectly by `_trace_assumptions_recursive`, which only records nodes that have no rows in `rms_justifications`.

## 3. Parameters

- `node_ids` — an iterable of node-ID strings (typically a `list` or `tuple`). This is the *nogood*: a set of beliefs the caller suspects contradict each other.

  Edge cases:
  - **Empty iterable** → SQL returns no rows, `all_premises` stays empty, and the function returns `{"culprits": []}`. No error.
  - **Missing IDs** → the `SELECT id, truth_value ...` simply doesn't return them; they are treated as if OUT and skipped.
  - **OUT nodes in the input** → skipped by the `if node_tvs.get(nid) != "IN": continue` guard.
  - **Non-string IDs** → the type is not validated here; psycopg will bind whatever is passed. Callers are expected to pass strings (schema has `id TEXT`).

## 4. Return Value

```python
{"culprits": [
    {
        "premise": str,             # node ID of an ultimate assumption
        "would_resolve": [str],     # input node IDs whose assumption set includes this premise
        "dependent_count": int,     # len(self._find_dependents(cur, [premise]))
        "entrenchment": int,        # higher = more established (from _entrenchment)
    },
    ...
]}
```

- **Empty list** is returned when none of the input nodes are currently IN, or when no IN node in the input has any premise ancestors.
- The list is sorted **ascending by entrenchment**. Callers who want the "cheapest" retraction should either consume `culprits[0]` (as `add_nogood` does) or scan from the head.
- A caller must not assume `would_resolve` is a *guaranteed* set of nodes that will flip OUT — see §7.

## 5. Algorithm

The wrapper itself is trivial; the algorithm is entirely in `_find_culprits_internal`:

1. **Snapshot truth values.** `SELECT id, truth_value FROM rms_nodes WHERE domain_id = %s AND id = ANY(%s)`. Only nodes with `truth_value = 'IN'` participate.

2. **Trace premises for each IN node.** For every IN node, call `_trace_assumptions_recursive`. That routine does a DFS:
   - Fetch all `type, antecedents` rows from `rms_justifications` for the node.
   - If there are **no justifications**, the node is a **premise** — append it to `premises` and stop.
   - Otherwise, recurse into every antecedent of every justification (using a shared `visited` set to avoid cycles).
   - Result: `assumptions_by_node[nid]` = the union of *all* ultimate premises that (transitively) support *any* justification of `nid`.

3. **Rank premises.** Union all premises into `all_premises`. For each premise `p`:
   - `would_resolve` = the subset of input nodes whose assumption set contains `p`.
   - `dependent_count` = number of rows whose antecedents **or** outlist mention `p` (via `_find_dependents`, which uses JSONB `@>` containment on `rms_justifications`).
   - `entrenchment` = score from `_entrenchment` (see below).

4. **Sort ascending** by `entrenchment` and return.

`_entrenchment` scores a node as:
- `+100` per premise (if it has 0 justifications),
- `+50` if `source` is non-empty,
- `+25` if `source_hash` is non-empty,
- `+10 * dependent_count`,
- plus a `beliefs_type` bonus (`AXIOM/WARNING = 90`, `OBSERVATION = 80`, `DERIVED = 40`, `PREDICTED = 30`, `NOTE = 10`, default `20`).

Lower is weaker — hence the ascending sort yields "best candidates to retract first".

## 6. Side Effects

- Opens a psycopg cursor via `with self.conn.cursor() as cur:` and closes it on exit.
- Issues **read-only** queries against `rms_nodes`, `rms_justifications`, and the JSONB `antecedents`/`outlist` columns.
- **Does not** call `commit()` or `rollback()`. The cursor context manager in psycopg v3 closes the cursor but does not end the transaction, so any pending writes the caller has open remain pending.
- No log entries are written (`_log` is not invoked anywhere on this path).

## 7. Error Handling

- No `try`/`except` — psycopg exceptions propagate directly. A closed/broken connection, missing tables, or permission errors will raise to the caller.
- Missing node IDs are silently ignored — no `KeyError`.
- Empty input is silently tolerated.
- **Assumption not enforced by the type system** (and arguably a minor semantic gap): `would_resolve` is computed from the *union* of antecedents across *all* justifications of a node. If a node has multiple justifications, retracting the premise listed in `would_resolve` may **not** flip the node OUT, because another justification may still hold. So `would_resolve` really means *"appears in at least one justification path of"* — not *"retraction is guaranteed to resolve."* Callers who need guaranteed resolution must loop (`retract → propagate → re-check the nogood`).
- Similarly, `entrenchment` is a heuristic; there is no guarantee that retracting `culprits[0]` actually resolves the nogood, which is why `add_nogood` falls back to "retract node with fewest dependents" only when `culprits` is empty.

## 8. Usage Patterns

- **Public API / CLI**: the caller passes a list of node IDs and receives `{"culprits": [...]}` for presentation. It does **not** need to wrap this in a `with self.conn.cursor()` block — the function opens its own.
- **Internal use — `add_nogood`**: this method is already inside a `with self.conn.cursor() as cur:` block and calls `_find_culprits_internal` directly (bypassing the wrapper) so the same cursor/transaction is reused, then does:
  ```python
  if culprits:
      victim_id = culprits[0]["premise"]
  ```
  That is, it takes the **least-entrenched** premise as the backtracking victim.
- **Symmetry with `what_if_*`**: `what_if_retract` / `what_if_assert` are conceptually related operations, but they run inside a `try/finally: self.conn.rollback()` — they demonstrate that *read-only* helpers like `find_culprits` do not need explicit rollback.

**Caller obligations**
- Wrap retraction in a new transaction and **propagate** afterwards; do not assume a single retraction resolves the nogood.
- Treat `entrenchment` as a ranking heuristic, not a hard semantic guarantee.

## 9. Dependencies

- `self.conn` — a psycopg v3 connection (`psycopg.connect(...)` or injected).
- `self.domain_id` — bound as the first parameter to every query.
- `self._find_culprits_internal` — the actual implementation (`reasons/pg.py`).
- `self._trace_assumptions_recursive` — DFS through `rms_justifications.antecedents`.
- `self._entrenchment` — score computation for a premise.
- `self._find_dependents` — JSONB `@>` containment lookup against `antecedents` and `outlist`.
- Schema constants + `INDEXES` (defined at module top) — including the GIN indexes on `antecedents` and `outlist`.

---

## Topics to Explore

- [function] `reasons/pg.py:_find_culprits_internal` — The core logic that computes candidates, `would_resolve`, `dependent_count`, and entrenchment; the wrapper is only meaningful in relation to this method.
- [function] `reasons/pg.py:_trace_assumptions_recursive` — The DFS that walks `rms_justifications.antecedents` down to premises; understanding it is essential to understanding what `would_resolve` actually means.
- [function] `reasons/pg.py:_entrenchment` — The scoring rubric (premise bonus, source weights, dependent count, `beliefs_type` tiers) that determines retraction preference order.
- [function] `reasons/pg.py:add_nogood` — The primary consumer that selects `culprits[0].premise` as a backtrack victim and reconciles truth values via `_propagate`.
- [file] `tests/test_backtracking.py` — End-to-end tests that pin the backtracking semantics `find_culprits` is meant to support.

## Beliefs

- `find-culprits-returns-dict-wrapper` — `find_culprits` returns `{"culprits": [...]}` and never returns the raw list directly; callers must unwrap the `"culprits"` key.
- `find-culprits-sorts-by-entrenchment-ascending` — `_find_culprits_internal` sorts candidates by `entrenchment` in ascending order, so the weakest premise is at index 0.
- `find-culprits-traces-union-of-antecedents` — `_trace_assumptions_recursive` recurses through *every* justification of a derived node, so `would_resolve` reflects "any supporting justification," not "all supporting justifications."
- `find-culprits-skips-out-nodes` — Only input nodes whose `truth_value = 'IN'` contribute assumptions to the culprit search; OUT nodes are silently skipped.
- `find-culprits-is-read-only` — The function performs only `SELECT`s and issues no `commit()`, `rollback()`, or `_log()` call, so it cannot mutate the belief network or propagation log.
- `add-nogood-uses-first-culprit` — `add_nogood` invokes `_find_culprits_internal` directly (not the wrapper) and retracts `culprits[0]["premise"]` as the backtrack victim.
