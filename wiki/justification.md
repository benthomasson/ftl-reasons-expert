# justification

[Back to index](index.md)

A justification in ftl-reasons defines the logical basis for a node's truth value. Drawing from Doyle's 1979 Truth Maintenance System, justifications encode dependency relationships between nodes using a two-level logical structure: conjunction within each justification and disjunction across multiple justifications on the same node.

## Validity Semantics

A justification is valid if and only if all of its antecedent nodes are IN and all of its outlist nodes are OUT (`justification-validity-requires-inlist-in-and-outlist-out`). This single rule is the engine behind retraction cascades, kill-switch behavior, challenges, and supersession — every truth maintenance operation ultimately reduces to repeated evaluation of this condition.

When a node has multiple justifications, the node is IN if *any* of its justifications is valid (`node-in-if-any-justification-valid`, `sl-justification-semantics`). This gives the system disjunctive semantics at the node level: a node with three justifications needs only one to hold. Within each justification, however, the semantics are conjunctive — every antecedent must be IN and every outlist member must be OUT (`network-disjunctive-justification`). Truth computation exploits this by short-circuiting on the first valid justification rather than evaluating all of them (`sl-justification-is-disjunctive`).

Both SL and CP justification types use this same validity rule, evaluated uniformly (`[cp-and-sl-evaluated-identically](other.md#cp-and-sl-evaluated-identically)`). The evaluation function `_justification_valid` is a pure query — no side effects, no logging, no mutations to network state (`justification-valid-is-pure`). This purity, combined with type-uniform evaluation, makes truth computation fully context-free (`justification-evaluation-is-uniform-and-pure`).

## Timing Invariance

Justification evaluation produces identical results regardless of *when* a justification is attached to a node (`justification-timing-is-irrelevant-to-evaluation`). Two paths lead to the same outcome:

- **At creation**: when `add_node` is called with justifications, the node's initial truth value is computed immediately from the current state of its antecedents — a derived node added while its antecedent is OUT starts OUT (`add-node-evaluates-justification-at-insertion`).
- **Post-creation**: `add_justification` triggers identical BFS propagation when a justification is attached later, including restoring OUT nodes whose justifications become newly valid (`add-justification-triggers-propagation`).

This temporal invariance, together with evaluation purity and type uniformity, establishes that justification evaluation is fully context-independent (`justification-evaluation-is-context-independent`).

## Adding Justifications

Adding a justification to an existing node triggers complete multi-dimensional propagation (`add-justification-is-fully-propagating`). Three things happen in a single operation:

1. **Truth value cascade**: if the justification changes the node's truth value, BFS propagation reaches all transitive dependents (`add-justification-triggers-propagation`).
2. **Dependents index update**: the `dependents` set is updated on both antecedent and outlist nodes so that future retraction and restoration propagation can reach the target node (`add-justification-registers-dependents`).
3. **Access tag recomputation**: `access_tags` are recomputed for the target and all its transitive dependents, enabling retroactive tag propagation (`add-justification-propagates-tags-downstream`).

Propagation terminates deterministically — BFS traversal with stop-on-unchanged semantics guarantees the network reaches a fully consistent state (`add-justification-achieves-consistent-propagation`). This consistency holds even when the dependency graph contains dangling references or lifecycle-marked nodes, because propagation safely handles both graph anomalies and node lifecycle states (`justification-addition-is-robust-across-graph-states`).

The operation returns a change dictionary with keys `node_id`, `old_truth_value`, `new_truth_value`, and `changed` (the list of all nodes whose truth value changed), following the system's convention that every mutation reports its effects (`add-justification-returns-change-dict`).

## Constraints and Invariants

The API enforces several structural invariants around justifications:

- **Mandatory specification**: `api.add_justification` raises `ValueError` if none of `sl`, `cp`, or `unless` is provided — at least one justification specification is required (`api-add-justification-requires-justification-arg`).
- **Minimum count**: `Network.remove_justification` refuses to remove the last justification from a derived node, enforced with a `ValueError` (`remove-justification-enforces-minimum-count`). Similarly, `remove_justification()` raises `ValueError` when a node has exactly one justification, directing callers to use `convert_to_premise` or `retract` instead (`network-single-justification-removal-blocked`). This prevents accidental creation of unjustified non-premise nodes.
- **Referential integrity**: in the PostgreSQL backend, `PgApi._validate_refs` checks that all antecedent and outlist node IDs exist before inserting a justification, compensating for JSONB arrays' inability to enforce foreign key constraints (`pgapi-validates-refs-before-justification-insert`).

## Premises and Justifications

Premise nodes — those initially created without justifications — can receive justifications via `add_justification` (`premise-can-receive-justification`). This gives the premise a derived backup path: even if its premise status is retracted, it remains IN as long as the added justification holds. This flexibility means that premise identity is inherently transient — a node's character as a premise or derived belief can change over its lifetime.

The `premise_count` metric in return values reports the maximum antecedent count across a node's justifications, not the total across all justifications. A node in `any_mode` with three single-premise justifications reports a `premise_count` of 1, not 3 (`premise-count-is-per-justification-max`).

## Persistence and Ordering

Justification order matters because justification priority affects truth maintenance — the first valid justification found during evaluation short-circuits further checks. The system preserves insertion order across save/load cycles using SQLite's `AUTOINCREMENT` rowid and `ORDER BY rowid` on read (`justification-order-preserved-via-rowid`, `storage-justification-order-preserved`). Justifications are inserted in list order and loaded in the same sequence, ensuring deterministic evaluation after persistence round-trips.

## Review Semantics

The LLM-based belief review system mirrors the TMS justification semantics: the review prompt instructs the model that a belief is valid if *any* of its justifications is sound (`review-multi-justification-disjunctive`), maintaining consistency between automated truth maintenance and human-readable review.
