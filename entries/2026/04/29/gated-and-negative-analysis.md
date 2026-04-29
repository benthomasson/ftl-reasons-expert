# Gated OUT and Negative IN Belief Analysis

**Date:** 2026-04-29
**Source:** `reasons list-gated` and `reasons list-negative` against ftl-reasons-expert/reasons.db

## Gated OUT Beliefs: 4 blockers gating 14 beliefs

After the derive --exhaust run (10 rounds, 119 new derived beliefs), the network has 4 active blockers preventing 14 derived beliefs from going IN.

### Blocker: `outlist-nodes-not-in-dependents-index` (9 gated)

The largest blocker. The SQLite `_propagate` BFS uses `node.dependents` to find nodes to re-evaluate, but outlist nodes are not tracked in that index. When a blocker goes OUT, gated beliefs aren't enqueued for re-evaluation. This gates 9 beliefs about propagation completeness, including `incremental-propagation-is-fully-complete`, `defeat-reversal-propagates-automatically`, and `dependency-tracking-is-complete-for-all-reference-types`.

**Note:** The PgApi implementation does NOT have this bug — `_find_dependents` queries both `antecedents @> ... OR outlist @> ...`. This is a SQLite/in-memory Network issue only. The fix would be to add outlist nodes to `node.dependents` in `Network.add_node` and `Network.add_justification`.

### Blocker: `pgapi-partial-api-coverage` (3 gated)

PgApi is missing namespace support, import/export, and some maintenance operations. This gates beliefs about dual-backend parity and completeness. PRs #52-56 narrowed the gap (added challenge/defend/compact/list-gated/what-if), but namespace and import/export remain.

### Blocker: `pg-antecedent-refs-have-no-fk-constraints` (1 gated)

JSONB arrays in `rms_justifications` have no foreign key enforcement. Phantom node references are possible within a project. Gates `pg-multi-tenancy-is-referentially-complete`.

### Blocker: `cmd-propagate-bypasses-api` (1 gated)

The `cmd_propagate` CLI handler goes directly to Storage → Network instead of through the API layer. Gates beliefs about CLI being a pure delegation layer.

## Negative IN Beliefs: 9 out of 588

The LLM classified 9 genuinely negative beliefs from 196 keyword candidates (95.4% of candidates were false positives — beliefs about error handling mechanisms, not problems).

### Structural concerns (highest priority)

- **outlist-nodes-not-in-dependents-index** — The root cause blocker. Also the most impactful negative belief: it gates 9 other beliefs and represents an incomplete implementation of Doyle's TMS.
- **cp-equals-sl** — CP justifications use SL evaluation logic. Either an intentional simplification or incomplete implementation of Doyle's conditional-proof mechanism.
- **pg-antecedent-refs-have-no-fk-constraints** — JSONB arrays can't enforce referential integrity. A PostgreSQL design tradeoff.

### API design concerns

- **access-control-enforced-at-read-not-write** — Write operations don't check `_is_visible`. Read-side enforcement only.
- **derive-validate-before-apply** — `apply_proposals` trusts input unconditionally. Defensive but fragile contract.
- **pgapi-partial-api-coverage** — Narrowing but still incomplete PgApi coverage.

### Implementation details

- **storage-handles-schema-evolution-via-try-except** — Silent try/except for missing tables instead of migrations.
- **invoke-claude-raises-when-binary-missing** — Uncaught exception path in `ask()`.
- **api-tests-cover-subset** — Coverage inventory, not a behavioral defect.

## Observations

1. The `outlist-nodes-not-in-dependents-index` blocker is the single most impactful issue — fixing it would ungate 9 derived beliefs and resolve the largest negative IN belief simultaneously.
2. The PgApi already handles outlist dependents correctly, so this is a SQLite/Network-only fix.
3. The negative belief classifier worked well on this database — 196 candidates down to 9, with good precision. The false positives were almost entirely beliefs describing error handling patterns.
