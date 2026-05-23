# Proposed Beliefs

Edit each entry: change `[ACCEPT/REJECT]` to `[ACCEPT]` or `[REJECT]`.
Then run: `code-expert accept-beliefs`

---

**Generated:** 2026-04-23
**Source:** 17 entries from entries/
**Model:** claude

No existing beliefs. Here are the proposed beliefs extracted from the entries:

---

## From `reasons_lib/__init__.py`

### [REJECT] node-in-if-any-justification-valid
A Node is IN when at least one of its Justifications is valid; a Justification is valid when all antecedents are IN and all outlist nodes are OUT (disjunctive over justifications, conjunctive within each).
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: same claim as proposed sl-justification-semantics with less precision

### [REJECT] premises-have-no-justifications
A premise node is represented by an empty `justifications` list and defaults to `truth_value="IN"`; the system treats the empty-justifications case as a special unconditional belief.
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: split across existing challenge-destroys-premise-identity and proposed premise-defaults-to-in

### [ACCEPT] init-is-pure-data-model
`reasons_lib/__init__.py` contains only dataclass definitions (`Node`, `Justification`, `Nogood`) with no behavior, validation, or I/O; it imports nothing from the project and sits at the bottom of the import graph.
- Source: entries/2026/04/23/reasons_lib-__init__.md

### [REJECT] outlist-enables-non-monotonic-reasoning
The `outlist` field on `Justification` allows beliefs to be retracted when a defeating node becomes IN — this is the core non-monotonic mechanism powering `supersede`, `challenge`, and default-logic patterns.
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: covered by existing absence-and-outlist-form-complete-negative-semantics and challenge-is-outlist-injection

### [REJECT] dependents-is-manual-reverse-index
`Node.dependents` is a denormalized reverse pointer set that must be kept in sync by external code (primarily `network.py`); nothing in the data model enforces consistency.
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: already exists as `dependents-is-manual-reverse-index` [OUT]

---

## From `reasons_lib/api.py`

### [REJECT] api-functions-return-dicts
Every public API function returns a `dict` (or `str` for markdown/compact), never a `Network` or `Node` object, ensuring JSON-serializability at the boundary for CLI, HTTP, and tool-call consumers.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: already exists as `api-functions-return-dicts` [IN]

### [ACCEPT] write-false-prevents-persistence
Functions using `_with_network(write=False)` can mutate the in-memory network (as `what_if_retract` does) but changes are never saved to SQLite; write-or-not is declared upfront and never conditional.
- Source: entries/2026/04/23/reasons_lib-api.md

### [REJECT] namespace-active-premise-invariant
When `namespace` is set, `add_node` auto-creates a `{namespace}:active` premise node and wires it as an antecedent into every namespaced justification; retracting that single premise cascades OUT every belief from that namespace.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: covered by `active-inactive-relay-pair`, `active-not-in-antecedents`, `agent-isolation-through-namespace-and-relay`, and `import-agent-namespace-prefix`

### [REJECT] any-mode-creates-per-premise-justifications
When `any_mode=True` and multiple antecedents are given, each antecedent gets its own SL justification (OR semantics: node is IN if *any* antecedent is IN), rather than the default single multi-antecedent justification (AND semantics).
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: already exists as `any-mode-creates-per-premise-justifications` [IN]

### [REJECT] transaction-per-function
Every API function opens the database, does its work, and closes — no shared state, no connection pooling, no long-lived sessions; each invocation is fully independent.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: captured by `api-layer-ensures-atomic-isolated-mutations` ("per-function storage access") and `api-uses-with-network-context-manager`

### [REJECT] colon-means-already-namespaced
`_resolve_namespace` treats a colon in a node ID as "already namespaced" and never double-prefixes; this is the convention for cross-namespace references.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: already exists as `colon-means-already-namespaced` [IN]

---

## From `reasons_lib/check_stale.py`

### [REJECT] check-stale-is-read-only
`check_stale` never mutates the network; it returns a list of stale-node dicts and leaves all nodes unchanged — staleness detection is separated from staleness resolution.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Rejected: duplicate: already exists as `check-stale-is-read-only` [IN]

### [REJECT] check-stale-skips-out-nodes
Only nodes with `truth_value == "IN"` are checked for staleness; retracted (OUT) nodes are ignored even if their source file has changed.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Rejected: duplicate: already exists as `check-stale-skips-out-nodes` [IN]

### [REJECT] check-stale-requires-both-source-fields
A node must have both `source` (non-empty) and `source_hash` (non-empty) to be eligible for staleness checking; nodes missing either field are silently skipped.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Rejected: duplicate: already exists as `check-stale-requires-both-source-fields` [IN]

### [REJECT] hash-truncation-is-16-hex
STALE: Fixed in PR #40 — now uses full SHA-256 hash.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

### [REJECT] missing-source-file-is-silent
STALE: Fixed in PR #32 (issue #25) — check_stale now reports source_deleted for missing files.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

---

## From `reasons_lib/cli.py`

### [REJECT] cli-is-pure-formatter
Every `cmd_*` function delegates to `api.*` and only formats the returned dict for terminal output; no business logic lives in the CLI layer (sole exception: `cmd_propagate` touches `Storage` directly).
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `cli-is-pure-formatter` [IN]

### [REJECT] check-stale-exits-nonzero
`cmd_check_stale` calls `sys.exit(1)` when any stale nodes are found, making it usable as a CI or pre-commit gate.
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `check-stale-exits-nonzero` [IN]

### [REJECT] derive-strips-claudecode-env
`_derive_one_round` explicitly removes the `CLAUDECODE` environment variable before spawning the model subprocess, preventing recursive Claude Code invocation.
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `derive-strips-claudecode-env` [IN]

### [REJECT] exhaust-implies-auto
In `_derive_one_round`, proposals are auto-applied when either `args.auto` or `args.exhaust` is true; the `--exhaust` flag does not require the user to also pass `--auto`.
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `exhaust-implies-auto` [IN]

---

## From `reasons_lib/compact.py`

### [REJECT] compact-budget-only-limits-in-nodes
The token budget only constrains the IN nodes section; nogoods and OUT nodes are always emitted regardless of budget, so compact output can exceed the specified budget value.
- Source: entries/2026/04/23/reasons_lib-compact.md
- Rejected: duplicate: already exists as `compact-budget-only-limits-in-nodes` [OUT]

### [REJECT] compact-summary-hiding-requires-in
A summary node only hides its covered nodes when the summary itself is IN; if the summary goes OUT, covered nodes reappear in the compact output.
- Source: entries/2026/04/23/reasons_lib-compact.md
- Rejected: duplicate: already exists as `compact-summary-hiding-requires-in` [IN]

### [REJECT] compact-token-estimate-is-word-count
`estimate_tokens` counts whitespace-separated words, not BPE tokens; the budget parameter throughout the compact module is measured in this unit.
- Source: entries/2026/04/23/reasons_lib-compact.md
- Rejected: duplicate: already exists as OUT belief; contradicts IN beliefs `compact-estimate-tokens-chars-div-4` and `estimate-tokens-chars-div-4` which correctly state chars/4

### [REJECT] compact-in-nodes-ordered-by-dependents
IN nodes are sorted by descending dependent count so structurally important nodes (those depended on by many others) are emitted first and survive budget truncation.
- Source: entries/2026/04/23/reasons_lib-compact.md


I'll review the entries and extract architectural/behavioral beliefs from each.
- Rejected: duplicate: already exists as `compact-in-nodes-ordered-by-dependents` [IN]

### [REJECT] derive-retraction-guard-uses-jaccard
`validate_proposals` rejects any proposed belief ID with Jaccard similarity >= 0.5 to an existing OUT node, preventing re-derivation of retracted beliefs
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-retraction-guard-uses-jaccard` [IN]

### [REJECT] derive-prompt-roundtrips-through-parser
The `### DERIVE` / `### GATE` format is a shared contract between `DERIVE_PROMPT` LLM output, `parse_proposals()` input, and `write_proposals_file()` output, forming a closed serialization loop
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-prompt-roundtrips-through-parser` [IN]

### [REJECT] derive-depth-cycle-guard
`_get_depth` sets `memo[node_id] = 0` before recursing to prevent infinite recursion on cyclic justification chains; cycles resolve to depth 0
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-depth-cycle-guard` [IN]

### [REJECT] derive-agent-budget-proportional
When agents are present, `_build_beliefs_section` allocates prompt token budget proportionally to each agent's belief count, with a floor of 5 beliefs per agent
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-agent-budget-proportional` [IN]

### [REJECT] derive-fail-soft-validation
`validate_proposals` filters invalid proposals into a skipped list rather than raising; `apply_proposals` catches per-item exceptions so one bad proposal never blocks others
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-fail-soft-validation` [IN]

### [REJECT] derive-agent-count-bug
STALE: Fixed in PR #33 (issue #23).
- Source: entries/2026/04/23/reasons_lib-derive.md

### [REJECT] import-agent-namespace-prefix
Every node imported from agent X gets the ID prefix `X:`, including infrastructure nodes `X:active` and `X:inactive`, ensuring zero collision with local or other-agent beliefs
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: already exists as `import-agent-namespace-prefix` [IN]

### [REJECT] kill-switch-uses-outlist-not-antecedent
The `agent:inactive` node is placed in each imported belief's outlist (not antecedents) so that retracting `agent:active` cascades all imported beliefs to OUT, while per-belief retraction still works independently
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: already exists; referenced by `import-agent-outlist-not-antecedent` [IN]

### [REJECT] out-beliefs-imported-without-justifications
Beliefs that are OUT or STALE in the source are imported with an empty justification list, preventing `recompute_all` from resurrecting them to IN
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: covered by `import-handles-heterogeneous-truth-states` [IN]

### [REJECT] sync-is-remote-wins
`_sync_claims` implements remote-wins reconciliation: remote text/metadata overwrites local, beliefs removed from remote are retracted locally, and beliefs remotely IN but locally OUT are re-asserted
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: covered by existing `import-skips-existing-sync-is-remote-wins`

### [REJECT] import-two-phase-truth-maintenance
Import/sync adds all nodes first, then runs `recompute_all()` to propagate truth values, then performs explicit retractions — this ordering prevents incorrect cascades from partially-constructed networks
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: covered by `deferred-retraction-ordering` [IN] and `import-achieves-ordered-convergent-reconciliation` [IN]

### [ACCEPT] import-topo-sort-tolerates-cycles
`_topo_sort_claims` attempts topological ordering but appends remaining nodes when progress stalls, gracefully handling dependency cycles instead of erroring
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] sl-outlist-asymmetry
Missing antecedents invalidate a justification, but missing outlist nodes do not — this asymmetry enables "believe X unless Y" where Y may not yet exist in the network
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [REJECT] cp-equals-sl
CP (conditional-proof) justifications are evaluated with the exact same logic as SL justifications, despite being a distinct type in Doyle's TMS — either an intentional simplification or incomplete implementation
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Rejected: duplicate: already exists as `cp-equals-sl` [IN]

### [REJECT] justification-valid-is-pure
`_justification_valid` is a pure query with no side effects, no logging, and no mutations to network state
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Rejected: trivial: expected behavior for a validation predicate

### [REJECT] empty-antecedents-vacuously-valid
An SL justification with an empty antecedent list is valid (vacuous truth via `all([])`), allowing outlist-only justifications to function as "IN unless Y" — used by `challenge` and `supersede` for converted premises
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Rejected: duplicate: already exists as `empty-antecedents-vacuously-valid` [IN]

### [ACCEPT] propagate-skips-retracted-nodes
`_propagate` never recomputes truth values for nodes with `_retracted` in metadata, even if their justifications would support IN; only `assert_node` can restore them
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPT] propagate-does-not-change-trigger
The seed node (`changed_id`) is added to `visited` immediately and never has its own truth value recomputed; callers must update it before calling `_propagate`
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPT] propagate-cascade-stops-on-unchanged
If a dependent's recomputed truth value equals its current value, it is not enqueued — the cascade terminates along that path, making propagation selective rather than exhaustive
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [REJECT] propagate-assumes-dependents-exist
STALE: Fixed in PR #27 (issue #22) — _propagate now guards against dangling dependent references.
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [REJECT] add-nogood-always-records
`add_nogood` appends a `Nogood` record unconditionally before checking whether the contradiction is active, so nogoods are preserved even when not all member nodes are currently IN
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [REJECT] add-nogood-retraction-prefers-least-entrenched
The primary retraction path traces back through justification chains to premises and chooses the one with the lowest entrenchment score (speculative assumptions before evidence-backed observations)
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [REJECT] add-nogood-fallback-uses-dependent-count
When `find_culprits` returns no candidates (all nogood nodes are premises with no justification chains), the fallback retracts the nogood member with the fewest direct dependents
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [REJECT] nogood-ids-assume-append-only
STALE: Fixed in PR #35 (issue #26) — now uses a persisted monotonic counter.
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [REJECT] unknown-type-returns-false
Low value — any justification type other than SL/CP returning False is defensive behavior unlikely to affect a developer working in this codebase; the system only creates SL and CP types
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [REJECT] add-nogood-empty-list-crashes
Bug report about an edge case (empty `node_ids` causes `IndexError`) — this is a bug to fix, not an architectural invariant to track as a belief
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [REJECT] propagate-bfs-order-guarantee
The BFS ordering of changed nodes is an implementation detail that no caller currently depends on for correctness — tracking it as a belief adds noise without preventing breakage
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md


