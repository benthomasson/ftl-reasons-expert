---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: accept_proposal in reasons/api.py

# `reasons/api.py:accept_proposal`

## Purpose

`accept_proposal` is the **"apply" side of the proposals workflow**. Proposals are *pending, reversible, unapplied* change requests (retract / supersede / add / nogood) stored in a `proposals` SQLite table, typically created by `propose_retraction`, `propose_supersession`, `propose_addition`, `propose_nogood`, or `store_llm_proposals`. Accepting a proposal is the moment a proposal becomes a truth-value change.

It exists because proposal creation may happen outside the request's authority (an LLM, a remote agent, a human reviewer). By the time a reviewer clicks "accept", the network may have drifted — nodes may have been added, retracted, changed text, or won/lost justifications. `accept_proposal` **re-validates against the live network** before applying, and if drift is detected, marks the proposal **`stale`** instead of applying it. The intent is to avoid "blindly replaying" a proposal whose assumptions are no longer true.

The docstring claims this is done "atomically after re-validation"; in practice it is *not* a single transaction (see **Assumptions** below).

## Contract

**Preconditions**
- `proposal_id` must exist in the `proposals` table.
- The proposal's `status` must be `'pending'` (runtime-checked, not typed).
- For action `add`: `target_id` must NOT already exist in the network.
- For action `retract`: `target_id` must exist, be `IN`, and its `truth_value` and `text_hash` must equal those stored in `snapshot_json`.
- For action `supersede`: same drift checks as `retract`, plus `new_id` must not already exist in the network.
- For action `nogood`: `snapshot_json` must contain `node_ids`, and all those node IDs must still exist.
- `action` must be one of `{"add", "retract", "supersede", "nogood"}`.

**Postconditions**
- On success: the network mutation is applied (via `add_node` / `retract_node` / `supersede_with_text` / `add_nogood`), and the row is updated to `status='accepted'`, with `result_json` holding the operational result. Returns `applied: True`.
- On drift: NO network mutation occurs; the row is updated to `status='stale'` with `result_json={"reason": ...}`. Returns `applied: False`.
- On loss of the accept-race: `status='stale'` with a "Concurrent accept race" reason; returns `applied: False`.
- On invalid input (unknown proposal / not pending / unknown action): raises.

**Invariants (asserted by code)**
- `status` can only transition from `pending` to `accepted`, `rejected`, `withdrawn`, or `stale`. `accept_proposal` only emits the first and third.
- The `UPDATE … WHERE status = 'pending'` at the end is the *only* serialization point between concurrent acceptors. Whoever flips the row first wins.

## Parameters

| name | type | meaning |
|---|---|---|
| `proposal_id` | `str` | Primary key of the proposal to accept. |
| `voter` | `str` (default `""`) | Free-form actor identifier recorded as `resolved_by` in the `proposals` row. Not authenticated/validated. |
| `db_path` | `str` (default `"reasons.db"`) | SQLite file holding both the network and the proposals table. Note: proposals live in **the same DB file**, not a separate store. |
| `pg_conninfo` | `str \| None` | If truthy, delegates to `_pg_dispatch(…, "accept_proposal", …)` and returns whatever the Postgres backend returns. The SQLite path is bypassed entirely. |
| `domain_id` | unknown (default `None`) | Postgres domain scoping only; unused on the SQLite path. |

Edge cases:
- `voter=""` is allowed; affects only bookkeeping.
- `db_path` must exist and contain a `proposals` table (`_proposals_db` runs `PROPOSALS_SCHEMA` idempotently, so a fresh DB gets one auto-created).
- If `pg_conninfo` is set, none of the SQLite-specific re-validation runs in this process — it is entirely on the PG backend.

## Return Value

Always a `dict` (JSON-serializable) on the SQLite path, unless an exception is raised.

| Situation | Shape |
|---|---|
| Applied successfully | `{"proposal_id", "applied": True, "status": "accepted", "action": <str>, "result": {...}}` where `result` is one of `{"add_result"/"retract_result"/"supersede_result"/"nogood_result": <dict>}` |
| Drift detected (any of the pre-checks failed) | `{"proposal_id", "applied": False, "status": "stale", "reason": <str>}` |
| Lost the accept race | `{"proposal_id", "applied": False, "status": "stale", "reason": "Concurrent accept race: proposal no longer pending"}` |

The caller **must** branch on `applied` / `status`, not on presence of an exception — stale is a normal outcome, not an error.

## Algorithm

1. **PG dispatch.** If `pg_conninfo` is set, forward to `PgApi.accept_proposal(proposal_id=…, voter=…)` and return.
2. **Read + validate (transaction 1).** Open `_proposals_db(db_path)`, `SELECT id, action, target_id, new_id, proposed_text, reason, snapshot_json, impact_json FROM proposals WHERE id = ? AND status = 'pending'`.
   - If no row: re-query `status` for the same ID to distinguish "not found" (`KeyError`) from "not pending" (`ValueError`), then close.
   - Otherwise unpack the row, parse `snapshot_json` into `snapshot`. Close connection.
