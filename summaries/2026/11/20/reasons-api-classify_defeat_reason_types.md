---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: classify_defeat_reason_types in reasons/api.py

# `classify_defeat_reason_types` — Explanation

## 1. Purpose

This function performs a **batch classification pass** over a Reason Maintenance System network: it finds "defeater" nodes that have not yet been tagged with a logical failure mode (`defeat_reason_type` metadata), sends each one — together with the belief it defeats — to an LLM, and optionally writes the resulting classification back to the database.

It exists so that a network which has accumulated defeaters (from `defeat_justification`, `defeat_with_scope`, `migrate_retract_to_defeaters`, etc.) can be retroactively annotated with a consistent taxonomy of *why* each justification was defeated (e.g. `over-generalizes`, `invalid-inference`, `missing-scope`), enabling downstream analytics, reports, and grouped review. Because it is the only path in the API for populating `defeat_reason_type` in bulk, it doubles as the "backfill" tool.

## 2. Contract

**Preconditions**
- `db_path` points to an existing, valid RMS SQLite database (loadable by `export_network`).
- The node metadata schema uses `metadata["defeats_node"]` to identify defeater nodes and `metadata["defeater_type"]` to categorize them (as produced by `defeat_justification` / `defeat_with_scope`).
- `classify_defeat_reason` in `reasons.review` accepts the positional signature `(defeater_text, defeated_text, model, timeout)`.

**Postconditions**
- Returns a dict with exactly the keys `classified`, `skipped`, `errors`.
- The database is **unmodified** when `dry_run=True` (the default).
- When `dry_run=False`, every node in `classified` has, at return time, `metadata["defeat_reason_type"]` set to the classification string.
- Nodes whose classification returned an empty/falsy string are **not** written and appear only in `errors`.

**Invariants**
- A node is a candidate only if `metadata["defeats_node"]` is truthy. Plain beliefs, challengers without defeater metadata, and outlist-only defeaters are not touched.
- A node is never both in `classified` and in `skipped` or `errors`.
- `skipped` does **not** include nodes that are not defeaters at all — those are silently ignored. The list only contains defeaters rejected for classification (already classified, or filtered by `defeater_type`).
- Iteration order follows `nodes.items()` order from `export_network` (sorted by node ID); output list order is thus deterministic per network state.

## 3. Parameters

| Name | Type | Default | Meaning & Edge Cases |
|---|---|---|---|
| `defeater_type_filter` | `str \| None` | `None` | If set, restrict classification to defeaters whose `metadata["defeater_type"]` **equals** this string. Defeaters with **no** `defeater_type` metadata are skipped (since `None != filter`). No validation that the value is a known type. |
| `model` | `str` | `"claude"` | Passed verbatim to `classify_defeat_reason`; not validated here. |
| `timeout` | `int` | `300` | Passed to `classify_defeat_reason` as the per-call timeout (seconds). Applied **per candidate**, not as a total budget. |
| `dry_run` | `bool` | `True` | Safe default. `True` → classify but do not persist. `False` → call `set_metadata` per classified node. |
| `db_path` | `str` | `DEFAULT_DB` (`"reasons.db"`) | Path to SQLite DB. No `pg_conninfo`/`domain_id` params — **PostgreSQL backend is not supported by this function**, unlike most API functions. |

## 4. Return Value

Always returns a dict (never `None`) shaped:

```python
{
  "classified": [{"id": str, "defeat_reason_type": str}, ...],
  "skipped":    [{"id": str, "reason": str}, ...],
  "errors":     [{"id": str, "reason": str}, ...],
}
```

- `classified` — entries whose LLM classification was non-empty. **These entries are reported even when `dry_run=True`** — the caller must check `dry_run` to know whether the DB actually reflects them.
- `skipped` — defeaters deliberately excluded. Reasons are `"already classified"` or `"defeater_type '<T>' != '<filter>'"`. Useful as an audit trail.
- `errors` — attempts that failed. Only path currently generated: `"classification returned empty"`. LLM exceptions are **not** captured here (see §7).

