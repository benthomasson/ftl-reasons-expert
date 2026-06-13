# beliefs

[Back to index](index.md)

The belief system in ftl-reasons provides the core data model for tracking claims, their truth values, and their interdependencies. Beliefs flow through several subsystems — clustering, import, review, and contradiction detection — each of which enforces specific invariants about how beliefs are selected, filtered, and evaluated.

## Clustering and Budget Control

When the system needs to select a representative subset of beliefs — for example, to fit within an LLM prompt's context budget — it uses a clustering pipeline. The `list_clusters` function partitions the entire input set so that every belief lands in exactly one cluster with no drops or duplicates (`list-clusters-partitions-all-beliefs`). From there, `cluster_beliefs` selects representatives subject to a strict budget: it never returns more IDs than the `budget` parameter allows, capping each cluster's allocation individually (`cluster-beliefs-respects-budget`). When the input set exceeds the budget, the function returns exactly the requested number of belief IDs; when it's smaller, all items are returned (`cluster-beliefs-returns-exact-budget`).

This selection process is fully deterministic. Given the same beliefs, budget, and random seed, `cluster_beliefs` produces identical output across calls (`cluster-beliefs-deterministic-with-seed`). Together, these properties guarantee that the selection is both deterministic and budget-exact (`[cluster-selection-is-deterministic-and-budget-exact](deterministic.md#cluster-selection-is-deterministic-and-budget-exact)`).

## Import and Truth State Handling

When beliefs are imported from an external source, the system must handle heterogeneous truth states — some beliefs may be IN, while others are OUT or STALE. Beliefs that are not currently held are imported with an empty justification list, which prevents `recompute_all` from inadvertently resurrecting them to IN status (`out-beliefs-imported-without-justifications`). This invariant is a key part of how the import pipeline handles mixed truth states (`[import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states)`).

## Review Pipeline

The review subsystem sends beliefs to an LLM for validation, but it is carefully scoped. Only derived beliefs — those with at least one justification — are eligible for review. Premises, which are direct observations with no justifications, are filtered out before anything reaches the LLM (`review-only-evaluates-derived-beliefs`, `review-only-validates-derived-beliefs`). This scoping ensures the review pipeline remains focused on reasoned conclusions rather than foundational observations, and that it does not inadvertently mutate the premise base (`[review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe)`).

## Contradiction Detection

The contradiction detection system applies a similar filtering strategy. `detect_contradictions` restricts its input to beliefs with a truth value of IN before sending anything to the LLM for analysis (`contradictions-only-checks-in-beliefs`). OUT beliefs are excluded entirely, avoiding false positives from retracted claims that may appear to conflict with currently held ones.

## Tool Integration

The belief system is surfaced to LLM interactions through a tool-building layer. The `_build_tools_section` function always includes the built-in `search_beliefs` tool in its output, regardless of whether any MCP bridges are configured (`build-tools-section-always-includes-search-beliefs`). This ensures that belief search is the baseline capability available in every ask prompt, providing a consistent interface for querying the knowledge base.