All claims confirmed against the source. Here are the proposed beliefs extracted from the four entries:

---

## From `scan-ftl-reasons.md`

### [REJECT] three-layer-architecture
The codebase is a three-layer stack: data model (`__init__.py`), TMS engine (`network.py`), and persistence (`storage.py`), with `api.py` providing functional API and `cli.py` as a thin argparse wrapper.
- Source: entries/2026/04/23/scan-ftl-reasons.md
- Rejected: trivial: architectural layers are obvious from file names (network.py, storage.py, api.py, cli.py)

### [REJECT] network-is-central-dependency
`network.py` is imported by essentially every other module — api, storage, import, export, compact, check_stale, and all test files — making it the central data structure of the project.
- Source: entries/2026/04/23/scan-ftl-reasons.md
- Rejected: duplicate: captured by existing `central-dependency-is-safely-contained`

### [REJECT] api-uses-with-network-context-manager
`api.py` uses a `_with_network` context manager to ensure load-operate-save atomicity for all network mutations.
- Source: entries/2026/04/23/scan-ftl-reasons.md
- Rejected: duplicate: already exists as accepted belief with same ID

---

## From `reasons_lib-network.md`

### [ACCEPT] sl-justification-semantics
An SL justification is valid iff ALL antecedents are IN AND ALL outlist nodes are OUT; a node is IN iff ANY of its justifications is valid (conjunction within a justification, disjunction across justifications).
- Source: entries/2026/04/23/reasons_lib-network.md

### [REJECT] propagation-is-bfs
Truth value propagation in `_propagate` uses `deque`-based BFS through the `dependents` graph, not DFS, ensuring breadth-first wavefront expansion.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: BFS already captured in add-justification-triggers-propagation

### [ACCEPT] premise-defaults-to-in
A node with no justifications (a premise) defaults to IN; `_compute_truth` preserves its current truth value rather than recomputing it.
- Source: entries/2026/04/23/reasons_lib-network.md

### [REJECT] dependents-bidirectional-index
Each node maintains a `dependents` set (reverse of antecedent/outlist edges), eagerly maintained by `add_node`, `add_justification`, `supersede`, `challenge`, and `convert_to_premise`.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: already exists as accepted belief with same ID (currently OUT)

### [ACCEPT] recompute-all-uses-fixpoint
`recompute_all` iterates until no truth values change, bounded by `len(nodes) + 1` iterations, handling cascading dependencies from arbitrary node ordering.
- Source: entries/2026/04/23/reasons_lib-network.md

### [REJECT] retracted-nodes-skipped-in-propagation
Retracted nodes (marked with `_retracted` metadata) are skipped during BFS propagation but remain in the network for potential restoration.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: same claim as proposed propagate-skips-retracted-nodes

### [REJECT] missing-outlist-nodes-pass-validation
In `_justification_valid`, missing antecedent nodes cause the check to fail (node goes OUT), but missing outlist nodes pass (don't block) — an open-world default.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: same claim as proposed sl-outlist-asymmetry

### [REJECT] backtracking-retracts-least-entrenched
`add_nogood` resolves contradictions via dependency-directed backtracking: `find_culprits` traces to premises, scores by `_entrenchment`, and retracts the least-entrenched premise to minimize disruption.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: already exists as accepted belief with same ID

---

## From `reasons_lib-network-challenge.md`

### [REJECT] challenge-uses-outlist-mechanism
`challenge` works by creating an IN premise node and adding it to the target's outlist in every justification, reusing the same non-monotonic mechanism as `supersede`.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: same claim as existing `challenge-is-outlist-injection`

### [REJECT] challenge-converts-premises-to-justified
When a premise (node with no justifications) is challenged, it is converted to a justified node with an SL justification containing empty antecedents and the challenge in the outlist.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [REJECT] challenge-modifies-all-justifications
When the target has multiple justifications, the challenge node is added to the outlist of every justification, ensuring no single justification can independently keep the target IN.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [REJECT] challenge-id-auto-generation
Auto-generated challenge IDs follow the pattern `challenge-{target}`, then `challenge-{target}-2`, `-3`, etc.; explicit IDs that collide raise `ValueError` rather than auto-deduplicating.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [REJECT] defend-is-challenge-of-challenge
`defend` works by calling `challenge` on the challenge node itself, creating a recursive dialectical structure where truth values resolve automatically through the same outlist mechanism.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

---

## From `reasons_lib-storage.md`

### [ACCEPT] storage-save-is-full-replace
`Storage.save()` deletes all rows from every table before re-inserting the entire network; there is no incremental or differential update path.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [REJECT] storage-load-bypasses-propagation
`load()` constructs nodes directly into `network.nodes` rather than calling `add_node`, so truth maintenance propagation does not fire during deserialization.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: `bootstrap-bypasses-incremental-propagation` already captures that persistence loading constructs the full node graph before truth maintenance

### [ACCEPT] justification-order-preserved-via-rowid
Justification insertion order is preserved across save/load cycles using `AUTOINCREMENT` rowid and `ORDER BY rowid` on read, which matters because justification priority affects truth maintenance.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [REJECT] dependents-index-derived-on-load
The `node.dependents` set is never persisted to SQLite; it is rebuilt by walking all justification antecedents and outlists during `load()`.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: already exists as belief `dependents-index-derived-on-load` (currently OUT)

### [REJECT] storage-trusts-stored-truth-values
`load()` trusts the stored `truth_value` without re-running propagation, making the database the source of truth for node status.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: covered by bootstrap-bypasses-incremental-propagation which already states "load trusts stored truth values"

---

## From `topic-doyle-1979-tms.md`

### [REJECT] cp-and-sl-evaluated-identically
CP and SL justifications use the same validity check in `_justification_valid`; the distinction is semantic (support vs. consistency), not computational.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md
- Rejected: duplicate: already exists as belief `cp-and-sl-evaluated-identically` (currently IN)

### [REJECT] sl-justification-is-conjunction-of-inlist-disjunction-of-outlist
Redundant with `sl-justification-semantics` proposed above, which captures the same claim more concisely.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md

### [REJECT] retracted-nodes-stay-in-network
Redundant with `retracted-nodes-skipped-in-propagation` which captures the same fact plus the propagation behavior.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md


No existing beliefs. Here are the proposed beliefs from both entries:

---

## From `entries/2026/04/23/topic-multi-agent-federation.md`

### [REJECT] namespace-prefix-is-colon-separated
Agent namespacing uses the format `agent_name:belief_id`; `_resolve_namespace()` skips prefixing any ID that already contains a colon, preventing double-prefixing of cross-namespace references
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: covered by existing `colon-means-already-namespaced` and `import-agent-namespace-prefix`

### [REJECT] active-inactive-relay-pair
Each imported agent gets exactly two infrastructure nodes: `agent:active` (premise, starts IN) and `agent:inactive` (derived via SL with `outlist=[active_id]`, starts OUT); every imported belief includes `inactive_id` in its outlist
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists as belief `active-inactive-relay-pair` (currently IN)

### [REJECT] active-not-in-antecedents
The `active` premise is deliberately excluded from imported beliefs' antecedents; if it were an antecedent, it would provide a second always-valid justification path that defeats per-belief retraction semantics
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists as belief `active-not-in-antecedents` (currently IN)

### [REJECT] kill-switch-cascade-is-reversible
Retracting `agent:active` cascades all agent beliefs to OUT via the inactive relay flipping IN; re-asserting `agent:active` reverses the cascade, restoring all beliefs to IN via the same BFS propagation
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: covered by `all-defeat-mechanisms-are-reversible` and `defeat-reversal-propagates-automatically`

### [REJECT] agent-cascades-are-isolated-by-namespace
Retracting one agent's active premise does not affect other agents' beliefs, because each agent's imported beliefs reference only their own `inactive` node in their outlist
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists as belief `agent-cascades-are-isolated-by-namespace` (currently IN)

### [REJECT] justification-validity-requires-inlist-in-and-outlist-out
Redundant with sl-justification-semantics.
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [REJECT] import-skips-existing-sync-is-remote-wins
Import mode (`import_agent`) is a one-time load that skips existing nodes; sync mode updates text/justifications/truth values with remote-wins semantics and retracts locally any beliefs removed from the remote
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists (referenced by `idempotent-reimport-skips-all`)

---

## From `entries/2026/04/23/topic-outlist-semantics.md`

### [REJECT] outlist-absent-means-out
Redundant with missing-outlist-nodes-pass-validation and sl-outlist-asymmetry.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [REJECT] node-in-if-any-justification-valid
Duplicate of node-in-if-any-justification-valid proposed earlier.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [REJECT] challenge-is-outlist-injection
Redundant with challenge-uses-outlist-mechanism.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [REJECT] defend-is-recursive-challenge
Redundant with defend-is-challenge-of-challenge.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [REJECT] supersession-is-reversible
`supersede()` adds the new node's ID to the old node's outlist rather than deleting the old node; retracting the new belief automatically restores the old one through normal propagation
- Source: entries/2026/04/23/topic-outlist-semantics.md
- Rejected: duplicate: already captured by `all-defeat-mechanisms-are-reversible` and `defeat-reversal-propagates-automatically`

### [REJECT] multiple-outlist-is-conjunction
When a justification has multiple outlist entries, ALL must be OUT for the justification to be valid; any single outlist node going IN defeats the entire justification
- Source: entries/2026/04/23/topic-outlist-semantics.md
- Rejected: duplicate: subsumed by proposed sl-justification-semantics ("ALL outlist nodes are OUT")

### [REJECT] outlist-relationships-survive-persistence
Outlists are stored as `outlist_json` in the SQLite `justifications` table; on load, the dependent index is rebuilt for both antecedents and outlist nodes, preserving propagation behavior across save/load cycles
- Source: entries/2026/04/23/topic-outlist-semantics.md
- Rejected: duplicate: `dependents-survive-storage-roundtrip` + `dependency-tracking-is-complete-for-all-reference-types` already cover that outlist references persist and the dependents index is rebuilt for both antecedents and outlists



---

**Generated:** 2026-05-04
**Source:** 3 entries from entries/
**Model:** claude

### [ACCEPT] pgapi-find-dependents-queries-outlist
PgApi's `_find_dependents` queries both `antecedents @>` and `outlist @>` JSONB containment, correctly enqueuing outlist-dependent nodes for re-evaluation — a capability the in-memory Network historically lacked until PR #31.
- Source: entries/2026/04/29/gated-and-negative-analysis.md

### [ACCEPT] pgapi-validates-refs-before-justification-insert
PgApi's `_validate_refs` checks all antecedent and outlist node IDs exist in `rms_nodes` before inserting a justification, providing application-level referential integrity that compensates for JSONB arrays' inability to enforce foreign key constraints.
- Source: entries/2026/04/29/session-summary.md

### [REJECT] pgapi-what-if-uses-transaction-rollback
PgApi's `what_if_retract` and `what_if_assert` wrap BFS cascade analysis inside a database transaction with `try/finally` guaranteed ROLLBACK, achieving read-only simulation by never committing the exploratory mutations.
- Source: entries/2026/04/29/session-summary.md
- Rejected: duplicate: already captured by `both-backends-support-safe-hypothetical-reasoning`

### [ACCEPT] list-negative-json-parser-tolerates-prose-preamble
The `list_negative` LLM classification response parser uses `re.finditer` to extract JSON objects from responses that include prose preamble, handling the common LLM pattern of prefacing structured output with natural language rather than requiring clean JSON.
- Source: entries/2026/04/29/session-summary.md



---

**Generated:** 2026-05-05
**Source:** 23 entries from entries/
**Model:** claude

### [REJECT] update-2026-05-04
Status update listing already-tracked beliefs and their blockers; no new architectural claims to extract beyond what is already accepted.
- Source: entries/2026/05/04/update.md

---

### [REJECT] derived-belief-soundness-is-llm-only
Structural validation ensures justification references exist and are IN, but the logical soundness of the inference from antecedents to derived conclusion is validated only by the proposing LLM — no code-level check verifies that the reasoning step is logically valid.
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md
- Rejected: duplicate: belief `derived-belief-soundness-is-llm-only` already exists IN the current network with the same claim

### [ACCEPT] premise-derived-verifiability-asymmetry
Premises record source file and line range at assertion time, enabling check-stale to verify them against current code; derived beliefs have no equivalent source-level grounding and can only be structurally validated (references exist and are IN).
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md

### [REJECT] tms-provides-revisability-not-certainty
The TMS design provides traceable, revisable reasoning rather than certainty: every belief records its complete justification chain back to premises, beliefs are automatically retracted when support changes, and the epistemological status of each belief is mechanically traceable via `reasons explain`.
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md
- Rejected: speculative: editorial judgment about design philosophy, not an observable code fact

---

### [REJECT] ftl-reasons-zero-runtime-deps
The core library declares `dependencies = []` in pyproject.toml; the TMS engine runs on the Python standard library alone (including `sqlite3`), with PostgreSQL support available only via the `pg` optional extra.
- Source: entries/2026/05/05/pyproject.md
- Rejected: duplicate: already captured by `architecture-is-self-contained-and-safely-layered` (zero runtime dependencies)

### [REJECT] reasons-cli-entrypoint-is-cli-main
The `reasons` console script always resolves to `reasons_lib.cli:main()` with no plugin system, alternate entry point, or dispatch indirection.
- Source: entries/2026/05/05/pyproject.md
- Rejected: duplicate: already captured by `entry-point-mapping`

### [REJECT] tests-excluded-from-distribution
Only `reasons_lib/` is packaged for distribution; the `tests/` directory is excluded via explicit `[tool.setuptools] packages = ["reasons_lib"]` configuration.
- Source: entries/2026/05/05/pyproject.md
- Rejected: trivial: standard setuptools packaging config directly visible in pyproject.toml

### [ACCEPT] pg-extra-required-for-postgres
PostgreSQL support requires installing the `pg` or `test-pg` optional extra (`psycopg[binary]>=3.1`); it is not available in a bare install.
- Source: entries/2026/05/05/pyproject.md

### [REJECT] python-310-minimum-required
The project requires Python >= 3.10 (`requires-python = ">=3.10"` in pyproject.toml); anything below fails at install time.
- Source: entries/2026/05/05/pyproject.md
- Rejected: trivial: single field directly visible in pyproject.toml

---

### [REJECT] node-premise-is-implicit
A node is a premise (believed without justification) iff its `justifications` list is empty; there is no explicit `is_premise` flag or type discriminator.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: already captured by `challenge-destroys-premise-identity` (premise identity emerges from absence of justifications)

### [REJECT] node-truth-disjunctive
A node is IN if ANY of its justifications is valid (disjunctive semantics); adding a justification to a node can never cause it to go OUT.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: disjunctive semantics already stated in proposed sl-justification-semantics; monotonicity corollary follows logically and is partially covered by add-justification-achieves-consistent-propagation

### [REJECT] dependents-not-self-enforced
The `Node.dependents` reverse index is not maintained by the data model itself; `network.py` is responsible for keeping it consistent with justification references (antecedents and outlists).
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: same claim as existing `dependents-is-manual-reverse-index`

### [REJECT] data-model-uses-string-enums
Both `Justification.type` ("SL"/"CP") and `Node.truth_value` ("IN"/"OUT") are plain strings, not Python enums; consumers must validate values themselves as invalid states like `"MAYBE"` or `"XYZ"` are representable.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: already exists as `data-model-uses-string-enums [IN]`

### [REJECT] init-has-zero-external-deps
`reasons_lib/__init__.py` imports only `dataclasses` from the standard library, making it the dependency-free root of the package — imported by 29 test files and transitively by every module in `reasons_lib`.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: trivial: standard-library-only imports visible from file; "29 test files" is an ephemeral count

---

### [REJECT] namespace-kill-switch-uses-active-premise
Retracting the `{ns}:active` premise node cascades all beliefs in that namespace to OUT; `ensure_namespace()` creates this node with `role: agent_premise` metadata as the namespace's kill switch.
- Source: entries/2026/05/05/reasons_lib-api.md
- Rejected: duplicate: covered by `active-inactive-relay-pair` and `kill-switch-uses-outlist-not-antecedent`

### [ACCEPT] fts-errors-silently-caught-in-search
FTS5 query errors in `_fts_search` are silently caught and return an empty list, falling back to substring matching — the only place in the API where errors are deliberately swallowed (FTS5 table may not exist).
- Source: entries/2026/05/05/reasons_lib-api.md

### [REJECT] cross-namespace-refs-skip-prefixing
`_resolve_namespace` skips node IDs that already contain `:`, treating them as cross-namespace references — so `alice:some-belief` referenced from Bob's namespace stays `alice:some-belief`, not `bob:alice:some-belief`.
- Source: entries/2026/05/05/reasons_lib-api.md
- Rejected: duplicate: same claim as `colon-means-already-namespaced`

### [ACCEPT] search-relaxation-capped-at-50-queries
FTS progressive relaxation drops terms via `combinations()` (largest subsets first) until results appear, capped at 50 total relaxation queries to prevent combinatorial explosion on long search strings.
- Source: entries/2026/05/05/reasons_lib-api.md


### [REJECT] ask-dual-mode-makes-three-llm-calls
Dual mode makes up to 3 LLM calls (1 TMS synthesis + 1 FTS RAG + 1 merge), short-circuiting to 2 if either retrieval path returns empty.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: already exists as `ask-dual-mode-makes-three-llm-calls [IN]`

### [REJECT] ask-tool-call-parsing-is-line-based-json
Tool calls are detected by scanning each line of the LLM response for valid JSON with a `"tool"` key; multi-line JSON objects are silently missed.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: core claim matches `ask-uses-text-based-tool-protocol`

### [REJECT] ask-sources-db-failure-silently-degrades
If the `sources_db` SQLite file is missing or corrupt, `_search_source_chunks` catches `OperationalError`/`DatabaseError` and returns empty string, degrading to belief-only mode without user-visible errors.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: already exists as `ask-sources-db-failure-silently-degrades [IN]`

### [REJECT] ask-tool-loop-max-three-rounds
Covered by existing `ask-has-tiered-query-modes` which already describes the "bounded 3-iteration tool loop."
- Source: entries/2026/05/05/reasons_lib-ask.md

### [REJECT] ask-llm-failure-returns-raw-beliefs
Covered by existing `ask-falls-back-to-raw-search` and `ask-never-raises-on-llm-failure`.
- Source: entries/2026/05/05/reasons_lib-ask.md

### [ACCEPT] truncated-hash-threshold-is-16-chars
A stored `source_hash` of exactly 16 characters that is a prefix of the current full SHA-256 hash is classified as `truncated_hash` (legacy format), not `content_changed`.
- Source: entries/2026/05/05/reasons_lib-check_stale.md

### [REJECT] check-stale-and-hash-sources-mutate-in-place
Both `check_stale` (with `upgrade_hashes=True`) and `hash_sources` modify `node.source_hash` directly on the Network object; neither persists — the caller must save.
- Source: entries/2026/05/05/reasons_lib-check_stale.md
- Rejected: duplicate: already exists as `check-stale-and-hash-sources-mutate-in-place [IN]`

### [REJECT] check-stale-only-examines-in-nodes
Already exists as `check-stale-skips-out-nodes`.
- Source: entries/2026/05/05/reasons_lib-check_stale.md

### [REJECT] cli-is-pure-adapter
Already exists as `cli-is-pure-delegation-layer` and `cli-is-verified-pure-delegation`.
- Source: entries/2026/05/05/reasons_lib-cli.md

### [REJECT] check-stale-exits-nonzero-on-stale
`cmd_check_stale` calls `sys.exit(1)` when stale nodes are found (even on success), making the command usable as a CI or pre-commit gate.
- Source: entries/2026/05/05/reasons_lib-cli.md
- Rejected: duplicate: identical to existing `check-stale-exits-nonzero`

### [REJECT] commands-dict-must-mirror-subparsers
Adding a CLI subcommand requires entries in both the argparse subparser definitions and the `commands` dispatch dict in `main()`; omitting either silently breaks the command.
- Source: entries/2026/05/05/reasons_lib-cli.md
- Rejected: duplicate: already exists as `commands-dict-must-mirror-subparsers [IN]`

### [REJECT] derive-one-round-return-protocol
The key invariant (negative = error) is captured by existing `derive-round-returns-negative-on-error`; positive = count added and 0 = saturated are the obvious complement.
- Source: entries/2026/05/05/reasons_lib-cli.md

### [REJECT] deferred-imports-for-llm-modules
Same architectural pattern already captured by `api-uses-lazy-imports`; the cli.py instance follows the same rationale.
- Source: entries/2026/05/05/reasons_lib-cli.md

### [REJECT] import-agent-inactive-in-outlist-not-antecedent
Already exists as `active-not-in-antecedents` and `kill-switch-uses-outlist-not-antecedent`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPT] import-agent-out-beliefs-get-empty-justifications
Beliefs that are OUT or STALE in the source are imported with empty justification lists, preventing `recompute_all` from resurrecting them to IN.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [REJECT] import-agent-topo-sort-before-add
Already exists as `import-json-uses-topological-sort`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [REJECT] sync-agent-remote-wins-semantics
Already exists as `import-skips-existing-sync-is-remote-wins`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [REJECT] import-agent-dangling-refs-silently-dropped
Already exists as `normalization-drops-unknown-refs`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPT] llm-prompt-always-via-stdin
Prompts are passed to LLM subprocesses via `subprocess.run(input=...)`, never as command-line arguments — avoiding shell injection and argument length limits.
- Source: entries/2026/05/05/reasons_lib-llm.md

