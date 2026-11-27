---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Clarify whether missing values, deletions, and `None` are distinguished elsewhere in the system.

## Short answer

They are **partially distinguished, but not by a single system-wide `None`/missing/tombstone convention**. The domain/reporting layer uses explicit reason strings and counters such as `source_deleted`, `missing_hashes`, `missing_property`, and `missing nodes`. The SQLite layer, however, mostly flattens missing string fields to `""` and preserves actual `NULL` only in a few places. So the answer depends on which boundary you mean:

- At the **belief/staleness/reporting layer**: yes, several kinds of “missing” are distinguished.
- At the **SQLite persistence layer**: mostly no; absent strings and empty strings are usually the same value.
- For **`None` specifically**: it is used field-by-field, not as a universal missing-value sentinel.

For `reasons/storage.py`, the observation did not include line numbers, so I reference the `SCHEMA` definition and `Storage.save` / `Storage.load` symbols. Other line references below come from the supplied observations.

## Where missing / deletion / `None` are distinguished

### 1. Missing source file vs empty source file

`check_stale` emits a specific reason for a source file that has disappeared:

- `reasons/check_stale.py:101` — `"reason": "source_deleted"`
- `reasons/cli.py:1136` — checks `if item.get("reason") == "source_deleted":`
- `reasons/pg.py:893` — same `source_deleted` reason in the PostgreSQL path

The tests explicitly distinguish a **missing source file** from an **empty file**:

- `tests/test_check_stale_issue25.py:21` — “Core behavior: missing source files produce source_deleted results.”
- `tests/test_check_stale_issue25.py:141` — `test_empty_file_is_not_deleted`

So at the staleness-check layer, “file is missing” is not represented as `None`; it is represented as a structured result with `reason == "source_deleted"`, and an empty file is deliberately not treated as deleted.

### 2. Missing hashes are counted/reported separately

Missing Merkle/content hashes are handled as a count, not as `None`:

- `reasons/api.py:2734` — return dict includes `"missing_hashes": int`
- `reasons/api.py:2744` — maps `result["missing_hashes"]`
- `reasons/cli.py:1171-1172` — prints `Missing hashes: ...`
- `reasons/merkle.py:113` and `reasons/merkle.py:120` — `missing = 0`, `missing += 1`

So a missing hash is a reporting/backfill concept. However, at rest, the storage schema defaults `source_hash` and `text_hash` to `''`, so the database itself does not necessarily preserve “missing” vs “empty hash” as distinct values.

### 3. Missing references/properties are explicit domain findings or skipped items

Missing antecedents, properties, and dependencies are often represented as explicit findings rather than `None`:

- `reasons/derive.py:627-630` — missing antecedents/unless nodes are collected into `missing` / `missing_unless`, then reported in `skipped` as `"missing nodes: ..."`.
- `reasons/api.py:1418`, `1436`, `1502` — `missing_property` is a named finding when a derived belief claims a property no antecedent establishes.
- `reasons/import_beliefs.py:201`, `217-218` — missing dependencies are handled gracefully; remaining claims are added anyway and will compute as `OUT` if deps are missing.

So these are context-specific “missing” semantics, not a generic `None`.

### 4. Deletion vs retraction

The deletion/term search found `source_deleted` and its tests, but **no `tombstone`, `soft delete`, or `hard delete` matches** in the supplied results. That suggests there is not an observed generic tombstone/soft-delete mechanism in the searched terms.

Separately, the code has belief-lifecycle concepts:

- `reasons/storage.py` `nodes` table includes `retracted_at` and `truth_value`.
- `reasons/propose_update.py` supports action `"retract"` and uses `proposed_text: None` for retractions.
- `tests/test_propose_update.py` `TestParseUpdateProposals.test_retract_proposal` asserts `results[0]["proposed_text"] is None`.

So there appear to be at least two different concepts:

1. **Source deletion/staleness**: `source_deleted`.
2. **Belief retraction**: `retract`, `retracted_at`, `truth_value` / propagation.

But the observations do not include the full retraction implementation, so we cannot fully prove how `retracted_at` behaves or whether any hard-delete path exists outside the searched matches.

