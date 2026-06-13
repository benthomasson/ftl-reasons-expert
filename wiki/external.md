# external

[Back to index](index.md)

The concept of "external" in ftl-reasons operates along two distinct axes: the project's relationship to external software dependencies, and its handling of externally-sourced beliefs within the truth maintenance network. In both cases, the design philosophy emphasizes isolation, safety, and self-containment.

## Zero External Dependencies

The ftl-reasons project enforces a strict zero-external-coupling policy at every level. The core `Network` class imports only Python standard library modules (`collections.deque`, `datetime`) alongside the project's own data types — `Node`, `Justification`, and `Nogood` (network-no-external-deps, network-has-zero-external-dependencies). This isn't incidental; the packaging configuration in `pyproject.toml` declares an empty `dependencies` list, making the constraint explicit at the distribution level as well (project-has-zero-external-coupling). The result is the elimination of supply-chain risk and simplified deployment — the system carries no transitive dependency burden.

## External Belief Lifecycle

While the codebase itself avoids external dependencies, the belief network is designed to *accept* beliefs from external sources and manage them across their full lifecycle. Import and sync operations provide dual reconciliation modes with heterogeneous truth state handling and automatic namespace wiring, while staleness checking detects source drift suitable for CI gating (external-belief-lifecycle-is-complete). This lifecycle management extends to automated ongoing maintenance: idempotent sync preserves existing cascade structures while providing full source staleness coverage, enabling what amounts to zero-touch external belief management (external-lifecycle-is-complete-and-automatically-maintained).

The governance layer adds further structure. External beliefs operate within a metadata-enabled, source-grounded framework where creation, reconciliation, staleness detection, and maintenance all respect structured lifecycle state beyond simple binary truth values (external-lifecycle-operates-within-rich-governance).

### Import Semantics

When external beliefs arrive whose `depends_on` references are all absent from the existing beliefs file, they receive no justifications and are added as premises — nodes that are IN by default (external-deps-become-premises). This ensures that imported claims can stand on their own even when their original justification context isn't available in the local network.

## Safe Integration of External Inputs

External inputs enter the system through two channels, each with its own safety mechanism. LLM-derived beliefs undergo defensive validation with retraction guards that bound the impact of model output, while agent-imported beliefs pass through complete reconciliation with dual modes and heterogeneous truth handling (all-external-inputs-safely-integrated). Together, these mechanisms ensure that the belief network remains consistent regardless of the quality or provenance of incoming data.

All LLM-facing operations execute through subprocess isolation with environment scrubbing rather than direct SDK imports. The `derive` command shells out to CLI binaries (achieving provider agnosticism), and both `derive` and `ask` independently strip the `CLAUDECODE` environment variable to prevent recursive invocation (all-external-execution-is-subprocess-isolated). This single architectural choice — subprocess isolation — simultaneously achieves two independent safety goals: provider independence and recursion prevention.