### [REJECT] ollama-thinking-strip-is-fragile
The ollama output post-processing strips exact string markers (`Thinking...\n` / `...done thinking.\n`) that are coupled to ollama's current output format and may break across versions.
- Source: entries/2026/05/05/reasons_lib-llm.md
- Rejected: speculative: editorial judgment about fragility; "may break across versions" is a prediction

### [REJECT] llm-invoke-is-subprocess-only
Covered by existing `all-external-execution-is-subprocess-isolated`.
- Source: entries/2026/05/05/reasons_lib-llm.md

### [REJECT] claudecode-env-stripped-from-child
Covered by existing `llm-subprocess-isolation-prevents-recursion`.
- Source: entries/2026/05/05/reasons_lib-llm.md

### [REJECT] llm-module-stdlib-only
Covered by existing `project-has-zero-external-coupling`.
- Source: entries/2026/05/05/reasons_lib-llm.md


### [ACCEPT] network-is-sole-truth-propagation-engine
All truth value computation and propagation in the system flows through `Network._propagate()` and `Network._compute_truth()`; no other module modifies truth values directly.
- Source: entries/2026/05/05/reasons_lib-network.md

### [REJECT] sl-justification-is-disjunctive
A node is IN if *any* of its justifications is valid (disjunctive semantics); `_compute_truth()` short-circuits on the first valid justification rather than requiring all justifications to hold.
- Source: entries/2026/05/05/reasons_lib-network.md
- Rejected: duplicate: same claim as proposed `node-truth-disjunctive` with minor implementation detail added

### [REJECT] retracted-nodes-skip-propagation
Duplicate of existing `metadata-actively-governs-truth-propagation` which states "retracted nodes are skipped during BFS traversal and trigger nodes are never recomputed."
- Source: entries/2026/05/05/reasons_lib-network.md

### [REJECT] forward-references-are-silently-tolerated
Covered by existing `outlist-absent-means-out` (missing outlist nodes treated as OUT) and `active-not-in-antecedents` (missing antecedents treated as not-IN), which together describe the forward-reference tolerance behavior.
- Source: entries/2026/05/05/reasons_lib-network.md

### [REJECT] outlist-semantics-enable-nonmonotonic-reasoning
Covered by existing `challenge-is-outlist-injection`, `kill-switch-uses-outlist-not-antecedent`, and `dialectics-inherit-complete-outlist-semantics`, which together establish that all non-monotonic operations reduce to outlist manipulation.
- Source: entries/2026/05/05/reasons_lib-network.md

### [ACCEPT] pgapi-has-no-retry-logic
`PgApi` uses one connection per instance with no retry on transient database errors; callers are expected to handle `psycopg` exceptions from connection failures or serialization conflicts.
- Source: entries/2026/05/05/reasons_lib-pg.md

### [REJECT] access-tags-are-monotonically-inherited
A derived node's `access_tags` are the union of its own tags and all antecedent tags; tags are never removed by inheritance, only accumulated, so access restrictions can only grow along derivation chains.
- Source: entries/2026/05/05/reasons_lib-pg.md
- Rejected: duplicate: monotonic growth is a direct consequence of union semantics in `access-tags-union-inheritance`

### [ACCEPT] review-only-evaluates-derived-beliefs
`review_beliefs` filters out premises (nodes without justifications); only derived beliefs with at least one justification are sent for LLM review.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPT] review-parse-defaults-fail-safe
`parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True`, so a missing or malformed field in LLM output never triggers a false alarm.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPT] review-batch-failure-is-silent-skip
When an LLM call fails for a review batch, the error is logged to stderr but the batch is skipped with no indication in the returned results; callers cannot distinguish "skipped due to error" from "no problems found."
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPT] review-has-no-storage-dependency
The review module operates entirely on an in-memory `nodes` dict (from `export_network()`) and never reads from or writes to the database directly.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPT] review-result-schema-is-normalized
Every result dict returned by `parse_review_response` is guaranteed to have exactly six keys (`id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`) regardless of what the LLM returned, via normalization with safe defaults.
- Source: entries/2026/05/05/reasons_lib-review.md

