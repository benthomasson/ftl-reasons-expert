---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _justification_valid_cached in reasons/pg.py

## `_justification_valid_cached` (in `reasons/pg.py`)

### 1. Purpose

Evaluates whether a single justification (an `SL`/`CP` rule with `antecedents` and `outlist`) is currently satisfied against the live `truth_value`s in the database. It is the read-only check used by `_explain_recursive` to decide which justification is responsible for an `IN` node's truth, so the explanation can recurse through that justification's antecedents.

Despite the name, it does **not** accept a caller-provided cache and does **not** persist a cache across calls. The word "cached" refers only to the fact that it batches all reference lookups into one `SELECT` and keeps the results in a local `truth_cache` dict for the duration of the call — avoiding an N+1 query pattern within that one invocation.

It mirrors the logic inline in `_compute_truth` and `_propagate`, and is essentially a copy of that same validity predicate for the read/explain path.

### 2. Contract

**Preconditions**
- `cur` is an *open* psycopg v3 cursor from `self.conn`, executing inside a valid transaction (this function does not commit or rollback).
- `antecedents` and `outlist` are **re-iterable** collections of node-ID strings (typically lists). If they were loaded from JSONB, they must already have been `json.loads`-decoded by the caller — a raw JSON string would be treated as a sequence of characters.
- `self.domain_id` correctly identifies the tenant/domain whose `rms_nodes` rows should be queried.

**Postconditions**
- Returns `True` iff *every* entry in `antecedents` has `truth_value = 'IN'` **and** *every* entry in `outlist` has `truth_value = 'OUT'`.
- If both inputs are empty, returns `True` (vacuously valid — e.g. a premise-style or unguarded ``SL`` justification).
- No writes, no logging, no metadata changes, no transaction state change.

**Invariants**
- The query is always scoped by `domain_id = self.domain_id`, so IDs from other domains are treated as missing.
- "Out" is the default for referenced nodes that don't resolve in the DB — i.e. missing = `"OUT"` on both sides.

### 3. Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `cur` | `psycopg.Cursor` | Active cursor used for one `SELECT` | If closed or aborted transaction, `psycopg` raises |
| `antecedents` | Iterable[str] | Node IDs that must currently be `IN` | Empty → check trivially true. Missing ID → treated as `"OUT"` → the whole check fails |
| `outlist` | Iterable[str] | Node IDs that must currently be `OUT` | Empty → trivially true. Missing ID → treated as `"OUT"` → the whole check **passes** (see note below) |

**Non-type-enforced assumptions** (worth flagging for a code reviewer):

- Both arguments must be **re-iterable** — they are each consumed twice (once inside `set(...)`, once inside the subsequent `all(...)`). Passing a generator would make the second pass iterate over nothing, silently turning `inlist_ok`/`outlist_ok` into vacuous `True`s.
- Both arguments must be **plain Python iterables**, *not* the raw JSON string returned from a `JSONB` column. Passing `'["a","b"]'` would iterate over characters and query the DB for `"["`, `"\""`, etc. — which won't match any row; the check would return `False`, not raise.
- The function *assumes* referential integrity of `antecedents`/`outlist` is not strictly enforced at the DB layer (they are JSONB, not FKs), and therefore defensively tolerates unknown IDs.

### 4. Return Value

- Returns a Python `bool`.
- `True` = at least the intent of an `SL`/`CP` rule is met.
- `False` = rule is broken or undefined.

The caller does not need to distinguish "invalid because antecedent is OUT" from "invalid because outlist element is IN" — that distinction is reconstructed separately by `_explain_recursive` in the `OUT` branch by fetching the same referenced IDs and diffing them itself.

### 5. Algorithm

1. `pid = self.domain_id`
2. Compute `all_refs = set(antecedents) | set(outlist)` — this deduplicates the union of both lists, capturing every ID that will need to be checked.
3. Short-circuit: `if not all_refs: return True` (a justification with no dependencies is valid).
4. Single batched query:
   ```sql
   SELECT id, truth_value FROM rms_nodes
   WHERE domain_id = %s AND id = ANY(%s)
   ```
   using the union list. Rows returned are those referenced IDs that actually exist in this domain.
5. Build `truth_cache = {row.id: row.truth_value}`.
6. `inlist_ok = all(truth_cache.get(a, "OUT") == "IN" for a in antecedents)` — a missing antecedent fails (default `"OUT"` ≠ `"IN"`).
7. `outlist_ok = all(truth_cache.get(o, "OUT") == "OUT" for o in outlist)` — a missing outlist element *passes* (default `"OUT"` == `"OUT"`).
8. Return `inlist_ok and outlist_ok`.

Note the asymmetry in step 6/7: "missing → OUT" is the ambiguous default that produces opposite results on the two sides.

### 6. Side Effects

