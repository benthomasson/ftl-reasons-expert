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

### [ACCEPT] storage-load-bypasses-propagation
`load()` constructs nodes directly into `network.nodes` rather than calling `add_node`, so truth maintenance propagation does not fire during deserialization.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [ACCEPT] justification-order-preserved-via-rowid
Justification insertion order is preserved across save/load cycles using `AUTOINCREMENT` rowid and `ORDER BY rowid` on read, which matters because justification priority affects truth maintenance.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [REJECT] dependents-index-derived-on-load
The `node.dependents` set is never persisted to SQLite; it is rebuilt by walking all justification antecedents and outlists during `load()`.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: already exists as belief `dependents-index-derived-on-load` (currently OUT)

### [ACCEPT] storage-trusts-stored-truth-values
`load()` trusts the stored `truth_value` without re-running propagation, making the database the source of truth for node status.
- Source: entries/2026/04/23/reasons_lib-storage.md

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

### [ACCEPT] multiple-outlist-is-conjunction
When a justification has multiple outlist entries, ALL must be OUT for the justification to be valid; any single outlist node going IN defeats the entire justification
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] outlist-relationships-survive-persistence
Outlists are stored as `outlist_json` in the SQLite `justifications` table; on load, the dependent index is rebuilt for both antecedents and outlist nodes, preserving propagation behavior across save/load cycles
- Source: entries/2026/04/23/topic-outlist-semantics.md



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

### [ACCEPT] derived-belief-soundness-is-llm-only
Structural validation ensures justification references exist and are IN, but the logical soundness of the inference from antecedents to derived conclusion is validated only by the proposing LLM — no code-level check verifies that the reasoning step is logically valid.
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md

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

### [ACCEPT] node-truth-disjunctive
A node is IN if ANY of its justifications is valid (disjunctive semantics); adding a justification to a node can never cause it to go OUT.
- Source: entries/2026/05/05/reasons_lib-__init__.md

### [REJECT] dependents-not-self-enforced
The `Node.dependents` reverse index is not maintained by the data model itself; `network.py` is responsible for keeping it consistent with justification references (antecedents and outlists).
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: same claim as existing `dependents-is-manual-reverse-index`

### [ACCEPT] data-model-uses-string-enums
Both `Justification.type` ("SL"/"CP") and `Node.truth_value` ("IN"/"OUT") are plain strings, not Python enums; consumers must validate values themselves as invalid states like `"MAYBE"` or `"XYZ"` are representable.
- Source: entries/2026/05/05/reasons_lib-__init__.md

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


### [ACCEPT] ask-dual-mode-makes-three-llm-calls
Dual mode makes up to 3 LLM calls (1 TMS synthesis + 1 FTS RAG + 1 merge), short-circuiting to 2 if either retrieval path returns empty.
- Source: entries/2026/05/05/reasons_lib-ask.md

### [REJECT] ask-tool-call-parsing-is-line-based-json
Tool calls are detected by scanning each line of the LLM response for valid JSON with a `"tool"` key; multi-line JSON objects are silently missed.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: core claim matches `ask-uses-text-based-tool-protocol`

### [ACCEPT] ask-sources-db-failure-silently-degrades
If the `sources_db` SQLite file is missing or corrupt, `_search_source_chunks` catches `OperationalError`/`DatabaseError` and returns empty string, degrading to belief-only mode without user-visible errors.
- Source: entries/2026/05/05/reasons_lib-ask.md

### [REJECT] ask-tool-loop-max-three-rounds
Covered by existing `ask-has-tiered-query-modes` which already describes the "bounded 3-iteration tool loop."
- Source: entries/2026/05/05/reasons_lib-ask.md

### [REJECT] ask-llm-failure-returns-raw-beliefs
Covered by existing `ask-falls-back-to-raw-search` and `ask-never-raises-on-llm-failure`.
- Source: entries/2026/05/05/reasons_lib-ask.md

### [ACCEPT] truncated-hash-threshold-is-16-chars
A stored `source_hash` of exactly 16 characters that is a prefix of the current full SHA-256 hash is classified as `truncated_hash` (legacy format), not `content_changed`.
- Source: entries/2026/05/05/reasons_lib-check_stale.md

### [ACCEPT] check-stale-and-hash-sources-mutate-in-place
Both `check_stale` (with `upgrade_hashes=True`) and `hash_sources` modify `node.source_hash` directly on the Network object; neither persists — the caller must save.
- Source: entries/2026/05/05/reasons_lib-check_stale.md

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

### [ACCEPT] commands-dict-must-mirror-subparsers
Adding a CLI subcommand requires entries in both the argparse subparser definitions and the `commands` dispatch dict in `main()`; omitting either silently breaks the command.
- Source: entries/2026/05/05/reasons_lib-cli.md

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

### [ACCEPT] sl-justification-is-disjunctive
A node is IN if *any* of its justifications is valid (disjunctive semantics); `_compute_truth()` short-circuits on the first valid justification rather than requiring all justifications to hold.
- Source: entries/2026/05/05/reasons_lib-network.md

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

### [ACCEPT] storage-load-bypasses-add-node
`load()` assigns nodes directly to `network.nodes` and calls `_rebuild_dependents()` afterward, deliberately skipping `add_node()` to avoid triggering truth maintenance propagation during state restoration.
- Source: entries/2026/05/05/reasons_lib-storage.md

### [ACCEPT] storage-justification-order-preserved
Justifications are inserted in list order and loaded via `ORDER BY rowid`, preserving the ordering that determines which justification is evaluated first during truth computation.
- Source: entries/2026/05/05/reasons_lib-storage.md

### [REJECT] storage-nogood-counter-persisted
Covered by existing `all-identifiers-are-durable-across-persistence-boundaries` which states "nogood IDs survive save/load cycles via the network_meta table's high-water mark."
- Source: entries/2026/05/05/reasons_lib-storage.md

### [ACCEPT] api-tests-use-real-sqlite
All API tests run against a real SQLite database with FTS5; storage is never mocked, ensuring the API contract includes correct SQL and full-text search behavior.
- Source: entries/2026/05/05/tests-test_api.md

### [ACCEPT] list-negative-batches-at-50
`list_negative` splits candidate nodes into batches of approximately 50 for LLM classification, verified by the test suite asserting exactly 3 LLM calls for 120 candidates.
- Source: entries/2026/05/05/tests-test_api.md

### [ACCEPT] fts-relaxation-bounded
Progressive FTS query relaxation is bounded: a 20-term query produces at most 51 `_fts_query` invocations, preventing unbounded search expansion on long input queries.
- Source: entries/2026/05/05/tests-test_api.md

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

### [ACCEPT] ask-dual-requires-sources-db
Calling `ask(..., dual=True)` without providing a `sources_db` path raises `ValueError`.
- Source: entries/2026/05/05/tests-test_ask.md

### [ACCEPT] ask-natural-mode-strips-metadata-and-cite
When `natural=True`, all belief metadata (`**Status:**`, `### ` headers, `**Source:**`) is stripped from the prompt context and the "Cite belief IDs" instruction is replaced with "plain natural language".
- Source: entries/2026/05/05/tests-test_ask.md

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

### [ACCEPT] auto-retract-respects-dry-run
The `--auto-retract` flag in the `review-beliefs` CLI is gated by `--dry-run`: when dry-run is active, findings are displayed but no database mutation occurs, even for beliefs flagged as invalid.
- Source: entries/2026/05/05/tests-test_review.md

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


