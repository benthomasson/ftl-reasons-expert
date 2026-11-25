---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: sync_agent in reasons/pg.py

# `PgApi.sync_agent` — Explanation

## Purpose

Synchronizes a foreign agent's belief set into the current PostgreSQL domain with **remote-wins** semantics. Unlike `import_agent` (which is additive: adds new, never deletes), `sync_agent` performs a full reconciliation: it adds new claims, updates changed claims, retracts claims the remote agent no longer contains, and re-aligns the `is_out` truth state of every claim — all within the current agent's colon-delimited namespace (`{agent_name}:`).

It exists to support a "mirror this agent's current state" operation, typically used when the remote is authoritative (e.g., pulling from a remote brain / peer RMS) and the local view should become a faithful replica of that agent's current beliefs.

## Contract

**Preconditions**
- `self.conn` is an open psycopg v3 connection; `self.domain_id` is a valid UUID in the target `rms_nodes`.
- `claims` is a list of pre-normalized dicts produced by `import_agent._normalize_*` (not raw JSON). Each must have at least: `id` (str, unprefixed), `text` (str), `is_out` (bool), `raw_justifications` (list of dicts with `type`, `antecedents`, `outlist`, optional `label`), and optionally `source`, `source_url`, `source_hash`, `date`, `metadata`.
- Each `claim["id"]` must be stable across calls — it is the reconciliation key. Two claims with the same `id` produce an undefined (probably doubled) result.
- `agent_name` must not contain `:` (used as the namespace separator).
- `nogoods` items must have `nodes` (list of unprefixed str) and `id` (str).

