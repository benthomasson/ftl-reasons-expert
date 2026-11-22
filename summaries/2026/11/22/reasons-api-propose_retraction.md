---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: propose_retraction in reasons/api.py

## Purpose

`propose_retraction` creates a **pending proposal** to retract an existing belief without changing the belief network’s truth values. It is part of the proposal workflow: a caller records an intended retraction, the system stores a snapshot and simulated impact, and later `accept_proposal`, `reject_proposal`, or `withdraw_proposal` handles the outcome.

It exists so agents/humans can suggest retractions safely and asynchronously. The actual retraction only happens when a proposal is accepted.

## Contract

### Preconditions

On the SQLite path:

- `target_id` must exist in the network loaded from `db_path`.
- `target_id` must currently have truth value `"IN"`.
- `tags`, if provided, should be a list of strings.

On the PostgreSQL path, this function delegates to `PgApi.propose_retraction`; equivalent validation is assumed to occur inside the PG backend.

### Postconditions

On success, SQLite path:

- A new `proposals` row exists with:
  - `action = "retract"`
  - `status = "pending"`
  - `target_id = target_id`
  - a JSON snapshot containing `truth_value` and `text_hash`
  - JSON `impact` computed by `what_if_retract`
  - JSON tags list
- Any existing pending proposal rows for the same `target_id` and `action = "retract"` are marked `"stale"`.
- The network truth values are unchanged.
- No propagation, retraction, or node mutation is saved.

### Important invariants / assumptions not enforced by types

- The code assumes truth values are `"IN"` or `"OUT"`.
- The code assumes `tags` is a list of strings. If a plain string is passed, `sorted(tags)` will sort its characters.
- `tags` are sorted but not deduplicated.
- `basis`, `failure_mode`, `evidence`, `reason`, and `proposer` are stored as opaque strings; no enum validation is performed.
- The “only one pending retraction per target” behavior is procedural, not shown as a DB-level unique constraint. Concurrent callers may still race.
- The snapshot and impact are not computed from one atomic read. Another process could modify the network between the initial validation and `what_if_retract`.
- The stored impact is advisory. `accept_proposal` revalidates the target and applies the retraction; the actual impact can differ if the network changed after proposal creation.
- On PostgreSQL, the `tags` argument is not passed to `_pg_dispatch`, so tags are effectively ignored by this wrapper on the PG path.

## Parameters

- `target_id: str`  
  Node ID to retract. Must exist and be `"IN"` on the SQLite path.

- `reason: str = ""`  
  Human-readable explanation stored with the proposal.

- `failure_mode: str = ""`  
  Optional classification of why the belief fails. Stored unchanged.

- `basis: str = "prior-knowledge"`  
  Basis for the proposed retraction. Stored unchanged; no allowed-value check.

- `evidence: str = ""`  
  Supporting evidence or rationale. Stored unchanged.

- `proposer: str = ""`  
  Identity of the proposer. Stored unchanged.

- `tags: list[str] | None = None`  
  Optional tags stored as JSON on the proposal. On SQLite, stored as `json.dumps(sorted(tags) if tags else [])`. On PostgreSQL, not forwarded.

- `db_path: str = DEFAULT_DB`  
  SQLite database path used both to load the network and to write proposal rows.

- `pg_conninfo=None`  
  If truthy, the function dispatches to the PostgreSQL backend instead of using SQLite locally.

- `domain_id=None`  
  Domain identifier passed to the PostgreSQL backend when `pg_conninfo` is used.

## Return Value

On the SQLite path, returns a dict like:

```python
{
    "proposal_id": "prop-<target_id>-retract-1",
    "target_id": target_id,
    "action": "retract",
    "status": "pending",
    "impact": {...},
    "staled": ["prop-..."],
}
```

- `proposal_id`: generated as `prop-{target_id}-retract-{count+1}`, where `count` is the number of existing proposals for the same target/action.
- `target_id`: the node proposed for retraction.
- `action`: always `"retract"` on the SQLite path.
- `status`: always `"pending"` for a newly inserted proposal.
- `impact`: result of `what_if_retract`, containing simulated downstream effects such as `retracted`, `restored`, and `total_affected`.
- `staled`: list of older pending retraction proposal IDs that were marked stale by this call.

On the PostgreSQL path, the function returns whatever `PgApi.propose_retraction` returns; this wrapper does not post-process it.

## Algorithm

1. If `pg_conninfo` is set:
   - Dispatch to `PgApi.propose_retraction` with `target_id`, `reason`, `failure_mode`, `basis`, `evidence`, and `proposer`.
   - Return immediately. `tags` and `db_path` are not part of this dispatch.

2. Otherwise, open the network read-only with `_with_network(db_path, write=False)`.

3. Validate the target:
   - If `target_id` is not in `net.nodes`, raise `KeyError`.
   - If the node’s truth value is `"OUT"`, raise `ValueError`.
   - Capture a snapshot:
     ```python
     {
         "truth_value": node.truth_value,
         "text_hash": node.text_hash,
     }
     ```

