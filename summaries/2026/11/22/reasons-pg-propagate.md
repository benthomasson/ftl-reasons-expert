---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: propagate in reasons/pg.py

# `PgApi.propagate`

## Purpose

`propagate` is the "recompute everything from scratch" operation for the PostgreSQL-backed dependency network. It walks every **derived** node in the domain, re-derives its truth value from its `rms_justifications` rows against the current state of the belief graph, and repeats the sweep until nothing changes (a fixpoint) — then commits.

It exists because the persisted `truth_value` column can drift out of sync with the justification graph:

- Bulk `import_json` / `import_beliefs` / `import_agent` build the graph in topo order but may leave nodes whose justifications referenced not-yet-final truth values.
- Direct DB edits, external migrations, or crashes between a node write and a dependent write can leave stale values.
- It's the audit / repair primitive, whereas `_propagate` (BFS from one changed id) is the incremental fast path used by `add_justification`, `retract_node`, `assert_node`, `challenge`, `supersede`, etc.

Crucially, it is **not** a drop-in replacement for `_propagate`: `_propagate` takes a cursor, doesn't commit, and returns `(went_out, went_in)`; `propagate` opens its own cursor, commits, and returns a flat `{"changed": [...]}`.

## Contract

**Preconditions**
- `self.conn` is an open psycopg v3 connection; `self.domain_id` is a UUID string scoping every query (multi-tenant safe).
- The `_retracted` metadata flag is the authoritative source of "sticky OUT". Any code that retired a node by writing `truth_value = 'OUT'` *without* `_retracted` will be considered "recomputable" and could be flipped back to `IN` if its justifications validate.

**Postconditions (on fixpoint)**
- Every non-retracted node with ≥ 1 row in `rms_justifications` has `truth_value` equal to what `_compute_truth` would return given the final state of all references.
- Every such flip is written to `rms_propagation_log` with `action = 'recompute'`.
- The transaction is committed exactly once at the end.

**Invariants**
- Premises (nodes with **no** `rms_justifications` rows) are invisible to this method — the query inner-joins on `rms_justifications`, so they are never selected, never recomputed, and never corrected.
- Nodes whose `metadata._retracted` is truthy are skipped each pass and retain `OUT`.
- `domain_id` scoping is enforced on every query.

**Non-guarantees**
- It does **not** guarantee convergence. If the fixpoint isn't reached within `node_count + 1` passes (e.g. oscillating cycles, or a pathological ordering), the loop exits silently and returns without error — the network may still be inconsistent.
- It does **not** guarantee that `changed` equals the initial-versus-final diff (see below).

## Parameters

None. The method reads `self.conn` and `self.domain_id` from `self`.

There is an implicit dependency on well-formed state:
- `rms_nodes.metadata` must be `NULL`, `{}`, or a JSON object. `json.loads(meta)` and `meta.get("_retracted")` will raise / fail if metadata is a JSON scalar or malformed.
- `rms_justifications.antecedents` / `.outlist` must parse to lists — enforced by early `.json.loads`, but not by the schema (JSONB accepts anything).

## Return Value

```python
{"changed": [<node_id>, ...]}
```

- A **list** (converted from a set, so ordering is unstable — treat as a set if order matters).
- Contains the *union* of all nodes that flipped at least once across every pass. It is **cumulative**, not an initial-vs-final diff: a node that flips `IN→OUT` on pass 1 and `OUT→IN` on pass 2 will still appear.
- Empty list `[]` means the network was already at a fixpoint (`changed_this_pass` was empty on the first sweep).
- Because the method commits internally and never raises on non-convergence, callers **cannot distinguish** "converged, nothing to do" from "converged after N flips" vs "did not converge, ran out of iterations." If they care, they must call again and check that the second call returns `[]`.

## Algorithm

