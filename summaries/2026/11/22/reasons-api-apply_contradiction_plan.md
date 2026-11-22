---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: apply_contradiction_plan in reasons/api.py

## `apply_contradiction_plan` — explanation

### 1. Purpose

`apply_contradiction_plan` is the **write-side counterpart** to `parse_contradiction_plan` (and ultimately to `detect_contradictions` / `write_contradiction_plan`). It takes a human-reviewed list of nogoods and applies them to the RMS, turning detected contradictions into graph-native *nogoods* — atomic records asserting "these beliefs cannot all be IN". Applying a nogood triggers dependency-directed backtracking: if all of its claims are currently IN, one of their supporting premises is retracted so the contradiction resolves.

It exists so callers (typically the CLI after a human edits the plan file) don't have to loop over entries, handle per-entry errors, and reformat `add_nogood`'s result themselves.

### 2. Contract

**Preconditions (assumed, not enforced):**
- `plan` is a `list[dict]` where each entry has at least keys `"id"` (str) and `"claims"` (list[str]) — this is exactly what `parse_contradiction_plan` produces.
- The claim IDs name node(s) that still exist (or existed) in the network; stale claims are tolerated but only recorded as dormant nogoods.
- `db_path` points to a valid SQLite RMS database.

**Postconditions:**
- For every entry, either an `add_nogood` side effect occurred (DB write + propagation + possibly a cascade retraction) **or** an error string was collected.
- `len(nogoods) + len(errors) == len(plan)` in the normal path.
- `applied == len(nogoods)`.

**Invariants:**
- Per-entry atomicity: each entry is committed independently via `add_nogood`'s own `_with_network(write=True)` block. If entry *N* fails, entries *0…N‑1* remain committed.
- No cross-entry transaction.

### 3. Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `plan` | `list[dict]` | Reviewed entries `{"id": str, "claims": list[str]}` | Empty list → returns `{"applied": 0, "nogoods": [], "errors": []}`. Missing `"claims"` → `KeyError` caught, error string appended. Missing `"id"` → **uncaught** `KeyError` from the exception handler (see §7). |
| `db_path` | `str` | SQLite database path; defaults to `DEFAULT_DB` (`"reasons.db"`) | Unlike most other functions in `api.py`, this one has **no `pg_conninfo`/`domain_id`** — it is SQLite-only. |

### 4. Return Value

```python
{
  "applied": int,                     # number of successfully applied nogoods
  "nogoods": [                        # one per successful entry
    {
      "id": str,                      # the plan entry's original "id"
      "nogood_id": str | None,        # the internal nogood id (usually populated)
      "changed": list[str],           # nodes whose truth_value flipped
      "backtracked_to": str | None,   # premise actually retracted, or None
    }
  ],
  "errors": list[str],                # free-text failure messages
}
```

Callers must treat `backtracked_to=None` as "nogood recorded, no backtracking performed" — which happens when the claims were not all IN at apply time (`add_nogood` only computes culprits when `all_in`). It is **not** an error.

`errors` entries are opaque strings (`"Failed to apply {id}: {exc}"`), not structured — callers that need to distinguish "missing claim" from "DB error" must parse them.

### 5. Algorithm

```
nogoods = []
errors  = []
for entry in plan:
    try:
        result = add_nogood(entry["claims"], db_path=db_path)
        nogoods.append({
            "id": entry["id"],
            "nogood_id": result.get("nogood_id"),
            "changed": result.get("changed", []),
            "backtracked_to": result.get("backtracked_to"),
        })
    except Exception as e:
        errors.append(f"Failed to apply {entry['id']}: {e}")
return {"applied": len(nogoods), "nogoods": nogoods, "errors": errors}
```

The heavy lifting is in `add_nogood`, which this function calls once per plan entry:

1. Opens a write context (`_with_network(db_path, write=True)`).
2. Checks whether *all* claims are currently IN.
3. If so, calls `net.find_culprits(node_ids)` and picks the first culprit's premise as `backtracked_to`.
4. Calls `net.add_nogood(node_ids)` — records the nogood and, if all claims were IN, retracts that premise and cascades.
5. Saves the network, publishes pubsub events if subscribers exist.
6. Returns `{"nogood_id", "nodes", "changed", "backtracked_to"}`.

So `apply_contradiction_plan` itself is a thin, fault-isolating loop with a normalized result shape.

### 6. Side Effects

Per plan entry, the function (via `add_nogood`):
- **Writes to the database** — persists the new nogood and any retracted/asserted nodes.
- **May retract a premise**, cascading truth-value changes to all dependents.
- **Emits pubsub events** via `_with_network.__exit__` when there are subscribers.
- **Updates the in-memory network cache** keyed by `(db_path, mtime)`.
- **Opens and closes a `Storage` connection** for each entry (no connection reuse across the plan).