### [REJECT] storage-load-bypasses-add-node
`load()` assigns nodes directly to `network.nodes` and calls `_rebuild_dependents()` afterward, deliberately skipping `add_node()` to avoid triggering truth maintenance propagation during state restoration.
- Source: entries/2026/05/05/reasons_lib-storage.md
- Rejected: duplicate: covered by `bootstrap-bypasses-incremental-propagation` which already captures that loading constructs the full graph before truth maintenance, skipping incremental propagation

### [ACCEPT] storage-justification-order-preserved
Justifications are inserted in list order and loaded via `ORDER BY rowid`, preserving the ordering that determines which justification is evaluated first during truth computation.
- Source: entries/2026/05/05/reasons_lib-storage.md

### [REJECT] storage-nogood-counter-persisted
Covered by existing `all-identifiers-are-durable-across-persistence-boundaries` which states "nogood IDs survive save/load cycles via the network_meta table's high-water mark."
- Source: entries/2026/05/05/reasons_lib-storage.md

### [REJECT] api-tests-use-real-sqlite
All API tests run against a real SQLite database with FTS5; storage is never mocked, ensuring the API contract includes correct SQL and full-text search behavior.
- Source: entries/2026/05/05/tests-test_api.md
- Rejected: duplicate: identical belief ID already exists as IN

### [ACCEPT] list-negative-batches-at-50
`list_negative` splits candidate nodes into batches of approximately 50 for LLM classification, verified by the test suite asserting exactly 3 LLM calls for 120 candidates.
- Source: entries/2026/05/05/tests-test_api.md

### [REJECT] fts-relaxation-bounded
Progressive FTS query relaxation is bounded: a 20-term query produces at most 51 `_fts_query` invocations, preventing unbounded search expansion on long input queries.
- Source: entries/2026/05/05/tests-test_api.md
- Rejected: duplicate: same claim as search-relaxation-capped-at-50-queries — both assert FTS relaxation is bounded

### [REJECT] retract-assert-inverse-tested
Test coverage claim rather than code invariant; the behavioral property (retract/assert are inverses) is already captured by `defeat-reversal-propagates-automatically` and `supersession-is-reversible-and-view-consistent`.
- Source: entries/2026/05/05/tests-test_api.md

### [REJECT] api-error-contract-exceptions
Duplicate of existing `api-enforces-typed-preconditions` which covers typed exceptions (ValueError, KeyError, PermissionError) at API boundaries.
- Source: entries/2026/05/05/tests-test_api.md


### [REJECT] ask-tool-call-budget-is-three
The agentic `ask` loop permits at most 3 tool-call rounds before forcing synthesis via `build_final_prompt` (no tool definition), resulting in exactly 4 LLM invocations maximum.
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: same claim as existing `ask-tool-loop-capped-at-three` (3-round cap with forced synthesis)

### [REJECT] ask-dual-requires-sources-db
Calling `ask(..., dual=True)` without providing a `sources_db` path raises `ValueError`.
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: identical belief ID already exists as IN

### [REJECT] ask-natural-mode-strips-metadata-and-cite
When `natural=True`, all belief metadata (`**Status:**`, `### ` headers, `**Source:**`) is stripped from the prompt context and the "Cite belief IDs" instruction is replaced with "plain natural language".
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: identical belief ID already exists as IN

### [ACCEPT] search-source-chunks-filters-stop-words
`_search_source_chunks` filters out single-character tokens and common stop words before constructing FTS5 queries, returning empty string when no usable terms remain.
- Source: entries/2026/05/05/tests-test_ask.md

### [ACCEPT] resolve-source-path-db-dir-precedence
`resolve_source_path` follows a strict precedence chain: `db_dir` resolution takes priority over agent repo resolution, which takes priority over repo-key-split resolution; missing files return `None`.
- Source: entries/2026/05/05/tests-test_check_stale.md

### [ACCEPT] truncated-hash-upgrade-opt-in
Prefix hash upgrade (16-char truncated SHA-256 to full digest) only occurs when `upgrade_hashes=True` is passed to `check_stale`; without it, truncated hashes produce a warning result with `reason="truncated_hash"`.
- Source: entries/2026/05/05/tests-test_check_stale.md

### [ACCEPT] import-agent-retraction-survives-recompute
After retracting an imported belief, `recompute_all()` must not resurrect it — the justification structure must not provide alternative paths to IN (issue #16 regression invariant).
- Source: entries/2026/05/05/tests-test_import_agent.md

### [ACCEPT] llm-resolve-ollama-preserves-inner-colons
`resolve_model_cmd("ollama:model:tag")` splits on the first colon only, keeping `model:tag` intact as the Ollama model identifier.
- Source: entries/2026/05/05/tests-test_llm.md

### [ACCEPT] llm-thinking-strip-ollama-only
Thinking-marker stripping (`Thinking...` / `...done thinking.`) is applied only to Ollama model output; Claude and Gemini output passes through unmodified even if it contains the same markers.
- Source: entries/2026/05/05/tests-test_llm.md

### [REJECT] ask-graceful-degradation-on-llm-failure
Duplicate of existing `ask-never-raises-on-llm-failure` and `all-query-operations-degrade-gracefully`.
- Source: entries/2026/05/05/tests-test_ask.md

### [REJECT] check-stale-returns-dicts-not-exceptions
Duplicate of existing `staleness-results-are-dicts-not-exceptions`.
- Source: entries/2026/05/05/tests-test_check_stale.md

### [REJECT] check-stale-requires-source-and-hash
Duplicate of existing `check-stale-requires-both-source-fields`.
- Source: entries/2026/05/05/tests-test_check_stale_issue25.md

### [REJECT] source-deleted-nulls-hash-and-path
Duplicate of existing `check-stale-source-deleted-returns-none-hashes`.
- Source: entries/2026/05/05/tests-test_check_stale_issue25.md

### [REJECT] import-agent-namespaces-all-ids
Duplicate of existing `import-agent-namespace-prefix`.
- Source: entries/2026/05/05/tests-test_import_agent.md

### [REJECT] import-agent-active-premise-not-antecedent
Duplicate of existing `active-not-in-antecedents`.
- Source: entries/2026/05/05/tests-test_import_agent.md

### [REJECT] llm-invoke-strips-claudecode-env
Covered by existing `llm-subprocess-isolation-prevents-recursion` and `all-external-execution-is-subprocess-isolated`.
- Source: entries/2026/05/05/tests-test_llm.md

### [REJECT] llm-invoke-checks-path-before-run
Covered by existing `invoke-claude-raises-when-binary-missing` (same behavior generalized to all model binaries).
- Source: entries/2026/05/05/tests-test_llm.md

### [REJECT] llm-three-error-types
Too granular — the three error types (ValueError, FileNotFoundError, RuntimeError) are implementation details of the error taxonomy already captured by existing beliefs about fail-fast and fail-soft behavior.
- Source: entries/2026/05/05/tests-test_llm.md


Looking at the three entries, I'll extract beliefs and cross-check against the existing 189-belief registry.

---

## From `tests/test_pg.py`

### [ACCEPT] sl-param-is-comma-separated-node-ids
`add_node()` encodes SL justifications as comma-separated node IDs in the `sl=` parameter and outlist nodes in `unless=`, mirroring the TMS (SL, OL) formalism directly in the API signature.
- Source: entries/2026/05/05/tests-test_pg.md

### [REJECT] pg-tests-gated-by-database-availability
The entire `test_pg.py` suite (18 test classes) is conditionally skipped via `skip_no_pg` marker from conftest when PostgreSQL is unavailable, and selectively collectible via `pytest.mark.pg` — enabling CI without a Postgres dependency.
- Source: entries/2026/05/05/tests-test_pg.md
- Rejected: trivial: conditional skip markers for unavailable services are standard pytest practice visible from conftest

### [ACCEPT] pg-fixture-provides-per-test-isolation
Each pg test receives an isolated `PgApi` instance via the `pg_api` fixture, which creates a fresh project namespace (UUID) and tears down after each test, ensuring zero shared state between tests.
- Source: entries/2026/05/05/tests-test_pg.md

### [ACCEPT] pg-keyerror-includes-missing-node-id
When `add_node` references a nonexistent node in `sl=` or `unless=`, PgApi raises `KeyError` with the missing node ID in the exception message — tested via `pytest.raises(KeyError, match="ghost")` — enabling callers to identify which reference is dangling.
- Source: entries/2026/05/05/tests-test_pg.md

### [ACCEPT] pg-test-suite-is-backend-parity
`test_pg.py` is a parity test suite: every behavior tested has a corresponding expected behavior from the SQLite backend, validating that PgApi returns the same dict shapes and enforces the same invariants to ensure backend interchangeability.
- Source: entries/2026/05/05/tests-test_pg.md

---

## From `tests/test_review.py`

### [ACCEPT] review-skips-premises
`review_beliefs` only sends beliefs with at least one justification to the LLM; premise nodes (empty justifications list) are excluded from review entirely.
- Source: entries/2026/05/05/tests-test_review.md

### [ACCEPT] parse-review-defaults-to-passing
`parse_review_response` defaults missing fields to valid=True, sufficient=True, necessary=True, unnecessary_antecedents=[] — the LLM only needs to report failures explicitly; omission means the belief passed.
- Source: entries/2026/05/05/tests-test_review.md

### [ACCEPT] review-parse-requires-json-array
`parse_review_response` only accepts JSON arrays; a bare JSON object `{...}` is treated as unparseable and returns an empty list — the LLM must return a list even for single-item reviews.
- Source: entries/2026/05/05/tests-test_review.md

### [REJECT] auto-retract-respects-dry-run
The `--auto-retract` flag in the `review-beliefs` CLI is gated by `--dry-run`: when dry-run is active, findings are displayed but no database mutation occurs, even for beliefs flagged as invalid.
- Source: entries/2026/05/05/tests-test_review.md
- Rejected: duplicate: already exists as `auto-retract-respects-dry-run` [IN] in the belief network

### [ACCEPT] review-format-handles-missing-antecedents
`format_belief_for_review` renders `"(not found in network)"` for antecedent IDs that don't exist in the nodes dict rather than crashing, and returns an empty string for a nonexistent belief ID.
- Source: entries/2026/05/05/tests-test_review.md

### [REJECT] review-tolerates-llm-failure
Duplicates existing beliefs `llm-integration-fails-softly-across-modules` and `all-llm-interactions-are-bounded-and-fail-soft`, which already establish that all LLM integration fails softly across the system.
- Source: entries/2026/05/05/tests-test_review.md

---

## From `tests/test_sync_agent.py`

### [ACCEPT] sync-agent-is-idempotent
Calling `sync_agent` twice with identical data produces the same database state; the second call returns zero adds, removes, and updates — a no-op by design.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPT] sync-returns-structured-diff-counts
`sync_agent` returns a dict with `beliefs_added`, `beliefs_updated`, `beliefs_unchanged`, and `beliefs_removed` counts that accurately reflect the diff between remote file and local state — no double-counting across sync cycles.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPT] sync-registers-agent-repo-path
`sync_agent` records the agent's source file path in the repo registry (`list_repos`), enabling the system to track which file each agent's beliefs originated from.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [REJECT] sync-agent-is-remote-wins
Duplicates existing belief `import-skips-existing-sync-is-remote-wins`, which already establishes that sync mode uses remote-wins semantics including overriding local retractions.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [REJECT] sync-supports-markdown-and-json
Duplicates existing belief `import-agent-normalizers-share-intermediate-schema`, which covers that both markdown and JSON formats flow through the same normalization pipeline for import/sync operations.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [REJECT] stale-beliefs-import-as-out
Duplicates existing belief `staleness-information-survives-binary-truth-model`, which already establishes that STALE beliefs map to OUT with stale_reason metadata preserved end-to-end.
- Source: entries/2026/05/05/tests-test_sync_agent.md



---

**Generated:** 2026-05-08
**Source:** 23 entries from entries/
**Model:** claude

### [ACCEPT] review-evaluates-direct-antecedents-only
`review-beliefs` presents each derived belief with only its direct antecedents to the LLM reviewer, not the transitive chain — producing occasional sufficiency false positives when support exists one level deeper, which must be triaged manually.
- Source: entries/2026/05/05/review-beliefs-in-practice.md

### [REJECT] deep-derivations-accumulate-generalization-error
Derived beliefs at depth 8-15 accumulate generalization error through successive scope expansion at each derivation step — individually reasonable generalizations compound into conclusions unsupported by original premises, with retracting one invalid belief at depth 8 cascading 52 dependents OUT.
- Source: entries/2026/05/05/review-beliefs-in-practice.md
- Rejected: speculative: observed LLM behavior pattern with ephemeral depth/cascade counts, not a code invariant

### [ACCEPT] review-and-contradictions-catch-orthogonal-errors
`review-beliefs` catches invalid reasoning within individual derivation steps (over-generalization, missing bridges, strength escalation) while `contradictions` catches incompatible facts across independently valid beliefs (absolute claims vs. documented exceptions) — the two commands are complementary with different cascade profiles (review: high cascade at depth, contradictions: low cascade at leaves).
- Source: entries/2026/05/06/contradictions-first-run.md

