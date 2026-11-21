---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Trace how a proposal transitions `pending → accepted/rejected/withdrawn/stale`; check whether the storage layer enforces transitions or the server does.

# Proposal status lifecycle

## What the schema defines

The only proposal-status definition visible is `PROPOSALS_SCHEMA` in `reasons/storage.py`:

```sql
status TEXT NOT NULL DEFAULT 'pending'
       CHECK (status IN ('pending','accepted','rejected','withdrawn','stale'))
```

So the storage layer enforces **allowed status values** at the database-row level:

- New proposal rows default to `pending`.
- A direct `INSERT`/`UPDATE` can set `status` to any of the five values.
- Any other status value is rejected by SQLite’s `CHECK` constraint.

The schema also provides audit/context fields:

- `resolved_at`
- `resolved_by`
- `result_json`
- `snapshot_json`
- `impact_json`
- `tags_json`

These fields are consistent with tracking resolution, but the observed code does not show who writes them.

Indexes:

- `idx_proposals_target ON proposals (target_id, status)`
- `idx_proposals_status ON proposals (status)`

These support queries such as “pending proposals for this target” or “all accepted proposals,” but they are not transition enforcement.

## What the storage layer does **not** enforce

The storage layer does **not** appear to enforce a transition state machine.

Evidence:

1. `PROPOSALS_SCHEMA` contains only a value constraint:
   - It allows `pending`, `accepted`, `rejected`, `withdrawn`, and `stale`.
   - It does not encode rules like “only `pending` may transition to `accepted`.”
   - There is no trigger, no transition table, and no SQL expression comparing old and new status.

2. `Storage.save()` in `reasons/storage.py` persists the `Network`:
   - It rewrites `nodes`, `justifications`, `nogoods`, `network_meta`, `repos`, and `propagation_log`.
   - It does **not** insert, update, or delete rows in the `proposals` table.
   - Therefore normal network saves do not drive proposal status transitions.

3. `PROPOSALS_SCHEMA` is defined in `reasons/storage.py`, but `Storage._init_schema()` executes only `SCHEMA`, not `PROPOSALS_SCHEMA`. `api.py` imports `PROPOSALS_SCHEMA` from `storage`, so the proposal table initialization likely happens elsewhere in `api.py`, outside the observed 600-line excerpt.

So: **storage enforces the status vocabulary, but not the transition legality.**

## What the server/API layer does

The server/API layer is where transition enforcement would have to live. However, the observations do not include the actual proposal lifecycle code.

`reasons/api.py` imports `PROPOSALS_SCHEMA` from `reasons/storage.py`, but the first 600 lines do not show proposal creation, acceptance, rejection, withdrawal, or stale-marking functions.

The explicit search for proposal status transitions returned no matches:

```json
"search_proposal_status_transitions": {
  "matches": [],
  "total_count": 0
}
```

That result conflicts with the fact that `PROPOSALS_SCHEMA` contains the status literals, so the search observation is likely incomplete or scoped incorrectly. Regardless, the visible API excerpt does not contain the transition code.

The related LLM proposal code is not the same as the `proposals` table:

- `reasons/propose_update.py` generates LLM proposals with `action` values `update` or `retract`.
- The `proposals` table `action` constraint allows `retract`, `supersede`, `add`, and `nogood` — not `update`.
- `propose_update.py` formats proposals as markdown and tells the user to run commands; it does not set `proposals.status`.

`reasons/check_stale.py` detects stale **source files/nodes** by comparing hashes and git history. It does not mark proposal rows as `stale`.

## Trace conclusion

Based on the supplied observations:

- A proposal row starts as `pending` because of the column default.
- The database permits the row to become `accepted`, `rejected`, `withdrawn`, or `stale`, because the `CHECK` constraint allows those values.
- The database does **not** enforce legal transitions between those values.
- The server/API layer must contain the transition logic, but the actual functions are not present in the observed excerpts.

Missing to complete the trace:

- Proposal CRUD/lifecycle functions in `reasons/api.py`, likely beyond the first 600 lines.
- The SQL `UPDATE proposals SET status = ... WHERE ...` statements.
- Any server-side validation such as “only `pending` can be accepted/rejected/withdrawn.”
- Where and when `resolved_at`, `resolved_by`, and `result_json` are populated.
- Where `stale` proposals are detected and marked.
- Tests covering proposal state transitions.

Because `reasons/api.py` is 5,886 lines and only the first 600 were returned, the actual transition implementation is very likely in the unobserved portion.

## Topics to Explore

- [file] `reasons/api.py` — Review beyond the first 600 lines for proposal lifecycle endpoints and status-update SQL.
- [function] `reasons/api.py:accept_proposal` — Locate or confirm the server-side function that transitions a proposal from `pending` to `accepted`.
- [function] `reasons/storage.py:Storage._init_schema` — Confirm whether `PROPOSALS_SCHEMA` is executed here or elsewhere in the API layer.
- [general] `proposal-state-machine-tests` — Search tests for assertions covering `pending → accepted`, `pending → rejected`, `pending → withdrawn`, and `pending → stale`.
- [file] `reasons/pg.py` — Check whether the PostgreSQL backend has a proposals table or equivalent proposal lifecycle; the observed PostgreSQL schema does not define one.

## Beliefs

- `proposals-table-defaults-to-pending` — The `proposals.status` column defaults to `'pending'` in `PROPOSALS_SCHEMA`.
- `storage-schema-enforces-status-values-not-transitions` — The SQLite schema constrains `proposals.status` to five literals but contains no trigger or transition constraint.
- `storage-save-does-not-persist-proposals` — `Storage.save()` rewrites network tables but does not insert, update, or delete proposal rows.
- `propose-update-output-is-not-proposal-row-status` — `propose_update.py` returns LLM proposal dictionaries and markdown commands, not `proposals.status` mutations.
- `postgres-schema-lacks-proposals-table` — The observed PostgreSQL `SCHEMA` in `reasons/pg.py` defines nodes, justifications, nogoods, propagation log, and metadata but no proposals table.
