# Update Summary

**Date:** 2026-05-05
**Time:** 16:48

## New Gated OUT Beliefs

_None_

## New Negative IN Beliefs

- **ask-sources-db-failure-silently-degrades**: If the `sources_db` SQLite file is missing or corrupt, `_search_source_chunks` catches `OperationalError`/`DatabaseError` and returns empty string, degrading to belief-only mode without user-visible errors.
- **llm-prompt-always-via-stdin**: Prompts are passed to LLM subprocesses via `subprocess.run(input=...)`, never as command-line arguments — avoiding shell injection and argument length limits.
- **parse-review-defaults-to-passing**: `parse_review_response` defaults missing fields to valid=True, sufficient=True, necessary=True, unnecessary_antecedents=[] — the LLM only needs to report failures explicitly; omission means the belief passed.
- **pg-keyerror-includes-missing-node-id**: When `add_node` references a nonexistent node in `sl=` or `unless=`, PgApi raises `KeyError` with the missing node ID in the exception message — tested via `pytest.raises(KeyError, match="ghost")` — enabling callers to identify which reference is dangling.
- **resolve-source-path-db-dir-precedence**: `resolve_source_path` follows a strict precedence chain: `db_dir` resolution takes priority over agent repo resolution, which takes priority over repo-key-split resolution; missing files return `None`.
- **review-parse-defaults-fail-safe**: `parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True`, so a missing or malformed field in LLM output never triggers a false alarm.

## Critical Watch List

### Gated (blocked)

- **storage-is-fully-production-grade-across-backends**: Both storage backends achieve fully production-grade operation — concurrent access optimization (WAL mode, derived FTS5 indexes), equivalent safety guarantees (atomic isolated mutations through backend-appropriate mechanisms), and multi-tenant isolation — when both backends provide complete API coverage.
  - Blocked by: `pgapi-partial-api-coverage` — PgApi implements core operations (add/retract/assert/search/nogood/explain) but defers simulation, dialectics, namespace support, import/export, and maintenance operations to future work
- **verified-correctness-extends-to-all-backends**: Verified production correctness — spanning all belief origins with deterministic state trajectories and full maintenance loop observability for indefinite independent verification — extends identically across both SQLite and PostgreSQL storage backends with no backend-specific safety gaps.
  - Blocked by: `pgapi-partial-api-coverage` — PgApi implements core operations (add/retract/assert/search/nogood/explain) but defers simulation, dialectics, namespace support, import/export, and maintenance operations to future work

### Active Issues

- **all-information-flow-is-fault-tolerant-and-governed**: Every information flow path through the system is simultaneously fault-tolerant (graceful degradation on FTS5 index failures, LLM timeouts, and missing source files) and governed (pure deterministic functions with fixed priority ordering, access-tag authorization gating, and token-budget constraints) — no information retrieval can fail catastrophically, and no information output can bypass authorization or exceed bounds.
- **external-integration-is-hardened-and-boundary-controlled**: LLM integration achieves production-grade robustness (bounded execution, fail-soft error handling, process isolation, fault tolerance) while information boundaries are controlled at every level (authorization gating, budget constraints, defensive ingestion) — the system neither leaks sensitive information outward nor admits unvalidated beliefs inward.
- **llm-prompt-always-via-stdin**: Prompts are passed to LLM subprocesses via `subprocess.run(input=...)`, never as command-line arguments — avoiding shell injection and argument length limits.
- **system-assurance-spans-correction-reliability-and-control**: The system's operational assurance spans three independent dimensions: temporal completeness of self-correction (creation-time contradiction resolution and maintenance-time staleness detection), resource sustainability (bounded token budgets preventing exhaustion), and external controllability (bidirectional interface bounds with defensive ingestion) — no assurance gap exists along any axis.
- **trust-and-information-boundaries-are-comprehensively-enforced**: The system enforces comprehensive boundaries spanning both architecture and information flow: architectural trust boundaries through self-containment and defensive ingestion pipelines ensure no unvalidated data enters the network, while information boundaries through access-tag authorization, token-budget constraints, and bidirectional external surface control ensure no unauthorized or unbounded data leaves it


## Statistics

- **Total beliefs:** 824
- **IN:** 750
- **OUT:** 74
- **Derived:** 448
- **Gated OUT (all):** 13
- **Negative IN (all):** 50
- **New beliefs this run:** 47
- **New gated OUT:** 0
- **New negative IN:** 6