3. **Compute `now`** as UTC ISO with second precision.
4. **Define `_mark_stale(reason_text)`** — a closure that opens a fresh connection, sets `status='stale'`, `updated_at`, `resolved_at`, `resolved_by=voter`, `result_json={"reason": ...}` (guarded by `status='pending'`), commits, closes, and returns the stale dict. This is the escape hatch used by every drift check.
5. **Branch on `action`:**
   - **`add`**: consistency check via `_with_network(write=False)` — if `target_id` is now in the network, `_mark_stale`. Otherwise pull `sl`, `unless`, `label`, `source`, `source_url` from `snapshot`; call `add_node(target_id, proposed_text, …)`. Capture in `result_data`.
   - **`retract`**: consistency check via `_with_network(write=False)` — `target_id` must exist, be `IN`, `truth_value` matches `snapshot["truth_value"]`, `text_hash` matches `snapshot["text_hash"]` (only compared if both sides are non-empty). Then call `retract_node(target_id, reason=reason)`.
   - **`supersede`**: consistency check as above **plus** `new_id not in net.nodes`. Then call `supersede_with_text(target_id, proposed_text, new_id=new_id or None)`.
   - **`nogood`**: pull `snapshot["node_ids"]`, require non-empty, verify all still exist, then call `add_nogood(nogood_node_ids)`.
   - **else**: `raise ValueError(f"Unknown action '{action}'")`.
6. **Finalize (transaction 2).** Open a **new** `_proposals_db` connection and `UPDATE proposals SET status='accepted', updated_at, resolved_at, resolved_by=voter, result_json=? WHERE id=? AND status='pending'`.
   - If `rowcount == 0`, another acceptor already flipped the row — return the "Concurrent accept race" stale dict. **Note: the network mutation from step 5 has already succeeded here; this only prevents double-marking.**
7. Return the accepted dict.

## Side Effects

- **Proposals table mutations**: flips `status`, timestamps, `resolved_by`, `result_json`. Emits an `ALTER TABLE … ADD COLUMN tags_json` if the schema is behind (`_proposals_db`).
- **Network mutations** (per action) via `add_node` / `retract_node` / `supersede_with_text` / `add_nogood`, all of which use `_with_network(db_path, write=True)` — meaning they:
  - Invalidate/update the `_network_cache`.
  - **Publish pubsub events** (`pubsub.compute_changes` + `pubsub.publish`) to subscribers keyed by `db_path` when truth values change.
- **Concurrency**: `_proposals_db` sets `PRAGMA journal_mode=WAL` and `PRAGMA foreign_keys=ON`. WAL allows concurrent readers while a writer holds the write lock; the "atomicity" is really "applied under a WAL transaction, but not under the *same* transaction as the proposals update".
- **No filesystem writes** other than to the SQLite DB(s) at `db_path`.

## Error Handling

| Exception | When |
|---|---|
| `KeyError` | `proposal_id` not in `proposals` (from the SQLite path). |
| `ValueError` | Proposal exists but status ≠ `'pending'`; or `action` is unknown. |
| `KeyError`/`ValueError` | May propagate from `add_node`, `retract_node`, `supersede_with_text`, `add_nogood` when *they* validate inputs against the live network (e.g., target vanished between the drift check and the mutation — a race). |
| `json.JSONDecodeError` | Malformed `snapshot_json` (should not happen; not caught). |

**Errors returned, not raised**: all drift detections. Notably, `_mark_stale` swallows concurrency-related UPDATE failures (rowcount is ignored inside the closure — it always returns the stale dict). So a `_mark_stale` on a row that a concurrent acceptor also marked stale is indistinguishable from the "we marked it" case.

**Errors not caught**: an exception raised by the network mutation (step 5) will skip step 6 entirely and leave the proposal in `pending` even though the change may have been partially applied — a real failure mode to be aware of.

## Usage Patterns

Typical caller is CLI/MCP/server glue that:

```python
from reasons.api import list_proposals, show_proposal, accept_proposal

pending = list_proposals(status="pending", db_path=db_path)["proposals"]
for p in pending:
    detail = show_proposal(p["id"], db_path=db_path)  # for human/LLM review
    if reviewer_wants_to_apply(detail):
        outcome = accept_proposal(p["id"], voter="alice", db_path=db_path)
        if not outcome["applied"]:
            log.warning("stale: %s", outcome["reason"])
```