4. Compute simulated impact:
   - Call `what_if_retract(target_id, db_path=db_path)`.
   - This loads the network, deep-copies it, performs the retraction in memory, and reports which nodes would go OUT or be restored. It does not save changes.

5. Create a timestamp in UTC ISO format with second precision.

6. Open the proposals SQLite DB via `_proposals_db(db_path)`:
   - Sets `PRAGMA journal_mode=WAL`.
   - Sets `PRAGMA foreign_keys=ON`.
   - Runs `PROPOSALS_SCHEMA` to ensure the proposals table exists.
   - Adds a `tags_json` column if migrating an older schema.

7. Within that connection:
   - Call `_stale_older_pending(conn, target_id, "retract", now)`.
     - Finds all pending proposals for the same target and action.
     - Updates them to `status = "stale"`, sets `updated_at` and `resolved_at`, and returns their IDs.
   - Generate the next proposal ID with `_next_proposal_id`.
   - Insert a new row with action `"retract"`, status `"pending"`, the snapshot JSON, impact JSON, tags JSON, and timestamps.
   - Commit.

8. Close the connection and return the proposal details.

## Side Effects

- Reads the belief network from `db_path` and may populate/update the in-memory `_network_cache`.
- Calls `what_if_retract`, which deep-copies the network in memory; no graph changes are saved.
- Opens or creates the proposals table/schema in the SQLite database.
- May alter the proposals table by adding `tags_json` if missing.
- Sets SQLite journal mode to WAL on the proposals connection.
- Updates older pending retraction proposals to `"stale"`.
- Inserts a new pending proposal row.
- Does not call `retract_node`, `propagate`, or `recompute_all`, and does not modify node truth values.

## Error Handling

SQLite path:

- `KeyError` if `target_id` does not exist.
- `ValueError` if `target_id` is already `"OUT"`.
- SQLite errors, JSON serialization errors, or `TypeError` from malformed `tags` propagate.
- The function does not catch and convert exceptions into error dicts.

PostgreSQL path:

- Any exception from `PgApi.propose_retraction` propagates.
- No local validation or local proposal-table side effects occur.

Callers should handle at least `KeyError` and `ValueError` when using the SQLite backend.

## Usage Patterns

Typical usage:

```python
result = propose_retraction(
    "belief-x",
    reason="Contradicts newer evidence",
    failure_mode="invalid-inference",
    basis="prior-knowledge",
    evidence="See source section 3",
    proposer="agent-a",
    tags=["review-needed"],
)

proposal_id = result["proposal_id"]
# Later:
# accept_proposal(proposal_id)
# reject_proposal(proposal_id)
# withdraw_proposal(proposal_id)
```

Callers should:

- Inspect `impact` before deciding whether to accept.
- Expect `staled` to contain older pending retraction proposals for the same target that were automatically superseded by this new proposal.
- Not expect the belief to go OUT until `accept_proposal` is called.
- Avoid relying on `tags` being stored when using PostgreSQL, because this wrapper does not forward them.

## Dependencies

- `_with_network` for read-only network loading and cache handling.
- `what_if_retract` for simulated retraction impact.
- `_proposals_db` for opening/migrating the proposals SQLite table.
- `_stale_older_pending` for invalidating older pending retraction proposals.
- `_next_proposal_id` for generating proposal IDs.
- `_pg_dispatch` and `PgApi` for PostgreSQL delegation.
- `PROPOSALS_SCHEMA` from `reasons.storage`.
- Standard library: `datetime`, `json`, `sqlite3`.

## Topics to Explore

- [function] `reasons/api.py:accept_proposal` — Shows how pending retractions are revalidated and actually applied.
- [function] `reasons/api.py:what_if_retract` — Computes the stored `impact` by simulating retraction on a deep copy of the network.
- [file] `reasons/storage.py` — Contains `PROPOSALS_SCHEMA` and the persistence details for proposal rows.
- [function] `reasons/api.py:_proposals_db` — Handles SQLite connection setup, WAL mode, schema creation, and `tags_json` migration.
- [general] `postgres-proposal-parity` — Investigate whether `PgApi.propose_retraction` supports tags and matches SQLite validation/staling semantics.

## Beliefs

- `propose-retraction-does-not-mutate-truth-values` — On the SQLite path it only writes/stales proposal rows; graph truth values are not changed.
- `propose-retraction-requires-target-in` — In the SQLite path, missing targets raise `KeyError` and already-OUT targets raise `ValueError`.
- `propose-retraction-auto-stales-pending-retractions` — Before inserting, it marks existing pending proposals for the same target and `action="retract"` as stale.
- `propose-retraction-stores-snapshot-and-impact-json` — The proposal row stores a JSON snapshot of `truth_value`/`text_hash` and the `what_if_retract` impact.
- `postgres-propose-retraction-ignores-tags` — The `pg_conninfo` branch delegates without passing `tags`, so tags are not persisted by this wrapper on PostgreSQL.
