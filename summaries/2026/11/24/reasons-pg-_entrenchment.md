---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _entrenchment in reasons/pg.py

# `_entrenchment` — Explanation

## Purpose

`_entrenchment` computes a **heuristic score** that quantifies how deeply "rooted" a node is in the belief network. Higher scores mean the belief is more settled and should be held onto; lower scores mean it is a good candidate for retraction.

It exists to support **dependency-directed backtracking**. When a nogood contradiction fires (a set of nodes cannot all be `IN` at once), the system traces each `IN` node back to its underlying **premises** (via `_trace_assumptions_recursive`) and then needs to decide *which premise to retract*. `_find_culprits_internal` sorts those premises **ascending** by `_entrenchment`, so the least entrenched (weakest / most consequential) premise is the victim.

## Contract

**Preconditions**
- `cur` is a live `psycopg` cursor attached to an open connection whose schema was created by `init_db` (`rms_nodes`, `rms_justifications` exist).
- `self.domain_id` is set to a valid domain (string/UUID).
- Caller holds no expectation about the cursor afterward — see Return Value.

**Postconditions**
- No database rows are modified; the function is a pure read.
- Cursor ends positioned wherever the last inner `SELECT` (from `_find_dependents`) left it — callers must not assume it is reusable without re-`execute`.
- Returns an `int` (never `None`). Missing node short-circuits to `0`.

**Invariants**
- Score is monotonically non-negative.
- Score is a heuristic sum, **not normalized**: a node with 100 dependents dwarfs any static classification bonus.
- If the node exists, the score is always ≥ 20 (default type bonus) + 0 or 100 (premise) + 0/50 (source) + 0/25 (hash) + 10·dependents.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `cur` | cursor | Shared psycopg cursor. Reused for several `SELECT`s. | Must be on `self.conn`; caller owns transaction state. |
| `node_id` | `str` | Node primary key within `self.domain_id`. | Unknown id → returns `0` silently (no `KeyError`). |

Note: `_entrenchment` **does not** read from a cache. It hits the DB every call. In `_find_culprits_internal` this means one round trip per premise candidate.

## Return Value

- `0` — node does not exist for this domain.
- Otherwise — an integer in roughly `[20, 265 + 10·N_dependents]`, where the `+100` premise bonus and `+90` AXIOM/WARNING bonus can dominate.

Callers must handle the `0` case: since `_find_culprits_internal` sorts **ascending**, a missing node (score 0) is treated as the *least* entrenched and will be selected first as a retraction victim. There is no explicit "node missing" signal in the return value.

## Algorithm

1. **Fetch the node row.**  
   `SELECT source, source_hash, metadata FROM rms_nodes WHERE id=%s AND domain_id=%s`.
   If empty → `0`.
2. **Coerce metadata.**  
   If the driver returned JSON as a string, parse it. Assume it is a `dict` afterward (`meta.get(...)` is called unguarded).
3. **Basic score = 0.**

