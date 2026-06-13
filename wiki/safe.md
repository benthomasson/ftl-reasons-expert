# safe

[Back to index](index.md)

Safety in ftl-reasons refers to the property that operations complete without crashes, data corruption, or unbounded resource consumption — regardless of input quality, graph state, or storage backend. Safety guarantees are layered: the TMS core provides crash-free propagation, storage backends enforce atomic isolated mutations, and the LLM review pipeline defaults to fail-safe parsing. Several higher-level claims that composed these guarantees into universal safety across all mutation sources have been retracted, reflecting a more cautious current understanding.

## Core Propagation Safety

The foundation of the system's safety story is deterministic, terminating truth propagation. Propagation is both lifecycle-safe and guaranteed to terminate: retracted nodes are skipped, trigger nodes are never recomputed, BFS prevents stack overflow, and stop-on-unchanged prevents oscillation (`propagation-is-safe-and-terminating`). This single belief supports a wide range of downstream guarantees — it is a dependency for atomic mutations, transitive retraction cascades, LLM mutation bounding, and contradiction resolution.

Propagation also achieves correctness under graph inconsistency. When the dependency graph contains dangling references, missing nodes are skipped with structured warnings rather than crashing, and dangling IDs are excluded from the changed and visited sets (`propagation-is-safe-under-graph-inconsistency`). This graceful degradation composes with the termination guarantees for all reachable nodes.

Together, these properties support the top-level claim that the TMS core is crash-safe: deterministic termination, pure evaluation, and conservative failure semantics ensure correct results across all reachable nodes (`tms-core-is-crash-safe`). The dialectical challenge/defend system inherits this safety — it reaches correct truth states through recursive outlist injection evaluated by the same deterministic terminating propagation (`challenge-defense-is-crash-safe`).

## Storage Backend Safety

Safety properties are independent of backend choice. Both SQLite and PostgreSQL provide equivalent guarantees through backend-appropriate mechanisms: atomic isolated mutations via context-managed load/save (SQLite) or per-method transactions (PgApi), and safe hypothetical reasoning via write-flag gating (SQLite) or transaction rollback (PgApi) (`storage-layer-is-backend-agnostic-and-safe`). The hypothetical reasoning guarantee means both backends enable what-if analysis without permanent mutation (`both-backends-support-safe-hypothetical-reasoning`).

This backend-agnostic safety extends to initialization. System initialization produces identical belief states regardless of both initialization path and storage backend, eliminating all bootstrap-time variation (`initialization-is-safe-and-path-independent-across-backends`).

## Review Pipeline Safety

The LLM-driven belief review pipeline has multiple layers of fail-safe behavior. At the parsing level, `parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True`, so a missing or malformed field in LLM output never triggers a false alarm (`review-parse-defaults-fail-safe`, `review-parse-defaults-safe`). The parser accepts only JSON arrays as valid input and normalizes every result to a guaranteed six-key schema, producing uniform structured output regardless of LLM response quality (`review-output-is-uniform-and-fail-safe`).

At the pipeline level, review restricts evaluation to derived beliefs only — premises are excluded — and gates auto-retraction behind the dry-run flag, ensuring operations are scope-limited and mutation-safe by default (`review-pipeline-is-scoped-and-mutation-safe`).

## Multi-Agent Synchronization Safety

The `sync_agent` function is safe for automated reconciliation: idempotent execution with cascade structure preservation means repeated synchronization never corrupts outlist-based justification hierarchies or produces accumulating side effects (`sync-is-safe-for-automated-reconciliation`).

## Bidirectional Modification Safety

Bidirectional belief modifications — contradiction resolution through backtracking and defeat reversal with guided recovery — operate within richly-governed exception-safe revision, producing metadata-enriched recoverable state changes within a deterministic lifecycle (`bidirectional-modification-is-richly-governed-and-exception-safe`).

## Retracted Universal Safety Claims

Several ambitious claims that composed individual safety guarantees into universal coverage have been retracted. The claim that all belief modification paths are operationally safe (`all-belief-modification-paths-are-operationally-safe`, OUT) depended on dialectical transformation being operationally safe (`dialectical-transformation-is-operationally-safe`, OUT), which itself combined semantic and operational safety claims about the irreversible premise-to-justified conversion during challenge.

This retraction cascaded upward. The claim that every mutation source is simultaneously safe and semantically uniform (`all-mutation-sources-are-safe-and-uniform`, OUT) lost its foundation, as did the derived claim of safe universal revisability — that any mutation source can safely revise any belief without restrictions (`safe-universal-revisability`, OUT). The claim that determinism enables safe dialectical extension without dedicated safety machinery (`determinism-enables-safe-dialectical-extension`, OUT) was also retracted.

The broader operational profile claim — that the system achieves both safety and assurance without resource trade-offs (`operational-profile-is-safe-assured-and-resource-bounded`, OUT) — is similarly retracted.

The current state of the belief network reflects a layered understanding: individual safety mechanisms (propagation termination, atomic storage, fail-safe parsing) remain well-supported, but the composed claim that these mechanisms seamlessly cover every mutation path has been withdrawn. The core is crash-safe; the universal composition story is under revision.
