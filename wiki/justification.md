# justification

[Back to index](index.md)

Justifications are the primary mechanism by which the Truth Maintenance System determines and propagates truth values across a belief network. Every derived node holds one or more justifications, each encoding a set of conditions under which the node should be considered true. The interplay between justifications — their evaluation, addition, removal, and persistence — forms the backbone of non-monotonic reasoning in ftl-reasons.


## Validity and Disjunctive Semantics

A justification is valid if and only if all of its antecedents are IN and all of its outlist nodes are OUT (`justification-validity-requires-inlist-in-and-outlist-out`). This single rule is the engine behind retraction cascades, kill-switch behavior, challenges, and supersession — every truth-maintenance operation ultimately reduces to repeated application of this test.

At the node level, truth is computed disjunctively: a node is IN when *any* of its justifications is valid (`node-in-if-any-justification-valid`, `sl-justification-semantics`). Each individual justification is conjunctive internally — all antecedents must hold and all outlist members must be absent — but a node only needs one valid justification to survive. The `_compute_truth()` implementation reflects this by short-circuiting on the first valid justification rather than evaluating all of them (`sl-justification-is-disjunctive`). This conjunction-within, disjunction-across structure follows Doyle's 1979 SL justification semantics (`network-disjunctive-justification`).

Both SL and CP justification types use this same validity rule, evaluated as a pure function with no side effects (`[cp-and-sl-evaluated-identically](other.md#cp-and-sl-evaluated-identically)`, `justification-evaluation-is-uniform-and-pure`). The evaluation function `_justification_valid` performs no logging, no mutations, and no network state changes (`justification-valid-is-pure`), which grounds several important system-level properties including crash safety, determinism, and the emergent uniformity of truth semantics.


## Context Independence and Timing Invariance

Justification evaluation is fully context-free (`justification-evaluation-is-context-independent`). This property emerges from two independent guarantees: the purity and uniformity of evaluation described above, and the system's temporal invariance with respect to justification attachment.

A justification produces identical truth outcomes regardless of *when* it is attached to a node. When `add_node` is called with justifications, the node's initial truth value is computed immediately from the current state of its antecedents — a derived node added while its antecedent is OUT starts OUT (`add-node-evaluates-justification-at-insertion`). When `add_justification` attaches a justification to an existing node post-creation, it triggers identical propagation logic (`add-justification-triggers-propagation`). The system has no time-of-attachment sensitivity (`justification-timing-is-irrelevant-to-evaluation`).


## Adding Justifications

Adding a justification to an existing node triggers complete multi-dimensional propagation (`add-justification-is-fully-propagating`). Three things happen in a single operation:

1. **Truth value propagation**: if the new justification changes the node's truth value, the change cascades via BFS through all transitive dependents, including restoring OUT nodes whose justifications become newly valid (`add-justification-triggers-propagation`).

2. **Dependents index update**: the `dependents` set is updated on both antecedent and outlist nodes so that future retraction and restoration propagation can reach the target node (`add-justification-registers-dependents`).

3. **Access tag recomputation**: `access_tags` are recomputed for the target and all its transitive dependents, enabling retroactive tag propagation (`add-justification-propagates-tags-downstream`).

This multi-dimensional propagation terminates deterministically — BFS traversal with stop-on-unchanged semantics guarantees that the network reaches a fully consistent state (`add-justification-achieves-consistent-propagation`). The operation remains robust even when the dependency graph contains dangling references or lifecycle-marked nodes, because propagation safely handles both graph anomalies and node lifecycle states (`justification-addition-is-robust-across-graph-states`).

Every mutation reports its effects: `Network.add_justification` returns a dictionary with `node_id`, `old_truth_value`, `new_truth_value`, and a `changed` list of all nodes whose truth value changed (`add-justification-returns-change-dict`).


## Constraints and Invariants

The system enforces several invariants around justification lifecycle. The API layer requires that at least one justification specification (`sl`, `cp`, or `unless`) be provided when calling `add_justification` (`api-add-justification-requires-justification-arg`). At the other end, `remove_justification` refuses to remove the last justification from a derived node, raising a `ValueError` to prevent accidental creation of unjustified non-premise nodes (`remove-justification-enforces-minimum-count`, `network-single-justification-removal-blocked`). Callers who need to strip a node's final justification must explicitly use `convert_to_premise` or `retract` instead.

A premise node — one initially without justifications — can receive a justification via `add_justification`, giving it a derived backup path that keeps it IN even if its premise status is later retracted (`premise-can-receive-justification`). This means premise identity is inherently transient: a node that starts as a premise can acquire derived support over time.

The `premise_count` field in return values reports the maximum antecedent count across a node's justifications, not the total across all of them — a node in any-mode with three single-premise justifications reports a `premise_count` of 1, not 3 (`premise-count-is-per-justification-max`).

For the PostgreSQL backend, `PgApi` validates that all antecedent and outlist node IDs exist before inserting a justification, compensating for the inability of JSONB arrays to enforce foreign key constraints (`pgapi-validates-refs-before-justification-insert`).


## Persistence and Ordering

Justification ordering is semantically significant: it determines which justification is evaluated first during truth computation. The system preserves this ordering across save/load cycles using SQLite's `AUTOINCREMENT` rowid for insertion and `ORDER BY rowid` on read (`justification-order-preserved-via-rowid`, `storage-justification-order-preserved`). This ensures that justification priority — and therefore truth maintenance behavior — is stable across persistence boundaries.


## Review Semantics

The LLM-driven review system mirrors the TMS's disjunctive semantics: the review prompt instructs the model that a belief is valid if *any* of its justifications is sound (`review-multi-justification-disjunctive`), ensuring that automated belief review is consistent with the engine's own evaluation logic.
