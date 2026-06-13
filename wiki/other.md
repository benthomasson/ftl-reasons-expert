# Other

[Back to index](index.md)

### absence-has-consistent-dual-semantics
**Status:** IN

Absence has deliberate, defined semantics throughout the system at two levels: structural absence (no justifications) creates premise behavior via vacuous truth over empty lists, while referential absence (missing nodes) follows conservative/permissive asymmetry — both forms of absence produce predictable behavior rather than errors or undefined state.

**Depends on:** [missing-nodes-have-asymmetric-fail-semantics](other.md#missing-nodes-have-asymmetric-fail-semantics), [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence)
**Supports:** [absence-and-outlist-form-complete-negative-semantics](complete.md#absence-and-outlist-form-complete-negative-semantics), [all-semantic-edge-cases-are-uniform](other.md#all-semantic-edge-cases-are-uniform)

### access-control-enforced-at-read-not-write
**Status:** IN

Access control (`_is_visible`) is enforced at read/query boundaries (`show_node`, `explain_node`, `trace_assumptions`) via `PermissionError`, but write operations (`add_node`, `retract_node`, `assert_node`) do not check visibility.

**Supports:** [access-control-is-transitive-subset-gated](other.md#access-control-is-transitive-subset-gated)

### access-control-is-transitive-subset-gated
**Status:** IN

Access control enforces transitive subset-based authorization: visibility requires the caller's tags to be a superset of the node's tags, derived nodes inherit the sorted union of all ancestor tags transitively, and enforcement occurs at read boundaries only — write operations are unrestricted.

**Depends on:** [access-control-enforced-at-read-not-write](other.md#access-control-enforced-at-read-not-write), [access-tags-subset-gate](other.md#access-tags-subset-gate), [tag-inheritance-is-transitive-union](other.md#tag-inheritance-is-transitive-union)
**Supports:** [agent-isolation-spans-identity-and-authorization](agent.md#agent-isolation-spans-identity-and-authorization), [information-flow-is-authorization-and-budget-controlled](other.md#information-flow-is-authorization-and-budget-controlled), [information-pipeline-is-resource-governed-and-access-controlled](other.md#information-pipeline-is-resource-governed-and-access-controlled)

### access-tags-subset-gate
**Status:** IN

A tagged node is visible only when its `access_tags` are a subset of the caller's `visible_to` set; partial overlap (intersection without containment) is insufficient for access.

**Supports:** [access-control-is-transitive-subset-gated](other.md#access-control-is-transitive-subset-gated)

### access-tags-union-inheritance
**Status:** IN

A derived node's `access_tags` is the sorted, deduplicated union of all antecedent tags across all justifications, merged with any explicit tags on the node itself.


### active-inactive-relay-pair
**Status:** IN

Each imported agent gets exactly two infrastructure nodes: `agent:active` (premise, starts IN) and `agent:inactive` (derived via SL with `outlist=[active_id]`, starts OUT); every imported belief includes `inactive_id` in its outlist

**Supports:** [agent-isolation-through-namespace-and-relay](agent.md#agent-isolation-through-namespace-and-relay)

### active-not-in-antecedents
**Status:** IN

The `active` premise is deliberately excluded from imported beliefs' antecedents; if it were an antecedent, it would provide a second always-valid justification path that defeats per-belief retraction semantics

**Supports:** [agent-isolation-through-namespace-and-relay](agent.md#agent-isolation-through-namespace-and-relay)

### active-premise-not-in-antecedents
**Status:** IN

Covered by existing `active-not-in-antecedents` and `kill-switch-uses-outlist-not-antecedent`


### add-nogood-always-records
**Status:** IN

`add_nogood` appends a `Nogood` record unconditionally before checking whether the contradiction is active, so nogoods are preserved even when not all member nodes are currently IN

**Supports:** [contradiction-resolution-is-minimal-disruption](other.md#contradiction-resolution-is-minimal-disruption), [nogood-resolution-maintains-consistent-ids](other.md#nogood-resolution-maintains-consistent-ids)

### add-nogood-fallback-uses-dependent-count
**Status:** IN

When `find_culprits` returns no candidates (all nogood nodes are premises with no justification chains), the fallback retracts the nogood member with the fewest direct dependents

**Supports:** [contradiction-resolution-is-minimal-disruption](other.md#contradiction-resolution-is-minimal-disruption)

### add-nogood-retraction-prefers-least-entrenched
**Status:** IN

The primary retraction path traces back through justification chains to premises and chooses the one with the lowest entrenchment score (speculative assumptions before evidence-backed observations)

**Supports:** [contradiction-resolution-is-minimal-disruption](other.md#contradiction-resolution-is-minimal-disruption)

### all-api-functions-support-pg-dispatch
**Status:** IN

Every public API function (`export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`) accepts `pg_conninfo`/`project_id` kwargs and routes to the corresponding `PgApi` method


### all-belief-inspection-is-non-mutating-and-fault-tolerant
**Status:** OUT

All belief inspection operations — quality review (read-only with fault-tolerant batch handling), staleness checking (conservative non-mutating CI gate), and negative classification (defensively bounded with graceful degradation) — are uniformly non-mutating and fault-tolerant, ensuring observation never perturbs the observed system.

**Depends on:** [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded), [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)
**Supports:** [fault-tolerance-spans-inspection-through-self-correction](spans.md#fault-tolerance-spans-inspection-through-self-correction)

### all-corrections-are-reliable-and-auditable
**Status:** OUT

Every belief correction — whether intentional (dialectical challenge/defend) or automated (contradiction-triggered backtracking, staleness-driven revision) — is both reliable (reaching correct truth state through complete mechanisms) and auditable (maintaining full traceable history of nogoods and resolutions).

**Depends on:** [dispute-resolution-is-complete-and-reliable](complete.md#dispute-resolution-is-complete-and-reliable), [self-correction-has-complete-traceable-history](self.md#self-correction-has-complete-traceable-history)
**Supports:** [corrections-span-all-origins-with-full-auditability](other.md#corrections-span-all-origins-with-full-auditability)

### all-defeat-mechanisms-are-reversible
**Status:** IN

Every outlist-based defeat operation (challenge, kill-switch, supersession) is inherently reversible because outlist semantics flip truth values without deleting nodes

**Depends on:** [challenge-uses-outlist-mechanism](other.md#challenge-uses-outlist-mechanism), [defend-is-recursive-challenge](other.md#defend-is-recursive-challenge), [kill-switch-cascade-is-reversible](other.md#kill-switch-cascade-is-reversible), [supersession-is-reversible](other.md#supersession-is-reversible)
**Supports:** [agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained), [defeat-reversal-propagates-automatically](other.md#defeat-reversal-propagates-automatically), [defeat-reversal-with-guided-recovery](other.md#defeat-reversal-with-guided-recovery), [defeat-reversals-are-lifecycle-governed](lifecycle.md#defeat-reversals-are-lifecycle-governed), [dialectical-defeat-is-reversible-but-identity-is-permanent](other.md#dialectical-defeat-is-reversible-but-identity-is-permanent), [non-monotonic-system-is-single-reversible-primitive](system.md#non-monotonic-system-is-single-reversible-primitive)

### all-exceptions-are-safely-handled
**Status:** IN

The system handles both automated exceptions (contradictions triggering dependency-directed backtracking) and manual challenges (irreversible premise-to-justified transformation via dialectics) safely — exceptional conditions from any origin reach correct truth states through deterministic propagation, and no exception pathway corrupts network consistency.

**Depends on:** [dialectical-transformation-is-fully-reliable](other.md#dialectical-transformation-is-fully-reliable), [tms-handles-all-conditions-safely](other.md#tms-handles-all-conditions-safely)
**Supports:** [exception-safety-spans-tms-and-source-lifecycle](spans.md#exception-safety-spans-tms-and-source-lifecycle), [revision-is-exception-safe-and-recoverable](revision.md#revision-is-exception-safe-and-recoverable), [system-is-self-correcting-and-exception-proof](system.md#system-is-self-correcting-and-exception-proof)

### all-identifiers-are-durable-across-persistence-boundaries
**Status:** OUT

All system-generated identifiers are not only deterministic and collision-free at creation time but also durable across persistence boundaries — nogood IDs survive save/load cycles via the network_meta table's high-water mark, ensuring the collision-free property is maintained across the system's entire operational lifetime rather than just within a single session

**Depends on:** [all-identification-is-deterministic-and-collision-free](deterministic.md#all-identification-is-deterministic-and-collision-free), [nogood-ids-are-durable-and-collision-free](other.md#nogood-ids-are-durable-and-collision-free)
**Supports:** [references-are-durable-across-persistence-and-evolution](other.md#references-are-durable-across-persistence-and-evolution)

### all-information-flow-is-fault-tolerant-and-governed
**Status:** OUT

Every information flow path through the system is simultaneously fault-tolerant (graceful degradation on FTS5 index failures, LLM timeouts, and missing source files) and governed (pure deterministic functions with fixed priority ordering, access-tag authorization gating, and token-budget constraints) — no information retrieval can fail catastrophically, and no information output can bypass authorization or exceed bounds.

**Depends on:** [all-information-retrieval-is-fault-tolerant](other.md#all-information-retrieval-is-fault-tolerant), [all-output-is-deterministic-authorized-and-resilient](deterministic.md#all-output-is-deterministic-authorized-and-resilient)
**Supports:** [user-interface-is-verified-and-fault-tolerant](other.md#user-interface-is-verified-and-fault-tolerant)

### all-information-retrieval-is-fault-tolerant
**Status:** OUT

Every information retrieval path — structured reads (FTS search with index self-healing, staleness checking with deterministic output, compact with pure bounded distillation) and LLM-synthesized queries (ask with bounded tool loops and raw search fallback) — degrades gracefully rather than failing, ensuring the system always returns useful results

**Depends on:** [all-read-paths-are-deterministic-and-resilient](deterministic.md#all-read-paths-are-deterministic-and-resilient), [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded)
**Supports:** [all-information-flow-is-fault-tolerant-and-governed](other.md#all-information-flow-is-fault-tolerant-and-governed)

### all-modifications-converge-with-reporting-and-recovery
**Status:** OUT

Every modification — addition, removal, or correction — converges to a deterministic stable state with complete effect reporting and guided recovery, ensuring the system's operational history is fully transparent and every change can be understood and reversed.

**Depends on:** [removal-effects-are-fully-reported-and-recoverable](other.md#removal-effects-are-fully-reported-and-recoverable), [system-reaches-equilibrium-from-all-modification-paths](system.md#system-reaches-equilibrium-from-all-modification-paths)
**Supports:** [all-operations-converge-with-topology-and-recovery](topology.md#all-operations-converge-with-topology-and-recovery)

### all-mutations-preserve-integrity-under-adverse-conditions
**Status:** OUT

Every structural modification to the belief network preserves integrity even under adverse graph conditions: mutations are uniquely identifiable, auditable, and topology-preserving, while justification addition achieves consistent multi-dimensional propagation even when the dependency graph contains dangling references.

**Depends on:** [all-structural-changes-are-identified-auditable-and-topology-preserving](topology.md#all-structural-changes-are-identified-auditable-and-topology-preserving), [justification-addition-is-robust-across-graph-states](justification.md#justification-addition-is-robust-across-graph-states)
**Supports:** [operational-safety-is-universal-and-condition-independent](other.md#operational-safety-is-universal-and-condition-independent)

### all-query-operations-degrade-gracefully
**Status:** IN

All query operations degrade gracefully through multiple independent fallback tiers: ask cascades from LLM synthesis through bounded tool loops to raw FTS5 results on any failure, while search self-heals via index rebuilds on every save and falls back to substring matching on FTS5 unavailability — ensuring queries always return useful results regardless of LLM availability or index state.

**Depends on:** [ask-has-tiered-query-modes](other.md#ask-has-tiered-query-modes), [search-is-resilient-across-index-states](other.md#search-is-resilient-across-index-states)
**Supports:** [query-degradation-is-deterministic-across-all-access-paths](deterministic.md#query-degradation-is-deterministic-across-all-access-paths)

### all-reconciliation-converges-deterministically
**Status:** OUT

All reconciliation operations converge deterministically to stable states: individual propagation terminates via BFS with stop-on-unchanged, while system-wide operations (sync, dependents rebuild, recompute) all reach idempotent fixed points — the system has no divergent operational paths.

**Depends on:** [critical-operations-converge-to-fixed-points](other.md#critical-operations-converge-to-fixed-points), [propagation-terminates-deterministically](other.md#propagation-terminates-deterministically)
**Supports:** [bulk-operations-converge-and-preserve-topology](topology.md#bulk-operations-converge-and-preserve-topology), [external-ingestion-is-resilient-and-convergent](external.md#external-ingestion-is-resilient-and-convergent), [import-reconciliation-converges-deterministically](import.md#import-reconciliation-converges-deterministically)

### all-removals-provide-reporting-and-recovery
**Status:** OUT

Every belief removal — whether intentional retraction (structured before/after diffs with surgical restoration hints targeting only cascade victims with surviving premises) or automated contradiction resolution (consistent nogood records with traceable dependency-directed backtracking to least-entrenched culprits) — provides both complete effect reporting and actionable recovery guidance.

**Depends on:** [contradiction-resolution-is-traceable-and-recoverable](other.md#contradiction-resolution-is-traceable-and-recoverable), [retraction-effects-are-reported-with-recovery-guidance](other.md#retraction-effects-are-reported-with-recovery-guidance)
**Supports:** [removal-effects-are-fully-reported-and-recoverable](other.md#removal-effects-are-fully-reported-and-recoverable)

### all-safety-dimensions-converge
**Status:** OUT

Four independently-established safety dimensions — dialectical determinism, edge-case uniformity, node lifecycle awareness, and mutation-source coverage — converge into a single comprehensive safety property, because each was independently derived from the same minimal evaluation core and they impose no conflicting constraints on each other.

**Depends on:** [revision-spans-lifecycle-and-all-sources](revision.md#revision-spans-lifecycle-and-all-sources), [safety-and-uniformity-are-co-derived](other.md#safety-and-uniformity-are-co-derived)

### all-semantic-edge-cases-are-uniform
**Status:** IN

All semantic edge cases — absence of justifications yielding premise behavior, absence of nodes producing asymmetric fail semantics, and empty antecedent lists satisfying vacuous truth — emerge from the same uniform evaluation rules without special-case handling, including the emergent disjunctive-over-conjunctive truth structure.

**Depends on:** [absence-has-consistent-dual-semantics](other.md#absence-has-consistent-dual-semantics), [truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)
**Supports:** [belief-revision-covers-all-cases-uniformly](revision.md#belief-revision-covers-all-cases-uniformly)

### all-transformations-are-evaluation-transparent
**Status:** OUT

Every structural transformation the system performs — mode expansion (conjunctive to disjunctive justifications), negation semantics (outlist defeat and absence), and dialectical identity changes (premise to justified) — is transparent to truth evaluation, producing identical results regardless of transformation history.

**Depends on:** [any-mode-expansion-is-evaluation-invisible](other.md#any-mode-expansion-is-evaluation-invisible), [negation-is-transparent-to-evaluation](other.md#negation-is-transparent-to-evaluation)
**Supports:** [transformations-converge-to-documented-equilibria](other.md#transformations-converge-to-documented-equilibria)

### all-truth-effects-propagate-through-outlist-paths
**Status:** IN

All truth effects — both incremental truth changes and outlist-based defeat reversals — propagate completely through outlist-connected paths without requiring full network recomputation, ensuring outlist-based non-monotonic reasoning achieves the same incremental efficiency as antecedent-based reasoning

**Depends on:** [defeat-reversal-propagates-automatically](other.md#defeat-reversal-propagates-automatically), [incremental-propagation-is-fully-complete](complete.md#incremental-propagation-is-fully-complete)
**Supports:** [all-truth-effects-are-topology-complete-and-traceable](topology.md#all-truth-effects-are-topology-complete-and-traceable)

### any-mode-creates-per-premise-justifications
**Status:** IN

When `any_mode=True` and multiple antecedents are given, each antecedent gets its own SL justification (OR semantics: node is IN if *any* antecedent is IN), rather than the default single multi-antecedent justification (AND semantics).

**Supports:** [any-mode-preserves-full-outlist-semantics](other.md#any-mode-preserves-full-outlist-semantics), [truth-is-disjunctive-over-conjunctive-rules](other.md#truth-is-disjunctive-over-conjunctive-rules)

### any-mode-expansion-is-evaluation-invisible
**Status:** OUT

Any-mode expansion from conjunctive to disjunctive justifications is invisible to truth evaluation: the expanded justifications propagate completely through the same BFS mechanisms, and truth evaluation produces identical results regardless of whether justifications arrived via original specification or any-mode expansion — a consequence of transformation invariance.

**Depends on:** [any-mode-expansion-propagates-completely](other.md#any-mode-expansion-propagates-completely), [truth-evaluation-is-transformation-invariant](other.md#truth-evaluation-is-transformation-invariant)
**Supports:** [all-transformations-are-evaluation-transparent](other.md#all-transformations-are-evaluation-transparent)

### any-mode-expansion-propagates-completely
**Status:** IN

When any_mode expands a conjunctive justification into per-premise disjunctive justifications, each resulting justification inherits the complete outlist specification (conjunction semantics, absence tolerance, persistence across save/load), and all resulting truth-value changes propagate completely to every affected dependent — but only when outlist nodes are tracked in the dependents index, ensuring outlist-mediated effects are not silently dropped.

**Depends on:** [any-mode-preserves-full-outlist-semantics](other.md#any-mode-preserves-full-outlist-semantics), [outlist-semantics-are-fully-specified](other.md#outlist-semantics-are-fully-specified)
**Supports:** [any-mode-expansion-is-evaluation-invisible](other.md#any-mode-expansion-is-evaluation-invisible)

### any-mode-is-structural-expansion
**Status:** IN

Duplicates existing belief `any-mode-creates-per-premise-justifications` which already captures that any_mode expands N premises into N single-premise SL justifications.


### any-mode-outlist-preserved
**Status:** IN

When `any_mode` expands `sl="a,b" unless="enemy"`, each resulting single-premise justification inherits `"enemy"` in its outlist.

**Supports:** [any-mode-preserves-full-outlist-semantics](other.md#any-mode-preserves-full-outlist-semantics)

### any-mode-preserves-full-outlist-semantics
**Status:** IN

When any_mode expands a single multi-antecedent justification into per-premise justifications (OR semantics), each resulting justification preserves the original outlist entries — ensuring non-monotonic defeat works correctly under disjunctive expansion with no semantic loss.

**Depends on:** [any-mode-creates-per-premise-justifications](other.md#any-mode-creates-per-premise-justifications), [any-mode-outlist-preserved](other.md#any-mode-outlist-preserved)
**Supports:** [any-mode-expansion-propagates-completely](other.md#any-mode-expansion-propagates-completely)

### api-cascade-symmetry-tested
**Status:** IN

Test coverage claim; the underlying behavioral invariant (symmetric retract/restore cascades) is already covered by existing beliefs including `reasoning-engine-is-deterministic-and-reversible`.


### api-enforces-typed-preconditions
**Status:** OUT

API functions enforce preconditions at the system boundary with typed exceptions: duplicate node IDs raise ValueError, missing justification arguments raise ValueError, and unauthorized single-node access raises PermissionError — establishing a consistent error contract at every entry point.

**Depends on:** [api-add-justification-requires-justification-arg](justification.md#api-add-justification-requires-justification-arg), [duplicate-node-id-raises-valueerror](other.md#duplicate-node-id-raises-valueerror), [single-node-api-raises-permissionerror](other.md#single-node-api-raises-permissionerror)
**Supports:** [input-validation-is-comprehensive-at-all-boundaries](other.md#input-validation-is-comprehensive-at-all-boundaries)

### api-fts-search-bounds-query-calls
**Status:** IN

`_fts_search` makes at most 51 internal `_fts_query` calls regardless of input query length, preventing combinatorial explosion from progressive relaxation on long queries.


### api-functions-return-dicts
**Status:** IN

Every public API function returns a `dict` (or `str` for markdown/compact), never a `Network` or `Node` object, ensuring JSON-serializability at the boundary for CLI, HTTP, and tool-call consumers.

**Supports:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations)

### api-idempotent-retract-assert
**Status:** IN

Already exists as an accepted belief with the same ID and content.


### api-layer-ensures-atomic-isolated-mutations
**Status:** IN

The API layer enforces mutation safety through four mechanisms: context-managed load/save, per-function transaction scope, write-flag gating to prevent unintended persistence, and dict-only returns that prevent callers from holding live network references.

**Depends on:** [api-functions-return-dicts](other.md#api-functions-return-dicts), [api-uses-with-network-context-manager](other.md#api-uses-with-network-context-manager), [transaction-per-function](other.md#transaction-per-function), [write-false-prevents-persistence](other.md#write-false-prevents-persistence)
**Supports:** [architecture-enforces-structural-and-operational-safety](other.md#architecture-enforces-structural-and-operational-safety), [architecture-has-no-hidden-fragility](other.md#architecture-has-no-hidden-fragility), [atomicity-is-backend-independent](other.md#atomicity-is-backend-independent), [dual-backend-atomicity-is-feature-complete](complete.md#dual-backend-atomicity-is-feature-complete), [llm-driven-mutations-are-safely-bounded](llm.md#llm-driven-mutations-are-safely-bounded), [mutation-pipeline-is-atomic-snapshot](other.md#mutation-pipeline-is-atomic-snapshot), [mutations-are-atomic-audited-and-index-consistent](other.md#mutations-are-atomic-audited-and-index-consistent)

### api-list-negative-filters-hallucinated-ids
**Status:** IN

`list_negative()` discards any node IDs returned by the LLM that don't exist in the database, preventing hallucinated IDs from appearing in results.

**Supports:** [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded), [reference-validation-is-defense-in-depth](other.md#reference-validation-is-defense-in-depth)

### api-mutating-ops-use-before-after-diffing
**Status:** IN

Mutating operations (`retract_node`, `assert_node`, `what_if_retract`, `what_if_assert`) snapshot all truth values before the operation and diff afterward to classify changes into `went_out`/`went_in` lists.

**Supports:** [every-mutation-reports-its-effects](other.md#every-mutation-reports-its-effects), [mutations-are-observable-audited-and-index-consistent](other.md#mutations-are-observable-audited-and-index-consistent)

### api-retract-cascade-is-transitive
**Status:** IN

`api.retract_node()` propagates OUT to all transitively dependent SL-derived nodes, not just direct children — retracting a root premise retracts the entire downstream chain.

**Supports:** [retraction-cascade-is-transitive-and-terminating](other.md#retraction-cascade-is-transitive-and-terminating)

### api-superseded-nodes-excluded-from-gated
**Status:** IN

`list_gated()` omits nodes that have been superseded via `api.supersede()`, even if they still have active blockers — stale conclusions don't pollute the blocker view.

**Supports:** [supersession-is-reversible-and-view-consistent](other.md#supersession-is-reversible-and-view-consistent)

### api-tests-black-box
**Status:** IN

Test methodology claim, not a behavioral invariant about the codebase. Developers learn this from reading the tests, not from a belief registry.


### api-tests-cover-subset
**Status:** IN

Test coverage inventory, not a behavioral claim about the codebase. Coverage gaps are better tracked as project-level issues.


### api-tests-use-real-sqlite
**Status:** IN

All API tests run against a real SQLite database with FTS5; storage is never mocked, ensuring the API contract includes correct SQL and full-text search behavior.


### api-uses-lazy-imports
**Status:** IN

Heavy modules (`derive`, `compact`, `export_markdown`, `check_stale`, `import_beliefs`, `import_agent`) are imported inside function bodies in `api.py`, not at module level, to keep the module fast to import for callers that only need a subset of operations.

**Supports:** [startup-performance-uses-lazy-loading](other.md#startup-performance-uses-lazy-loading)

### api-uses-with-network-context-manager
**Status:** IN

`api.py` uses a `_with_network` context manager to ensure load-operate-save atomicity for all network mutations.

**Supports:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [three-layer-stack-has-clean-boundaries](other.md#three-layer-stack-has-clean-boundaries)

### api-visible-to-filters-both-result-and-prompt
**Status:** IN

Already exists as `api-visible-to-filters-both-result-and-prompt`


### apply-dedup-plan-collects-errors-not-raises
**Status:** IN

`apply_dedup_plan` collects errors into `result["errors"]` rather than raising, allowing partial application — one missing node does not block processing of the remaining dedup plan


### architecture-enforces-structural-and-operational-safety
**Status:** IN

Architectural safety is enforced along two independent dimensions: structurally, the central network dependency is contained within clean three-layer boundaries preventing cross-layer corruption; operationally, every mutation path is atomic and isolated preventing within-layer partial state — neither dimension alone is sufficient, but together they eliminate both classes of corruption.

**Depends on:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [central-dependency-is-safely-contained](other.md#central-dependency-is-safely-contained)
**Supports:** [architecture-sustains-gapless-lifecycle](lifecycle.md#architecture-sustains-gapless-lifecycle), [external-integration-is-architecturally-safe](external.md#external-integration-is-architecturally-safe), [invariant-preservation-is-architecturally-grounded](other.md#invariant-preservation-is-architecturally-grounded), [safety-is-enforced-across-all-layers-and-backends](other.md#safety-is-enforced-across-all-layers-and-backends)

### architecture-has-no-hidden-fragility
**Status:** OUT

The system's architectural safety is robust end-to-end: structural containment via clean layer boundaries and operational atomicity via context-managed mutations leave no hidden consistency hazards across the persistence boundary.

**Depends on:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [central-dependency-is-safely-contained](other.md#central-dependency-is-safely-contained)
**Supports:** [deterministic-reasoning-operates-on-sound-architecture](deterministic.md#deterministic-reasoning-operates-on-sound-architecture), [lifecycle-operates-on-unfragile-architecture](lifecycle.md#lifecycle-operates-on-unfragile-architecture)

### ask-agentic-loop-is-bounded
**Status:** IN

The tool-call loop in `ask()` has a maximum iteration count; an LLM that perpetually requests more searches is terminated and the raw response is returned.

**Supports:** [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded)

### ask-always-returns-string
**Status:** IN

`ask()` returns a string on every code path — LLM response, raw search results, or fallback; it never raises an exception to the caller.

**Supports:** [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded), [llm-integration-fails-softly-across-modules](llm.md#llm-integration-fails-softly-across-modules)

### ask-dual-requires-sources-db
**Status:** IN

Calling `ask(..., dual=True)` without providing a `sources_db` path raises `ValueError`.


### ask-falls-back-to-raw-search
**Status:** IN

When LLM synthesis fails for any reason (timeout, missing CLI, non-zero exit), `ask()` returns the raw FTS5 search results as fallback.

**Supports:** [ask-has-tiered-query-modes](other.md#ask-has-tiered-query-modes)

### ask-has-tiered-query-modes
**Status:** IN

The ask module supports tiered query modes with graceful degradation: full LLM synthesis with a bounded 3-iteration tool loop, no-synth mode that bypasses the LLM entirely, and automatic fallback from LLM failure to raw FTS5 search results.

**Depends on:** [ask-falls-back-to-raw-search](other.md#ask-falls-back-to-raw-search), [ask-no-synth-bypasses-llm](llm.md#ask-no-synth-bypasses-llm), [ask-tool-loop-capped-at-three](other.md#ask-tool-loop-capped-at-three)
**Supports:** [all-query-operations-degrade-gracefully](other.md#all-query-operations-degrade-gracefully)

### ask-is-fault-tolerant-and-bounded
**Status:** IN

The ask module is fault-tolerant (always returns a string, catches LLM failures, falls back to raw FTS5 search) and execution-bounded (tool loop capped at 3 iterations), ensuring reliable bounded knowledge retrieval regardless of LLM availability.

**Depends on:** [ask-agentic-loop-is-bounded](other.md#ask-agentic-loop-is-bounded), [ask-always-returns-string](other.md#ask-always-returns-string), [ask-never-raises-on-llm-failure](llm.md#ask-never-raises-on-llm-failure)
**Supports:** [all-information-retrieval-is-fault-tolerant](other.md#all-information-retrieval-is-fault-tolerant), [all-llm-interactions-are-bounded-and-fail-soft](llm.md#all-llm-interactions-are-bounded-and-fail-soft), [all-llm-operations-achieve-coverage-and-fault-tolerance](llm.md#all-llm-operations-achieve-coverage-and-fault-tolerance), [ask-degrades-across-all-external-dependencies](external.md#ask-degrades-across-all-external-dependencies)

### ask-mcp-achieves-accurate-bounded-tool-use
**Status:** OUT

MCP tool integration in ask() achieves both bounded safety (iteration caps, error tolerance, transport timeouts at two layers) and accurate tool discovery (catalog reflects current server capabilities rather than a stale snapshot).

**Depends on:** [ask-mcp-integration-is-safely-bounded](other.md#ask-mcp-integration-is-safely-bounded), [mcp-bridge-is-timeout-bounded-at-all-phases](other.md#mcp-bridge-is-timeout-bounded-at-all-phases)

### ask-mcp-errors-non-fatal
**Status:** IN

When an MCP bridge's `call_tool` raises an exception during the ask loop, `ask()` catches the error, feeds it back to the LLM as context, and continues the tool loop rather than propagating to the caller.

**Supports:** [ask-mcp-integration-is-safely-bounded](other.md#ask-mcp-integration-is-safely-bounded), [ask-mcp-tool-use-has-current-catalog](other.md#ask-mcp-tool-use-has-current-catalog)

### ask-mcp-integration-is-safely-bounded
**Status:** IN

MCP tool calls in `ask()` are both error-tolerant (exceptions caught and fed back as context for alternative tool selection) and iteration-bounded (5 tool-call rounds max), preventing both crashes from MCP server failures and runaway tool loops.

**Depends on:** [ask-mcp-errors-non-fatal](other.md#ask-mcp-errors-non-fatal), [ask-mcp-iteration-limit-is-five](other.md#ask-mcp-iteration-limit-is-five)
**Supports:** [ask-mcp-achieves-accurate-bounded-tool-use](other.md#ask-mcp-achieves-accurate-bounded-tool-use), [ask-mcp-is-defense-in-depth-bounded](other.md#ask-mcp-is-defense-in-depth-bounded)

### ask-mcp-is-defense-in-depth-bounded
**Status:** IN

MCP-backed ask queries are bounded at two independent layers: application-level iteration caps with error-tolerant fallback at the ask layer, plus per-call and connection timeouts at the MCP bridge transport layer — no single timeout failure can cause unbounded execution.

**Depends on:** [ask-mcp-integration-is-safely-bounded](other.md#ask-mcp-integration-is-safely-bounded), [mcp-bridge-is-timeout-bounded-at-all-phases](other.md#mcp-bridge-is-timeout-bounded-at-all-phases)

### ask-mcp-iteration-limit-is-five
**Status:** IN

When `mcp_servers` is non-empty, `ask()` allows up to 5 tool-call iterations before forcing a final LLM response (6 total invocations), compared to the lower limit without MCP servers.

**Supports:** [ask-mcp-integration-is-safely-bounded](other.md#ask-mcp-integration-is-safely-bounded), [ask-mcp-tool-use-has-current-catalog](other.md#ask-mcp-tool-use-has-current-catalog)

### ask-mcp-tool-use-has-current-catalog
**Status:** OUT

Ask's MCP tool integration achieves full reliability — errors caught, iterations bounded — with the tool catalog always reflecting the MCP server's current capabilities rather than a stale connection-time snapshot.

**Depends on:** [ask-mcp-errors-non-fatal](other.md#ask-mcp-errors-non-fatal), [ask-mcp-iteration-limit-is-five](other.md#ask-mcp-iteration-limit-is-five)

### ask-natural-mode-strips-metadata-and-cite
**Status:** IN

When `natural=True`, all belief metadata (`**Status:**`, `### ` headers, `**Source:**`) is stripped from the prompt context and the "Cite belief IDs" instruction is replaced with "plain natural language".


### ask-prompt-no-template-placeholders
**Status:** IN

`build_ask_prompt` never leaves `{{` or `}}` template markers in the generated prompt string — all placeholders are resolved before output.


### ask-sources-db-failure-silently-degrades
**Status:** IN

If the `sources_db` SQLite file is missing or corrupt, `_search_source_chunks` catches `OperationalError`/`DatabaseError` and returns empty string, degrading to belief-only mode without user-visible errors.

**Supports:** [ask-degrades-across-all-external-dependencies](external.md#ask-degrades-across-all-external-dependencies)

### ask-stop-word-fallback-ensures-nonempty-query
**Status:** IN

`_search_source_chunks` strips stop words from the question before building the FTS5 query, but falls back to all words longer than 1 character if every word is a stop word — ensuring the query is never empty


### ask-strips-claudecode-env
**Status:** IN

`_invoke_claude` removes the `CLAUDECODE` environment variable to prevent recursive invocation when running inside Claude Code.

**Supports:** [llm-integration-is-bounded-fail-soft-and-process-isolated](llm.md#llm-integration-is-bounded-fail-soft-and-process-isolated)

### ask-tool-loop-capped-at-three
**Status:** IN

The LLM synthesis loop runs at most `MAX_ITERATIONS` (3) rounds, with `FINAL_TURN_INSTRUCTION` appended on the last iteration to force a final answer.

**Supports:** [ask-has-tiered-query-modes](other.md#ask-has-tiered-query-modes)

### ask-uses-text-based-tool-protocol
**Status:** IN

`ask.py` implements tool use by parsing JSON lines with a `"tool"` key from LLM text output, not Claude's native tool-use API, because it invokes `claude -p` (pipe mode) which doesn't support function calling.


### atomicity-is-backend-independent
**Status:** IN

Both storage backends enforce atomic isolated operations through backend-appropriate mechanisms: the SQLite backend uses context-managed load/save with write-flag gating and per-function transaction scope, while PostgreSQL uses per-method transactions with composite-key multi-tenancy — achieving the same transactional guarantee at different architectural levels

**Depends on:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [pgapi-is-sql-native-multi-tenant](other.md#pgapi-is-sql-native-multi-tenant)
**Supports:** [dual-storage-backends-are-interchangeable](other.md#dual-storage-backends-are-interchangeable), [storage-layer-is-backend-agnostic-and-safe](safe.md#storage-layer-is-backend-agnostic-and-safe)

### auto-retract-respects-dry-run
**Status:** IN

The `--auto-retract` flag in the `review-beliefs` CLI is gated by `--dry-run`: when dry-run is active, findings are displayed but no database mutation occurs, even for beliefs flagged as invalid.

**Supports:** [review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe)

### autonomous-convergence-preserves-trust-boundaries
**Status:** OUT

The system simultaneously achieves autonomous self-maintenance (converging to deterministic stable states while actively detecting and resolving inconsistencies) AND comprehensive boundary enforcement (architectural trust through self-containment and information flow control through authorization and budget constraints) — convergence never requires relaxing defensive controls, and boundary enforcement never prevents convergence.

**Depends on:** [system-autonomously-converges-and-self-corrects](system.md#system-autonomously-converges-and-self-corrects), [trust-and-information-boundaries-are-comprehensively-enforced](other.md#trust-and-information-boundaries-are-comprehensively-enforced)
**Supports:** [trust-boundaries-are-self-maintaining](self.md#trust-boundaries-are-self-maintaining)

### autonomous-convergence-produces-documented-equilibria
**Status:** OUT

The system's autonomous convergence to evaluation-invariant equilibria generates consistently identifiable artifacts with deterministic traceable history at every step — every equilibrium state is not merely stable and transformation-invariant but fully explainable through its documented convergence path.

**Depends on:** [convergence-produces-evaluation-invariant-equilibria](other.md#convergence-produces-evaluation-invariant-equilibria), [system-history-is-consistently-referenceable](system.md#system-history-is-consistently-referenceable)
**Supports:** [transformations-converge-to-documented-equilibria](other.md#transformations-converge-to-documented-equilibria)

### backend-agnostic-operational-assurance
**Status:** OUT

The system's comprehensive operational assurance — spanning temporal self-correction, end-to-end reliability, and external control — holds identically across both storage backends with equivalent safety guarantees, provided PgApi achieves full API parity with the SQLite path.

**Depends on:** [safety-is-enforced-across-all-layers-and-backends](other.md#safety-is-enforced-across-all-layers-and-backends), [system-assurance-spans-correction-reliability-and-control](system.md#system-assurance-spans-correction-reliability-and-control)

### backtracking-retracts-least-entrenched
**Status:** IN

`add_nogood` resolves contradictions via dependency-directed backtracking: `find_culprits` traces to premises, scores by `_entrenchment`, and retracts the least-entrenched premise to minimize disruption.

**Supports:** [contradiction-resolution-is-minimal-disruption](other.md#contradiction-resolution-is-minimal-disruption), [nogood-resolution-maintains-consistent-ids](other.md#nogood-resolution-maintains-consistent-ids)

### belief-currency-is-actively-managed
**Status:** OUT

The system actively manages belief currency bidirectionally: the production-ready derive pipeline safely introduces new beliefs through defensive validation, while the staleness CI gate detects drift in existing beliefs against source material — together preventing both unsafe additions and undetected obsolescence.

**Depends on:** [derive-pipeline-is-production-ready](derive.md#derive-pipeline-is-production-ready), [staleness-gate-catches-all-drift](other.md#staleness-gate-catches-all-drift)
**Supports:** [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [external-beliefs-are-safe-and-current](external.md#external-beliefs-are-safe-and-current), [resource-management-supports-belief-currency](other.md#resource-management-supports-belief-currency), [self-correction-spans-creation-and-maintenance](self.md#self-correction-spans-creation-and-maintenance)

### belief-text-truncated-at-200-chars
**Status:** IN

`format_beliefs_for_contradiction_check` truncates any belief text longer than 200 characters, appending `...` as a suffix, to keep LLM prompts bounded.


### bootstrap-bypasses-incremental-propagation
**Status:** IN

Both persistence loading and import construct the full node graph before truth maintenance — load trusts stored truth values and skips propagation entirely, while import adds all nodes then propagates via recompute_all — sharing a bulk-construction pattern that avoids per-node incremental propagation.

**Depends on:** [import-two-phase-truth-maintenance](import.md#import-two-phase-truth-maintenance), [storage-load-bypasses-propagation](other.md#storage-load-bypasses-propagation)
**Supports:** [initialization-is-path-independent](other.md#initialization-is-path-independent)

### budget-enforcement-is-efficient-across-pipeline
**Status:** OUT

All budget-constrained operations — compact output distillation and derive belief allocation — achieve computationally efficient tracking with representation-safe minimum bounds, ensuring budget enforcement never becomes a performance bottleneck.

**Depends on:** [compact-budget-tracking-is-efficient-and-approximate](compact.md#compact-budget-tracking-is-efficient-and-approximate), [derive-budget-is-efficient-and-floor-bounded](derive.md#derive-budget-is-efficient-and-floor-bounded)
**Supports:** [resource-efficiency-spans-footprint-through-budgets](spans.md#resource-efficiency-spans-footprint-through-budgets)

### budget-floor-is-five
**Status:** IN

`_build_beliefs_section` guarantees local beliefs get at least 5 slots regardless of agent count, enforced by `max(5, max_beliefs - count)`


### build-prompt-validates-custom-templates
**Status:** IN

`build_prompt` raises `ValueError("unknown placeholder")` for unrecognized `{fields}` and `ValueError("malformed braces")` for unclosed braces in custom prompt templates


### canonical-equilibria-are-negation-transparent
**Status:** OUT

The system converges to canonical evaluation-invariant equilibria where negative semantics are fully transparent — the final stable state is determined solely by the logical content of justifications, independent of both the transformation path taken and whether beliefs were established through positive assertion or negative defeat.

**Depends on:** [convergence-produces-evaluation-invariant-equilibria](other.md#convergence-produces-evaluation-invariant-equilibria), [negation-is-transparent-to-evaluation](other.md#negation-is-transparent-to-evaluation)
**Supports:** [equilibria-are-negation-transparent-with-complete-fidelity](complete.md#equilibria-are-negation-transparent-with-complete-fidelity)

### central-dependency-is-safely-contained
**Status:** IN

Despite `network.py` being imported by virtually every module in the codebase, the three-layer architecture with clean boundaries ensures this central coupling does not create cross-cutting mutation paths — layer separation contains the dependency's blast radius so that the hub topology does not compromise architectural integrity.

**Depends on:** [network-is-central-dependency](other.md#network-is-central-dependency), [three-layer-stack-has-clean-boundaries](other.md#three-layer-stack-has-clean-boundaries)
**Supports:** [architecture-enforces-structural-and-operational-safety](other.md#architecture-enforces-structural-and-operational-safety), [architecture-has-no-hidden-fragility](other.md#architecture-has-no-hidden-fragility), [architecture-is-self-contained-and-safely-layered](self.md#architecture-is-self-contained-and-safely-layered)

### challenge-converts-premises-to-justified
**Status:** IN

When a premise (node with no justifications) is challenged, it is converted to a justified node with an SL justification containing empty antecedents and the challenge in the outlist.

**Supports:** [challenge-destroys-premise-identity](other.md#challenge-destroys-premise-identity)

### challenge-destroys-premise-identity
**Status:** IN

When a premise is challenged, it loses its defining characteristic: premise identity emerges from absence of justifications, but challenge adds a justification (converting the premise to a justified node), meaning the target's truth value becomes conditional on the challenge node being OUT rather than unconditionally held — challenge reclassifies the target in the node type system.

**Depends on:** [challenge-converts-premises-to-justified](other.md#challenge-converts-premises-to-justified), [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence)
**Supports:** [dialectical-defeat-is-reversible-but-identity-is-permanent](other.md#dialectical-defeat-is-reversible-but-identity-is-permanent), [dialectical-transformation-preserves-semantics](other.md#dialectical-transformation-preserves-semantics), [premise-identity-is-bidirectionally-transformable](other.md#premise-identity-is-bidirectionally-transformable)

### challenge-id-auto-generation
**Status:** IN

Auto-generated challenge IDs follow the pattern `challenge-{target}`, then `challenge-{target}-2`, `-3`, etc.; explicit IDs that collide raise `ValueError` rather than auto-deduplicating.

**Supports:** [system-artifacts-maintain-consistent-identification](system.md#system-artifacts-maintain-consistent-identification)

### challenge-is-outlist-injection
**Status:** IN

The challenge mechanism creates a new premise node and adds it to the target's outlist; all truth-value changes flow through normal BFS propagation, not direct mutation

**Supports:** [dialectical-structure-is-recursive-outlist](other.md#dialectical-structure-is-recursive-outlist), [outlist-is-universal-defeat-mechanism](other.md#outlist-is-universal-defeat-mechanism)

### challenge-modifies-all-justifications
**Status:** IN

When the target has multiple justifications, the challenge node is added to the outlist of every justification, ensuring no single justification can independently keep the target IN.

**Supports:** [dialectical-structure-is-recursive-outlist](other.md#dialectical-structure-is-recursive-outlist)

### challenge-uses-outlist-mechanism
**Status:** IN

`challenge` works by creating an IN premise node and adding it to the target's outlist in every justification, reusing the same non-monotonic mechanism as `supersede`.

**Supports:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [dialectical-structure-is-recursive-outlist](other.md#dialectical-structure-is-recursive-outlist)

### check-stale-and-hash-sources-mutate-in-place
**Status:** IN

Both `check_stale` (with `upgrade_hashes=True`) and `hash_sources` modify `node.source_hash` directly on the Network object; neither persists — the caller must save.


### check-stale-exits-nonzero
**Status:** IN

`cmd_check_stale` calls `sys.exit(1)` when any stale nodes are found, making it usable as a CI or pre-commit gate.

**Supports:** [staleness-checking-is-comprehensive](other.md#staleness-checking-is-comprehensive), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)

### check-stale-is-read-only
**Status:** IN

`check_stale` never mutates the network; it returns a list of stale-node dicts and leaves all nodes unchanged — staleness detection is separated from staleness resolution.

**Supports:** [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)

### check-stale-never-raises-on-missing-files
**Status:** IN

`check_stale()` returns a structured `source_deleted` dict for nodes whose source files don't exist on disk; it never raises `FileNotFoundError` or any exception.

**Supports:** [check-stale-output-is-deterministic-and-structured](deterministic.md#check-stale-output-is-deterministic-and-structured)

### check-stale-report-only
**Status:** OUT

Duplicate of existing belief `check-stale-is-read-only`.


### check-stale-result-schema-uniform
**Status:** IN

All `check_stale` result dicts share the same 6-key schema (`node_id`, `old_hash`, `new_hash`, `source`, `source_path`, `reason`) regardless of reason type, so consumers can iterate results without type-checking.

**Supports:** [check-stale-output-is-deterministic-and-structured](deterministic.md#check-stale-output-is-deterministic-and-structured)

### check-stale-results-sorted-by-node-id
**Status:** IN

The list returned by `check_stale` is sorted ascending by `node_id`, providing deterministic output for consumers.

**Supports:** [check-stale-output-is-deterministic-and-structured](deterministic.md#check-stale-output-is-deterministic-and-structured)

### check-stale-skips-out-nodes
**Status:** IN

Only nodes with `truth_value == "IN"` are checked for staleness; retracted (OUT) nodes are ignored even if their source file has changed.

**Supports:** [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)

### cli-backend-kwargs-controls-storage
**Status:** IN

`_backend_kwargs(args)` is the single chokepoint that determines whether a command runs against SQLite or PostgreSQL; every `cmd_*` function must spread its return value into the corresponding `api.*` call


### cli-dispatch-is-flat-dict-lookup
**Status:** IN

CLI dispatch uses a flat `commands` dict mapping subcommand strings to `cmd_*` handler functions — no plugin system or subclass hierarchy.

**Supports:** [cli-is-deterministic-and-stream-correct](deterministic.md#cli-is-deterministic-and-stream-correct), [cli-is-pure-delegation-layer](other.md#cli-is-pure-delegation-layer)

### cli-errors-use-stderr-success-uses-stdout
**Status:** IN

CLI error diagnostics are written to stderr and success output to stdout; tests consistently assert on the correct stream.

**Supports:** [cli-is-deterministic-and-stream-correct](deterministic.md#cli-is-deterministic-and-stream-correct)

### cli-exit-1-on-error
**Status:** IN

All CLI error paths catch specific exceptions from the API (`KeyError`, `ValueError`, `PermissionError`, `FileNotFoundError`), print to stderr, and call `sys.exit(1)`.


### cli-exit-code-contract-is-binary
**Status:** IN

Every CLI subcommand returns exit code 0 for success and 1 for any user-facing error; no other exit codes are used or tested.

**Supports:** [cli-is-deterministic-and-stream-correct](deterministic.md#cli-is-deterministic-and-stream-correct)

### cli-flags-override-env-vars
**Status:** IN

`_backend_kwargs` gives precedence to CLI `--pg`/`--project-id` flags over `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables when both are present


### cli-is-pure-delegation-layer
**Status:** OUT

The CLI is a pure delegation layer: every handler dispatches through a flat dict lookup to API functions with no business logic, producing binary exit codes and correct stream separation — a complete separation of formatting from computation.

**Depends on:** [cli-dispatch-is-flat-dict-lookup](other.md#cli-dispatch-is-flat-dict-lookup), [cli-is-pure-formatter](other.md#cli-is-pure-formatter)
**Supports:** [cli-is-verified-pure-delegation](other.md#cli-is-verified-pure-delegation)

### cli-is-pure-formatter
**Status:** IN

Every cmd_* function delegates to api.* and only formats the returned dict for terminal output; no business logic lives in the CLI layer.

**Supports:** [cli-is-pure-delegation-layer](other.md#cli-is-pure-delegation-layer), [three-layer-stack-has-clean-boundaries](other.md#three-layer-stack-has-clean-boundaries)

### cli-is-verified-end-to-end
**Status:** IN

The CLI is verified through hermetic end-to-end integration tests (isolated databases per test, full argv-parsing pipeline, deterministic stream-correct output) — unless cmd_propagate bypasses the API layer, leaving one code path's safety guarantees unverifiable through the standard integration testing harness.

**Depends on:** [cli-is-deterministic-and-stream-correct](deterministic.md#cli-is-deterministic-and-stream-correct), [each-cli-test-creates-isolated-db](other.md#each-cli-test-creates-isolated-db)
**Supports:** [cli-is-verified-pure-delegation](other.md#cli-is-verified-pure-delegation)

### cli-is-verified-pure-delegation
**Status:** OUT

The CLI is both structurally pure (every handler delegates to API functions with no business logic) and end-to-end verified (hermetic integration tests confirm delegation produces correct output through the full argv-parsing pipeline).

**Depends on:** [cli-is-pure-delegation-layer](other.md#cli-is-pure-delegation-layer), [cli-is-verified-end-to-end](other.md#cli-is-verified-end-to-end)
**Supports:** [full-user-stack-is-verified-atomic-delegation](other.md#full-user-stack-is-verified-atomic-delegation)

### cli-plan-review-apply-pattern
**Status:** IN

Several commands (`derive`, `deduplicate`, `contradictions`) follow a three-phase workflow: (1) generate proposals to a file, (2) human reviews/edits the file, (3) `--accept FILE` parses and applies the reviewed plan — with `--auto` collapsing all three phases


### cli-sqlite-only-commands-exist
**Status:** IN

Commands `derive`, `ask`, `review-beliefs`, `deduplicate`, and `contradictions` are guarded by `_require_sqlite()` and exit with an error if `--pg` is set — they do not support PostgreSQL


### cli-tests-are-black-box-integration
**Status:** IN

All CLI tests invoke `main()` through the full argv-parsing pipeline via the `run_cli` harness rather than calling internal APIs, with one exception (`TestPropagateWithChanges` directly mutates storage to create inconsistent state).


### cli-uses-lazy-imports-for-heavy-modules
**Status:** IN

`asyncio`, `derive`, `ask`, and `Storage` are imported inside function bodies rather than at module level, keeping `reasons --help` fast.

**Supports:** [startup-performance-uses-lazy-loading](other.md#startup-performance-uses-lazy-loading)

### closed-loop-is-origin-agnostic
**Status:** OUT

The minimality-sustained closed maintenance loop operates identically across all belief origins — external beliefs achieve full integration parity within the same forward-computation and backward-revision cycle as internally-derived beliefs, making the maintenance loop source-agnostic.

**Depends on:** [external-beliefs-achieve-integration-parity](external.md#external-beliefs-achieve-integration-parity), [minimality-sustains-closed-loop-maintenance](other.md#minimality-sustains-closed-loop-maintenance)
**Supports:** [origin-agnostic-loop-grounds-external-invariants](external.md#origin-agnostic-loop-grounds-external-invariants), [origin-agnostic-trustworthiness-is-fully-verifiable](other.md#origin-agnostic-trustworthiness-is-fully-verifiable), [system-is-fully-characterized-self-maintaining-loop](system.md#system-is-fully-characterized-self-maintaining-loop)

### closed-loop-preserves-all-invariants
**Status:** OUT

The closed revision-and-lifecycle maintenance loop not only sustains belief consistency but preserves all system invariants through architecturally grounded enforcement — the loop is both self-maintaining and invariant-preserving.

**Depends on:** [invariant-preservation-is-architecturally-grounded](other.md#invariant-preservation-is-architecturally-grounded), [revision-and-lifecycle-form-closed-loop](revision.md#revision-and-lifecycle-form-closed-loop)
**Supports:** [invariant-preservation-is-comprehensive](other.md#invariant-preservation-is-comprehensive)

### cluster-and-sample-are-mutually-exclusive
**Status:** IN

The `--cluster` and `--sample` flags in `cmd_derive` are mutually exclusive, enforced with an explicit check and `sys.exit(1)` — they represent competing strategies for belief subset selection.


### cluster-auto-k-heuristic
**Status:** IN

Auto cluster count is computed as `len(beliefs) // 5`, clamped between 2 and `min(budget // 3, 20)`, targeting approximately 5 beliefs per cluster with at least 3 beliefs per cluster given the budget.


### cluster-cache-keys-include-content-hash
**Status:** IN

`ClusterCache` keys embeddings by `(node_id, sha256_prefix)`, so editing a belief's text with the same ID forces re-embedding rather than serving stale vectors.


### cluster-cache-no-recompute
**Status:** IN

`ClusterCache.embed()` does not recompute embeddings for previously cached belief texts; cache size stays constant on repeated calls with the same input and grows by exactly the count of new texts on superset calls.


### cluster-deps-are-optional
**Status:** IN

`sentence-transformers` and `scikit-learn` are optional dependencies behind a `HAS_CLUSTER_DEPS` gate; the module degrades to a clear `ImportError` with install instructions when they are absent.


### cluster-deps-optional-with-graceful-skip
**Status:** IN

The clustering module (`reasons_lib.cluster`) is behind an optional `[cluster]` install extra; when `sentence-transformers` or `scikit-learn` are missing, `_require_cluster_deps` raises `ImportError` and all dependent tests skip cleanly.


### cluster-remainder-favors-largest
**Status:** IN

When the budget doesn't divide evenly across clusters, extra slots are distributed one-per-cluster to the largest clusters first via descending size sort.


### cluster-skips-ml-when-under-budget
**Status:** IN

When the number of beliefs is less than or equal to the budget, all ML work (embedding, clustering, sampling) is skipped and every belief is returned directly.


### cluster-stats-sizes-sum-to-input
**Status:** IN

The `cluster_sizes` list in the stats dict returned by `cluster_beliefs` always sums to the total number of input beliefs, enforcing that every belief is assigned to exactly one cluster.


### cmd-propagate-bypasses-api
**Status:** OUT

`cmd_propagate` is the only CLI handler that bypasses `api.py`, going directly to `Storage` → `Network.recompute_all()` → `Storage.save()` — a design inconsistency in the otherwise pure-presentation CLI layer.

**Supports:** [cli-is-pure-delegation-layer](other.md#cli-is-pure-delegation-layer), [cli-is-verified-end-to-end](other.md#cli-is-verified-end-to-end)

### colon-means-already-namespaced
**Status:** IN

`_resolve_namespace` treats a colon in a node ID as "already namespaced" and never double-prefixes; this is the convention for cross-namespace references.

**Supports:** [namespace-is-colon-convention-with-auto-wiring](other.md#namespace-is-colon-convention-with-auto-wiring)

### commands-dict-must-mirror-subparsers
**Status:** IN

Adding a CLI subcommand requires entries in both the argparse subparser definitions and the `commands` dispatch dict in `main()`; omitting either silently breaks the command.


### completeness-and-minimality-are-unified
**Status:** IN

The reasoning-and-revision architecture achieves completeness through minimality rather than despite it — both forward truth computation and backward belief revision derive from the same small set of primitives (outlist, disjunctive truth, vacuous validity), so completeness requires no feature accumulation beyond what minimality already provides.

**Depends on:** [reasoning-and-revision-form-complete-architecture](revision.md#reasoning-and-revision-form-complete-architecture), [semantics-and-revision-share-minimal-foundations](revision.md#semantics-and-revision-share-minimal-foundations)
**Supports:** [complete-architecture-preserves-invariants-minimally](complete.md#complete-architecture-preserves-invariants-minimally), [completeness-determinism-and-minimality-are-unified](other.md#completeness-determinism-and-minimality-are-unified), [evaluation-purity-enables-complete-minimal-architecture](complete.md#evaluation-purity-enables-complete-minimal-architecture), [rich-governance-emerges-from-minimal-foundations](governance.md#rich-governance-emerges-from-minimal-foundations)

### completeness-determinism-and-minimality-are-unified
**Status:** IN

The reasoning-and-revision architecture achieves completeness through minimality, and that same minimality produces operational determinism — completeness and determinism are not independently established but co-derived from shared minimal foundations: uniform outlist primitives simultaneously enable complete revision coverage and deterministic evaluation, revealing a single architectural root for both properties.

**Depends on:** [completeness-and-minimality-are-unified](other.md#completeness-and-minimality-are-unified), [semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism)
**Supports:** [rich-governance-inherits-minimality-completeness-determinism-unity](governance.md#rich-governance-inherits-minimality-completeness-determinism-unity)

### context-agnosticism-follows-from-minimality
**Status:** OUT

Context-agnostic evaluation — producing identical results regardless of evaluation timing, attachment history, or structural origin — is a consequence of semantic minimality with operational determinism: because truth evaluation derives from uniform minimal rules with deterministic pure evaluation, it naturally cannot distinguish between contexts.

**Depends on:** [evaluation-is-uniformly-context-and-origin-agnostic](other.md#evaluation-is-uniformly-context-and-origin-agnostic), [semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism)

### contradiction-dry-run-overrides-auto-apply
**Status:** IN

When both `--dry-run` and `--auto-apply` are passed to the `contradictions` CLI subcommand, no nogoods are recorded in the database (applied count is 0).


### contradiction-in-only-filter
**Status:** IN

`detect_contradictions` excludes OUT nodes from LLM prompts even when explicitly listed in `belief_ids`.


### contradiction-min-two-claims
**Status:** IN

`parse_contradiction_response` drops any nogood with fewer than 2 valid claim IDs; this is enforced both before and after `valid_ids` filtering.


### contradiction-plan-round-trips-apply-entries
**Status:** IN

Writing a contradiction plan with `write_contradiction_plan` and parsing it back with `parse_contradiction_plan` preserves all `[APPLY]`-tagged NOGOOD entries with their IDs and claims, while discarding `[SKIP]`-tagged entries — enabling a human review workflow.


### contradiction-resolution-is-minimal-disruption
**Status:** IN

The nogood resolution system minimizes network disruption through layered heuristics: the primary path traces justification chains back to premises and selects the least-entrenched for retraction, the fallback uses dependent count when no traceable chain exists, and all contradictions are unconditionally recorded regardless of resolution outcome.

**Depends on:** [add-nogood-always-records](other.md#add-nogood-always-records), [add-nogood-fallback-uses-dependent-count](other.md#add-nogood-fallback-uses-dependent-count), [add-nogood-retraction-prefers-least-entrenched](other.md#add-nogood-retraction-prefers-least-entrenched), [backtracking-retracts-least-entrenched](other.md#backtracking-retracts-least-entrenched)
**Supports:** [belief-revision-is-comprehensive-and-minimal](revision.md#belief-revision-is-comprehensive-and-minimal), [belief-revision-is-fully-reliable](revision.md#belief-revision-is-fully-reliable), [contradiction-resolution-minimizes-disruption-and-guides-recovery](other.md#contradiction-resolution-minimizes-disruption-and-guides-recovery), [contradiction-triggers-deterministic-resolution](deterministic.md#contradiction-triggers-deterministic-resolution)

### contradiction-resolution-is-traceable-and-recoverable
**Status:** IN

Contradiction resolution provides complete operational support along two independent dimensions: it minimizes disruption with guided recovery (least-entrenched culprit selection plus surgical restoration hints for cascade victims), AND creates consistently identifiable artifacts (nogoods with durable collision-free IDs, challenge nodes with deterministic auto-IDs), enabling both forensic root-cause analysis and practical guided recovery.

**Depends on:** [contradiction-resolution-minimizes-disruption-and-guides-recovery](other.md#contradiction-resolution-minimizes-disruption-and-guides-recovery), [system-artifacts-maintain-consistent-identification](system.md#system-artifacts-maintain-consistent-identification)
**Supports:** [all-removals-provide-reporting-and-recovery](other.md#all-removals-provide-reporting-and-recovery)

### contradiction-resolution-minimizes-disruption-and-guides-recovery
**Status:** IN

Contradiction resolution achieves both minimal impact and guided recovery: dependency-directed backtracking selects the least-entrenched culprit premise to minimize the retraction cascade, while restoration hints identify specific cascade victims that have surviving alternative premises — providing a complete resolve-and-recover pipeline

**Depends on:** [contradiction-resolution-is-minimal-disruption](other.md#contradiction-resolution-is-minimal-disruption), [restoration-hints-are-surgical](other.md#restoration-hints-are-surgical)
**Supports:** [contradiction-resolution-achieves-minimal-impact-complete-cascades](complete.md#contradiction-resolution-achieves-minimal-impact-complete-cascades), [contradiction-resolution-is-traceable-and-recoverable](other.md#contradiction-resolution-is-traceable-and-recoverable)

### contradictions-belief-text-truncated-at-200-chars
**Status:** IN

`format_beliefs_for_contradiction_check` truncates each belief's text at 200 characters in the LLM prompt to avoid blowing context windows on large belief descriptions


### contradictions-cross-batch-pairs-undetected
**Status:** IN

Batch boundaries are non-overlapping — each belief appears in exactly one batch per run — so contradictions between beliefs in different batches cannot be detected in a single run.


### contradictions-min-two-claims-per-nogood
**Status:** IN

The contradiction parser enforces that every returned nogood has at least 2 valid claim IDs; single-claim or empty results are silently dropped.


### contradictions-no-storage-dependency
**Status:** IN

`contradictions.py` operates on in-memory node dicts and has no import of `storage.py` or any database layer, making it testable in full isolation.


### contradictions-semantic-skips-singleton-clusters
**Status:** IN

`detect_contradictions_semantic` skips any cluster containing fewer than 2 beliefs, since no pairwise contradiction is possible within a singleton


### contradictions-shuffles-before-batching
**Status:** IN

The `contradictions` command randomly shuffles all IN beliefs before partitioning into batches of 50, ensuring that repeated runs probabilistically cover cross-belief comparisons that fixed sequential batching would never surface.


### convergence-produces-evaluation-invariant-equilibria
**Status:** OUT

The system converges to equilibrium states where truth evaluation is transformation-invariant: regardless of the mutation path taken — order of additions, retractions, challenges, imports — the converged state evaluates all beliefs identically, because autonomous convergence reaches deterministic stable states and truth evaluation is agnostic to both temporal context and structural origin.

**Depends on:** [system-autonomously-converges-and-self-corrects](system.md#system-autonomously-converges-and-self-corrects), [truth-evaluation-is-transformation-invariant](other.md#truth-evaluation-is-transformation-invariant)
**Supports:** [autonomous-convergence-produces-documented-equilibria](other.md#autonomous-convergence-produces-documented-equilibria), [canonical-equilibria-are-negation-transparent](other.md#canonical-equilibria-are-negation-transparent), [convergent-equilibria-have-complete-propagation-fidelity](complete.md#convergent-equilibria-have-complete-propagation-fidelity), [equilibrium-trajectory-is-deterministic-and-referenceable](deterministic.md#equilibrium-trajectory-is-deterministic-and-referenceable)

### convergence-trajectories-are-permanently-documented
**Status:** OUT

Every convergence trajectory toward an evaluation-invariant equilibrium — deterministic in path and consistently identifiable in its artifacts — is backed by a permanent, comprehensive audit trail covering all self-corrections along that trajectory, ensuring complete retrospective analysis of how the system reached any given stable state

**Depends on:** [equilibrium-trajectory-is-deterministic-and-referenceable](deterministic.md#equilibrium-trajectory-is-deterministic-and-referenceable), [self-correction-audit-trail-is-permanent-and-comprehensive](self.md#self-correction-audit-trail-is-permanent-and-comprehensive)
**Supports:** [equilibria-are-transparent-and-trajectory-documented](other.md#equilibria-are-transparent-and-trajectory-documented)

### convergent-equilibria-are-documented-and-indefinitely-auditable
**Status:** OUT

The system's convergent equilibria are simultaneously trajectory-documented (every path to equilibrium generates deterministic identifiable artifacts with negation-transparent final states) and indefinitely auditable (every invariant in the equilibrium state is independently verifiable without temporal degradation), providing complete operational transparency across both the convergence journey and the resulting stable state.

**Depends on:** [equilibria-are-transparent-and-trajectory-documented](other.md#equilibria-are-transparent-and-trajectory-documented), [total-preservation-is-indefinitely-auditable](other.md#total-preservation-is-indefinitely-auditable)
**Supports:** [verified-correctness-has-indefinitely-auditable-equilibria](other.md#verified-correctness-has-indefinitely-auditable-equilibria)

### convert-to-premise-preserves-dependents-invariant
**Status:** IN

Converting a derived node to a premise correctly maintains the dependents index by removing the node from former antecedents' dependents sets — the same invariant maintained by every other network mutation.

**Depends on:** [convert-to-premise-removes-dependents](other.md#convert-to-premise-removes-dependents), [every-network-mutation-maintains-dependents](other.md#every-network-mutation-maintains-dependents)
**Supports:** [premise-identity-is-bidirectionally-transformable](other.md#premise-identity-is-bidirectionally-transformable)

### convert-to-premise-removes-dependents
**Status:** IN

When a derived node is converted to a premise via `convert_to_premise`, it is removed from its former antecedents' `dependents` sets because the old justification edges are deleted.

**Supports:** [convert-to-premise-preserves-dependents-invariant](other.md#convert-to-premise-preserves-dependents-invariant)

### corrections-span-all-origins-with-full-auditability
**Status:** OUT

Every correction mechanism — intentional dialectical challenge/defend and automated contradiction resolution — is both reliable and fully auditable with traceable history, and this complete correction coverage spans all belief origins (human, LLM, agent) — no belief from any provenance can undergo an untraced or unreliable correction.

**Depends on:** [all-corrections-are-reliable-and-auditable](other.md#all-corrections-are-reliable-and-auditable), [dispute-resolution-spans-all-origins](spans.md#dispute-resolution-spans-all-origins)
**Supports:** [external-beliefs-are-correctable-and-invariant-equivalent](external.md#external-beliefs-are-correctable-and-invariant-equivalent), [revision-is-evaluation-invariant-and-auditable-across-origins](revision.md#revision-is-evaluation-invariant-and-auditable-across-origins), [revision-system-is-reliable-and-auditable](revision.md#revision-system-is-reliable-and-auditable), [self-maintenance-is-fully-auditable](self.md#self-maintenance-is-fully-auditable)

### count-accumulates-linearly
**Status:** IN

Documents the bug fix for issue #23 — already covered by existing `derive-agent-count-bug` which tracks this defect


### cp-and-sl-evaluated-identically
**Status:** IN

CP and SL justifications use the same validity check in `_justification_valid`; the distinction is semantic (support vs. consistency), not computational.

**Supports:** [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure)

### cp-equals-sl
**Status:** IN

CP (conditional-proof) justifications are evaluated with the exact same logic as SL justifications, despite being a distinct type in Doyle's TMS — either an intentional simplification or incomplete implementation


### critical-operations-converge-to-fixed-points
**Status:** OUT

The system's three critical reconciliation operations are all convergent: agent sync produces no changes on re-run with identical input, dependents index rebuilding yields identical results on repeated execution, and truth recomputation iterates to a fixpoint — ensuring the system reaches stable consistent state regardless of operation ordering.

**Depends on:** [rebuild-dependents-is-idempotent](other.md#rebuild-dependents-is-idempotent), [recompute-all-uses-fixpoint](other.md#recompute-all-uses-fixpoint), [sync-agent-idempotent](agent.md#sync-agent-idempotent)
**Supports:** [all-reconciliation-converges-deterministically](other.md#all-reconciliation-converges-deterministically)

### dangling-dependent-guard-skips-missing-nodes
**Status:** IN

`_propagate` skips dependent IDs not present in `net.nodes` rather than raising `KeyError`, and emits a structured warning log entry for each (fix for issue #22).

**Supports:** [dangling-dependents-are-safely-contained](other.md#dangling-dependents-are-safely-contained)

### dangling-dependents-are-safely-contained
**Status:** IN

Dangling dependent references are safely contained across all propagation dimensions: BFS skips missing nodes with structured warnings, the changed set never includes ghost IDs, and the visited set excludes dangling IDs so later-created nodes propagate normally.

**Depends on:** [dangling-dependent-guard-skips-missing-nodes](other.md#dangling-dependent-guard-skips-missing-nodes), [dangling-ids-excluded-from-changed](other.md#dangling-ids-excluded-from-changed), [dangling-ids-excluded-from-visited](other.md#dangling-ids-excluded-from-visited)
**Supports:** [propagation-is-safe-under-graph-inconsistency](safe.md#propagation-is-safe-under-graph-inconsistency)

### dangling-dependents-log-not-crash
**Status:** IN

Covered by existing `propagate-assumes-dependents-exist` (documents the assumption) and `tms-core-is-crash-safe` (documents crash safety)


### dangling-guard-is-continue-not-raise
**Status:** IN

When `_propagate` encounters a dependent ID not in `self.nodes`, it logs a structured warning and continues the BFS loop rather than raising `KeyError` or silently skipping.


### dangling-ids-excluded-from-changed
**Status:** IN

The `changed` set returned by `retract()` and `assert_node()` never contains IDs that don't correspond to real nodes in the network.

**Supports:** [dangling-dependents-are-safely-contained](other.md#dangling-dependents-are-safely-contained)

### dangling-ids-excluded-from-visited
**Status:** IN

The propagation visited set does not include dangling IDs, so a formerly-dangling ID that becomes a real node will propagate correctly on subsequent operations.

**Supports:** [dangling-dependents-are-safely-contained](other.md#dangling-dependents-are-safely-contained)

### dangling-refs-excluded-from-changed-set
**Status:** IN

The `changed` set returned by `retract`/`assert_node` never contains node IDs that don't exist in the network.


### dangling-refs-excluded-from-visited-set
**Status:** IN

Dangling dependent IDs are not added to the propagation visited set, so later-created nodes with the same ID propagate normally.


### data-model-uses-string-enums
**Status:** IN

Both `Justification.type` ("SL"/"CP") and `Node.truth_value` ("IN"/"OUT") are plain strings, not Python enums; consumers must validate values themselves as invalid states like `"MAYBE"` or `"XYZ"` are representable.


### dedup-auto-keeps-most-dependents
**Status:** IN

In auto mode, `deduplicate` retains the cluster member with the most dependents and retracts all others


### dedup-keeps-most-connected-node
**Status:** IN

In auto-dedup mode, the node with the most dependents survives each cluster; ties break by lexicographic ID, and losers are retracted after dependents are rewired.

**Supports:** [dedup-is-topology-preserving-and-auditable](topology.md#dedup-is-topology-preserving-and-auditable), [dedup-survivor-selection-is-topology-reliable](topology.md#dedup-survivor-selection-is-topology-reliable)

### dedup-plan-is-user-editable
**Status:** IN

The dedup plan format uses KEEP/RETRACT markers that users can swap before applying, making deduplication decisions reviewable and overridable

**Supports:** [dedup-is-topology-preserving-and-auditable](topology.md#dedup-is-topology-preserving-and-auditable)

### dedup-rewrites-both-antecedents-and-outlist
**Status:** IN

When a duplicate is retracted via dedup, all justification references (both antecedent and outlist) across the network are rewritten to point at the kept node

**Supports:** [dedup-is-topology-preserving-and-auditable](topology.md#dedup-is-topology-preserving-and-auditable), [dedup-survivor-selection-is-topology-reliable](topology.md#dedup-survivor-selection-is-topology-reliable)

### defeat-reversal-is-automatic-with-guided-recovery
**Status:** IN

All outlist-based defeat mechanisms (challenge, kill-switch, supersession) not only reverse automatically through BFS propagation cascades — recovering all transitively dependent nodes — but also provide surgical recovery guidance through restoration hints that target cascade victims with surviving premises, enabling both automatic and manual recovery paths.

**Depends on:** [defeat-reversal-propagates-automatically](other.md#defeat-reversal-propagates-automatically), [defeat-reversal-with-guided-recovery](other.md#defeat-reversal-with-guided-recovery)
**Supports:** [all-revision-mechanisms-are-traceable-and-recoverable](revision.md#all-revision-mechanisms-are-traceable-and-recoverable), [belief-modification-is-bidirectionally-complete-and-traceable](complete.md#belief-modification-is-bidirectionally-complete-and-traceable), [defeat-reversal-is-topology-complete-with-guided-recovery](topology.md#defeat-reversal-is-topology-complete-with-guided-recovery), [negative-semantics-are-complete-reversible-and-recoverable](complete.md#negative-semantics-are-complete-reversible-and-recoverable)

### defeat-reversal-propagates-automatically
**Status:** IN

All outlist-based defeat mechanisms (challenge, kill-switch, supersession) not only reverse in principle but propagate recovery automatically through safe terminating BFS — when a defeating node is retracted, the outlist entry becomes satisfied, and propagation cascades truth-value restoration to all affected nodes without manual re-assertion

**Depends on:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating)
**Supports:** [all-truth-effects-propagate-through-outlist-paths](other.md#all-truth-effects-propagate-through-outlist-paths), [defeat-reversal-is-automatic-with-guided-recovery](other.md#defeat-reversal-is-automatic-with-guided-recovery)

### defeat-reversal-with-guided-recovery
**Status:** IN

All defeat mechanisms (challenge, kill-switch, supersession) are reversible through outlist semantics, and the system provides surgical restoration hints for cascade victims with viable recovery paths — enabling guided recovery from retraction cascades where multi-premise justifications have surviving premises.

**Depends on:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [restoration-hints-require-surviving-premises](other.md#restoration-hints-require-surviving-premises)
**Supports:** [defeat-reversal-is-automatic-with-guided-recovery](other.md#defeat-reversal-is-automatic-with-guided-recovery)

### defend-is-challenge-of-challenge
**Status:** IN

`defend` works by calling `challenge` on the challenge node itself, creating a recursive dialectical structure where truth values resolve automatically through the same outlist mechanism.


### defend-is-recursive-challenge
**Status:** IN

Defense is implemented by calling `challenge()` on the challenge node itself, enabling arbitrarily deep dialectical chains using the same outlist mechanism recursively with no special-case code

**Supports:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [dialectical-structure-is-recursive-outlist](other.md#dialectical-structure-is-recursive-outlist)

### defense-in-depth-is-resource-efficient
**Status:** OUT

The system's defense-in-depth across LLM and system boundaries — layered defenses including bounded execution, fail-soft error handling, process isolation, and referential integrity validation — achieves comprehensive protection within the same resource-efficient pipeline that spans packaging, startup, and runtime with zero external dependencies and lazy loading.

**Depends on:** [defense-in-depth-spans-llm-and-system-boundaries](spans.md#defense-in-depth-spans-llm-and-system-boundaries), [resource-efficiency-spans-full-pipeline](spans.md#resource-efficiency-spans-full-pipeline)
**Supports:** [operational-safety-is-resource-efficient-defense-in-depth](other.md#operational-safety-is-resource-efficient-defense-in-depth)

### deferred-retraction-ordering
**Status:** IN

During agent import, nodes are added and truth values propagated before explicit retractions are applied, ensuring the dependency graph is fully wired before OUT transitions are forced.

**Supports:** [import-ordering-ensures-correct-final-state](import.md#import-ordering-ensures-correct-final-state)

### dependency-completeness-enables-accurate-dedup
**Status:** IN

Complete dependency tracking for all reference types — both antecedent and outlist entries maintained eagerly by every network mutation — ensures deduplication accurately reflects the complete network topology: survivor selection considers all incoming dependencies including outlist references, and reference rewiring targets both antecedent and outlist positions across all justifications, preventing dedup from creating dangling references or miscounting dependents.

**Depends on:** [dedup-reflects-complete-dependency-graph](complete.md#dedup-reflects-complete-dependency-graph), [dependency-tracking-is-complete-for-all-reference-types](complete.md#dependency-tracking-is-complete-for-all-reference-types)
**Supports:** [topology-soundness-is-accurate-and-convergent](topology.md#topology-soundness-is-accurate-and-convergent)

### dependents-bidirectional-index
**Status:** OUT

Each node maintains a `dependents` set (reverse of antecedent/outlist edges), eagerly maintained by `add_node`, `add_justification`, `supersede`, `challenge`, and `convert_to_premise`.

**Supports:** [dependents-index-is-fragile-denormalization](other.md#dependents-index-is-fragile-denormalization)

### dependents-index-derived-on-load
**Status:** OUT

The `node.dependents` set is never persisted to SQLite; it is rebuilt by walking all justification antecedents and outlists during `load()`.

**Supports:** [dependents-index-is-fragile-denormalization](other.md#dependents-index-is-fragile-denormalization), [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental)

### dependents-index-is-fragile-denormalization
**Status:** OUT

The dependents set is a manually-maintained denormalized reverse index that is never persisted and must be rebuilt on every load, creating a consistency obligation on all mutation paths

**Depends on:** [dependents-bidirectional-index](other.md#dependents-bidirectional-index), [dependents-index-derived-on-load](other.md#dependents-index-derived-on-load), [dependents-is-manual-reverse-index](other.md#dependents-is-manual-reverse-index)
**Supports:** [architecture-has-no-hidden-fragility](other.md#architecture-has-no-hidden-fragility), [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [dedup-survivor-selection-is-topology-reliable](topology.md#dedup-survivor-selection-is-topology-reliable), [full-system-integrity-is-gap-free](system.md#full-system-integrity-is-gap-free), [mutation-pipeline-produces-consistent-state](other.md#mutation-pipeline-produces-consistent-state), [operational-integrity-survives-all-graph-states](other.md#operational-integrity-survives-all-graph-states), [persistence-round-trip-is-lossless](other.md#persistence-round-trip-is-lossless), [unified-system-is-a-closed-self-maintaining-architecture](system.md#unified-system-is-a-closed-self-maintaining-architecture)

### dependents-is-manual-reverse-index
**Status:** OUT

`Node.dependents` is a denormalized reverse pointer set that must be kept in sync by external code (primarily `network.py`); nothing in the data model enforces consistency.

**Supports:** [dependents-index-is-fragile-denormalization](other.md#dependents-index-is-fragile-denormalization)

### dependents-survive-storage-roundtrip
**Status:** IN

After `Storage.save()` followed by `Storage.load()`, the loaded network's dependents index passes `verify_dependents()` with no errors.


### derived-belief-pipeline-achieves-code-enforced-quality
**Status:** OUT

The derived belief pipeline — creation via defensive derivation with structural validation, Jaccard retraction guards, and environment isolation, followed by independent review with scope restricted to derived beliefs and mutation gated behind dry-run — achieves completely code-enforced quality assurance including logical soundness validation, only when inference soundness checking is implemented in code rather than relying solely on LLM prompt instructions.

**Depends on:** [derive-pipeline-is-safe-and-complete](derive.md#derive-pipeline-is-safe-and-complete), [review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe)

### dialectical-defeat-is-reversible-but-identity-is-permanent
**Status:** IN

The dialectical system exhibits a fundamental asymmetry between defeat and identity: the truth-value defeat caused by a challenge is fully reversible (defending or retracting the challenge node restores IN status via outlist semantics), but the premise-to-justified identity transformation is permanent — a challenged premise can never return to unjustified status because the added justification cannot be removed, only defeated.

**Depends on:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [challenge-destroys-premise-identity](other.md#challenge-destroys-premise-identity)
**Supports:** [negative-semantics-have-reversible-defeat-but-permanent-identity-effects](other.md#negative-semantics-have-reversible-defeat-but-permanent-identity-effects), [premise-identity-transformation-is-architecturally-asymmetric](other.md#premise-identity-transformation-is-architecturally-asymmetric)

### dialectical-structure-is-recursive-outlist
**Status:** IN

The entire challenge/defend dialectical system is implemented as recursive outlist injection with no dedicated dialectical machinery

**Depends on:** [challenge-is-outlist-injection](other.md#challenge-is-outlist-injection), [challenge-modifies-all-justifications](other.md#challenge-modifies-all-justifications), [challenge-uses-outlist-mechanism](other.md#challenge-uses-outlist-mechanism), [defend-is-recursive-challenge](other.md#defend-is-recursive-challenge)
**Supports:** [challenge-defense-is-crash-safe](safe.md#challenge-defense-is-crash-safe), [dialectics-inherit-complete-outlist-semantics](complete.md#dialectics-inherit-complete-outlist-semantics), [non-monotonic-system-is-single-reversible-primitive](system.md#non-monotonic-system-is-single-reversible-primitive)

### dialectical-transformation-is-fully-reliable
**Status:** IN

The irreversible premise-to-justified transformation during challenge is both semantics-preserving (the resulting node inherits complete outlist evaluation with conjunction, absence, and persistence semantics) and crash-safe (recursive dialectical chains terminate deterministically), making dialectical operations reliable despite their irreversibility.

**Depends on:** [challenge-defense-is-crash-safe](safe.md#challenge-defense-is-crash-safe), [dialectical-transformation-preserves-semantics](other.md#dialectical-transformation-preserves-semantics)
**Supports:** [all-exceptions-are-safely-handled](other.md#all-exceptions-are-safely-handled), [dialectics-are-deterministic-and-reliable](deterministic.md#dialectics-are-deterministic-and-reliable), [dispute-resolution-is-complete-and-reliable](complete.md#dispute-resolution-is-complete-and-reliable), [identity-transformation-is-complete-and-reliable](complete.md#identity-transformation-is-complete-and-reliable)

### dialectical-transformation-preserves-semantics
**Status:** IN

Challenging a premise irreversibly transforms its identity from unjustified to justified node, but the resulting dialectical structure inherits complete outlist semantics — conjunction over multiple outlists, absence-as-OUT permissiveness, and persistence survival — ensuring the transformation preserves well-defined evaluable behavior.

**Depends on:** [challenge-destroys-premise-identity](other.md#challenge-destroys-premise-identity), [dialectics-inherit-complete-outlist-semantics](complete.md#dialectics-inherit-complete-outlist-semantics)
**Supports:** [both-revision-paths-preserve-system-invariants](revision.md#both-revision-paths-preserve-system-invariants), [dialectical-transformation-is-fully-reliable](other.md#dialectical-transformation-is-fully-reliable), [dialectical-transformation-is-operationally-safe](safe.md#dialectical-transformation-is-operationally-safe), [revision-is-lifecycle-safe-and-semantics-preserving](revision.md#revision-is-lifecycle-safe-and-semantics-preserving)

### dialectics-achieve-forward-reliability-and-backward-recovery
**Status:** IN

Dialectical operations achieve complete bidirectional assurance: forward activation is deterministic, reliable, and semantically complete (challenge/defend evaluated uniformly with controlled irreversibility), while backward reversal is topology-complete with surgical guided recovery (recovery reaches all transitively dependent nodes, hints target only cascade victims with surviving premises) — the full dialectical cycle from engagement through resolution is assured in both directions.

**Depends on:** [defeat-reversal-is-topology-complete-with-guided-recovery](topology.md#defeat-reversal-is-topology-complete-with-guided-recovery), [dialectical-revision-is-deterministic-reliable-and-complete](revision.md#dialectical-revision-is-deterministic-reliable-and-complete)
**Supports:** [grounded-dialectics-achieve-complete-bidirectional-assurance](complete.md#grounded-dialectics-achieve-complete-bidirectional-assurance)

### dialectics-are-atomic-and-transparent
**Status:** OUT

Challenge/defend dialectics are both semantically transparent (indistinguishable from ordinary beliefs, evaluated by uniform outlist rules) and atomically safe (mutations follow the same context-managed load/save pipeline as all other operations), requiring no special transaction handling.

**Depends on:** [dialectics-are-semantically-transparent](other.md#dialectics-are-semantically-transparent), [mutations-are-atomic-and-safely-propagated](other.md#mutations-are-atomic-and-safely-propagated)
**Supports:** [dialectics-complete-the-revision-system](complete.md#dialectics-complete-the-revision-system), [extensions-compose-transparently-on-core](other.md#extensions-compose-transparently-on-core)

### dialectics-are-dually-grounded-by-purity-and-uniformity
**Status:** IN

Dialectical operations achieve dual semantic grounding from independent sources: evaluation purity (uniform, deterministic, side-effect-free validity checking) enables richly-governed exception-safe dialectics, while uniform edge-case semantics transitively ground deterministic reliable dialectics through complete negative semantics — together ensuring dialectics are both governable and semantically well-founded from first principles.

**Depends on:** [pure-evaluation-enables-richly-governed-dialectics](other.md#pure-evaluation-enables-richly-governed-dialectics), [uniform-semantics-transitively-ground-deterministic-dialectics](deterministic.md#uniform-semantics-transitively-ground-deterministic-dialectics)
**Supports:** [governance-has-dual-independent-grounding-chains](governance.md#governance-has-dual-independent-grounding-chains), [grounded-dialectics-achieve-complete-bidirectional-assurance](complete.md#grounded-dialectics-achieve-complete-bidirectional-assurance)

### dialectics-are-semantically-transparent
**Status:** IN

Challenge/defend dialectics are semantically indistinguishable from ordinary beliefs: they inherit fully-specified outlist semantics (conjunction, absence-as-OUT, persistence) and are evaluated by the same uniform pure rules that govern all truth maintenance — no dialectical special cases exist anywhere in the engine.

**Depends on:** [dialectics-inherit-complete-outlist-semantics](complete.md#dialectics-inherit-complete-outlist-semantics), [truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)
**Supports:** [dialectics-are-atomic-and-transparent](other.md#dialectics-are-atomic-and-transparent), [dialectics-are-deterministic-by-transparency](deterministic.md#dialectics-are-deterministic-by-transparency), [evaluation-is-uniformly-context-and-origin-agnostic](other.md#evaluation-is-uniformly-context-and-origin-agnostic), [identity-transformation-is-semantically-invisible](other.md#identity-transformation-is-semantically-invisible)

### direct-access-raises-list-access-filters
**Status:** IN

API functions for single-node access (`show_node`, `explain_node`, `trace_assumptions`) raise `PermissionError` on forbidden nodes, while list/export functions (`list_nodes`, `search`, `export_network`) silently exclude them from results.


### dry-run-prevents-both-retraction-and-metadata
**Status:** IN

The `dry_run` flag in review-beliefs prevents both truth-value changes (retraction of invalid beliefs) and metadata side-effects (`last_reviewed` timestamp, `review_result` classification), making it fully read-only — extending beyond what `auto-retract-respects-dry-run` covers.


### dual-quality-gates-are-complementary-and-non-mutating
**Status:** IN

The system enforces belief quality through dual non-mutating gates targeting complementary validity dimensions: review validates logical soundness of derived beliefs (scoped to justified nodes, dry-run gated auto-retraction), while staleness checking validates source currency of all IN beliefs (conservative CI gate with nonzero exit on drift) — neither gate can corrupt network state.

**Depends on:** [review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)

### dual-storage-backends-are-interchangeable
**Status:** IN

Both SQLite and PostgreSQL backends can be used interchangeably for any operation, with identical safety guarantees through backend-appropriate mechanisms and complete API surface coverage, so applications can switch backends without behavioral changes.

**Depends on:** [atomicity-is-backend-independent](other.md#atomicity-is-backend-independent), [storage-layer-is-backend-agnostic-and-safe](safe.md#storage-layer-is-backend-agnostic-and-safe)

### duplicate-node-id-raises-valueerror
**Status:** IN

`api.add_node()` raises `ValueError` when given a node ID that already exists in the network — node IDs are unique.

**Supports:** [api-enforces-typed-preconditions](other.md#api-enforces-typed-preconditions)

### each-cli-test-creates-isolated-db
**Status:** IN

Every CLI test method initializes a fresh SQLite database via `run_cli("init")` in a pytest `tmp_path`, ensuring zero shared state between tests.

**Supports:** [cli-is-verified-end-to-end](other.md#cli-is-verified-end-to-end)

### edge-case-uniformity-follows-from-minimality
**Status:** IN

Uniform handling of all semantic edge cases — vacuous premises, asymmetric absence, empty antecedents — is a consequence of semantic minimality: because every edge case derives from the same primitives that drive deterministic core semantics, no special-case logic exists.

**Depends on:** [belief-revision-covers-all-cases-uniformly](revision.md#belief-revision-covers-all-cases-uniformly), [semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism)
**Supports:** [edge-case-safety-spans-creation-and-maintenance](spans.md#edge-case-safety-spans-creation-and-maintenance), [edge-case-uniformity-reinforces-complete-negative-semantics](complete.md#edge-case-uniformity-reinforces-complete-negative-semantics), [minimality-generates-universal-revision-safety](revision.md#minimality-generates-universal-revision-safety), [minimality-produces-uniformity-and-determinism](other.md#minimality-produces-uniformity-and-determinism), [minimality-yields-extensibility-and-robustness](other.md#minimality-yields-extensibility-and-robustness), [negative-semantics-are-uniform-through-minimality](other.md#negative-semantics-are-uniform-through-minimality), [revision-completeness-follows-from-minimality](revision.md#revision-completeness-follows-from-minimality), [safety-and-uniformity-are-co-derived](other.md#safety-and-uniformity-are-co-derived)

### empty-antecedents-vacuously-valid
**Status:** IN

An SL justification with an empty antecedent list is valid (vacuous truth via `all([])`), allowing outlist-only justifications to function as "IN unless Y" — used by `challenge` and `supersede` for converted premises

**Supports:** [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence)

### entry-point-mapping
**Status:** IN

The `reasons` CLI command maps to `reasons_lib.cli:main` via `[project.scripts]` in `pyproject.toml` and is the only registered script entry point.


### equilibria-are-transparent-and-trajectory-documented
**Status:** OUT

The system's convergent equilibria are simultaneously negation-transparent (the final stable state is uniquely determined by evaluation rules with complete propagation fidelity) and trajectory-documented (every convergence path generates deterministic traceable events backed by permanent durable audit trails) — convergence is not just mathematically guaranteed but operationally verifiable.

**Depends on:** [convergence-trajectories-are-permanently-documented](other.md#convergence-trajectories-are-permanently-documented), [knowledge-growth-reaches-transparent-equilibria](other.md#knowledge-growth-reaches-transparent-equilibria)
**Supports:** [convergent-equilibria-are-documented-and-indefinitely-auditable](other.md#convergent-equilibria-are-documented-and-indefinitely-auditable)

### estimate-tokens-chars-div-4
**Status:** IN

`estimate_tokens` uses `len(text) // 4` with a minimum return value of 1; it never returns 0, even for empty strings.


### evaluation-is-traceable-and-context-agnostic
**Status:** OUT

Every truth evaluation is simultaneously context-agnostic (producing identical results regardless of evaluation timing, attachment history, or belief origin) and fully traceable (every state change follows a deterministic path recorded in the system's operational history), enabling both reproducibility and post-hoc explanation of any evaluation outcome.

**Depends on:** [complete-system-history-is-deterministic-and-traceable](complete.md#complete-system-history-is-deterministic-and-traceable), [evaluation-is-uniformly-context-and-origin-agnostic](other.md#evaluation-is-uniformly-context-and-origin-agnostic)
**Supports:** [evaluation-traceability-persists-through-equilibria](other.md#evaluation-traceability-persists-through-equilibria)

### evaluation-is-uniformly-context-and-origin-agnostic
**Status:** IN

Truth evaluation produces identical results regardless of both attachment history (when/how a justification was added) and structural origin (ordinary belief vs. dialectical construct) — no belief receives special treatment based on provenance, timing, or role in the network.

**Depends on:** [dialectics-are-semantically-transparent](other.md#dialectics-are-semantically-transparent), [justification-evaluation-is-context-independent](justification.md#justification-evaluation-is-context-independent)
**Supports:** [context-agnosticism-follows-from-minimality](other.md#context-agnosticism-follows-from-minimality), [evaluation-is-traceable-and-context-agnostic](other.md#evaluation-is-traceable-and-context-agnostic), [evaluation-purity-grounds-agnosticism-and-minimality](other.md#evaluation-purity-grounds-agnosticism-and-minimality), [truth-evaluation-is-transformation-invariant](other.md#truth-evaluation-is-transformation-invariant)

### evaluation-purity-grounds-agnosticism-and-minimality
**Status:** IN

Evaluation purity (uniform, deterministic, no metadata inspection) independently grounds both context-agnosticism (identical results regardless of timing/origin) and semantic minimality (no special-case logic), making them co-occurring consequences of the same architectural choice rather than causally related.

**Depends on:** [evaluation-is-uniformly-context-and-origin-agnostic](other.md#evaluation-is-uniformly-context-and-origin-agnostic), [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure), [semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism)
**Supports:** [evaluation-purity-enables-complete-minimal-architecture](complete.md#evaluation-purity-enables-complete-minimal-architecture)

### evaluation-purity-grounds-dialectics-through-minimal-architecture
**Status:** IN

Evaluation purity — uniform, deterministic, side-effect-free justification validity checking — enables the complete minimal architecture whose negative semantics ground deterministic dialectics, establishing a causal chain from the most fundamental computational property through architectural completeness to dialectical reliability.

**Depends on:** [evaluation-purity-enables-complete-minimal-architecture](complete.md#evaluation-purity-enables-complete-minimal-architecture), [negative-semantics-ground-deterministic-dialectics](deterministic.md#negative-semantics-ground-deterministic-dialectics)
**Supports:** [pure-evaluation-enables-richly-governed-dialectics](other.md#pure-evaluation-enables-richly-governed-dialectics)

### evaluation-traceability-persists-through-equilibria
**Status:** OUT

Every truth evaluation is traceable and context-agnostic from individual computation through system-wide convergence: all structural transformations converge to documented equilibria with deterministic identifiable artifacts, and every evaluation along those convergence trajectories is deterministically reproducible regardless of timing or origin.

**Depends on:** [evaluation-is-traceable-and-context-agnostic](other.md#evaluation-is-traceable-and-context-agnostic), [transformations-converge-to-documented-equilibria](other.md#transformations-converge-to-documented-equilibria)
**Supports:** [operational-profile-is-traceable-through-equilibria](other.md#operational-profile-is-traceable-through-equilibria)

### every-mutation-reports-its-effects
**Status:** OUT

All mutating operations report their effects as structured data: retract returns the full changed set, add_justification returns a change dict with old/new truth values, and API mutating operations use before/after truth-value diffing to capture deltas.

**Depends on:** [add-justification-returns-change-dict](justification.md#add-justification-returns-change-dict), [api-mutating-ops-use-before-after-diffing](other.md#api-mutating-ops-use-before-after-diffing), [retract-returns-changed-set](other.md#retract-returns-changed-set)
**Supports:** [retraction-effects-are-reported-with-recovery-guidance](other.md#retraction-effects-are-reported-with-recovery-guidance)

### every-network-mutation-maintains-dependents
**Status:** IN

After any public mutation method on `Network` (`add_node`, `retract`, `assert_node`, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, `summarize`), `verify_dependents()` returns an empty list.

**Supports:** [convert-to-premise-preserves-dependents-invariant](other.md#convert-to-premise-preserves-dependents-invariant), [dependency-tracking-is-complete-for-all-reference-types](complete.md#dependency-tracking-is-complete-for-all-reference-types), [network-mutations-are-audited-and-index-consistent](other.md#network-mutations-are-audited-and-index-consistent)

### every-network-mutation-maintains-dependents-invariant
**Status:** IN

After any public Network mutation (add_node, retract, assert_node, add_justification, supersede, challenge, defend, convert_to_premise, add_nogood, summarize), the dependents index passes `verify_dependents()` — completeness and minimality are maintained incrementally


### exhaust-implies-auto
**Status:** IN

In `_derive_one_round`, proposals are auto-applied when either `args.auto` or `args.exhaust` is true; the `--exhaust` flag does not require the user to also pass `--auto`.

**Supports:** [derive-pipeline-is-exhaustive-and-terminating](derive.md#derive-pipeline-is-exhaustive-and-terminating)

### exhaustive-knowledge-expansion-within-controlled-boundaries
**Status:** OUT

The system achieves exhaustive knowledge expansion — deterministic reversible reasoning combined with complete LLM-driven derivation with guaranteed termination — within multi-level information boundaries that gate authorization, constrain output size, and defensively validate all ingested beliefs, ensuring unbounded knowledge growth never escapes system controls.

**Depends on:** [information-boundaries-are-controlled-at-all-levels](other.md#information-boundaries-are-controlled-at-all-levels), [reasoning-and-knowledge-expansion-are-both-exhaustive](other.md#reasoning-and-knowledge-expansion-are-both-exhaustive)
**Supports:** [knowledge-expansion-is-exhaustive-within-hardened-boundaries](other.md#knowledge-expansion-is-exhaustive-within-hardened-boundaries)

### expert-pipeline-extracts-per-document
**Status:** IN

The expert-agent-builder pipeline extracts beliefs per-document (summarize entire document → propose beliefs → record file path), not per-section — the connection between a belief and its source material is a file-level pointer, not a section-level one.


### export-markdown-pg-reconstructs-network
**Status:** IN

The `export_markdown` PostgreSQL path reconstructs a full `Network` object from `export_network()` output — creating Node/Justification objects and wiring the dependents index — because the markdown exporter requires a wired dependency graph, not a flat dict.


### extensions-compose-transparently-on-core
**Status:** OUT

Both extension systems — dialectical challenge/defend and multi-agent federation — compose transparently on the core TMS because each is evaluated by uniform outlist rules, propagated deterministically, reversed by the same primitive, and isolated from the other's namespace.

**Depends on:** [dialectics-are-atomic-and-transparent](other.md#dialectics-are-atomic-and-transparent), [multi-agent-reasoning-is-sound-and-scalable](agent.md#multi-agent-reasoning-is-sound-and-scalable)
**Supports:** [minimality-yields-extensibility-and-robustness](other.md#minimality-yields-extensibility-and-robustness)

### extract-tool-call-returns-first-match
**Status:** IN

When LLM output contains multiple JSON objects with a `"tool"` key, `extract_tool_call()` returns only the first valid match and ignores the rest; malformed JSON lines are silently skipped.


### extras-map-one-to-one-to-modules
**Status:** IN

Each optional dependency group maps 1:1 to a specific module: `pg` extra gates `reasons_lib/pg.py`, `cluster` extra gates `reasons_lib/cluster.py`, and `test-pg` is a superset combining `pg` and `test`.


### format-resilient-boundaries-enforce-validated-trust
**Status:** OUT

All system boundaries simultaneously tolerate format variation at every level — from LLM response parsing through schema migration to derive output versioning — while enforcing strict validated trust through typed exceptions, referential integrity checks, and hallucination filtering.

**Depends on:** [format-resilience-spans-all-external-interfaces](spans.md#format-resilience-spans-all-external-interfaces), [system-boundaries-are-validating-and-evolution-tolerant](system.md#system-boundaries-are-validating-and-evolution-tolerant)
**Supports:** [trust-enforcement-is-structural-and-operationally-resilient](other.md#trust-enforcement-is-structural-and-operationally-resilient)

### ftl-reasons-zero-runtime-deps
**Status:** IN

The core `reasons_lib` package has no mandatory runtime dependencies — all external packages (psycopg, sentence-transformers, scikit-learn, mcp) are gated behind optional install extras (`[pg]`, `[cluster]`, `[mcp]`).


### fts-errors-silently-caught-in-search
**Status:** IN

FTS5 query errors in `_fts_search` are silently caught and return an empty list, falling back to substring matching — the only place in the API where errors are deliberately swallowed (FTS5 table may not exist).


### fts-progressive-relaxation
**Status:** IN

When a multi-term FTS query returns no results, the search engine progressively drops terms until it finds matches or exhausts all subsets.


### fts-relaxation-bounded
**Status:** IN

Progressive FTS query relaxation is bounded: a 20-term query produces at most 51 `_fts_query` invocations, preventing unbounded search expansion on long input queries.


### fts-relaxation-budget-caps-at-50
**Status:** IN

Progressive FTS5 search relaxation — dropping query terms via combinations when the full-term query returns no results — is capped at `_MAX_RELAXATION_QUERIES` (50) to prevent combinatorial blowup on many-term queries.


### fts-relaxation-capped-at-fifty-queries
**Status:** IN

`_fts_search` caps progressive term relaxation at 50 FTS5 queries to prevent combinatorial explosion on long search inputs, dropping terms one at a time via `combinations` down to `len(terms) // 2`.


### fts-stop-words-filtered-before-query
**Status:** IN

FTS queries filter a `_STOP_WORDS` frozenset before querying FTS5; if all terms are stop words, the search falls back to terms longer than 1 character.


### full-user-stack-is-verified-atomic-delegation
**Status:** OUT

The full user-facing stack forms a verified chain of atomic delegation: the CLI is structurally pure delegation verified through hermetic integration tests, and every mutation flowing through the API layer is atomic, audited, and produces observable before/after diffs — ensuring end-to-end traceability from user command to persisted state change.

**Depends on:** api-functions-are-transactional, [cli-is-verified-pure-delegation](other.md#cli-is-verified-pure-delegation), [mutations-are-observable-audited-and-index-consistent](other.md#mutations-are-observable-audited-and-index-consistent)
**Supports:** [user-interface-is-verified-and-fault-tolerant](other.md#user-interface-is-verified-and-fault-tolerant)

### fully-characterized-loop-sustains-indefinitely
**Status:** OUT

The fully characterized self-maintaining loop — origin-agnostic, fully observable, and minimality-sustained — can operate without temporal bound because its self-correction is resource-sustainable within a deterministic, structurally sound lifecycle; characterization completeness combined with resource sustainability yields indefinite operability.

**Depends on:** [self-correction-sustains-lifecycle-indefinitely](self.md#self-correction-sustains-lifecycle-indefinitely), [system-is-fully-characterized-self-maintaining-loop](system.md#system-is-fully-characterized-self-maintaining-loop)
**Supports:** [verified-correctness-is-indefinitely-observable](other.md#verified-correctness-is-indefinitely-observable)

### growth-preserves-universal-assurance
**Status:** OUT

The system grows its knowledge base exhaustively — through deterministic reasoning and LLM-driven derivation with guaranteed termination — while simultaneously maintaining universal multidimensional operational assurance spanning temporal self-correction, end-to-end reliability, and information flow control; growth never compromises any assurance dimension.

**Depends on:** [knowledge-growth-is-exhaustive-and-information-governed](other.md#knowledge-growth-is-exhaustive-and-information-governed), [system-assurance-is-universal-and-multidimensional](system.md#system-assurance-is-universal-and-multidimensional)
**Supports:** [growth-converges-with-topology-and-assurance](topology.md#growth-converges-with-topology-and-assurance)

### hash-file-full-sha256
**Status:** IN

`hash_file` returns a full 64-character hex SHA-256 digest (per the fix in PR #40 that removed the earlier `[:16]` truncation).

**Supports:** [source-tracking-is-collision-resistant-and-safe](source.md#source-tracking-is-collision-resistant-and-safe)

### hash-sources-idempotent-without-force
**Status:** IN

`hash_sources` with default `force=False` skips nodes that already have a `source_hash`, making repeated backfill calls safe; `force=True` rehashes unconditionally.


### hash-sources-is-additive-by-default
**Status:** IN

`hash_sources` without `force=True` never overwrites an existing non-empty `source_hash`; it only backfills nodes with empty or missing hashes.

**Supports:** [source-tracking-is-collision-resistant-and-safe](source.md#source-tracking-is-collision-resistant-and-safe)

### hash-sources-mutates-network
**Status:** IN

`hash_sources` writes directly to `node.source_hash` on the in-memory network, while `check_stale` is purely read-only — an intentional asymmetry.


### hash-sources-no-overwrite-default
**Status:** IN

`hash_sources` with `force=False` (the default) only backfills missing hashes and will never modify a node that already has a `source_hash` value.


### hash-truncation-is-16-hex
**Status:** OUT

Source hashes are SHA-256 truncated to the first 16 hex characters (64 bits), reducing collision resistance to ~32 bits for birthday attacks compared to the full 256-bit hash.

**Supports:** [architecture-sustains-gapless-lifecycle](lifecycle.md#architecture-sustains-gapless-lifecycle), [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [external-beliefs-are-safe-and-current](external.md#external-beliefs-are-safe-and-current), [lifecycle-management-is-gapless](lifecycle.md#lifecycle-management-is-gapless), [self-correction-is-resource-sustainable](self.md#self-correction-is-resource-sustainable), [staleness-gate-catches-all-drift](other.md#staleness-gate-catches-all-drift)

### hints-exclude-directly-retracted-node
**Status:** IN

The node passed to `retract_node` never appears in the `restoration_hints` list — only cascade victims with surviving premises do.

**Supports:** [restoration-hints-are-surgical](other.md#restoration-hints-are-surgical)

### idempotent-reimport-skips-all
**Status:** IN

Covered by existing `import-skips-existing-sync-is-remote-wins` which captures the idempotent import behavior


### identity-transformation-is-semantically-invisible
**Status:** OUT

Challenge creates an irreversible structural transformation (premise → justified node), yet the resulting dialectical structure receives identical evaluation to any other belief — the permanent identity change has no lasting semantic consequence because evaluation is uniformly origin-agnostic and context-independent.

**Depends on:** [dialectics-are-semantically-transparent](other.md#dialectics-are-semantically-transparent), [premise-identity-transformation-is-architecturally-asymmetric](other.md#premise-identity-transformation-is-architecturally-asymmetric)
**Supports:** [truth-evaluation-is-transformation-invariant](other.md#truth-evaluation-is-transformation-invariant)

### information-boundaries-are-controlled-at-all-levels
**Status:** OUT

The system controls information flow at every boundary: internally through access-tag authorization gating and token-budget constraints on output, and externally through bidirectional bounds on LLM input/output quality and defensive belief ingestion pipelines

**Depends on:** [external-surface-is-fully-controlled](external.md#external-surface-is-fully-controlled), [information-flow-is-authorization-and-budget-controlled](other.md#information-flow-is-authorization-and-budget-controlled)
**Supports:** [exhaustive-knowledge-expansion-within-controlled-boundaries](other.md#exhaustive-knowledge-expansion-within-controlled-boundaries), [external-integration-is-hardened-and-boundary-controlled](external.md#external-integration-is-hardened-and-boundary-controlled), [trust-and-information-boundaries-are-comprehensively-enforced](other.md#trust-and-information-boundaries-are-comprehensively-enforced)

### information-flow-is-authorization-and-budget-controlled
**Status:** OUT

Information flow from the belief network is controlled along two independent dimensions: access tags gate which beliefs are visible to each caller (authorization control via transitive subset checks), while token budgets constrain how much of the visible network is emitted (volume control via priority-ordered truncation).

**Depends on:** [access-control-is-transitive-subset-gated](other.md#access-control-is-transitive-subset-gated), [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size)
**Supports:** [information-boundaries-are-controlled-at-all-levels](other.md#information-boundaries-are-controlled-at-all-levels), [information-output-is-authorized-budgeted-and-deterministic](deterministic.md#information-output-is-authorized-budgeted-and-deterministic)

### information-flow-is-controlled-in-both-directions
**Status:** OUT

Information flow is controlled at every system boundary: inbound data passes through production-hardened LLM integration (bounded execution, fail-soft handling, process isolation) and boundary-controlled information isolation (access tags, namespace partitioning), while outbound data is deterministic, authorized via transitive subset-gated access control, and budget-constrained — no uncontrolled data enters or leaves the belief network

**Depends on:** [all-output-is-deterministic-authorized-and-resilient](deterministic.md#all-output-is-deterministic-authorized-and-resilient), [external-integration-is-hardened-and-boundary-controlled](external.md#external-integration-is-hardened-and-boundary-controlled)
**Supports:** [knowledge-growth-is-exhaustive-and-information-governed](other.md#knowledge-growth-is-exhaustive-and-information-governed), [verified-interface-controls-bidirectional-flow](other.md#verified-interface-controls-bidirectional-flow)

### information-pipeline-is-resource-governed-and-access-controlled
**Status:** OUT

The complete information pipeline is governed along two orthogonal axes: token budgets accurately constrain both input (proportional derive allocation) and output (compact distillation with budget enforcement), while access tags enforce transitive subset-based authorization at every read boundary — every piece of information is simultaneously resource-bounded and access-controlled.

**Depends on:** [access-control-is-transitive-subset-gated](other.md#access-control-is-transitive-subset-gated), [token-budgets-are-accurate-bidirectionally](other.md#token-budgets-are-accurate-bidirectionally)
**Supports:** [information-governance-is-end-to-end-authorized-and-resource-constrained](governance.md#information-governance-is-end-to-end-authorized-and-resource-constrained), [system-boundary-enforcement-spans-validation-resilience-and-resources](system.md#system-boundary-enforcement-spans-validation-resilience-and-resources)

### init-db-refuses-existing-without-force
**Status:** IN

`api.init_db()` raises `FileExistsError` when the database file already exists, unless `force=True` is passed to allow overwrite.


### init-is-pure-data-model
**Status:** IN

`reasons_lib/__init__.py` contains only dataclass definitions (`Node`, `Justification`, `Nogood`) with no behavior, validation, or I/O; it imports nothing from the project and sits at the bottom of the import graph.

**Supports:** [three-layer-stack-has-clean-boundaries](other.md#three-layer-stack-has-clean-boundaries)

### initialization-and-reconciliation-converge-equivalently
**Status:** IN

Both initialization paths (stored-state bootstrap trusting persisted values, and deterministic reasoning computing from scratch) and reconciliation operations (dual import/sync modes with heterogeneous truth state handling) converge to equivalent correct belief states — the system reaches the same outcome regardless of how or when beliefs enter the network.

**Depends on:** [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation), [initialization-is-path-independent](other.md#initialization-is-path-independent)

### initialization-is-path-independent
**Status:** IN

Whether beliefs enter through stored-state bootstrap (load trusts stored truth values, import builds nodes before recompute_all) or through the deterministic reasoning engine (uniform pure evaluation with guaranteed termination), the system reaches correct truth states — bootstrap trusts values that were originally computed by the same deterministic engine, and import re-derives them via recompute_all, establishing path independence of initialization.

**Depends on:** [bootstrap-bypasses-incremental-propagation](other.md#bootstrap-bypasses-incremental-propagation), [reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible)
**Supports:** [initialization-and-reconciliation-converge-equivalently](other.md#initialization-and-reconciliation-converge-equivalently), [initialization-is-safe-and-path-independent-across-backends](safe.md#initialization-is-safe-and-path-independent-across-backends)

### input-validation-is-comprehensive-at-all-boundaries
**Status:** OUT

Input validation is enforced at every system boundary through complementary mechanisms: typed exceptions (ValueError for duplicates, PermissionError for access violations) enforce API-level preconditions at the call boundary, while defense-in-depth reference validation (import normalization dropping unknown refs, nogood filtering skipping invalid nodes, hallucinated ID rejection) catches invalid node references at every data-acceptance boundary.

**Depends on:** [api-enforces-typed-preconditions](other.md#api-enforces-typed-preconditions), [reference-validation-is-defense-in-depth](other.md#reference-validation-is-defense-in-depth)
**Supports:** [pg-data-integrity-achieves-defense-in-depth](other.md#pg-data-integrity-achieves-defense-in-depth), [system-boundaries-are-validating-and-evolution-tolerant](system.md#system-boundaries-are-validating-and-evolution-tolerant)

### inspection-outputs-are-uniformly-normalized
**Status:** IN

Both inspection mechanisms — belief review and staleness checking — produce normalized, schema-consistent, fail-safe output with deterministic structure suitable for automated consumption.

**Depends on:** [check-stale-output-is-deterministic-and-structured](deterministic.md#check-stale-output-is-deterministic-and-structured), [review-output-is-uniform-and-fail-safe](safe.md#review-output-is-uniform-and-fail-safe)
**Supports:** [all-outputs-are-normalized-deterministic-and-resilient](deterministic.md#all-outputs-are-normalized-deterministic-and-resilient)

### integrity-and-scalability-are-complementary
**Status:** OUT

The system achieves comprehensive integrity (unified across all internal mutations and external belief ingestion) and sound multi-agent scalability (isolated namespaces, minimal primitives, deterministic propagation) simultaneously — these properties reinforce rather than trade off against each other because both derive from the same uniform evaluation rules.

**Depends on:** [internal-and-external-integrity-are-unified](external.md#internal-and-external-integrity-are-unified), [system-is-minimal-sound-and-scalable](system.md#system-is-minimal-sound-and-scalable)
**Supports:** [system-properties-emerge-from-unified-design](system.md#system-properties-emerge-from-unified-design)

### integrity-is-an-emergent-consequence-of-minimality
**Status:** OUT

End-to-end integrity across all mutation paths and architectural boundaries is not an independently-achieved property requiring separate enforcement — it falls out of minimality as another emergent consequence, because uniform primitive evaluation leaves no gaps for inconsistency to enter.

**Depends on:** [minimality-is-the-universal-generative-principle](other.md#minimality-is-the-universal-generative-principle), [unified-system-maintains-end-to-end-integrity](system.md#unified-system-maintains-end-to-end-integrity)

### invariant-preservation-is-architecturally-grounded
**Status:** OUT

The complete reasoning-and-revision architecture preserves invariants through minimal foundations not in a vacuum but atop concrete architectural safety — three-layer containment and atomic mutations provide the structural substrate within which minimal invariant preservation operates.

**Depends on:** [architecture-enforces-structural-and-operational-safety](other.md#architecture-enforces-structural-and-operational-safety), [complete-architecture-preserves-invariants-minimally](complete.md#complete-architecture-preserves-invariants-minimally)
**Supports:** [closed-loop-preserves-all-invariants](other.md#closed-loop-preserves-all-invariants), [complete-architecture-achieves-verified-production-correctness](complete.md#complete-architecture-achieves-verified-production-correctness), [invariants-are-origin-time-and-structurally-grounded](other.md#invariants-are-origin-time-and-structurally-grounded), [invariants-are-structurally-and-dynamically-preserved](other.md#invariants-are-structurally-and-dynamically-preserved)

### invariant-preservation-is-comprehensive
**Status:** OUT

System invariants are comprehensively preserved through two complementary mechanisms: the closed revision/lifecycle loop ensures temporal coverage across forward computation and backward revision, while dual structural/dynamic enforcement provides orthogonal protection through architectural grounding and minimality-enforced self-correction.

**Depends on:** [closed-loop-preserves-all-invariants](other.md#closed-loop-preserves-all-invariants), [invariants-are-structurally-and-dynamically-preserved](other.md#invariants-are-structurally-and-dynamically-preserved)
**Supports:** [invariant-preservation-is-self-sustaining](self.md#invariant-preservation-is-self-sustaining), [invariant-preservation-is-total](other.md#invariant-preservation-is-total), [system-is-self-sustaining-and-invariant-preserving](system.md#system-is-self-sustaining-and-invariant-preserving)

### invariant-preservation-is-total
**Status:** OUT

Invariant preservation is both comprehensive in scope (spanning revision loops, lifecycle management, and structural/dynamic enforcement) and grounded across all independent dimensions (origin, time, and structure), achieving total invariant coverage with no gaps in either what is preserved or where preservation holds.

**Depends on:** [invariant-preservation-is-comprehensive](other.md#invariant-preservation-is-comprehensive), [invariants-are-origin-time-and-structurally-grounded](other.md#invariants-are-origin-time-and-structurally-grounded)
**Supports:** [total-invariant-preservation-encompasses-all-beliefs](beliefs.md#total-invariant-preservation-encompasses-all-beliefs)

### invariants-are-origin-time-and-structurally-grounded
**Status:** OUT

System invariants are anchored along three dimensions: they hold across all belief origins and through time (comprehensive scope), and they are grounded in the concrete architecture's clean layer boundaries and atomic operations (structural foundation) — the invariants are both broad in what they cover and deep in how they are enforced.

**Depends on:** [invariant-preservation-is-architecturally-grounded](other.md#invariant-preservation-is-architecturally-grounded), [invariants-hold-across-origin-and-time](other.md#invariants-hold-across-origin-and-time)
**Supports:** [external-beliefs-are-fully-invariant-grounded](external.md#external-beliefs-are-fully-invariant-grounded), [invariant-preservation-is-total](other.md#invariant-preservation-is-total)

### invariants-are-structurally-and-dynamically-preserved
**Status:** OUT

System invariants are preserved through two complementary layers: architectural grounding provides structural enforcement via clean layer boundaries and atomic mutations, while minimality-enforced self-correction actively detects and resolves violations through contradiction resolution and staleness detection.

**Depends on:** [invariant-preservation-is-architecturally-grounded](other.md#invariant-preservation-is-architecturally-grounded), [self-correction-is-minimality-enforced](self.md#self-correction-is-minimality-enforced)
**Supports:** [invariant-preservation-is-comprehensive](other.md#invariant-preservation-is-comprehensive)

### invariants-hold-across-origin-and-time
**Status:** OUT

System invariants hold along every independent dimension: across all belief origins (human-initiated, LLM-derived, agent-imported) via shared minimal foundations, and across both temporal phases (creation-time edge-case handling and maintenance-time staleness detection) including all semantic edge cases.

**Depends on:** [edge-case-safety-spans-creation-and-maintenance](spans.md#edge-case-safety-spans-creation-and-maintenance), [revision-invariants-span-all-origins](revision.md#revision-invariants-span-all-origins)
**Supports:** [invariants-are-origin-time-and-structurally-grounded](other.md#invariants-are-origin-time-and-structurally-grounded)

### invoke-claude-raises-when-binary-missing
**Status:** IN

`_invoke_claude()` raises `FileNotFoundError` if the `claude` CLI is not on `PATH`, rather than returning an error string — this is the one exception that `ask()` does not catch internally.


### invoke-model-strips-claudecode-env
**Status:** IN

invoke_model() in llm.py strips the CLAUDECODE environment variable before all subprocess.run() calls, preventing recursive Claude Code entry from any module that uses the shared LLM interface (ask, derive, review).

**Supports:** [llm-subprocess-isolation-prevents-recursion](llm.md#llm-subprocess-isolation-prevents-recursion)

### issue-121-evolution-tolerance-audit
**Status:** IN

Issue #121: Audit evolution tolerance at all system boundaries — not all boundaries have documented forward-compatibility mechanisms

**Supports:** [rich-governance-has-verified-evolution-tolerance](governance.md#rich-governance-has-verified-evolution-tolerance), [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)

### issue-122-review-fault-tolerance-audit
**Status:** IN

Issue #122: Audit review module for unhandled failure modes — three specific handlers do not establish coverage of all failure modes

**Supports:** [review-achieves-verified-fault-tolerance](other.md#review-achieves-verified-fault-tolerance), [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant)

### issue-123-resource-footprint-audit
**Status:** IN

Issue #123: Audit resource footprint across all lifecycle phases — only deployment and startup phases are currently evidenced

**Supports:** [system-resource-footprint-is-minimal-at-all-phases](system.md#system-resource-footprint-is-minimal-at-all-phases)

### issue-126-reference-validation-audit
**Status:** IN

Issue #126: Audit all node ID reference boundaries for validation — three specific boundaries do not establish coverage of every boundary

**Supports:** [governance-and-dialectics-have-verified-references](governance.md#governance-and-dialectics-have-verified-references), [governance-topology-is-reference-verified](governance.md#governance-topology-is-reference-verified), [reference-validation-is-defense-in-depth](other.md#reference-validation-is-defense-in-depth), [verified-revision-completeness-at-all-reference-boundaries](revision.md#verified-revision-completeness-at-all-reference-boundaries)

### jaccard-tokenizer-splits-on-hyphens-and-colons
**Status:** IN

`_tokenize_id` splits belief IDs on hyphens and colons into token sets for Jaccard similarity comparison, so `foo-bar-baz` and `foo-bar-qux` share 2/4 tokens (0.5 similarity)


### kill-switch-cascade-is-reversible
**Status:** IN

Retracting `agent:active` cascades all agent beliefs to OUT via the inactive relay flipping IN; re-asserting `agent:active` reverses the cascade, restoring all beliefs to IN via the same BFS propagation

**Supports:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible)

### kill-switch-uses-outlist-not-antecedent
**Status:** IN

The `agent:inactive` node is placed in each imported belief's outlist (not antecedents) so that retracting `agent:active` cascades all imported beliefs to OUT, while per-belief retraction still works independently

**Supports:** [outlist-is-universal-defeat-mechanism](other.md#outlist-is-universal-defeat-mechanism)

### knowledge-equilibria-are-fully-characterized
**Status:** OUT

Knowledge revision converges to equilibria that are simultaneously self-sustaining through minimality's fixed-point, invariant-preserving across all belief types, correction-convergent with complete dispute resolution fidelity, and topology-accurate with verified dependency propagation — the complete set of equilibrium properties.

**Depends on:** [knowledge-equilibria-are-correction-convergent-and-topology-accurate](correction.md#knowledge-equilibria-are-correction-convergent-and-topology-accurate), [knowledge-revision-converges-to-self-sustaining-equilibria](revision.md#knowledge-revision-converges-to-self-sustaining-equilibria)

### knowledge-expansion-is-exhaustive-within-hardened-boundaries
**Status:** OUT

Exhaustive knowledge expansion — deterministic reversible reasoning combined with complete LLM-driven derivation with guaranteed termination — is achieved through production-hardened LLM integration operating within controlled information boundaries, ensuring the system discovers all derivable conclusions while maintaining robustness guarantees at every stage of the pipeline.

**Depends on:** [exhaustive-knowledge-expansion-within-controlled-boundaries](other.md#exhaustive-knowledge-expansion-within-controlled-boundaries), [llm-integration-is-production-hardened](llm.md#llm-integration-is-production-hardened)
**Supports:** [knowledge-growth-is-exhaustive-and-information-governed](other.md#knowledge-growth-is-exhaustive-and-information-governed)

### knowledge-growth-is-exhaustive-and-information-governed
**Status:** OUT

Exhaustive knowledge expansion — deterministic reversible reasoning combined with complete LLM-driven derivation with guaranteed termination within hardened integration boundaries — operates under comprehensive bidirectional information governance: inbound data passes through production-hardened LLM integration with process isolation and fail-soft semantics, while outbound information is constrained by access-tag authorization and token-budget limits.

**Depends on:** [information-flow-is-controlled-in-both-directions](other.md#information-flow-is-controlled-in-both-directions), [knowledge-expansion-is-exhaustive-within-hardened-boundaries](other.md#knowledge-expansion-is-exhaustive-within-hardened-boundaries)
**Supports:** [growth-preserves-universal-assurance](other.md#growth-preserves-universal-assurance)

### knowledge-growth-reaches-transparent-equilibria
**Status:** OUT

The system's knowledge growth converges to equilibria that are simultaneously negation-transparent (the final stable state is uniquely determined by evaluation order-invariant rules over negative semantics) and propagation-complete (every truth change cascades to every transitively dependent node), with indefinite self-correction ensuring these equilibrium properties are maintained across unbounded operational time

**Depends on:** [equilibria-are-negation-transparent-with-complete-fidelity](complete.md#equilibria-are-negation-transparent-with-complete-fidelity), [knowledge-growth-is-convergent-assured-and-indefinitely-self-correcting](self.md#knowledge-growth-is-convergent-assured-and-indefinitely-self-correcting)
**Supports:** [equilibria-are-transparent-and-trajectory-documented](other.md#equilibria-are-transparent-and-trajectory-documented), [knowledge-equilibria-are-correction-convergent-and-topology-accurate](correction.md#knowledge-equilibria-are-correction-convergent-and-topology-accurate), [knowledge-equilibria-are-invariant-preserving-and-self-sustaining](self.md#knowledge-equilibria-are-invariant-preserving-and-self-sustaining)

### list-negative-batches-at-40
**Status:** IN

`api.list_negative` splits candidates into batches of 40, so 120 keyword-matching nodes produce exactly 3 LLM calls.


### list-negative-batches-at-50
**Status:** IN

`list_negative` splits candidate nodes into batches of approximately 50 for LLM classification, verified by the test suite asserting exactly 3 LLM calls for 120 candidates.

**Supports:** [list-negative-is-bounded-and-batch-scalable](other.md#list-negative-is-bounded-and-batch-scalable)

### list-negative-is-bounded-and-batch-scalable
**Status:** OUT

The list-negative classification pipeline is both defensively bounded (two-stage keyword + LLM filtering with hallucination rejection and graceful malformed-output handling) and scalably partitioned (fixed batch size of ~50 candidates per LLM call), ensuring predictable resource usage and bounded LLM costs regardless of belief network size.

**Depends on:** [list-negative-batches-at-50](other.md#list-negative-batches-at-50), [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded)
**Supports:** [llm-belief-operations-span-creation-and-classification](llm.md#llm-belief-operations-span-creation-and-classification)

### list-negative-is-defensively-bounded
**Status:** IN

The negative belief listing pipeline applies defense-in-depth: keyword pre-filtering narrows candidates before LLM classification, hallucinated node IDs are discarded against the actual network, and malformed LLM output falls back gracefully to zero count rather than raising.

**Depends on:** [api-list-negative-filters-hallucinated-ids](other.md#api-list-negative-filters-hallucinated-ids), [api-list-negative-graceful-on-malformed-llm](llm.md#api-list-negative-graceful-on-malformed-llm), [list-negative-uses-two-stage-classification](other.md#list-negative-uses-two-stage-classification)
**Supports:** [all-belief-inspection-is-non-mutating-and-fault-tolerant](other.md#all-belief-inspection-is-non-mutating-and-fault-tolerant), [all-llm-operations-are-defensively-bounded](llm.md#all-llm-operations-are-defensively-bounded), [list-negative-is-bounded-and-batch-scalable](other.md#list-negative-is-bounded-and-batch-scalable), [llm-belief-pipeline-is-fully-quality-enforced](llm.md#llm-belief-pipeline-is-fully-quality-enforced), [llm-integration-fails-softly-across-modules](llm.md#llm-integration-fails-softly-across-modules)

### list-negative-json-parser-tolerates-prose-preamble
**Status:** IN

The `list_negative` LLM classification response parser uses `re.finditer` to extract JSON objects from responses that include prose preamble, handling the common LLM pattern of prefacing structured output with natural language rather than requiring clean JSON.

**Supports:** [list-negative-parser-is-fully-resilient](other.md#list-negative-parser-is-fully-resilient)

### list-negative-parser-is-fully-resilient
**Status:** IN

The list-negative LLM response parser handles all degradation levels: regex extraction recovers JSON objects from prose-laden responses, and completely unparseable output returns zero results gracefully rather than raising exceptions.

**Depends on:** [api-list-negative-graceful-on-malformed-llm](llm.md#api-list-negative-graceful-on-malformed-llm), [list-negative-json-parser-tolerates-prose-preamble](other.md#list-negative-json-parser-tolerates-prose-preamble)
**Supports:** [format-resilience-spans-all-external-interfaces](spans.md#format-resilience-spans-all-external-interfaces)

### list-negative-uses-two-stage-classification
**Status:** IN

`list_negative` uses keyword pre-filtering against a hardcoded `NEGATIVE_TERMS` list (~50 words), then LLM classification via `ask._invoke_claude` to eliminate false positives.

**Supports:** [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded)

### maintenance-loop-is-fully-observable
**Status:** OUT

The minimality-sustained closed maintenance loop has complete observability: every self-correction leaves traceable history (nogoods, retraction records, staleness reports), enabling full audit of how the system maintains itself over time.

**Depends on:** [minimality-sustains-closed-loop-maintenance](other.md#minimality-sustains-closed-loop-maintenance), [self-correction-has-complete-traceable-history](self.md#self-correction-has-complete-traceable-history)
**Supports:** [origin-agnostic-trustworthiness-is-fully-verifiable](other.md#origin-agnostic-trustworthiness-is-fully-verifiable), [system-is-fully-characterized-self-maintaining-loop](system.md#system-is-fully-characterized-self-maintaining-loop), [trustworthiness-is-verifiable-through-observability](other.md#trustworthiness-is-verifiable-through-observability), [verified-correctness-is-independently-observable](other.md#verified-correctness-is-independently-observable)

### make-nodes-excludes-active-premises
**Status:** IN

Test helper implementation detail, not a claim about production code behavior


### make-nodes-omits-active-premises
**Status:** IN

Test helper implementation detail, not a production code invariant


### mcp-bridge-call-tool-timeout-60s
**Status:** IN

`call_tool()` blocks for up to 60 seconds per invocation via `future.result(timeout=60)`; if the MCP server hangs, the calling thread unblocks with `TimeoutError`.

**Supports:** [mcp-bridge-is-timeout-bounded-at-all-phases](other.md#mcp-bridge-is-timeout-bounded-at-all-phases)

### mcp-bridge-connect-blocks-up-to-30s
**Status:** IN

`connect()` blocks the calling thread for up to 30 seconds waiting for MCP session initialization via `_ready.wait(timeout=30)`, then raises `TimeoutError` if the server doesn't respond.

**Supports:** [mcp-bridge-is-timeout-bounded-at-all-phases](other.md#mcp-bridge-is-timeout-bounded-at-all-phases)

### mcp-bridge-is-timeout-bounded-at-all-phases
**Status:** IN

MCP bridge operations are timeout-bounded at both connection establishment (30s) and per-tool execution (60s), preventing indefinite blocking on unresponsive MCP servers at any lifecycle phase.

**Depends on:** [mcp-bridge-call-tool-timeout-60s](other.md#mcp-bridge-call-tool-timeout-60s), [mcp-bridge-connect-blocks-up-to-30s](other.md#mcp-bridge-connect-blocks-up-to-30s)
**Supports:** [ask-mcp-achieves-accurate-bounded-tool-use](other.md#ask-mcp-achieves-accurate-bounded-tool-use), [ask-mcp-is-defense-in-depth-bounded](other.md#ask-mcp-is-defense-in-depth-bounded)

### mcp-bridge-runs-dedicated-event-loop-thread
**Status:** IN

Each `McpBridge` instance runs its own asyncio event loop on a daemon thread; the MCP session stays alive until `close()` signals the shutdown event, bridging the sync/async boundary via `run_coroutine_threadsafe`.


### mcp-bridge-tools-snapshot-at-connect
**Status:** IN

The tool catalog and server instructions are populated once during `connect()` and never refreshed — there is no mechanism to pick up tools added after the initial handshake.

**Supports:** [ask-mcp-achieves-accurate-bounded-tool-use](other.md#ask-mcp-achieves-accurate-bounded-tool-use), [ask-mcp-tool-use-has-current-catalog](other.md#ask-mcp-tool-use-has-current-catalog)

### mcp-is-optional-dependency
**Status:** IN

The `mcp` package is guarded by try/except at import time; `_require_mcp()` defers the `ImportError` to `McpBridge` construction so the module can be imported unconditionally without the SDK installed.


### metadata-actively-governs-truth-propagation
**Status:** IN

Lifecycle state carried in node metadata (retraction flags, stale reasons) is not passive storage but actively governs truth propagation behavior — retracted nodes are skipped during BFS traversal and trigger nodes are never recomputed — ensuring that the universal extension mechanism directly controls truth maintenance rather than merely recording state

**Depends on:** [metadata-is-universal-extension-mechanism](other.md#metadata-is-universal-extension-mechanism), [propagation-respects-node-lifecycle](lifecycle.md#propagation-respects-node-lifecycle)
**Supports:** [metadata-governance-flows-through-safe-topology](governance.md#metadata-governance-flows-through-safe-topology), [metadata-governs-lifecycle-across-read-and-write-paths](lifecycle.md#metadata-governs-lifecycle-across-read-and-write-paths)

### metadata-is-universal-extension-mechanism
**Status:** IN

Node metadata is the universal extension mechanism carrying all structured lifecycle state (retraction flags, stale reasons, challenges, access tags, supersession markers), and retraction flags pinned in metadata survive recomputation to enforce sticky retraction.

**Depends on:** [network-metadata-carries-structured-state](other.md#network-metadata-carries-structured-state), [retracted-pin-survives-recompute](other.md#retracted-pin-survives-recompute)
**Supports:** [metadata-actively-governs-truth-propagation](other.md#metadata-actively-governs-truth-propagation), [network-state-is-extensible-and-consistently-tracked](other.md#network-state-is-extensible-and-consistently-tracked)

### minimality-is-both-generative-and-unifying
**Status:** OUT

Minimality is simultaneously the generative source of each individual system property (extensibility, robustness, revision completeness) and the unifying principle that makes them cohere — the system achieves unity not by coordinating independently-designed features but because every feature is a different manifestation of the same minimal primitive set.

**Depends on:** [minimality-is-the-universal-generative-principle](other.md#minimality-is-the-universal-generative-principle), [system-properties-emerge-from-unified-design](system.md#system-properties-emerge-from-unified-design)

### minimality-is-the-universal-generative-principle
**Status:** OUT

Minimality is the single generative architectural principle underlying all emergent system properties — extensibility and robustness arise from transparent extension composition on the minimal core, while revision completeness arises from uniform edge-case handling within the same core — revealing that these typically independent qualities share a common origin rather than requiring separate design effort.

**Depends on:** [minimality-yields-extensibility-and-robustness](other.md#minimality-yields-extensibility-and-robustness), [revision-completeness-follows-from-minimality](revision.md#revision-completeness-follows-from-minimality)
**Supports:** [integrity-is-an-emergent-consequence-of-minimality](other.md#integrity-is-an-emergent-consequence-of-minimality), [minimality-is-both-generative-and-unifying](other.md#minimality-is-both-generative-and-unifying)

### minimality-produces-uniformity-and-determinism
**Status:** OUT

Minimality is the shared generative root of two independently-established system properties: edge-case uniformity (all cases handled by the same rules without special-casing) and operational determinism (predictable terminating evaluation with conservative failure), demonstrating that a single design principle produces both semantic and operational guarantees simultaneously.

**Depends on:** [edge-case-uniformity-follows-from-minimality](other.md#edge-case-uniformity-follows-from-minimality), [semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism)
**Supports:** [minimality-spans-computation-and-revision](spans.md#minimality-spans-computation-and-revision)

### minimality-sustains-closed-loop-maintenance
**Status:** OUT

Minimality generates both forward computation properties (uniformity, determinism) and backward revision properties (universal safety), while lifecycle management ensures every generated belief remains under active maintenance with no escape path — together forming a self-sustaining architecture where the generative principle and the maintenance loop are co-dependent.

**Depends on:** [minimality-spans-computation-and-revision](spans.md#minimality-spans-computation-and-revision), [revision-and-lifecycle-form-closed-loop](revision.md#revision-and-lifecycle-form-closed-loop)
**Supports:** [closed-loop-is-origin-agnostic](other.md#closed-loop-is-origin-agnostic), [maintenance-loop-is-fully-observable](other.md#maintenance-loop-is-fully-observable), [minimality-is-self-sustaining](self.md#minimality-is-self-sustaining)

### minimality-yields-extensibility-and-robustness
**Status:** OUT

The minimal core simultaneously enables two independent emergent properties — transparent extension composition (dialectics, multi-agent federation) and uniform edge-case handling (vacuous premises, asymmetric absence) — demonstrating that minimality is operationally productive, not merely aesthetically elegant.

**Depends on:** [edge-case-uniformity-follows-from-minimality](other.md#edge-case-uniformity-follows-from-minimality), [extensions-compose-transparently-on-core](other.md#extensions-compose-transparently-on-core)
**Supports:** [minimality-is-the-universal-generative-principle](other.md#minimality-is-the-universal-generative-principle), [system-properties-emerge-from-unified-design](system.md#system-properties-emerge-from-unified-design)

### missing-nodes-have-asymmetric-fail-semantics
**Status:** IN

Missing nodes are treated asymmetrically: absent antecedents fail validation (conservative), absent outlist nodes pass (permissive), creating a "believe unless proven otherwise" default

**Depends on:** [missing-outlist-nodes-pass-validation](other.md#missing-outlist-nodes-pass-validation), [outlist-absent-means-out](other.md#outlist-absent-means-out), [sl-outlist-asymmetry](other.md#sl-outlist-asymmetry)
**Supports:** [absence-has-consistent-dual-semantics](other.md#absence-has-consistent-dual-semantics), [tms-core-is-deterministic-and-conservative](deterministic.md#tms-core-is-deterministic-and-conservative)

### missing-outlist-nodes-pass-validation
**Status:** IN

In `_justification_valid`, missing antecedent nodes cause the check to fail (node goes OUT), but missing outlist nodes pass (don't block) — an open-world default.

**Supports:** [missing-nodes-have-asymmetric-fail-semantics](other.md#missing-nodes-have-asymmetric-fail-semantics)

### multiple-outlist-is-conjunction
**Status:** IN

When a justification has multiple outlist entries, ALL must be OUT for the justification to be valid; any single outlist node going IN defeats the entire justification

**Supports:** [outlist-semantics-are-fully-specified](other.md#outlist-semantics-are-fully-specified)

### mutation-pipeline-is-atomic-snapshot
**Status:** OUT

Every network mutation follows an atomic snapshot pipeline: API context management ensures load/save atomicity with write-flag gating, while storage performs full-replace persistence — no partial state is ever visible between operations.

**Depends on:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental)
**Supports:** [mutation-pipeline-produces-consistent-state](other.md#mutation-pipeline-produces-consistent-state), [mutations-are-atomic-and-safely-propagated](other.md#mutations-are-atomic-and-safely-propagated)

### mutation-pipeline-produces-consistent-state
**Status:** OUT

Every mutation produces a fully consistent persisted network: atomic load/save ensures no partial writes, deterministic propagation ensures all truth values are correctly derived, and lifecycle-aware traversal prevents stale recomputations.

**Depends on:** [mutation-pipeline-is-atomic-snapshot](other.md#mutation-pipeline-is-atomic-snapshot), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating)

### mutations-achieve-full-traceability
**Status:** IN

Every mutation is fully traceable from initiation to persisted outcome: atomicity ensures operations complete or roll back entirely, the audit log and structured before/after diffs provide historical context, and consistent artifact identification (deterministic challenge auto-IDs, monotonic nogood IDs) enables referencing specific mutation outcomes indefinitely.

**Depends on:** [mutations-are-atomic-audited-and-index-consistent](other.md#mutations-are-atomic-audited-and-index-consistent), [system-artifacts-maintain-consistent-identification](system.md#system-artifacts-maintain-consistent-identification)
**Supports:** [mutations-are-traceable-through-transitive-cascades](other.md#mutations-are-traceable-through-transitive-cascades)

### mutations-are-atomic-and-safely-propagated
**Status:** OUT

Every network mutation follows an end-to-end safety pipeline: API context management ensures atomic load/save with write-flag gating, truth propagation terminates deterministically with lifecycle-aware BFS traversal, and snapshot persistence captures the final consistent state — no mutation can produce an inconsistent or divergent network.

**Depends on:** [mutation-pipeline-is-atomic-snapshot](other.md#mutation-pipeline-is-atomic-snapshot), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating)
**Supports:** [dialectical-transformation-is-operationally-safe](safe.md#dialectical-transformation-is-operationally-safe), [dialectics-are-atomic-and-transparent](other.md#dialectics-are-atomic-and-transparent), [full-system-integrity-is-gap-free](system.md#full-system-integrity-is-gap-free), [operational-integrity-is-end-to-end](other.md#operational-integrity-is-end-to-end), [system-operations-never-crash](system.md#system-operations-never-crash)

### mutations-are-atomic-audited-and-index-consistent
**Status:** IN

Every network mutation achieves three simultaneous guarantees: transactional atomicity (context-managed load/save with write-flag gating), historical auditability (timestamped audit log entries), and structural consistency (dependents index updated synchronously) — forming a complete mutation-safety contract.

**Depends on:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [network-mutations-are-audited-and-index-consistent](other.md#network-mutations-are-audited-and-index-consistent)
**Supports:** [all-network-modifications-are-auditable-and-topology-preserving](topology.md#all-network-modifications-are-auditable-and-topology-preserving), [mutations-achieve-full-traceability](other.md#mutations-achieve-full-traceability)

### mutations-are-observable-audited-and-index-consistent
**Status:** IN

Every network mutation achieves triple-layered traceability: callers receive structured before/after diffs at the API level, the internal audit log records timestamped events for historical analysis, and the dependents index is simultaneously maintained — providing both external and internal observability.

**Depends on:** [api-mutating-ops-use-before-after-diffing](other.md#api-mutating-ops-use-before-after-diffing), [network-mutations-are-audited-and-index-consistent](other.md#network-mutations-are-audited-and-index-consistent)
**Supports:** [full-user-stack-is-verified-atomic-delegation](other.md#full-user-stack-is-verified-atomic-delegation)

### mutations-are-traceable-through-transitive-cascades
**Status:** IN

Every network mutation and its resulting retraction cascades are simultaneously traceable (atomic operations with audit logging, structured before/after diffs, consistent artifact identification) and transitively complete (propagation reaching all dependent nodes with guaranteed termination), ensuring every cascaded effect is as auditable as the triggering mutation

**Depends on:** [mutations-achieve-full-traceability](other.md#mutations-achieve-full-traceability), [retraction-cascade-is-transitive-and-terminating](other.md#retraction-cascade-is-transitive-and-terminating)
**Supports:** [all-truth-effects-are-topology-complete-and-traceable](topology.md#all-truth-effects-are-topology-complete-and-traceable)

### namespace-active-premise-invariant
**Status:** IN

When `namespace` is set, `add_node` auto-creates a `{namespace}:active` premise node and wires it as an antecedent into every namespaced justification; retracting that single premise cascades OUT every belief from that namespace.

**Supports:** [namespace-is-colon-convention-with-auto-wiring](other.md#namespace-is-colon-convention-with-auto-wiring)

### namespace-is-colon-convention-with-auto-wiring
**Status:** IN

The namespace system is a colon-based convention with automatic infrastructure wiring: colon presence prevents double-prefixing, the `agent:id` format provides scoping, and node creation auto-wires a `{ns}:active` premise as an antecedent.

**Depends on:** [colon-means-already-namespaced](other.md#colon-means-already-namespaced), [namespace-active-premise-invariant](other.md#namespace-active-premise-invariant), [namespace-prefix-is-colon-separated](other.md#namespace-prefix-is-colon-separated)
**Supports:** [all-identification-is-deterministic-and-collision-free](deterministic.md#all-identification-is-deterministic-and-collision-free), [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation)

### namespace-prefix-is-colon-separated
**Status:** IN

Agent namespacing uses the format `agent_name:belief_id`; `_resolve_namespace()` skips prefixing any ID that already contains a colon, preventing double-prefixing of cross-namespace references

**Supports:** [namespace-is-colon-convention-with-auto-wiring](other.md#namespace-is-colon-convention-with-auto-wiring)

### negation-is-transparent-to-evaluation
**Status:** OUT

The system's complete negative semantics — structural absence creating premise behavior, explicit outlist defeat with automatic reversal, and guided recovery — operate within transformation-invariant truth evaluation: negation mechanisms alter belief topology but never create special-case evaluation paths, because the same uniform rules evaluate all resulting structures identically regardless of how they were produced.

**Depends on:** [negative-semantics-are-complete-reversible-and-recoverable](complete.md#negative-semantics-are-complete-reversible-and-recoverable), [truth-evaluation-is-transformation-invariant](other.md#truth-evaluation-is-transformation-invariant)
**Supports:** [all-transformations-are-evaluation-transparent](other.md#all-transformations-are-evaluation-transparent), [canonical-equilibria-are-negation-transparent](other.md#canonical-equilibria-are-negation-transparent)

### negative-semantics-are-uniform-through-minimality
**Status:** OUT

The system's complete negative semantics — structural absence creating premise behavior, explicit outlist defeat with automatic reversal, permanent identity effects from challenge — handle all edge cases uniformly because they derive from the same minimal primitives as all other truth evaluation, with no special-case handling for negation.

**Depends on:** [edge-case-uniformity-follows-from-minimality](other.md#edge-case-uniformity-follows-from-minimality), [negative-semantics-are-complete-reversible-and-recoverable](complete.md#negative-semantics-are-complete-reversible-and-recoverable)

### negative-semantics-have-reversible-defeat-but-permanent-identity-effects
**Status:** IN

The system's complete negative semantics — structural absence creating premise behavior and explicit outlist defeat — exhibit a fundamental asymmetry: all outlist-based defeat operations (challenge, kill-switch, supersession) are fully reversible in truth value, but dialectical challenge permanently destroys premise identity by injecting a justification into a formerly unjustified node, an irreversible structural transformation.

**Depends on:** [absence-and-outlist-form-complete-negative-semantics](complete.md#absence-and-outlist-form-complete-negative-semantics), [dialectical-defeat-is-reversible-but-identity-is-permanent](other.md#dialectical-defeat-is-reversible-but-identity-is-permanent)
**Supports:** [revision-has-complete-semantics-with-controlled-irreversibility](revision.md#revision-has-complete-semantics-with-controlled-irreversibility)

### network-add-node-rejects-duplicates
**Status:** IN

`Network.add_node()` raises `ValueError` if a node with the given ID already exists in the network.


### network-dependents-eagerly-maintained
**Status:** IN

Duplicates existing beliefs `dependents-bidirectional-index` and `dependents-is-manual-reverse-index`.


### network-is-central-dependency
**Status:** IN

`network.py` is imported by essentially every other module — api, storage, import, export, compact, check_stale, and all test files — making it the central data structure of the project.

**Supports:** [central-dependency-is-safely-contained](other.md#central-dependency-is-safely-contained)

### network-is-sole-truth-propagation-engine
**Status:** OUT

All truth value computation and propagation in the system flows through `Network._propagate()` and `Network._compute_truth()`; no other module modifies truth values directly.


### network-metadata-carries-structured-state
**Status:** IN

Node state such as `_retracted`, `retract_reason`, `superseded_by`, `challenges`, `access_tags`, and `summarized_by` lives in the generic `metadata` dict rather than typed Node fields, keeping the dataclass stable while features layer on behavior.

**Supports:** [metadata-is-universal-extension-mechanism](other.md#metadata-is-universal-extension-mechanism)

### network-missing-outlist-passes
**Status:** IN

Duplicates existing belief `missing-outlist-nodes-pass-validation`.


### network-mutations-append-to-audit-log
**Status:** IN

Every network mutation records a timestamped event in `self.log`, an append-only list that serves as the propagation audit trail.

**Supports:** [network-mutations-are-audited-and-index-consistent](other.md#network-mutations-are-audited-and-index-consistent)

### network-mutations-are-audited-and-index-consistent
**Status:** IN

Every network mutation simultaneously maintains dual invariants: the audit log receives a timestamped event providing historical traceability, and the dependents reverse index is kept consistent ensuring correct future propagation.

**Depends on:** [every-network-mutation-maintains-dependents](other.md#every-network-mutation-maintains-dependents), [network-mutations-append-to-audit-log](other.md#network-mutations-append-to-audit-log)
**Supports:** [mutations-are-atomic-audited-and-index-consistent](other.md#mutations-are-atomic-audited-and-index-consistent), [mutations-are-observable-audited-and-index-consistent](other.md#mutations-are-observable-audited-and-index-consistent), [network-state-is-extensible-and-consistently-tracked](other.md#network-state-is-extensible-and-consistently-tracked)

### network-retracted-nodes-persist
**Status:** IN

Retracted nodes remain in `self.nodes` with `_retracted` metadata set; they are never deleted from the graph, enabling later restoration via `assert_node()` without rederivation.


### network-retracted-skips-propagation
**Status:** IN

Duplicates existing belief `retracted-nodes-skipped-in-propagation`.


### network-state-is-extensible-and-consistently-tracked
**Status:** IN

Network state management is both extensible (metadata carries all lifecycle state — retraction flags, stale reasons, challenges, access tags, supersession — as a universal key-value mechanism) and consistently tracked (every mutation maintains the audit log and dependents index simultaneously), ensuring new state dimensions can be added without compromising existing consistency guarantees

**Depends on:** [metadata-is-universal-extension-mechanism](other.md#metadata-is-universal-extension-mechanism), [network-mutations-are-audited-and-index-consistent](other.md#network-mutations-are-audited-and-index-consistent)
**Supports:** [metadata-provides-extensible-lifecycle-governance](lifecycle.md#metadata-provides-extensible-lifecycle-governance)

### network-tests-are-database-free
**Status:** IN

Test infrastructure detail, not a production code claim


### node-truth-disjunctive
**Status:** IN

A node is IN if ANY of its justifications is valid (disjunctive semantics); adding a justification to a node can never cause it to go OUT.


### nogood-id-backwards-compat
**Status:** IN

Databases created before the monotonic counter fix (lacking `network_meta` table) load without error; the counter is derived from max existing nogood ID + 1, or defaults to 1 if none exist


### nogood-id-collision-prevention
**Status:** IN

After importing nogoods, `_next_nogood_id` is set to one past the highest imported ID, preventing `network.record_nogood` from generating a conflicting ID later.

**Supports:** [nogood-ids-are-durable-and-collision-free](other.md#nogood-ids-are-durable-and-collision-free)

### nogood-id-counter-is-monotonic
**Status:** IN

`Network._next_nogood_id` only ever increases — deletion, clearing, save/load, and import never decrease it, preventing ID reuse after the issue #26 fix

**Supports:** [nogood-ids-are-durable-and-collision-free](other.md#nogood-ids-are-durable-and-collision-free)

### nogood-id-format-is-zero-padded
**Status:** IN

Nogood IDs follow the format `nogood-NNN` with 3-digit zero-padded integers (e.g., `nogood-001`), and `_next_nogood_id` holds the next integer to assign, not the last used


### nogood-id-monotonic
**Status:** IN

`Network._next_nogood_id` only ever increases; deletions and clears do not decrement it, preventing ID reuse (fix for issue #26)


### nogood-id-persisted-in-meta
**Status:** IN

The nogood counter is stored in the `network_meta` SQLite table and survives save/load cycles as a high-water mark, not a count of current nogoods

**Supports:** [nogood-ids-are-durable-and-collision-free](other.md#nogood-ids-are-durable-and-collision-free)

### nogood-ids-are-durable-and-collision-free
**Status:** IN

Nogood IDs form a durable collision-free sequence: the counter only increases (surviving deletions and clears), persists across save/load cycles via the network_meta table, and advances past imported IDs to prevent collisions — no operation can produce a reused or ambiguous contradiction identifier

**Depends on:** [nogood-id-collision-prevention](other.md#nogood-id-collision-prevention), [nogood-id-counter-is-monotonic](other.md#nogood-id-counter-is-monotonic), [nogood-id-persisted-in-meta](other.md#nogood-id-persisted-in-meta)
**Supports:** [all-identifiers-are-durable-across-persistence-boundaries](other.md#all-identifiers-are-durable-across-persistence-boundaries)

### nogood-ids-assume-append-only
**Status:** OUT

Nogood IDs are derived from `len(self.nogoods) + 1`, so deleting a nogood from the list would cause ID collisions on subsequent calls

**Supports:** [belief-revision-is-fully-reliable](revision.md#belief-revision-is-fully-reliable), [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [nogood-resolution-maintains-consistent-ids](other.md#nogood-resolution-maintains-consistent-ids), [unified-system-is-a-closed-self-maintaining-architecture](system.md#unified-system-is-a-closed-self-maintaining-architecture)

### nogood-resolution-maintains-consistent-ids
**Status:** IN

Nogood recording and resolution produces a consistent, referenceable history of contradictions

**Depends on:** [add-nogood-always-records](other.md#add-nogood-always-records), [backtracking-retracts-least-entrenched](other.md#backtracking-retracts-least-entrenched)
**Supports:** [contradiction-management-is-complete-and-traceable](complete.md#contradiction-management-is-complete-and-traceable), [system-artifacts-maintain-consistent-identification](system.md#system-artifacts-maintain-consistent-identification)

### nogoods-require-valid-nodes
**Status:** IN

Nogoods whose `Affects` list references node IDs not present in the network are silently skipped rather than raising an error during import

**Supports:** [reference-validation-is-defense-in-depth](other.md#reference-validation-is-defense-in-depth)

### normalization-drops-unknown-refs
**Status:** IN

Both `_normalize_markdown` and `_normalize_json` silently drop antecedent/outlist references to IDs not present in the import set, preventing dangling edges in the dependency graph.

**Supports:** [reference-validation-is-defense-in-depth](other.md#reference-validation-is-defense-in-depth)

### only-reasons-lib-is-distributed
**Status:** IN

Only the `reasons_lib` package is included in built distributions — `tests/`, `entries/`, and `reviews/` are excluded by the explicit `packages = ["reasons_lib"]` declaration in pyproject.toml.


### only-reasons-lib-packaged
**Status:** IN

Setuptools is configured to include only the `reasons_lib` package in distributions; tests, entries, reviews, and knowledge-base artifacts are excluded from wheels


### operational-assurance-is-resource-efficient
**Status:** OUT

The system's comprehensive operational assurance — spanning temporal self-correction, end-to-end reliability, and external control — is achieved within a resource-efficient pipeline that minimizes footprint at every lifecycle phase, ensuring operational guarantees do not degrade under resource constraints.

**Depends on:** [resource-efficiency-spans-full-pipeline](spans.md#resource-efficiency-spans-full-pipeline), [system-assurance-spans-correction-reliability-and-control](system.md#system-assurance-spans-correction-reliability-and-control)
**Supports:** [operational-profile-is-safe-assured-and-resource-bounded](safe.md#operational-profile-is-safe-assured-and-resource-bounded), [source-correction-achieves-resource-efficient-assurance](source.md#source-correction-achieves-resource-efficient-assurance)

### operational-guarantees-span-safety-and-trust
**Status:** OUT

The system's operational guarantees span two independent enforcement dimensions: safety that is universal and condition-independent (holding across all layers, backends, and adverse conditions) and trust boundaries that are comprehensively enforced (architectural self-containment and information flow control at every boundary) — together ensuring no operational path can compromise system integrity regardless of layer, backend, or external interaction.

**Depends on:** [operational-safety-is-universal-and-condition-independent](other.md#operational-safety-is-universal-and-condition-independent), [trust-and-information-boundaries-are-comprehensively-enforced](other.md#trust-and-information-boundaries-are-comprehensively-enforced)
**Supports:** [operational-safety-is-defense-in-depth-reinforced](other.md#operational-safety-is-defense-in-depth-reinforced)

### operational-integrity-is-end-to-end
**Status:** OUT

Every network operation achieves both transactional atomicity (load/save gating, snapshot persistence) and semantic determinism (uniform pure evaluation, terminating propagation, reversible defeat), ensuring every mutation produces a predictable, recoverable network state.

**Depends on:** [mutations-are-atomic-and-safely-propagated](other.md#mutations-are-atomic-and-safely-propagated), [reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible)
**Supports:** [internal-and-external-integrity-are-unified](external.md#internal-and-external-integrity-are-unified), [operational-integrity-survives-all-graph-states](other.md#operational-integrity-survives-all-graph-states), [operational-safety-spans-all-mutation-sources](spans.md#operational-safety-spans-all-mutation-sources)

### operational-integrity-survives-all-graph-states
**Status:** OUT

End-to-end operational integrity holds across all semantic edge cases — including vacuous premises, asymmetric absence, and empty antecedents — only when the dependents graph is consistent and propagation handles all node references safely.

**Depends on:** [belief-revision-covers-all-cases-uniformly](revision.md#belief-revision-covers-all-cases-uniformly), [operational-integrity-is-end-to-end](other.md#operational-integrity-is-end-to-end)

### operational-profile-is-traceable-through-equilibria
**Status:** OUT

The system's safe, assured, resource-bounded operational profile produces evaluations that are traceable from individual computation through system-wide convergence to documented equilibria — operational guarantees hold not just at a point in time but longitudinally across the system's entire trajectory toward stable states.

**Depends on:** [evaluation-traceability-persists-through-equilibria](other.md#evaluation-traceability-persists-through-equilibria), [operational-profile-is-safe-assured-and-resource-bounded](safe.md#operational-profile-is-safe-assured-and-resource-bounded)
**Supports:** [operational-traceability-enables-efficient-self-correction](self.md#operational-traceability-enables-efficient-self-correction)

### operational-safety-is-defense-in-depth-reinforced
**Status:** OUT

The system's operational safety guarantees — universal across all architectural layers and condition-independent — are concretely reinforced by defense-in-depth at every external boundary: LLM integration applies layered defenses (bounded execution, fail-soft handling, subprocess isolation) and system boundaries enforce validation, resilience, and resource constraints simultaneously

**Depends on:** [defense-in-depth-spans-llm-and-system-boundaries](spans.md#defense-in-depth-spans-llm-and-system-boundaries), [operational-guarantees-span-safety-and-trust](other.md#operational-guarantees-span-safety-and-trust)
**Supports:** [operational-safety-is-resource-efficient-defense-in-depth](other.md#operational-safety-is-resource-efficient-defense-in-depth)

### operational-safety-is-resource-efficient-defense-in-depth
**Status:** OUT

The system's operational safety guarantees — universal across all architectural layers and condition-independent — are concretely reinforced by defense-in-depth spanning LLM and system boundaries, and this defense-in-depth is achieved within a resource-efficient pipeline from packaging through runtime, meaning comprehensive safety does not trade off against operational cost.

**Depends on:** [defense-in-depth-is-resource-efficient](other.md#defense-in-depth-is-resource-efficient), [operational-safety-is-defense-in-depth-reinforced](other.md#operational-safety-is-defense-in-depth-reinforced)
**Supports:** [operational-profile-is-safe-assured-and-resource-bounded](safe.md#operational-profile-is-safe-assured-and-resource-bounded)

### operational-safety-is-universal-and-condition-independent
**Status:** OUT

Operational safety holds universally across two independent dimensions: across all architectural layers and storage backends (SQLite and PostgreSQL enforce equivalent safety through backend-appropriate mechanisms), AND under all graph conditions including adverse states (dangling references trigger graceful degradation, cyclic justifications are bounded, concurrent access uses WAL mode)

**Depends on:** [all-mutations-preserve-integrity-under-adverse-conditions](other.md#all-mutations-preserve-integrity-under-adverse-conditions), [safety-is-enforced-across-all-layers-and-backends](other.md#safety-is-enforced-across-all-layers-and-backends)
**Supports:** [operational-guarantees-span-safety-and-trust](other.md#operational-guarantees-span-safety-and-trust), [system-assurance-is-universal-and-multidimensional](system.md#system-assurance-is-universal-and-multidimensional)

### origin-agnostic-trustworthiness-is-fully-verifiable
**Status:** OUT

The system's complete revision trustworthiness holds identically across all belief origins and is independently verifiable through full maintenance loop observability — trustworthiness is not merely claimed but provable through origin-agnostic audit trails — provided propagation's dependents invariant holds.

**Depends on:** [closed-loop-is-origin-agnostic](other.md#closed-loop-is-origin-agnostic), [maintenance-loop-is-fully-observable](other.md#maintenance-loop-is-fully-observable), [revision-achieves-complete-trustworthiness](revision.md#revision-achieves-complete-trustworthiness)
**Supports:** [origin-agnosticism-unifies-trustworthiness-and-grounding](other.md#origin-agnosticism-unifies-trustworthiness-and-grounding)

### origin-agnosticism-unifies-trustworthiness-and-grounding
**Status:** OUT

The system's origin-agnostic closed loop simultaneously delivers two independent guarantees from a single architectural source: verifiable trustworthiness across all belief origins and complete invariant grounding for external beliefs — origin indifference is not merely a property but the shared mechanism producing both guarantees.

**Depends on:** [origin-agnostic-loop-grounds-external-invariants](external.md#origin-agnostic-loop-grounds-external-invariants), [origin-agnostic-trustworthiness-is-fully-verifiable](other.md#origin-agnostic-trustworthiness-is-fully-verifiable)
**Supports:** [origin-agnostic-guarantees-are-verifiable-and-self-sustaining](self.md#origin-agnostic-guarantees-are-verifiable-and-self-sustaining)

### out-nodes-excluded-from-staleness
**Status:** IN

Duplicate of existing `check-stale-skips-out-nodes`.


### outlist-absent-means-out
**Status:** IN

An outlist node that doesn't exist in the network is treated as OUT (justification satisfied); absent antecedent nodes fail validation — this asymmetry makes missing counter-evidence permissive while missing supporting evidence is strict

**Supports:** [missing-nodes-have-asymmetric-fail-semantics](other.md#missing-nodes-have-asymmetric-fail-semantics), [outlist-semantics-are-fully-specified](other.md#outlist-semantics-are-fully-specified)

### outlist-enables-non-monotonic-reasoning
**Status:** IN

The `outlist` field on `Justification` allows beliefs to be retracted when a defeating node becomes IN — this is the core non-monotonic mechanism powering `supersede`, `challenge`, and default-logic patterns.

**Supports:** [outlist-is-universal-defeat-mechanism](other.md#outlist-is-universal-defeat-mechanism)

### outlist-is-universal-defeat-mechanism
**Status:** IN

The outlist primitive is the sole defeat mechanism underlying all non-monotonic features: challenges, agent kill-switches, supersession, and direct defeasible reasoning

**Depends on:** [challenge-is-outlist-injection](other.md#challenge-is-outlist-injection), [kill-switch-uses-outlist-not-antecedent](other.md#kill-switch-uses-outlist-not-antecedent), [outlist-enables-non-monotonic-reasoning](other.md#outlist-enables-non-monotonic-reasoning), [supersession-is-reversible](other.md#supersession-is-reversible)
**Supports:** [non-monotonic-system-is-single-reversible-primitive](system.md#non-monotonic-system-is-single-reversible-primitive)

### outlist-nodes-not-in-dependents-index
**Status:** OUT

Outlist nodes are not tracked in the `dependents` index, so when an outlist node is retracted (goes OUT), dependent GATE beliefs are not enqueued for re-evaluation by `_propagate` — requiring manual `reasons assert` as a workaround.

**Supports:** [all-truth-effects-propagate-through-outlist-paths](other.md#all-truth-effects-propagate-through-outlist-paths), [any-mode-expansion-propagates-completely](other.md#any-mode-expansion-propagates-completely), [dedup-reflects-complete-dependency-graph](complete.md#dedup-reflects-complete-dependency-graph), [defeat-reversal-propagates-automatically](other.md#defeat-reversal-propagates-automatically), [defeat-reversal-with-guided-recovery](other.md#defeat-reversal-with-guided-recovery), [dependency-tracking-is-complete-for-all-reference-types](complete.md#dependency-tracking-is-complete-for-all-reference-types), [incremental-propagation-is-fully-complete](complete.md#incremental-propagation-is-fully-complete), [propagation-automatically-cascades-on-all-truth-changes](other.md#propagation-automatically-cascades-on-all-truth-changes), [retraction-reporting-reflects-complete-cascades](complete.md#retraction-reporting-reflects-complete-cascades)

### outlist-relationships-survive-persistence
**Status:** IN

Outlists are stored as `outlist_json` in the SQLite `justifications` table; on load, the dependent index is rebuilt for both antecedents and outlist nodes, preserving propagation behavior across save/load cycles

**Supports:** [outlist-semantics-are-fully-specified](other.md#outlist-semantics-are-fully-specified), [persistence-round-trip-is-lossless](other.md#persistence-round-trip-is-lossless)

### outlist-semantics-are-fully-specified
**Status:** IN

The outlist primitive has complete, well-defined semantics: multiple entries form a conjunction (all must be OUT), absent nodes are treated as OUT (permissive default), and outlist relationships survive persistence through JSON serialization with rebuilt dependent indexes.

**Depends on:** [multiple-outlist-is-conjunction](other.md#multiple-outlist-is-conjunction), [outlist-absent-means-out](other.md#outlist-absent-means-out), [outlist-relationships-survive-persistence](other.md#outlist-relationships-survive-persistence)
**Supports:** [absence-and-outlist-form-complete-negative-semantics](complete.md#absence-and-outlist-form-complete-negative-semantics), [any-mode-expansion-propagates-completely](other.md#any-mode-expansion-propagates-completely), [dependency-tracking-is-complete-for-all-reference-types](complete.md#dependency-tracking-is-complete-for-all-reference-types), [dialectics-inherit-complete-outlist-semantics](complete.md#dialectics-inherit-complete-outlist-semantics), [incremental-propagation-is-fully-complete](complete.md#incremental-propagation-is-fully-complete), [propagation-automatically-cascades-on-all-truth-changes](other.md#propagation-automatically-cascades-on-all-truth-changes)

### package-name-split
**Status:** IN

The pip-installable name is `ftl-reasons` but the importable Python package is `reasons_lib`; these are deliberately decoupled.


### parse-review-defaults-to-passing
**Status:** IN

`parse_review_response` defaults missing fields to valid=True, sufficient=True, necessary=True, unnecessary_antecedents=[] — the LLM only needs to report failures explicitly; omission means the belief passed.


### parse-review-response-never-raises
**Status:** IN

`parse_review_response` returns an empty list on any malformed input — bad JSON, non-list JSON, items missing `id` fields — rather than raising exceptions, with missing boolean fields defaulting to `True`.


### persistence-is-snapshot-not-incremental
**Status:** OUT

The storage layer operates as a full snapshot: save replaces all rows, load trusts stored values without re-propagation, and the dependents index is rebuilt from scratch

**Depends on:** [dependents-index-derived-on-load](other.md#dependents-index-derived-on-load), [storage-load-bypasses-propagation](other.md#storage-load-bypasses-propagation), [storage-save-is-full-replace](other.md#storage-save-is-full-replace), [storage-trusts-stored-truth-values](other.md#storage-trusts-stored-truth-values)
**Supports:** [data-integrity-spans-architecture](spans.md#data-integrity-spans-architecture), [mutation-pipeline-is-atomic-snapshot](other.md#mutation-pipeline-is-atomic-snapshot), [persistence-round-trip-is-lossless](other.md#persistence-round-trip-is-lossless)

### persistence-round-trip-is-lossless
**Status:** OUT

The save/load round trip preserves all network state faithfully: snapshot persistence captures the full graph, stored truth values are trusted without re-propagation, justification insertion order is preserved via rowid, and outlist relationships survive serialization.

**Depends on:** [justification-order-preserved-via-rowid](justification.md#justification-order-preserved-via-rowid), [outlist-relationships-survive-persistence](other.md#outlist-relationships-survive-persistence), [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental)

### pg-access-control-raises-permission-error
**Status:** IN

`PgApi.show_node()` raises `PermissionError` when called with a `visible_to` list that doesn't intersect the node's `access_tags`, enforcing per-node access control at the API boundary


### pg-antecedent-refs-have-no-fk-constraints
**Status:** IN

Antecedent and outlist references in `rms_justifications` are JSONB arrays without foreign key constraints; nonexistent referenced nodes default to truth value OUT via `truth_cache.get(a, "OUT")`.

**Supports:** [pg-data-integrity-achieves-defense-in-depth](other.md#pg-data-integrity-achieves-defense-in-depth), [pg-multi-tenancy-is-referentially-complete](complete.md#pg-multi-tenancy-is-referentially-complete), [pgapi-referential-integrity-is-database-enforced](other.md#pgapi-referential-integrity-is-database-enforced)

### pg-api-what-if-never-mutates
**Status:** IN

`PgApi.what_if_retract()` and `what_if_assert()` are read-only operations — they return cascade analysis (changed/went_in/went_out) without modifying database state, verified by checking `get_status()` before and after

**Supports:** [pg-what-if-is-safely-simulated](other.md#pg-what-if-is-safely-simulated)

### pg-cleanup-covers-five-tables
**Status:** IN

Fixture teardown deletes from exactly `rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, and `rms_nodes` — adding a new `project_id`-scoped table without updating the fixture will leak test data.


### pg-conninfo-accepts-flag-envvar-or-both
**Status:** IN

PostgreSQL connection is configured via `--pg`/`--project-id` CLI flags or `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables, with CLI flags taking precedence over env vars — `_backend_kwargs(args)` handles the dispatch.


### pg-data-integrity-achieves-defense-in-depth
**Status:** OUT

PgApi's data integrity achieves defense-in-depth through both application-level enforcement (write-time referential validation and outlist-aware dependent queries spanning antecedent and outlist references) and comprehensive input validation at all system boundaries, but full defense-in-depth requires database-level foreign key constraints to provide a second independent enforcement layer

**Depends on:** [input-validation-is-comprehensive-at-all-boundaries](other.md#input-validation-is-comprehensive-at-all-boundaries), [pgapi-enforces-referential-integrity-bidirectionally](other.md#pgapi-enforces-referential-integrity-bidirectionally)

### pg-dispatch-is-function-level-early-return
**Status:** IN

PostgreSQL routing uses a function-level early-return pattern — each API function checks `pg_conninfo` and short-circuits to `_pg_dispatch`, which instantiates PgApi and calls the matching method via `getattr` — rather than using abstract base classes, subclassing, or factory patterns.


### pg-dispatch-requires-project-id
**Status:** IN

When `pg_conninfo` is provided (via CLI `--pg` flag or `REASONS_PG_CONNINFO` env var) without a corresponding `project_id`, the system calls `sys.exit()` rather than proceeding with an incomplete configuration


### pg-extra-required-for-postgres
**Status:** IN

PostgreSQL support requires installing the `pg` or `test-pg` optional extra (`psycopg[binary]>=3.1`); it is not available in a bare install.


### pg-fixture-provides-per-test-isolation
**Status:** IN

Each pg test receives an isolated `PgApi` instance via the `pg_api` fixture, which creates a fresh project namespace (UUID) and tears down after each test, ensuring zero shared state between tests.


### pg-fixture-uses-uuid-isolation
**Status:** IN

Each `pg_api` test fixture partitions its data by a unique UUID v4 `project_id` rather than using transactions or ephemeral databases, enabling safe concurrent test execution against a shared database.


### pg-keyerror-includes-missing-node-id
**Status:** IN

When `add_node` references a nonexistent node in `sl=` or `unless=`, PgApi raises `KeyError` with the missing node ID in the exception message — tested via `pytest.raises(KeyError, match="ghost")` — enabling callers to identify which reference is dangling.


### pg-multi-tenancy-via-project-id
**Status:** IN

`PgApi` isolates beliefs by `project_id` so identical node IDs in different projects store independent data with no cross-contamination — each test gets a unique UUID project


### pg-psycopg-is-optional-dependency
**Status:** IN

`psycopg` (v3) is soft-imported with fallback to `None`; `_require_psycopg()` raises `ImportError` with install instructions at `PgApi` construction time, mirroring the `mcp_client.py` lazy-guard pattern.


### pg-reimplements-network-in-sql
**Status:** OUT

`PgApi` reimplements the in-memory `Network`'s algorithms (BFS propagation, entrenchment scoring, nogood resolution, dialectical operations) directly in SQL rather than delegating to the `Network` class.

**Supports:** [pgapi-achieves-implementation-parity](other.md#pgapi-achieves-implementation-parity), [pgapi-is-complete-sql-reimplementation](complete.md#pgapi-is-complete-sql-reimplementation)

### pg-search-includes-one-hop-neighbors
**Status:** IN

PostgreSQL full-text search via `plainto_tsquery` includes 1-hop neighbor expansion — direct antecedents and dependents of matching nodes are included in results, providing richer context than exact-match-only search.


### pg-teardown-rollback-before-delete
**Status:** IN

The `pg_api` fixture calls `conn.rollback()` before cleanup deletes to recover from any failed-transaction state left by the test, preventing cleanup failures from cascading.


### pg-test-suite-is-backend-parity
**Status:** IN

`test_pg.py` is a parity test suite: every behavior tested has a corresponding expected behavior from the SQLite backend, validating that PgApi returns the same dict shapes and enforces the same invariants to ensure backend interchangeability.


### pg-unsupported-params-raise-not-implemented
**Status:** IN

When PgApi-routed commands receive unsupported parameters (e.g., search with `depth != 1`, list with `challenged`/`min_depth`, add with `namespace`), the dispatch raises `NotImplementedError` with a clear message rather than silently ignoring the parameter.


### pg-uses-jsonb-with-gin-for-dependent-queries
**Status:** IN

PgApi stores antecedents and outlist as JSONB arrays with GIN indexes, enabling per-operation dependent lookups via `@>` containment queries — unlike the SQLite backend which loads the full network, PgApi queries relationships incrementally.


### pg-uses-project-id-for-multi-tenancy
**Status:** IN

Every `PgApi` query includes `project_id` in its WHERE clause, providing multi-tenant isolation with no cross-project queries and composite primary keys `(id, project_id)`.


### pg-what-if-is-safely-simulated
**Status:** IN

PgApi's what-if operations achieve safe simulation: mutations are performed against real PostgreSQL data for accurate cascade analysis, then rolled back within a transaction to guarantee zero persistent side effects — combining fidelity with safety.

**Depends on:** [pg-api-what-if-never-mutates](other.md#pg-api-what-if-never-mutates), [pg-what-if-uses-transaction-rollback](other.md#pg-what-if-uses-transaction-rollback)
**Supports:** [both-backends-support-safe-hypothetical-reasoning](safe.md#both-backends-support-safe-hypothetical-reasoning)

### pg-what-if-uses-transaction-rollback
**Status:** IN

`what_if_retract`/`what_if_assert` perform real mutations inside a transaction, collect cascade effects, then rollback — reusing the propagation engine without duplicating logic.

**Supports:** [pg-what-if-is-safely-simulated](other.md#pg-what-if-is-safely-simulated)

### pgapi-achieves-implementation-parity
**Status:** OUT

PgApi achieves full behavioral parity with the in-memory Network implementation: it reimplements the core algorithms (entrenchment scoring, nogood resolution, BFS propagation, dialectics) in SQL with BFS propagation executed in application-level Python.

**Depends on:** [pg-reimplements-network-in-sql](other.md#pg-reimplements-network-in-sql), [pgapi-bfs-propagation-in-python](other.md#pgapi-bfs-propagation-in-python)

### pgapi-bfs-propagation-in-python
**Status:** IN

PgApi implements BFS propagation in application-level Python (not stored procedures), using JSONB containment queries (`@>`) against GIN indexes to find dependents

**Supports:** [pg-multi-tenancy-is-referentially-complete](complete.md#pg-multi-tenancy-is-referentially-complete), [pgapi-achieves-implementation-parity](other.md#pgapi-achieves-implementation-parity)

### pgapi-enforces-referential-integrity-bidirectionally
**Status:** IN

PgApi enforces referential integrity in both directions: write-time validation checks all antecedent and outlist IDs exist before inserting justifications, while read-time dependent discovery queries both antecedent and outlist JSONB containment to find all affected nodes.

**Depends on:** [pgapi-find-dependents-queries-outlist](other.md#pgapi-find-dependents-queries-outlist), [pgapi-validates-refs-before-justification-insert](justification.md#pgapi-validates-refs-before-justification-insert)
**Supports:** [pg-data-integrity-achieves-defense-in-depth](other.md#pg-data-integrity-achieves-defense-in-depth), [pgapi-referential-integrity-is-database-enforced](other.md#pgapi-referential-integrity-is-database-enforced)

### pgapi-find-dependents-queries-outlist
**Status:** IN

PgApi's `_find_dependents` queries both `antecedents @>` and `outlist @>` JSONB containment, correctly enqueuing outlist-dependent nodes for re-evaluation — a capability the in-memory Network historically lacked until PR #31.

**Supports:** [pgapi-enforces-referential-integrity-bidirectionally](other.md#pgapi-enforces-referential-integrity-bidirectionally)

### pgapi-has-no-retry-logic
**Status:** IN

`PgApi` uses one connection per instance with no retry on transient database errors; callers are expected to handle `psycopg` exceptions from connection failures or serialization conflicts.


### pgapi-is-sql-native-multi-tenant
**Status:** IN

PgApi operates as a SQL-native multi-tenant implementation: all operations execute directly against PostgreSQL with no in-memory Network object constructed, composite primary keys on all tables provide project-level isolation, and each public method is a single committed transaction.

**Depends on:** [pgapi-multi-tenant-composite-keys](other.md#pgapi-multi-tenant-composite-keys), [pgapi-no-in-memory-network](other.md#pgapi-no-in-memory-network), [pgapi-one-transaction-per-method](other.md#pgapi-one-transaction-per-method)
**Supports:** [atomicity-is-backend-independent](other.md#atomicity-is-backend-independent), [dual-backend-atomicity-is-feature-complete](complete.md#dual-backend-atomicity-is-feature-complete), [pg-multi-tenancy-is-referentially-complete](complete.md#pg-multi-tenancy-is-referentially-complete), [pgapi-is-complete-sql-reimplementation](complete.md#pgapi-is-complete-sql-reimplementation), [pgapi-referential-integrity-is-database-enforced](other.md#pgapi-referential-integrity-is-database-enforced)

### pgapi-multi-tenant-composite-keys
**Status:** IN

All PgApi tables use composite primary keys `(id, project_id)` for multi-tenancy, with JSONB columns (not TEXT) for antecedents, outlist, and metadata

**Supports:** [pgapi-is-sql-native-multi-tenant](other.md#pgapi-is-sql-native-multi-tenant)

### pgapi-no-in-memory-network
**Status:** IN

`PgApi` executes all operations as direct SQL against PostgreSQL — no in-memory `Network` object is ever constructed, enabling concurrent writers

**Supports:** [pgapi-is-sql-native-multi-tenant](other.md#pgapi-is-sql-native-multi-tenant)

### pgapi-one-transaction-per-method
**Status:** IN

Each PgApi public method is a single PostgreSQL transaction that commits or rolls back at the end; __exit__ rolls back on exception

**Supports:** [pgapi-is-sql-native-multi-tenant](other.md#pgapi-is-sql-native-multi-tenant)

### pgapi-partial-api-coverage
**Status:** OUT

PgApi implements core operations (add/retract/assert/search/nogood/explain) but defers simulation, dialectics, namespace support, import/export, and maintenance operations to future work

**Supports:** [backend-agnostic-operational-assurance](other.md#backend-agnostic-operational-assurance), [backend-independent-self-correction](self.md#backend-independent-self-correction), [complete-architecture-is-backend-portable](complete.md#complete-architecture-is-backend-portable), [dual-backend-atomicity-is-feature-complete](complete.md#dual-backend-atomicity-is-feature-complete), [dual-storage-backends-are-interchangeable](other.md#dual-storage-backends-are-interchangeable), [operational-profile-spans-all-backends](spans.md#operational-profile-spans-all-backends), [pgapi-achieves-implementation-parity](other.md#pgapi-achieves-implementation-parity), [pgapi-is-complete-sql-reimplementation](complete.md#pgapi-is-complete-sql-reimplementation), [rich-governance-spans-all-backends](governance.md#rich-governance-spans-all-backends), [storage-is-fully-production-grade-across-backends](other.md#storage-is-fully-production-grade-across-backends), [verified-correctness-extends-to-all-backends](other.md#verified-correctness-extends-to-all-backends)

### pgapi-referential-integrity-is-database-enforced
**Status:** OUT

PgApi achieves complete referential integrity: application-level bidirectional enforcement (write-time validation and outlist-aware dependent querying) is complemented by database-level foreign key constraints, preventing orphaned justification references even under direct SQL manipulation outside the application layer.

**Depends on:** [pgapi-enforces-referential-integrity-bidirectionally](other.md#pgapi-enforces-referential-integrity-bidirectionally), [pgapi-is-sql-native-multi-tenant](other.md#pgapi-is-sql-native-multi-tenant)

### pgapi-what-if-uses-transaction-rollback
**Status:** IN

PgApi's `what_if_retract` and `what_if_assert` wrap BFS cascade analysis inside a database transaction with `try/finally` guaranteed ROLLBACK, achieving read-only simulation by never committing the exploratory mutations.


### premise-behavior-emerges-from-absence
**Status:** IN

Premise behavior is not explicitly implemented — it emerges from three defaults: nodes with no justifications default to IN, empty antecedent lists are vacuously valid, and the system preserves a premise's current truth value rather than deriving it.

**Depends on:** [empty-antecedents-vacuously-valid](other.md#empty-antecedents-vacuously-valid), [premise-defaults-to-in](other.md#premise-defaults-to-in), [premises-have-no-justifications](other.md#premises-have-no-justifications)
**Supports:** [absence-has-consistent-dual-semantics](other.md#absence-has-consistent-dual-semantics), [challenge-destroys-premise-identity](other.md#challenge-destroys-premise-identity), [premise-identity-is-inherently-transient](other.md#premise-identity-is-inherently-transient), [truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)

### premise-default-in
**Status:** IN

Covered by existing `premise-behavior-emerges-from-absence` which captures this as an emergent property


### premise-defaults-to-in
**Status:** IN

A node with no justifications (a premise) defaults to IN; `_compute_truth` preserves its current truth value rather than recomputing it.

**Supports:** [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence)

### premise-derived-verifiability-asymmetry
**Status:** IN

Premises record source file and line range at assertion time, enabling check-stale to verify them against current code; derived beliefs have no equivalent source-level grounding and can only be structurally validated (references exist and are IN).


### premise-identity-is-bidirectionally-transformable
**Status:** OUT

Premise identity can be both destroyed (via dialectical challenge adding justifications) and created (via convert-to-premise removing them), with both directions preserving the dependents invariant — making premise/derived status a fully reversible structural property of the network.

**Depends on:** [challenge-destroys-premise-identity](other.md#challenge-destroys-premise-identity), [convert-to-premise-preserves-dependents-invariant](other.md#convert-to-premise-preserves-dependents-invariant)
**Supports:** [identity-transformation-is-complete-and-reliable](complete.md#identity-transformation-is-complete-and-reliable)

### premise-identity-is-inherently-transient
**Status:** OUT

Premise identity is inherently transient because it emerges from the absence of justifications, and any justification addition — whether from dialectical challenge, defend, or direct add_justification — irreversibly transforms a premise into a derived node without explicit opt-in.

**Depends on:** [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence), [premise-can-receive-justification](justification.md#premise-can-receive-justification)
**Supports:** [premise-identity-transformation-is-architecturally-asymmetric](other.md#premise-identity-transformation-is-architecturally-asymmetric)

### premise-identity-transformation-is-architecturally-asymmetric
**Status:** OUT

Premise identity transformation exhibits a fundamental architectural asymmetry rooted in the same emergent property: premise identity is inherently transient because it arises from the absence of justifications, and dialectical challenge exploits this transience to permanently transform premises into justified nodes — while the truth-value defeat itself remains fully reversible through outlist semantics, creating an irreversible identity change layered atop reversible truth dynamics

**Depends on:** [dialectical-defeat-is-reversible-but-identity-is-permanent](other.md#dialectical-defeat-is-reversible-but-identity-is-permanent), [premise-identity-is-inherently-transient](other.md#premise-identity-is-inherently-transient)
**Supports:** [identity-transformation-is-semantically-invisible](other.md#identity-transformation-is-semantically-invisible)

### premises-have-no-justifications
**Status:** IN

A premise node is represented by an empty `justifications` list and defaults to `truth_value="IN"`; the system treats the empty-justifications case as a special unconditional belief.

**Supports:** [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence)

### propagate-assumes-dependents-exist
**Status:** OUT

Every ID in `node.dependents` is accessed via `self.nodes[dep_id]` without a membership check; a dangling dependent reference will raise `KeyError` — this is intentional (broken invariant = bug)

**Supports:** [all-exceptions-are-safely-handled](other.md#all-exceptions-are-safely-handled), [belief-revision-is-fully-reliable](revision.md#belief-revision-is-fully-reliable), [challenge-defense-is-crash-safe](safe.md#challenge-defense-is-crash-safe), [complete-architecture-achieves-verified-production-correctness](complete.md#complete-architecture-achieves-verified-production-correctness), [complete-operational-uniformity-across-all-sources](complete.md#complete-operational-uniformity-across-all-sources), [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [convergent-equilibria-have-complete-propagation-fidelity](complete.md#convergent-equilibria-have-complete-propagation-fidelity), [dialectical-transformation-is-fully-reliable](other.md#dialectical-transformation-is-fully-reliable), [full-system-integrity-is-gap-free](system.md#full-system-integrity-is-gap-free), [maintenance-loop-is-fully-observable](other.md#maintenance-loop-is-fully-observable), [mutation-pipeline-produces-consistent-state](other.md#mutation-pipeline-produces-consistent-state), [operational-integrity-survives-all-graph-states](other.md#operational-integrity-survives-all-graph-states), [origin-agnostic-trustworthiness-is-fully-verifiable](other.md#origin-agnostic-trustworthiness-is-fully-verifiable), [propagation-is-crash-free](other.md#propagation-is-crash-free), [revision-achieves-complete-trustworthiness](revision.md#revision-achieves-complete-trustworthiness), [revision-coverage-is-verifiably-sound](revision.md#revision-coverage-is-verifiably-sound), [revision-coverage-requires-sound-propagation](revision.md#revision-coverage-requires-sound-propagation), [self-correction-is-resource-sustainable](self.md#self-correction-is-resource-sustainable), [self-maintenance-is-fully-auditable](self.md#self-maintenance-is-fully-auditable), [system-achieves-full-correctness](system.md#system-achieves-full-correctness), [system-operations-never-crash](system.md#system-operations-never-crash), [system-properties-are-indefinitely-maintained](system.md#system-properties-are-indefinitely-maintained), [tms-core-is-crash-safe](safe.md#tms-core-is-crash-safe), [tms-handles-all-conditions-safely](other.md#tms-handles-all-conditions-safely), [unified-system-is-a-closed-self-maintaining-architecture](system.md#unified-system-is-a-closed-self-maintaining-architecture), [verified-mutation-correctness-across-boundaries](other.md#verified-mutation-correctness-across-boundaries)

### propagate-cascade-stops-on-unchanged
**Status:** IN

If a dependent's recomputed truth value equals its current value, it is not enqueued — the cascade terminates along that path, making propagation selective rather than exhaustive

**Supports:** [propagation-is-crash-free](other.md#propagation-is-crash-free), [propagation-terminates-deterministically](other.md#propagation-terminates-deterministically)

### propagate-does-not-change-trigger
**Status:** IN

The seed node (`changed_id`) is added to `visited` immediately and never has its own truth value recomputed; callers must update it before calling `_propagate`

**Supports:** [propagation-respects-node-lifecycle](lifecycle.md#propagation-respects-node-lifecycle)

### propagate-skips-retracted-nodes
**Status:** IN

`_propagate` never recomputes truth values for nodes with `_retracted` in metadata, even if their justifications would support IN; only `assert_node` can restore them


### propagation-automatically-cascades-on-all-truth-changes
**Status:** OUT

Truth propagation automatically cascades to all dependent nodes whenever any node's truth value changes, including nodes that appear only in outlists — making GATE belief re-evaluation fully automatic with no manual intervention required after outlist node retraction.

**Depends on:** [outlist-semantics-are-fully-specified](other.md#outlist-semantics-are-fully-specified), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating)

### propagation-is-bfs
**Status:** IN

Truth value propagation in `_propagate` uses `deque`-based BFS through the `dependents` graph, not DFS, ensuring breadth-first wavefront expansion.

**Supports:** [propagation-is-crash-free](other.md#propagation-is-crash-free), [propagation-terminates-deterministically](other.md#propagation-terminates-deterministically)

### propagation-is-crash-free
**Status:** IN

Truth propagation completes without runtime errors across all reachable nodes

**Depends on:** [derive-depth-cycle-guard](derive.md#derive-depth-cycle-guard), [propagate-cascade-stops-on-unchanged](other.md#propagate-cascade-stops-on-unchanged), [propagation-is-bfs](other.md#propagation-is-bfs), [propagation-terminates-deterministically](other.md#propagation-terminates-deterministically)
**Supports:** [read-and-write-paths-are-both-reliable](other.md#read-and-write-paths-are-both-reliable)

### propagation-is-immediate
**Status:** IN

Covered by existing `mutations-are-atomic-and-safely-propagated` and `propagation-is-safe-and-terminating`


### propagation-terminates-deterministically
**Status:** IN

Truth propagation is guaranteed to terminate: BFS prevents stack overflow, stop-on-unchanged prevents oscillation, and fixpoint iteration bounds the outer loop

**Depends on:** [propagate-cascade-stops-on-unchanged](other.md#propagate-cascade-stops-on-unchanged), [propagation-is-bfs](other.md#propagation-is-bfs), [recompute-all-uses-fixpoint](other.md#recompute-all-uses-fixpoint)
**Supports:** [add-justification-achieves-consistent-propagation](justification.md#add-justification-achieves-consistent-propagation), [all-reconciliation-converges-deterministically](other.md#all-reconciliation-converges-deterministically), [belief-revision-is-fully-reliable](revision.md#belief-revision-is-fully-reliable), [challenge-defense-is-crash-safe](safe.md#challenge-defense-is-crash-safe), [contradiction-triggers-deterministic-resolution](deterministic.md#contradiction-triggers-deterministic-resolution), [propagation-is-crash-free](other.md#propagation-is-crash-free), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating), [tms-core-is-crash-safe](safe.md#tms-core-is-crash-safe), [tms-core-is-deterministic-and-conservative](deterministic.md#tms-core-is-deterministic-and-conservative)

### pure-evaluation-enables-richly-governed-dialectics
**Status:** IN

Evaluation purity — grounding dialectics through the minimal architecture — simultaneously enables richly-governed exception-safe revision, so dialectical structures are both minimality-grounded in their computation and richly-governed in the state they produce: challenge/defend operations inherit pure deterministic evaluation while producing metadata-enriched recoverable state changes.

**Depends on:** [evaluation-purity-grounds-dialectics-through-minimal-architecture](other.md#evaluation-purity-grounds-dialectics-through-minimal-architecture), [revision-is-richly-governed-and-exception-safe](revision.md#revision-is-richly-governed-and-exception-safe)
**Supports:** [dialectics-are-dually-grounded-by-purity-and-uniformity](other.md#dialectics-are-dually-grounded-by-purity-and-uniformity), [evaluation-purity-grounds-governance-that-exception-safety-preserves](governance.md#evaluation-purity-grounds-governance-that-exception-safety-preserves)

### python-310-floor
**Status:** IN

The project requires Python 3.10+ (`requires-python = ">=3.10"`), establishing the minimum language features available throughout the codebase.


### python-310-minimum
**Status:** IN

The project requires Python >= 3.10 (`requires-python = ">=3.10"`), enabling use of structural pattern matching and `X | Y` union type syntax throughout the codebase.


### python-version-floor-3-10
**Status:** IN

The project requires Python >=3.10, allowing use of match statements, union type syntax with `|`, and other 3.10+ features throughout `reasons_lib`.


### read-and-write-paths-are-both-reliable
**Status:** IN

Both the read path (staleness checking detects all forms of source drift without false negatives) and the write path (truth propagation completes without runtime errors across all reachable nodes) are operationally reliable, ensuring the system functions correctly in both observational and mutational modes.

**Depends on:** [propagation-is-crash-free](other.md#propagation-is-crash-free), [staleness-checking-is-comprehensive](other.md#staleness-checking-is-comprehensive)
**Supports:** [revision-is-end-to-end-reliable](revision.md#revision-is-end-to-end-reliable), [system-reliability-spans-internal-and-external](system.md#system-reliability-spans-internal-and-external)

### reasoning-and-knowledge-expansion-are-both-exhaustive
**Status:** OUT

The system achieves exhaustive coverage in both formal reasoning (deterministic reversible truth evaluation with guaranteed-terminating exploration of all derivable conclusions) and LLM-driven knowledge expansion (complete coverage with fault tolerance across all interactive and batch LLM operations)

**Depends on:** [all-llm-operations-achieve-coverage-and-fault-tolerance](llm.md#all-llm-operations-achieve-coverage-and-fault-tolerance), [reasoning-is-exhaustively-deterministic](deterministic.md#reasoning-is-exhaustively-deterministic)
**Supports:** [exhaustive-knowledge-expansion-within-controlled-boundaries](other.md#exhaustive-knowledge-expansion-within-controlled-boundaries), [system-sustainably-grows-and-self-corrects](system.md#system-sustainably-grows-and-self-corrects)

### reasons-cli-entrypoint-is-cli-main
**Status:** IN

The `reasons` CLI command is registered as `reasons_lib.cli:main` via `[project.scripts]` in pyproject.toml — changing that function's signature or module location breaks the installed command.


### rebuild-dependents-clears-before-rebuilding
**Status:** IN

`_rebuild_dependents()` wipes all existing dependent sets before recomputing from justifications, so stale entries are always removed rather than incrementally patched


### rebuild-dependents-is-idempotent
**Status:** IN

Calling `_rebuild_dependents()` twice in succession produces identical `dependents` sets on every node.

**Supports:** [critical-operations-converge-to-fixed-points](other.md#critical-operations-converge-to-fixed-points)

### recompute-all-uses-fixpoint
**Status:** IN

`recompute_all` iterates until no truth values change, bounded by `len(nodes) + 1` iterations, handling cascading dependencies from arbitrary node ordering.

**Supports:** [critical-operations-converge-to-fixed-points](other.md#critical-operations-converge-to-fixed-points), [propagation-terminates-deterministically](other.md#propagation-terminates-deterministically)

### reference-validation-is-defense-in-depth
**Status:** OUT

Every system boundary that accepts node ID references validates them against the actual network: import normalization drops unknown antecedent/outlist refs, nogood recording skips invalid node IDs, and LLM-returned negative-list IDs are filtered against existing nodes.

**Depends on:** [api-list-negative-filters-hallucinated-ids](other.md#api-list-negative-filters-hallucinated-ids), [nogoods-require-valid-nodes](other.md#nogoods-require-valid-nodes), [normalization-drops-unknown-refs](other.md#normalization-drops-unknown-refs)
**Supports:** [input-validation-is-comprehensive-at-all-boundaries](other.md#input-validation-is-comprehensive-at-all-boundaries), [system-boundaries-are-evolution-tolerant-and-reference-safe](system.md#system-boundaries-are-evolution-tolerant-and-reference-safe)

### references-are-durable-across-persistence-and-evolution
**Status:** OUT

All system-generated identifiers survive both persistence boundaries (save/load cycles, cross-session durability via high-water marks) and format evolution boundaries (parser versioning, schema migration, forward-compatible metadata) — references remain valid and resolvable across time and system versions.

**Depends on:** [all-identifiers-are-durable-across-persistence-boundaries](other.md#all-identifiers-are-durable-across-persistence-boundaries), [system-boundaries-are-evolution-tolerant-and-reference-safe](system.md#system-boundaries-are-evolution-tolerant-and-reference-safe)
**Supports:** [self-correction-history-is-durably-documented](self.md#self-correction-history-is-durably-documented)

### removal-effects-are-fully-reported-and-recoverable
**Status:** OUT

Every belief removal — whether intentional retraction or contradiction-triggered backtracking — provides three simultaneous guarantees: complete cascade coverage (transitive propagation captures every truth-value change), accurate effect reporting (structured before/after diffs reflect all affected nodes), and surgical recovery guidance (restoration hints target only cascade victims with surviving premises).

**Depends on:** [all-removals-provide-reporting-and-recovery](other.md#all-removals-provide-reporting-and-recovery), [retraction-reporting-reflects-complete-cascades](complete.md#retraction-reporting-reflects-complete-cascades)
**Supports:** [all-modifications-converge-with-reporting-and-recovery](other.md#all-modifications-converge-with-reporting-and-recovery)

### rename-from-rms-at-0.3.0
**Status:** IN

The project was renamed from `rms` to `reasons` in version 0.3.0, driven by a measured 5 percentage-point LLM accuracy improvement in ablation study


### require-sqlite-blocks-pg-for-guarded-commands
**Status:** IN

`_require_sqlite` enforces that certain subcommands (e.g., `hash-sources`) cannot run against a Postgres backend, checking both CLI flags and environment variables and calling `sys.exit()` if PG is detected


### resource-efficient-guarantees-are-universal-and-permanent
**Status:** OUT

The system's universal and permanent guarantees are achieved within resource-efficient bounds — self-sustainability, comprehensive auditability, and resource efficiency form a self-reinforcing triad that extends to all belief types without temporal degradation and without resource exhaustion.

**Depends on:** [system-guarantees-are-universal-and-permanent](system.md#system-guarantees-are-universal-and-permanent), [system-is-resource-efficient-self-sustaining-and-auditable](system.md#system-is-resource-efficient-self-sustaining-and-auditable)

### resource-management-supports-belief-currency
**Status:** OUT

Active belief currency management — sustainable derivation of new beliefs and staleness detection for existing ones — operates with accurate bidirectional token budget control, ensuring derivation rounds allocate resources correctly per agent and output fits context-limited consumer constraints.

**Depends on:** [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [token-budgets-are-accurate-bidirectionally](other.md#token-budgets-are-accurate-bidirectionally)
**Supports:** [resource-sustainable-lifecycle-has-no-gaps](lifecycle.md#resource-sustainable-lifecycle-has-no-gaps), [self-correction-is-resource-sustainable](self.md#self-correction-is-resource-sustainable)

### restoration-hints-are-surgical
**Status:** IN

Restoration hints provide surgical recovery guidance after retraction cascades: hints exclude the directly retracted node (targeting only cascade victims) and require at least one surviving premise in a multi-premise justification — narrowing recovery scope to nodes that can actually be independently re-justified.

**Depends on:** [hints-exclude-directly-retracted-node](other.md#hints-exclude-directly-retracted-node), [restoration-hints-require-surviving-premises](other.md#restoration-hints-require-surviving-premises)
**Supports:** [contradiction-resolution-minimizes-disruption-and-guides-recovery](other.md#contradiction-resolution-minimizes-disruption-and-guides-recovery), [retraction-effects-are-reported-with-recovery-guidance](other.md#retraction-effects-are-reported-with-recovery-guidance)

### restoration-hints-require-surviving-premises
**Status:** IN

A retraction cascade only produces a `restoration_hint` for a node if it has a multi-premise SL justification and at least one of its premises is still IN after the cascade.

**Supports:** [defeat-reversal-with-guided-recovery](other.md#defeat-reversal-with-guided-recovery), [restoration-hints-are-surgical](other.md#restoration-hints-are-surgical)

### result-schema-uniformity
**Status:** IN

Both `content_changed` and `source_deleted` results from `check_stale` share exactly the same six keys: `node_id`, `old_hash`, `new_hash`, `source`, `source_path`, `reason` — consumers can handle both uniformly.


### results-sorted-by-node-id
**Status:** IN

`check_stale()` returns results sorted lexicographically by `node_id`, enforced by iterating `sorted(network.nodes.items())`.


### retract-computes-restoration-hints
**Status:** IN

`retract_node` computes `restoration_hints` by examining surviving premises of multi-antecedent justifications, enabling callers to identify which premises could rebuild retracted derived beliefs.


### retract-returns-changed-set
**Status:** IN

`Network.retract()` returns a list of all node IDs whose truth value changed, including the target and all transitively affected dependents; retracting an already-OUT node returns `[]`

**Supports:** [every-mutation-reports-its-effects](other.md#every-mutation-reports-its-effects)

### retracted-nodes-skipped-in-propagation
**Status:** IN

Retracted nodes (marked with `_retracted` metadata) are skipped during BFS propagation but remain in the network for potential restoration.

**Supports:** [propagation-respects-node-lifecycle](lifecycle.md#propagation-respects-node-lifecycle)

### retracted-pin-survives-recompute
**Status:** IN

A node explicitly retracted via `retract()` gets a `_retracted` metadata flag that pins it OUT — surviving both `assert_node` on its antecedents and `recompute_all()`, clearable only by `assert_node` on the pinned node itself

**Supports:** [metadata-is-universal-extension-mechanism](other.md#metadata-is-universal-extension-mechanism)

### retraction-cascade-is-transitive-and-terminating
**Status:** IN

Retraction cascades are both transitive in reach (propagating OUT to all transitively dependent SL-derived nodes, not just direct children) and guaranteed to terminate safely (BFS prevents oscillation, retracted nodes are skipped, and the cascade stops when no truth values change) — ensuring thorough impact without risk of runaway propagation.

**Depends on:** [api-retract-cascade-is-transitive](other.md#api-retract-cascade-is-transitive), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating)
**Supports:** [contradiction-resolution-achieves-minimal-impact-complete-cascades](complete.md#contradiction-resolution-achieves-minimal-impact-complete-cascades), [graph-traversal-is-complete-and-terminating-in-both-directions](complete.md#graph-traversal-is-complete-and-terminating-in-both-directions), [mutations-are-traceable-through-transitive-cascades](other.md#mutations-are-traceable-through-transitive-cascades), [retraction-reporting-reflects-complete-cascades](complete.md#retraction-reporting-reflects-complete-cascades), [system-converges-from-addition-and-removal](system.md#system-converges-from-addition-and-removal)

### retraction-effects-are-reported-with-recovery-guidance
**Status:** OUT

Every retraction provides both complete effect reporting (structured before/after diffs for all mutating operations, full changed set) and surgical recovery guidance (restoration hints targeting only cascade victims with surviving premises, excluding the directly retracted node), giving callers full situational awareness and actionable recovery paths from a single operation.

**Depends on:** [every-mutation-reports-its-effects](other.md#every-mutation-reports-its-effects), [restoration-hints-are-surgical](other.md#restoration-hints-are-surgical)
**Supports:** [all-removals-provide-reporting-and-recovery](other.md#all-removals-provide-reporting-and-recovery), [retraction-reporting-reflects-complete-cascades](complete.md#retraction-reporting-reflects-complete-cascades)

### review-achieves-verified-fault-tolerance
**Status:** OUT

The review pipeline's scoped mutation-safe operation combined with uniform fail-safe output achieves verified fault tolerance across all failure modes — batch failures, missing antecedent references, and malformed LLM responses.

**Depends on:** [review-output-is-uniform-and-fail-safe](safe.md#review-output-is-uniform-and-fail-safe), [review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe)

### review-and-contradictions-catch-orthogonal-errors
**Status:** IN

`review-beliefs` catches invalid reasoning within individual derivation steps (over-generalization, missing bridges, strength escalation) while `contradictions` catches incompatible facts across independently valid beliefs (absolute claims vs. documented exceptions) — the two commands are complementary with different cascade profiles (review: high cascade at depth, contradictions: low cascade at leaves).


### review-batch-failure-is-silent-skip
**Status:** IN

When an LLM call fails for a review batch, the error is logged to stderr but the batch is skipped with no indication in the returned results; callers cannot distinguish "skipped due to error" from "no problems found."

**Supports:** [batch-fault-isolation-is-universal-across-llm-operations](llm.md#batch-fault-isolation-is-universal-across-llm-operations), [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant)

### review-evaluates-direct-antecedents-only
**Status:** IN

`review-beliefs` presents each derived belief with only its direct antecedents to the LLM reviewer, not the transitive chain — producing occasional sufficiency false positives when support exists one level deeper, which must be triaged manually.


### review-format-handles-missing-antecedents
**Status:** IN

`format_belief_for_review` renders `"(not found in network)"` for antecedent IDs that don't exist in the nodes dict rather than crashing, and returns an empty string for a nonexistent belief ID.

**Supports:** [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant)

### review-has-no-storage-dependency
**Status:** IN

The review module operates entirely on an in-memory `nodes` dict (from `export_network()`) and never reads from or writes to the database directly.

**Supports:** [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant)

### review-is-read-only-and-fault-tolerant
**Status:** OUT

The review module operates entirely on in-memory snapshots with no storage dependency, handles missing antecedent references with placeholder text rather than exceptions, and silently skips failed LLM batches — achieving fault-tolerant read-only operation across all failure modes.

**Depends on:** [review-batch-failure-is-silent-skip](other.md#review-batch-failure-is-silent-skip), [review-format-handles-missing-antecedents](other.md#review-format-handles-missing-antecedents), [review-has-no-storage-dependency](other.md#review-has-no-storage-dependency)
**Supports:** [all-belief-inspection-is-non-mutating-and-fault-tolerant](other.md#all-belief-inspection-is-non-mutating-and-fault-tolerant), [dual-quality-enforcement-spans-automated-and-explicit](spans.md#dual-quality-enforcement-spans-automated-and-explicit), [llm-integration-fails-softly-across-modules](llm.md#llm-integration-fails-softly-across-modules)

### review-parse-requires-json-array
**Status:** IN

`parse_review_response` only accepts JSON arrays; a bare JSON object `{...}` is treated as unparseable and returns an empty list — the LLM must return a list even for single-item reviews.

**Supports:** [review-output-is-uniform-and-fail-safe](safe.md#review-output-is-uniform-and-fail-safe)

### review-result-schema-is-normalized
**Status:** IN

Every result dict returned by `parse_review_response` is guaranteed to have exactly six keys (`id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`) regardless of what the LLM returned, via normalization with safe defaults.

**Supports:** [review-output-is-uniform-and-fail-safe](safe.md#review-output-is-uniform-and-fail-safe)

### review-skips-premises
**Status:** IN

`review_beliefs` only sends beliefs with at least one justification to the LLM; premise nodes (empty justifications list) are excluded from review entirely.


### rewrite-dependents-updates-both-antecedents-and-outlists
**Status:** IN

`_rewrite_dependents(net, old, new)` in `api.py` rewrites justification references and dependent sets for both antecedent and outlist occurrences of the old node ID, not just one or the other


### run-cli-helper-catches-systemexit
**Status:** IN

The `run_cli` test harness intercepts `SystemExit` to extract exit codes, preventing argparse errors or explicit `sys.exit()` calls from terminating the test process.


### safety-and-uniformity-are-co-derived
**Status:** OUT

Dialectical safety (deterministic evaluation of irreversible premise transformations) and edge-case uniformity (consistent handling of vacuous premises, asymmetric absence, and empty antecedents) are independently derived from the same shared root — semantic minimality with operational determinism — revealing them as two faces of a single architectural property rather than independent design achievements.

**Depends on:** [determinism-enables-safe-dialectical-extension](safe.md#determinism-enables-safe-dialectical-extension), [edge-case-uniformity-follows-from-minimality](other.md#edge-case-uniformity-follows-from-minimality)
**Supports:** [all-safety-dimensions-converge](other.md#all-safety-dimensions-converge), [safety-integrity-and-uniformity-converge](other.md#safety-integrity-and-uniformity-converge)

### safety-integrity-and-uniformity-converge
**Status:** OUT

Three independently-established properties — boundary-agnostic integrity (internal/external indifference), dialectical safety (deterministic evaluation of irreversible transformations), and edge-case uniformity (consistent handling of vacuous and asymmetric cases) — converge to a single architectural invariant because all three derive from the same minimal evaluation rules applied uniformly.

**Depends on:** [integrity-is-boundary-and-source-agnostic](source.md#integrity-is-boundary-and-source-agnostic), [safety-and-uniformity-are-co-derived](other.md#safety-and-uniformity-are-co-derived)

### safety-is-enforced-across-all-layers-and-backends
**Status:** IN

Safety is enforced uniformly across both the architectural dimension (clean layer boundaries with atomic isolated mutations) and the storage dimension (equivalent guarantees through backend-appropriate mechanisms in SQLite and PostgreSQL) — no safety property depends on a specific backend or architectural layer.

**Depends on:** [architecture-enforces-structural-and-operational-safety](other.md#architecture-enforces-structural-and-operational-safety), [storage-layer-is-backend-agnostic-and-safe](safe.md#storage-layer-is-backend-agnostic-and-safe)
**Supports:** [backend-agnostic-operational-assurance](other.md#backend-agnostic-operational-assurance), [complete-architecture-is-backend-portable](complete.md#complete-architecture-is-backend-portable), [defeat-reversals-are-lifecycle-governed-across-all-backends](lifecycle.md#defeat-reversals-are-lifecycle-governed-across-all-backends), [initialization-is-safe-and-path-independent-across-backends](safe.md#initialization-is-safe-and-path-independent-across-backends), [operational-profile-spans-all-backends](spans.md#operational-profile-spans-all-backends), [operational-safety-is-universal-and-condition-independent](other.md#operational-safety-is-universal-and-condition-independent), [verified-correctness-extends-to-all-backends](other.md#verified-correctness-extends-to-all-backends)

### search-falls-back-to-substring
**Status:** IN

`api.search()` tries FTS5 full-text search first, then falls back to substring matching if FTS5 tables don't exist or error; it always produces results if substring matches exist.

**Supports:** [search-is-resilient-across-index-states](other.md#search-is-resilient-across-index-states)

### search-has-four-output-formats
**Status:** IN

The `search` function supports four output formats: markdown, json, minimal, and compact — selected by the caller to match the consumption context (human, API, LLM prompt, compact view).


### search-is-resilient-across-index-states
**Status:** IN

Search operates correctly regardless of FTS5 index availability: the index is derived (rebuilt from scratch on every save) so stale indexes are self-healing, and search falls back to substring matching when FTS tables don't exist or error

**Depends on:** [search-falls-back-to-substring](other.md#search-falls-back-to-substring), [storage-fts-is-derived-index](other.md#storage-fts-is-derived-index)
**Supports:** [all-query-operations-degrade-gracefully](other.md#all-query-operations-degrade-gracefully), [all-read-paths-are-deterministic-and-resilient](deterministic.md#all-read-paths-are-deterministic-and-resilient)

### search-relaxation-capped-at-50-queries
**Status:** IN

FTS progressive relaxation drops terms via `combinations()` (largest subsets first) until results appear, capped at 50 total relaxation queries to prevent combinatorial explosion on long search strings.


### search-uses-fts5-with-substring-fallback
**Status:** IN

`api.search()` tries FTS5 first (`_fts_search`), falls back to substring matching (`_substring_search`), then expands results with 1-hop neighbors from the dependency graph.


### semantic-contradiction-reuses-cluster-infrastructure
**Status:** IN

Semantic contradiction detection delegates embedding and clustering to the existing `list_clusters()` from `reasons_lib/cluster.py` — the same infrastructure that serves deduplication also serves contradiction detection, with no duplicate embedding/clustering implementation.


### semantic-contradiction-skips-singleton-clusters
**Status:** IN

Single-belief clusters are skipped during semantic contradiction detection (no contradiction possible within one belief), and clusters exceeding `CONTRADICTION_BATCH_SIZE` are sub-batched within the cluster boundary.


### semantic-minimality-with-operational-determinism
**Status:** IN

The system unifies semantic minimality (all non-monotonic features and truth semantics derive from uniform outlist/disjunction primitives) with operational determinism (all operations terminate predictably via BFS fixpoint with conservative failure semantics), yielding a small trusted kernel that powers all reasoning.

**Depends on:** [reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible), [system-semantics-are-minimal-and-complete](system.md#system-semantics-are-minimal-and-complete)
**Supports:** [completeness-determinism-and-minimality-are-unified](other.md#completeness-determinism-and-minimality-are-unified), [context-agnosticism-follows-from-minimality](other.md#context-agnosticism-follows-from-minimality), [determinism-enables-safe-dialectical-extension](safe.md#determinism-enables-safe-dialectical-extension), [edge-case-uniformity-follows-from-minimality](other.md#edge-case-uniformity-follows-from-minimality), [evaluation-purity-grounds-agnosticism-and-minimality](other.md#evaluation-purity-grounds-agnosticism-and-minimality), [minimality-produces-uniformity-and-determinism](other.md#minimality-produces-uniformity-and-determinism)

### single-cli-entry-point
**Status:** IN

The only registered console script is `reasons`, pointing to `reasons_lib.cli:main`; all subcommands are dispatched internally


### single-node-api-raises-permissionerror
**Status:** IN

API functions that target a single node by ID (`show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`) raise `PermissionError` when the caller lacks clearance; collection endpoints silently filter instead.

**Supports:** [api-enforces-typed-preconditions](other.md#api-enforces-typed-preconditions)

### sl-conjunction-any-disjunction
**Status:** IN

Covered by existing `sl-justification-semantics`, `truth-is-disjunctive-over-conjunctive-rules`, and `node-in-if-any-justification-valid`


### sl-outlist-asymmetry
**Status:** IN

Missing antecedents invalidate a justification, but missing outlist nodes do not — this asymmetry enables "believe X unless Y" where Y may not yet exist in the network

**Supports:** [missing-nodes-have-asymmetric-fail-semantics](other.md#missing-nodes-have-asymmetric-fail-semantics)

### sl-param-is-comma-separated-node-ids
**Status:** IN

`add_node()` encodes SL justifications as comma-separated node IDs in the `sl=` parameter and outlist nodes in `unless=`, mirroring the TMS (SL, OL) formalism directly in the API signature.


### stale-maps-to-out
**Status:** IN

STALE status in `beliefs.md` is mapped to OUT (retracted) in the network; there is no distinct STALE state in the TMS data model.


### stale-result-reasons
**Status:** IN

`check_stale` uses two distinct reason codes: `"content_changed"` when the source file exists but its hash differs, and `"source_deleted"` when the file is gone from disk.


### staleness-checking-is-comprehensive
**Status:** IN

Staleness checking detects all nodes whose source material has changed on disk

**Depends on:** [check-stale-exits-nonzero](other.md#check-stale-exits-nonzero), [check-stale-requires-both-source-fields](source.md#check-stale-requires-both-source-fields)
**Supports:** [external-interface-is-bidirectionally-bounded](external.md#external-interface-is-bidirectionally-bounded), [read-and-write-paths-are-both-reliable](other.md#read-and-write-paths-are-both-reliable)

### staleness-gate-catches-all-drift
**Status:** IN

The staleness CI gate detects all forms of source material drift without false negatives.

**Depends on:** [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)
**Supports:** [automated-sync-achieves-full-lifecycle-coverage](lifecycle.md#automated-sync-achieves-full-lifecycle-coverage), [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [lifecycle-management-is-gapless](lifecycle.md#lifecycle-management-is-gapless), [source-integrity-spans-hashing-through-detection](source.md#source-integrity-spans-hashing-through-detection)

### staleness-information-survives-binary-truth-model
**Status:** IN

Despite the TMS using binary IN/OUT truth values with no distinct STALE state, staleness information is preserved end-to-end: stale beliefs are mapped to OUT on import with stale_reason metadata, and the compact output surfaces this metadata for OUT nodes — so downstream consumers can distinguish intentional retractions from staleness-driven ones.

**Depends on:** [compact-surfaces-stale-reason-metadata](compact.md#compact-surfaces-stale-reason-metadata), [stale-maps-to-out-on-import](import.md#stale-maps-to-out-on-import)
**Supports:** [staleness-is-surfaced-despite-binary-truth-model](other.md#staleness-is-surfaced-despite-binary-truth-model)

### staleness-is-conservative-ci-gate
**Status:** IN

Staleness checking is designed as a safe CI gate: it never mutates state, only checks IN nodes, requires both source fields, and exits nonzero to fail the pipeline

**Depends on:** [check-stale-exits-nonzero](other.md#check-stale-exits-nonzero), [check-stale-is-read-only](other.md#check-stale-is-read-only), [check-stale-requires-both-source-fields](source.md#check-stale-requires-both-source-fields), [check-stale-skips-out-nodes](other.md#check-stale-skips-out-nodes)
**Supports:** [all-belief-inspection-is-non-mutating-and-fault-tolerant](other.md#all-belief-inspection-is-non-mutating-and-fault-tolerant), [data-integrity-spans-architecture](spans.md#data-integrity-spans-architecture), [dual-quality-gates-are-complementary-and-non-mutating](other.md#dual-quality-gates-are-complementary-and-non-mutating), [external-belief-lifecycle-is-complete](external.md#external-belief-lifecycle-is-complete), [lifecycle-awareness-spans-checking-and-propagation](lifecycle.md#lifecycle-awareness-spans-checking-and-propagation), [staleness-gate-catches-all-drift](other.md#staleness-gate-catches-all-drift), [staleness-output-is-ci-pipeline-ready](other.md#staleness-output-is-ci-pipeline-ready)

### staleness-is-surfaced-despite-binary-truth-model
**Status:** IN

Staleness information not only survives the binary IN/OUT truth model (via metadata-based preservation through import and compact surfacing) but also emerges as deterministic, uniformly-structured, machine-parseable CI output with conservative non-mutating semantics — no information is lost between the TMS representation and the external consumer.

**Depends on:** [staleness-information-survives-binary-truth-model](other.md#staleness-information-survives-binary-truth-model), [staleness-output-is-ci-pipeline-ready](other.md#staleness-output-is-ci-pipeline-ready)
**Supports:** [metadata-enables-lifecycle-governance-beyond-binary-truth](lifecycle.md#metadata-enables-lifecycle-governance-beyond-binary-truth)

### staleness-output-is-ci-pipeline-ready
**Status:** IN

Staleness checking produces deterministic, uniformly-structured, machine-parseable output with conservative non-mutating semantics and nonzero exit codes, making it directly consumable by automated CI pipelines without wrapper scripts

**Depends on:** [check-stale-output-is-deterministic-and-structured](deterministic.md#check-stale-output-is-deterministic-and-structured), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)
**Supports:** [all-read-paths-are-deterministic-and-resilient](deterministic.md#all-read-paths-are-deterministic-and-resilient), [staleness-is-surfaced-despite-binary-truth-model](other.md#staleness-is-surfaced-despite-binary-truth-model), [system-output-is-complete-bounded-and-ci-ready](system.md#system-output-is-complete-bounded-and-ci-ready)

### staleness-results-are-dicts-not-exceptions
**Status:** IN

Both `check_stale` and `hash_sources` report problems (missing files, changed content, deleted sources) as list-of-dict return values, never raising exceptions — designed for batch audit operations that report all problems rather than aborting on the first.


### staleness-uses-full-sha256
**Status:** IN

Staleness detection compares full 64-character SHA-256 hex digests via exact string equality, not truncated prefixes or fuzzy comparison.

**Supports:** [source-tracking-is-collision-resistant-and-safe](source.md#source-tracking-is-collision-resistant-and-safe)

### startup-performance-uses-lazy-loading
**Status:** IN

Both the API and CLI layers defer importing heavy modules (derive, compact, ask, asyncio, Storage) to function bodies rather than module top-level, minimizing import-time overhead for CLI responsiveness.

**Depends on:** [api-uses-lazy-imports](other.md#api-uses-lazy-imports), [cli-uses-lazy-imports-for-heavy-modules](other.md#cli-uses-lazy-imports-for-heavy-modules)
**Supports:** [system-resource-footprint-is-minimal-at-all-phases](system.md#system-resource-footprint-is-minimal-at-all-phases)

### sticky-retraction-survives-recompute-all
**Status:** IN

A node with `_retracted` metadata stays OUT even when `recompute_all()` re-evaluates the entire network — sticky retraction is a durable pin that only `assert_node` can clear


### storage-derives-counter-from-existing-nogoods
**Status:** IN

When loading a database without the `network_meta` table (pre-fix schema), `Storage.load()` derives `_next_nogood_id` from the max ID of existing nogoods rather than defaulting to 1, enabling backward-compatible upgrades


### storage-fts-is-derived-index
**Status:** IN

The FTS5 full-text search index is rebuilt from scratch during every `save()`; it is a derived index, never the source of truth.

**Supports:** [search-is-resilient-across-index-states](other.md#search-is-resilient-across-index-states), [storage-optimizes-concurrent-access-and-search](other.md#storage-optimizes-concurrent-access-and-search)

### storage-fts-rebuilt-on-every-save
**Status:** IN

The `nodes_fts` full-text index is deleted and rebuilt from scratch on each `save()`, guaranteeing consistency with the `nodes` table at the cost of full-reindex overhead.


### storage-handles-schema-evolution-via-try-except
**Status:** IN

Missing tables from older database versions (`repos`, `network_meta`) are handled by swallowing exceptions during `load()` rather than formal migrations — a backward-compatibility substitute that silently degrades.

**Supports:** [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)

### storage-is-fully-production-grade-across-backends
**Status:** IN

Both storage backends achieve fully production-grade operation — concurrent access optimization (WAL mode, derived FTS5 indexes), equivalent safety guarantees (atomic isolated mutations through backend-appropriate mechanisms), and multi-tenant isolation — when both backends provide complete API coverage.

**Depends on:** [storage-layer-is-backend-agnostic-and-safe](safe.md#storage-layer-is-backend-agnostic-and-safe), [storage-optimizes-concurrent-access-and-search](other.md#storage-optimizes-concurrent-access-and-search)

### storage-lists-as-json-columns
**Status:** IN

Antecedents, outlist, nogood node sets, and metadata dicts are stored as JSON text columns rather than normalized join tables, simplifying the schema at the cost of individual field queryability.


### storage-load-bypasses-add-node
**Status:** IN

`load()` assigns nodes directly to `network.nodes` and calls `_rebuild_dependents()` afterward, deliberately skipping `add_node()` to avoid triggering truth maintenance propagation during state restoration.


### storage-load-bypasses-propagation
**Status:** IN

`load()` constructs nodes directly into `network.nodes` rather than calling `add_node`, so truth maintenance propagation does not fire during deserialization.

**Supports:** [bootstrap-bypasses-incremental-propagation](other.md#bootstrap-bypasses-incremental-propagation), [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental)

### storage-no-partial-load
**Status:** IN

`Storage.load()` either returns a complete `Network` with all state or fails; there is no streaming, lazy-loading, or partial deserialization.


### storage-old-schema-compat
**Status:** IN

`load()` tolerates missing `network_meta` and `repos` tables via silent `try/except`, supporting databases created before those tables were added; `next_nogood_id` is derived from existing IDs as a fallback.


### storage-optimizes-concurrent-access-and-search
**Status:** IN

The storage layer optimizes for both concurrent access (WAL mode enables non-blocking reads during writes) and full-text search (derived FTS5 index rebuilt from scratch on every save guarantees consistency), making the persistence layer production-ready for multi-reader workloads with search capability.

**Depends on:** [storage-fts-is-derived-index](other.md#storage-fts-is-derived-index), [storage-uses-wal-mode](other.md#storage-uses-wal-mode)
**Supports:** [storage-is-fully-production-grade-across-backends](other.md#storage-is-fully-production-grade-across-backends)

### storage-round-trip-preserves-dependents
**Status:** IN

Covered by existing belief `persistence-round-trip-is-lossless` — dependents are part of the persisted state


### storage-save-is-atomic-snapshot
**Status:** IN

Duplicate of existing `storage-save-is-full-replace` and `mutation-pipeline-is-atomic-snapshot`


### storage-save-is-full-replace
**Status:** IN

`Storage.save()` deletes all rows from every table before re-inserting the entire network; there is no incremental or differential update path.

**Supports:** [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental)

### storage-trusts-stored-truth-values
**Status:** IN

`load()` trusts the stored `truth_value` without re-running propagation, making the database the source of truth for node status.

**Supports:** [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental)

### storage-uses-json-columns-for-lists
**Status:** IN

Antecedent IDs, outlist IDs, nogood node sets, and node metadata are stored as JSON text columns rather than junction tables — acceptable because the code always loads the full network, never queries relationships via SQL joins.


### storage-uses-wal-mode
**Status:** IN

SQLite connections enable WAL mode on initialization, allowing concurrent readers without blocking writes.

**Supports:** [storage-optimizes-concurrent-access-and-search](other.md#storage-optimizes-concurrent-access-and-search)

### supersession-is-reversible
**Status:** IN

`supersede()` adds the new node's ID to the old node's outlist rather than deleting the old node; retracting the new belief automatically restores the old one through normal propagation

**Supports:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [outlist-is-universal-defeat-mechanism](other.md#outlist-is-universal-defeat-mechanism), [supersession-is-reversible-and-view-consistent](other.md#supersession-is-reversible-and-view-consistent)

### supersession-is-reversible-and-view-consistent
**Status:** IN

Supersession is both mechanically reversible (implemented via outlist, so retracting the superseder restores the original node's truth value) and view-consistent (superseded nodes are excluded from gated belief lists even if they retain active blockers), making it a first-class lifecycle operation rather than just a truth-value toggle.

**Depends on:** [api-superseded-nodes-excluded-from-gated](other.md#api-superseded-nodes-excluded-from-gated), [supersession-is-reversible](other.md#supersession-is-reversible)
**Supports:** [belief-replacement-is-topology-safe-and-view-consistent](topology.md#belief-replacement-is-topology-safe-and-view-consistent)

### sync-is-remote-wins
**Status:** IN

`_sync_claims` implements remote-wins reconciliation: remote text/metadata overwrites local, beliefs removed from remote are retracted locally, and beliefs remotely IN but locally OUT are re-asserted

**Supports:** [import-sync-has-dual-reconciliation-modes](import.md#import-sync-has-dual-reconciliation-modes)

### sync-preserves-cascade-wiring
**Status:** IN

After `sync_agent` runs, the `agent:inactive` outlist entries on beliefs are preserved, so retracting `agent:active` still cascades all agent beliefs to OUT


### sync-returns-structured-diff-counts
**Status:** IN

`sync_agent` returns a dict with `beliefs_added`, `beliefs_updated`, `beliefs_unchanged`, and `beliefs_removed` counts that accurately reflect the diff between remote file and local state — no double-counting across sync cycles.


### tag-inheritance-is-transitive-union
**Status:** IN

Derived nodes inherit the sorted union of all ancestor `access_tags`, propagating transitively through arbitrarily long justification chains including diamond dependencies.

**Supports:** [access-control-is-transitive-subset-gated](other.md#access-control-is-transitive-subset-gated)

### tag-propagation-is-dynamic
**Status:** IN

Adding a justification to an existing node triggers access tag recomputation that cascades through all downstream dependents via BFS, not just at node creation time.


### test-network-covers-all-propagation-paths
**Status:** IN

Test coverage claim, not a codebase invariant — what the test suite covers can change without affecting production behavior


### tests-verify-via-output-headers
**Status:** IN

Test methodology detail (how assertions parse output), not a codebase architectural invariant


### three-layer-architecture
**Status:** IN

The codebase is a three-layer stack: data model (`__init__.py`), TMS engine (`network.py`), and persistence (`storage.py`), with `api.py` providing functional API and `cli.py` as a thin argparse wrapper.

**Supports:** [three-layer-stack-has-clean-boundaries](other.md#three-layer-stack-has-clean-boundaries)

### three-layer-stack-has-clean-boundaries
**Status:** IN

The architecture enforces strict layer separation: pure data model at bottom, context-managed API with dict returns in the middle, and pure-formatter CLI at the top

**Depends on:** [api-uses-with-network-context-manager](other.md#api-uses-with-network-context-manager), [cli-is-pure-formatter](other.md#cli-is-pure-formatter), [init-is-pure-data-model](other.md#init-is-pure-data-model), [three-layer-architecture](other.md#three-layer-architecture)
**Supports:** [central-dependency-is-safely-contained](other.md#central-dependency-is-safely-contained), [data-integrity-spans-architecture](spans.md#data-integrity-spans-architecture)

### tms-handles-all-conditions-safely
**Status:** IN

The TMS core handles both normal operation (crash-free truth propagation via BFS with stop-on-unchanged) and exceptional conditions (deterministic contradiction resolution via backtracking with least-entrenched selection) through shared propagation infrastructure — no reachable execution state leads to undefined behavior.

**Depends on:** [contradiction-triggers-deterministic-resolution](deterministic.md#contradiction-triggers-deterministic-resolution), [tms-core-is-crash-safe](safe.md#tms-core-is-crash-safe)
**Supports:** [all-exceptions-are-safely-handled](other.md#all-exceptions-are-safely-handled), [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting)

### token-budgets-are-accurate-bidirectionally
**Status:** OUT

Token budget management is accurate in both directions: the compact module reliably constrains output size for context-limited consumers, while the derive pipeline correctly allocates input budgets per agent — ensuring resource-bounded operation across the entire LLM integration surface.

**Depends on:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size), [derive-budget-allocation-is-accurate](derive.md#derive-budget-allocation-is-accurate)
**Supports:** [information-pipeline-is-resource-governed-and-access-controlled](other.md#information-pipeline-is-resource-governed-and-access-controlled), [resource-management-supports-belief-currency](other.md#resource-management-supports-belief-currency)

### topo-sort-breaks-cycles
**Status:** IN

Duplicates existing belief `import-topo-sort-tolerates-cycles`.


### total-preservation-is-indefinitely-auditable
**Status:** OUT

Total invariant preservation — comprehensive in scope and self-sustaining through minimality — is accompanied by indefinite auditability: every invariant-preserving action across all time leaves traceable history without temporal degradation, meaning the system can prove its own correctness at any point.

**Depends on:** [indefinite-self-correction-is-fully-auditable](self.md#indefinite-self-correction-is-fully-auditable), [invariant-preservation-is-total-and-self-sustaining](self.md#invariant-preservation-is-total-and-self-sustaining)
**Supports:** [convergent-equilibria-are-documented-and-indefinitely-auditable](other.md#convergent-equilibria-are-documented-and-indefinitely-auditable)

### transaction-per-function
**Status:** IN

Every API function opens the database, does its work, and closes — no shared state, no connection pooling, no long-lived sessions; each invocation is fully independent.

**Supports:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations)

### transformations-converge-to-documented-equilibria
**Status:** OUT

All structural transformations — mode expansion, negation semantics, identity transformation — are evaluation-transparent (producing identical truth regardless of transformation path), and the system autonomously converges to equilibria that generate deterministic traceable artifacts, meaning any sequence of transformations reaches the same documented stable state.

**Depends on:** [all-transformations-are-evaluation-transparent](other.md#all-transformations-are-evaluation-transparent), [autonomous-convergence-produces-documented-equilibria](other.md#autonomous-convergence-produces-documented-equilibria)
**Supports:** [evaluation-traceability-persists-through-equilibria](other.md#evaluation-traceability-persists-through-equilibria)

### transformations-produce-governed-traceable-output
**Status:** OUT

Every structural transformation — mode expansion, negation semantics, identity transformation — is deterministic, traceable, and boundary-safe, and all transformation results flow through comprehensive self-sustaining output governance that normalizes, authorizes, and self-corrects delivery — creating an end-to-end governed pipeline from belief mutation through information delivery.

**Depends on:** [all-transformations-are-deterministic-traceable-and-boundary-safe](deterministic.md#all-transformations-are-deterministic-traceable-and-boundary-safe), [output-governance-is-comprehensive-and-self-sustaining](governance.md#output-governance-is-comprehensive-and-self-sustaining)

### truncated-hash-threshold-is-16-chars
**Status:** IN

A stored `source_hash` of exactly 16 characters that is a prefix of the current full SHA-256 hash is classified as `truncated_hash` (legacy format), not `content_changed`.


### truncated-hash-upgrade-opt-in
**Status:** IN

Prefix hash upgrade (16-char truncated SHA-256 to full digest) only occurs when `upgrade_hashes=True` is passed to `check_stale`; without it, truncated hashes produce a warning result with `reason="truncated_hash"`.


### trust-and-information-boundaries-are-comprehensively-enforced
**Status:** OUT

The system enforces comprehensive boundaries spanning both architecture and information flow: architectural trust boundaries through self-containment and defensive ingestion pipelines ensure no unvalidated data enters the network, while information boundaries through access-tag authorization, token-budget constraints, and bidirectional external surface control ensure no unauthorized or unbounded data leaves it

**Depends on:** [information-boundaries-are-controlled-at-all-levels](other.md#information-boundaries-are-controlled-at-all-levels), [trust-boundary-is-architecturally-enforced](other.md#trust-boundary-is-architecturally-enforced)
**Supports:** [autonomous-convergence-preserves-trust-boundaries](other.md#autonomous-convergence-preserves-trust-boundaries), [external-surface-is-verified-and-trust-bounded](external.md#external-surface-is-verified-and-trust-bounded), [operational-guarantees-span-safety-and-trust](other.md#operational-guarantees-span-safety-and-trust)

### trust-boundary-is-architecturally-enforced
**Status:** OUT

The system's trust boundary is architecturally enforced through complementary internal and external mechanisms: internal self-containment (zero external dependencies, clean three-layer boundaries) eliminates supply-chain and cross-layer attack surfaces, while defensive external containment (layered validation pipelines, namespace isolation, agent kill-switches) prevents untrusted input from corrupting internal state

**Depends on:** [architecture-is-self-contained-and-safely-layered](self.md#architecture-is-self-contained-and-safely-layered), [external-beliefs-defensively-contained](external.md#external-beliefs-defensively-contained)
**Supports:** [trust-and-information-boundaries-are-comprehensively-enforced](other.md#trust-and-information-boundaries-are-comprehensively-enforced), [trust-enforcement-is-structural-and-operationally-resilient](other.md#trust-enforcement-is-structural-and-operationally-resilient)

### trust-enforcement-is-structural-and-operationally-resilient
**Status:** OUT

System trust boundaries are enforced through two complementary mechanisms: structural containment (zero external dependencies, defensive ingestion pipelines, safe three-layer architecture) provides static trust guarantees, while format resilience at all external interfaces (parser fallbacks, schema evolution tolerance, hallucination filtering) provides dynamic trust that adapts to changing external formats without relaxing validation.

**Depends on:** [format-resilient-boundaries-enforce-validated-trust](other.md#format-resilient-boundaries-enforce-validated-trust), [trust-boundary-is-architecturally-enforced](other.md#trust-boundary-is-architecturally-enforced)
**Supports:** [trust-boundaries-are-self-maintaining](self.md#trust-boundaries-are-self-maintaining)

### trustworthiness-is-verifiable-through-observability
**Status:** OUT

The revision system's complete trustworthiness (verifiable soundness, end-to-end reliability, full auditability) is independently verifiable because the minimality-sustained maintenance loop provides complete observability — every self-correction and maintenance action leaves traceable evidence that can be inspected.

**Depends on:** [maintenance-loop-is-fully-observable](other.md#maintenance-loop-is-fully-observable), [revision-achieves-complete-trustworthiness](revision.md#revision-achieves-complete-trustworthiness)
**Supports:** [self-sustaining-invariants-are-independently-verifiable](self.md#self-sustaining-invariants-are-independently-verifiable)

### truth-evaluation-is-transformation-invariant
**Status:** OUT

Truth evaluation produces identical results regardless of both temporal context (when a justification was attached — at node creation vs. later addition) and structural transformation (premise → justified node via challenge) — all forms of node history and identity change are invisible to the evaluation function, making truth a pure function of current network state.

**Depends on:** [evaluation-is-uniformly-context-and-origin-agnostic](other.md#evaluation-is-uniformly-context-and-origin-agnostic), [identity-transformation-is-semantically-invisible](other.md#identity-transformation-is-semantically-invisible)
**Supports:** [any-mode-expansion-is-evaluation-invisible](other.md#any-mode-expansion-is-evaluation-invisible), [convergence-produces-evaluation-invariant-equilibria](other.md#convergence-produces-evaluation-invariant-equilibria), [negation-is-transparent-to-evaluation](other.md#negation-is-transparent-to-evaluation), [richer-revision-preserves-evaluation-invariance](revision.md#richer-revision-preserves-evaluation-invariance)

### truth-is-disjunctive-over-conjunctive-rules
**Status:** IN

A node's truth is a disjunction over justifications (any valid justification makes it IN), where each justification is a conjunction (all antecedents IN and all outlist OUT), and any-mode explicitly reifies OR semantics as per-premise justifications.

**Depends on:** [any-mode-creates-per-premise-justifications](other.md#any-mode-creates-per-premise-justifications), [node-in-if-any-justification-valid](justification.md#node-in-if-any-justification-valid), [sl-justification-semantics](justification.md#sl-justification-semantics)
**Supports:** [truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)

### truth-semantics-are-emergent-and-uniform
**Status:** IN

Truth maintenance semantics are fully emergent from simple uniform rules: premise behavior arises from empty justification lists, evaluation is pure and type-agnostic across SL/CP, and node truth is a clean disjunction-of-conjunctions — no special cases exist anywhere in the evaluation path.

**Depends on:** [justification-evaluation-is-uniform-and-pure](justification.md#justification-evaluation-is-uniform-and-pure), [premise-behavior-emerges-from-absence](other.md#premise-behavior-emerges-from-absence), [truth-is-disjunctive-over-conjunctive-rules](other.md#truth-is-disjunctive-over-conjunctive-rules)
**Supports:** [all-semantic-edge-cases-are-uniform](other.md#all-semantic-edge-cases-are-uniform), [dialectics-are-semantically-transparent](other.md#dialectics-are-semantically-transparent), [system-semantics-are-minimal-and-complete](system.md#system-semantics-are-minimal-and-complete)

### untagged-always-visible
**Status:** IN

Nodes with no `access_tags` key in metadata are never filtered by `visible_to` — they are treated as unconditionally public.


### untagged-nodes-always-visible
**Status:** IN

Nodes without `access_tags` metadata pass all `visible_to` filters unconditionally and are never hidden by access control.


### update-node-preserves-justifications
**Status:** IN

`api.update_node` modifies text and source metadata without altering the node's justification list or truth value.


### user-interface-is-verified-and-fault-tolerant
**Status:** OUT

The complete user-facing stack is both structurally verified (pure delegation with hermetic integration tests ensuring no business logic leaks into the CLI) and operationally resilient (every information flow path is fault-tolerant with graceful degradation and governed output) — users never encounter unverified logic or ungoverned failure modes.

**Depends on:** [all-information-flow-is-fault-tolerant-and-governed](other.md#all-information-flow-is-fault-tolerant-and-governed), [full-user-stack-is-verified-atomic-delegation](other.md#full-user-stack-is-verified-atomic-delegation)
**Supports:** [external-surface-is-verified-and-trust-bounded](external.md#external-surface-is-verified-and-trust-bounded), [verified-interface-controls-bidirectional-flow](other.md#verified-interface-controls-bidirectional-flow)

### validate-proposals-rejects-jaccard-similar-to-out
**Status:** IN

`validate_proposals` skips any proposal whose tokenized ID has >= 0.5 Jaccard similarity to an existing OUT belief, returning "similar to retracted" in the skip reason — preventing re-derivation of retracted beliefs under variant names


### verified-correctness-extends-to-all-backends
**Status:** OUT

Verified production correctness — spanning all belief origins with deterministic state trajectories and full maintenance loop observability for indefinite independent verification — extends identically across both SQLite and PostgreSQL storage backends with no backend-specific safety gaps.

**Depends on:** [safety-is-enforced-across-all-layers-and-backends](other.md#safety-is-enforced-across-all-layers-and-backends), [verified-correctness-is-indefinitely-observable](other.md#verified-correctness-is-indefinitely-observable)

### verified-correctness-has-indefinitely-auditable-equilibria
**Status:** OUT

Verified production correctness — observable and permanently documented across all belief origins with deterministic state trajectories — converges to equilibria that are themselves trajectory-documented and indefinitely auditable, creating a self-reinforcing documentation loop where correctness verification and equilibrium auditability share the same permanent evidence base.

**Depends on:** [convergent-equilibria-are-documented-and-indefinitely-auditable](other.md#convergent-equilibria-are-documented-and-indefinitely-auditable), [verified-correctness-is-observable-and-permanently-documented](other.md#verified-correctness-is-observable-and-permanently-documented)

### verified-correctness-is-indefinitely-observable
**Status:** OUT

Verified production correctness — spanning all belief origins with deterministic state trajectories — is both independently observable through the fully characterized maintenance loop and indefinitely sustainable through minimality's fixed-point self-maintenance, ensuring correctness can be verified at any future point without temporal degradation of either the correctness or the ability to observe it.

**Depends on:** [fully-characterized-loop-sustains-indefinitely](other.md#fully-characterized-loop-sustains-indefinitely), [verified-correctness-is-independently-observable](other.md#verified-correctness-is-independently-observable)
**Supports:** [verified-correctness-extends-to-all-backends](other.md#verified-correctness-extends-to-all-backends)

### verified-correctness-is-independently-observable
**Status:** OUT

Production correctness verified across all belief origins is independently observable through the fully characterized maintenance loop — every correctness claim can be audited by inspecting the same deterministic traceable history that the maintenance loop produces, without trusting the system's self-assessment.

**Depends on:** [maintenance-loop-is-fully-observable](other.md#maintenance-loop-is-fully-observable), [verified-production-correctness-spans-all-origins](spans.md#verified-production-correctness-spans-all-origins)
**Supports:** [verified-correctness-is-indefinitely-observable](other.md#verified-correctness-is-indefinitely-observable), [verified-correctness-is-observable-and-permanently-documented](other.md#verified-correctness-is-observable-and-permanently-documented)

### verified-correctness-is-observable-and-permanently-documented
**Status:** OUT

Verified production correctness spanning all belief origins is both independently observable (every correction is visible through the fully characterized maintenance loop without requiring trust in the system's self-reports) and permanently documented (every state change produces comprehensive artifacts that survive persistence boundaries and format evolution) — correctness is not merely achieved but externally auditable with durable evidence.

**Depends on:** [verified-correctness-is-independently-observable](other.md#verified-correctness-is-independently-observable), [verified-correctness-is-permanently-documented](other.md#verified-correctness-is-permanently-documented)
**Supports:** [verified-correctness-has-indefinitely-auditable-equilibria](other.md#verified-correctness-has-indefinitely-auditable-equilibria)

### verified-correctness-is-permanently-documented
**Status:** OUT

Verified production correctness spanning all belief origins is backed by a permanent, comprehensive audit trail — the system not only achieves correct state trajectories across all provenance boundaries, but permanently documents every self-correction that maintained that correctness, with no temporal degradation of either the guarantees or their documentation.

**Depends on:** [self-correction-audit-trail-is-permanent-and-comprehensive](self.md#self-correction-audit-trail-is-permanent-and-comprehensive), [verified-production-correctness-spans-all-origins](spans.md#verified-production-correctness-spans-all-origins)
**Supports:** [verified-correctness-is-observable-and-permanently-documented](other.md#verified-correctness-is-observable-and-permanently-documented)

### verified-interface-controls-bidirectional-flow
**Status:** OUT

All information flowing through the system's verified, fault-tolerant user interface is controlled in both directions: inbound through production-hardened LLM integration with bounded execution and fail-soft semantics, outbound through deterministic authorized access-controlled output — the verified interface serves as a trustworthy gateway that neither admits uncontrolled inputs nor emits unauthorized outputs.

**Depends on:** [information-flow-is-controlled-in-both-directions](other.md#information-flow-is-controlled-in-both-directions), [user-interface-is-verified-and-fault-tolerant](other.md#user-interface-is-verified-and-fault-tolerant)
**Supports:** [external-surface-is-verified-trust-bounded-and-bidirectionally-controlled](external.md#external-surface-is-verified-trust-bounded-and-bidirectionally-controlled)

### verified-mutation-correctness-across-boundaries
**Status:** OUT

Every mutation source produces fully correct persisted state that preserves boundary-agnostic integrity — not just safe operation, but verified output correctness across internal/external boundaries and all source types — only when implementation-level defects in propagation and budget allocation are resolved.

**Depends on:** [all-mutation-sources-are-safe-and-uniform](safe.md#all-mutation-sources-are-safe-and-uniform), [integrity-is-boundary-and-source-agnostic](source.md#integrity-is-boundary-and-source-agnostic)

### verify-dependents-is-read-only
**Status:** IN

`Network.verify_dependents()` never modifies the dependents index; it returns a list of human-readable error strings without side effects


### verify-dependents-is-readonly
**Status:** IN

`verify_dependents()` never modifies `node.dependents`; it only reads and reports discrepancies as a list of human-readable strings containing `"extra"` or `"missing"`.


### version-is-manual
**Status:** IN

The version is statically declared in `pyproject.toml` with no dynamic version plugin, `__version__` import, or SCM-based versioning — it must be bumped manually.


### visibility-is-subset-check
**Status:** IN

`_is_visible` requires ALL of a node's `access_tags` to be present in the caller's `visible_to` list (subset check, not intersection); nodes with no tags are always visible.


### visible-to-is-optional-filter
**Status:** IN

`_parse_visible_to` returns `None` when `--visible-to` is absent, which the API interprets as "no access restriction"; it never defaults to an empty list.


### visible-to-superset-semantics
**Status:** IN

A node with `access_tags: ["finance", "hr"]` is only visible to callers whose `visible_to` list contains both tags (superset/subset check, not intersection).


### warning-log-contract-action-target-value
**Status:** IN

Dangling-dependent warnings in `net.log` have `action="warn"`, a `target` field with the ghost node ID, and a `value` string containing both `"dangling"` and the parent node ID.


### warning-log-schema-stable
**Status:** IN

Dangling-dependent warnings use the dict schema `{action: "warn", target: <ghost_id>, value: <str containing "dangling" and parent_id>, timestamp: <str>}` and tests enforce all four fields.


### write-false-prevents-persistence
**Status:** IN

Functions using `_with_network(write=False)` can mutate the in-memory network (as `what_if_retract` does) but changes are never saved to SQLite; write-or-not is declared upfront and never conditional.

**Supports:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [both-backends-support-safe-hypothetical-reasoning](safe.md#both-backends-support-safe-hypothetical-reasoning)

### zero-runtime-dependencies
**Status:** IN

`ftl-reasons` declares `dependencies = []` in pyproject.toml; the entire `reasons_lib` package runs on Python's standard library alone (sqlite3, json, argparse); PostgreSQL support is opt-in via the `pg` extra

**Supports:** [project-has-zero-external-coupling](external.md#project-has-zero-external-coupling)

### zero-runtime-deps
**Status:** IN

`ftl-reasons` has zero runtime dependencies; the entire library runs on Python's stdlib alone (SQLite via `sqlite3`, dataclasses, etc.).

