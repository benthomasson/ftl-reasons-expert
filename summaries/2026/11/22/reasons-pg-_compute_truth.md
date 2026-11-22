---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _compute_truth in reasons/pg.py

# `_compute_truth` — PostgreSQL-native belief recomputation

## Purpose

`PgApi._compute_truth(self, cur, node_id)` is the **truth-function evaluator** for the PostgreSQL storage backend of the reasons/dependency network. Given a node, it reads its justifications (`rms_justifications`) and the current truth values of every node referenced by those justifications (`rms_nodes`), and returns the node's computed truth value (`"IN"` or `"OUT"`) according to the standard RMS semantics:

> A node is `IN` iff **at least one** of its justifications is satisfied.
> A justification is satisfied iff **all** its antecedents are `IN` **and all** its outlist nodes are `OUT`.

This mirrors the classic AI "justification-based truth maintenance system" (JTMS). It exists so that every mutation path (`add_node`, `add_justification`, `remove_justification`, `challenge`, `supersede`, `summarize`, `propagate`) can recompute a node's status from first principles without loading the whole network — only the node's justifications and its direct references are read.

---

## Contract

**Preconditions**

- `self.domain_id` is set and valid for the active connection.
- `cur` is an open cursor on `self.conn`, inside a transaction that **also** carries any pending writes to `rms_justifications`/`rms_nodes` for `node_id` and its references. This is critical: the function relies on reading *uncommitted* state (see `_add_node_raw`, `add_justification`, etc. — every caller writes first, then calls `_compute_truth` without committing).
- The truth values of the referenced nodes are already correct at call time. `_compute_truth` does **not** recursively recompute them; the caller must either insert dependencies topologically (`import_json`, `import_beliefs`) or invoke the fixpoint loop `propagate()`.

**Postconditions**

- Returns the truth value as a `str` (`"IN"` or `"OUT"`).
- Does **not** modify the database. Caller must `UPDATE rms_nodes SET truth_value = ...` themselves.
- Does **not** commit or rollback.

**Invariants**

- Domain isolation: every query is scoped by `domain_id`, so cross-domain references return no rows and are treated as missing.
- Purely a function of the current DB state; no caching across calls.
- Empty `rms_justifications` ⇒ `"IN"` (premise).

---

## Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `cur` | `psycopg.Cursor` | An open cursor on the current transaction. Must not be committed while in use. |
| `node_id` | `str` | The node to evaluate. Must exist in `rms_nodes` for `self.domain_id` — but the function does **not** check this itself; a nonexistent `node_id` simply yields no justifications, so it returns `"IN"` (treated as premise). |

**Edge cases**

- Node with zero justifications (premise or retracted-then-recreated) → `"IN"`.
- Justification with empty antecedents and empty outlist → vacuously satisfied → returns `"IN"` (unconditional support).
- Referenced node deleted: `truth_cache.get(a, "OUT")` treats missing antecedent as `"OUT"` (fails the check), and missing outlist entry as `"OUT"` (passes the check). By design: a missing function/negation node doesn't block, but a missing prerequisite does.
- JSONB columns arriving as Python objects vs strings: the code defensively runs `json.loads` if `isinstance(..., str)` — psycopg v3 normally returns them parsed, so this is defensive.
- A `CP`-type justification (conditional proof) is evaluated with **identical** semantics as `SL`. Type is fetched but never consulted.

---

## Return Value

Returns a `str`, exactly one of `"IN"` or `"OUT"` — never `None`, never a bool.

Caller responsibilities:

