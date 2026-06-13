# beliefs

[Back to index](index.md)

### build-tools-section-always-includes-search-beliefs
**Status:** IN

`_build_tools_section` always includes the built-in `search_beliefs` tool in its output regardless of whether MCP bridges are provided — it is the baseline tool present in every ask prompt.


### cluster-beliefs-deterministic-with-seed
**Status:** IN

Given the same beliefs dict, budget, and seed, `cluster_beliefs` produces identical output across calls.

**Supports:** [cluster-selection-is-deterministic-and-budget-exact](deterministic.md#cluster-selection-is-deterministic-and-budget-exact)

### cluster-beliefs-respects-budget
**Status:** IN

`cluster_beliefs` never returns more IDs than the `budget` parameter; each cluster's allocation is individually capped by `min(alloc, len(members))`.


### cluster-beliefs-returns-exact-budget
**Status:** IN

`cluster_beliefs` returns exactly `budget` belief IDs when the input set is larger than the budget, and all items when the input set is smaller.

**Supports:** [cluster-selection-is-deterministic-and-budget-exact](deterministic.md#cluster-selection-is-deterministic-and-budget-exact)

### contradictions-only-checks-in-beliefs
**Status:** IN

`detect_contradictions` filters all input to `truth_value == "IN"` before processing; OUT beliefs are never sent to the LLM for contradiction checking.


### list-clusters-partitions-all-beliefs
**Status:** IN

`list_clusters` assigns every input belief to exactly one cluster with no drops or duplicates — the union of all cluster members equals the input set.


### out-beliefs-imported-as-bare-premises
**Status:** IN

Covered by existing `out-beliefs-imported-without-justifications` which captures the same invariant


### out-beliefs-imported-without-justifications
**Status:** IN

Beliefs that are OUT or STALE in the source are imported with an empty justification list, preventing `recompute_all` from resurrecting them to IN

**Supports:** [import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states)

### parse-beliefs-returns-dicts
**Status:** IN

Too granular — the dict-return pattern is partially covered by `api-functions-return-dicts`, and the specific field list is an implementation detail likely to evolve


### review-only-evaluates-derived-beliefs
**Status:** IN

`review_beliefs` filters out premises (nodes without justifications); only derived beliefs with at least one justification are sent for LLM review.

**Supports:** [review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe)

### review-only-validates-derived-beliefs
**Status:** IN

The review pipeline filters out premises (nodes with empty justifications) before sending anything to the LLM; premises are never submitted for review validation.


### total-invariant-preservation-encompasses-all-beliefs
**Status:** OUT

Total invariant preservation — comprehensive in scope and architecturally grounded — fully encompasses externally-integrated beliefs that achieve integration parity along all quality axes, establishing that the system's invariant regime makes no distinction between internal and external beliefs at any level.

**Depends on:** [external-beliefs-achieve-total-integration](external.md#external-beliefs-achieve-total-integration), [invariant-preservation-is-total](other.md#invariant-preservation-is-total)
**Supports:** [invariant-preservation-is-total-and-self-sustaining](self.md#invariant-preservation-is-total-and-self-sustaining)
