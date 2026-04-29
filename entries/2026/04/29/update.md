# Update Summary

**Date:** 2026-04-29
**Time:** 17:40

## New Gated OUT Beliefs

_None_

## New Negative IN Beliefs

- **ask-falls-back-to-raw-search**: When LLM synthesis fails for any reason (timeout, missing CLI, non-zero exit), `ask()` returns the raw FTS5 search results as fallback.
- **check-stale-no-dedup-by-source-path**: Multiple nodes referencing the same missing source file each produce independent `source_deleted` results — no deduplication by file path.
- **check-stale-source-deleted-returns-none-hashes**: When a source file is missing, `check_stale` returns a result dict with `reason="source_deleted"`, `new_hash=None`, and `source_path=None` (fix for issue #25 — previously this case was silently skipped).
- **compact-is-infallible**: `compact()` handles empty networks, zero-budget, and missing metadata without raising exceptions — designed to always produce a valid string.
- **count-accumulates-linearly**: Documents the bug fix for issue #23 — already covered by existing `derive-agent-count-bug` which tracks this defect
- **derive-unused-imports**: Dead code observation (`subprocess`, `sys`, `Path` imported but unused) — unstable detail that will change when cleaned up.
- **derive-validate-before-apply**: `apply_proposals` trusts its input unconditionally; callers must run `validate_proposals` first or risk database errors from missing antecedents or duplicate IDs.
- **hash-sources-is-additive-by-default**: `hash_sources` without `force=True` never overwrites an existing non-empty `source_hash`; it only backfills nodes with empty or missing hashes.
- **hash-sources-no-overwrite-default**: `hash_sources` with `force=False` (the default) only backfills missing hashes and will never modify a node that already has a `source_hash` value.
- **import-json-uses-topological-sort**: `api.import_json()` reconstructs networks via a multi-pass topological sort loop — repeatedly adding nodes whose dependencies are already loaded — with a force-add fallback for cycles or missing deps.
- **list-negative-uses-two-stage-classification**: `list_negative` uses keyword pre-filtering against a hardcoded `NEGATIVE_TERMS` list (~50 words), then LLM classification via `ask._invoke_claude` to eliminate false positives.
- **network-missing-outlist-passes**: Duplicates existing belief `missing-outlist-nodes-pass-validation`.
- **pg-cleanup-covers-five-tables**: Fixture teardown deletes from exactly `rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, and `rms_nodes` — adding a new `project_id`-scoped table without updating the fixture will leak test data.
- **staleness-results-are-dicts-not-exceptions**: Both `check_stale` and `hash_sources` report problems (missing files, changed content, deleted sources) as list-of-dict return values, never raising exceptions — designed for batch audit operations that report all problems rather than aborting on the first.
- **storage-handles-schema-evolution-via-try-except**: Missing tables from older database versions (`repos`, `network_meta`) are handled by swallowing exceptions during `load()` rather than formal migrations — a backward-compatibility substitute that silently degrades.
- **storage-old-schema-compat**: `load()` tolerates missing `network_meta` and `repos` tables via silent `try/except`, supporting databases created before those tables were added; `next_nogood_id` is derived from existing IDs as a fallback.
- **unused-path-import**: Dead code observation (`Path` imported but unused) — unstable detail that will change when cleaned up.
- **verify-dependents-is-readonly**: `verify_dependents()` never modifies `node.dependents`; it only reads and reports discrepancies as a list of human-readable strings containing `"extra"` or `"missing"`.

## Critical Watch List

No active issues. All 10 defect premises have been retracted and their GATE beliefs resolved.


## Statistics

- **Total beliefs:** 542
- **IN:** 483
- **OUT:** 59
- **Derived:** 217
- **Gated OUT (all):** 0
- **Negative IN (all):** 35
- **New beliefs this run:** 242
- **New gated OUT:** 0
- **New negative IN:** 18