Caller obligations:
- **Always** inspect `applied` / `status` — stale is normal.
- Do not assume "no exception = applied". The race path returns `applied: False`.
- If you care about strict single-transaction guarantees, wrap the accept in your own DB lock or use a dedicated "one acceptor per DB" policy — the code does not enforce one.
- `voter` should be the identity of the actor that authorized the change; it lands in `resolved_by` for auditing.

## Dependencies

- `_proposals_db`, `_pg_dispatch`, `_with_network` (from same module).
- `PROPOSALS_SCHEMA` (`reasons.storage`) — used by `_proposals_db` to ensure the `proposals` table exists.
- `Storage`, `Network` — indirectly, through `_with_network`.
- `pubsub` — indirectly, when truth values change.
- `reasons.pg.PgApi` — only when `pg_conninfo` is set.
- Network operation helpers: `add_node`, `retract_node`, `supersede_with_text`, `add_nogood` (all in this same module).
- `datetime.timezone.utc`, `json`.

## Assumptions Not Enforced by Types

1. **Docstring claims "single write transaction"** — but the code does the read, the network mutation, and the status update as **three separate connections/transactions**. Between them another acceptor or another mutator can interpose. The only serialization guard is `UPDATE … WHERE status='pending'` at the end.
2. **Snapshot shape is action-specific and untyped.** `add` expects `{sl, unless, label, source, source_url}` in `snapshot`; `retract`/`supersede` expect `{truth_value, text_hash}`; `nogood` expects `{node_ids, truth_values}`. If a future action is added or a snapshot key is renamed, the failure is a `KeyError`/silent emptiness, not a type error.
3. **`snapshot["text_hash"]` may be empty.** Drift check is `if snapshot.get("text_hash") and node.text_hash and …` — an empty or missing hash effectively disables the text-change check.
4. **The race detection is post-hoc.** If the network mutation in step 5 succeeds but the `UPDATE … status='pending'` returns 0, the truth is already changed. In that case, `accept_proposal` reports `applied: False` while the DB has the change — the caller's view and the network's view could disagree.
5. **`impact_json` is selected but unused.** Not a bug, just evidence it was meant for display only.
6. **`new_id` for supersede is taken from the row, not recomputed.** If the proposal's `new_id` collides with a node that appeared after propose-time, the drift check catches it — but only for `supersede`, and only if `new_id` is truthy (empty `new_id` for `supersede` was allowed by `propose_supersession`, which auto-generates).
7. **`voter` is unauthenticated metadata.** No authorization check that `voter` is allowed to accept. Trust boundary is entirely in the caller / transport (CLI, MCP, HTTP).
8. **PG path bypasses all of the above validation**. `_pg_dispatch` sends only `proposal_id` and `voter`; the Postgres backend must reimplement consistency checks for parity.

---

## Topics to Explore

- [function] `reasons/api.py:_pg_dispatch` — The dispatcher used here; understanding it clarifies what guarantees are lost when `pg_conninfo` is set and how the Postgres backend must mirror this logic.
- [function] `reasons/api.py:propose_retraction` — The complement of `accept`; shows exactly what goes into `snapshot_json` and `impact_json` for the `retract` branch (and by symmetry the other `propose_*` helpers).
- [function] `reasons/api.py:_with_network` — Explains the "read vs write" cache semantics, why `write=True` disables the cache, and how pubsub events are emitted after writes.
- [file] `reasons/storage.py` — Holds `PROPOSALS_SCHEMA` and defines the on-disk shape of the `proposals` table; the real schema authority for status transitions and `result_json`.
- [general] `proposal-concurrency-model` — The accept race between two operators on the same `db_path` is only mitigated by a single guarded `UPDATE`. Worth a dedicated study to decide whether to add an advisory lock, a proposals-level lease, or a stricter single-writer story.

## Beliefs

- `accept-proposal-revalidates-against-live-network` — Every action re-reads the network and compares against the stored `snapshot_json` before mutating; drift short-circuits to `status='stale'`.
- `accept-proposal-returns-stale-not-raises-on-drift` — Drift is reported as `{"applied": False, "status": "stale"}` rather than as an exception; callers must branch on `applied`.
- `accept-proposal-is-not-a-single-transaction` — The read, the network mutation, and the proposals-row update happen in three separate SQLite connections; the docstring's "single write transaction" claim is aspirational.
- `accept-proposal-requires-proposal-status-pending` — The initial `SELECT … WHERE status='pending'` is the sole precondition checked; already-accepted/rejected/withdrawn proposals raise `ValueError`.
- `accept-proposal-race-detection-is-post-hoc` — A losing acceptor only learns it lost after the network mutation may have already been applied; the `UPDATE … WHERE status='pending'` returning `rowcount=0` is the only signal.
- `accept-proposal-pg-dispatch-skips-sqlite-validation` — When `pg_conninfo` is set, none of the SQLite-side drift checks run in this process; the Postgres backend must replicate them.