### [REJECT] contradictions-shuffles-before-batching
The `contradictions` command randomly shuffles all IN beliefs before partitioning into batches of 50, ensuring that repeated runs probabilistically cover cross-belief comparisons that fixed sequential batching would never surface.
- Source: entries/2026/05/06/contradictions-first-run.md
- Rejected: duplicate: already exists as `contradictions-shuffles-before-batching` [IN] and `contradictions-shuffle-prevents-deterministic-batching` [IN]

### [REJECT] check-stale-detects-source-staleness-only
`check-stale` detects source staleness (source file changed on disk) via hash comparison but cannot detect world staleness (the belief is outdated even though the source file hasn't changed) — beliefs can silently decay without any artifact triggering re-examination.
- Source: entries/2026/05/06/belief-staleness-in-business-planning.md
- Rejected: duplicate: already exists as `check-stale-detects-source-staleness-only` [IN]

### [REJECT] derive-review-contradict-is-convergent-cycle
The full belief maintenance cycle is `derive --exhaust → review-beliefs → retract invalid → contradictions → retract/fix/file → derive --exhaust`, where each tool catches a different error class (derive generates, review validates individual steps, contradictions finds cross-belief inconsistencies) and the cycle converges toward a stable belief set.
- Source: entries/2026/05/06/contradictions-first-run.md
- Rejected: meta: describes belief network maintenance workflow, not codebase behavior

### [REJECT] absolute-claims-are-primary-contradiction-source
Most contradictions detected by the `contradictions` command involve a belief making an absolute claim ("all", "never", "sole", "every") when another belief documents a specific exception — imprecise premises (fixable via `reasons update`) are a secondary category.
- Source: entries/2026/05/06/contradictions-first-run.md
- Rejected: speculative: observed pattern in contradiction results from a particular run, not a code invariant

### [REJECT] scope-over-generalization-is-primary-derivation-failure
The most common invalid derivation pattern is scope over-generalization — claiming "all X" or "every X" when antecedents establish only specific instances — followed by conjunction distribution fallacy, strength escalation, causal overclaim, and missing bridge assumptions.
- Source: entries/2026/05/06/full-review-beliefs-results.md


## Proposed Beliefs
- Rejected: speculative: observed LLM behavior pattern, not a code invariant

---

### From `entries/2026/05/06/pageindex-and-tms-integration.md`

### [REJECT] belief-source-metadata-is-file-level
Belief source tracking uses flat file-level pointers (source path + source_hash) with no structural awareness of where within a document the claim originated — section, page, or paragraph level provenance is not supported.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md
- Rejected: duplicate: already exists as `belief-source-metadata-is-file-level` [IN]

### [REJECT] expert-pipeline-extracts-per-document
The expert-agent-builder pipeline extracts beliefs per-document (summarize entire document → propose beliefs → record file path), not per-section — the connection between a belief and its source material is a file-level pointer, not a section-level one.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md
- Rejected: meta: describes the knowledge base construction pipeline, not the target codebase

### [REJECT] pageindex-hierarchical-rag
PageIndex is a separate tool, not part of the ftl-reasons codebase — claims about its architecture belong in a PageIndex knowledge base, not here.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md

### [REJECT] section-level-belief-extraction
This is a speculative integration design, not a testable claim about current code.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md

---

### From `entries/2026/05/08/pyproject.md`

### [REJECT] reasons-cli-entry-point
The `reasons` CLI command is a setuptools console_scripts entry point registered in pyproject.toml that calls `reasons_lib.cli:main`.
- Source: entries/2026/05/08/pyproject.md
- Rejected: trivial: standard setuptools console_scripts visible directly from pyproject.toml

### [REJECT] setuptools-explicit-packages
Only `reasons_lib/` is included in the distribution via explicit `[tool.setuptools] packages` configuration; `tests/`, `entries/`, `reviews/`, and other directories are excluded from packaging.
- Source: entries/2026/05/08/pyproject.md
- Rejected: trivial: packaging inclusion list directly readable from pyproject.toml

### [ACCEPT] python-version-floor-3-10
The project requires Python >=3.10, allowing use of match statements, union type syntax with `|`, and other 3.10+ features throughout `reasons_lib`.
- Source: entries/2026/05/08/pyproject.md

### [ACCEPT] extras-map-one-to-one-to-modules
Each optional dependency group maps 1:1 to a specific module: `pg` extra gates `reasons_lib/pg.py`, `cluster` extra gates `reasons_lib/cluster.py`, and `test-pg` is a superset combining `pg` and `test`.
- Source: entries/2026/05/08/pyproject.md

### [REJECT] ftl-reasons-zero-runtime-deps
Duplicate of existing belief `zero-runtime-dependencies`.
- Source: entries/2026/05/08/pyproject.md

---

### From `entries/2026/05/08/reasons_lib-api.md`

### [ACCEPT] fts-relaxation-capped-at-fifty-queries
`_fts_search` caps progressive term relaxation at 50 FTS5 queries to prevent combinatorial explosion on long search inputs, dropping terms one at a time via `combinations` down to `len(terms) // 2`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [ACCEPT] fts-stop-words-filtered-before-query
FTS queries filter a `_STOP_WORDS` frozenset before querying FTS5; if all terms are stop words, the search falls back to terms longer than 1 character.
- Source: entries/2026/05/08/reasons_lib-api.md

### [REJECT] namespace-resolve-preserves-cross-refs
`_resolve_namespace` only prefixes a node ID with `namespace:` when the ID contains no colon — cross-namespace references like `other-agent:some-belief` are left intact.
- Source: entries/2026/05/08/reasons_lib-api.md
- Rejected: duplicate: same claim as `colon-means-already-namespaced` which already states colon means already-namespaced and never double-prefixes

### [ACCEPT] search-has-four-output-formats
The `search` function supports four output formats: markdown, json, minimal, and compact — selected by the caller to match the consumption context (human, API, LLM prompt, compact view).
- Source: entries/2026/05/08/reasons_lib-api.md

### [REJECT] api-transaction-per-call
Covered by existing belief `api-layer-ensures-atomic-isolated-mutations`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [REJECT] api-dict-return-contract
Covered by existing belief `api-functions-return-dicts`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [REJECT] api-lazy-import-pattern
Covered by existing belief `api-uses-lazy-imports`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [REJECT] dedup-retains-most-connected
Covered by existing belief `belief-replacement-is-topology-safe-and-view-consistent` which states "deduplication rewires all justification references to the most-connected survivor."
- Source: entries/2026/05/08/reasons_lib-api.md

---

### From `entries/2026/05/08/reasons_lib-cli.md`

### [REJECT] derive-one-round-return-protocol
`_derive_one_round` returns positive for beliefs added, 0 for network-saturated, and negative for errors; `cmd_derive`'s `--exhaust` loop depends on this sentinel contract for termination.
- Source: entries/2026/05/08/reasons_lib-cli.md
- Rejected: duplicate: same return protocol already captured by `derive-returns-negative-one-on-error` and `derive-round-returns-negative-on-error`

### [REJECT] cluster-and-sample-are-mutually-exclusive
The `--cluster` and `--sample` flags in `cmd_derive` are mutually exclusive, enforced with an explicit check and `sys.exit(1)` — they represent competing strategies for belief subset selection.
- Source: entries/2026/05/08/reasons_lib-cli.md
- Rejected: duplicate: exact match exists as `cluster-and-sample-are-mutually-exclusive` [IN]

### [ACCEPT] derive-reports-survive-partial-runs
`cmd_derive` and `cmd_review_beliefs` write partial JSON reports after each round/batch via `_write_derive_report`, so crash recovery is possible from the last completed step.
- Source: entries/2026/05/08/reasons_lib-cli.md

### [REJECT] cli-dispatch-uses-commands-dict
`main()` routes subcommands through a `commands` dict mapping subcommand strings to `cmd_*` functions, rather than using argparse's `set_defaults(func=...)` — keeping all routing visible in one place.
- Source: entries/2026/05/08/reasons_lib-cli.md
- Rejected: duplicate: same claim as `cli-dispatch-is-flat-dict-lookup` which already describes flat commands dict vs set_defaults pattern

### [REJECT] cli-delegates-all-logic-to-api
Duplicate of existing belief `cli-is-pure-formatter`.
- Source: entries/2026/05/08/reasons_lib-cli.md

### [REJECT] cli-lazy-imports-heavy-deps
Duplicate of existing belief `cli-uses-lazy-imports-for-heavy-modules`.
- Source: entries/2026/05/08/reasons_lib-cli.md

### [REJECT] visible-to-threading
Covered by existing belief `access-control-is-transitive-subset-gated`.
- Source: entries/2026/05/08/reasons_lib-cli.md

---

### From `entries/2026/05/08/reasons_lib-cluster.md`

### [REJECT] cluster-beliefs-respects-budget
`cluster_beliefs` never returns more IDs than the `budget` parameter; each cluster's allocation is individually capped by `min(alloc, len(members))`.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-beliefs-respects-budget` [IN]

### [REJECT] cluster-cache-keys-include-content-hash
`ClusterCache` keys embeddings by `(node_id, sha256_prefix)`, so editing a belief's text with the same ID forces re-embedding rather than serving stale vectors.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-cache-keys-include-content-hash` [IN]

### [REJECT] cluster-deps-are-optional
`sentence-transformers` and `scikit-learn` are optional dependencies behind a `HAS_CLUSTER_DEPS` gate; the module degrades to a clear `ImportError` with install instructions when they are absent.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-deps-are-optional` [IN]

### [REJECT] cluster-embed-order-is-deterministic
Beliefs are sorted by ID before embedding (`ids = sorted(beliefs.keys())`), making cluster assignments reproducible given the same random seed.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-embed-order-is-deterministic` [IN]

### [REJECT] cluster-remainder-favors-largest
When the budget doesn't divide evenly across clusters, extra slots are distributed one-per-cluster to the largest clusters first via descending size sort.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-remainder-favors-largest` [IN]

### [REJECT] cluster-auto-k-heuristic
Auto cluster count is computed as `len(beliefs) // 5`, clamped between 2 and `min(budget // 3, 20)`, targeting approximately 5 beliefs per cluster with at least 3 beliefs per cluster given the budget.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-auto-k-heuristic` [IN]

### [REJECT] cluster-skips-ml-when-under-budget
When the number of beliefs is less than or equal to the budget, all ML work (embedding, clustering, sampling) is skipped and every belief is returned directly.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-skips-ml-when-under-budget` [IN]

---

**Summary:** 21 proposed beliefs — 17 ACCEPT, 4 REJECT (duplicates of existing beliefs).


## Proposed Beliefs

---

### entries/2026/05/08/reasons_lib-contradictions.md

### [REJECT] contradictions-only-checks-in-beliefs
`detect_contradictions` filters all input to `truth_value == "IN"` before processing; OUT beliefs are never sent to the LLM for contradiction checking.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-only-checks-in-beliefs` [IN]

### [REJECT] contradictions-shuffle-prevents-deterministic-batching
Belief IDs are randomly shuffled before batching so that repeated runs cover different pairwise combinations across batch boundaries, increasing cross-batch contradiction coverage.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-shuffle-prevents-deterministic-batching` [IN]

### [REJECT] contradictions-min-two-claims-per-nogood
The contradiction parser enforces that every returned nogood has at least 2 valid claim IDs; single-claim or empty results are silently dropped.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-min-two-claims-per-nogood` [IN]

### [REJECT] contradictions-no-storage-dependency
`contradictions.py` operates on in-memory node dicts and has no import of `storage.py` or any database layer, making it testable in full isolation.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-no-storage-dependency` [IN]

### [REJECT] contradictions-cross-batch-pairs-undetected
Batch boundaries are non-overlapping — each belief appears in exactly one batch per run — so contradictions between beliefs in different batches cannot be detected in a single run.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-cross-batch-pairs-undetected` [IN]

### [REJECT] contradictions-swallows-batch-errors
Duplicates the existing system-wide belief `batch-fault-isolation-is-universal-across-llm-operations` which already covers per-batch error containment across all LLM-facing modules.
- Source: entries/2026/05/08/reasons_lib-contradictions.md

---

### entries/2026/05/08/reasons_lib-derive.md

### [REJECT] derive-parse-proposals-backward-compat
`parse_proposals` tries the new `### DERIVE id` format first and falls back to the legacy `### DERIVE: \`id\`` format with bold markdown labels, so it can consume outputs from any version.
- Source: entries/2026/05/08/reasons_lib-derive.md
- Rejected: duplicate: identical to existing `derive-parse-supports-two-format-versions` and `derive-parse-supports-two-formats`

### [REJECT] derive-validate-rejects-similar-retracted
`validate_proposals` rejects any proposed belief whose tokenized ID has ≥50% Jaccard overlap with an existing OUT belief, preventing re-derivation of retracted conclusions.
- Source: entries/2026/05/08/reasons_lib-derive.md
- Rejected: duplicate: identical to existing `derive-retraction-guard-uses-jaccard`

### [REJECT] derive-apply-fail-soft
Duplicates the existing belief `derive-apply-isolates-per-proposal-errors` which already captures that `apply_proposals` catches exceptions per-proposal.
- Source: entries/2026/05/08/reasons_lib-derive.md

### [REJECT] derive-budget-three-strategies
`_build_beliefs_section` supports three budget strategies — alphabetical truncation (default), random sampling (`sample=True`), and semantic clustering (`cluster=True`) — all controlled by a single `max_beliefs` parameter.
- Source: entries/2026/05/08/reasons_lib-derive.md
- Rejected: duplicate: exact match exists as `derive-budget-three-strategies` [IN]

### [ACCEPT] derive-validate-rejects-duplicate-ids
`validate_proposals` rejects any proposal whose belief ID already exists in the network, preventing overwrites of existing beliefs through the derive pipeline.
- Source: entries/2026/05/08/reasons_lib-derive.md

---

### entries/2026/05/08/reasons_lib-import_agent.md

### [REJECT] import-agent-inactive-always-in-outlist
Duplicates the existing belief `kill-switch-uses-outlist-not-antecedent` which already captures that agent kill-switch nodes are placed in outlists.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [REJECT] import-agent-inactive-never-in-antecedents
Duplicates the existing belief `kill-switch-uses-outlist-not-antecedent` — the "not antecedent" half is already captured.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [REJECT] retracted-metadata-blocks-recompute
Duplicates the existing belief `metadata-actively-governs-truth-propagation` which already captures that metadata fields (including `_retracted`) govern propagation behavior.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [ACCEPT] import-agent-topo-sort-breaks-cycles
`_topo_sort_claims` breaks dependency cycles by appending all remaining unsorted nodes after `max_passes = len(remaining) + 1` iterations rather than raising an error, ensuring import always completes.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [REJECT] import-agent-deferred-retraction
Nodes that should be OUT are first created as IN (so they participate in dependency registration), collected in `retract_after`, and retracted only after `recompute_all()` — ordering matters for correct graph construction.
- Source: entries/2026/05/08/reasons_lib-import_agent.md
- Rejected: duplicate: same claim as existing `deferred-retraction-ordering`

---

### entries/2026/05/08/reasons_lib-review.md

### [REJECT] review-parse-defaults-safe
When the LLM response is missing fields, `parse_review_response` defaults to `valid=True`, `sufficient=True`, `necessary=True` — a parse error never generates a false-positive validity warning.
- Source: entries/2026/05/08/reasons_lib-review.md
- Rejected: duplicate: covered by parse-review-response-never-raises which states "missing boolean fields defaulting to `True`"

### [REJECT] review-batch-failure-non-fatal
Duplicates the existing belief `review-batch-failure-is-silent-skip` which already captures that failed batches are skipped with a warning.
- Source: entries/2026/05/08/reasons_lib-review.md

### [ACCEPT] review-multi-justification-disjunctive
The review prompt instructs the LLM that a belief is valid if ANY of its justifications is sound, matching Doyle's TMS disjunctive support semantics.
- Source: entries/2026/05/08/reasons_lib-review.md

### [REJECT] review-result-schema-normalized
Duplicates the existing belief `review-result-schema-is-normalized` under a slightly different ID.
- Source: entries/2026/05/08/reasons_lib-review.md

---

### entries/2026/05/08/tests-test_api.md

### [REJECT] api-tests-one-class-per-function
Test organization pattern, not a code invariant — violating it wouldn't break functionality.
- Source: entries/2026/05/08/tests-test_api.md

### [REJECT] api-retract-cascade-symmetry
Retracting a root node and re-asserting it must produce identical `changed` sets, verified end-to-end by `TestEndToEnd.test_retract_and_restore_chain`.
- Source: entries/2026/05/08/tests-test_api.md
- Rejected: duplicate: existing `api-cascade-symmetry-tested` already captures this invariant

### [ACCEPT] list-negative-batches-at-40
`api.list_negative` splits candidates into batches of 40, so 120 keyword-matching nodes produce exactly 3 LLM calls.
- Source: entries/2026/05/08/tests-test_api.md

### [ACCEPT] fts-progressive-relaxation
When a multi-term FTS query returns no results, the search engine progressively drops terms until it finds matches or exhausts all subsets.
- Source: entries/2026/05/08/tests-test_api.md

### [ACCEPT] update-node-preserves-justifications
`api.update_node` modifies text and source metadata without altering the node's justification list or truth value.
- Source: entries/2026/05/08/tests-test_api.md


### [REJECT] cli-exit-code-contract-zero-or-one
Every CLI command returns exit code 0 on success and 1 on error; no other exit codes are used across the entire test suite (~90 tests).
- Source: entries/2026/05/08/tests-test_cli.md
- Rejected: duplicate: identical to existing `cli-exit-code-contract-is-binary`

### [REJECT] cli-tests-use-black-box-pattern
Duplicates existing `cli-tests-are-black-box-integration`.
- Source: entries/2026/05/08/tests-test_cli.md

### [REJECT] cli-tests-no-shared-db-state
Duplicates existing `each-cli-test-creates-isolated-db`.
- Source: entries/2026/05/08/tests-test_cli.md

### [REJECT] run-cli-catches-system-exit-only
Test harness implementation detail, not a production code invariant; the relevant production behavior is captured by `cli-exit-code-contract-zero-or-one`.
- Source: entries/2026/05/08/tests-test_cli.md

### [REJECT] cli-errors-route-to-stderr
Covered by existing `cli-is-deterministic-and-stream-correct`.
- Source: entries/2026/05/08/tests-test_cli.md

---

### [REJECT] cluster-beliefs-returns-exact-budget
`cluster_beliefs` returns exactly `budget` belief IDs when the input set is larger than the budget, and all items when the input set is smaller.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-beliefs-returns-exact-budget` [IN]

### [REJECT] cluster-beliefs-deterministic-with-seed
Given the same beliefs dict, budget, and seed, `cluster_beliefs` produces identical output across calls.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-beliefs-deterministic-with-seed` [IN]

### [REJECT] cluster-cache-no-recompute
`ClusterCache.embed()` does not recompute embeddings for previously cached belief texts; cache size stays constant on repeated calls with the same input and grows by exactly the count of new texts on superset calls.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-cache-no-recompute` [IN]

### [REJECT] cluster-stats-sizes-sum-to-input
The `cluster_sizes` list in the stats dict returned by `cluster_beliefs` always sums to the total number of input beliefs, enforcing that every belief is assigned to exactly one cluster.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-stats-sizes-sum-to-input` [IN]

### [REJECT] cluster-deps-optional-with-graceful-skip
The clustering module (`reasons_lib.cluster`) is behind an optional `[cluster]` install extra; when `sentence-transformers` or `scikit-learn` are missing, `_require_cluster_deps` raises `ImportError` and all dependent tests skip cleanly.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-deps-optional-with-graceful-skip` [IN]

---

### [REJECT] contradiction-min-two-claims
`parse_contradiction_response` drops any nogood with fewer than 2 valid claim IDs; this is enforced both before and after `valid_ids` filtering.
- Source: entries/2026/05/08/tests-test_contradictions.md
- Rejected: duplicate: already exists as `contradiction-min-two-claims` [IN]

### [REJECT] contradiction-in-only-filter
`detect_contradictions` excludes OUT nodes from LLM prompts even when explicitly listed in `belief_ids`.
- Source: entries/2026/05/08/tests-test_contradictions.md
- Rejected: duplicate: already exists as `contradiction-in-only-filter` [IN]

### [REJECT] contradiction-batch-fault-tolerance
Covered by existing `batch-fault-isolation-is-universal-across-llm-operations` and `llm-fault-tolerance-is-multi-granular`.
- Source: entries/2026/05/08/tests-test_contradictions.md

### [REJECT] contradiction-dry-run-overrides-auto-apply
When both `--dry-run` and `--auto-apply` are passed to the `contradictions` CLI subcommand, no nogoods are recorded in the database (applied count is 0).
- Source: entries/2026/05/08/tests-test_contradictions.md
- Rejected: duplicate: already exists as `contradiction-dry-run-overrides-auto-apply` [IN]

---

### [REJECT] validate-proposals-rejects-jaccard-similar-to-out
Covered by existing `derive-pipeline-has-end-to-end-quality-enforcement` which specifies Jaccard retraction guards preventing re-derivation of known-bad conclusions.
- Source: entries/2026/05/08/tests-test_derive.md

### [REJECT] dedup-rewrites-justifications-on-retract
Already exists as `dedup-rewrites-both-antecedents-and-outlist`.
- Source: entries/2026/05/08/tests-test_derive.md

### [REJECT] gate-belief-out-when-outlist-in
Covered by existing `absence-and-outlist-form-complete-negative-semantics` and `any-mode-preserves-full-outlist-semantics`.
- Source: entries/2026/05/08/tests-test_derive.md

### [ACCEPT] derive-report-json-has-rounds-array
The `reasons derive --auto --report-dir` command writes a JSON report containing a `rounds` array where each entry has `proposals_found` and `added` counts.
- Source: entries/2026/05/08/tests-test_derive.md

---

### [REJECT] derive-budget-sample-is-deterministic
Covered by existing `derive-prompt-is-deterministic-and-reproducible`.
- Source: entries/2026/05/08/tests-test_derive_budget.md

### [REJECT] derive-budget-tests-parse-header-format
Test infrastructure coupling detail (regex parsing of `_build_beliefs_section` headers), not a production code invariant.
- Source: entries/2026/05/08/tests-test_derive_budget.md


## Proposed Beliefs

---

### [ACCEPT] import-agent-out-beliefs-not-resurrected
Beliefs marked OUT or STALE in the source are imported as OUT and are never resurrected by `recompute_all`, even when their justification antecedents are all IN in the local database.
- Source: entries/2026/05/08/tests-test_import_agent.md

### [REJECT] import-agent-namespaces-all-beliefs
Duplicates existing `import-agent-namespace-prefix` and `namespace-is-colon-convention-with-auto-wiring`.
- Source: entries/2026/05/08/tests-test_import_agent.md

### [REJECT] import-agent-inactive-outlist-kill-switch
Duplicates existing `kill-switch-uses-outlist-not-antecedent` and `active-inactive-relay-pair`.
- Source: entries/2026/05/08/tests-test_import_agent.md

---

### [ACCEPT] review-only-validates-derived-beliefs
The review pipeline filters out premises (nodes with empty justifications) before sending anything to the LLM; premises are never submitted for review validation.
- Source: entries/2026/05/08/tests-test_review.md

### [ACCEPT] parse-review-response-never-raises
`parse_review_response` returns an empty list on any malformed input — bad JSON, non-list JSON, items missing `id` fields — rather than raising exceptions, with missing boolean fields defaulting to `True`.
- Source: entries/2026/05/08/tests-test_review.md

### [REJECT] review-result-priority-invalid-over-insufficient
Already exists as an accepted belief with this exact ID.
- Source: entries/2026/05/08/tests-test_review.md

### [ACCEPT] dry-run-prevents-both-retraction-and-metadata
The `dry_run` flag in review-beliefs prevents both truth-value changes (retraction of invalid beliefs) and metadata side-effects (`last_reviewed` timestamp, `review_result` classification), making it fully read-only — extending beyond what `auto-retract-respects-dry-run` covers.
- Source: entries/2026/05/08/tests-test_review.md

### [REJECT] llm-subprocess-mock-requires-two-patches
Testing infrastructure detail (which objects to patch), not a production code invariant. The underlying fact about `shutil.which` + `subprocess.run` is already captured by `all-external-execution-is-subprocess-isolated` and `invoke-claude-raises-when-binary-missing`.
- Source: entries/2026/05/08/tests-test_review.md

---

### [ACCEPT] sync-agent-preserves-cascade-structure
After `sync_agent` completes, the agent's outlist-based justification structure remains intact — revoking `agent:active` still cascades all agent beliefs to OUT, proving sync does not corrupt kill-switch wiring.
- Source: entries/2026/05/08/tests-test_sync_agent.md

### [REJECT] sync-agent-handles-both-formats
Covered by existing `format-resilience-spans-all-external-interfaces` and `import-sync-has-dual-reconciliation-modes`.
- Source: entries/2026/05/08/tests-test_sync_agent.md

### [ACCEPT] sync-agent-first-sync-equals-import
When no prior import exists for an agent, `sync_agent` behaves identically to `import_agent` — creating the `agent:active` premise and returning `created_premise: True` — rather than requiring a separate import step.
- Source: entries/2026/05/08/tests-test_sync_agent.md



---

**Generated:** 2026-05-11
**Source:** 24 entries from entries/
**Model:** claude

### [ACCEPT] pg-dispatch-is-function-level-early-return
PostgreSQL routing uses a function-level early-return pattern — each API function checks `pg_conninfo` and short-circuits to `_pg_dispatch`, which instantiates PgApi and calls the matching method via `getattr` — rather than using abstract base classes, subclassing, or factory patterns.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPT] export-markdown-pg-reconstructs-network
The `export_markdown` PostgreSQL path reconstructs a full `Network` object from `export_network()` output — creating Node/Justification objects and wiring the dependents index — because the markdown exporter requires a wired dependency graph, not a flat dict.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPT] sqlite-only-commands-are-filesystem-llm-or-bulk
The ~21 SQLite-only CLI commands fall into three categories: filesystem-dependent (hash-sources, check-stale, add-repo), LLM-powered (derive, review-beliefs, detect-contradictions, ask), and bulk import/sync — all requiring either local file access or load-entire-network-modify-save semantics incompatible with PgApi's per-operation transaction model.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPT] pg-unsupported-params-raise-not-implemented
When PgApi-routed commands receive unsupported parameters (e.g., search with `depth != 1`, list with `challenged`/`min_depth`, add with `namespace`), the dispatch raises `NotImplementedError` with a clear message rather than silently ignoring the parameter.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPT] pg-conninfo-accepts-flag-envvar-or-both
PostgreSQL connection is configured via `--pg`/`--project-id` CLI flags or `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables, with CLI flags taking precedence over env vars — `_backend_kwargs(args)` handles the dispatch.
- Source: entries/2026/05/10/postgresql-backend-support.md

---

### [ACCEPT] semantic-contradictions-cluster-before-llm
The `--semantic` flag on `detect-contradictions` embeds beliefs via sentence-transformers, clusters with KMeans via `list_clusters()`, and sends each cluster to the LLM as a batch — ensuring topically related beliefs are analyzed together instead of being scattered across random batches of 50.
- Source: entries/2026/05/10/semantic-contradiction-detection.md

### [ACCEPT] semantic-contradiction-reuses-cluster-infrastructure
Semantic contradiction detection delegates embedding and clustering to the existing `list_clusters()` from `reasons_lib/cluster.py` — the same infrastructure that serves deduplication also serves contradiction detection, with no duplicate embedding/clustering implementation.
- Source: entries/2026/05/10/semantic-contradiction-detection.md

### [ACCEPT] semantic-contradiction-skips-singleton-clusters
Single-belief clusters are skipped during semantic contradiction detection (no contradiction possible within one belief), and clusters exceeding `CONTRADICTION_BATCH_SIZE` are sub-batched within the cluster boundary.
- Source: entries/2026/05/10/semantic-contradiction-detection.md

---

### [ACCEPT] ftl-reasons-zero-runtime-deps
The core `reasons_lib` package has no mandatory runtime dependencies — all external packages (psycopg, sentence-transformers, scikit-learn, mcp) are gated behind optional install extras (`[pg]`, `[cluster]`, `[mcp]`).
- Source: entries/2026/05/11/pyproject.md

### [ACCEPT] reasons-cli-entrypoint-is-cli-main
The `reasons` CLI command is registered as `reasons_lib.cli:main` via `[project.scripts]` in pyproject.toml — changing that function's signature or module location breaks the installed command.
- Source: entries/2026/05/11/pyproject.md

### [ACCEPT] only-reasons-lib-is-distributed
Only the `reasons_lib` package is included in built distributions — `tests/`, `entries/`, and `reviews/` are excluded by the explicit `packages = ["reasons_lib"]` declaration in pyproject.toml.
- Source: entries/2026/05/11/pyproject.md

### [ACCEPT] python-310-minimum
The project requires Python >= 3.10 (`requires-python = ">=3.10"`), enabling use of structural pattern matching and `X | Y` union type syntax throughout the codebase.
- Source: entries/2026/05/11/pyproject.md

---

### [ACCEPT] fts-relaxation-budget-caps-at-50
Progressive FTS5 search relaxation — dropping query terms via combinations when the full-term query returns no results — is capped at `_MAX_RELAXATION_QUERIES` (50) to prevent combinatorial blowup on many-term queries.
- Source: entries/2026/05/11/reasons_lib-api.md

### [REJECT] namespace-colon-convention-enables-cross-refs
`_resolve_namespace` only prefixes belief IDs that don't already contain `:` — the colon serves as both the namespace delimiter and the signal that no prefixing is needed, allowing cross-namespace references to pass through unchanged.
- Source: entries/2026/05/11/reasons_lib-api.md
- Rejected: duplicate: same claim as `colon-means-already-namespaced` which already captures that colon means already-namespaced and no double-prefixing occurs

### [REJECT] access-tags-require-subset-for-visibility
A node is visible only when all of its access tags are present in the caller's `visible_to` set (subset semantics); nodes with empty access tags are always visible. `show_node`/`explain_node` raise `PermissionError` on violation while list operations silently filter.
- Source: entries/2026/05/11/reasons_lib-api.md
- Rejected: duplicate: same claim as `access-tags-subset-gate` and `access-control-is-transitive-subset-gated`

### [REJECT] with-network-is-sole-db-lifecycle-manager
Every API function accesses the SQLite database exclusively through `_with_network(db_path, write=False)` — a context manager that loads a Network from Storage, yields it, and saves only on clean exit when `write=True`. No API function manages Storage directly.
- Source: entries/2026/05/11/reasons_lib-api.md


## Proposed Beliefs
- Rejected: duplicate: same claim as `api-uses-with-network-context-manager` which already captures the context-manager pattern for load-operate-save atomicity

---

### Entry: `reasons_lib/ask.py`

### [REJECT] ask-tool-loop-max-3-rounds
Duplicates existing `ask-tool-loop-capped-at-three`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [REJECT] ask-extract-tool-call-is-line-based-json
Duplicates existing `ask-uses-text-based-tool-protocol` and `extract-tool-call-returns-first-match`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [REJECT] ask-llm-failure-returns-raw-context
Duplicates existing `ask-falls-back-to-raw-search` and `ask-never-raises-on-llm-failure`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [REJECT] ask-natural-strips-metadata-and-citations
Duplicates existing `ask-natural-mode-strips-metadata-and-cite`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [ACCEPT] ask-stop-word-fallback-ensures-nonempty-query
`_search_source_chunks` strips stop words from the question before building the FTS5 query, but falls back to all words longer than 1 character if every word is a stop word — ensuring the query is never empty
- Source: entries/2026/05/11/reasons_lib-ask.md

---

### Entry: `reasons_lib/cli.py`

### [REJECT] cli-delegates-all-logic-to-api
Every `cmd_*` function in the CLI delegates to `reasons_lib.api` and contains no business logic, database access, or network manipulation — the CLI is a pure presentation/argument-parsing layer
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: same claim as `cli-is-pure-formatter` which already states every cmd_* delegates to api.* with no business logic

### [ACCEPT] cli-backend-kwargs-controls-storage
`_backend_kwargs(args)` is the single chokepoint that determines whether a command runs against SQLite or PostgreSQL; every `cmd_*` function must spread its return value into the corresponding `api.*` call
- Source: entries/2026/05/11/reasons_lib-cli.md

### [REJECT] cli-heavy-imports-are-deferred
Duplicates existing `cli-uses-lazy-imports-for-heavy-modules`
- Source: entries/2026/05/11/reasons_lib-cli.md

### [ACCEPT] cli-sqlite-only-commands-exist
Commands `derive`, `ask`, `review-beliefs`, `deduplicate`, and `contradictions` are guarded by `_require_sqlite()` and exit with an error if `--pg` is set — they do not support PostgreSQL
- Source: entries/2026/05/11/reasons_lib-cli.md

### [ACCEPT] cli-plan-review-apply-pattern
Several commands (`derive`, `deduplicate`, `contradictions`) follow a three-phase workflow: (1) generate proposals to a file, (2) human reviews/edits the file, (3) `--accept FILE` parses and applies the reviewed plan — with `--auto` collapsing all three phases
- Source: entries/2026/05/11/reasons_lib-cli.md

### [REJECT] cli-check-stale-exits-1-on-stale
`cmd_check_stale` exits with code 1 when stale nodes are found, making it usable as a CI gate that fails the build on belief staleness
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: same claim as existing `check-stale-exits-nonzero`

---

### Entry: `reasons_lib/cluster.py`

### [REJECT] cluster-cache-key-includes-text-hash
`ClusterCache` keys embeddings by `(node_id, sha256(text)[:16])`, so modified belief text invalidates the cached embedding without requiring explicit cache eviction
- Source: entries/2026/05/11/reasons_lib-cluster.md
- Rejected: duplicate: same claim as existing `cluster-cache-keys-include-content-hash`

### [REJECT] cluster-auto-k-floors-at-two
`_auto_k` always returns at least 2 clusters (when auto-computed), ensuring cross-domain sampling is meaningful even for small belief sets
- Source: entries/2026/05/11/reasons_lib-cluster.md
- Rejected: duplicate: floor of 2 already captured in existing `cluster-auto-k-heuristic` ("clamped between 2 and...")

### [REJECT] cluster-budget-is-hard-ceiling
`cluster_beliefs` never returns more than `budget` belief IDs; when the belief set is smaller than budget, it returns all beliefs without clustering
- Source: entries/2026/05/11/reasons_lib-cluster.md
- Rejected: duplicate: same claim as existing `cluster-beliefs-respects-budget` and `cluster-beliefs-returns-exact-budget`

### Entry: `reasons_lib/contradictions.py`

### [REJECT] contradictions-require-two-claims-minimum
Duplicates existing `contradictions-min-two-claims-per-nogood`
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [REJECT] contradictions-batch-errors-never-propagate
Covered by existing `batch-fault-isolation-is-universal-across-llm-operations`
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [REJECT] contradictions-shuffle-for-coverage
Duplicates existing `contradictions-shuffle-prevents-deterministic-batching`
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [ACCEPT] contradictions-semantic-skips-singleton-clusters
`detect_contradictions_semantic` skips any cluster containing fewer than 2 beliefs, since no pairwise contradiction is possible within a singleton
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [ACCEPT] contradictions-belief-text-truncated-at-200-chars
`format_beliefs_for_contradiction_check` truncates each belief's text at 200 characters in the LLM prompt to avoid blowing context windows on large belief descriptions
- Source: entries/2026/05/11/reasons_lib-contradictions.md

---

### Entry: `reasons_lib/derive.py`

### [ACCEPT] derive-prompt-round-trippable
`write_proposals_file` output can be parsed back by `parse_proposals` with no data loss, enabling a write-review-accept cycle where humans edit proposals in the same format the parser reads
- Source: entries/2026/05/11/reasons_lib-derive.md

### [ACCEPT] derive-validate-blocks-retracted-rediscovery
`validate_proposals` rejects any proposal whose ID has >= 50% Jaccard token overlap (tokenized on hyphens/colons) with an existing OUT belief, preventing re-derivation of previously retracted conclusions
- Source: entries/2026/05/11/reasons_lib-derive.md

### [REJECT] derive-apply-is-fault-tolerant
Duplicates existing `derive-apply-isolates-per-proposal-errors`
- Source: entries/2026/05/11/reasons_lib-derive.md


## Proposed Beliefs

---

### entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPT] import-agent-inactive-always-in-outlist
Every imported belief unconditionally has `agent:inactive` in its outlist, enforced in `_build_justifications()` with no code path that omits it — this is the mechanism behind the per-agent kill switch.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPT] import-topo-sort-cycle-tolerant
Topological sorting of imported claims is best-effort: cycles cause remaining nodes to be appended in arbitrary order rather than raising an error, a pragmatic choice that avoids crashing on circular references.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [REJECT] import-agent-active-never-in-antecedents
Duplicate of existing `active-not-in-antecedents`.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [REJECT] sync-agent-remote-wins-semantics
Covered by existing `import-skips-existing-sync-is-remote-wins` which captures both modes.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [REJECT] retracted-flag-survives-recompute
Covered by existing `metadata-actively-governs-truth-propagation` and `metadata-governs-lifecycle-across-read-and-write-paths`.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