- Persist the value into `rms_nodes.truth_value` if it changed.
- Trigger downstream propagation (`_propagate`) or scheduling (`propagate` fixpoint) if the value changed.
- Log the change (e.g. `self._log(cur, "recompute", ...)` — that's done by callers, not here).

---

## Algorithm

1. **Fetch justifications** (1st round-trip):
   ```sql
   SELECT type, antecedents, outlist FROM rms_justifications
   WHERE node_id = %s AND domain_id = %s
   ```
   If empty → return `"IN"` (it's a premise).

2. **Parse and collect references.** Iterate the rows, coercing `antecedents` and `outlist` from strings to lists if needed, and accumulate all referenced node IDs into `all_refs`. Store parsed triples in `parsed`. `type` is captured but never read afterwards — this is an important latent assumption (see Topics).

3. **Batch-fetch truth values** (2nd round-trip, only if `all_refs` is non-empty):
   ```sql
   SELECT id, truth_value FROM rms_nodes
   WHERE domain_id = %s AND id = ANY(%s)
   ```
   Populate `truth_cache: dict[str, str]`.

4. **Evaluate each justification** (in insertion order — `SERIAL id` ordering by default, though `ORDER BY` is not specified, so evaluation order is undefined for short-circuit efficiency only; the *result* is order-independent because it's a disjunction):
   - `inlist_ok = all(truth_cache.get(a, "OUT") == "IN" for a in ants)`
   - `outlist_ok = all(truth_cache.get(o, "OUT") == "OUT" for o in outs)`
   - If both, return `"IN"` immediately.

5. **Fallthrough** → `"OUT"`.

Note the short-circuit: on the first satisfied justification, evaluation stops. This affects only performance, not result.

---

## Side Effects

- **Reads** `rms_justifications` and `rms_nodes` (2 queries max).
- **Modifies transaction-local state** only by consuming cursor rows — no writes.
- **No** commits, no logs, no notifications.
- All logic is executed inside the caller's transaction, so any writes the caller made *before* this call are visible.

---

## Error Handling

- No `try/except`. `psycopg` errors (connection failure, query errors) propagate as-is.
- `json.loads` failures propagate as `json.JSONDecodeError` — but only in the (rare) case where `ants`/`outs` is a string that is not valid JSON. Since the columns are `JSONB NOT NULL DEFAULT '[]'`, this should not happen in practice.
- Nonexistent `node_id` does **not** raise — it silently returns `"IN"` (called out above). This is a latent footgun: if callers rely on this raising an error to detect stale IDs, they'll be surprised.
- Missing referenced nodes are silently handled by the `truth_cache.get(..., "OUT")` default rather than raising.

---

## Usage Patterns

Callers, and what they do around `_compute_truth`:

| Caller | Pattern |
|--------|---------|
| `_add_node_raw` | Insert justifications → `_compute_truth` → `UPDATE truth_value` → `_log`. Skips calling `_compute_truth` entirely when there are no justifications (inline `"IN"`). |
| `add_justification` | Insert new justification → recompute → if changed, `UPDATE` + `_propagate` cascade. |
| `remove_justification` | Delete justification → recompute → cascade. |
| `challenge` | Append challenge id to target's outlist → recompute → cascade. |
| `supersede` | Append `new_id` to old node's outlist → recompute → cascade. |
| `summarize` | Insert a summarizing justification → recompute summary node. |
| `propagate` | Iterates **all** derived nodes, calling `_compute_truth` per node per pass, until a fixpoint (`changed_this_pass` empty). |
| `_propagate` | **Duplicates the same logic inline** in the BFS cascade path, instead of calling this method — a maintenance seam. |

Obligations every caller respects:

1. Do the write **before** calling.
2. Caller owns `UPDATE` of the node's truth value.
3. Caller owns cascade / propagation and `_log`.
4. Do not commit in between.

---

## Dependencies

- `json` (stdlib) — used only for defensive `json.loads` of JSONB columns.
- `self.domain_id`, `self.conn` (via `cur`).
- Tables `rms_nodes` and `rms_justifications`.
- Implicitly, the schema CHECK `truth_value IN ('IN','OUT')` keeps the values sane.

No external calls, no LLM, no filesystem — this is the fast, DB-only core of the truth engine.

---

## Topics to Explore

- [function] `reasons/pg.py:_propagate` — Contains an inline copy of `_compute_truth`'s evaluation logic; verifying these stay in sync is an ongoing risk.
- [function] `reasons/pg.py:propagate` — The fixpoint driver; understanding it clarifies what fixpoint convergence really means in this pull-based recompute model.
- [function] `reasons/pg.py:_justification_valid_cached` — A third copy of the same "all antecedents IN, all outlist OUT" predicate used inside `explain_node`.
- [file] `tests/test_pg.py` — Look for tests covering CP justifications, empty-outlist justifications, and cascade behavior; these pin down the actual semantics assumed by the evaluator.
- [general] `rms-cp-semantics` — Whether `CP` (conditional proof) should be treated as logically distinct from `SL`; the current code treats them identically, which may be either simplification or a bug.

## Beliefs

- `_compute_truth-ignores-justification-type` — The `type` column is fetched (`SL`/`CP`) but never consumed; both are evaluated with identical conjunctive/negative semantics.
- `_compute_truth-is-read-only` — The function only issues SELECT statements and never updates, commits, or logs; the caller is responsible for persisting the returned value.
- `_compute_truth-returns-IN-for-premises` — If `rms_justifications` contains no rows for `node_id` in the current domain, the function returns `"IN"`.
- `_compute_truth-treats-missing-refs-as-OUT` — The `truth_cache.get(..., "OUT")` fallback means a referenced node that no longer exists causes antecedents to fail the check but outlist entries to pass it.
- `_compute_truth-requires-uncommitted-visibility` — Callers must invoke it inside the same transaction that wrote the justifications being evaluated, because it reads transaction-local state.