Also note: `Storage.save` does `DELETE FROM nodes`, `DELETE FROM justifications`, etc. before rewriting the network. That is a persistence rewrite strategy, **not** a domain-level deletion marker. See `reasons/storage.py:Storage.save`.

### 5. `None` is field-specific, not global “missing”

The clearest observed `None` semantics are in proposal handling:

- `reasons/propose_update.py:parse_update_proposals` uses `item.get("proposed_text")`.
- For `"action": "retract"`, the prompt says `proposed_text` should be `null`.
- The test asserts `results[0]["proposed_text"] is None` for a retraction.

But because `.get("proposed_text")` is used, an absent key and an explicit JSON `null` both produce `None`. So `propose_update` does **not** distinguish “key missing” from “explicitly null” for `proposed_text`; it only distinguishes retraction from update via the `action` field.

In storage, `None`/`NULL` is also limited:

- `reasons/storage.py` `nodes.supporting_justification INTEGER DEFAULT NULL` is the explicit nullable node field.
- Most other node string fields default to `''`: `source`, `source_url`, `source_hash`, `text_hash`, `date`, `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at`.
- `Storage.load` coerces several values with `or ""`, e.g. `source_url or ""`, `text_hash or ""`, timestamps `or ""`, which erases `NULL` vs empty-string distinctions for those fields.

So at the SQLite boundary, most missing strings are stored as empty strings, not as `None`. The notable exception is `supporting_justification`.

## Practical interpretation

If you are writing code against this system:

- Do **not** assume `None` means “missing” globally.
- Treat `""` as the persisted empty/absent value for most node string fields.
- Treat `source_deleted` as a staleness result for a missing source file, not as a generic deleted record marker.
- Treat `retract` / `retracted_at` / `truth_value` as belief-lifecycle semantics, separate from source-file deletion.
- Treat `missing_hashes`, `missing_property`, and `"missing nodes"` as domain-specific reporting concepts.
- If you need to distinguish absent from empty in storage, you will likely need to inspect or change the storage schema; the current schema mostly does not preserve that distinction.

## Topics to Explore

- [file] `reasons/check_stale.py` — Confirm the exact branch that emits `source_deleted` and how missing file, empty file, and missing hash are separated.
- [function] `reasons/propose_update.py:parse_update_proposals` — Verify whether absent `proposed_text` and explicit `null` are intentionally conflated, and how `action` defaults interact.
- [function] `reasons/storage.py:Storage.load` — Check exactly which `NULL` values are preserved vs coerced to `""`.
- [general] `retraction-vs-source-deletion` — Explore `retracted_at`, `truth_value`, propagation, and any tombstone/soft-delete paths outside the searched terms.
- [file] `reasons/derive.py` — Inspect how missing antecedents/outlists are reported and whether they ever become `OUT` instead of remaining skipped.

## Beliefs

- `source-deleted-reason-distinguishes-missing-file-from-empty-file` — `check_stale` emits `reason: "source_deleted"` for missing source files (`reasons/check_stale.py:101`), while `tests/test_check_stale_issue25.py:141` asserts empty files are not treated as deleted.
- `storage-flattens-most-missing-strings-to-empty-string` — The SQLite `nodes` schema defaults `source`, `source_url`, `source_hash`, `text_hash`, and timestamp fields to `''`, and `Storage.load` coerces several to `""`, so most missing strings are not preserved as `NULL`.
- `supporting-justification-is-the-explicit-nullable-node-field` — In the SQLite `nodes` table, `supporting_justification INTEGER DEFAULT NULL` is the observed nullable node field; most other scalar fields are `NOT NULL` or default to `''`/`'{}'`.
- `proposed-text-none-is-retraction-specific` — In `parse_update_proposals`, retraction proposals use `proposed_text: None`, but `.get("proposed_text")` also returns `None` when the key is absent, so absent and explicit null are not distinguished there.
- `missing-hashes-are-reported-as-a-count` — `api.py` returns `missing_hashes` (`reasons/api.py:2734`), `cli.py` prints it (`reasons/cli.py:1171-1172`), and `merkle.py` increments a `missing` counter (`reasons/merkle.py:113`, `reasons/merkle.py:120`), making missing hashes a reporting concept rather than a `None` value.