---

### entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPT] mcp-bridge-runs-dedicated-event-loop-thread
Each `McpBridge` instance runs its own asyncio event loop on a daemon thread; the MCP session stays alive until `close()` signals the shutdown event, bridging the sync/async boundary via `run_coroutine_threadsafe`.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPT] mcp-bridge-connect-blocks-up-to-30s
`connect()` blocks the calling thread for up to 30 seconds waiting for MCP session initialization via `_ready.wait(timeout=30)`, then raises `TimeoutError` if the server doesn't respond.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPT] mcp-bridge-call-tool-timeout-60s
`call_tool()` blocks for up to 60 seconds per invocation via `future.result(timeout=60)`; if the MCP server hangs, the calling thread unblocks with `TimeoutError`.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPT] mcp-bridge-tools-snapshot-at-connect
The tool catalog and server instructions are populated once during `connect()` and never refreshed — there is no mechanism to pick up tools added after the initial handshake.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPT] mcp-is-optional-dependency
The `mcp` package is guarded by try/except at import time; `_require_mcp()` defers the `ImportError` to `McpBridge` construction so the module can be imported unconditionally without the SDK installed.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

---

### entries/2026/05/11/reasons_lib-network.md

### [ACCEPT] network-disjunctive-justification
A node is IN if ANY of its justifications is valid (disjunction); each individual justification requires ALL antecedents IN and ALL outlist members OUT (conjunction) — the SL justification semantics from Doyle's 1979 paper.
- Source: entries/2026/05/11/reasons_lib-network.md

