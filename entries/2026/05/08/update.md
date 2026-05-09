# Update Summary

**Date:** 2026-05-08
**Time:** 14:50

## New Gated OUT Beliefs

_None_

## New Negative IN Beliefs

- **cluster-deps-optional-with-graceful-skip**: The clustering module (`reasons_lib.cluster`) is behind an optional `[cluster]` install extra; when `sentence-transformers` or `scikit-learn` are missing, `_require_cluster_deps` raises `ImportError` and all dependent tests skip cleanly.
- **parse-review-response-never-raises**: `parse_review_response` returns an empty list on any malformed input — bad JSON, non-list JSON, items missing `id` fields — rather than raising exceptions, with missing boolean fields defaulting to `True`.
- **review-and-contradictions-catch-orthogonal-errors**: `review-beliefs` catches invalid reasoning within individual derivation steps (over-generalization, missing bridges, strength escalation) while `contradictions` catches incompatible facts across independently valid beliefs (absolute claims vs. documented exceptions) — the two commands are complementary with different cascade profiles (review: high cascade at depth, contradictions: low cascade at leaves).
- **review-parse-defaults-safe**: When the LLM response is missing fields, `parse_review_response` defaults to `valid=True`, `sufficient=True`, `necessary=True` — a parse error never generates a false-positive validity warning.

## Critical Watch List

### Gated (blocked)

- **storage-is-fully-production-grade-across-backends**: Both storage backends achieve fully production-grade operation — concurrent access optimization (WAL mode, derived FTS5 indexes), equivalent safety guarantees (atomic isolated mutations through backend-appropriate mechanisms), and multi-tenant isolation — when both backends provide complete API coverage.
  - Blocked by: `pgapi-partial-api-coverage` — PgApi implements core operations (add/retract/assert/search/nogood/explain) but defers simulation, dialectics, namespace support, import/export, and maintenance operations to future work
- **verified-correctness-extends-to-all-backends**: Verified production correctness — spanning all belief origins with deterministic state trajectories and full maintenance loop observability for indefinite independent verification — extends identically across both SQLite and PostgreSQL storage backends with no backend-specific safety gaps.
  - Blocked by: `pgapi-partial-api-coverage` — PgApi implements core operations (add/retract/assert/search/nogood/explain) but defers simulation, dialectics, namespace support, import/export, and maintenance operations to future work

### Active Issues

- **llm-prompt-always-via-stdin**: Prompts are passed to LLM subprocesses via `subprocess.run(input=...)`, never as command-line arguments — avoiding shell injection and argument length limits.


## Statistics

- **Total beliefs:** 939
- **IN:** 569
- **OUT:** 370
- **Derived:** 515
- **Gated OUT (all):** 18
- **Negative IN (all):** 46
- **New beliefs this run:** 48
- **New gated OUT:** 0
- **New negative IN:** 4

