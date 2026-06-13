# safe

[Back to index](index.md)

Safety is a pervasive design property in ftl-reasons, enforced at every layer from truth propagation through storage backends to LLM-driven review pipelines. Rather than relying on a single safety mechanism, the system composes multiple guarantees — deterministic termination, fail-safe defaults, mutation gating, and atomic state management — so that no individual failure mode can corrupt the belief network.

## Propagation Safety

Truth propagation is both lifecycle-safe and guaranteed to terminate: retracted nodes are skipped, trigger nodes are never recomputed, BFS traversal prevents stack overflow, and a stop-on-unchanged guard prevents oscillation (propagation-is-safe-and-terminating). These properties make propagation the foundation that higher-level guarantees build on — it directly supports atomic mutation, transitive retraction cascades, and bounded LLM-driven changes.

Crucially, propagation remains correct even when the dependency graph is inconsistent. Missing nodes are skipped with structured warnings rather than crashes, and dangling identifiers are excluded from both the changed and visited sets (propagation-is-safe-under-graph-inconsistency). This graceful degradation composes with the termination and lifecycle-awareness guarantees, so all reachable nodes are evaluated correctly regardless of graph integrity.

## Crash Safety

The TMS core provides crash-free truth maintenance by combining deterministic termination, pure justification evaluation, and conservative failure semantics (tms-core-is-crash-safe). These three properties ensure correct results across all reachable nodes without risking runtime exceptions.

The dialectical challenge/defend system inherits this robustness: challenges are resolved through recursive outlist injection evaluated by the same deterministic terminating propagation, so the adversarial reasoning layer cannot introduce crashes or non-termination (challenge-defense-is-crash-safe).

## Storage Backend Safety

Both storage backends — SQLite and PostgreSQL — provide equivalent safety guarantees through backend-appropriate mechanisms (storage-layer-is-backend-agnostic-and-safe). SQLite achieves atomicity through context-managed load/save cycles, while PostgreSQL uses per-method transactions. This makes safety properties independent of backend choice.

### Hypothetical Reasoning

Both backends support safe what-if reasoning without permanent mutation (both-backends-support-safe-hypothetical-reasoning). The PostgreSQL backend performs real mutations inside a transaction then rolls back, while the in-memory backend uses write-flag gating to discard speculative changes after analysis. Either way, hypothetical exploration cannot corrupt persisted state.

### Initialization

System initialization produces identical belief states regardless of both the initialization path — whether bootstrapping from stored state or deterministic reasoning — and the storage backend, since both backends enforce equivalent safety through their respective mechanisms (initialization-is-safe-and-path-independent-across-backends). This eliminates all bootstrap-time variation.

## Mutation Safety

### Belief Modification

Every bidirectional belief modification — contradiction resolution through traceable backtracking, defeat reversal with guided recovery — operates within richly-governed exception-safe revision (bidirectional-modification-is-richly-governed-and-exception-safe). The system manages state beyond binary truth values, producing metadata-enriched recoverable state changes within a deterministic lifecycle. This ensures that modifications in either direction are fully traceable and reversible.

### Review Pipeline

The belief review pipeline is both scope-limited and mutation-safe by default (review-pipeline-is-scoped-and-mutation-safe). Evaluation is restricted to derived beliefs only — premises are excluded — and auto-retraction is gated behind a dry-run flag. This two-layer protection ensures that review operations cannot inadvertently retract foundational observations.

## Fail-Safe Defaults in Review Parsing

The review response parser employs a fail-safe defaulting strategy: when LLM output is missing fields, `parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True`, ensuring a malformed response never triggers a false alarm (review-parse-defaults-fail-safe, review-parse-defaults-safe). Combined with strict input validation — only JSON arrays are accepted — and normalization to a guaranteed six-key schema, the parser produces uniform structured output regardless of LLM response quality (review-output-is-uniform-and-fail-safe).

## Synchronization Safety

The multi-agent synchronization layer is safe for automated reconciliation: `sync_agent` can be re-run on any schedule without risk, because idempotent execution with cascade structure preservation means repeated synchronization never corrupts outlist-based justification hierarchies or produces accumulating side effects (sync-is-safe-for-automated-reconciliation).