### [ACCEPT] network-retracted-nodes-persist
Retracted nodes remain in `self.nodes` with `_retracted` metadata set; they are never deleted from the graph, enabling later restoration via `assert_node()` without rederivation.
- Source: entries/2026/05/11/reasons_lib-network.md

### [ACCEPT] network-single-justification-removal-blocked
`remove_justification()` raises `ValueError` when a node has exactly one justification, forcing callers to use `convert_to_premise` or `retract` instead — preventing accidental creation of unjustified non-premise nodes.
- Source: entries/2026/05/11/reasons_lib-network.md

### [REJECT] network-propagate-skips-retracted
Covered by existing `metadata-actively-governs-truth-propagation` which states retracted nodes are skipped during BFS traversal.
- Source: entries/2026/05/11/reasons_lib-network.md

### [REJECT] network-entrenchment-prefers-premises
Covered by existing `add-nogood-retraction-prefers-least-entrenched` which captures the entrenchment scoring and premise preference.
- Source: entries/2026/05/11/reasons_lib-network.md

---

### entries/2026/05/11/reasons_lib-pg.md

### [ACCEPT] pg-psycopg-is-optional-dependency
`psycopg` (v3) is soft-imported with fallback to `None`; `_require_psycopg()` raises `ImportError` with install instructions at `PgApi` construction time, mirroring the `mcp_client.py` lazy-guard pattern.
- Source: entries/2026/05/11/reasons_lib-pg.md

