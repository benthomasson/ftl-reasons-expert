# justification

[Back to index](index.md)

### add-justification-achieves-consistent-propagation
**Status:** IN

Adding a justification to an existing node produces a fully consistent network state through guaranteed-terminating multi-dimensional propagation: truth values cascade via BFS through dependents, the reverse index is updated, and access tags recompute transitively — all within a single operation whose termination is guaranteed by BFS traversal and stop-on-unchanged semantics.

**Depends on:** [add-justification-is-fully-propagating](justification.md#add-justification-is-fully-propagating), [propagation-terminates-deterministically](other.md#propagation-terminates-deterministically)
**Supports:** [justification-addition-is-robust-across-graph-states](justification.md#justification-addition-is-robust-across-graph-states)

### add-justification-is-fully-propagating
**Status:** IN

Adding a justification triggers complete multi-dimensional propagation: truth values cascade through dependents via BFS, the dependents reverse index is updated on both antecedent and outlist nodes, and access tags flow downstream transitively through all dependent chains.

**Depends on:** [add-justification-propagates-tags-downstream](justification.md#add-justification-propagates-tags-downstream), [add-justification-registers-dependents](justification.md#add-justification-registers-dependents), [add-justification-triggers-propagation](justification.md#add-justification-triggers-propagation)
**Supports:** [add-justification-achieves-consistent-propagation](justification.md#add-justification-achieves-consistent-propagation)

### add-justification-propagates-tags-downstream
**Status:** IN

Calling `add_justification` on an existing node recomputes `access_tags` for the target and all its transitive dependents, enabling retroactive tag propagation.

**Supports:** [add-justification-is-fully-propagating](justification.md#add-justification-is-fully-propagating)

### add-justification-registers-dependents
**Status:** IN

`add_justification` updates the `dependents` set on both antecedent and outlist nodes so that future retraction/restoration propagation reaches the target node.

**Supports:** [add-justification-is-fully-propagating](justification.md#add-justification-is-fully-propagating)

### add-justification-returns-change-dict
**Status:** IN

`Network.add_justification` returns a dict with keys `node_id`, `old_truth_value`, `new_truth_value`, and `changed` (list of all nodes whose truth value changed).

**Supports:** [every-mutation-reports-its-effects](other.md#every-mutation-reports-its-effects)

### add-justification-triggers-propagation
**Status:** IN

Adding a justification that changes a node's truth value triggers BFS propagation that cascades to all transitive dependents, including restoring OUT nodes whose justifications become valid.

**Supports:** [add-justification-is-fully-propagating](justification.md#add-justification-is-fully-propagating), [justification-timing-is-irrelevant-to-evaluation](justification.md#justification-timing-is-irrelevant-to-evaluation)

### add-node-evaluates-justification-at-insertion
**Status:** IN

When `add_node` is called with justifications, the node's initial truth value is computed immediately from the current state of its antecedents — a derived node added when its antecedent is OUT starts OUT

**Supports:** [justification-timing-is-irrelevant-to-evaluation](justification.md#justification-timing-is-irrelevant-to-evaluation)

### api-add-justification-requires-justification-arg
**Status:** IN

`api.add_justification` raises `ValueError` if none of `sl`, `cp`, or `unless` is provided — at least one justification specification is mandatory.

**Supports:** [api-enforces-typed-preconditions](other.md#api-enforces-typed-preconditions)

### justification-addition-is-robust-across-graph-states
**Status:** IN

Adding a justification to an existing node achieves fully consistent multi-dimensional propagation — truth values, dependents index, and access tags — even when the dependency graph contains dangling references or lifecycle-marked nodes, because propagation safely handles both graph anomalies and node lifecycle states.

**Depends on:** [add-justification-achieves-consistent-propagation](justification.md#add-justification-achieves-consistent-propagation), [propagation-is-safe-under-graph-inconsistency](safe.md#propagation-is-safe-under-graph-inconsistency)
**Supports:** [all-mutations-preserve-integrity-under-adverse-conditions](other.md#all-mutations-preserve-integrity-under-adverse-conditions), [all-truth-changes-are-topology-complete-and-robust](topology.md#all-truth-changes-are-topology-complete-and-robust)

### justification-evaluation-is-context-independent
**Status:** IN

Justification evaluation produces identical results regardless of evaluation context: it is pure (no side effects), uniform across types (SL/CP use the same validity rule), and temporally invariant (attaching a justification at creation or later yields the same truth outcome) — making truth computation fully context-free

**Depends on:** [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure), [justification-timing-is-irrelevant-to-evaluation](justification.md#justification-timing-is-irrelevant-to-evaluation)
**Supports:** [evaluation-is-uniformly-context-and-origin-agnostic](other.md#evaluation-is-uniformly-context-and-origin-agnostic)

### justification-evaluation-is-uniform-and-pure
**Status:** IN

All justification types (SL and CP) use the same validity rule (antecedents IN, outlist OUT), evaluated as a pure function with no side effects

**Depends on:** [cp-and-sl-evaluated-identically](other.md#cp-and-sl-evaluated-identically), [justification-valid-is-pure](justification.md#justification-valid-is-pure), [justification-validity-requires-inlist-in-and-outlist-out](justification.md#justification-validity-requires-inlist-in-and-outlist-out)
**Supports:** [evaluation-purity-grounds-agnosticism-and-minimality](other.md#evaluation-purity-grounds-agnosticism-and-minimality), [justification-evaluation-is-context-independent](justification.md#justification-evaluation-is-context-independent), [justification-timing-is-irrelevant-to-evaluation](justification.md#justification-timing-is-irrelevant-to-evaluation), [tms-core-is-crash-safe](safe.md#tms-core-is-crash-safe), [tms-core-is-deterministic-and-conservative](deterministic.md#tms-core-is-deterministic-and-conservative), [truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)

### justification-order-preserved-by-rowid
**Status:** IN

Already exists as `justification-order-preserved-via-rowid`


### justification-order-preserved-via-rowid
**Status:** IN

Justification insertion order is preserved across save/load cycles using `AUTOINCREMENT` rowid and `ORDER BY rowid` on read, which matters because justification priority affects truth maintenance.

**Supports:** [persistence-round-trip-is-lossless](other.md#persistence-round-trip-is-lossless)

### justification-timing-is-irrelevant-to-evaluation
**Status:** IN

Justification evaluation produces identical truth semantics regardless of when a justification is attached: add_node evaluates justifications immediately at insertion, and add_justification triggers identical propagation when attached post-creation — the system has no time-of-attachment sensitivity.

**Depends on:** [add-justification-triggers-propagation](justification.md#add-justification-triggers-propagation), [add-node-evaluates-justification-at-insertion](justification.md#add-node-evaluates-justification-at-insertion), [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure)
**Supports:** [justification-evaluation-is-context-independent](justification.md#justification-evaluation-is-context-independent)

### justification-valid-is-pure
**Status:** IN

`_justification_valid` is a pure query with no side effects, no logging, and no mutations to network state

**Supports:** [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure)

### justification-validity-requires-inlist-in-and-outlist-out
**Status:** IN

A justification is valid iff all antecedents are IN and all outlist nodes are OUT; this single rule drives retraction cascades, kill-switch behavior, challenges, and supersession

**Supports:** [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure)

### network-any-mode-justification
**Status:** IN

Duplicates existing belief `node-in-if-any-justification-valid`.


### network-disjunctive-justification
**Status:** IN

A node is IN if ANY of its justifications is valid (disjunction); each individual justification requires ALL antecedents IN and ALL outlist members OUT (conjunction) — the SL justification semantics from Doyle's 1979 paper.


### network-single-justification-removal-blocked
**Status:** IN

`remove_justification()` raises `ValueError` when a node has exactly one justification, forcing callers to use `convert_to_premise` or `retract` instead — preventing accidental creation of unjustified non-premise nodes.


### node-in-if-any-justification-valid
**Status:** IN

A Node is IN when at least one of its Justifications is valid; a Justification is valid when all antecedents are IN and all outlist nodes are OUT (disjunctive over justifications, conjunctive within each).

**Supports:** [truth-is-disjunctive-over-conjunctive-rules](other.md#truth-is-disjunctive-over-conjunctive-rules)

### pgapi-validates-refs-before-justification-insert
**Status:** IN

PgApi's `_validate_refs` checks all antecedent and outlist node IDs exist in `rms_nodes` before inserting a justification, providing application-level referential integrity that compensates for JSONB arrays' inability to enforce foreign key constraints.

**Supports:** [pgapi-enforces-referential-integrity-bidirectionally](other.md#pgapi-enforces-referential-integrity-bidirectionally)

### premise-can-receive-justification
**Status:** IN

A premise node (initially no justifications) can receive a justification via `add_justification`, giving it a derived backup path that keeps it IN even if its premise status is retracted.

**Supports:** [premise-identity-is-inherently-transient](other.md#premise-identity-is-inherently-transient)

### premise-count-is-per-justification-max
**Status:** IN

`premise_count` in the return value reports the maximum antecedent count across the node's justifications, not the total across all justifications (any_mode with 3 premises returns 1, not 3).


### remove-justification-enforces-minimum-count
**Status:** IN

`Network.remove_justification` refuses to remove the last justification from a derived node — a derived node must retain at least one justification, enforced with `ValueError("only one justification")`


### review-multi-justification-disjunctive
**Status:** IN

The review prompt instructs the LLM that a belief is valid if ANY of its justifications is sound, matching Doyle's TMS disjunctive support semantics.


### sl-justification-is-disjunctive
**Status:** IN

A node is IN if *any* of its justifications is valid (disjunctive semantics); `_compute_truth()` short-circuits on the first valid justification rather than requiring all justifications to hold.


### sl-justification-semantics
**Status:** IN

An SL justification is valid iff ALL antecedents are IN AND ALL outlist nodes are OUT; a node is IN iff ANY of its justifications is valid (conjunction within a justification, disjunction across justifications).

**Supports:** [truth-is-disjunctive-over-conjunctive-rules](other.md#truth-is-disjunctive-over-conjunctive-rules)

### storage-justification-order-preserved
**Status:** IN

Justifications are inserted in list order and loaded via `ORDER BY rowid`, preserving the ordering that determines which justification is evaluated first during truth computation.