4. **Premise bonus (+100).**  
   `SELECT COUNT(*) FROM rms_justifications WHERE node_id=%s AND domain_id=%s`.  
   Zero justifications ⇒ the node is a **premise / axiom** ⇒ +100.  
   (In practice, `_find_culprits_internal` only passes premises here, so this bonus is a *constant* for all candidates and doesn't change their relative ordering — it only matters if `_entrenchment` is ever called on derived nodes.)

5. **Evidence bonuses.**  
   - `source` non-empty → **+50**
   - `source_hash` non-empty → **+25** (implies the source has actually been hashed / verified)  
   Combined maximum = **+75** for well-cited, verified beliefs.

6. **Structural impact bonus (+10 per dependent).**  
   `self._find_dependents(cur, [node_id])` returns a `set` of nodes whose justifications reference this node (as an antecedent **or** in the outlist).  
   `score += len(deps) * 10`.  
   Rationale: retracting a node with many dependents causes more downstream cascades, so we prefer keeping it.  
   This is the only **unbounded** term.

7. **Classification bonus.**  
   Read `metadata["beliefs_type"]`, uppercase it, look up in the table:
   ```
   AXIOM  : 90      OBSERVATION : 80     DERIVED   : 40
   WARNING: 90      PREDICTED   : 30     NOTE      : 10
   default (unknown or missing)         : 20
   ```
   Missing/empty → `""` → uppercased → `""` → falls back to 20.

8. **Return sum.**

### Score breakdown table

| Component | With | Without |
|---|---|---|
| Premise (no justifications) | +100 | 0 |
| `source` present | +50 | 0 |
| `source_hash` present | +25 | 0 |
| Dependents | +10·`len(deps)` | 0 |
| `beliefs_type` AXIOM/WARNING | +90 | — |
| `beliefs_type` OBSERVATION | +80 | — |
| `beliefs_type` DERIVED | +40 | — |
| `beliefs_type` PREDICTED | +30 | — |
| `beliefs_type` NOTE | +10 | — |
| Unknown/missing type | — | +20 |

## Side Effects

- **Reads only** against `rms_nodes` and `rms_justifications`.
- Mutates the local `meta` variable only (no DB write, no shared state).
- Issues **N+2** queries per call, where `N = len(node_ids)` passed to `_find_dependents` — here `N = 1`, so 3 statements total: node lookup, justification count, dependent lookup.
- Cursor position changes (callers must treat `cur` as "used").

## Error Handling

- **Missing node**: swallowed — returns `0`.
- **Schema missing** (`rms_justifications` not created): `psycopg.errors.UndefinedTable` propagates. This is unreachable if `init_db` ran.
- **Malformed JSONB metadata** (`meta` arrives as an unparseable string): `json.loads` raises `json.JSONDecodeError`.
- **SQL `NULL` metadata**: schema declares `DEFAULT '{}'`, but if a row was inserted through an out-of-band path, `meta` would be `None` and `meta.get(...)` raises `AttributeError`. Not covered by the type system.
- **`beliefs_type` stored as a non-string** (e.g., `None` explicitly set via `set_metadata("beliefs_type", None)` — the default-arg `""` only covers *missing*, not *present-and-None*): `.upper()` raises `AttributeError`.
- **Broken connection / aborted transaction**: propagates from psycopg; the caller's enclosing `try/except` (in `add_nogood` etc.) is the safety net.

There is **no** internal try/except; every downstream error surfaces to the caller.

## Usage Patterns

Only call site inside the module:

```python
# in _find_culprits_internal
entrenchment = self._entrenchment(cur, premise_id)
candidates.append({
    "premise": premise_id,
    "would_resolve": would_resolve,
    "dependent_count": len(deps),   # computed separately with _find_dependents
    "entrenchment": entrenchment,
})
candidates.sort(key=lambda c: c["entrenchment"])
```

Caller obligations:
1. **Assumes `cur` already has `self.domain_id` context** — the method re-fetches and re-queries with `pid` each time.
2. **Ascending sort ⇒ lower score = retracted first.** The relative ranking of premises, *not* the absolute value, is what matters in the current caller.
3. **Not a transaction boundary** — cannot `commit()` or `rollback()` here; caller (`add_nogood`) owns the transaction.
4. **Must not mutate `cur`'s expected state** between calls to `_entrenchment`. Since `_find_culprits_internal` interleaves `_find_dependents` and `_entrenchment` calls, this is fine — but adding stateful queries in between would break the pattern.

This function could be reused anywhere a "which belief should I retract" heuristic is needed (e.g., a hypothetical `select_victim`), but today it is **entrenchment-for-backtracking-specific**.

## Dependencies

- `json` (module-level import) — for `isinstance(meta, str) → json.loads`.
- `self.domain_id` — supplied by `PgApi.__init__`.
- `self._find_dependents(cur, node_ids)` — `reasons/pg.py:_find_dependents`, uses JSONB `@>` containment on `rms_justifications.antecedents` and `.outlist`.
- Schema tables from `SCHEMA` (module constant): `rms_nodes`, `rms_justifications`.
- `psycopg` cursor API (`.execute`, `.fetchone`).
- Indirectly, callers rely on `_trace_assumptions_recursive` (produces the premise list passed in).

---

## Topics to Explore

- [function] `reasons/pg.py:_find_culprits_internal` — The only in-module caller; sorts candidates ascending by `_entrenchment` to choose a retraction victim.
- [function] `reasons/pg.py:_find_dependents` — Produces the dependent-count component (`10 × N`) and is called twice per candidate (once inside `_entrenchment`, once by the caller).
- [function] `reasons/pg.py:_trace_assumptions_recursive` — Determines the candidate premise set whose entrenchment is being scored.
- [file] `tests/test_backtracking.py` — Exercises the nogood → culprit → retract pipeline that consumes these scores.
- [general] `beliefs-type-taxonomy` — Where `metadata["beliefs_type"]` is populated (`import_beliefs`, `import_agent`) and how those values map to the static score tiers.

---

## Beliefs

- `_entrenchment-is-read-only` — The function issues only `SELECT` statements; no inserts/updates/deletes and no in-memory mutation escapes the call.
- `_entrenchment-returns-zero-for-missing-node` — A `node_id` absent from `rms_nodes` returns `0` rather than raising, which the ascending sort in `_find_culprits_internal` treats as "least entrenched."
- `_entrenchment-premise-bonus-is-100` — Zero rows in `rms_justifications` for the node adds exactly `100`, the single largest fixed component in the score.
- `_entrenchment-assumes-non-null-metadata` — `meta.get("beliefs_type", "")` and `.upper()` will raise `AttributeError` if `metadata` is SQL `NULL` or if `beliefs_type` is present-but-`None`; the type system does not enforce against this.
- `_entrenchment-dependents-dominate-static-bonuses` — Because `len(deps) * 10` is unbounded, a node with more than ~27 dependents can outscore any premise with an AXIOM classification and no source/hash bonuses.