```
pid           = self.domain_id
max_iterations = (SELECT COUNT(*) FROM rms_nodes WHERE domain_id = pid) + 1

for _ in range(max_iterations):
    rows = SELECT DISTINCT n.id, n.truth_value, n.metadata
           FROM rms_nodes n JOIN rms_justifications j
             ON j.node_id = n.id AND j.domain_id = n.domain_id
           WHERE n.domain_id = pid
           -- snapshot of every derived node's truth at start of pass

    changed_this_pass = []
    for (node_id, old_tv, meta) in rows:
        if meta is a JSON string: meta = json.loads(meta)
        if meta.get("_retracted"):     continue     # sticky OUT
        new_tv = self._compute_truth(cur, node_id)  # re-reads DB
        if old_tv != new_tv:
            UPDATE rms_nodes SET truth_value = new_tv
            self._log(cur, "recompute", node_id, new_tv)
            changed_this_pass.append(node_id)

    if not changed_this_pass: break     # fixpoint reached
    all_changed.update(changed_this_pass)

self.conn.commit()
return {"changed": list(all_changed)}
```

Key details:

1. **`DISTINCT`** collapses rows from a node with multiple justifications — the node is visited exactly once per pass.
2. **`old_tv` is start-of-pass**, while **`new_tv` reads the live DB**. That means mid-pass updates from earlier nodes are visible to later nodes within the same pass (progressive relaxation). The number of passes needed depends on row iteration order — worst case, one level of a chain per pass.
3. **`max_iterations = N_total + 1`** is a safe upper bound for acyclic graphs (a length-N chain needs at most N passes). It's also a safety stop for cyclic graphs.
4. **`_compute_truth` runs one query per visited node per pass**. This is deliberately simple, not batched. Compare with `_propagate`, which batches dependent and reference lookups.
5. **Fixpoint exit** is `if not changed_this_pass: break` — i.e. an entire sweep found nothing to change. Note that the final sweep that produces empty `changed_this_pass` is *always executed* after the last productive sweep; that's the standard two-phase convergence check.
6. The `_log` calls accumulate "recompute" entries; a flapping node will produce multiple log rows.

## Side Effects

- **Writes**: up to O(N·P) `UPDATE`s to `rms_nodes` and same number of inserts to `rms_propagation_log` (where P is passes; often P=1 or 2).
- **Transaction**: opens a cursor, does all its work inside it, and calls `self.conn.commit()` **once** after the cursor is closed. There are no intermediate commits — so if an exception occurs mid-loop, nothing from `propagate` is persisted *by this method*. Whether the enclosing caller's uncommitted work is committed or rolled back depends on how they handle the exception (the `__exit__` on `PgApi` rolls back on exception).
- **No in-memory cache invalidation** — the class is stateless apart from `conn` and `domain_id`.
- **Concurrency**: no locking (`SELECT … FOR UPDATE` absent), so concurrent writers can interleave with a pass and produce cosmetically spurious "changed" entries (the next `propagate` will settle them).

## Error Handling

- No try/except internally. Any exception propagates to the caller:
  - `psycopg` errors (connection broken, constraint violation) bubble up.
  - `json.loads` on malformed `metadata` raises `json.JSONDecodeError` — not handled.
  - `AttributeError` if `meta` is a non-dict truthy JSON value (e.g. list) and `.get` is called.
  - `_compute_truth` errors propagate unchanged.
- **Silent failure on non-convergence** — the most important error semantic. If `max_iterations` is exhausted without reaching a fixpoint, no exception and no warning are raised; the caller gets a `changed` list and a possibly inconsistent network.
- **Empty domain:** `max_iterations = 1`, first sweep yields no rows, `break` on first check, returns `{"changed": []}`.

## Usage Patterns

- **Post-import normalization** — the typical caller:
  - `PgApi.import_json(...)` calls `self.propagate()` after bulk `_add_node_raw` calls.
  - `PgApi.import_beliefs(...)` and `PgApi.import_agent(...)` / `PgApi.sync_agent(...)` also call it; the latter two call it multiple times around retract/assert fixups.
  - All of them **discard the return value** — they only care about the network state.