### [ACCEPT] pg-search-includes-one-hop-neighbors
PostgreSQL full-text search via `plainto_tsquery` includes 1-hop neighbor expansion — direct antecedents and dependents of matching nodes are included in results, providing richer context than exact-match-only search.
- Source: entries/2026/05/11/reasons_lib-pg.md

### [ACCEPT] pg-uses-jsonb-with-gin-for-dependent-queries
PgApi stores antecedents and outlist as JSONB arrays with GIN indexes, enabling per-operation dependent lookups via `@>` containment queries — unlike the SQLite backend which loads the full network, PgApi queries relationships incrementally.
- Source: entries/2026/05/11/reasons_lib-pg.md

---

### entries/2026/05/11/tests-test_api.md

### [ACCEPT] api-fts-search-bounds-query-calls
`_fts_search` makes at most 51 internal `_fts_query` calls regardless of input query length, preventing combinatorial explosion from progressive relaxation on long queries.
- Source: entries/2026/05/11/tests-test_api.md

### [REJECT] api-retract-cascade-tested
Describes test coverage rather than an architectural invariant; the cascade behavior itself is already captured by retraction beliefs.
- Source: entries/2026/05/11/tests-test_api.md

### [REJECT] api-list-negative-graceful-on-bad-llm
Covered by existing `list-negative-is-defensively-bounded` and `llm-integration-fails-softly-across-modules`.
- Source: entries/2026/05/11/tests-test_api.md

### [REJECT] api-init-db-refuses-overwrite
Covered by existing `api-enforces-typed-preconditions` which captures typed exception enforcement at the API boundary.
- Source: entries/2026/05/11/tests-test_api.md

### [REJECT] api-list-gated-excludes-superseded
Covered by existing `supersession-is-reversible-and-view-consistent` which explicitly states superseded nodes are excluded from gated belief lists.
- Source: entries/2026/05/11/tests-test_api.md


## Proposed Beliefs from Test Suite Entries

---

### entries/2026/05/11/tests-test_ask.md

### [ACCEPT] ask-source-chunks-persist-across-tool-iterations
Source document content included in the initial ask prompt is re-included in all subsequent prompts after tool-call round-trips, ensuring the LLM retains source context across the entire multi-turn loop.
- Source: entries/2026/05/11/tests-test_ask.md

### [REJECT] ask-tool-loop-bounded-at-three
Covered by existing `ask-has-tiered-query-modes` which already states "bounded 3-iteration tool loop."
- Source: entries/2026/05/11/tests-test_ask.md

### [REJECT] ask-absorbs-llm-failures
Covered by existing `ask-never-raises-on-llm-failure` and `ask-falls-back-to-raw-search`.
- Source: entries/2026/05/11/tests-test_ask.md

### [REJECT] ask-natural-strips-all-metadata-fields
Covered by existing `ask-natural-mode-strips-metadata-and-cite`.
- Source: entries/2026/05/11/tests-test_ask.md

---

### entries/2026/05/11/tests-test_ask_mcp.md

### [ACCEPT] ask-mcp-errors-non-fatal
When an MCP bridge's `call_tool` raises an exception during the ask loop, `ask()` catches the error, feeds it back to the LLM as context, and continues the tool loop rather than propagating to the caller.
- Source: entries/2026/05/11/tests-test_ask_mcp.md

### [ACCEPT] ask-mcp-iteration-limit-is-five
When `mcp_servers` is non-empty, `ask()` allows up to 5 tool-call iterations before forcing a final LLM response (6 total invocations), compared to the lower limit without MCP servers.
- Source: entries/2026/05/11/tests-test_ask_mcp.md

### [ACCEPT] build-tools-section-always-includes-search-beliefs
`_build_tools_section` always includes the built-in `search_beliefs` tool in its output regardless of whether MCP bridges are provided — it is the baseline tool present in every ask prompt.
- Source: entries/2026/05/11/tests-test_ask_mcp.md

### [ACCEPT] ask-prompt-no-template-placeholders
`build_ask_prompt` never leaves `{{` or `}}` template markers in the generated prompt string — all placeholders are resolved before output.
- Source: entries/2026/05/11/tests-test_ask_mcp.md

### [REJECT] extract-tool-call-schema-agnostic
Covered by existing `extract-tool-call-returns-first-match` and `ask-uses-text-based-tool-protocol` which together establish that tool call parsing is generic.
- Source: entries/2026/05/11/tests-test_ask_mcp.md

---

### entries/2026/05/11/tests-test_cli.md

### [REJECT] cli-exit-code-zero-success-one-error
All CLI subcommands use exit code 0 for success and exit code 1 for every error condition (missing nodes, access denied, duplicate IDs, file not found); no other exit codes appear in the test suite.
- Source: entries/2026/05/11/tests-test_cli.md
- Rejected: duplicate: same claim as existing `cli-exit-code-contract-is-binary`

### [REJECT] cli-tests-use-blackbox-main-patching
Covered by existing `cli-tests-are-black-box-integration`.
- Source: entries/2026/05/11/tests-test_cli.md

### [REJECT] cli-tests-are-self-contained
Covered by existing `each-cli-test-creates-isolated-db`.
- Source: entries/2026/05/11/tests-test_cli.md

### [REJECT] cli-errors-go-to-stderr
Covered by existing `cli-is-deterministic-and-stream-correct` which addresses stdout/stderr routing.
- Source: entries/2026/05/11/tests-test_cli.md

### [REJECT] what-if-is-readonly
Covered by existing `pg-api-what-if-never-mutates`.
- Source: entries/2026/05/11/tests-test_cli.md

---

### entries/2026/05/11/tests-test_cluster.md

### [REJECT] auto-k-floor-is-two
`_auto_k` never returns fewer than 2 clusters regardless of input size (using the `n // 5` heuristic), unless clamped by `n` itself when there are fewer beliefs than clusters.
- Source: entries/2026/05/11/tests-test_cluster.md
- Rejected: duplicate: floor of 2 already captured in existing `cluster-auto-k-heuristic` ("clamped between 2 and...")

### [ACCEPT] list-clusters-partitions-all-beliefs
`list_clusters` assigns every input belief to exactly one cluster with no drops or duplicates — the union of all cluster members equals the input set.
- Source: entries/2026/05/11/tests-test_cluster.md

### [REJECT] cluster-deps-are-optional-extras
Covered by existing `cluster-deps-are-optional` and `cluster-deps-optional-with-graceful-skip`.
- Source: entries/2026/05/11/tests-test_cluster.md

### [REJECT] cluster-cache-is-incremental
Covered by existing `cluster-cache-no-recompute`.
- Source: entries/2026/05/11/tests-test_cluster.md

---

### entries/2026/05/11/tests-test_contradictions.md

### [REJECT] contradiction-detection-filters-out-nodes
`detect_contradictions` and `detect_contradictions_semantic` both exclude OUT-truth-value nodes before sending any prompt to the LLM, even if those node IDs are explicitly requested in the `belief_ids` parameter.
- Source: entries/2026/05/11/tests-test_contradictions.md
- Rejected: duplicate: same claim as existing `contradiction-in-only-filter` and `contradictions-only-checks-in-beliefs`

### [ACCEPT] contradiction-plan-round-trips-apply-entries
Writing a contradiction plan with `write_contradiction_plan` and parsing it back with `parse_contradiction_plan` preserves all `[APPLY]`-tagged NOGOOD entries with their IDs and claims, while discarding `[SKIP]`-tagged entries — enabling a human review workflow.
- Source: entries/2026/05/11/tests-test_contradictions.md

### [ACCEPT] belief-text-truncated-at-200-chars
`format_beliefs_for_contradiction_check` truncates any belief text longer than 200 characters, appending `...` as a suffix, to keep LLM prompts bounded.
- Source: entries/2026/05/11/tests-test_contradictions.md

### [REJECT] parse-contradiction-requires-two-valid-claims
Covered by existing `contradictions-min-two-claims-per-nogood`.
- Source: entries/2026/05/11/tests-test_contradictions.md

### [REJECT] batch-llm-failure-returns-empty-not-raises
Covered by existing `batch-fault-isolation-is-universal-across-llm-operations` and `llm-fault-tolerance-is-multi-granular`.
- Source: entries/2026/05/11/tests-test_contradictions.md


### [ACCEPT] validate-proposals-rejects-jaccard-similar-to-out
`validate_proposals` skips any proposal whose tokenized ID has >= 0.5 Jaccard similarity to an existing OUT belief, returning "similar to retracted" in the skip reason — preventing re-derivation of retracted beliefs under variant names
- Source: entries/2026/05/11/tests-test_derive.md

### [ACCEPT] jaccard-tokenizer-splits-on-hyphens-and-colons
`_tokenize_id` splits belief IDs on hyphens and colons into token sets for Jaccard similarity comparison, so `foo-bar-baz` and `foo-bar-qux` share 2/4 tokens (0.5 similarity)
- Source: entries/2026/05/11/tests-test_derive.md

### [REJECT] validate-proposals-partitions-not-raises
`validate_proposals` never raises exceptions — it partitions proposals into `(valid, skipped)` tuples where each skip carries a human-readable reason string
- Source: entries/2026/05/11/tests-test_derive.md
- Rejected: duplicate: same claim as existing `derive-fail-soft-validation` which already states validate_proposals filters into skipped list rather than raising

### [ACCEPT] build-prompt-validates-custom-templates
`build_prompt` raises `ValueError("unknown placeholder")` for unrecognized `{fields}` and `ValueError("malformed braces")` for unclosed braces in custom prompt templates
- Source: entries/2026/05/11/tests-test_derive.md

### [ACCEPT] apply-dedup-plan-collects-errors-not-raises
`apply_dedup_plan` collects errors into `result["errors"]` rather than raising, allowing partial application — one missing node does not block processing of the remaining dedup plan
- Source: entries/2026/05/11/tests-test_derive.md

### [REJECT] pg-tests-skip-without-postgres-connection
All tests in `test_pg.py` carry the `pytest.mark.pg` marker and `skip_no_pg` conditional skip — they are silently skipped when no PostgreSQL connection is available
- Source: entries/2026/05/11/tests-test_pg.md
- Rejected: trivial: standard pytest conditional skip via marker — obvious from the decorator itself

### [ACCEPT] pg-dispatch-requires-project-id
When `pg_conninfo` is provided (via CLI `--pg` flag or `REASONS_PG_CONNINFO` env var) without a corresponding `project_id`, the system calls `sys.exit()` rather than proceeding with an incomplete configuration
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPT] cli-flags-override-env-vars
`_backend_kwargs` gives precedence to CLI `--pg`/`--project-id` flags over `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables when both are present
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPT] import-functions-parse-before-dispatch
API import functions (`import_json`, `import_beliefs`, `import_agent`) read and parse files into structured data in the API layer before delegating to `PgApi` — PgApi never receives raw file paths
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPT] all-api-functions-support-pg-dispatch
Every public API function (`export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`) accepts `pg_conninfo`/`project_id` kwargs and routes to the corresponding `PgApi` method
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPT] require-sqlite-blocks-pg-for-guarded-commands
`_require_sqlite` enforces that certain subcommands (e.g., `hash-sources`) cannot run against a Postgres backend, checking both CLI flags and environment variables and calling `sys.exit()` if PG is detected
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPT] remove-justification-enforces-minimum-count
`Network.remove_justification` refuses to remove the last justification from a derived node — a derived node must retain at least one justification, enforced with `ValueError("only one justification")`
- Source: entries/2026/05/11/tests-test_remove_justification.md


