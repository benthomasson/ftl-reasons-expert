---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _stale_older_pending in reasons/api.py

# `_stale_older_pending` — `reasons/api.py`

## Purpose

`_stale_older_pending` is a private helper used **inside the proposal-creation flow**. When a new pending proposal is created for a given `(target_id, action)` pair, any *existing* pending proposals for that same pair are considered superseded and are transitioned to `'stale'` before the new proposal is inserted.

This exists to keep the proposal queue unambiguous: acceptance logic (`accept_proposal`) is written assuming the proposal being accepted is the current one for that target/action. Older pending ideas would otherwise linger and could be accepted later against a network that has drifted past their snapshot — a source of user confusion. Staling them proactively keeps the "pending" state meaningful.

## Contract

**Preconditions (not type-enforced):**
- `conn` is an open `sqlite3.Connection` to a database where the `proposals` table already exists (guaranteed by `_proposals_db()`, which executes `PROPOSALS_SCHEMA` and applies the `tags_json` migration).
- The caller owns the transaction lifecycle — this function does **not** commit.
- `now` is a valid timestamp string (callers pass `datetime.now(timezone.utc).isoformat(timespec="seconds")`).

**Postconditions:**
- Every row with `target_id = ?` AND `action = ?` AND `status = 'pending'` has `status` set to `'stale'`, and `updated_at` / `resolved_at` set to `now`.
- The equivalent row set is returned as a list of proposal IDs.

**Invariants preserved:**
- `resolved_at` is only ever populated when a proposal leaves the `'pending'` state. This function follows that convention — matching `accept_proposal`, `reject_proposal`, and `withdraw_proposal`.
- The caller's insert of the new proposal happens **after** this call, inside the same uncommitted transaction, so staling + insertion is atomic when `conn.commit()` finally runs.

**Not enforced by type or schema:**
- `action` is assumed to be one of `'retract'`, `'supersede'`, `'add'`, `'nogood'`. An unknown action silently matches zero rows — no error is raised.
- `target_id` for `'nogood'` is a synthetic string produced by `"-".join(sorted(node_ids))` in `propose_nogood`, not a real node ID.
- The function does **not** filter on `proposer` — proposals from any author are staled.

## Parameters

| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `conn` | `sqlite3.Connection` | Open DB connection bound to the proposals table. Must be usable (not closed). | Raises `sqlite3.ProgrammingError` if closed; `sqlite3.OperationalError` if table missing or DB locked. |
| `target_id` | `str` | The node / proposal target. Matched exactly (`=`, not `LIKE`). | Empty string is valid SQL but produces no match. |
| `action` | `str` | The action kind. Must match a value stored in the `action` column. | Unknown values silently select nothing. |
| `now` | `str` | Timestamp written to both `updated_at` and `resolved_at` on staled rows. | Not validated — passing `""` stores an empty string. |

## Return Value

`list[str]` — the IDs (primary keys) of all proposals transitioned to `'stale'`.

- Returns `[]` when no pending proposals exist for that `(target_id, action)` pair.
- Returns multiple IDs if the caller or another writer previously inserted more than one pending proposal for the same pair (normal flow serialises via `_next_proposal_id`, but concurrent inserts or re-created proposals can stack).
- The caller normally exposes this list to callers of `propose_*` as the `"staled"` field in their returned dict, so it is user-facing.

## Algorithm

1. **Select candidates first:** `SELECT id FROM proposals WHERE target_id = ? AND action = ? AND status = 'pending'` — fully materialised via `cursor.fetchall()`.
2. **Iterate:** For each `(id,)` in the fetched rows:
   - `UPDATE proposals SET status = 'stale', updated_at = ?, resolved_at = ? WHERE id = ?` — note the WHERE is by `id` only. The `status = 'pending'` guard is not re-checked inside the loop, so a concurrent transition is not detected; it just gets overwritten with `'stale'`.
   - Append the ID to `staled`.
3. **Return** the accumulated list. No commit.

Two structural observations:
- The `SELECT … fetchall()` is consumed before the loop begins, so the `UPDATE`s do not affect an open cursor.
- Each row is a separate `UPDATE` — `N+1` round-trips, not a bulk `UPDATE … WHERE target_id = ? AND action = ? AND status = 'pending'`. This is why the function needs the loop and can return IDs; it's also why it's not atomic per-row from the caller's perspective if the caller does not hold a transaction.

