---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Map whether the guarded `UPDATE` is the whole accept operation or merely one statement inside a larger read-modify-write workflow.

## Mapping the guarded `UPDATE`

**Verdict:** The guarded `UPDATE` is **not** the whole accept operation. It is a status-transition statement inside a larger accept workflow that also handles drift detection, concurrency guarding, and post-accept mutation.

| Accept phase | Evidence | Implication |
|---|---|---|
| Entry point | `reasons/api.py:5622` defines `accept_proposal(...)` | Accept is implemented as a function, not a single SQL statement. |
| Backend dispatch | `reasons/api.py:5636` calls `_pg_dispatch(pg_conninfo, domain_id, "accept_proposal", ...)` | Postgres accept behavior is delegated elsewhere; the local/SQLite path is only part of the picture. |
| Drift handling | `reasons/api.py:5633` says “On drift, marks it stale without applying”; `reasons/api.py:5670` has `UPDATE proposals SET status = 'stale'...` | Accept includes validation/read logic before deciding whether to apply the proposal. |
| Guarded accepted transition | `reasons/api.py:5776` has `UPDATE proposals SET status = 'accepted', updated_at = ?, ...` | This is the likely guarded UPDATE: a compare-and-swap style status transition. |
| Race guard | `reasons/api.py:5787` says `"Concurrent accept race: proposal no longer pending"` | The code checks whether the guarded UPDATE actually affected a pending proposal. This is a concurrency guard around the status transition. |
| Return state | `reasons/api.py:5795` returns `"status": "accepted"` | The accepted status is reported after the guarded transition, but the transition is not the only work in accept. |
| Side effects beyond status | `reasons/api.py:5461` says “On accept, calls add_nogood() which triggers dependency-directed...” | Accept also performs domain mutation / propagation, not just `proposals.status = 'accepted'`. |
| No named accept guard found | The `guard` grep only found cycle guards at `reasons/api.py:3378`, `reasons/derive.py:123`, and a test reference | The guard is likely SQL/rowcount-based, not a named helper function. |

So the guarded `UPDATE` at `reasons/api.py:5776` is best understood as **one statement inside a larger read-modify-write workflow**:

1. Read/validate the proposal and detect drift.
2. Possibly mark it stale instead of applying it.
3. Attempt a guarded status transition to `accepted`.
4. Detect concurrent accept races if the transition did not affect a pending row.
5. Apply accepted proposal effects, including `add_nogood()`-triggered dependency-directed behavior.
6. Return an accepted result.

### What is missing

The observations do **not** include the full body of `accept_proposal` around `reasons/api.py:5622-5800`, nor the exact `WHERE` clause on the `UPDATE` at line 5776, nor the `_pg_dispatch` Postgres implementation. Therefore we cannot confirm the exact transaction boundary or whether the status transition happens before or after applying proposal effects. But the available evidence is enough to rule out the guarded `UPDATE` being the entire accept operation.

## Topics to Explore

- [file] `reasons/api.py` — Read the full `accept_proposal` region around lines 5622–5800, especially the `WHERE` clause and rowcount handling around line 5776.
- [function] `reasons/api.py:accept_proposal` — Understand how local/SQLite accept differs from the Postgres dispatch path.
- [function] `reasons/api.py:_pg_dispatch` — Check whether the Postgres accept implementation contains its own guarded `UPDATE` and transaction boundary.
- [general] `accept-proposal-transaction-boundaries` — Determine whether the status transition and `add_nogood()` side effects are atomic.
- [file] `reasons/pg.py` — Inspect Postgres mutations to `rms_nodes` and `rms_justifications` that may be invoked during accept.

## Beliefs

- `accept-proposal-dispatches-to-pg-backend` — `reasons/api.py:5622` `accept_proposal` calls `_pg_dispatch` at line 5636 when a Postgres connection is provided.
- `accepted-status-update-is-inside-accept-handler` — The `UPDATE proposals SET status = 'accepted'` at `reasons/api.py:5776` is part of accept handling, not a standalone operation.
- `accept-handler-has-stale-without-apply-branch` — `accept_proposal` documents at `reasons/api.py:5633` that drift marks a proposal stale without applying it, with a stale `UPDATE` at line 5670.
- `accept-path-detects-concurrent-accept-race` — The accept path reports `"Concurrent accept race: proposal no longer pending"` at `reasons/api.py:5787`, indicating a conditional pending-status check.
- `accept-operation-invokes-add_nogood` — The code documents at `reasons/api.py:5461` that accepting a proposal calls `add_nogood()`, producing dependency-directed effects beyond the status `UPDATE`.