Callers must be prepared for an empty result `{"classified": [], "skipped": [], "errors": []}` when the network has no defeaters at all — this is success, not failure.

## 5. Algorithm

1. **Lazy import** `classify_defeat_reason` from `.review` (avoids entangling `reasons.api` with LLM-client imports at module load).
2. **Export** the whole network via `export_network(db_path=db_path)` and pull `nodes` from it. This is a full materialization of every node (text, metadata, justifications) — memory cost grows with network size.
3. **Candidate selection** — single pass over `nodes.items()`:
   - Skip any node whose `metadata` is falsy or lacks `defeats_node`.
   - Skip and record `"already classified"` if `defeat_reason_type` is present.
   - Skip and record the mismatch if `defeater_type_filter` is set and the node's `defeater_type` differs (including "missing").
   - Otherwise append `nid` to `candidates`.
4. **Classify loop** — for each candidate `nid`:
   - Resolve the defeated node as `nodes.get(meta["defeats_node"], {})`. If the defeater points at a node that no longer exists in the export, `defeated` becomes `{}` and `defeated.get("text", "")` returns an empty string — **this happens silently, no error is raised or recorded**.
   - Call `classify_defeat_reason(defeater_text, defeated_text, model, timeout)`.
   - If the return value is falsy → append to `errors` with `"classification returned empty"`; `continue`.
   - Else append `{"id": nid, "defeat_reason_type": reason_type}` to `classified`.
   - If `not dry_run`, call `set_metadata(nid, "defeat_reason_type", reason_type, db_path=db_path)`.
5. Return the three lists.

## 6. Side Effects

- **Reads/IO**: `export_network` opens the DB, may reuse the module-level `_network_cache`, and reads `node_tags`/`node_sources` tables. Cost is O(network).
- **External calls**: `classify_defeat_reason` invokes an LLM for **each** candidate, sequentially. No batching, no parallelism, no rate-limit handling. `timeout` is per-call.
- **Writes (when `dry_run=False`)**: One `set_metadata` call per classified node. `set_metadata` acquires `_with_network(db_path, write=True)`, meaning each write is a full load → mutate → save → publish cycle. This is **N full network round-trips** for N classifications, plus one DB write per classification — a notable efficiency concern on large networks.
- **Pubsub**: Because `set_metadata` uses `_with_network(write=True)`, each write also triggers `pubsub.publish` for subscribers. Downstream subscribers will see N events, not one batched event.
- **Returns nothing to stdout/stderr** (unlike `list_negative`, which prints progress).

## 7. Error Handling

- **No `try/except` around `classify_defeat_reason`.** Any exception from the LLM (network failure, bad model name, timeout that raises rather than returns) propagates out of the function, aborting the loop. Prior classifications that were already written via `set_metadata` in this run **remain committed** — there is no rollback. This makes partial application possible for a caller that catches and retries.
- **No `try/except` around `export_network`.** Missing DB or corrupt schema raises end-to-end.
- **Empty classification is soft-failed**: reported in `errors`, function continues.
- **Dangling `defeats_node` reference is not reported**: `nodes.get(defeated_id, {})` returns empty; the LLM is invoked with an empty "defeated text" and may still return a classification that gets persisted. This is a real correctness footgun on refactored networks.
- **`KeyError` risk inside `set_metadata`**: if the node disappears between the initial export and the per-node `set_metadata` load (concurrent writer), `set_metadata` raises `KeyError`. No guard here.
- **No validation of `defeater_type_filter`** against known types; a typo yields an empty `classified` list and a full-scale skip list, which is easy to miss.

## 8. Usage Patterns

Typical call site (CLI / script / maintenance job):

```python
result = classify_defeat_reason_types(
    defeater_type_filter="invalid-inference",
    model="claude",
    timeout=120,
    dry_run=True,          # preview
    db_path="reasons.db",
)
# inspect result["classified"]; if acceptable:
result = classify_defeat_reason_types(..., dry_run=False)
```