- **Manual repair / consistency audit** — external callers can call it directly after out-of-band writes and then inspect `changed`.
- **NOT used by single-mutation APIs** (`add_justification`, `retract_node`, `assert_node`, `remove_justification`, `supersede`, `convert_to_premise`, `challenge`) — those use `_propagate(cur, node_id)` for localized BFS.
- **Idempotent post-convergence** — running twice in a row: second call's first sweep finds nothing to change, returns `{"changed": []}` and produces no log rows.

Caller obligations:
- Treat the returned list as a *set* and do not interpret it as an initial-vs-final delta.
- If convergence matters for your use case, verify by re-invoking and checking for `[]`.
- Do not call inside a caller-managed transaction expecting it to stay open — `propagate` commits.

## Dependencies

- `self.conn.cursor()` / `self.conn.commit()` — psycopg v3.
- `self._compute_truth(cur, node_id)` — the truth-evaluation primitive; itself batches `SELECT id, truth_value … WHERE id = ANY(...)` for dependencies and outlists.
- `self._log(cur, action, target, value)` — inserts into `rms_propagation_log`.
- `json.loads` for the `metadata` JSONB column.
- Schema tables `rms_nodes`, `rms_justifications`, `rms_propagation_log`, `rms_network_meta` (indirectly).

**Assumptions not enforced by types:**
- `metadata` is a JSON *object* (`_retracted`/`.get` assume dict semantics).
- `_retracted` is the sole authoritative retraction flag — `truth_value = 'OUT'` alone does **not** prevent recomputation.
- Derived-vs-premise is determined purely by the presence of `rms_justifications` rows the join sees; `_add_node_raw` maintains this invariant but nothing in the schema enforces a "premises cannot have justifications" rule beyond the FK.
- The comparison `old_tv != new_tv` treats truth values as opaque strings; only `'IN'`/`'OUT'` are valid per the CHECK constraint.
- Row factory is the default tuple form (`for node_id, old_tv, meta in rows`), even though the module imports `dict_row` — that import is unused here.

## Topics to Explore

- [function] `reasons/pg.py:_propagate` — The targeted BFS counterpart; contrast its batched evaluation, cursor-scoped (non-committing) contract, and `(went_out, went_in)` return shape.
- [function] `reasons/pg.py:_compute_truth` — The evaluator that `propagate` depends on; explains how SL/CP justifications and outlists resolve to `IN`/`OUT`.
- [file] `tests/test_pg.py` — Exercises the PG backend; look for propagation fixtures that reveal convergence expectations and the `_retracted` skip behavior.
- [function] `reasons/pg.py:import_json` — A representative caller that relies on `propagate` to normalize a freshly-bulk-inserted network and then re-applies per-node `truth_value` overrides.
- [general] `retracted-vs-derived-out` — How `_retracted` metadata differs from a derived `OUT` truth; critical because `propagate` only respects the former.

## Beliefs

- `propagate-skips-retracted-nodes` — The loop reads `metadata._retracted` and `continue`s before recomputing, so retraction is sticky across `propagate` calls.
- `propagate-excludes-premises` — The `JOIN rms_justifications` in the node-selection query means a node with zero justification rows is never visited, so premises are never corrected by this method.
- `propagate-changed-list-is-union-not-diff` — The returned `all_changed` set accumulates node ids across every pass; it is not the symmetric difference between the initial and final truth vectors.
- `propagate-treats-truth-value-strings-as-opaque` — Comparison is `old_tv != new_tv` on raw strings; no semantic interpretation of `'IN'`/`'OUT'` occurs here (the CHECK constraint does enforcement).
- `propagate-has-silent-non-convergence` — When the fixpoint is not reached within `node_count + 1` passes, the loop exits with `all_changed` populated and no diagnostic flag; the caller cannot tell from the return value whether the network is actually consistent.