- One `SELECT` against `rms_nodes` per call.
- No INSERT/UPDATE/DELETE, no `_log()` calls, no cursor state changes, no transaction commits.
- The cursor's `rowcount`/result set is left pointing at the last fetch — same as any other `cur.execute(...)`.

### 7. Error Handling

- No `try`/`except`.
- Any `psycopg` error (closed cursor, aborted transaction, missing table, permission, encoding, etc.) propagates to the caller.
- Missing referenced nodes are **silently tolerated** via `.get(..., "OUT")`. There is no warning, no exception, no log.
- Invalid argument *shape* (generator, JSON string) is silently tolerated with the consequences described above — no validation, no type check.

### 8. Usage Patterns

Only caller observed in this file is `_explain_recursive`:

```python
if self._justification_valid_cached(cur, ants, outs):
    step = {... "reason": f"{jtype} justification valid" ...}
    steps.append(step)
    for ant_id in ants:
        steps.extend(self._explain_recursive(cur, ant_id, visible_to, visited))
    break
```

Caller obligations:

- Must pass an open cursor (typically the same one being used for the rest of the explain walk).
- Must have already `json.loads`-decoded `ants` and `outs` (which `_explain_recursive` does).
- Must not expect the function to distinguish *why* a justification is invalid — that responsibility is handled in the sibling `else` branch of `_explain_recursive`.
- Should be aware that the returned verdict reflects a snapshot: if a concurrent writer (another transaction) changes a referenced `truth_value` between this call and the caller's use of its result, the caller's expansion is based on the older snapshot.
- Must tolerate missing referenced nodes being treated as `"OUT"` on both sides (see caveat below).

### 9. Dependencies

- **`psycopg`** — cursor protocol only (`cur.execute`, `cur.fetchall`).
- **`rms_nodes` table** — schema columns `id`, `domain_id`, `truth_value` (see `SCHEMA` at top of file).
- **`self.domain_id`** — the tenant scope.
- No dependency on other private methods or module-level helpers.

### 10. Notable Assumptions / Sharp Edges

1. **Asymmetric default for missing IDs.** A missing *antecedent* makes the justification invalid; a missing *outlist* element silently makes the rule *more likely* to be valid. If a referenced node can be physically deleted without cleaning `rms_justifications.antecedents`/`outlist` (the FK is only on the owning node, not the JSON reference arrays), an `outlist` entry pointing to a deleted node will not gate its dependent node. This is a real behavioural assumption, not enforced by the schema.

2. **"Re-iterable" and "already decoded" are not enforced.** Passing a generator silently returns a wrong answer; passing a JSON string silently queries for one-character IDs and typically returns `False`. Neither raises.

3. **Duplicated logic.** The default-`"OUT"` validity predicate also appears inline in `_compute_truth` (the write/precedence path) and again in `_propagate` (BFS propagation). The three implementations must stay consistent — a natural refactor target would be to lift this into a shared helper taking a pre-built `truth_cache`.

4. **Snapshot semantics.** The `truth_cache` is built from a single `SELECT`; results are consistent within the transaction under PostgreSQL's default READ COMMITTED only for that statement. Any subsequent reads by the caller of the same nodes could observe a different value if another transaction committed in between.

---

## Topics to Explore

- [function] `reasons/pg.py:_explain_recursive` — The sole caller; shows how validity gates the recursive walk and how the `else` branch reconstructs the failure reason.
- [function] `reasons/pg.py:_compute_truth` — Duplicate of the same "missing ⇒ OUT" validity predicate on the write path; good for comparing the two implementations.
- [function] `reasons/pg.py:_propagate` — Another copy of the predicate inside BFS propagation; shows how the invariant is reused at scale.
- [file] `reasons/pg.py` — The `rms_nodes`/`rms_justifications` schema at the top is needed to reason about referential integrity for the JSON reference arrays.
- [general] `outlist-semantics` — Why missing `outlist` IDs default to "satisfied" and whether referencing-but-deleted nodes can silently unblock dependents.

## Beliefs

- `justification-validity-uses-out-default-for-missing-nodes` — `truth_cache.get(x, "OUT")` causes missing antecedents to fail validity while missing outlist IDs pass it, an unenforced referential-integrity assumption.
- `_justification_valid_cached-requires-reiterable-arguments` — The function iterates `antecedents` and `outlist` twice each, so generators would silently produce vacuous `True`s rather than raising.
- `justification-validity-predicate-duplicated-in-compute-truth-propagate-and-explain` — Equivalent logic appears in `_compute_truth`, `_propagate`, and `_justification_valid_cached`, so all three must remain semantically in sync manually.
- `_justification_valid_cached-is-a-helper-only-for-explain-node` — Its only caller is `_explain_recursive`, which uses it to select the responsible justification for an IN node.
- `_justification_valid_cached-produces-bool-and-never-raises-for-missing-nodes` — It returns a plain boolean for present references and treats absent ones without error or warning.
