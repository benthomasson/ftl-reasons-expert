# beliefs

[Back to index](index.md)

The belief system provides structured mechanisms for organizing, reviewing, importing, and validating beliefs within the knowledge base. Its design enforces determinism, budget constraints, and clear scoping rules that govern which beliefs participate in each operation.

## Clustering

Beliefs can be grouped into thematic clusters for navigation and budget-constrained selection. The `cluster_beliefs` function partitions the full input set so that every belief lands in exactly one cluster with no drops or duplicates (`list-clusters-partitions-all-beliefs`). When the input set exceeds the requested budget, the function returns exactly `budget` belief IDs; when it is smaller, all items are returned (`cluster-beliefs-returns-exact-budget`). Each cluster's allocation is individually capped to prevent over-selection from any single group (`cluster-beliefs-respects-budget`).

Clustering is fully deterministic: given the same beliefs, budget, and random seed, `cluster_beliefs` produces identical output across calls (`cluster-beliefs-deterministic-with-seed`). Together, the exact-budget and deterministic properties ensure that cluster-based selection is both predictable and precisely sized (`[cluster-selection-is-deterministic-and-budget-exact](deterministic.md#cluster-selection-is-deterministic-and-budget-exact)`).

## Contradiction Detection

The contradiction detection pipeline operates exclusively on beliefs that are currently held. `detect_contradictions` filters all input to `truth_value == "IN"` before processing, so OUT beliefs are never sent to the LLM for analysis (`contradictions-only-checks-in-beliefs`). This scoping avoids false positives from retracted beliefs that may contain stale or already-resolved contradictions.

## Review Pipeline

The review system is similarly scoped. `review_beliefs` filters out premises — nodes without justifications — before sending anything to the LLM (`review-only-evaluates-derived-beliefs`, `review-only-validates-derived-beliefs`). Only derived beliefs with at least one justification are submitted for review validation. This design reflects the principle that premises represent direct observations and are not candidates for automated re-evaluation; review focuses on whether derived conclusions still follow from their antecedents.

## Import Handling

When beliefs are imported from an external source, the system preserves their truth state faithfully. Beliefs that are OUT or STALE in the source are imported with an empty justification list, which prevents `recompute_all` from resurrecting them to IN (`out-beliefs-imported-without-justifications`). This invariant ensures that import does not inadvertently revive retracted beliefs and supports correct handling of heterogeneous truth states across sources (`[import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states)`).

## Tool Integration

The `_build_tools_section` function always includes the built-in `search_beliefs` tool in its output, regardless of whether MCP bridges are provided (`build-tools-section-always-includes-search-beliefs`). This makes `search_beliefs` the baseline capability available in every ask prompt, ensuring that belief querying is never gated on external tool availability.

## Invariant Scope

An earlier belief held that invariant preservation was total and made no distinction between internal and external beliefs at any level (`total-invariant-preservation-encompasses-all-beliefs`, OUT). This claim depended on external beliefs achieving full integration parity and on invariant preservation being comprehensive in scope. It has since been retracted, suggesting that the boundary between internal and external beliefs is more nuanced than a blanket equivalence would imply.
