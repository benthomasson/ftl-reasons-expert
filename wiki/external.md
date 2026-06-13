# external

[Back to index](index.md)

The ftl-reasons system interacts with the external world along two distinct axes: its dependency footprint (what it requires from outside) and its belief integration boundaries (what it accepts from outside). The project maintains zero external runtime dependencies while providing layered mechanisms for ingesting, reconciling, and lifecycle-managing beliefs that originate beyond the local network.

## Zero External Dependencies

The project enforces zero external coupling at both the packaging and implementation levels (`project-has-zero-external-coupling`). The `Network` class imports only Python standard library modules — `collections.deque` and `datetime` — plus the project's own data types (`network-has-zero-external-dependencies`, `network-no-external-deps`). The `pyproject.toml` declares an empty `dependencies` list, eliminating supply-chain risk entirely (`zero-runtime-dependencies`). This self-containment is a deliberate architectural choice: the core TMS engine carries no third-party baggage.

## External Belief Integration

External beliefs enter the system through two pathways: LLM-driven derivation and multi-agent import. Both are currently held to be safely integrated (`all-external-inputs-safely-integrated`): defensive validation with retraction guards bounds LLM output, while complete reconciliation with dual modes and heterogeneous truth handling manages agent imports (`import-provides-complete-reconciliation`, `llm-driven-mutations-are-safely-bounded`).

All LLM-facing operations execute through subprocess isolation with environment scrubbing (`all-external-execution-is-subprocess-isolated`). The `derive` command shells out to CLI binaries rather than importing provider SDKs — achieving both provider agnosticism and a security boundary in a single architectural choice. Both `derive` and `ask` independently strip the `CLAUDECODE` environment variable to prevent recursive invocation (`derive-uses-subprocess-not-sdk`, `llm-subprocess-isolation-prevents-recursion`).

A notable edge case arises during belief import: when a claim's `depends_on` references are all absent from the beliefs file, it receives no justifications and enters the network as a premise, held IN by default (`external-deps-become-premises`). This means external beliefs whose dependency context is missing are silently promoted to foundational status rather than rejected.

## External Belief Lifecycle

The system manages external beliefs across their full lifecycle (`external-belief-lifecycle-is-complete`). Import and sync provide dual reconciliation modes with heterogeneous truth state handling and namespace auto-wiring, while staleness checking detects source drift for CI gating — beliefs are tracked from initial ingestion through ongoing validity monitoring (`staleness-is-conservative-ci-gate`).

This lifecycle management is further automated: idempotent sync with cascade preservation and full source staleness coverage enables zero-touch external belief management (`external-lifecycle-is-complete-and-automatically-maintained`). The combination operates within a metadata-enabled, source-grounded governance framework that extends beyond binary truth values to structured lifecycle state (`external-lifecycle-operates-within-rich-governance`).

## Retracted Safety Claims

A significant cluster of higher-level beliefs about external integration safety have been retracted. The claim that both external pathways are defense-in-depth pipelines that "cannot corrupt the host network" is OUT (`all-external-belief-paths-are-safely-bounded`), as are the stronger claims that external beliefs achieve full invariant equivalence with internal beliefs (`external-beliefs-are-invariant-equivalent`), total integration along all quality axes (`external-beliefs-achieve-total-integration`), and complete bidirectional boundary control (`external-surface-is-fully-controlled`).

These retractions cascade from dependencies on claims about multi-agent safety spanning all layers (`multi-agent-safety-spans-all-layers`) and architectural enforcement properties that no longer hold. The foundational integration and lifecycle mechanisms remain intact — it is the comprehensive safety *guarantees* layered atop them that have been withdrawn. The system still integrates external beliefs through validated pipelines; it simply no longer claims that those pipelines are provably airtight across every possible dimension.

The retraction of `ask-degrades-across-all-external-dependencies` similarly reflects a gap: while the `ask` module handles LLM binary failures and FTS5 index issues gracefully, the claim of complete graceful degradation across *all* external dependencies lost support when the sources-database fault tolerance claim (`ask-sources-db-failure-silently-degrades`) was retracted.

## Relationship to Other Topics

External belief management connects closely to the [import](import.md) subsystem (which provides the reconciliation infrastructure), [LLM integration](llm.md) (which provides the derivation pipeline), and the [agent](agent.md) subsystem (which manages multi-agent federation). The [lifecycle](lifecycle.md) and [governance](governance.md) pages cover the broader frameworks within which external belief management operates. The zero-dependency property contributes to the system's overall [self-containment](self.md) architecture.
