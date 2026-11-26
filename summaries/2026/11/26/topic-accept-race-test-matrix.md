---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Need concurrency tests for two operators accepting the same proposal on the same `db_path`, including expected winner/loser behavior and side-effect duplication checks.

## Why concurrency tests are needed

`accept_proposal` is a state-transition plus side-effect operation, not just a read. The SQLite path checks proposal state around `reasons/api.py:5645-5656`, can mark a proposal stale on drift (`reasons/api.py:5633`), applies the proposal’s effect, records resolution fields, and may publish pubsub events after a write (`reasons/api.py:139-158`). Two operators calling `accept_proposal` on the same `db_path` and same `proposal_id` can race between “read pending” and “write accepted.”

Without an atomic claim such as `UPDATE ... WHERE status='pending'` with a row-count check, both callers can observe `pending`, both can apply side effects, and the loser can overwrite `resolved_by`, `resolved_at`, or `result_json`. The tests should prove exactly one winner, one deterministic loser, and exactly one set of side effects.

## What the observed code tells us

- `reasons/api.py:5622` defines `accept_proposal`.
- `reasons/api.py:5636` shows a PostgreSQL dispatch path via `_pg_dispatch`; the same `db_path` SQLite path is the relevant target for these tests.
- `reasons/api.py:5645-5656` show proposal lookup/status handling. The non-pending case builds the message at `reasons/api.py:5654`: `Proposal '{proposal_id}' is '{existing[0]}', not pending`; missing proposals raise `KeyError` at `reasons/api.py:5656`.
- `reasons/api.py:5633` documents drift behavior: “On drift, marks it stale without applying.”
- `reasons/storage.py` defines `PROPOSALS_SCHEMA` with `id` primary key, `status` check constraint, and `resolved_at`, `resolved_by`, `result_json`. There is no separate uniqueness constraint for “accepted”; correct single acceptance must be enforced by accept logic.
- `Storage.save` in `reasons/storage.py` clears and rewrites many tables, including `propagation_log`, inside a transaction. That makes it important to compare final row counts and log entries after the race, not just return values.
- Existing accept tests are sequential: `tests/test_api.py:1177-1267`, `tests/test_cli.py:1408-1419`, and `tests/test_mcp_server.py:513-546`. The provided observations do not show any concurrent accept test.
- `tests/conftest.py` has a `pg_api` fixture for PostgreSQL with a unique `domain_id`; that is not the same as two operators racing on one SQLite `db_path`.

## Expected winner/loser behavior to assert

For a normal pending proposal:

- **Winner**: the first transaction that atomically commits the `pending -> accepted` transition. It returns the accepted result, applies the proposal effect, records its `voter` in `resolved_by`, sets `resolved_at`/`result_json`, and publishes any enabled pubsub event.
- **Loser**: must observe the already-resolved proposal and surface the same non-pending contract represented by `reasons/api.py:5654`—for example a conflict/error saying the proposal is `accepted`, not `pending`. The exact exception class is not visible in the observations. The loser must not apply the proposal and must not overwrite `resolved_by`, `resolved_at`, or `result_json`.
- If the proposal is missing, the loser/winner should follow the `KeyError` path at `reasons/api.py:5656`.
- If the first acceptance detects drift, it should mark the proposal `stale` without applying side effects (`reasons/api.py:5633`). The second caller should then see `stale`, not apply, and not overwrite the stale resolution.
- A SQLite lock/`database is locked` error should not be the normal loser outcome unless the API contract explicitly says so. The test should distinguish “loser sees non-pending” from “API leaked a raw SQLite lock error.”

## Side-effect duplication checks

The concurrency test should snapshot state before the race and compare after:

- Proposal row: exactly one final `status='accepted'`; `resolved_by` equals the winner’s voter; `resolved_at` is set once; `result_json` corresponds to the winner, not the loser.
- Nodes/justifications/nogoods: exactly one acceptance’s worth of mutation. For `retract`, target state changes once; for `add`, the new node appears once; for `supersede`, old/new node effects happen once; for `nogood`, one nogood is added.
- `propagation_log`: exactly the expected delta. Because `Storage.save` clears and rewrites `propagation_log`, count rows before/after and assert no duplicate cascade entries or lost entries.
- Pubsub: if `pubsub.has_subscribers(db_path)` is true, `pubsub.publish` should be called only for the winner, not the loser (`reasons/api.py:139-158`).
- Idempotency: after the race, a subsequent sequential `accept_proposal` on the same proposal should fail as non-pending and create no additional side effects.

A practical test shape is: create a pending proposal, use a `threading.Barrier(2)` with two workers that each open their own API/Storage connection to the same `db_path`, call `accept_proposal(proposal_id, voter=...)`, then assert exactly one success and one non-pending conflict. For real two-operator coverage, also run two `reasons accept-proposal` subprocesses using the CLI path at `reasons/cli.py:1994-1996` and parser mapping at `reasons/cli.py:3957`.

## What is missing from the observations

The exact loser exception type, transaction boundaries, and whether acceptance uses an atomic status-guarded `UPDATE` are not visible. We need the full body of `reasons/api.py:accept_proposal` around `5622-5680`, plus the exact assertions in `tests/test_api.py:1177-1267`. Without those, we can define the required behavior, but not guarantee the current implementation’s exact loser return/raise contract.

## Topics to Explore

- [function] `reasons/api.py:accept_proposal` — Read the full body around lines 5622-5680 to confirm transaction boundaries, status guard, row-count checks, and loser exception type.
- [file] `tests/test_api.py` — Review sequential accept tests around lines 1177-1267 for the existing non-pending, stale, and missing-proposal contracts.
- [file] `reasons/storage.py` — Understand how `Storage.save` clears/rewrites `propagation_log` and other tables, which affects duplicate-side-effect assertions.
- [general] `sqlite-wal-write-serialization` — Decide whether `Storage` needs explicit `busy_timeout`/retry behavior so the loser gets a non-pending conflict instead of `database is locked`.
- [general] `pubsub-side-effect-counting` — Verify publish events are emitted only by the winner when subscribers exist.

## Beliefs

- `accept-proposal-has-status-check` — `accept_proposal` reads proposal status and builds a non-pending error at `reasons/api.py:5645-5654`; missing proposals raise `KeyError` at `reasons/api.py:5656`.
- `proposals-status-has-no-dedicated-unique-constraint` — `PROPOSALS_SCHEMA` in `reasons/storage.py` uses `id` as primary key and constrains `status` values, but does not add a separate unique constraint for “accepted”; single acceptance must be enforced by accept logic.
- `storage-save-rewrites-propagation-log` — `Storage.save` deletes and reinserts `propagation_log` and other tables inside `with self.conn`, so concurrency tests must compare final row counts and log deltas.
- `accept-proposal-can-publish-pubsub-events` — `reasons/api.py:139-158` publishes events after a write when subscribers exist; the loser should not trigger publication.
- `observed-accept-tests-are-sequential` — Existing accept tests at `tests/test_api.py:1177-1267`, `tests/test_cli.py:1408-1419`, and `tests/test_mcp_server.py:513-546` exercise single-caller acceptance; no concurrent same-proposal test appears in the provided observations.