**Postconditions**
- Every node matching `{agent_name}:%` in the domain is either (a) present in `remote_ids`, (b) the namespace `active` premise, or (c) the `inactive` kill-switch — anything else is retracted.
- Every claim from `remote_ids` is present as a node under `{agent_name}:{claim.id}` and its justifications match the incoming claim (delete-and-reinsert if changed).
- All nodes under this namespace are gated behind the `{agent_name}:inactive` kill switch (every claim's outlist contains `inactive_id`).
- The network's truth values are recomputed to a fixpoint via `propagate()`.
- All affected transactions are committed.

**Invariants maintained**
- Retracted nodes carry `metadata._retracted = true` in addition to `truth_value = 'OUT'`. The `_retracted` flag is the *actual* no-resurrect marker; `truth_value='OUT'` alone is only the current state and can be flipped by propagation.
- `active_id` = `{agent_name}:active` is a premise (`ensure_namespace`); `inactive_id` is derived from an SL justification `antecedents=[], outlist=[active_id]`, so it is IN iff `active` is OUT.
- Client-provided `metadata` keys starting with `_` are (mostly — see "Assumptions" below) reserved and ignored/merged carefully so a remote cannot inject `_retracted`.

## Parameters

| Name | Type | Meaning |
|---|---|---|
| `agent_name` | `str` | Namespace for the remote agent. Becomes the `prefix`. Must not contain `:`. |
| `claims` | `list[dict]` | Normalized claims from `import_agent._normalize_*`. Sorted internally by `_topo_sort_claims`, so input order need not be topologically valid. |
| `nogoods` | `list[dict]` | Each with `id` and `nodes` (unprefixed). Only imported if at least 2 nodes resolve locally after prefixing; skipped if the nogood ID already exists. |
| `source_path` | `str` (default `""`) | Stored as `metadata.imported_from` on every claim and as `source` on the `inactive` premise. |

**Edge cases to be aware of**
- `claims` that reference each other by unprefixed ID work — the code prefixes all antecedents/outlist uniformly.
- An empty `claims` list will retract every belief under `{agent_name}:` (except infra), so it must be paired with a `nogoods=[]` to just "kill the agent".
- Nested namespace IDs like `agent:sub:node` inside a claim's `id` are supported by the prefix logic but **also** match `LIKE 'agent:%'` in `local_agent_ids`, so `sync_agent` treats them as owned by this agent — a nested-namespace-aware design would break this.

## Return Value

A dict with:

```
{
    "agent": str,
    "prefix": str,
    "active_node": str,           # "{agent_name}:active"
    "created_premise": bool,      # True if THIS call created the inactive kill-switch premise
    "beliefs_added": int,
    "beliefs_updated": int,       # node existed and text/source/justs/truth changed
    "beliefs_unchanged": int,     # node existed and nothing changed
    "beliefs_removed": int,       # local-only nodes retracted (in remote set but not present locally)
    "beliefs_retracted": int,     # remote claims flagged is_out with no justifications, explicitly retracted
    "beliefs_unchanged": int,
    "beliefs_propagated": 0,      # ALWAYS 0 — placeholder, kept for API parity with import_agent
    "nogoods_imported": int,
}
```

Caller obligations: the returned `beliefs_propagated` is not meaningful; if you need propagation counts, compute from `get_log()` or diff before/after. `created_premise` is misnamed — it tracks the *inactive* node, not the namespace premise (that one is created by `ensure_namespace` and its `created` flag is discarded).

## Algorithm

1. **Kill-switch setup.** Call `ensure_namespace(agent_name)` to guarantee `{agent_name}:active` exists. Then check for `{agent_name}:inactive`; if missing, insert it as an SL node with `antecedents=[]` and `outlist=[active_id]`. `created_premise` records whether this insert actually happened (i.e., whether the kill switch is fresh). Commit.

2. **Compute the two ID sets**
   - `remote_ids` = `{ prefix + claim["id"] for claim in claims }`
   - `local_agent_ids` = all existing `{agent_name}:%` IDs **minus** `{active, inactive}` (loaded with `LIKE '{agent_name}:%'`).

3. **Topo-sort claims** with `_topo_sort_claims` so that a dependent claim's antecedents have been processed (and thus have correct truth values) before `_add_node_raw` calls `_compute_truth` on it.

4. **Main reconciliation loop** for each claim in topo order:
   - Build prefixed `justs`. Every justification gets `inactive_id` prepended to its outlist — this is the kill switch.
   - If the claim is a premise (no `raw_justifications`), synthesize a default `SL` justification `antecedents=[], outlist=[inactive_id]`, so even "premise" claims are gated by the kill switch.
   - **Existing node path**:
     - UPDATE `text`/`source`/`source_url`/`source_hash`/`date` only for fields where the remote supplied a value (else keep local).
     - Merge `claim["metadata"]` into `old_meta`, **skipping keys starting with `_`**. Always set `new_meta["imported_from"]`.
     - Diff justifications by length then element-wise (order-sensitive `zip`); on diff, `DELETE` + `INSERT` all justifications.
     - Handle `is_out`:
       - **`is_out` and no justifications**: append to `retract_after`. Do not touch truth_value here.
       - **`is_out` with justifications**: set `truth_value='OUT'`; set `_retracted=True` if not already set.
       - **not `is_out`**: ensure `_retracted` is removed; if old was `OUT`, add to `assert_after` (assert/recompute it later).
     - Increment `updated` or `unchanged`.
   - **New node path**: build metadata (note: `_`-prefixed keys are NOT filtered here — remote metadata wins for new node), `_add_node_raw` with prefixed justs. Same `is_out` truth handling (retract_after or set OUT + `_retracted`). Increment `added`.

5. **Commit** the loop's changes.

6. **Remove local-only nodes.** `removed_ids = local_agent_ids - remote_ids`. For each, if currently IN: set `truth_value='OUT'` + `_retracted` and log. If already OUT but not `_retracted`: mark it retracted so propagation won't resurrect it. Count as `beliefs_removed`. Commit.

7. **Clear `_retracted` for `assert_after`** with `metadata - '_retracted'` (removes the JSONB key). This is what lets propagation later flip these OUT → IN.

8. **`self.propagate()`** — full-network recompute to a fixpoint (skips `_retracted` nodes).

9. **`retract_after` phase** — for nodes that are OUT-and-premise-by-remote: set `truth_value='OUT'` (if not already), attach `_retracted=True`, log `"retract"`, `beliefs_retracted++`. Commit + `self.propagate()` again (so the freshly-retracted OUTs cascade to their dependents).

10. **Import nogoods** via `_import_agent_nogoods(prefix, nogoods)`. Returns count of newly inserted nogoods (only when ≥2 prefixed nodes exist, and not already present).

11. Return the stat dict.

## Side Effects

- **DB writes**: inserts/updates/deletes across `rms_nodes`, `rms_justifications`, `rms_nogoods`, and `rms_propagation_log`. Multiple `self.conn.commit()` calls — the method is NOT atomic across all steps. If the process crashes mid-way, the domain may be left in an intermediate state (e.g., additions committed but `removed_ids` retraction not applied).
- **Commits are interleaved**: after kill-switch creation, after the main loop, after removal, after `assert_after`, inside each `propagate()`, after retract_after phase, after `_import_agent_nogoods`. This prevents a single-giant-transaction rollback but means a partial failure across phases leaves a partially-synced state.
- **Log entries** via `_log(cur, "retract", node_id, "OUT")` for removals and retract_after retractions. Logged even for previously-OUT nodes (in the removed case as just metadata flip — note the log call is unconditional in that branch).
- **Full propagation pass** — O(nodes × justifications) each call, twice for the retract_after path.
- **No new external I/O** — pure DB operations, no filesystem/network.

## Error Handling

- **No `try/except`.** Any psycopg error (`IntegrityError`, `ProgrammingError`, `OperationalError`) propagates to the caller. Callers that expect to handle failure must wrap the method.
- **`KeyError`** is possible from `_add_node_raw` only if the caller (this method) pre-populates `existing` incorrectly — it doesn't, but a concurrent writer to the same domain could cause a unique-constraint violation on `PRIMARY KEY (id, domain_id)` because INSERT doesn't `ON CONFLICT`.
- **`json` decode errors**: `json.loads(old_meta)` when `isinstance(old_meta, str)` — protected because psycopg returns JSONB as dict/list already by default; the string branch is defensive.
- **Silently ignored**: `_import_agent_nogoods` drops nogoods with fewer than 2 valid nodes or that already exist.
- **Silent partial sync** on exceptions: an error in the loop leaves previously committed steps applied but the loop's own `commit()` never runs. Callers should treat this as "unknown state, re-sync to recover."

## Usage Patterns

Called from a top-level dispatch layer (see `tests/test_pg_dispatch.py`, `tests/test_sync_agent.py`). Typical call:

```python
claims = [ c for c in _normalize_remote(remote_export) ]
nogoods = remote_export.get("nogoods", [])
result = api.sync_agent("bob", claims, nogoods, source_path="manual-sync")
# result["beliefs_added"], ["beliefs_updated"], etc.
```

Caller obligations:
- Must supply ALL of the remote's claims in one call. Splitting a single agent's claims across multiple calls will cause the previous call's not-yet-seen claims to be *removed* in the second call.
- Should treat `sync_agent` as non-idempotent under failure — re-run to converge.
- Kill switch is controlled by `retract_node("{agent}:active")` — when that premise goes OUT, `inactive` goes IN, and every claim's outlist (which contains `inactive`) invalidates the claim, forcing all claims OUT. `assert_node("{agent}:active")` reverses it.

## Dependencies

- `reasons/import_agent.py:_topo_sort_claims` — lazy import to avoid circular dependency (import_agent likely imports pg at module load).
- `PgApi.ensure_namespace`, `_add_node_raw`, `_import_agent_nogoods`, `propagate`, `_log` — internal helpers.
- `psycopg` v3 cursors with JSONB, `LIKE`, `ANY`, `||`, `-` operators.

## Assumptions not enforced by the type system

1. **`agent_name` contains no `:`** — the namespace scheme relies on this.
2. **`claim["id"]` values are unique within `claims`** — duplicates will double-count or produce order-dependent results.
3. **`claim["metadata"]` keys are strings** — JSONB requires object keys be strings; numeric keys will raise at `json.dumps`.
4. **New-node `metadata` is NOT filtered for `_`-prefixed keys** (unlike existing-node path), so `claim["metadata"]` could inject `_retracted`, `_agent`, etc. into a fresh node. This is an inconsistency — remote data on the "existing" path is sandboxed but on the "new" path is not.
5. **For existing nodes that become `retract_after`** (`is_out=True` and no `raw_justifications`), the merged `new_meta` is computed but **never written to the DB** — only the `updates` dict (text/source/date) is persisted, and the later retract_after phase only appends `_retracted`. Metadata changes from the remote for those nodes are silently dropped. May be intentional (retracted nodes don't need fresh metadata) or a bug.
6. **Justification equality is order-sensitive** (`zip` comparison) — reordered but semantically identical justifications trigger a DELETE+INSERT and count as `updated`.
7. **`local_agent_ids` uses `LIKE '{agent}:%'`** — a nested namespace like `agent:sub:node` owned by a different sync will be retracted by this call.

## Topics to Explore

- [file] `reasons/import_agent.py` — Where `_topo_sort_claims` lives and how `_normalize_*` shapes claims; the `import_agent` vs `sync_agent` distinction lives here.
- [function] `reasons/pg.py:import_agent` — The additive-only sibling of `sync_agent`; diffing the two reveals which sync-specific logic (removal, `_retracted` clearing, `assert_after`) was added.
- [function] `reasons/pg.py:propagate` — Understanding how `_retracted` is honored during fixpoint recomputation (this is the mechanism `retract_after` relies on).
- [function] `reasons/pg.py:_import_agent_nogoods` — The nogood import helper; only-imports-≥2-valid-nodes rule and ID collision handling.
- [general] `kill-switch-semantics` — How `active` / `inactive` gate every claim via outlists; the central trick of the entire agent-import/sync design.

## Beliefs

- `sync-agent-adds-inactive-to-every-claim-outlist` — Every imported claim's justifications get `{agent}:inactive` prepended to `outlist`, so the kill switch gates all beliefs even premises.
- `_retracted-metadata-flag-prevents-propagation-resurrection` — `propagate()` and `_propagate()` skip any node whose metadata has `_retracted: true`, which is why `truth_value='OUT'` alone does not protect a node from being flipped back to IN.
- `sync-agent-considered-removal-is-truth-out-plus-retracted` — Local nodes not present in the remote claim set are given `truth_value='OUT'` AND `metadata._retracted=true`; the second is necessary, the first is cosmetic for the current state.
- `sync-agent-runs-propagate-twice-when-retracting-premises` — When `retract_after` is non-empty, `propagate()` executes once before the retract_after loop and once after, to cascade OUT → dependents.
- `sync-agent-new-node-metadata-is-not-filtered-for-underscore-keys` — The existing-node metadata merge skips keys starting with `_`, but the new-node path (`metadata = claim.get("metadata", {}).copy()`) does not, creating an inconsistency a caller could exploit.