No filesystem I/O beyond the DB, no LLM calls, no network. There is **no rollback** — a partial save leaves a partially applied plan.

### 7. Error Handling

- The `try` block is broad — `except Exception as e` — so failures inside `add_nogood` (unknown claim IDs downstream, propagation errors, SQLite errors) are captured as error strings and execution continues.
- **Silent catalogue problem:** the exception handler formats with `entry['id']`. If `entry` lacks `"id"`, that `KeyError` fires *inside* the handler and propagates out of the function. So `"id"` is a hard precondition for error reporting, even though the type annotation only says `list[dict]`.
- `add_nogood` itself returns `backtracked_to=None` rather than raising when claims aren't all IN — this is treated as a successful (dormant) application, not an error.
- No row-count or return-value checks: if `add_nogood` returns a dict missing `"nogood_id"`, `.get(...)` returns `None` and the nogood is still counted as applied. The function trusts its dependency.

### 8. Usage Patterns

Typical CLI flow (round-trips with the rest of `api.py`):

```
detect_contradictions(...)          # LLM finds candidate contradictions
    → write_contradiction_plan(...) # writes reasons-contradiction.md
    → (human edits: APPLY / SKIP / delete)
    → parse_contradiction_plan(text)  # → list[{"id", "claims"}]
    → apply_contradiction_plan(parsed)  # this function
```

Caller obligations:
- Do the human review **before** calling — nothing here filters or gates entries.
- Treat `"errors"` as non-fatal; decide whether any errors warrant re-running.
- Treat `"applied"` as "entries that produced a nogood record", not "contradictions resolved" — a claim set that's already OUT will be applied with `backtracked_to=None`.
- Because each entry commits independently, re-invoking with the same plan will record duplicates; the caller must guarantee idempotency (e.g., by re-parsing a fresh, human-updated plan file).

### 9. Dependencies

- **`add_nogood`** (`reasons/api.py`) — the per-entry primitive; the real work lives here.
- **`DEFAULT_DB`** (`reasons/api.py`) — the default path constant.
- Indirectly, via `add_nogood`:
  - `_with_network` — write-mode load/save + pubsub diffing.
  - `Storage` (`reasons/storage.py`) — SQLite persistence.
  - `Network.add_nogood`, `Network.find_culprits`, `Network.recompute_all` (in `reasons/network.py`) — propagation and backtracking semantics.
  - `pubsub` (`reasons/pubsub.py`) — event emission.
- Not used here: `pg_conninfo` / `_pg_dispatch`, LLM modules, `export_network`.

### Assumptions not enforced by the type system

- Every dict in `plan` contains `"id"` — required, because the error branch dereferences it.
- `"claims"` is a list of node IDs, not datalog literals or raw text.
- Claim IDs refer to **already-resolved** namespaced IDs (this function does no `_resolve_namespace`).
- Idempotency is the caller's responsibility; the function will happily insert duplicate nogoods.
- SQLite-only — passing a PostgreSQL `db_path` will not work and there is no dispatch guard.

---

## Topics to Explore

- [file] `reasons/contradictions.py` — Source of the contradictions this plan applies; shows the LLM-side candidate format and severity taxonomy.
- [function] `reasons/api.py:add_nogood` — The primitive behind each entry; its all_in gate, culprit selection, and `backtracked_to` semantics determine the observable behavior of `apply_contradiction_plan`.
- [function] `reasons/api.py:parse_contradiction_plan` — Defines the exact `{"id", "claims"}` shape consumed here; worth testing the parse→apply round-trip including backtick stripping.
- [file] `reasons/network.py` — `Network.add_nogood` and `Network.find_culprits` implement the actual backtracking algorithm that produces `backtracked_to` and `changed`.
- [file] `tests/test_contradictions.py` — Acceptance tests for the apply flow; useful to confirm expected behavior for dormancy (claims not all IN) and partial failures.

## Beliefs

- `apply-contradiction-plan-is-sqlite-only` — Unlike most other `api.py` functions, this function has no `pg_conninfo` / `domain_id` parameters and therefore only works against SQLite.
- `apply-contradiction-plan-is-non-atomic` — Each plan entry is applied by a separate `add_nogood` write transaction; a failure on entry *N* does not roll back entries *0…N‑1*.
- `apply-contradiction-plan-requires-id-key` — The exception handler formats errors with `entry['id']`, so a missing `"id"` raises an uncaught `KeyError` rather than being reported in `errors`.
- `apply-contradiction-plan-maps-to-add-nogood` — The function is a thin loop over `add_nogood`; it adds no validation, no filtering, and no idempotency checks beyond what the caller supplies.
- `apply-contradiction-plan-produces-backtracked-to` — Each successful result includes the premise actually retracted by `add_nogood`, or `None` when the claim set was not fully IN at apply time.