Caller obligations:
- **Flip `dry_run=False` explicitly** to persist — the default does nothing to the DB, which is the intended safe posture but easy to overlook.
- **Retry after LLM exceptions** — since no per-item error capture exists, wrap calls with backoff / batching at the caller level if the model is flaky.
- **Ensure the referenced defeated nodes still exist** — otherwise empty "defeated text" classifications may be persisted silently.
- **Be aware this is not idempotent in cost** — a second run classifies nothing but re-imports and re-iterates the whole network (all candidates show up in `skipped`).
- **This function has no PostgreSQL counterpart** — do not call it on a pg-backed deployment.

## 9. Dependencies

Internal (`reasons.api`):
- `export_network` — full network snapshot as a plain dict.
- `set_metadata` — writes a single metadata key on a node (opens its own `_with_network` transaction).
- `DEFAULT_DB` — module constant `"reasons.db"`.
- `_with_network` — indirectly, through the above.

Lazy internal import:
- `reasons.review:classify_defeat_reason` — the actual LLM prompt/harness that decides the failure mode string.

Implicit external:
- SQLite (via `export_network` / `set_metadata`).
- Whatever transport `classify_defeat_reason` uses (LLM API / CLI subprocess — defined in `.review`).

## Assumptions Not Enforced by the Type System

1. **Referential integrity of `defeats_node`** — nothing verifies the pointed-to node exists; `nodes.get(defeated_id, {})` silently degrades.
2. **`classify_defeat_reason` arity** — a positional call `(defeater_text, defeated_text, model, timeout)`; signature changes in `.review` would break here at runtime.
3. **`defeat_reason_type` is a string** — no type check before `set_metadata`; a non-string return would be persisted opaquely.
4. **Export node shape** — assumes every node dict has `"metadata"` (may be `None`) and `"text"`; this is guaranteed by `export_network` today, but not by contract.
5. **Concurrency** — assumes no other writer mutates the node between the export and each `set_metadata`; safe in single-process use, racy under concurrent writes.

---

## Topics to Explore

- [function] `reasons/review.py:classify_defeat_reason` — The LLM prompt, expected output format, and failure-mode taxonomy that this batch driver depends on.
- [function] `reasons/api.py:set_metadata` — Understand why each classification triggers a full load/save/pubsub cycle, and whether a batched write path exists elsewhere.
- [function] `reasons/api.py:defeat_justification` — Where `defeats_node` and `defeater_type` metadata are produced; shows the upstream contract this classifier consumes.
- [function] `reasons/api.py:export_network` — Determines what metadata survives export (e.g. `_`-prefixed keys are stripped) and how `_network_cache` affects repeated reads.
- [general] `defeater-taxonomy` — Which `defeat_reason_type` values are meaningful in this system and how downstream reports group on them.

## Beliefs

- `classify-defeat-reason-types-is-a-batch-backfill` — The function's only job is to populate `defeat_reason_type` metadata on existing defeaters; it never creates or removes nodes.
- `dry-run-defaults-to-true` — The function does not mutate the database unless the caller explicitly passes `dry_run=False`.
- `skipped-list-excludes-non-defeaters` — Nodes without `defeats_node` metadata are silently ignored, not included in the `skipped` audit list.
- `llm-exceptions-propagate-unwrapped` — There is no `try/except` around `classify_defeat_reason`, so an LLM failure aborts the loop with already-written classifications left committed.
- `each-write-is-a-separate-transaction` — When `dry_run=False`, each classification invokes `set_metadata`, causing one load/save/pubsub cycle per node (N+1 DB work).
- `missing-defeated-node-is-not-an-error` — A `defeats_node` pointing at a non-existent node causes the classifier to be invoked with empty defeated text, and any non-empty result is persisted without warning.
- `postgres-backend-unsupported` — This function lacks `pg_conninfo`/`domain_id` parameters and only runs against the SQLite backend.