## Side Effects

- **Database mutation:** Multiple `UPDATE` statements against the `proposals` table (`status`, `updated_at`, `resolved_at`).
- **No commit:** Callers wrap this in a `try/finally` that ends with `conn.commit()` (on success) or `conn.close()` (which implicitly rolls back the open transaction).
- **No other I/O:** No logging, no file handles, no network.

## Error Handling

- **No internal try/except** — all errors propagate.
- `sqlite3.OperationalError` if the `proposals` table is absent (should not happen via `_proposals_db`).
- `sqlite3.DatabaseError` / `sqlite3.OperationalError` for lock contention when multiple writers are active against the WAL DB.
- `sqlite3.ProgrammingError` if `conn` is closed.
- **Silent no-op** for unknown `action` values or unmatched `target_id` — this is the main thing to be aware of: mistakes in the `action` string are not detected by the function.

## Usage Patterns

Called by all four `propose_*` entry points (`propose_retraction`, `propose_supersession`, `propose_addition`, `propose_nogood`) inside the same transaction as the insert:

```python
now = datetime.now(timezone.utc).isoformat(timespec="seconds")
conn = _proposals_db(db_path)
try:
    staled = _stale_older_pending(conn, target_id, "retract", now)
    proposal_id = _next_proposal_id(conn, target_id, "retract")
    conn.execute("INSERT INTO proposals ...", (...))
    conn.commit()
finally:
    conn.close()
```

**Caller obligations:**

1. **Must commit** after this returns — otherwise the `'stale'` transitions are rolled back when `conn.close()` runs.
2. **Must call before `_next_proposal_id`** so the monotonic ID counter (which counts *all* proposals for the target/action, including newly-staled ones) does not collide with the new ID.
3. **Must not close `conn`** before returning from the enclosing block, and must not interleave other writers without holding the transaction.

The `"staled"` list is surfaced verbatim in every `propose_*` return value, so downstream callers (CLI, LangGraph tools) can report which older proposals were invalidated.

## Dependencies

- **`sqlite3`** (Python standard library) — connection, cursor, and execute API.
- **`_proposals_db`** (same module) — must have been called to produce the connection and ensure the schema exists.
- **`_next_proposal_id`** (same module) — invoked *after* this function by every caller; relies on staleness not removing rows so ID numbering stays monotonic.
- **`PROPOSALS_SCHEMA`** (imported from `reasons.storage`) — defines the columns used here (`id`, `target_id`, `action`, `status`, `updated_at`, `resolved_at`).
- **`datetime` / `timezone`** (imported at module level) — used by *callers* to build `now`, not by this function itself.

---

## Topics to Explore

- [file] `reasons/storage.py` — Contains `PROPOSALS_SCHEMA`; the exact column set and constraints for `proposals` that this function depends on (status enum, NOT NULL on `resolved_at`, etc.).
- [function] `reasons/api.py:_next_proposal_id` — Determines the new proposal ID; note it counts *all* proposals for `(target, action)` regardless of status, so staling does not recycle IDs.
- [function] `reasons/api.py:propose_supersession` — Representative caller; shows the full stale-then-insert transaction wrap.
- [function] `reasons/api.py:accept_proposal` — Uses the same `'stale'` status + `resolved_at` convention for drift-detection, illustrating the terminal-state semantics this function participates in.
- [general] `proposal-lifecycle` — The full pending → accepted / rejected / withdrawn / stale state machine and how each terminal state is recorded (status, `resolved_at`, `resolved_by`, `result_json`).

## Beliefs

- `_stale_older_pending-does-not-commit` — This function mutates `proposals` rows but never calls `conn.commit()`; persistence depends on the caller.
- `_stale_older_pending-filters-by-target-action-and-status` — Only rows with matching `target_id`, `action`, and `status = 'pending'` are updated to `'stale'`.
- `_stale_older_pending-sets-resolved-at` — For every staled row, `resolved_at` is set to the supplied `now`, matching the convention used by `accept_proposal`, `reject_proposal`, and `withdraw_proposal`.
- `_stale_older_pending-ignores-proposer` — The query does not filter by `proposer`, so pending proposals from any author for the same `(target, action)` are staled.
- `_stale_older_pending-runs-before-next-id` — Callers invoke it immediately before `_next_proposal_id`, so the monotonic ID counter still counts the just-staled rows.
