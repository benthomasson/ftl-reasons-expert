# Proposed Beliefs

Edit each entry: change `[ACCEPT]` to `[ACCEPT]` or `[ACCEPT]`.
Then run: `code-expert accept-beliefs`

---

**Generated:** 2026-04-23
**Source:** 17 entries from entries/
**Model:** claude

No existing beliefs. Here are the proposed beliefs extracted from the entries:

---

## From `reasons_lib/__init__.py`

### [ACCEPTED] node-in-if-any-justification-valid
A Node is IN when at least one of its Justifications is valid; a Justification is valid when all antecedents are IN and all outlist nodes are OUT (disjunctive over justifications, conjunctive within each).
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: same claim as proposed sl-justification-semantics with less precision

### [ACCEPTED] premises-have-no-justifications
A premise node is represented by an empty `justifications` list and defaults to `truth_value="IN"`; the system treats the empty-justifications case as a special unconditional belief.
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: split across existing challenge-destroys-premise-identity and proposed premise-defaults-to-in

### [ACCEPTED] init-is-pure-data-model
`reasons_lib/__init__.py` contains only dataclass definitions (`Node`, `Justification`, `Nogood`) with no behavior, validation, or I/O; it imports nothing from the project and sits at the bottom of the import graph.
- Source: entries/2026/04/23/reasons_lib-__init__.md

### [ACCEPTED] outlist-enables-non-monotonic-reasoning
The `outlist` field on `Justification` allows beliefs to be retracted when a defeating node becomes IN — this is the core non-monotonic mechanism powering `supersede`, `challenge`, and default-logic patterns.
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: covered by existing absence-and-outlist-form-complete-negative-semantics and challenge-is-outlist-injection

### [ACCEPTED] dependents-is-manual-reverse-index
`Node.dependents` is a denormalized reverse pointer set that must be kept in sync by external code (primarily `network.py`); nothing in the data model enforces consistency.
- Source: entries/2026/04/23/reasons_lib-__init__.md
- Rejected: duplicate: already exists as `dependents-is-manual-reverse-index` [OUT]

---

## From `reasons_lib/api.py`

### [ACCEPTED] api-functions-return-dicts
Every public API function returns a `dict` (or `str` for markdown/compact), never a `Network` or `Node` object, ensuring JSON-serializability at the boundary for CLI, HTTP, and tool-call consumers.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: already exists as `api-functions-return-dicts` [IN]

### [ACCEPTED] write-false-prevents-persistence
Functions using `_with_network(write=False)` can mutate the in-memory network (as `what_if_retract` does) but changes are never saved to SQLite; write-or-not is declared upfront and never conditional.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: Core invariant (what-if mutations are in-memory only, never persisted) is already captured by `both-backends-support-safe-hypothetical-reasoning`

### [ACCEPTED] namespace-active-premise-invariant
When `namespace` is set, `add_node` auto-creates a `{namespace}:active` premise node and wires it as an antecedent into every namespaced justification; retracting that single premise cascades OUT every belief from that namespace.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: covered by `active-inactive-relay-pair`, `active-not-in-antecedents`, `agent-isolation-through-namespace-and-relay`, and `import-agent-namespace-prefix`

### [ACCEPTED] any-mode-creates-per-premise-justifications
When `any_mode=True` and multiple antecedents are given, each antecedent gets its own SL justification (OR semantics: node is IN if *any* antecedent is IN), rather than the default single multi-antecedent justification (AND semantics).
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: already exists as `any-mode-creates-per-premise-justifications` [IN]

### [ACCEPTED] transaction-per-function
Every API function opens the database, does its work, and closes — no shared state, no connection pooling, no long-lived sessions; each invocation is fully independent.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: captured by `api-layer-ensures-atomic-isolated-mutations` ("per-function storage access") and `api-uses-with-network-context-manager`

### [ACCEPTED] colon-means-already-namespaced
`_resolve_namespace` treats a colon in a node ID as "already namespaced" and never double-prefixes; this is the convention for cross-namespace references.
- Source: entries/2026/04/23/reasons_lib-api.md
- Rejected: duplicate: already exists as `colon-means-already-namespaced` [IN]

---

## From `reasons_lib/check_stale.py`

### [ACCEPTED] check-stale-is-read-only
`check_stale` never mutates the network; it returns a list of stale-node dicts and leaves all nodes unchanged — staleness detection is separated from staleness resolution.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Rejected: duplicate: already exists as `check-stale-is-read-only` [IN]

### [ACCEPTED] check-stale-skips-out-nodes
Only nodes with `truth_value == "IN"` are checked for staleness; retracted (OUT) nodes are ignored even if their source file has changed.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Rejected: duplicate: already exists as `check-stale-skips-out-nodes` [IN]

### [ACCEPTED] check-stale-requires-both-source-fields
A node must have both `source` (non-empty) and `source_hash` (non-empty) to be eligible for staleness checking; nodes missing either field are silently skipped.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Rejected: duplicate: already exists as `check-stale-requires-both-source-fields` [IN]

### [ACCEPTED] hash-truncation-is-16-hex
STALE: Fixed in PR #40 — now uses full SHA-256 hash.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

### [ACCEPTED] missing-source-file-is-silent
STALE: Fixed in PR #32 (issue #25) — check_stale now reports source_deleted for missing files.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

---

## From `reasons_lib/cli.py`

### [ACCEPTED] cli-is-pure-formatter
Every `cmd_*` function delegates to `api.*` and only formats the returned dict for terminal output; no business logic lives in the CLI layer (sole exception: `cmd_propagate` touches `Storage` directly).
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `cli-is-pure-formatter` [IN]

### [ACCEPTED] check-stale-exits-nonzero
`cmd_check_stale` calls `sys.exit(1)` when any stale nodes are found, making it usable as a CI or pre-commit gate.
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `check-stale-exits-nonzero` [IN]

### [ACCEPTED] derive-strips-claudecode-env
`_derive_one_round` explicitly removes the `CLAUDECODE` environment variable before spawning the model subprocess, preventing recursive Claude Code invocation.
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `derive-strips-claudecode-env` [IN]

### [ACCEPTED] exhaust-implies-auto
In `_derive_one_round`, proposals are auto-applied when either `args.auto` or `args.exhaust` is true; the `--exhaust` flag does not require the user to also pass `--auto`.
- Source: entries/2026/04/23/reasons_lib-cli.md
- Rejected: duplicate: already exists as `exhaust-implies-auto` [IN]

---

## From `reasons_lib/compact.py`

### [ACCEPTED] compact-budget-only-limits-in-nodes
The token budget only constrains the IN nodes section; nogoods and OUT nodes are always emitted regardless of budget, so compact output can exceed the specified budget value.
- Source: entries/2026/04/23/reasons_lib-compact.md
- Rejected: duplicate: already exists as `compact-budget-only-limits-in-nodes` [OUT]

### [ACCEPTED] compact-summary-hiding-requires-in
A summary node only hides its covered nodes when the summary itself is IN; if the summary goes OUT, covered nodes reappear in the compact output.
- Source: entries/2026/04/23/reasons_lib-compact.md
- Rejected: duplicate: already exists as `compact-summary-hiding-requires-in` [IN]

### [ACCEPTED] compact-token-estimate-is-word-count
`estimate_tokens` counts whitespace-separated words, not BPE tokens; the budget parameter throughout the compact module is measured in this unit.
- Source: entries/2026/04/23/reasons_lib-compact.md
- Rejected: duplicate: already exists as OUT belief; contradicts IN beliefs `compact-estimate-tokens-chars-div-4` and `estimate-tokens-chars-div-4` which correctly state chars/4

### [ACCEPTED] compact-in-nodes-ordered-by-dependents
IN nodes are sorted by descending dependent count so structurally important nodes (those depended on by many others) are emitted first and survive budget truncation.
- Source: entries/2026/04/23/reasons_lib-compact.md


I'll review the entries and extract architectural/behavioral beliefs from each.
- Rejected: duplicate: already exists as `compact-in-nodes-ordered-by-dependents` [IN]

### [ACCEPTED] derive-retraction-guard-uses-jaccard
`validate_proposals` rejects any proposed belief ID with Jaccard similarity >= 0.5 to an existing OUT node, preventing re-derivation of retracted beliefs
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-retraction-guard-uses-jaccard` [IN]

### [ACCEPTED] derive-prompt-roundtrips-through-parser
The `### DERIVE` / `### GATE` format is a shared contract between `DERIVE_PROMPT` LLM output, `parse_proposals()` input, and `write_proposals_file()` output, forming a closed serialization loop
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-prompt-roundtrips-through-parser` [IN]

### [ACCEPTED] derive-depth-cycle-guard
`_get_depth` sets `memo[node_id] = 0` before recursing to prevent infinite recursion on cyclic justification chains; cycles resolve to depth 0
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-depth-cycle-guard` [IN]

### [ACCEPTED] derive-agent-budget-proportional
When agents are present, `_build_beliefs_section` allocates prompt token budget proportionally to each agent's belief count, with a floor of 5 beliefs per agent
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-agent-budget-proportional` [IN]

### [ACCEPTED] derive-fail-soft-validation
`validate_proposals` filters invalid proposals into a skipped list rather than raising; `apply_proposals` catches per-item exceptions so one bad proposal never blocks others
- Source: entries/2026/04/23/reasons_lib-derive.md
- Rejected: duplicate: already exists as `derive-fail-soft-validation` [IN]

### [ACCEPTED] derive-agent-count-bug
STALE: Fixed in PR #33 (issue #23).
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPTED] import-agent-namespace-prefix
Every node imported from agent X gets the ID prefix `X:`, including infrastructure nodes `X:active` and `X:inactive`, ensuring zero collision with local or other-agent beliefs
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: already exists as `import-agent-namespace-prefix` [IN]

### [ACCEPTED] kill-switch-uses-outlist-not-antecedent
The `agent:inactive` node is placed in each imported belief's outlist (not antecedents) so that retracting `agent:active` cascades all imported beliefs to OUT, while per-belief retraction still works independently
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: already exists; referenced by `import-agent-outlist-not-antecedent` [IN]

### [ACCEPTED] out-beliefs-imported-without-justifications
Beliefs that are OUT or STALE in the source are imported with an empty justification list, preventing `recompute_all` from resurrecting them to IN
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: covered by `import-handles-heterogeneous-truth-states` [IN]

### [ACCEPTED] sync-is-remote-wins
`_sync_claims` implements remote-wins reconciliation: remote text/metadata overwrites local, beliefs removed from remote are retracted locally, and beliefs remotely IN but locally OUT are re-asserted
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: covered by existing `import-skips-existing-sync-is-remote-wins`

### [ACCEPTED] import-two-phase-truth-maintenance
Import/sync adds all nodes first, then runs `recompute_all()` to propagate truth values, then performs explicit retractions — this ordering prevents incorrect cascades from partially-constructed networks
- Source: entries/2026/04/23/reasons_lib-import_agent.md
- Rejected: duplicate: covered by `deferred-retraction-ordering` [IN] and `import-achieves-ordered-convergent-reconciliation` [IN]

### [ACCEPTED] import-topo-sort-tolerates-cycles
`_topo_sort_claims` attempts topological ordering but appends remaining nodes when progress stalls, gracefully handling dependency cycles instead of erroring
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPTED] sl-outlist-asymmetry
Missing antecedents invalidate a justification, but missing outlist nodes do not — this asymmetry enables "believe X unless Y" where Y may not yet exist in the network
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [ACCEPTED] cp-equals-sl
CP (conditional-proof) justifications are evaluated with the exact same logic as SL justifications, despite being a distinct type in Doyle's TMS — either an intentional simplification or incomplete implementation
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Rejected: duplicate: already exists as `cp-equals-sl` [IN]

### [ACCEPTED] justification-valid-is-pure
`_justification_valid` is a pure query with no side effects, no logging, and no mutations to network state
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Rejected: trivial: expected behavior for a validation predicate

### [ACCEPTED] empty-antecedents-vacuously-valid
An SL justification with an empty antecedent list is valid (vacuous truth via `all([])`), allowing outlist-only justifications to function as "IN unless Y" — used by `challenge` and `supersede` for converted premises
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Rejected: duplicate: already exists as `empty-antecedents-vacuously-valid` [IN]

### [ACCEPTED] propagate-skips-retracted-nodes
`_propagate` never recomputes truth values for nodes with `_retracted` in metadata, even if their justifications would support IN; only `assert_node` can restore them
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPTED] propagate-does-not-change-trigger
The seed node (`changed_id`) is added to `visited` immediately and never has its own truth value recomputed; callers must update it before calling `_propagate`
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPTED] propagate-cascade-stops-on-unchanged
If a dependent's recomputed truth value equals its current value, it is not enqueued — the cascade terminates along that path, making propagation selective rather than exhaustive
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPTED] propagate-assumes-dependents-exist
STALE: Fixed in PR #27 (issue #22) — _propagate now guards against dangling dependent references.
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPTED] add-nogood-always-records
`add_nogood` appends a `Nogood` record unconditionally before checking whether the contradiction is active, so nogoods are preserved even when not all member nodes are currently IN
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [ACCEPTED] add-nogood-retraction-prefers-least-entrenched
The primary retraction path traces back through justification chains to premises and chooses the one with the lowest entrenchment score (speculative assumptions before evidence-backed observations)
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [ACCEPTED] add-nogood-fallback-uses-dependent-count
When `find_culprits` returns no candidates (all nogood nodes are premises with no justification chains), the fallback retracts the nogood member with the fewest direct dependents
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [ACCEPTED] nogood-ids-assume-append-only
STALE: Fixed in PR #35 (issue #26) — now uses a persisted monotonic counter.
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [ACCEPTED] unknown-type-returns-false
Low value — any justification type other than SL/CP returning False is defensive behavior unlikely to affect a developer working in this codebase; the system only creates SL and CP types
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [ACCEPTED] add-nogood-empty-list-crashes
Bug report about an edge case (empty `node_ids` causes `IndexError`) — this is a bug to fix, not an architectural invariant to track as a belief
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [ACCEPTED] propagate-bfs-order-guarantee
The BFS ordering of changed nodes is an implementation detail that no caller currently depends on for correctness — tracking it as a belief adds noise without preventing breakage
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md


All claims confirmed against the source. Here are the proposed beliefs extracted from the four entries:

---

## From `scan-ftl-reasons.md`

### [ACCEPTED] three-layer-architecture
The codebase is a three-layer stack: data model (`__init__.py`), TMS engine (`network.py`), and persistence (`storage.py`), with `api.py` providing functional API and `cli.py` as a thin argparse wrapper.
- Source: entries/2026/04/23/scan-ftl-reasons.md
- Rejected: trivial: architectural layers are obvious from file names (network.py, storage.py, api.py, cli.py)

### [ACCEPTED] network-is-central-dependency
`network.py` is imported by essentially every other module — api, storage, import, export, compact, check_stale, and all test files — making it the central data structure of the project.
- Source: entries/2026/04/23/scan-ftl-reasons.md
- Rejected: duplicate: captured by existing `central-dependency-is-safely-contained`

### [ACCEPTED] api-uses-with-network-context-manager
`api.py` uses a `_with_network` context manager to ensure load-operate-save atomicity for all network mutations.
- Source: entries/2026/04/23/scan-ftl-reasons.md
- Rejected: duplicate: already exists as accepted belief with same ID

---

## From `reasons_lib-network.md`

### [ACCEPTED] sl-justification-semantics
An SL justification is valid iff ALL antecedents are IN AND ALL outlist nodes are OUT; a node is IN iff ANY of its justifications is valid (conjunction within a justification, disjunction across justifications).
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPTED] propagation-is-bfs
Truth value propagation in `_propagate` uses `deque`-based BFS through the `dependents` graph, not DFS, ensuring breadth-first wavefront expansion.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: BFS already captured in add-justification-triggers-propagation

### [ACCEPTED] premise-defaults-to-in
A node with no justifications (a premise) defaults to IN; `_compute_truth` preserves its current truth value rather than recomputing it.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPTED] dependents-bidirectional-index
Each node maintains a `dependents` set (reverse of antecedent/outlist edges), eagerly maintained by `add_node`, `add_justification`, `supersede`, `challenge`, and `convert_to_premise`.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: already exists as accepted belief with same ID (currently OUT)

### [ACCEPTED] recompute-all-uses-fixpoint
`recompute_all` iterates until no truth values change, bounded by `len(nodes) + 1` iterations, handling cascading dependencies from arbitrary node ordering.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPTED] retracted-nodes-skipped-in-propagation
Retracted nodes (marked with `_retracted` metadata) are skipped during BFS propagation but remain in the network for potential restoration.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: same claim as proposed propagate-skips-retracted-nodes

### [ACCEPTED] missing-outlist-nodes-pass-validation
In `_justification_valid`, missing antecedent nodes cause the check to fail (node goes OUT), but missing outlist nodes pass (don't block) — an open-world default.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: same claim as proposed sl-outlist-asymmetry

### [ACCEPTED] backtracking-retracts-least-entrenched
`add_nogood` resolves contradictions via dependency-directed backtracking: `find_culprits` traces to premises, scores by `_entrenchment`, and retracts the least-entrenched premise to minimize disruption.
- Source: entries/2026/04/23/reasons_lib-network.md
- Rejected: duplicate: already exists as accepted belief with same ID

---

## From `reasons_lib-network-challenge.md`

### [ACCEPTED] challenge-uses-outlist-mechanism
`challenge` works by creating an IN premise node and adding it to the target's outlist in every justification, reusing the same non-monotonic mechanism as `supersede`.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: same claim as existing `challenge-is-outlist-injection`

### [ACCEPTED] challenge-converts-premises-to-justified
When a premise (node with no justifications) is challenged, it is converted to a justified node with an SL justification containing empty antecedents and the challenge in the outlist.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [ACCEPTED] challenge-modifies-all-justifications
When the target has multiple justifications, the challenge node is added to the outlist of every justification, ensuring no single justification can independently keep the target IN.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [ACCEPTED] challenge-id-auto-generation
Auto-generated challenge IDs follow the pattern `challenge-{target}`, then `challenge-{target}-2`, `-3`, etc.; explicit IDs that collide raise `ValueError` rather than auto-deduplicating.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

### [ACCEPTED] defend-is-challenge-of-challenge
`defend` works by calling `challenge` on the challenge node itself, creating a recursive dialectical structure where truth values resolve automatically through the same outlist mechanism.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md
- Rejected: duplicate: already exists as accepted belief with same ID

---

## From `reasons_lib-storage.md`

### [ACCEPTED] storage-save-is-full-replace
`Storage.save()` deletes all rows from every table before re-inserting the entire network; there is no incremental or differential update path.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [ACCEPTED] storage-load-bypasses-propagation
`load()` constructs nodes directly into `network.nodes` rather than calling `add_node`, so truth maintenance propagation does not fire during deserialization.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: `bootstrap-bypasses-incremental-propagation` already captures that persistence loading constructs the full node graph before truth maintenance

### [ACCEPTED] justification-order-preserved-via-rowid
Justification insertion order is preserved across save/load cycles using `AUTOINCREMENT` rowid and `ORDER BY rowid` on read, which matters because justification priority affects truth maintenance.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [ACCEPTED] dependents-index-derived-on-load
The `node.dependents` set is never persisted to SQLite; it is rebuilt by walking all justification antecedents and outlists during `load()`.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: already exists as belief `dependents-index-derived-on-load` (currently OUT)

### [ACCEPTED] storage-trusts-stored-truth-values
`load()` trusts the stored `truth_value` without re-running propagation, making the database the source of truth for node status.
- Source: entries/2026/04/23/reasons_lib-storage.md
- Rejected: duplicate: covered by bootstrap-bypasses-incremental-propagation which already states "load trusts stored truth values"

---

## From `topic-doyle-1979-tms.md`

### [ACCEPTED] cp-and-sl-evaluated-identically
CP and SL justifications use the same validity check in `_justification_valid`; the distinction is semantic (support vs. consistency), not computational.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md
- Rejected: duplicate: already exists as belief `cp-and-sl-evaluated-identically` (currently IN)

### [ACCEPTED] sl-justification-is-conjunction-of-inlist-disjunction-of-outlist
Redundant with `sl-justification-semantics` proposed above, which captures the same claim more concisely.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md

### [ACCEPTED] retracted-nodes-stay-in-network
Redundant with `retracted-nodes-skipped-in-propagation` which captures the same fact plus the propagation behavior.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md


No existing beliefs. Here are the proposed beliefs from both entries:

---

## From `entries/2026/04/23/topic-multi-agent-federation.md`

### [ACCEPTED] namespace-prefix-is-colon-separated
Agent namespacing uses the format `agent_name:belief_id`; `_resolve_namespace()` skips prefixing any ID that already contains a colon, preventing double-prefixing of cross-namespace references
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: covered by existing `colon-means-already-namespaced` and `import-agent-namespace-prefix`

### [ACCEPTED] active-inactive-relay-pair
Each imported agent gets exactly two infrastructure nodes: `agent:active` (premise, starts IN) and `agent:inactive` (derived via SL with `outlist=[active_id]`, starts OUT); every imported belief includes `inactive_id` in its outlist
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists as belief `active-inactive-relay-pair` (currently IN)

### [ACCEPTED] active-not-in-antecedents
The `active` premise is deliberately excluded from imported beliefs' antecedents; if it were an antecedent, it would provide a second always-valid justification path that defeats per-belief retraction semantics
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists as belief `active-not-in-antecedents` (currently IN)

### [ACCEPTED] kill-switch-cascade-is-reversible
Retracting `agent:active` cascades all agent beliefs to OUT via the inactive relay flipping IN; re-asserting `agent:active` reverses the cascade, restoring all beliefs to IN via the same BFS propagation
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: covered by `all-defeat-mechanisms-are-reversible` and `defeat-reversal-propagates-automatically`

### [ACCEPTED] agent-cascades-are-isolated-by-namespace
Retracting one agent's active premise does not affect other agents' beliefs, because each agent's imported beliefs reference only their own `inactive` node in their outlist
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists as belief `agent-cascades-are-isolated-by-namespace` (currently IN)

### [ACCEPTED] justification-validity-requires-inlist-in-and-outlist-out
Redundant with sl-justification-semantics.
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPTED] import-skips-existing-sync-is-remote-wins
Import mode (`import_agent`) is a one-time load that skips existing nodes; sync mode updates text/justifications/truth values with remote-wins semantics and retracts locally any beliefs removed from the remote
- Source: entries/2026/04/23/topic-multi-agent-federation.md
- Rejected: duplicate: already exists (referenced by `idempotent-reimport-skips-all`)

---

## From `entries/2026/04/23/topic-outlist-semantics.md`

### [ACCEPTED] outlist-absent-means-out
Redundant with missing-outlist-nodes-pass-validation and sl-outlist-asymmetry.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPTED] node-in-if-any-justification-valid
Duplicate of node-in-if-any-justification-valid proposed earlier.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPTED] challenge-is-outlist-injection
Redundant with challenge-uses-outlist-mechanism.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPTED] defend-is-recursive-challenge
Redundant with defend-is-challenge-of-challenge.
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPTED] supersession-is-reversible
`supersede()` adds the new node's ID to the old node's outlist rather than deleting the old node; retracting the new belief automatically restores the old one through normal propagation
- Source: entries/2026/04/23/topic-outlist-semantics.md
- Rejected: duplicate: already captured by `all-defeat-mechanisms-are-reversible` and `defeat-reversal-propagates-automatically`

### [ACCEPTED] multiple-outlist-is-conjunction
When a justification has multiple outlist entries, ALL must be OUT for the justification to be valid; any single outlist node going IN defeats the entire justification
- Source: entries/2026/04/23/topic-outlist-semantics.md
- Rejected: duplicate: subsumed by proposed sl-justification-semantics ("ALL outlist nodes are OUT")

### [ACCEPTED] outlist-relationships-survive-persistence
Outlists are stored as `outlist_json` in the SQLite `justifications` table; on load, the dependent index is rebuilt for both antecedents and outlist nodes, preserving propagation behavior across save/load cycles
- Source: entries/2026/04/23/topic-outlist-semantics.md
- Rejected: duplicate: `dependents-survive-storage-roundtrip` + `dependency-tracking-is-complete-for-all-reference-types` already cover that outlist references persist and the dependents index is rebuilt for both antecedents and outlists



---

**Generated:** 2026-05-04
**Source:** 3 entries from entries/
**Model:** claude

### [ACCEPTED] pgapi-find-dependents-queries-outlist
PgApi's `_find_dependents` queries both `antecedents @>` and `outlist @>` JSONB containment, correctly enqueuing outlist-dependent nodes for re-evaluation — a capability the in-memory Network historically lacked until PR #31.
- Source: entries/2026/04/29/gated-and-negative-analysis.md

### [ACCEPTED] pgapi-validates-refs-before-justification-insert
PgApi's `_validate_refs` checks all antecedent and outlist node IDs exist in `rms_nodes` before inserting a justification, providing application-level referential integrity that compensates for JSONB arrays' inability to enforce foreign key constraints.
- Source: entries/2026/04/29/session-summary.md

### [ACCEPTED] pgapi-what-if-uses-transaction-rollback
PgApi's `what_if_retract` and `what_if_assert` wrap BFS cascade analysis inside a database transaction with `try/finally` guaranteed ROLLBACK, achieving read-only simulation by never committing the exploratory mutations.
- Source: entries/2026/04/29/session-summary.md
- Rejected: duplicate: already captured by `both-backends-support-safe-hypothetical-reasoning`

### [ACCEPTED] list-negative-json-parser-tolerates-prose-preamble
The `list_negative` LLM classification response parser uses `re.finditer` to extract JSON objects from responses that include prose preamble, handling the common LLM pattern of prefacing structured output with natural language rather than requiring clean JSON.
- Source: entries/2026/04/29/session-summary.md
- Rejected: trivial: The behavioral contract (graceful handling of unparseable LLM output) is already in `api-list-negative-graceful-on-malformed-llm`; the regex mechanism is an implementation detail visible in the source



---

**Generated:** 2026-05-05
**Source:** 23 entries from entries/
**Model:** claude

### [ACCEPTED] update-2026-05-04
Status update listing already-tracked beliefs and their blockers; no new architectural claims to extract beyond what is already accepted.
- Source: entries/2026/05/04/update.md

---

### [ACCEPTED] derived-belief-soundness-is-llm-only
Structural validation ensures justification references exist and are IN, but the logical soundness of the inference from antecedents to derived conclusion is validated only by the proposing LLM — no code-level check verifies that the reasoning step is logically valid.
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md
- Rejected: duplicate: belief `derived-belief-soundness-is-llm-only` already exists IN the current network with the same claim

### [ACCEPTED] premise-derived-verifiability-asymmetry
Premises record source file and line range at assertion time, enabling check-stale to verify them against current code; derived beliefs have no equivalent source-level grounding and can only be structurally validated (references exist and are IN).
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md

### [ACCEPTED] tms-provides-revisability-not-certainty
The TMS design provides traceable, revisable reasoning rather than certainty: every belief records its complete justification chain back to premises, beliefs are automatically retracted when support changes, and the epistemological status of each belief is mechanically traceable via `reasons explain`.
- Source: entries/2026/05/05/epistemology-of-derived-beliefs.md
- Rejected: speculative: editorial judgment about design philosophy, not an observable code fact

---

### [ACCEPTED] ftl-reasons-zero-runtime-deps
The core library declares `dependencies = []` in pyproject.toml; the TMS engine runs on the Python standard library alone (including `sqlite3`), with PostgreSQL support available only via the `pg` optional extra.
- Source: entries/2026/05/05/pyproject.md
- Rejected: duplicate: already captured by `architecture-is-self-contained-and-safely-layered` (zero runtime dependencies)

### [ACCEPTED] reasons-cli-entrypoint-is-cli-main
The `reasons` console script always resolves to `reasons_lib.cli:main()` with no plugin system, alternate entry point, or dispatch indirection.
- Source: entries/2026/05/05/pyproject.md
- Rejected: duplicate: already captured by `entry-point-mapping`

### [ACCEPTED] tests-excluded-from-distribution
Only `reasons_lib/` is packaged for distribution; the `tests/` directory is excluded via explicit `[tool.setuptools] packages = ["reasons_lib"]` configuration.
- Source: entries/2026/05/05/pyproject.md
- Rejected: trivial: standard setuptools packaging config directly visible in pyproject.toml

### [ACCEPTED] pg-extra-required-for-postgres
PostgreSQL support requires installing the `pg` or `test-pg` optional extra (`psycopg[binary]>=3.1`); it is not available in a bare install.
- Source: entries/2026/05/05/pyproject.md

### [ACCEPTED] python-310-minimum-required
The project requires Python >= 3.10 (`requires-python = ">=3.10"` in pyproject.toml); anything below fails at install time.
- Source: entries/2026/05/05/pyproject.md
- Rejected: trivial: single field directly visible in pyproject.toml

---

### [ACCEPTED] node-premise-is-implicit
A node is a premise (believed without justification) iff its `justifications` list is empty; there is no explicit `is_premise` flag or type discriminator.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: already captured by `challenge-destroys-premise-identity` (premise identity emerges from absence of justifications)

### [ACCEPTED] node-truth-disjunctive
A node is IN if ANY of its justifications is valid (disjunctive semantics); adding a justification to a node can never cause it to go OUT.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: disjunctive semantics already stated in proposed sl-justification-semantics; monotonicity corollary follows logically and is partially covered by add-justification-achieves-consistent-propagation

### [ACCEPTED] dependents-not-self-enforced
The `Node.dependents` reverse index is not maintained by the data model itself; `network.py` is responsible for keeping it consistent with justification references (antecedents and outlists).
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: same claim as existing `dependents-is-manual-reverse-index`

### [ACCEPTED] data-model-uses-string-enums
Both `Justification.type` ("SL"/"CP") and `Node.truth_value` ("IN"/"OUT") are plain strings, not Python enums; consumers must validate values themselves as invalid states like `"MAYBE"` or `"XYZ"` are representable.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: duplicate: already exists as `data-model-uses-string-enums [IN]`

### [ACCEPTED] init-has-zero-external-deps
`reasons_lib/__init__.py` imports only `dataclasses` from the standard library, making it the dependency-free root of the package — imported by 29 test files and transitively by every module in `reasons_lib`.
- Source: entries/2026/05/05/reasons_lib-__init__.md
- Rejected: trivial: standard-library-only imports visible from file; "29 test files" is an ephemeral count

---

### [ACCEPTED] namespace-kill-switch-uses-active-premise
Retracting the `{ns}:active` premise node cascades all beliefs in that namespace to OUT; `ensure_namespace()` creates this node with `role: agent_premise` metadata as the namespace's kill switch.
- Source: entries/2026/05/05/reasons_lib-api.md
- Rejected: duplicate: covered by `active-inactive-relay-pair` and `kill-switch-uses-outlist-not-antecedent`

### [ACCEPTED] fts-errors-silently-caught-in-search
FTS5 query errors in `_fts_search` are silently caught and return an empty list, falling back to substring matching — the only place in the API where errors are deliberately swallowed (FTS5 table may not exist).
- Source: entries/2026/05/05/reasons_lib-api.md

### [ACCEPTED] cross-namespace-refs-skip-prefixing
`_resolve_namespace` skips node IDs that already contain `:`, treating them as cross-namespace references — so `alice:some-belief` referenced from Bob's namespace stays `alice:some-belief`, not `bob:alice:some-belief`.
- Source: entries/2026/05/05/reasons_lib-api.md
- Rejected: duplicate: same claim as `colon-means-already-namespaced`

### [ACCEPTED] search-relaxation-capped-at-50-queries
FTS progressive relaxation drops terms via `combinations()` (largest subsets first) until results appear, capped at 50 total relaxation queries to prevent combinatorial explosion on long search strings.
- Source: entries/2026/05/05/reasons_lib-api.md
- Rejected: duplicate: `api-fts-search-bounds-query-calls` already states the 51-call cap (1 original + 50 relaxation), which is the same invariant


### [ACCEPTED] ask-dual-mode-makes-three-llm-calls
Dual mode makes up to 3 LLM calls (1 TMS synthesis + 1 FTS RAG + 1 merge), short-circuiting to 2 if either retrieval path returns empty.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: already exists as `ask-dual-mode-makes-three-llm-calls [IN]`

### [ACCEPTED] ask-tool-call-parsing-is-line-based-json
Tool calls are detected by scanning each line of the LLM response for valid JSON with a `"tool"` key; multi-line JSON objects are silently missed.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: core claim matches `ask-uses-text-based-tool-protocol`

### [ACCEPTED] ask-sources-db-failure-silently-degrades
If the `sources_db` SQLite file is missing or corrupt, `_search_source_chunks` catches `OperationalError`/`DatabaseError` and returns empty string, degrading to belief-only mode without user-visible errors.
- Source: entries/2026/05/05/reasons_lib-ask.md
- Rejected: duplicate: already exists as `ask-sources-db-failure-silently-degrades [IN]`

### [ACCEPTED] ask-tool-loop-max-three-rounds
Covered by existing `ask-has-tiered-query-modes` which already describes the "bounded 3-iteration tool loop."
- Source: entries/2026/05/05/reasons_lib-ask.md

### [ACCEPTED] ask-llm-failure-returns-raw-beliefs
Covered by existing `ask-falls-back-to-raw-search` and `ask-never-raises-on-llm-failure`.
- Source: entries/2026/05/05/reasons_lib-ask.md

### [ACCEPTED] truncated-hash-threshold-is-16-chars
A stored `source_hash` of exactly 16 characters that is a prefix of the current full SHA-256 hash is classified as `truncated_hash` (legacy format), not `content_changed`.
- Source: entries/2026/05/05/reasons_lib-check_stale.md

### [ACCEPTED] check-stale-and-hash-sources-mutate-in-place
Both `check_stale` (with `upgrade_hashes=True`) and `hash_sources` modify `node.source_hash` directly on the Network object; neither persists — the caller must save.
- Source: entries/2026/05/05/reasons_lib-check_stale.md
- Rejected: duplicate: already exists as `check-stale-and-hash-sources-mutate-in-place [IN]`

### [ACCEPTED] check-stale-only-examines-in-nodes
Already exists as `check-stale-skips-out-nodes`.
- Source: entries/2026/05/05/reasons_lib-check_stale.md

### [ACCEPTED] cli-is-pure-adapter
Already exists as `cli-is-pure-delegation-layer` and `cli-is-verified-pure-delegation`.
- Source: entries/2026/05/05/reasons_lib-cli.md

### [ACCEPTED] check-stale-exits-nonzero-on-stale
`cmd_check_stale` calls `sys.exit(1)` when stale nodes are found (even on success), making the command usable as a CI or pre-commit gate.
- Source: entries/2026/05/05/reasons_lib-cli.md
- Rejected: duplicate: identical to existing `check-stale-exits-nonzero`

### [ACCEPTED] commands-dict-must-mirror-subparsers
Adding a CLI subcommand requires entries in both the argparse subparser definitions and the `commands` dispatch dict in `main()`; omitting either silently breaks the command.
- Source: entries/2026/05/05/reasons_lib-cli.md
- Rejected: duplicate: already exists as `commands-dict-must-mirror-subparsers [IN]`

### [ACCEPTED] derive-one-round-return-protocol
The key invariant (negative = error) is captured by existing `derive-round-returns-negative-on-error`; positive = count added and 0 = saturated are the obvious complement.
- Source: entries/2026/05/05/reasons_lib-cli.md

### [ACCEPTED] deferred-imports-for-llm-modules
Same architectural pattern already captured by `api-uses-lazy-imports`; the cli.py instance follows the same rationale.
- Source: entries/2026/05/05/reasons_lib-cli.md

### [ACCEPTED] import-agent-inactive-in-outlist-not-antecedent
Already exists as `active-not-in-antecedents` and `kill-switch-uses-outlist-not-antecedent`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-out-beliefs-get-empty-justifications
Beliefs that are OUT or STALE in the source are imported with empty justification lists, preventing `recompute_all` from resurrecting them to IN.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-topo-sort-before-add
Already exists as `import-json-uses-topological-sort`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPTED] sync-agent-remote-wins-semantics
Already exists as `import-skips-existing-sync-is-remote-wins`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-dangling-refs-silently-dropped
Already exists as `normalization-drops-unknown-refs`.
- Source: entries/2026/05/05/reasons_lib-import_agent.md

### [ACCEPTED] llm-prompt-always-via-stdin
Prompts are passed to LLM subprocesses via `subprocess.run(input=...)`, never as command-line arguments — avoiding shell injection and argument length limits.
- Source: entries/2026/05/05/reasons_lib-llm.md

### [ACCEPTED] ollama-thinking-strip-is-fragile
The ollama output post-processing strips exact string markers (`Thinking...\n` / `...done thinking.\n`) that are coupled to ollama's current output format and may break across versions.
- Source: entries/2026/05/05/reasons_lib-llm.md
- Rejected: speculative: editorial judgment about fragility; "may break across versions" is a prediction

### [ACCEPTED] llm-invoke-is-subprocess-only
Covered by existing `all-external-execution-is-subprocess-isolated`.
- Source: entries/2026/05/05/reasons_lib-llm.md

### [ACCEPTED] claudecode-env-stripped-from-child
Covered by existing `llm-subprocess-isolation-prevents-recursion`.
- Source: entries/2026/05/05/reasons_lib-llm.md

### [ACCEPTED] llm-module-stdlib-only
Covered by existing `project-has-zero-external-coupling`.
- Source: entries/2026/05/05/reasons_lib-llm.md


### [ACCEPTED] network-is-sole-truth-propagation-engine
All truth value computation and propagation in the system flows through `Network._propagate()` and `Network._compute_truth()`; no other module modifies truth values directly.
- Source: entries/2026/05/05/reasons_lib-network.md

### [ACCEPTED] sl-justification-is-disjunctive
A node is IN if *any* of its justifications is valid (disjunctive semantics); `_compute_truth()` short-circuits on the first valid justification rather than requiring all justifications to hold.
- Source: entries/2026/05/05/reasons_lib-network.md
- Rejected: duplicate: same claim as proposed `node-truth-disjunctive` with minor implementation detail added

### [ACCEPTED] retracted-nodes-skip-propagation
Duplicate of existing `metadata-actively-governs-truth-propagation` which states "retracted nodes are skipped during BFS traversal and trigger nodes are never recomputed."
- Source: entries/2026/05/05/reasons_lib-network.md

### [ACCEPTED] forward-references-are-silently-tolerated
Covered by existing `outlist-absent-means-out` (missing outlist nodes treated as OUT) and `active-not-in-antecedents` (missing antecedents treated as not-IN), which together describe the forward-reference tolerance behavior.
- Source: entries/2026/05/05/reasons_lib-network.md

### [ACCEPTED] outlist-semantics-enable-nonmonotonic-reasoning
Covered by existing `challenge-is-outlist-injection`, `kill-switch-uses-outlist-not-antecedent`, and `dialectics-inherit-complete-outlist-semantics`, which together establish that all non-monotonic operations reduce to outlist manipulation.
- Source: entries/2026/05/05/reasons_lib-network.md

### [ACCEPTED] pgapi-has-no-retry-logic
`PgApi` uses one connection per instance with no retry on transient database errors; callers are expected to handle `psycopg` exceptions from connection failures or serialization conflicts.
- Source: entries/2026/05/05/reasons_lib-pg.md

### [ACCEPTED] access-tags-are-monotonically-inherited
A derived node's `access_tags` are the union of its own tags and all antecedent tags; tags are never removed by inheritance, only accumulated, so access restrictions can only grow along derivation chains.
- Source: entries/2026/05/05/reasons_lib-pg.md
- Rejected: duplicate: monotonic growth is a direct consequence of union semantics in `access-tags-union-inheritance`

### [ACCEPTED] review-only-evaluates-derived-beliefs
`review_beliefs` filters out premises (nodes without justifications); only derived beliefs with at least one justification are sent for LLM review.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPTED] review-parse-defaults-fail-safe
`parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True`, so a missing or malformed field in LLM output never triggers a false alarm.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPTED] review-batch-failure-is-silent-skip
When an LLM call fails for a review batch, the error is logged to stderr but the batch is skipped with no indication in the returned results; callers cannot distinguish "skipped due to error" from "no problems found."
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPTED] review-has-no-storage-dependency
The review module operates entirely on an in-memory `nodes` dict (from `export_network()`) and never reads from or writes to the database directly.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPTED] review-result-schema-is-normalized
Every result dict returned by `parse_review_response` is guaranteed to have exactly six keys (`id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`) regardless of what the LLM returned, via normalization with safe defaults.
- Source: entries/2026/05/05/reasons_lib-review.md

### [ACCEPTED] storage-load-bypasses-add-node
`load()` assigns nodes directly to `network.nodes` and calls `_rebuild_dependents()` afterward, deliberately skipping `add_node()` to avoid triggering truth maintenance propagation during state restoration.
- Source: entries/2026/05/05/reasons_lib-storage.md
- Rejected: duplicate: covered by `bootstrap-bypasses-incremental-propagation` which already captures that loading constructs the full graph before truth maintenance, skipping incremental propagation

### [ACCEPTED] storage-justification-order-preserved
Justifications are inserted in list order and loaded via `ORDER BY rowid`, preserving the ordering that determines which justification is evaluated first during truth computation.
- Source: entries/2026/05/05/reasons_lib-storage.md

### [ACCEPTED] storage-nogood-counter-persisted
Covered by existing `all-identifiers-are-durable-across-persistence-boundaries` which states "nogood IDs survive save/load cycles via the network_meta table's high-water mark."
- Source: entries/2026/05/05/reasons_lib-storage.md

### [ACCEPTED] api-tests-use-real-sqlite
All API tests run against a real SQLite database with FTS5; storage is never mocked, ensuring the API contract includes correct SQL and full-text search behavior.
- Source: entries/2026/05/05/tests-test_api.md
- Rejected: duplicate: identical belief ID already exists as IN

### [ACCEPTED] list-negative-batches-at-50
`list_negative` splits candidate nodes into batches of approximately 50 for LLM classification, verified by the test suite asserting exactly 3 LLM calls for 120 candidates.
- Source: entries/2026/05/05/tests-test_api.md

### [ACCEPTED] fts-relaxation-bounded
Progressive FTS query relaxation is bounded: a 20-term query produces at most 51 `_fts_query` invocations, preventing unbounded search expansion on long input queries.
- Source: entries/2026/05/05/tests-test_api.md
- Rejected: duplicate: same claim as search-relaxation-capped-at-50-queries — both assert FTS relaxation is bounded

### [ACCEPTED] retract-assert-inverse-tested
Test coverage claim rather than code invariant; the behavioral property (retract/assert are inverses) is already captured by `defeat-reversal-propagates-automatically` and `supersession-is-reversible-and-view-consistent`.
- Source: entries/2026/05/05/tests-test_api.md

### [ACCEPTED] api-error-contract-exceptions
Duplicate of existing `api-enforces-typed-preconditions` which covers typed exceptions (ValueError, KeyError, PermissionError) at API boundaries.
- Source: entries/2026/05/05/tests-test_api.md


### [ACCEPTED] ask-tool-call-budget-is-three
The agentic `ask` loop permits at most 3 tool-call rounds before forcing synthesis via `build_final_prompt` (no tool definition), resulting in exactly 4 LLM invocations maximum.
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: same claim as existing `ask-tool-loop-capped-at-three` (3-round cap with forced synthesis)

### [ACCEPTED] ask-dual-requires-sources-db
Calling `ask(..., dual=True)` without providing a `sources_db` path raises `ValueError`.
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: identical belief ID already exists as IN

### [ACCEPTED] ask-natural-mode-strips-metadata-and-cite
When `natural=True`, all belief metadata (`**Status:**`, `### ` headers, `**Source:**`) is stripped from the prompt context and the "Cite belief IDs" instruction is replaced with "plain natural language".
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: identical belief ID already exists as IN

### [ACCEPTED] search-source-chunks-filters-stop-words
`_search_source_chunks` filters out single-character tokens and common stop words before constructing FTS5 queries, returning empty string when no usable terms remain.
- Source: entries/2026/05/05/tests-test_ask.md
- Rejected: duplicate: existing belief `ask-stop-word-fallback-ensures-nonempty-query` already captures stop-word stripping and the empty-query fallback behavior in `_search_source_chunks`

### [ACCEPTED] resolve-source-path-db-dir-precedence
`resolve_source_path` follows a strict precedence chain: `db_dir` resolution takes priority over agent repo resolution, which takes priority over repo-key-split resolution; missing files return `None`.
- Source: entries/2026/05/05/tests-test_check_stale.md

### [ACCEPTED] truncated-hash-upgrade-opt-in
Prefix hash upgrade (16-char truncated SHA-256 to full digest) only occurs when `upgrade_hashes=True` is passed to `check_stale`; without it, truncated hashes produce a warning result with `reason="truncated_hash"`.
- Source: entries/2026/05/05/tests-test_check_stale.md

### [ACCEPTED] import-agent-retraction-survives-recompute
After retracting an imported belief, `recompute_all()` must not resurrect it — the justification structure must not provide alternative paths to IN (issue #16 regression invariant).
- Source: entries/2026/05/05/tests-test_import_agent.md

### [ACCEPTED] llm-resolve-ollama-preserves-inner-colons
`resolve_model_cmd("ollama:model:tag")` splits on the first colon only, keeping `model:tag` intact as the Ollama model identifier.
- Source: entries/2026/05/05/tests-test_llm.md

### [ACCEPTED] llm-thinking-strip-ollama-only
Thinking-marker stripping (`Thinking...` / `...done thinking.`) is applied only to Ollama model output; Claude and Gemini output passes through unmodified even if it contains the same markers.
- Source: entries/2026/05/05/tests-test_llm.md

### [ACCEPTED] ask-graceful-degradation-on-llm-failure
Duplicate of existing `ask-never-raises-on-llm-failure` and `all-query-operations-degrade-gracefully`.
- Source: entries/2026/05/05/tests-test_ask.md

### [ACCEPTED] check-stale-returns-dicts-not-exceptions
Duplicate of existing `staleness-results-are-dicts-not-exceptions`.
- Source: entries/2026/05/05/tests-test_check_stale.md

### [ACCEPTED] check-stale-requires-source-and-hash
Duplicate of existing `check-stale-requires-both-source-fields`.
- Source: entries/2026/05/05/tests-test_check_stale_issue25.md

### [ACCEPTED] source-deleted-nulls-hash-and-path
Duplicate of existing `check-stale-source-deleted-returns-none-hashes`.
- Source: entries/2026/05/05/tests-test_check_stale_issue25.md

### [ACCEPTED] import-agent-namespaces-all-ids
Duplicate of existing `import-agent-namespace-prefix`.
- Source: entries/2026/05/05/tests-test_import_agent.md

### [ACCEPTED] import-agent-active-premise-not-antecedent
Duplicate of existing `active-not-in-antecedents`.
- Source: entries/2026/05/05/tests-test_import_agent.md

### [ACCEPTED] llm-invoke-strips-claudecode-env
Covered by existing `llm-subprocess-isolation-prevents-recursion` and `all-external-execution-is-subprocess-isolated`.
- Source: entries/2026/05/05/tests-test_llm.md

### [ACCEPTED] llm-invoke-checks-path-before-run
Covered by existing `invoke-claude-raises-when-binary-missing` (same behavior generalized to all model binaries).
- Source: entries/2026/05/05/tests-test_llm.md

### [ACCEPTED] llm-three-error-types
Too granular — the three error types (ValueError, FileNotFoundError, RuntimeError) are implementation details of the error taxonomy already captured by existing beliefs about fail-fast and fail-soft behavior.
- Source: entries/2026/05/05/tests-test_llm.md


Looking at the three entries, I'll extract beliefs and cross-check against the existing 189-belief registry.

---

## From `tests/test_pg.py`

### [ACCEPTED] sl-param-is-comma-separated-node-ids
`add_node()` encodes SL justifications as comma-separated node IDs in the `sl=` parameter and outlist nodes in `unless=`, mirroring the TMS (SL, OL) formalism directly in the API signature.
- Source: entries/2026/05/05/tests-test_pg.md
- Rejected: trivial: the comma-separated format of the `sl` parameter is immediately visible from any function call site in the code; no non-obvious architectural constraint is conveyed

### [ACCEPTED] pg-tests-gated-by-database-availability
The entire `test_pg.py` suite (18 test classes) is conditionally skipped via `skip_no_pg` marker from conftest when PostgreSQL is unavailable, and selectively collectible via `pytest.mark.pg` — enabling CI without a Postgres dependency.
- Source: entries/2026/05/05/tests-test_pg.md
- Rejected: trivial: conditional skip markers for unavailable services are standard pytest practice visible from conftest

### [ACCEPTED] pg-fixture-provides-per-test-isolation
Each pg test receives an isolated `PgApi` instance via the `pg_api` fixture, which creates a fresh project namespace (UUID) and tears down after each test, ensuring zero shared state between tests.
- Source: entries/2026/05/05/tests-test_pg.md
- Rejected: trivial: test-fixture setup (fresh UUID per test) is immediately visible from the test file and is test infrastructure, not a behavioral invariant of the codebase

### [ACCEPTED] pg-keyerror-includes-missing-node-id
When `add_node` references a nonexistent node in `sl=` or `unless=`, PgApi raises `KeyError` with the missing node ID in the exception message — tested via `pytest.raises(KeyError, match="ghost")` — enabling callers to identify which reference is dangling.
- Source: entries/2026/05/05/tests-test_pg.md

### [ACCEPTED] pg-test-suite-is-backend-parity
`test_pg.py` is a parity test suite: every behavior tested has a corresponding expected behavior from the SQLite backend, validating that PgApi returns the same dict shapes and enforces the same invariants to ensure backend interchangeability.
- Source: entries/2026/05/05/tests-test_pg.md

---

## From `tests/test_review.py`

### [ACCEPTED] review-skips-premises
`review_beliefs` only sends beliefs with at least one justification to the LLM; premise nodes (empty justifications list) are excluded from review entirely.
- Source: entries/2026/05/05/tests-test_review.md

### [ACCEPTED] parse-review-defaults-to-passing
`parse_review_response` defaults missing fields to valid=True, sufficient=True, necessary=True, unnecessary_antecedents=[] — the LLM only needs to report failures explicitly; omission means the belief passed.
- Source: entries/2026/05/05/tests-test_review.md

### [ACCEPTED] review-parse-requires-json-array
`parse_review_response` only accepts JSON arrays; a bare JSON object `{...}` is treated as unparseable and returns an empty list — the LLM must return a list even for single-item reviews.
- Source: entries/2026/05/05/tests-test_review.md

### [ACCEPTED] auto-retract-respects-dry-run
The `--auto-retract` flag in the `review-beliefs` CLI is gated by `--dry-run`: when dry-run is active, findings are displayed but no database mutation occurs, even for beliefs flagged as invalid.
- Source: entries/2026/05/05/tests-test_review.md
- Rejected: duplicate: already exists as `auto-retract-respects-dry-run` [IN] in the belief network

### [ACCEPTED] review-format-handles-missing-antecedents
`format_belief_for_review` renders `"(not found in network)"` for antecedent IDs that don't exist in the nodes dict rather than crashing, and returns an empty string for a nonexistent belief ID.
- Source: entries/2026/05/05/tests-test_review.md

### [ACCEPTED] review-tolerates-llm-failure
Duplicates existing beliefs `llm-integration-fails-softly-across-modules` and `all-llm-interactions-are-bounded-and-fail-soft`, which already establish that all LLM integration fails softly across the system.
- Source: entries/2026/05/05/tests-test_review.md

---

## From `tests/test_sync_agent.py`

### [ACCEPTED] sync-agent-is-idempotent
Calling `sync_agent` twice with identical data produces the same database state; the second call returns zero adds, removes, and updates — a no-op by design.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPTED] sync-returns-structured-diff-counts
`sync_agent` returns a dict with `beliefs_added`, `beliefs_updated`, `beliefs_unchanged`, and `beliefs_removed` counts that accurately reflect the diff between remote file and local state — no double-counting across sync cycles.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPTED] sync-registers-agent-repo-path
`sync_agent` records the agent's source file path in the repo registry (`list_repos`), enabling the system to track which file each agent's beliefs originated from.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPTED] sync-agent-is-remote-wins
Duplicates existing belief `import-skips-existing-sync-is-remote-wins`, which already establishes that sync mode uses remote-wins semantics including overriding local retractions.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPTED] sync-supports-markdown-and-json
Duplicates existing belief `import-agent-normalizers-share-intermediate-schema`, which covers that both markdown and JSON formats flow through the same normalization pipeline for import/sync operations.
- Source: entries/2026/05/05/tests-test_sync_agent.md

### [ACCEPTED] stale-beliefs-import-as-out
Duplicates existing belief `staleness-information-survives-binary-truth-model`, which already establishes that STALE beliefs map to OUT with stale_reason metadata preserved end-to-end.
- Source: entries/2026/05/05/tests-test_sync_agent.md



---

**Generated:** 2026-05-08
**Source:** 23 entries from entries/
**Model:** claude

### [ACCEPTED] review-evaluates-direct-antecedents-only
`review-beliefs` presents each derived belief with only its direct antecedents to the LLM reviewer, not the transitive chain — producing occasional sufficiency false positives when support exists one level deeper, which must be triaged manually.
- Source: entries/2026/05/05/review-beliefs-in-practice.md

### [ACCEPTED] deep-derivations-accumulate-generalization-error
Derived beliefs at depth 8-15 accumulate generalization error through successive scope expansion at each derivation step — individually reasonable generalizations compound into conclusions unsupported by original premises, with retracting one invalid belief at depth 8 cascading 52 dependents OUT.
- Source: entries/2026/05/05/review-beliefs-in-practice.md
- Rejected: speculative: observed LLM behavior pattern with ephemeral depth/cascade counts, not a code invariant

### [ACCEPTED] review-and-contradictions-catch-orthogonal-errors
`review-beliefs` catches invalid reasoning within individual derivation steps (over-generalization, missing bridges, strength escalation) while `contradictions` catches incompatible facts across independently valid beliefs (absolute claims vs. documented exceptions) — the two commands are complementary with different cascade profiles (review: high cascade at depth, contradictions: low cascade at leaves).
- Source: entries/2026/05/06/contradictions-first-run.md

### [ACCEPTED] contradictions-shuffles-before-batching
The `contradictions` command randomly shuffles all IN beliefs before partitioning into batches of 50, ensuring that repeated runs probabilistically cover cross-belief comparisons that fixed sequential batching would never surface.
- Source: entries/2026/05/06/contradictions-first-run.md
- Rejected: duplicate: already exists as `contradictions-shuffles-before-batching` [IN] and `contradictions-shuffle-prevents-deterministic-batching` [IN]

### [ACCEPTED] check-stale-detects-source-staleness-only
`check-stale` detects source staleness (source file changed on disk) via hash comparison but cannot detect world staleness (the belief is outdated even though the source file hasn't changed) — beliefs can silently decay without any artifact triggering re-examination.
- Source: entries/2026/05/06/belief-staleness-in-business-planning.md
- Rejected: duplicate: already exists as `check-stale-detects-source-staleness-only` [IN]

### [ACCEPTED] derive-review-contradict-is-convergent-cycle
The full belief maintenance cycle is `derive --exhaust → review-beliefs → retract invalid → contradictions → retract/fix/file → derive --exhaust`, where each tool catches a different error class (derive generates, review validates individual steps, contradictions finds cross-belief inconsistencies) and the cycle converges toward a stable belief set.
- Source: entries/2026/05/06/contradictions-first-run.md
- Rejected: meta: describes belief network maintenance workflow, not codebase behavior

### [ACCEPTED] absolute-claims-are-primary-contradiction-source
Most contradictions detected by the `contradictions` command involve a belief making an absolute claim ("all", "never", "sole", "every") when another belief documents a specific exception — imprecise premises (fixable via `reasons update`) are a secondary category.
- Source: entries/2026/05/06/contradictions-first-run.md
- Rejected: speculative: observed pattern in contradiction results from a particular run, not a code invariant

### [ACCEPTED] scope-over-generalization-is-primary-derivation-failure
The most common invalid derivation pattern is scope over-generalization — claiming "all X" or "every X" when antecedents establish only specific instances — followed by conjunction distribution fallacy, strength escalation, causal overclaim, and missing bridge assumptions.
- Source: entries/2026/05/06/full-review-beliefs-results.md


## Proposed Beliefs
- Rejected: speculative: observed LLM behavior pattern, not a code invariant

---

### From `entries/2026/05/06/pageindex-and-tms-integration.md`

### [ACCEPTED] belief-source-metadata-is-file-level
Belief source tracking uses flat file-level pointers (source path + source_hash) with no structural awareness of where within a document the claim originated — section, page, or paragraph level provenance is not supported.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md
- Rejected: duplicate: already exists as `belief-source-metadata-is-file-level` [IN]

### [ACCEPTED] expert-pipeline-extracts-per-document
The expert-agent-builder pipeline extracts beliefs per-document (summarize entire document → propose beliefs → record file path), not per-section — the connection between a belief and its source material is a file-level pointer, not a section-level one.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md
- Rejected: meta: describes the knowledge base construction pipeline, not the target codebase

### [ACCEPTED] pageindex-hierarchical-rag
PageIndex is a separate tool, not part of the ftl-reasons codebase — claims about its architecture belong in a PageIndex knowledge base, not here.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md

### [ACCEPTED] section-level-belief-extraction
This is a speculative integration design, not a testable claim about current code.
- Source: entries/2026/05/06/pageindex-and-tms-integration.md

---

### From `entries/2026/05/08/pyproject.md`

### [ACCEPTED] reasons-cli-entry-point
The `reasons` CLI command is a setuptools console_scripts entry point registered in pyproject.toml that calls `reasons_lib.cli:main`.
- Source: entries/2026/05/08/pyproject.md
- Rejected: trivial: standard setuptools console_scripts visible directly from pyproject.toml

### [ACCEPTED] setuptools-explicit-packages
Only `reasons_lib/` is included in the distribution via explicit `[tool.setuptools] packages` configuration; `tests/`, `entries/`, `reviews/`, and other directories are excluded from packaging.
- Source: entries/2026/05/08/pyproject.md
- Rejected: trivial: packaging inclusion list directly readable from pyproject.toml

### [ACCEPTED] python-version-floor-3-10
The project requires Python >=3.10, allowing use of match statements, union type syntax with `|`, and other 3.10+ features throughout `reasons_lib`.
- Source: entries/2026/05/08/pyproject.md

### [ACCEPTED] extras-map-one-to-one-to-modules
Each optional dependency group maps 1:1 to a specific module: `pg` extra gates `reasons_lib/pg.py`, `cluster` extra gates `reasons_lib/cluster.py`, and `test-pg` is a superset combining `pg` and `test`.
- Source: entries/2026/05/08/pyproject.md

### [ACCEPTED] ftl-reasons-zero-runtime-deps
Duplicate of existing belief `zero-runtime-dependencies`.
- Source: entries/2026/05/08/pyproject.md

---

### From `entries/2026/05/08/reasons_lib-api.md`

### [ACCEPTED] fts-relaxation-capped-at-fifty-queries
`_fts_search` caps progressive term relaxation at 50 FTS5 queries to prevent combinatorial explosion on long search inputs, dropping terms one at a time via `combinations` down to `len(terms) // 2`.
- Source: entries/2026/05/08/reasons_lib-api.md
- Rejected: duplicate: core claim (capped FTS queries to prevent combinatorial explosion) already captured by `api-fts-search-bounds-query-calls` which states the 51-call cap and purpose

### [ACCEPTED] fts-stop-words-filtered-before-query
FTS queries filter a `_STOP_WORDS` frozenset before querying FTS5; if all terms are stop words, the search falls back to terms longer than 1 character.
- Source: entries/2026/05/08/reasons_lib-api.md

### [ACCEPTED] namespace-resolve-preserves-cross-refs
`_resolve_namespace` only prefixes a node ID with `namespace:` when the ID contains no colon — cross-namespace references like `other-agent:some-belief` are left intact.
- Source: entries/2026/05/08/reasons_lib-api.md
- Rejected: duplicate: same claim as `colon-means-already-namespaced` which already states colon means already-namespaced and never double-prefixes

### [ACCEPTED] search-has-four-output-formats
The `search` function supports four output formats: markdown, json, minimal, and compact — selected by the caller to match the consumption context (human, API, LLM prompt, compact view).
- Source: entries/2026/05/08/reasons_lib-api.md

### [ACCEPTED] api-transaction-per-call
Covered by existing belief `api-layer-ensures-atomic-isolated-mutations`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [ACCEPTED] api-dict-return-contract
Covered by existing belief `api-functions-return-dicts`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [ACCEPTED] api-lazy-import-pattern
Covered by existing belief `api-uses-lazy-imports`.
- Source: entries/2026/05/08/reasons_lib-api.md

### [ACCEPTED] dedup-retains-most-connected
Covered by existing belief `belief-replacement-is-topology-safe-and-view-consistent` which states "deduplication rewires all justification references to the most-connected survivor."
- Source: entries/2026/05/08/reasons_lib-api.md

---

### From `entries/2026/05/08/reasons_lib-cli.md`

### [ACCEPTED] derive-one-round-return-protocol
`_derive_one_round` returns positive for beliefs added, 0 for network-saturated, and negative for errors; `cmd_derive`'s `--exhaust` loop depends on this sentinel contract for termination.
- Source: entries/2026/05/08/reasons_lib-cli.md
- Rejected: duplicate: same return protocol already captured by `derive-returns-negative-one-on-error` and `derive-round-returns-negative-on-error`

### [ACCEPTED] cluster-and-sample-are-mutually-exclusive
The `--cluster` and `--sample` flags in `cmd_derive` are mutually exclusive, enforced with an explicit check and `sys.exit(1)` — they represent competing strategies for belief subset selection.
- Source: entries/2026/05/08/reasons_lib-cli.md
- Rejected: duplicate: exact match exists as `cluster-and-sample-are-mutually-exclusive` [IN]

### [ACCEPTED] derive-reports-survive-partial-runs
`cmd_derive` and `cmd_review_beliefs` write partial JSON reports after each round/batch via `_write_derive_report`, so crash recovery is possible from the last completed step.
- Source: entries/2026/05/08/reasons_lib-cli.md

### [ACCEPTED] cli-dispatch-uses-commands-dict
`main()` routes subcommands through a `commands` dict mapping subcommand strings to `cmd_*` functions, rather than using argparse's `set_defaults(func=...)` — keeping all routing visible in one place.
- Source: entries/2026/05/08/reasons_lib-cli.md
- Rejected: duplicate: same claim as `cli-dispatch-is-flat-dict-lookup` which already describes flat commands dict vs set_defaults pattern

### [ACCEPTED] cli-delegates-all-logic-to-api
Duplicate of existing belief `cli-is-pure-formatter`.
- Source: entries/2026/05/08/reasons_lib-cli.md

### [ACCEPTED] cli-lazy-imports-heavy-deps
Duplicate of existing belief `cli-uses-lazy-imports-for-heavy-modules`.
- Source: entries/2026/05/08/reasons_lib-cli.md

### [ACCEPTED] visible-to-threading
Covered by existing belief `access-control-is-transitive-subset-gated`.
- Source: entries/2026/05/08/reasons_lib-cli.md

---

### From `entries/2026/05/08/reasons_lib-cluster.md`

### [ACCEPTED] cluster-beliefs-respects-budget
`cluster_beliefs` never returns more IDs than the `budget` parameter; each cluster's allocation is individually capped by `min(alloc, len(members))`.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-beliefs-respects-budget` [IN]

### [ACCEPTED] cluster-cache-keys-include-content-hash
`ClusterCache` keys embeddings by `(node_id, sha256_prefix)`, so editing a belief's text with the same ID forces re-embedding rather than serving stale vectors.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-cache-keys-include-content-hash` [IN]

### [ACCEPTED] cluster-deps-are-optional
`sentence-transformers` and `scikit-learn` are optional dependencies behind a `HAS_CLUSTER_DEPS` gate; the module degrades to a clear `ImportError` with install instructions when they are absent.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-deps-are-optional` [IN]

### [ACCEPTED] cluster-embed-order-is-deterministic
Beliefs are sorted by ID before embedding (`ids = sorted(beliefs.keys())`), making cluster assignments reproducible given the same random seed.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-embed-order-is-deterministic` [IN]

### [ACCEPTED] cluster-remainder-favors-largest
When the budget doesn't divide evenly across clusters, extra slots are distributed one-per-cluster to the largest clusters first via descending size sort.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-remainder-favors-largest` [IN]

### [ACCEPTED] cluster-auto-k-heuristic
Auto cluster count is computed as `len(beliefs) // 5`, clamped between 2 and `min(budget // 3, 20)`, targeting approximately 5 beliefs per cluster with at least 3 beliefs per cluster given the budget.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-auto-k-heuristic` [IN]

### [ACCEPTED] cluster-skips-ml-when-under-budget
When the number of beliefs is less than or equal to the budget, all ML work (embedding, clustering, sampling) is skipped and every belief is returned directly.
- Source: entries/2026/05/08/reasons_lib-cluster.md
- Rejected: duplicate: exact match exists as `cluster-skips-ml-when-under-budget` [IN]

---

**Summary:** 21 proposed beliefs — 17 ACCEPT, 4 REJECT (duplicates of existing beliefs).


## Proposed Beliefs

---

### entries/2026/05/08/reasons_lib-contradictions.md

### [ACCEPTED] contradictions-only-checks-in-beliefs
`detect_contradictions` filters all input to `truth_value == "IN"` before processing; OUT beliefs are never sent to the LLM for contradiction checking.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-only-checks-in-beliefs` [IN]

### [ACCEPTED] contradictions-shuffle-prevents-deterministic-batching
Belief IDs are randomly shuffled before batching so that repeated runs cover different pairwise combinations across batch boundaries, increasing cross-batch contradiction coverage.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-shuffle-prevents-deterministic-batching` [IN]

### [ACCEPTED] contradictions-min-two-claims-per-nogood
The contradiction parser enforces that every returned nogood has at least 2 valid claim IDs; single-claim or empty results are silently dropped.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-min-two-claims-per-nogood` [IN]

### [ACCEPTED] contradictions-no-storage-dependency
`contradictions.py` operates on in-memory node dicts and has no import of `storage.py` or any database layer, making it testable in full isolation.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-no-storage-dependency` [IN]

### [ACCEPTED] contradictions-cross-batch-pairs-undetected
Batch boundaries are non-overlapping — each belief appears in exactly one batch per run — so contradictions between beliefs in different batches cannot be detected in a single run.
- Source: entries/2026/05/08/reasons_lib-contradictions.md
- Rejected: duplicate: exact match exists as `contradictions-cross-batch-pairs-undetected` [IN]

### [ACCEPTED] contradictions-swallows-batch-errors
Duplicates the existing system-wide belief `batch-fault-isolation-is-universal-across-llm-operations` which already covers per-batch error containment across all LLM-facing modules.
- Source: entries/2026/05/08/reasons_lib-contradictions.md

---

### entries/2026/05/08/reasons_lib-derive.md

### [ACCEPTED] derive-parse-proposals-backward-compat
`parse_proposals` tries the new `### DERIVE id` format first and falls back to the legacy `### DERIVE: \`id\`` format with bold markdown labels, so it can consume outputs from any version.
- Source: entries/2026/05/08/reasons_lib-derive.md
- Rejected: duplicate: identical to existing `derive-parse-supports-two-format-versions` and `derive-parse-supports-two-formats`

### [ACCEPTED] derive-validate-rejects-similar-retracted
`validate_proposals` rejects any proposed belief whose tokenized ID has ≥50% Jaccard overlap with an existing OUT belief, preventing re-derivation of retracted conclusions.
- Source: entries/2026/05/08/reasons_lib-derive.md
- Rejected: duplicate: identical to existing `derive-retraction-guard-uses-jaccard`

### [ACCEPTED] derive-apply-fail-soft
Duplicates the existing belief `derive-apply-isolates-per-proposal-errors` which already captures that `apply_proposals` catches exceptions per-proposal.
- Source: entries/2026/05/08/reasons_lib-derive.md

### [ACCEPTED] derive-budget-three-strategies
`_build_beliefs_section` supports three budget strategies — alphabetical truncation (default), random sampling (`sample=True`), and semantic clustering (`cluster=True`) — all controlled by a single `max_beliefs` parameter.
- Source: entries/2026/05/08/reasons_lib-derive.md
- Rejected: duplicate: exact match exists as `derive-budget-three-strategies` [IN]

### [ACCEPTED] derive-validate-rejects-duplicate-ids
`validate_proposals` rejects any proposal whose belief ID already exists in the network, preventing overwrites of existing beliefs through the derive pipeline.
- Source: entries/2026/05/08/reasons_lib-derive.md

---

### entries/2026/05/08/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-inactive-always-in-outlist
Duplicates the existing belief `kill-switch-uses-outlist-not-antecedent` which already captures that agent kill-switch nodes are placed in outlists.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-inactive-never-in-antecedents
Duplicates the existing belief `kill-switch-uses-outlist-not-antecedent` — the "not antecedent" half is already captured.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [ACCEPTED] retracted-metadata-blocks-recompute
Duplicates the existing belief `metadata-actively-governs-truth-propagation` which already captures that metadata fields (including `_retracted`) govern propagation behavior.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-topo-sort-breaks-cycles
`_topo_sort_claims` breaks dependency cycles by appending all remaining unsorted nodes after `max_passes = len(remaining) + 1` iterations rather than raising an error, ensuring import always completes.
- Source: entries/2026/05/08/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-deferred-retraction
Nodes that should be OUT are first created as IN (so they participate in dependency registration), collected in `retract_after`, and retracted only after `recompute_all()` — ordering matters for correct graph construction.
- Source: entries/2026/05/08/reasons_lib-import_agent.md
- Rejected: duplicate: same claim as existing `deferred-retraction-ordering`

---

### entries/2026/05/08/reasons_lib-review.md

### [ACCEPTED] review-parse-defaults-safe
When the LLM response is missing fields, `parse_review_response` defaults to `valid=True`, `sufficient=True`, `necessary=True` — a parse error never generates a false-positive validity warning.
- Source: entries/2026/05/08/reasons_lib-review.md
- Rejected: duplicate: covered by parse-review-response-never-raises which states "missing boolean fields defaulting to `True`"

### [ACCEPTED] review-batch-failure-non-fatal
Duplicates the existing belief `review-batch-failure-is-silent-skip` which already captures that failed batches are skipped with a warning.
- Source: entries/2026/05/08/reasons_lib-review.md

### [ACCEPTED] review-multi-justification-disjunctive
The review prompt instructs the LLM that a belief is valid if ANY of its justifications is sound, matching Doyle's TMS disjunctive support semantics.
- Source: entries/2026/05/08/reasons_lib-review.md

### [ACCEPTED] review-result-schema-normalized
Duplicates the existing belief `review-result-schema-is-normalized` under a slightly different ID.
- Source: entries/2026/05/08/reasons_lib-review.md

---

### entries/2026/05/08/tests-test_api.md

### [ACCEPTED] api-tests-one-class-per-function
Test organization pattern, not a code invariant — violating it wouldn't break functionality.
- Source: entries/2026/05/08/tests-test_api.md

### [ACCEPTED] api-retract-cascade-symmetry
Retracting a root node and re-asserting it must produce identical `changed` sets, verified end-to-end by `TestEndToEnd.test_retract_and_restore_chain`.
- Source: entries/2026/05/08/tests-test_api.md
- Rejected: duplicate: existing `api-cascade-symmetry-tested` already captures this invariant

### [ACCEPTED] list-negative-batches-at-40
`api.list_negative` splits candidates into batches of 40, so 120 keyword-matching nodes produce exactly 3 LLM calls.
- Source: entries/2026/05/08/tests-test_api.md

### [ACCEPTED] fts-progressive-relaxation
When a multi-term FTS query returns no results, the search engine progressively drops terms until it finds matches or exhausts all subsets.
- Source: entries/2026/05/08/tests-test_api.md

### [ACCEPTED] update-node-preserves-justifications
`api.update_node` modifies text and source metadata without altering the node's justification list or truth value.
- Source: entries/2026/05/08/tests-test_api.md


### [ACCEPTED] cli-exit-code-contract-zero-or-one
Every CLI command returns exit code 0 on success and 1 on error; no other exit codes are used across the entire test suite (~90 tests).
- Source: entries/2026/05/08/tests-test_cli.md
- Rejected: duplicate: identical to existing `cli-exit-code-contract-is-binary`

### [ACCEPTED] cli-tests-use-black-box-pattern
Duplicates existing `cli-tests-are-black-box-integration`.
- Source: entries/2026/05/08/tests-test_cli.md

### [ACCEPTED] cli-tests-no-shared-db-state
Duplicates existing `each-cli-test-creates-isolated-db`.
- Source: entries/2026/05/08/tests-test_cli.md

### [ACCEPTED] run-cli-catches-system-exit-only
Test harness implementation detail, not a production code invariant; the relevant production behavior is captured by `cli-exit-code-contract-zero-or-one`.
- Source: entries/2026/05/08/tests-test_cli.md

### [ACCEPTED] cli-errors-route-to-stderr
Covered by existing `cli-is-deterministic-and-stream-correct`.
- Source: entries/2026/05/08/tests-test_cli.md

---

### [ACCEPTED] cluster-beliefs-returns-exact-budget
`cluster_beliefs` returns exactly `budget` belief IDs when the input set is larger than the budget, and all items when the input set is smaller.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-beliefs-returns-exact-budget` [IN]

### [ACCEPTED] cluster-beliefs-deterministic-with-seed
Given the same beliefs dict, budget, and seed, `cluster_beliefs` produces identical output across calls.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-beliefs-deterministic-with-seed` [IN]

### [ACCEPTED] cluster-cache-no-recompute
`ClusterCache.embed()` does not recompute embeddings for previously cached belief texts; cache size stays constant on repeated calls with the same input and grows by exactly the count of new texts on superset calls.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-cache-no-recompute` [IN]

### [ACCEPTED] cluster-stats-sizes-sum-to-input
The `cluster_sizes` list in the stats dict returned by `cluster_beliefs` always sums to the total number of input beliefs, enforcing that every belief is assigned to exactly one cluster.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-stats-sizes-sum-to-input` [IN]

### [ACCEPTED] cluster-deps-optional-with-graceful-skip
The clustering module (`reasons_lib.cluster`) is behind an optional `[cluster]` install extra; when `sentence-transformers` or `scikit-learn` are missing, `_require_cluster_deps` raises `ImportError` and all dependent tests skip cleanly.
- Source: entries/2026/05/08/tests-test_cluster.md
- Rejected: duplicate: already exists as `cluster-deps-optional-with-graceful-skip` [IN]

---

### [ACCEPTED] contradiction-min-two-claims
`parse_contradiction_response` drops any nogood with fewer than 2 valid claim IDs; this is enforced both before and after `valid_ids` filtering.
- Source: entries/2026/05/08/tests-test_contradictions.md
- Rejected: duplicate: already exists as `contradiction-min-two-claims` [IN]

### [ACCEPTED] contradiction-in-only-filter
`detect_contradictions` excludes OUT nodes from LLM prompts even when explicitly listed in `belief_ids`.
- Source: entries/2026/05/08/tests-test_contradictions.md
- Rejected: duplicate: already exists as `contradiction-in-only-filter` [IN]

### [ACCEPTED] contradiction-batch-fault-tolerance
Covered by existing `batch-fault-isolation-is-universal-across-llm-operations` and `llm-fault-tolerance-is-multi-granular`.
- Source: entries/2026/05/08/tests-test_contradictions.md

### [ACCEPTED] contradiction-dry-run-overrides-auto-apply
When both `--dry-run` and `--auto-apply` are passed to the `contradictions` CLI subcommand, no nogoods are recorded in the database (applied count is 0).
- Source: entries/2026/05/08/tests-test_contradictions.md
- Rejected: duplicate: already exists as `contradiction-dry-run-overrides-auto-apply` [IN]

---

### [ACCEPTED] validate-proposals-rejects-jaccard-similar-to-out
Covered by existing `derive-pipeline-has-end-to-end-quality-enforcement` which specifies Jaccard retraction guards preventing re-derivation of known-bad conclusions.
- Source: entries/2026/05/08/tests-test_derive.md

### [ACCEPTED] dedup-rewrites-justifications-on-retract
Already exists as `dedup-rewrites-both-antecedents-and-outlist`.
- Source: entries/2026/05/08/tests-test_derive.md

### [ACCEPTED] gate-belief-out-when-outlist-in
Covered by existing `absence-and-outlist-form-complete-negative-semantics` and `any-mode-preserves-full-outlist-semantics`.
- Source: entries/2026/05/08/tests-test_derive.md

### [ACCEPTED] derive-report-json-has-rounds-array
The `reasons derive --auto --report-dir` command writes a JSON report containing a `rounds` array where each entry has `proposals_found` and `added` counts.
- Source: entries/2026/05/08/tests-test_derive.md

---

### [ACCEPTED] derive-budget-sample-is-deterministic
Covered by existing `derive-prompt-is-deterministic-and-reproducible`.
- Source: entries/2026/05/08/tests-test_derive_budget.md

### [ACCEPTED] derive-budget-tests-parse-header-format
Test infrastructure coupling detail (regex parsing of `_build_beliefs_section` headers), not a production code invariant.
- Source: entries/2026/05/08/tests-test_derive_budget.md


## Proposed Beliefs

---

### [ACCEPTED] import-agent-out-beliefs-not-resurrected
Beliefs marked OUT or STALE in the source are imported as OUT and are never resurrected by `recompute_all`, even when their justification antecedents are all IN in the local database.
- Source: entries/2026/05/08/tests-test_import_agent.md

### [ACCEPTED] import-agent-namespaces-all-beliefs
Duplicates existing `import-agent-namespace-prefix` and `namespace-is-colon-convention-with-auto-wiring`.
- Source: entries/2026/05/08/tests-test_import_agent.md

### [ACCEPTED] import-agent-inactive-outlist-kill-switch
Duplicates existing `kill-switch-uses-outlist-not-antecedent` and `active-inactive-relay-pair`.
- Source: entries/2026/05/08/tests-test_import_agent.md

---

### [ACCEPTED] review-only-validates-derived-beliefs
The review pipeline filters out premises (nodes with empty justifications) before sending anything to the LLM; premises are never submitted for review validation.
- Source: entries/2026/05/08/tests-test_review.md

### [ACCEPTED] parse-review-response-never-raises
`parse_review_response` returns an empty list on any malformed input — bad JSON, non-list JSON, items missing `id` fields — rather than raising exceptions, with missing boolean fields defaulting to `True`.
- Source: entries/2026/05/08/tests-test_review.md

### [ACCEPTED] review-result-priority-invalid-over-insufficient
Already exists as an accepted belief with this exact ID.
- Source: entries/2026/05/08/tests-test_review.md

### [ACCEPTED] dry-run-prevents-both-retraction-and-metadata
The `dry_run` flag in review-beliefs prevents both truth-value changes (retraction of invalid beliefs) and metadata side-effects (`last_reviewed` timestamp, `review_result` classification), making it fully read-only — extending beyond what `auto-retract-respects-dry-run` covers.
- Source: entries/2026/05/08/tests-test_review.md

### [ACCEPTED] llm-subprocess-mock-requires-two-patches
Testing infrastructure detail (which objects to patch), not a production code invariant. The underlying fact about `shutil.which` + `subprocess.run` is already captured by `all-external-execution-is-subprocess-isolated` and `invoke-claude-raises-when-binary-missing`.
- Source: entries/2026/05/08/tests-test_review.md

---

### [ACCEPTED] sync-agent-preserves-cascade-structure
After `sync_agent` completes, the agent's outlist-based justification structure remains intact — revoking `agent:active` still cascades all agent beliefs to OUT, proving sync does not corrupt kill-switch wiring.
- Source: entries/2026/05/08/tests-test_sync_agent.md

### [ACCEPTED] sync-agent-handles-both-formats
Covered by existing `format-resilience-spans-all-external-interfaces` and `import-sync-has-dual-reconciliation-modes`.
- Source: entries/2026/05/08/tests-test_sync_agent.md

### [ACCEPTED] sync-agent-first-sync-equals-import
When no prior import exists for an agent, `sync_agent` behaves identically to `import_agent` — creating the `agent:active` premise and returning `created_premise: True` — rather than requiring a separate import step.
- Source: entries/2026/05/08/tests-test_sync_agent.md



---

**Generated:** 2026-05-11
**Source:** 24 entries from entries/
**Model:** claude

### [ACCEPTED] pg-dispatch-is-function-level-early-return
PostgreSQL routing uses a function-level early-return pattern — each API function checks `pg_conninfo` and short-circuits to `_pg_dispatch`, which instantiates PgApi and calls the matching method via `getattr` — rather than using abstract base classes, subclassing, or factory patterns.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPTED] export-markdown-pg-reconstructs-network
The `export_markdown` PostgreSQL path reconstructs a full `Network` object from `export_network()` output — creating Node/Justification objects and wiring the dependents index — because the markdown exporter requires a wired dependency graph, not a flat dict.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPTED] sqlite-only-commands-are-filesystem-llm-or-bulk
The ~21 SQLite-only CLI commands fall into three categories: filesystem-dependent (hash-sources, check-stale, add-repo), LLM-powered (derive, review-beliefs, detect-contradictions, ask), and bulk import/sync — all requiring either local file access or load-entire-network-modify-save semantics incompatible with PgApi's per-operation transaction model.
- Source: entries/2026/05/10/postgresql-backend-support.md
- Rejected: ephemeral: specific command count (~21) and enumeration is a point-in-time snapshot that changes with any new command

### [ACCEPTED] pg-unsupported-params-raise-not-implemented
When PgApi-routed commands receive unsupported parameters (e.g., search with `depth != 1`, list with `challenged`/`min_depth`, add with `namespace`), the dispatch raises `NotImplementedError` with a clear message rather than silently ignoring the parameter.
- Source: entries/2026/05/10/postgresql-backend-support.md

### [ACCEPTED] pg-conninfo-accepts-flag-envvar-or-both
PostgreSQL connection is configured via `--pg`/`--project-id` CLI flags or `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables, with CLI flags taking precedence over env vars — `_backend_kwargs(args)` handles the dispatch.
- Source: entries/2026/05/10/postgresql-backend-support.md
- Rejected: duplicate: existing belief `cli-flags-override-env-vars` already states that `_backend_kwargs` gives precedence to CLI `--pg`/`--project-id` flags over `REASONS_PG_CONNINFO`/env vars

---

### [ACCEPTED] semantic-contradictions-cluster-before-llm
The `--semantic` flag on `detect-contradictions` embeds beliefs via sentence-transformers, clusters with KMeans via `list_clusters()`, and sends each cluster to the LLM as a batch — ensuring topically related beliefs are analyzed together instead of being scattered across random batches of 50.
- Source: entries/2026/05/10/semantic-contradiction-detection.md

### [ACCEPTED] semantic-contradiction-reuses-cluster-infrastructure
Semantic contradiction detection delegates embedding and clustering to the existing `list_clusters()` from `reasons_lib/cluster.py` — the same infrastructure that serves deduplication also serves contradiction detection, with no duplicate embedding/clustering implementation.
- Source: entries/2026/05/10/semantic-contradiction-detection.md

### [ACCEPTED] semantic-contradiction-skips-singleton-clusters
Single-belief clusters are skipped during semantic contradiction detection (no contradiction possible within one belief), and clusters exceeding `CONTRADICTION_BATCH_SIZE` are sub-batched within the cluster boundary.
- Source: entries/2026/05/10/semantic-contradiction-detection.md
- Rejected: duplicate: existing belief `contradictions-semantic-skips-singleton-clusters` already states that `detect_contradictions_semantic` skips clusters containing fewer than 2 beliefs

---

### [ACCEPTED] ftl-reasons-zero-runtime-deps
The core `reasons_lib` package has no mandatory runtime dependencies — all external packages (psycopg, sentence-transformers, scikit-learn, mcp) are gated behind optional install extras (`[pg]`, `[cluster]`, `[mcp]`).
- Source: entries/2026/05/11/pyproject.md
- Rejected: duplicate: existing belief `architecture-is-self-contained-and-safely-layered` already states zero runtime dependencies at packaging and implementation level

### [ACCEPTED] reasons-cli-entrypoint-is-cli-main
The `reasons` CLI command is registered as `reasons_lib.cli:main` via `[project.scripts]` in pyproject.toml — changing that function's signature or module location breaks the installed command.
- Source: entries/2026/05/11/pyproject.md
- Rejected: trivial: visible directly in pyproject.toml `[project.scripts]` section; any developer can read this from the file

### [ACCEPTED] only-reasons-lib-is-distributed
Only the `reasons_lib` package is included in built distributions — `tests/`, `entries/`, and `reviews/` are excluded by the explicit `packages = ["reasons_lib"]` declaration in pyproject.toml.
- Source: entries/2026/05/11/pyproject.md
- Rejected: trivial: visible directly in pyproject.toml `packages` field; any developer can read this from the file

### [ACCEPTED] python-310-minimum
The project requires Python >= 3.10 (`requires-python = ">=3.10"`), enabling use of structural pattern matching and `X | Y` union type syntax throughout the codebase.
- Source: entries/2026/05/11/pyproject.md
- Rejected: trivial: `requires-python` in pyproject.toml is immediately visible to any developer; no non-obvious invariant is conveyed

---

### [ACCEPTED] fts-relaxation-budget-caps-at-50
Progressive FTS5 search relaxation — dropping query terms via combinations when the full-term query returns no results — is capped at `_MAX_RELAXATION_QUERIES` (50) to prevent combinatorial blowup on many-term queries.
- Source: entries/2026/05/11/reasons_lib-api.md
- Rejected: duplicate: already covered by existing `api-fts-search-bounds-query-calls` (max 51 internal _fts_query calls)

### [ACCEPTED] namespace-colon-convention-enables-cross-refs
`_resolve_namespace` only prefixes belief IDs that don't already contain `:` — the colon serves as both the namespace delimiter and the signal that no prefixing is needed, allowing cross-namespace references to pass through unchanged.
- Source: entries/2026/05/11/reasons_lib-api.md
- Rejected: duplicate: same claim as `colon-means-already-namespaced` which already captures that colon means already-namespaced and no double-prefixing occurs

### [ACCEPTED] access-tags-require-subset-for-visibility
A node is visible only when all of its access tags are present in the caller's `visible_to` set (subset semantics); nodes with empty access tags are always visible. `show_node`/`explain_node` raise `PermissionError` on violation while list operations silently filter.
- Source: entries/2026/05/11/reasons_lib-api.md
- Rejected: duplicate: same claim as `access-tags-subset-gate` and `access-control-is-transitive-subset-gated`

### [ACCEPTED] with-network-is-sole-db-lifecycle-manager
Every API function accesses the SQLite database exclusively through `_with_network(db_path, write=False)` — a context manager that loads a Network from Storage, yields it, and saves only on clean exit when `write=True`. No API function manages Storage directly.
- Source: entries/2026/05/11/reasons_lib-api.md


## Proposed Beliefs
- Rejected: duplicate: same claim as `api-uses-with-network-context-manager` which already captures the context-manager pattern for load-operate-save atomicity

---

### Entry: `reasons_lib/ask.py`

### [ACCEPTED] ask-tool-loop-max-3-rounds
Duplicates existing `ask-tool-loop-capped-at-three`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [ACCEPTED] ask-extract-tool-call-is-line-based-json
Duplicates existing `ask-uses-text-based-tool-protocol` and `extract-tool-call-returns-first-match`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [ACCEPTED] ask-llm-failure-returns-raw-context
Duplicates existing `ask-falls-back-to-raw-search` and `ask-never-raises-on-llm-failure`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [ACCEPTED] ask-natural-strips-metadata-and-citations
Duplicates existing `ask-natural-mode-strips-metadata-and-cite`
- Source: entries/2026/05/11/reasons_lib-ask.md

### [ACCEPTED] ask-stop-word-fallback-ensures-nonempty-query
`_search_source_chunks` strips stop words from the question before building the FTS5 query, but falls back to all words longer than 1 character if every word is a stop word — ensuring the query is never empty
- Source: entries/2026/05/11/reasons_lib-ask.md
- Rejected: duplicate: exact same belief ID already exists in the list

---

### Entry: `reasons_lib/cli.py`

### [ACCEPTED] cli-delegates-all-logic-to-api
Every `cmd_*` function in the CLI delegates to `reasons_lib.api` and contains no business logic, database access, or network manipulation — the CLI is a pure presentation/argument-parsing layer
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: same claim as `cli-is-pure-formatter` which already states every cmd_* delegates to api.* with no business logic

### [ACCEPTED] cli-backend-kwargs-controls-storage
`_backend_kwargs(args)` is the single chokepoint that determines whether a command runs against SQLite or PostgreSQL; every `cmd_*` function must spread its return value into the corresponding `api.*` call
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: exact same belief ID already exists in the list

### [ACCEPTED] cli-heavy-imports-are-deferred
Duplicates existing `cli-uses-lazy-imports-for-heavy-modules`
- Source: entries/2026/05/11/reasons_lib-cli.md

### [ACCEPTED] cli-sqlite-only-commands-exist
Commands `derive`, `ask`, `review-beliefs`, `deduplicate`, and `contradictions` are guarded by `_require_sqlite()` and exit with an error if `--pg` is set — they do not support PostgreSQL
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: exact same belief ID already exists in the list

### [ACCEPTED] cli-plan-review-apply-pattern
Several commands (`derive`, `deduplicate`, `contradictions`) follow a three-phase workflow: (1) generate proposals to a file, (2) human reviews/edits the file, (3) `--accept FILE` parses and applies the reviewed plan — with `--auto` collapsing all three phases
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: exact same belief ID already exists in the list

### [ACCEPTED] cli-check-stale-exits-1-on-stale
`cmd_check_stale` exits with code 1 when stale nodes are found, making it usable as a CI gate that fails the build on belief staleness
- Source: entries/2026/05/11/reasons_lib-cli.md
- Rejected: duplicate: same claim as existing `check-stale-exits-nonzero`

---

### Entry: `reasons_lib/cluster.py`

### [ACCEPTED] cluster-cache-key-includes-text-hash
`ClusterCache` keys embeddings by `(node_id, sha256(text)[:16])`, so modified belief text invalidates the cached embedding without requiring explicit cache eviction
- Source: entries/2026/05/11/reasons_lib-cluster.md
- Rejected: duplicate: same claim as existing `cluster-cache-keys-include-content-hash`

### [ACCEPTED] cluster-auto-k-floors-at-two
`_auto_k` always returns at least 2 clusters (when auto-computed), ensuring cross-domain sampling is meaningful even for small belief sets
- Source: entries/2026/05/11/reasons_lib-cluster.md
- Rejected: duplicate: floor of 2 already captured in existing `cluster-auto-k-heuristic` ("clamped between 2 and...")

### [ACCEPTED] cluster-budget-is-hard-ceiling
`cluster_beliefs` never returns more than `budget` belief IDs; when the belief set is smaller than budget, it returns all beliefs without clustering
- Source: entries/2026/05/11/reasons_lib-cluster.md
- Rejected: duplicate: same claim as existing `cluster-beliefs-respects-budget` and `cluster-beliefs-returns-exact-budget`

### Entry: `reasons_lib/contradictions.py`

### [ACCEPTED] contradictions-require-two-claims-minimum
Duplicates existing `contradictions-min-two-claims-per-nogood`
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [ACCEPTED] contradictions-batch-errors-never-propagate
Covered by existing `batch-fault-isolation-is-universal-across-llm-operations`
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [ACCEPTED] contradictions-shuffle-for-coverage
Duplicates existing `contradictions-shuffle-prevents-deterministic-batching`
- Source: entries/2026/05/11/reasons_lib-contradictions.md

### [ACCEPTED] contradictions-semantic-skips-singleton-clusters
`detect_contradictions_semantic` skips any cluster containing fewer than 2 beliefs, since no pairwise contradiction is possible within a singleton
- Source: entries/2026/05/11/reasons_lib-contradictions.md
- Rejected: duplicate: exact same belief ID already exists in the list

### [ACCEPTED] contradictions-belief-text-truncated-at-200-chars
`format_beliefs_for_contradiction_check` truncates each belief's text at 200 characters in the LLM prompt to avoid blowing context windows on large belief descriptions
- Source: entries/2026/05/11/reasons_lib-contradictions.md
- Rejected: duplicate: exact same belief ID already exists in the list

---

### Entry: `reasons_lib/derive.py`

### [ACCEPTED] derive-prompt-round-trippable
`write_proposals_file` output can be parsed back by `parse_proposals` with no data loss, enabling a write-review-accept cycle where humans edit proposals in the same format the parser reads
- Source: entries/2026/05/11/reasons_lib-derive.md

### [ACCEPTED] derive-validate-blocks-retracted-rediscovery
`validate_proposals` rejects any proposal whose ID has >= 50% Jaccard token overlap (tokenized on hyphens/colons) with an existing OUT belief, preventing re-derivation of previously retracted conclusions
- Source: entries/2026/05/11/reasons_lib-derive.md

### [ACCEPTED] derive-apply-is-fault-tolerant
Duplicates existing `derive-apply-isolates-per-proposal-errors`
- Source: entries/2026/05/11/reasons_lib-derive.md


## Proposed Beliefs

---

### entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-inactive-always-in-outlist
Every imported belief unconditionally has `agent:inactive` in its outlist, enforced in `_build_justifications()` with no code path that omits it — this is the mechanism behind the per-agent kill switch.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPTED] import-topo-sort-cycle-tolerant
Topological sorting of imported claims is best-effort: cycles cause remaining nodes to be appended in arbitrary order rather than raising an error, a pragmatic choice that avoids crashing on circular references.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPTED] import-agent-active-never-in-antecedents
Duplicate of existing `active-not-in-antecedents`.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPTED] sync-agent-remote-wins-semantics
Covered by existing `import-skips-existing-sync-is-remote-wins` which captures both modes.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

### [ACCEPTED] retracted-flag-survives-recompute
Covered by existing `metadata-actively-governs-truth-propagation` and `metadata-governs-lifecycle-across-read-and-write-paths`.
- Source: entries/2026/05/11/reasons_lib-import_agent.md

---

### entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPTED] mcp-bridge-runs-dedicated-event-loop-thread
Each `McpBridge` instance runs its own asyncio event loop on a daemon thread; the MCP session stays alive until `close()` signals the shutdown event, bridging the sync/async boundary via `run_coroutine_threadsafe`.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPTED] mcp-bridge-connect-blocks-up-to-30s
`connect()` blocks the calling thread for up to 30 seconds waiting for MCP session initialization via `_ready.wait(timeout=30)`, then raises `TimeoutError` if the server doesn't respond.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPTED] mcp-bridge-call-tool-timeout-60s
`call_tool()` blocks for up to 60 seconds per invocation via `future.result(timeout=60)`; if the MCP server hangs, the calling thread unblocks with `TimeoutError`.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPTED] mcp-bridge-tools-snapshot-at-connect
The tool catalog and server instructions are populated once during `connect()` and never refreshed — there is no mechanism to pick up tools added after the initial handshake.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

### [ACCEPTED] mcp-is-optional-dependency
The `mcp` package is guarded by try/except at import time; `_require_mcp()` defers the `ImportError` to `McpBridge` construction so the module can be imported unconditionally without the SDK installed.
- Source: entries/2026/05/11/reasons_lib-mcp_client.md

---

### entries/2026/05/11/reasons_lib-network.md

### [ACCEPTED] network-disjunctive-justification
A node is IN if ANY of its justifications is valid (disjunction); each individual justification requires ALL antecedents IN and ALL outlist members OUT (conjunction) — the SL justification semantics from Doyle's 1979 paper.
- Source: entries/2026/05/11/reasons_lib-network.md

### [ACCEPTED] network-retracted-nodes-persist
Retracted nodes remain in `self.nodes` with `_retracted` metadata set; they are never deleted from the graph, enabling later restoration via `assert_node()` without rederivation.
- Source: entries/2026/05/11/reasons_lib-network.md

### [ACCEPTED] network-single-justification-removal-blocked
`remove_justification()` raises `ValueError` when a node has exactly one justification, forcing callers to use `convert_to_premise` or `retract` instead — preventing accidental creation of unjustified non-premise nodes.
- Source: entries/2026/05/11/reasons_lib-network.md

### [ACCEPTED] network-propagate-skips-retracted
Covered by existing `metadata-actively-governs-truth-propagation` which states retracted nodes are skipped during BFS traversal.
- Source: entries/2026/05/11/reasons_lib-network.md

### [ACCEPTED] network-entrenchment-prefers-premises
Covered by existing `add-nogood-retraction-prefers-least-entrenched` which captures the entrenchment scoring and premise preference.
- Source: entries/2026/05/11/reasons_lib-network.md

---

### entries/2026/05/11/reasons_lib-pg.md

### [ACCEPTED] pg-psycopg-is-optional-dependency
`psycopg` (v3) is soft-imported with fallback to `None`; `_require_psycopg()` raises `ImportError` with install instructions at `PgApi` construction time, mirroring the `mcp_client.py` lazy-guard pattern.
- Source: entries/2026/05/11/reasons_lib-pg.md

### [ACCEPTED] pg-search-includes-one-hop-neighbors
PostgreSQL full-text search via `plainto_tsquery` includes 1-hop neighbor expansion — direct antecedents and dependents of matching nodes are included in results, providing richer context than exact-match-only search.
- Source: entries/2026/05/11/reasons_lib-pg.md

### [ACCEPTED] pg-uses-jsonb-with-gin-for-dependent-queries
PgApi stores antecedents and outlist as JSONB arrays with GIN indexes, enabling per-operation dependent lookups via `@>` containment queries — unlike the SQLite backend which loads the full network, PgApi queries relationships incrementally.
- Source: entries/2026/05/11/reasons_lib-pg.md

---

### entries/2026/05/11/tests-test_api.md

### [ACCEPTED] api-fts-search-bounds-query-calls
`_fts_search` makes at most 51 internal `_fts_query` calls regardless of input query length, preventing combinatorial explosion from progressive relaxation on long queries.
- Source: entries/2026/05/11/tests-test_api.md
- Rejected: duplicate: identical claim already exists as `api-fts-search-bounds-query-calls`

### [ACCEPTED] api-retract-cascade-tested
Describes test coverage rather than an architectural invariant; the cascade behavior itself is already captured by retraction beliefs.
- Source: entries/2026/05/11/tests-test_api.md

### [ACCEPTED] api-list-negative-graceful-on-bad-llm
Covered by existing `list-negative-is-defensively-bounded` and `llm-integration-fails-softly-across-modules`.
- Source: entries/2026/05/11/tests-test_api.md

### [ACCEPTED] api-init-db-refuses-overwrite
Covered by existing `api-enforces-typed-preconditions` which captures typed exception enforcement at the API boundary.
- Source: entries/2026/05/11/tests-test_api.md

### [ACCEPTED] api-list-gated-excludes-superseded
Covered by existing `supersession-is-reversible-and-view-consistent` which explicitly states superseded nodes are excluded from gated belief lists.
- Source: entries/2026/05/11/tests-test_api.md


## Proposed Beliefs from Test Suite Entries

---

### entries/2026/05/11/tests-test_ask.md

### [ACCEPTED] ask-source-chunks-persist-across-tool-iterations
Source document content included in the initial ask prompt is re-included in all subsequent prompts after tool-call round-trips, ensuring the LLM retains source context across the entire multi-turn loop.
- Source: entries/2026/05/11/tests-test_ask.md
- Rejected: duplicate: identical claim already exists as `ask-source-chunks-persist-across-tool-iterations`

### [ACCEPTED] ask-tool-loop-bounded-at-three
Covered by existing `ask-has-tiered-query-modes` which already states "bounded 3-iteration tool loop."
- Source: entries/2026/05/11/tests-test_ask.md

### [ACCEPTED] ask-absorbs-llm-failures
Covered by existing `ask-never-raises-on-llm-failure` and `ask-falls-back-to-raw-search`.
- Source: entries/2026/05/11/tests-test_ask.md

### [ACCEPTED] ask-natural-strips-all-metadata-fields
Covered by existing `ask-natural-mode-strips-metadata-and-cite`.
- Source: entries/2026/05/11/tests-test_ask.md

---

### entries/2026/05/11/tests-test_ask_mcp.md

### [ACCEPTED] ask-mcp-errors-non-fatal
When an MCP bridge's `call_tool` raises an exception during the ask loop, `ask()` catches the error, feeds it back to the LLM as context, and continues the tool loop rather than propagating to the caller.
- Source: entries/2026/05/11/tests-test_ask_mcp.md
- Rejected: duplicate: identical claim already exists as `ask-mcp-errors-non-fatal`

### [ACCEPTED] ask-mcp-iteration-limit-is-five
When `mcp_servers` is non-empty, `ask()` allows up to 5 tool-call iterations before forcing a final LLM response (6 total invocations), compared to the lower limit without MCP servers.
- Source: entries/2026/05/11/tests-test_ask_mcp.md
- Rejected: duplicate: identical claim already exists as `ask-mcp-iteration-limit-is-five`

### [ACCEPTED] build-tools-section-always-includes-search-beliefs
`_build_tools_section` always includes the built-in `search_beliefs` tool in its output regardless of whether MCP bridges are provided — it is the baseline tool present in every ask prompt.
- Source: entries/2026/05/11/tests-test_ask_mcp.md
- Rejected: duplicate: identical claim already exists as `build-tools-section-always-includes-search-beliefs`

### [ACCEPTED] ask-prompt-no-template-placeholders
`build_ask_prompt` never leaves `{{` or `}}` template markers in the generated prompt string — all placeholders are resolved before output.
- Source: entries/2026/05/11/tests-test_ask_mcp.md
- Rejected: duplicate: identical claim already exists as `ask-prompt-no-template-placeholders`

### [ACCEPTED] extract-tool-call-schema-agnostic
Covered by existing `extract-tool-call-returns-first-match` and `ask-uses-text-based-tool-protocol` which together establish that tool call parsing is generic.
- Source: entries/2026/05/11/tests-test_ask_mcp.md

---

### entries/2026/05/11/tests-test_cli.md

### [ACCEPTED] cli-exit-code-zero-success-one-error
All CLI subcommands use exit code 0 for success and exit code 1 for every error condition (missing nodes, access denied, duplicate IDs, file not found); no other exit codes appear in the test suite.
- Source: entries/2026/05/11/tests-test_cli.md
- Rejected: duplicate: same claim as existing `cli-exit-code-contract-is-binary`

### [ACCEPTED] cli-tests-use-blackbox-main-patching
Covered by existing `cli-tests-are-black-box-integration`.
- Source: entries/2026/05/11/tests-test_cli.md

### [ACCEPTED] cli-tests-are-self-contained
Covered by existing `each-cli-test-creates-isolated-db`.
- Source: entries/2026/05/11/tests-test_cli.md

### [ACCEPTED] cli-errors-go-to-stderr
Covered by existing `cli-is-deterministic-and-stream-correct` which addresses stdout/stderr routing.
- Source: entries/2026/05/11/tests-test_cli.md

### [ACCEPTED] what-if-is-readonly
Covered by existing `pg-api-what-if-never-mutates`.
- Source: entries/2026/05/11/tests-test_cli.md

---

### entries/2026/05/11/tests-test_cluster.md

### [ACCEPTED] auto-k-floor-is-two
`_auto_k` never returns fewer than 2 clusters regardless of input size (using the `n // 5` heuristic), unless clamped by `n` itself when there are fewer beliefs than clusters.
- Source: entries/2026/05/11/tests-test_cluster.md
- Rejected: duplicate: floor of 2 already captured in existing `cluster-auto-k-heuristic` ("clamped between 2 and...")

### [ACCEPTED] list-clusters-partitions-all-beliefs
`list_clusters` assigns every input belief to exactly one cluster with no drops or duplicates — the union of all cluster members equals the input set.
- Source: entries/2026/05/11/tests-test_cluster.md

### [ACCEPTED] cluster-deps-are-optional-extras
Covered by existing `cluster-deps-are-optional` and `cluster-deps-optional-with-graceful-skip`.
- Source: entries/2026/05/11/tests-test_cluster.md

### [ACCEPTED] cluster-cache-is-incremental
Covered by existing `cluster-cache-no-recompute`.
- Source: entries/2026/05/11/tests-test_cluster.md

---

### entries/2026/05/11/tests-test_contradictions.md

### [ACCEPTED] contradiction-detection-filters-out-nodes
`detect_contradictions` and `detect_contradictions_semantic` both exclude OUT-truth-value nodes before sending any prompt to the LLM, even if those node IDs are explicitly requested in the `belief_ids` parameter.
- Source: entries/2026/05/11/tests-test_contradictions.md
- Rejected: duplicate: same claim as existing `contradiction-in-only-filter` and `contradictions-only-checks-in-beliefs`

### [ACCEPTED] contradiction-plan-round-trips-apply-entries
Writing a contradiction plan with `write_contradiction_plan` and parsing it back with `parse_contradiction_plan` preserves all `[APPLY]`-tagged NOGOOD entries with their IDs and claims, while discarding `[SKIP]`-tagged entries — enabling a human review workflow.
- Source: entries/2026/05/11/tests-test_contradictions.md
- Rejected: duplicate: identical claim already exists as `contradiction-plan-round-trips-apply-entries`

### [ACCEPTED] belief-text-truncated-at-200-chars
`format_beliefs_for_contradiction_check` truncates any belief text longer than 200 characters, appending `...` as a suffix, to keep LLM prompts bounded.
- Source: entries/2026/05/11/tests-test_contradictions.md
- Rejected: duplicate: identical claim already exists as `belief-text-truncated-at-200-chars` and `contradictions-belief-text-truncated-at-200-chars`

### [ACCEPTED] parse-contradiction-requires-two-valid-claims
Covered by existing `contradictions-min-two-claims-per-nogood`.
- Source: entries/2026/05/11/tests-test_contradictions.md

### [ACCEPTED] batch-llm-failure-returns-empty-not-raises
Covered by existing `batch-fault-isolation-is-universal-across-llm-operations` and `llm-fault-tolerance-is-multi-granular`.
- Source: entries/2026/05/11/tests-test_contradictions.md


### [ACCEPTED] validate-proposals-rejects-jaccard-similar-to-out
`validate_proposals` skips any proposal whose tokenized ID has >= 0.5 Jaccard similarity to an existing OUT belief, returning "similar to retracted" in the skip reason — preventing re-derivation of retracted beliefs under variant names
- Source: entries/2026/05/11/tests-test_derive.md

### [ACCEPTED] jaccard-tokenizer-splits-on-hyphens-and-colons
`_tokenize_id` splits belief IDs on hyphens and colons into token sets for Jaccard similarity comparison, so `foo-bar-baz` and `foo-bar-qux` share 2/4 tokens (0.5 similarity)
- Source: entries/2026/05/11/tests-test_derive.md

### [ACCEPTED] validate-proposals-partitions-not-raises
`validate_proposals` never raises exceptions — it partitions proposals into `(valid, skipped)` tuples where each skip carries a human-readable reason string
- Source: entries/2026/05/11/tests-test_derive.md
- Rejected: duplicate: same claim as existing `derive-fail-soft-validation` which already states validate_proposals filters into skipped list rather than raising

### [ACCEPTED] build-prompt-validates-custom-templates
`build_prompt` raises `ValueError("unknown placeholder")` for unrecognized `{fields}` and `ValueError("malformed braces")` for unclosed braces in custom prompt templates
- Source: entries/2026/05/11/tests-test_derive.md
- Rejected: duplicate: identical claim already exists as `build-prompt-validates-custom-templates`

### [ACCEPTED] apply-dedup-plan-collects-errors-not-raises
`apply_dedup_plan` collects errors into `result["errors"]` rather than raising, allowing partial application — one missing node does not block processing of the remaining dedup plan
- Source: entries/2026/05/11/tests-test_derive.md
- Rejected: duplicate: identical claim already exists as `apply-dedup-plan-collects-errors-not-raises`

### [ACCEPTED] pg-tests-skip-without-postgres-connection
All tests in `test_pg.py` carry the `pytest.mark.pg` marker and `skip_no_pg` conditional skip — they are silently skipped when no PostgreSQL connection is available
- Source: entries/2026/05/11/tests-test_pg.md
- Rejected: trivial: standard pytest conditional skip via marker — obvious from the decorator itself

### [ACCEPTED] pg-dispatch-requires-project-id
When `pg_conninfo` is provided (via CLI `--pg` flag or `REASONS_PG_CONNINFO` env var) without a corresponding `project_id`, the system calls `sys.exit()` rather than proceeding with an incomplete configuration
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPTED] cli-flags-override-env-vars
`_backend_kwargs` gives precedence to CLI `--pg`/`--project-id` flags over `REASONS_PG_CONNINFO`/`REASONS_PROJECT_ID` environment variables when both are present
- Source: entries/2026/05/11/tests-test_pg_dispatch.md
- Rejected: duplicate: identical claim already exists as `cli-flags-override-env-vars`

### [ACCEPTED] import-functions-parse-before-dispatch
API import functions (`import_json`, `import_beliefs`, `import_agent`) read and parse files into structured data in the API layer before delegating to `PgApi` — PgApi never receives raw file paths
- Source: entries/2026/05/11/tests-test_pg_dispatch.md

### [ACCEPTED] all-api-functions-support-pg-dispatch
Every public API function (`export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`) accepts `pg_conninfo`/`project_id` kwargs and routes to the corresponding `PgApi` method
- Source: entries/2026/05/11/tests-test_pg_dispatch.md
- Rejected: duplicate: identical claim already exists as `all-api-functions-support-pg-dispatch`

### [ACCEPTED] require-sqlite-blocks-pg-for-guarded-commands
`_require_sqlite` enforces that certain subcommands (e.g., `hash-sources`) cannot run against a Postgres backend, checking both CLI flags and environment variables and calling `sys.exit()` if PG is detected
- Source: entries/2026/05/11/tests-test_pg_dispatch.md
- Rejected: duplicate: core invariant (certain commands guarded from PG by `_require_sqlite`) already captured in `cli-sqlite-only-commands-exist`

### [ACCEPTED] remove-justification-enforces-minimum-count
`Network.remove_justification` refuses to remove the last justification from a derived node — a derived node must retain at least one justification, enforced with `ValueError("only one justification")`
- Source: entries/2026/05/11/tests-test_remove_justification.md



---

**Generated:** 2026-11-01 (batch 1/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] assert-flips-truth-without-removing-justifications
The `assert` command only flips a node's truth value to IN without removing or invalidating existing SL justifications; if a justification is structurally invalid (e.g., from import-beliefs contextual dependencies), the node will revert to OUT on next recompute, and `convert-to-premise` is the required workaround.
- Source: summaries/2026/11/01/.claude-skills-reasons-SKILL.md

### [ACCEPTED] agent-execution-bounded-to-reasons-cli
The Claude Code skill's `allowed-tools` field restricts the agent to `Bash(reasons *)`, `Bash(cd * && uv run reasons *)`, `Bash(uvx *reasons*)`, Read, Grep, and Glob only — no arbitrary shell commands, Python interpreter, sqlite3, or file writes beyond what `reasons` itself performs.
- Source: summaries/2026/11/01/.claude-skills-reasons-SKILL.md

### [ACCEPTED] execution-fallback-chain-three-strategies
CLI execution follows a prescribed three-step fallback chain: try the installed `reasons` binary → `cd <repo> && uv run reasons …` → `uvx --from ftl-reasons reasons …`, with the agent trying each in order until one succeeds.
- Source: summaries/2026/11/01/.claude-skills-reasons-SKILL.md

### [ACCEPTED] ci-gates-main-branch-only
The CI workflow triggers exclusively on push to `main` or pull_request targeting `main`; pushes to or PRs targeting any other branch produce no workflow run, meaning feature branches without a PR to main never execute tests.
- Source: summaries/2026/11/01/.github-workflows-tests.md

### [ACCEPTED] ci-matrix-jobs-are-independent
The four Python version jobs (3.10, 3.11, 3.12, 3.13) run as independent parallel jobs on separate GitHub Actions runners; a failure in one version's job does not cancel or affect the other three.
- Source: summaries/2026/11/01/.github-workflows-tests.md
- Rejected: trivial: independent matrix job execution is a GitHub Actions platform default, not a codebase-specific invariant

### [ACCEPTED] retracted-flag-is-idempotent-audit-marker
The `_retracted` flag is set even when a node is already OUT, making it an idempotent audit marker rather than a state transition; code that checks this flag to decide "should I retract?" will miss already-OUT nodes, while code checking it for logging/audit purposes will correctly catch all retractions.
- Source: summaries/2026/11/01/CHANGELOG.md

### [ACCEPTED] recompute-all-bounded-iteration
`recompute_all` uses a bounded iteration cap rather than an unbounded fixpoint to guarantee termination on cyclic graphs; any change to propagation logic must preserve or raise this cap to avoid infinite loops.
- Source: summaries/2026/11/01/CHANGELOG.md

### [ACCEPTED] reasons-canonical-reasons-lib-shim
`reasons/` is the canonical implementation package; `reasons_lib/` is a backwards-compat shim that re-exports from `reasons/`; all code changes must target `reasons/`, never `reasons_lib/`, and adding a module to `reasons/` without mirroring in `reasons_lib/` silently breaks the shim.
- Source: summaries/2026/11/01/CLAUDE.md

### [ACCEPTED] export-silently-excludes-restricted-nogoods
Export operations silently exclude nogoods that reference access-restricted nodes rather than raising a PermissionError, preventing information leakage at the cost of potentially incomplete nogood sets in exported data — this is the deliberate contrast with single-node read commands that do raise.
- Source: summaries/2026/11/01/CHANGELOG.md


---

**Generated:** 2026-11-01 (batch 2/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] llm-extra-excludes-openai
The `llm` meta-extra in pyproject.toml deliberately excludes `openai`; users requiring OpenAI must install `[llm,openai]` separately.
- Source: summaries/2026/11/01/pyproject.md
- Rejected: trivial: immediately visible in pyproject.toml `[project.optional-dependencies]` section

### [ACCEPTED] setuptools-packages-list-is-explicit
The `[tool.setuptools]` packages list is an explicit `["reasons", "reasons_lib"]` rather than auto-discovery; any new top-level package directory must be manually appended or it will be omitted from the wheel.
- Source: summaries/2026/11/01/pyproject.md

### [ACCEPTED] two-console-entry-points-contract
Two console scripts are registered in the distribution: `reasons` → `reasons.cli:main` and `reasons-mcp` → `reasons.mcp_server:main`; both targets must define a callable `main()` or the script will `ImportError`/`ModuleNotFoundError` at first invocation despite successful install.
- Source: summaries/2026/11/01/pyproject.md
- Rejected: trivial: entry_points table in pyproject.toml is directly readable; the failure mode (ImportError) is standard Python behavior

### [ACCEPTED] python-3-10-syntax-floor
`requires-python = ">=3.10"` is a hard interpreter floor; code may rely on `match` statements and `X | Y` type-union syntax without compatibility guards.
- Source: summaries/2026/11/01/pyproject.md
- Rejected: trivial: `requires-python` is visible in pyproject.toml; syntax capability follows directly from the declared version

### [ACCEPTED] data-model-omits-post-init-validation
The `Node`, `Justification`, and `Nogood` dataclasses in `reasons/__init__.py` deliberately omit `__post_init__` validation to keep construction fast in the hot propagation loop; all validation is deferred to boundary layers (API, persistence).
- Source: summaries/2026/11/01/reasons-__init__.md

### [ACCEPTED] data-model-imports-only-stdlib
`reasons/__init__.py` imports exclusively from the standard library (`dataclasses`); no third-party packages are required, ensuring the core data model loads in minimal environments without network or database dependencies.
- Source: summaries/2026/11/01/reasons-__init__.md
- Rejected: trivial: imports at the top of `reasons/__init__.py` are immediately visible to any reader

### [ACCEPTED] with-network-write-atomicity-and-pubsub
The `_with_network` context manager in `api.py` saves the network, publishes pubsub events, and updates the in-memory cache only on a successful write-path exit; if an exception occurs, none of these side effects execute and the prior cache entry is preserved.
- Source: summaries/2026/11/01/reasons-api.md

### [ACCEPTED] what-if-simulations-use-deepcopy
The SQLite-path `what_if_retract`, `what_if_assert`, and `what_if_supersede` functions operate on a `copy.deepcopy(net)` and never write to disk, returning predicted truth-value effects without mutating the cached or persisted network.
- Source: summaries/2026/11/01/reasons-api.md
- Rejected: duplicate: core invariant (what-if reasoning without permanent mutation) already captured in `both-backends-support-safe-hypothetical-reasoning`

### [ACCEPTED] accept-proposal-drift-detection
`accept_proposal` re-validates the stored `snapshot_json` against the live network state; if `truth_value` or `text_hash` has drifted since the proposal was created, the proposal is marked `stale` and the change is not applied.
- Source: summaries/2026/11/01/reasons-api.md

### [ACCEPTED] source-types-closed-set-of-four
`add_node` validates `source_type` against a closed set of exactly four values: `"code"`, `"document"`, `"self-description"`, `"derived"`; any other string is rejected.
- Source: summaries/2026/11/01/reasons-api.md

### [ACCEPTED] ask-last-iteration-structurally-forces-answer
On the final iteration of the ask tool loop, the prompt switches to `FINAL_ASK_PROMPT` which omits the tool definitions section entirely, making it structurally impossible for the LLM to issue another tool call via the defined protocol.
- Source: summaries/2026/11/01/reasons-ask.md

### [ACCEPTED] dual-mode-requires-sources-db
`ask()` with `dual=True` raises `ValueError` immediately if `sources_db` is `None`, since the dual retrieval path (TMS + FTS-RAG merge) is meaningless without a source-chunk database.
- Source: summaries/2026/11/01/reasons-ask.md
- Rejected: duplicate: identical claim already exists as `ask-dual-requires-sources-db`

### [ACCEPTED] ask-unknown-tool-treats-as-final-answer
In the ask tool loop, if the LLM response is not valid JSON with a recognized `"tool"` key (unknown tool name or non-JSON text), it is treated as the final answer and returned immediately rather than triggering a loop iteration.
- Source: summaries/2026/11/01/reasons-ask.md

### [ACCEPTED] wiki-llm-failure-silent-fallback-to-deterministic
If LLM page generation fails in `build_wiki` (timeout, model error, bad response), the affected topic group silently falls back to the deterministic per-node markdown rendering; no exception propagates to the caller and the page is still produced.
- Source: summaries/2026/11/01/reasons-build_wiki.md

### [ACCEPTED] linkify-is-idempotent
`_linkify` in `build_wiki.py` skips node IDs already embedded in `[id](...)` Markdown link syntax, so running the function multiple times on the same content produces no duplicate or nested links.
- Source: summaries/2026/11/01/reasons-build_wiki.md

### [ACCEPTED] topic-assignment-uses-exact-token-match
`_assign_topics` splits node IDs on `[-._:]` delimiters and assigns a node to the first topic whose label appears as a whole token; partial or substring matches (e.g., topic `"ml"` vs. node `"machine-learning-models"`) never trigger assignment.
- Source: summaries/2026/11/01/reasons-build_wiki.md

### [ACCEPTED] build-wiki-is-dependency-leaf
`build_wiki.py` is imported only by `reasons/api.py` and test files; no other `reasons/*.py` runtime module depends on it, placing it at a leaf of the internal dependency graph.
- Source: summaries/2026/11/01/reasons-build_wiki.md
- Rejected: trivial: import/dependency graph structure is directly observable from the file's import statements and the importers' source

### [ACCEPTED] supporting-justification-index-bounds-checked
Both `_format_node` and `_format_beliefs_for_prompt` in `build_wiki.py` guard the `supporting_justification` index with a `0 <= si < len(justifications)` check before dereferencing; out-of-range values fall through to rendering all antecedents rather than raising an `IndexError`.
- Source: summaries/2026/11/01/reasons-build_wiki.md


---

**Generated:** 2026-11-01 (batch 3/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] git-fail-open-assumes-changed
All git-checking functions (`file_changed_since`, `file_lines_changed_since`) return `True` (assume file changed) on any subprocess error, non-zero exit code, or timeout, forcing fallback to full content-hash comparison rather than silently marking a node as fresh.
- Source: summaries/2026/11/01/reasons-check_stale.md

### [ACCEPTED] pinned-lines-requires-pinned-sha
The line-range staleness fast path in `check_stale` only activates when `pinned_sha` exists in node metadata; `pin_lines` auto-creates `pinned_sha` (and `verified_at`) if missing, establishing a strict dependency where `pinned_lines` implies `pinned_sha`.
- Source: summaries/2026/11/01/reasons-check_stale.md

### [ACCEPTED] truncated-hash-is-one-time-migration
A 16-char `source_hash` is treated as a legacy format only when it is a strict prefix of the current full SHA-256; a 16-char value that is not a prefix is reported as genuine `content_changed`, making the 16-char case a one-time migration path rather than a permanent dual-format.
- Source: summaries/2026/11/01/reasons-check_stale.md

### [ACCEPTED] resolve-source-path-four-strategy-order
`resolve_source_path` attempts exactly four location strategies in fixed order: agent-specific repo → `db_dir` → named `repos` dict entry → `~/git/<name>/` home fallback, returning `None` only if no candidate exists on disk.
- Source: summaries/2026/11/01/reasons-check_stale.md

### [ACCEPTED] git-operations-require-git-binary-on-path
Git-aware pinning (`get_file_commit_sha`, `file_changed_since`, `file_lines_changed_since`) shells out via `subprocess` to `git log` and `git diff` rather than using a git library, making a working `git` binary on `PATH` a runtime requirement for the fast path.
- Source: summaries/2026/11/01/reasons-check_stale.md

### [ACCEPTED] project-id-is-deprecated-alias-for-domain-id
The `--project-id` CLI flag is a deprecated alias silently accepted by `_backend_kwargs`, which checks `domain_id` first and falls back to `project_id`; new code should use `--domain-id`.
- Source: summaries/2026/11/01/reasons-cli-main.md

### [ACCEPTED] research-alias-maps-to-cmd-repair
The `research` subcommand is the only alias in the CLI dispatch table, mapping to the same `cmd_repair` handler with identical argument definitions, not a separate implementation.
- Source: summaries/2026/11/01/reasons-cli-main.md
- Rejected: trivial: the alias mapping is directly visible in the CLI commands dispatch dict and argparse subparser definitions

### [ACCEPTED] backend-kwargs-reads-env-var-fallbacks
`_backend_kwargs` reads `REASONS_PG_CONNINFO` and `REASONS_DOMAIN_ID` environment variables as fallbacks when the corresponding `--pg` / `--domain-id` flags are not provided, enabling shell-level configuration without repeating flags on every invocation.
- Source: summaries/2026/11/01/reasons-cli-main.md
- Rejected: duplicate: existing belief `cli-flags-override-env-vars` already documents that `_backend_kwargs` falls back to `REASONS_PG_CONNINFO`/`REASONS_DOMAIN_ID` env vars when flags are absent

### [ACCEPTED] llm-cost-summary-printed-to-stderr
Every LLM-invoking CLI handler follows a consistent three-step cost-tracking pattern: `reset_cost_tracker()` before the API call, then `format_cost_summary()` printed to `sys.stderr` (not stdout) after, keeping cost reporting separated from result output.
- Source: summaries/2026/11/01/reasons-cli.md

### [ACCEPTED] intra-cluster-focus-is-round-robin
`cluster_beliefs_intra` concentrates the entire budget on a single cluster selected by `sorted_clusters[round_num % k]` (largest-first), meaning successive derive rounds cycle through clusters in deterministic round-robin order rather than always focusing on the largest.
- Source: summaries/2026/11/01/reasons-cluster.md

### [ACCEPTED] cluster-early-return-skips-ml-when-under-budget
All three public cluster functions (`cluster_beliefs`, `cluster_beliefs_intra`, `list_clusters`) check `len(beliefs) <= budget` (or `<= 3` for listing) and return immediately with a single-group result, avoiding loading the SentenceTransformer model and running KMeans entirely.
- Source: summaries/2026/11/01/reasons-cluster.md
- Rejected: duplicate: existing belief `cluster-skips-ml-when-under-budget` already captures the same early-return-when-beliefs-≤-budget behavior

### [ACCEPTED] cluster-cache-is-not-thread-safe
`ClusterCache._cache` is a plain Python dict with no locking; concurrent `embed()` calls on the same cache instance may double-encode or observe inconsistent intermediate state, relying on the derive pipeline being single-threaded.
- Source: summaries/2026/11/01/reasons-cluster.md

### [ACCEPTED] compact-in-nodes-ordered-by-dependent-count
Within the IN section of `compact()` output, nodes are sorted by `len(node.dependents)` descending so the most structurally important (most-referenced) beliefs appear first and are least likely to be cut by the budget.
- Source: summaries/2026/11/01/reasons-compact.md
- Rejected: duplicate: same claim as existing `compact-in-nodes-ordered-by-dependents`

### [ACCEPTED] compact-truncation-is-80-chars
When `truncate=True` (the default), node text longer than 80 characters is cut to 77 characters plus `"..."`; when `truncate=False`, text is emitted in full and may exceed the budget, shifting all enforcement to the line-level `_over_budget` gate.
- Source: summaries/2026/11/01/reasons-compact.md


---

**Generated:** 2026-11-01 (batch 4/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] contradiction-detector-checks-in-beliefs-only
The contradiction detector in `reasons/contradictions.py` filters the input to only beliefs with `truth_value == "IN"` before any LLM call; both `detect_contradictions` and `detect_contradictions_semantic` enforce this at entry.
- Source: summaries/2026/11/01/reasons-contradictions.md
- Rejected: duplicate: same claim as existing `contradictions-only-checks-in-beliefs`

### [ACCEPTED] contradiction-parser-rejects-hallucinated-ids
`parse_contradiction_response` threads a `valid_ids` set through the regex parser to discard any claim ID not present in the input nodes, and drops any nogood that has fewer than 2 surviving claims after filtering.
- Source: summaries/2026/11/01/reasons-contradictions.md
- Rejected: duplicate: core behavior already captured by `contradiction-min-two-claims` and `contradictions-min-two-claims-per-nogood`

### [ACCEPTED] contradiction-batch-failure-is-silent-skip
A per-batch LLM failure (timeout, API error, network blip) in contradiction detection is caught by a bare `except Exception`, logged to stderr, and the loop continues to the next batch — there is no retry logic and the function returns partial results.
- Source: summaries/2026/11/01/reasons-contradictions.md

### [ACCEPTED] contradiction-strategies-share-core-differ-in-grouping
`detect_contradictions` and `detect_contradictions_semantic` share the same `CONTRADICTION_PROMPT`, `format_beliefs_for_contradiction_check`, `parse_contradiction_response`, and `_flush_results`; they differ only in how beliefs are grouped into batches (random shuffle vs. semantic clusters from sentence-transformers).
- Source: summaries/2026/11/01/reasons-contradictions.md

### [ACCEPTED] contradiction-lazy-imports-break-circularity
`reasons/contradictions.py` imports `write_contradiction_plan` (from `.api`) and `list_clusters` (from `.cluster`) inside function bodies rather than at module top-level, preventing a `contradictions → api → contradictions` circular import and deferring sentence-transformers loading.
- Source: summaries/2026/11/01/reasons-contradictions.md

### [ACCEPTED] derive-delegates-llm-invocation-to-caller
`reasons/derive.py` owns prompt construction, response parsing, validation, and persistence but does NOT perform the LLM call itself — the `invoke_model` call is made by `api.py` or `cli.py`, which import and orchestrate the derive pipeline stages.
- Source: summaries/2026/11/01/reasons-derive.md

### [ACCEPTED] derive-parser-supports-legacy-format-fallback
`parse_proposals` in `reasons/derive.py` tries the v0.10+ regex pattern (`### DERIVE <id>`) first, then falls back to the v0.9 regex (`### DERIVE: \`belief-id\``), ensuring backward compatibility with previously-generated or user-edited files.
- Source: summaries/2026/11/01/reasons-derive.md

### [ACCEPTED] derive-min-antecedents-is-soft-constraint
The requirement that derived beliefs use ≥ 2 antecedents is enforced only through the prompt instruction to the LLM; `validate_proposals` does not check antecedent count, so a single-antecedent proposal will pass code-level validation.
- Source: summaries/2026/11/01/reasons-derive.md

### [ACCEPTED] derive-apply-tags-source-type-as-derived
`apply_proposals` in `reasons/derive.py` calls `api.add_node()` with `source_type="derived"` for every accepted proposal, marking the provenance of the belief so downstream tooling can distinguish LLM-derived beliefs from observations or imports.
- Source: summaries/2026/11/01/reasons-derive.md

### [ACCEPTED] derive-depth-cycle-guard-sets-memo-before-recursion
`_get_depth` in `reasons/derive.py` sets `memo[node_id] = 0` *before* recursing into antecedents, so a cyclic justification graph resolves the back-edge to depth 0 rather than causing infinite recursion.
- Source: summaries/2026/11/01/reasons-derive.md

### [ACCEPTED] export-card-partition-is-exhaustive
In `export_card`, the node partition is exhaustive: `in_count + out_count == total` (every node is either `"IN"` or not) and `premises + derived == total` (a node is a premise iff it has zero justifications), with no third state.
- Source: summaries/2026/11/01/reasons-export_card.md

### [ACCEPTED] export-card-depth-uses-supporting-justification-only
When `supporting_justification` is a valid index, `_node_depth` in `export_card.py` traverses only that single justification's antecedents (the "live" proof path), ignoring sibling justifications; an out-of-range or `None` index falls back to considering all justifications.
- Source: summaries/2026/11/01/reasons-export_card.md

### [ACCEPTED] export-markdown-sort-in-before-out
`export_markdown` sorts nodes by the composite key `(truth_value != "IN", id)`, guaranteeing all `IN` nodes appear before all `OUT`/`STALE` nodes with alphabetical tiebreaking — producing deterministic, diff-friendly output.
- Source: summaries/2026/11/01/reasons-export_markdown.md

### [ACCEPTED] export-markdown-deps-emit-dangling-refs-silently
The `all_deps` collection in `export_markdown` does not guard against antecedent IDs that reference non-existent nodes; a dangling antecedent is emitted as a plain string in the "Depends on" line without crashing, unlike the defeater lookup which uses a `nodes.get()` guard.
- Source: summaries/2026/11/01/reasons-export_markdown.md

### [ACCEPTED] export-markdown-type-inference-fallback
When `node.metadata["beliefs_type"]` is absent or empty, `export_markdown` infers the type as `DERIVED` if the node has at least one justification, or `OBSERVATION` otherwise — ensuring the output always conforms to the beliefs.md schema type token requirement.
- Source: summaries/2026/11/01/reasons-export_markdown.md

### [ACCEPTED] hf-token-resolution-chain
Token resolution in `reasons/hf.py` follows a strict priority chain: explicit argument → `HF_TOKEN` environment variable → `~/.cache/huggingface/token` file → `None`, mirroring `huggingface-cli` credential behavior so existing logged-in users work without passing `--token`.
- Source: summaries/2026/11/01/reasons-hf.md

### [ACCEPTED] hf-download-hardcoded-path
`download_network` in `reasons/hf.py` always fetches `network.json` from the `main` branch of the resolved repo (URL: `{org}/{name}/resolve/main/network.json`); there is no parameter to select a different branch or filename.
- Source: summaries/2026/11/01/reasons-hf.md

### [ACCEPTED] hf-write-ops-require-caller-provided-token
`create_repo` and `upload_file` in `reasons/hf.py` do NOT call `_resolve_token` internally — the caller must pass a non-`None` token — while `download_network` tolerates `None` (for public repos), creating an asymmetry between read and write paths.
- Source: summaries/2026/11/01/reasons-hf.md

### [ACCEPTED] hf-repo-id-normalization-with-env-override
`resolve_repo_id` in `reasons/hf.py` normalizes bare names by prepending `DEFAULT_HF_ORG` (`"EEM-Hub"`), which is overridable via the `REASONS_HF_ORG` environment variable; explicit `org/name` and full-URL forms pass through unchanged.
- Source: summaries/2026/11/01/reasons-hf.md


---

**Generated:** 2026-11-01 (batch 5/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] import-agent-normalization-diverges-on-out-justifications
`_normalize_markdown` drops all justifications for OUT nodes (sets to empty list), while `_normalize_json` preserves the full justification structure including outlists — meaning the shared `_import_claims`/`_sync_claims` logic receives structurally different data depending on source format.
- Source: summaries/2026/11/01/reasons-import_agent.md

### [ACCEPTED] build-justifications-falls-back-to-bare-sl
When a claim has zero `raw_justifications`, `_build_justifications` produces a single `Justification(type="SL", antecedents=[], outlist=[inactive_id])` as a fallback, ensuring even premise nodes are wired into the kill-switch cascade.
- Source: summaries/2026/11/01/reasons-import_agent.md

### [ACCEPTED] import-agent-silently-filters-invalid-references
`reasons/import_agent.py` raises no exceptions for normal operational issues: missing outlist/antecedent IDs are silently filtered during normalization, nogoods with <2 valid nodes are skipped, and topo-sort cycles are force-appended rather than raised.
- Source: summaries/2026/11/01/reasons-import_agent.md

### [ACCEPTED] strip-frontmatter-avoids-yaml-library
`strip_frontmatter` in `import_beliefs.py` deliberately avoids a YAML dependency, handling only flat `key: value` scalar pairs; nested structures produce silently corrupted values rather than raising an error.
- Source: summaries/2026/11/01/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-raises-no-exceptions
`import_into_network` and all its helpers (`parse_beliefs`, `parse_nogoods`, `parse_repos`, topo-sort) never raise in the normal code path; the returned summary dict (`claims_skipped`, `nogoods_imported`, etc.) is the sole error-reporting channel.
- Source: summaries/2026/11/01/reasons-import_beliefs.md

### [ACCEPTED] invoke-model-return-contract-is-str-or-raise
`invoke_model()` in `reasons/llm.py` always returns a `str` (the model's text response) or raises an exception; it never returns `None`, a dict, or a LangChain object — a contract relied upon by 15+ caller modules.
- Source: summaries/2026/11/01/reasons-llm.md

### [ACCEPTED] model-string-grammar-is-closed-set
Valid model strings for `invoke_model` are exactly: `claude`, `gemini`, `claude:<sub>`, `gemini:<sub>`, `ollama:<model>`, `api:<model>`, `vertex:<model>`, `openai:<model>`; any other form raises `ValueError` from `resolve_model_cmd`.
- Source: summaries/2026/11/01/reasons-llm.md

### [ACCEPTED] ollama-calls-skip-cost-recording
Ollama invocations bypass the JSON-parsing envelope and cost recording: the response is returned as raw text (after thinking-marker stripping) and `_record_cost` is never called, so `get_cost_summary()` will not include Ollama calls.
- Source: summaries/2026/11/01/reasons-llm.md

### [ACCEPTED] openai-integration-uses-responses-api
The `openai:` provider path in `llm.py` uses the OpenAI Responses API (`client.responses.create`) rather than Chat Completions, and records cost as `0.0` because the Responses API does not return USD cost in the expected shape.
- Source: summaries/2026/11/01/reasons-llm.md

### [ACCEPTED] provider-dispatch-is-string-prefix-not-plugin
Provider selection in `invoke_model` is a flat `startswith` cascade (`api:`, `vertex:`, `openai:`, then CLI) with no class hierarchy, plugin registry, or factory pattern — adding a provider requires one new `if` branch and one `_invoke_*` function.
- Source: summaries/2026/11/01/reasons-llm.md

### [ACCEPTED] mcp-bridge-is-one-shot-session
`McpBridge` has no retry logic, no reconnection, and no health-check; if the MCP server dies mid-session, the caller must create a new `McpBridge` instance and call `connect()` again.
- Source: summaries/2026/11/01/reasons-mcp_client.md

### [ACCEPTED] call-tool-returns-newline-joined-string
`McpBridge.call_tool()` flattens the MCP tool result's `content` list (which may contain multiple content blocks) into a single `\n`-joined string; non-text blocks are stringified via `str()`.
- Source: summaries/2026/11/01/reasons-mcp_client.md

### [ACCEPTED] mcp-bridge-command-is-shell-string
`McpBridge.__init__` expects a single shell command string (e.g. `"npx @mcp/server-fs /path"`) which is parsed via `shlex.split`; passing a list of arguments directly is not supported.
- Source: summaries/2026/11/01/reasons-mcp_client.md

### [ACCEPTED] mcp-server-stdio-purity-contract
The MCP server process must not emit anything to stdout other than JSON-RPC frames; any `print()` or stray output would corrupt the stdio transport and break the client, so `main()` and all tool handlers must avoid writing to stdout.
- Source: summaries/2026/11/01/reasons-mcp_server-main.md

### [ACCEPTED] db-resolution-priority-chain
The MCP server resolves the database path in strict priority order: `REASONS_DB` environment variable → upward filesystem walk from cwd looking for `reasons.db` → `api.DEFAULT_DB` constant fallback; `main()` accepts no parameters.
- Source: summaries/2026/11/01/reasons-mcp_server-main.md


---

**Generated:** 2026-11-01 (batch 6/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] mcp-tools-return-strings-only
All MCP tool handlers in `reasons/mcp_server.py` return `str` (JSON-dumped or markdown); no structured type crosses the MCP boundary, a requirement of the `FastMCP` tool contract.
- Source: summaries/2026/11/01/reasons-mcp_server.md

### [ACCEPTED] mcp-falsy-to-none-idempotency
The MCP layer converts falsy sentinels (`0` for limit, `""` for optional string params like status/namespace) to `None` via `x or None` before calling `reasons.api` functions, which treat `None` as "no filter / no limit."
- Source: summaries/2026/11/01/reasons-mcp_server.md

### [ACCEPTED] mcp-error-as-data-not-control-flow
Exceptions in MCP tool handlers are caught and serialized to `{"error": "..."}` JSON strings so the LLM receives a parseable recovery message; the MCP connection never dies from a tool exception.
- Source: summaries/2026/11/01/reasons-mcp_server.md

### [ACCEPTED] mcp-server-is-zero-logic-facade
`reasons/mcp_server.py` contains zero domain logic; every tool body is a one-or-two-line delegation into `reasons/api.*` with a try/except wrapper, and the module owns only wire-protocol serialization and DB path discovery.
- Source: summaries/2026/11/01/reasons-mcp_server.md
- Rejected: trivial: "every tool body is a one-or-two-line delegation into api" is directly observable from the file's function bodies and imports; no non-obvious behavioral contract beyond "it calls api functions"

### [ACCEPTED] mcp-db-resolution-precedence
Database path resolution follows a strict order: `REASONS_DB` env var → walk up from `cwd` looking for `reasons.db` → `api.DEFAULT_DB`, with the result cached in a module-level `_db` singleton resolved at most once per process.
- Source: summaries/2026/11/01/reasons-mcp_server.md

### [ACCEPTED] mcp-read-only-tools-lack-error-wrapping
The read-only tools (`compact`, `search`, `list_beliefs`, `status`, `list_gated`, `export_markdown`, `topics`) have no try/except wrapper, so any `api` exception propagates as a raw MCP protocol-level error rather than a structured `{"error": "..."}` JSON response.
- Source: summaries/2026/11/01/reasons-mcp_server.md

### [ACCEPTED] merkle-hash-follows-single-justification-path
The Merkle hash for a node incorporates only its `supporting_justification` and that justification's antecedents recursively; other justifications on the same node do not affect the computed hash, making the integrity guarantee per-path rather than per-node.
- Source: summaries/2026/11/01/reasons-merkle.md

### [ACCEPTED] merkle-verification-never-raises
`verify_node`, `verify_justification`, and `verify_all` never raise exceptions; they return `None` (clean) or a structured finding dict with a `"type"` discriminator (`"text_mutation"` or `"chain_mutation"`), making all integrity issues data rather than control flow.
- Source: summaries/2026/11/01/reasons-merkle.md

### [ACCEPTED] merkle-fresh-computation-is-non-circular
The `_fresh_merkle_hash` and `_fresh_content_hash` functions compute hashes exclusively from `node.text` and network topology; they never read `node.text_hash` or `justification.content_hash` from the network, ensuring verification compares a freshly-derived value against the stored one.
- Source: summaries/2026/11/01/reasons-merkle.md

### [ACCEPTED] merkle-cycle-produces-empty-string-hash
A cyclic justification chain does not raise or loop infinitely; `_fresh_merkle_hash` detects the cycle via the `visiting` set and returns `sha256("")` (hash of empty string), silently weakening the integrity guarantee without crashing.
- Source: summaries/2026/11/01/reasons-merkle.md

### [ACCEPTED] merkle-antecedents-sorted-for-determinism
In `_fresh_content_hash`, antecedent Merkle hashes are collected and `sorted(...)` before concatenation, guaranteeing the content hash is identical regardless of the order antecedents appear in the justification's list.
- Source: summaries/2026/11/01/reasons-merkle.md

### [ACCEPTED] merkle-module-is-stdlib-only
`reasons/merkle.py` imports only `hashlib` from the standard library with zero third-party or intra-package dependencies, enabling isolated unit testing and reuse without pulling in the full network stack.
- Source: summaries/2026/11/01/reasons-merkle.md
- Rejected: trivial: "imports only hashlib" is directly visible from the file's import statements; no behavioral or architectural invariant beyond what the import list already shows

### [ACCEPTED] build-meta-returns-fixed-six-key-dict
`build_meta()` always returns a flat dict with exactly six keys — `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, `generator` — no more, no fewer, and consumers can rely on this shape without defensive `.get()` calls.
- Source: summaries/2026/11/01/reasons-metadata.md

### [ACCEPTED] metadata-created-at-preservation-contract
If the caller passes a non-empty `created_at` string, it is preserved verbatim as the original creation timestamp; if empty or `None`, current UTC time is used. `updated_at` is always current UTC regardless of caller input, so the two diverge on re-export.
- Source: summaries/2026/11/01/reasons-metadata.md

### [ACCEPTED] metadata-generator-degrades-to-unknown
`_get_generator()` wraps `importlib.metadata.version("ftl-reasons")` in a `try/except Exception`; any failure (including `PackageNotFoundError`) results in `"ftl-reasons/unknown"` rather than raising, ensuring metadata stamping never blocks an export.
- Source: summaries/2026/11/01/reasons-metadata.md

### [ACCEPTED] metadata-module-is-formatter-not-validator
`build_meta()` contains no `raise` statements and performs no input validation; passing `node_count=-1`, `project_name=None`, or a non-ISO `created_at` string will not trigger an error — input correctness is the caller's responsibility.
- Source: summaries/2026/11/01/reasons-metadata.md

### [ACCEPTED] metadata-is-single-source-of-truth-for-envelope
Six production modules (`api.py`, `cli.py`, `export_card.py`, `export_markdown.py`, `pg.py`, `storage.py`) all call `build_meta()` rather than constructing the metadata dict inline, centralizing the schema to prevent field-name drift across export formats.
- Source: summaries/2026/11/01/reasons-metadata.md

### [ACCEPTED] mind-service-uses-stdlib-http-only
`reasons/mind_service.py` uses `urllib.request` for all HTTP operations; it has zero third-party HTTP dependencies (no `requests`, `httpx`, etc.), keeping it deployable in minimal environments.
- Source: summaries/2026/11/01/reasons-mind_service.md
- Rejected: trivial: "uses urllib.request" and "no requests/httpx" are immediately visible from the module's import block; no non-obvious behavioral invariant is stated

### [ACCEPTED] fetch-export-returns-unparsed-string
`fetch_export` returns the HTTP response body as a raw UTF-8 string without calling `json.loads`; the caller (`api.py`) is responsible for deserialization, keeping the function format-agnostic.
- Source: summaries/2026/11/01/reasons-mind_service.md

### [ACCEPTED] push-belief-omits-empty-optional-fields
`push_belief` conditionally excludes `sl` and `source` from the JSON request body when they are empty strings; the remote server must treat field absence as "not provided" rather than interpreting it as an empty value.
- Source: summaries/2026/11/01/reasons-mind_service.md

### [ACCEPTED] config-resolution-collects-all-missing-values
`_resolve_config` accumulates all missing configuration values (URL, agent_id, api_key) into a single `RuntimeError` message rather than failing on the first absent variable, so a misconfigured deployment reports every gap in one shot.
- Source: summaries/2026/11/01/reasons-mind_service.md

### [ACCEPTED] mind-service-404-handling-is-asymmetric
`fetch_export` special-cases HTTP 404 as a `RuntimeError` with a friendly message, but `push_belief` does not catch 404 — a 404 on the push endpoint propagates as a raw `HTTPError` to the caller.
- Source: summaries/2026/11/01/reasons-mind_service.md

### [ACCEPTED] recompute-all-is-bounded-by-node-count
`Network.recompute_all()` iterates at most `len(nodes) + 1` times to reach a fixpoint, providing a termination guarantee that prevents infinite recomputation loops even in pathological network states.
- Source: summaries/2026/11/01/reasons-network.md

### [ACCEPTED] defend-is-challenge-applied-to-challenge
The `defend` operation is implemented as `challenge()` applied to the challenge node itself, not as an independent mechanism; this makes dialectical defense a uniform instance of the same outlist-injection primitive.
- Source: summaries/2026/11/01/reasons-network.md

### [ACCEPTED] rebuild-dependents-is-single-canonical-path
`_rebuild_dependents()` is the single canonical method for reconstructing the reverse dependents index (wiping all `node.dependents` sets then re-deriving from all justifications' antecedents and outlists); the codebase explicitly states all other rebuild call sites should delegate to it.
- Source: summaries/2026/11/01/reasons-network.md

### [ACCEPTED] entrenchment-scoring-uses-weighted-heuristic
`find_culprits` scores premise candidates for retraction using additive weights: premise +100, has `source` +50, has `source_hash` +25, per-dependent +10, and `beliefs_type` ranges from 10 (NOTE) to 90 (AXIOM/WARNING), preferring retraction of speculative assumptions over evidence.
- Source: summaries/2026/11/01/reasons-network.md

### [ACCEPTED] node-without-justifications-is-premise-in-by-default
A node with no justifications is treated as a premise with `truth_value = "IN"` by default; it is not computed through `_compute_truth` and retains its initial value unless explicitly retracted or asserted.
- Source: summaries/2026/11/01/reasons-network.md
- Rejected: duplicate: existing `all-semantic-edge-cases-are-uniform` already states "absence of justifications yielding premise behavior," and "IN by default" is the definition of a premise in this system


---

**Generated:** 2026-11-01 (batch 7/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] pgapi-connection-ownership-dual-mode
`PgApi` constructor either owns a connection (opens from a connection string) or borrows an existing `psycopg.Connection`; `close()` semantics differ accordingly (closes owned, leaves borrowed intact)
- Source: summaries/2026/11/01/reasons-pg.md

### [ACCEPTED] pgapi-propagation-batch-fetches-avoiding-n1
`_propagate` drains the BFS queue in batches, issuing a single `SELECT … WHERE id = ANY(…)` per batch and a single `UPDATE` per changed node, avoiding N+1 queries on large cascades
- Source: summaries/2026/11/01/reasons-pg.md

### [ACCEPTED] pgapi-jsonb-defensive-parsing
JSONB column values returned by psycopg may be Python strings depending on the active `row_factory`; all read paths must guard with `isinstance(x, str): json.loads(x)` before using the value
- Source: summaries/2026/11/01/reasons-pg.md

### [ACCEPTED] pgapi-optional-psycopg-guard
The `psycopg` import is wrapped in `try/except ImportError` with `_require_psycopg()` raising a descriptive `ImportError`, making the PostgreSQL backend an optional dependency rather than a hard requirement
- Source: summaries/2026/11/01/reasons-pg.md

### [ACCEPTED] pgapi-import-uses-kahn-with-cycle-fallback
Both `import_json` and `import_beliefs` topologically sort nodes via a multi-pass ready-set loop (Kahn's algorithm) with a fallback insertion pass for cycles, ensuring all nodes are inserted in dependency order
- Source: summaries/2026/11/01/reasons-pg.md

### [ACCEPTED] propose-update-defaults-to-in-beliefs-only
`propose_updates` defaults `belief_ids` to all nodes with `truth_value == "IN"`; OUT beliefs are excluded from LLM auditing as already-retracted beliefs
- Source: summaries/2026/11/01/reasons-propose_update.md

### [ACCEPTED] propose-update-parser-coerces-closed-taxonomies
`parse_update_proposals` never raises on unknown `failure_mode`, `basis`, or `action` values; it silently substitutes `""`, `"prior-knowledge"`, and `"update"` respectively, ensuring structured output always conforms to the closed vocabularies
- Source: summaries/2026/11/01/reasons-propose_update.md

### [ACCEPTED] propose-update-batch-failure-preserves-partial-results
An exception in `invoke_model` for one batch is caught per-batch, logged to `stderr`, and remaining batches still execute; the function returns all successfully-parsed results from non-failed batches
- Source: summaries/2026/11/01/reasons-propose_update.md

### [ACCEPTED] propose-update-produces-review-artifact-not-mutation
The module's output contract is a markdown document containing ready-to-paste CLI commands (`reasons update`/`reasons retract`); it never applies changes to the network itself
- Source: summaries/2026/11/01/reasons-propose_update.md

### [ACCEPTED] pubsub-exposes-only-post-save-state
Subscribers observe only the final stable post-save snapshot; intermediate cascade states during fixpoint computation are structurally invisible because `publish()` is called only after the full transaction commits
- Source: summaries/2026/11/01/reasons-pubsub.md

### [ACCEPTED] pubsub-subscriber-exceptions-are-isolated
A raising subscriber does not prevent other subscribers from receiving the same event batch, nor does it propagate to the caller of `publish()`; each subscriber is wrapped in an independent `try/except`
- Source: summaries/2026/11/01/reasons-pubsub.md

### [ACCEPTED] pubsub-subscribe-allows-duplicates
Calling `subscribe(callback, db_path)` twice with the same callback registers it twice; `publish` will invoke it twice per event batch — there is no deduplication check
- Source: summaries/2026/11/01/reasons-pubsub.md

### [ACCEPTED] pubsub-insertion-indistinguishable-from-change
A new node (present in `after`, absent from `before`) produces a `ChangeEvent` with `old_truth_value=None` and `type="node_changed"`, making it indistinguishable from a value change by the `type` field alone
- Source: summaries/2026/11/01/reasons-pubsub.md

### [ACCEPTED] pubsub-is-synchronous-in-process
`publish()` blocks until all subscribers return; there is no queue, thread, or IPC mechanism — safe in single-threaded use but unsafe under concurrent calls without external synchronization
- Source: summaries/2026/11/01/reasons-pubsub.md

### [ACCEPTED] repair-search-link-requires-in-premises
Valid search-and-link targets must satisfy all four conditions: truth value is IN, the node has no justifications (is a premise), it is not the belief itself, and it is not already an antecedent
- Source: summaries/2026/11/01/reasons-repair.md

### [ACCEPTED] repair-patterns-are-closed-and-validated
`VALID_PATTERNS` is the 4-element set `{search_and_link, soften, abandon, research}`; `parse_triage_response` rejects any LLM-returned pattern outside this set, and an empty pattern causes the belief to be skipped with `status: "triage_failed"`
- Source: summaries/2026/11/01/reasons-repair.md

### [ACCEPTED] repair-lazy-imports-api-to-avoid-cycle
`from . import api` appears inside function bodies (not at module level) in `repair.py` to avoid circular imports, since `api.py` imports from `repair.py`
- Source: summaries/2026/11/01/reasons-repair.md
- Rejected: trivial: any developer can see the in-function import in repair.py and the api→repair import direction by reading both files' import sections

### [ACCEPTED] repair-matched-ids-validated-against-candidates
`parse_match_response` filters LLM-returned `matched_ids` through the actual candidate node set; any ID not in the candidate list is silently dropped, preventing hallucinated references from being wired into the network
- Source: summaries/2026/11/01/reasons-repair.md

### [ACCEPTED] repair-beliefs-aliased-as-research-beliefs
`repair_beliefs` is explicitly aliased as `research_beliefs` (line ~340), so both names reference the same function for backward-compatible import paths
- Source: summaries/2026/11/01/reasons-repair.md
- Rejected: trivial: a one-line alias assignment visible immediately in the source; no non-obvious behavioral constraint

### [ACCEPTED] repair-premises-parse-failure-degrades-to-retract
If the LLM response cannot be parsed as a valid JSON array, `_repair_one` returns `{"action": "retract"}` rather than raising, ensuring the TMS never retains an unverifiable premise
- Source: summaries/2026/11/01/reasons-repair_premises.md

### [ACCEPTED] repair-premises-parallel-ordering-uses-id-key
When `parallel > 0`, the returned result list is in thread-completion order (via `as_completed`), not input order; callers must join on the `"id"` field rather than positional index
- Source: summaries/2026/11/01/reasons-repair_premises.md

### [ACCEPTED] repair-premises-swalls-llm-exceptions
`_repair_one` catches all exceptions from `invoke_model` (timeout, network, API errors) and converts them to `{"action": "error"}` dicts; the caller never sees a raised exception from the LLM call itself
- Source: summaries/2026/11/01/reasons-repair_premises.md

### [ACCEPTED] repair-premises-parallel-uses-threadpool
Parallel execution in `repair_premises` uses `concurrent.futures.ThreadPoolExecutor` with `max_workers=parallel` (not asyncio), and `parallel=0` falls back to a sequential loop
- Source: summaries/2026/11/01/reasons-repair_premises.md

### [ACCEPTED] repair-premises-is-source-only
The `REPAIR_PREMISES_PROMPT` explicitly constrains the LLM to use only the provided source document text for rewrite decisions; external knowledge is forbidden, and a missing source is substituted with `"(source not available)"`
- Source: summaries/2026/11/01/reasons-repair_premises.md


---

**Generated:** 2026-11-01 (batch 8/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] report-belief-dual-mode-always-returns-markdown-str
`report_belief` dispatches on the `model` parameter (falsy → deterministic markdown builder, truthy → LLM synthesis) and always returns a `str` in both branches; the LLM path prepends the canonical `# Belief Report` heading because the prompt explicitly forbids the model from including a title.
- Source: summaries/2026/11/01/reasons-report_belief.md

### [ACCEPTED] report-truth-value-markers-are-binary
In all report renderers, the TMS truth-value marker is binary: `+` if and only if `truth_value == "IN"`, and `-` for every other value (OUT, UNKNOWN, retracted) with no distinct marker per non-IN state.
- Source: summaries/2026/11/01/reasons-report_belief.md

### [ACCEPTED] report-gated-is-pure-presentation-layer
`report_gated` does not gather any data from storage; all inputs (`gated_data`, `retracted_premises`, `blocker_details`, `in_count`, `out_count`) are collected upstream in `reasons/api.py` before the function is called, making this file a pure serialization/presentation concern.
- Source: summaries/2026/11/01/reasons-report_gated.md

### [ACCEPTED] report-gated-llm-failure-propagates-without-fallback
Unlike the review modules which isolate per-batch failures, `report_gated` has no try/except around the LLM call: if `invoke_model` raises (timeout, network, model-not-found), the exception propagates directly to the caller with no fallback to the deterministic structured path.
- Source: summaries/2026/11/01/reasons-report_gated.md

### [ACCEPTED] review-beliefs-explicit-ids-skip-truth-value-check
When `belief_ids` is explicitly provided to `review_beliefs`, the `truth_value == "IN"` filter is not enforced (only membership in `nodes` and non-empty justifications are checked), whereas the default auto-discovery path (belief_ids=None) requires both IN status and non-empty justifications.
- Source: summaries/2026/11/01/reasons-review.md

### [ACCEPTED] defeat-reason-types-are-closed-seven-value-set
`DEFEAT_REASON_TYPES` in `review.py` is a fixed list of exactly 7 strings that serves as both the LLM's allowed output vocabulary and the validation set in `classify_defeat_reason`; any downstream code matching on `defeat_reason_type` can exhaustively branch over these 7 values plus `""` for unclassified.
- Source: summaries/2026/11/01/reasons-review.md

### [ACCEPTED] review-justifications-scopes-to-multi-antecedent-sl-only
`review_justifications` only audits justifications where `type == "SL"` and `len(antecedents) >= min_antecedents` (default 2); single-antecedent SL justifications and all non-SL justification types are excluded from review as trivially unambiguous or out of scope.
- Source: summaries/2026/11/01/reasons-review_justifications.md

### [ACCEPTED] review-justifications-is-read-only-diagnosis
`review_justifications` explicitly never mutates the database or belief network; it produces a report of candidates for ALL/ANY/MIXED reclassification, with the actual conversion delegated to downstream modules (e.g., `repair.py` or `propose_update.py`).
- Source: summaries/2026/11/01/reasons-review_justifications.md

### [ACCEPTED] review-justifications-classification-is-trivocal
The justification mode classification vocabulary is exactly three values — `"ALL"`, `"ANY"`, `"MIXED"` — and the parser uppercases whatever the LLM returns, tolerating lowercase input while enforcing this closed set for downstream consumption.
- Source: summaries/2026/11/01/reasons-review_justifications.md

### [ACCEPTED] review-premises-groups-by-source-before-batching
Before splitting into LLM batches, `review_premises` regroups `premise_ids` by their node `source` field so premises drawn from the same document are contiguous within a batch, ensuring the LLM sees one source document rather than interleaving multiple documents per prompt.
- Source: summaries/2026/11/01/reasons-review_premises.md

### [ACCEPTED] review-premises-requires-explicit-premise-ids
Unlike `review_beliefs` and `review_justifications` which auto-discover qualifying nodes when the ID parameter is `None`, `review_premises` requires an explicit `premise_ids` list — there is no auto-discovery path, the caller must supply the IDs to audit.
- Source: summaries/2026/11/01/reasons-review_premises.md

### [ACCEPTED] review-premises-accurate-error-type-mutually-exclusive
The review output enforces mutual exclusion between `accurate` and `error_type`: if `accurate` is `true` then `error_type` must be `null`, and if `accurate` is `false` then `error_type` must be one of four specific values (`misread_source`, `overgeneralized`, `fabricated`, `unsupported`) — enforced by the prompt but not re-validated by the parser.
- Source: summaries/2026/11/01/reasons-review_premises.md

### [ACCEPTED] review-modules-vary-batch-size-by-content-weight
The three review modules use different `REVIEW_BATCH_SIZE` constants scaled to per-item content weight: `review.py` uses 20 (belief + justification chain), `review_justifications.py` uses 15 (belief + antecedent details), and `review_premises.py` uses 5 (premise + full source document text).
- Source: summaries/2026/11/01/reasons-review_premises.md
- Rejected: trivial: Specific constant values (20, 15, 5) are directly visible in source code; the "scaled to content weight" observation is a naming convention any developer can infer from the constants and their contexts.

### [ACCEPTED] reasons-and-reasons-lib-are-mirror-packages
The repository maintains parallel `reasons/` and `reasons_lib/` packages with mirrored module files (e.g., `report_belief.py`, `review.py`, `review_justifications.py`); `reasons/` is the canonical CLI-facing source and `reasons_lib/` is the library-distribution variant, requiring changes to be mirrored across both.
- Source: summaries/2026/11/01/reasons-review_justifications.md
- Rejected: speculative: The claim that changes "requir[e]… to be mirrored across both" is an editorial maintenance guideline, not an observable code fact; if the shim is a star-re-export, mirroring is unnecessary, making this assertion ungrounded.

### [ACCEPTED] review-justifications-imports-format-from-review
`review_justifications.py` imports `format_belief_for_review` from `reasons/review.py` as its belief serialization utility, creating a cross-module dependency within the review subsystem rather than duplicating the formatting logic.
- Source: summaries/2026/11/01/reasons-review_justifications.md
- Rejected: trivial: "Module X imports Y" is explicitly listed as a trivial example; the cross-module import is visible from the import statement.


---

**Generated:** 2026-11-01 (batch 9/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] storage-save-preserves-external-writes
`Storage.save()` snapshots `node_tags` and `node_sources` rows *before* clearing all tables, then re-inserts them *after* the network-derived rows; this preserves tags and sources added directly to the database by other modules (`add_tags()`, `add_source()`) that bypass the in-memory `Network` object — a regression here silently loses API-added metadata.
- Source: summaries/2026/11/01/reasons-storage.md

### [ACCEPTED] storage-save-exceptions-rollback-transaction
`Storage.save()` contains no explicit try/except; any `sqlite3.IntegrityError`, `OperationalError`, or disk-full condition propagates to the caller, and the `with self.conn:` context manager rolls back the entire transaction, leaving the database in its prior consistent state.
- Source: summaries/2026/11/01/reasons-storage.md

### [ACCEPTED] proposals-schema-not-owned-by-storage
`PROPOSALS_SCHEMA` DDL is defined as a module-level constant in `reasons/storage.py` but is NOT created by `Storage._init_schema()`; it is applied elsewhere (e.g., `propose_update.py` or the API layer), so the `proposals` table may not exist in a database opened via `Storage` alone.
- Source: summaries/2026/11/01/reasons-storage.md

### [ACCEPTED] storage-migrations-use-alter-table-not-framework
Schema evolution in `Storage._init_schema()` uses `PRAGMA table_info` to detect missing columns and issues `ALTER TABLE ADD COLUMN` inline, plus data backfill from `metadata_json` for pre-existing rows — no Alembic, no versioned migration files, no migration runner.
- Source: summaries/2026/11/01/reasons-storage.md

### [ACCEPTED] verify-beliefs-single-llm-call-no-batching
`verify_beliefs()` sends all beliefs in exactly one `invoke_model` call regardless of count; there is no internal batching, chunking, retry, or fallback model — the only safeguard against prompt overflow is the 30,000-character per-source truncation cap.
- Source: summaries/2026/11/01/reasons-verify.md

### [ACCEPTED] verify-read-source-resolves-relative-to-db
`read_source()` derives the resolution directory from the database file's parent path (`Path(db_path).parent`) and passes it as `db_dir` to `resolve_source_path`, so source paths are resolved relative to the database's location, not the current working directory.
- Source: summaries/2026/11/01/reasons-verify.md

### [ACCEPTED] verify-beliefs-never-reads-files
`verify_beliefs()` does not call `read_source()` or touch the filesystem; the caller (e.g., `api.py`) must resolve and read source files and pass the content as the second element of each `(belief_dict, source_content_or_None)` tuple.
- Source: summaries/2026/11/01/reasons-verify.md

### [ACCEPTED] verify-llm-call-exceptions-propagate
The "no-raise" guarantee in `reasons/verify.py` applies to file-reading and response-parsing paths only; exceptions from `invoke_model` (network timeout, auth error) propagate uncaught to the caller, so `api.py` must handle LLM-call failures.
- Source: summaries/2026/11/01/reasons-verify.md

### [ACCEPTED] verify-parse-drops-malformed-entries
`parse_verify_response()` silently discards individual JSON entries that lack a string `"verdict"` key (or where `verdict` is not a string) rather than failing the entire batch; only structurally valid entries appear in the returned dict, and verdicts are normalized to uppercase.
- Source: summaries/2026/11/01/reasons-verify.md

### [ACCEPTED] reasons-lib-is-pure-reexport-shim
`reasons_lib/__init__.py` contains only `from reasons import *` with zero own logic; every public name it exposes is the same object reference as in `reasons`, so any name added to `reasons` is automatically visible through `reasons_lib` without touching the shim, but removing a name from `reasons` silently breaks `reasons_lib` consumers.
- Source: summaries/2026/11/01/reasons_lib-__init__.md

### [ACCEPTED] reasons-lib-requires-reasons-package
Importing `reasons_lib` raises `ModuleNotFoundError` if the `reasons` package is not present in the Python path; there is no fallback, stub, or lazy-import mechanism — `reasons` is a hard import-time dependency of `reasons_lib`.
- Source: summaries/2026/11/01/reasons_lib-__init__.md

### [ACCEPTED] reasons-lib-contains-mirror-submodules
The `reasons_lib/` directory holds per-module shim files (`api.py`, `ask.py`, `cli.py`, `storage.py`, etc.) in addition to `__init__.py`; `from reasons_lib.api import X` requires `reasons_lib/api.py` to exist and perform its own star re-export — the package-level `__init__.py` alone does not make submodules accessible.
- Source: summaries/2026/11/01/reasons_lib-__init__.md
- Rejected: trivial: Directory contents are visible via `ls`; the Python packaging fact that `from pkg.sub import X` requires `pkg/sub.py` to exist is basic Python knowledge.

### [ACCEPTED] reasons-lib-is-stale-and-incomplete
The `reasons_lib/` package is missing at least two modules that exist in `reasons/` (`merkle.py`, `storage.py`), indicating the legacy package is not kept in sync with the canonical implementation and is slated for eventual removal.
- Source: summaries/2026/11/01/reasons_lib-api.md
- Rejected: ephemeral: "Missing at least two modules" is a point-in-time count that changes with any sync commit; "slated for eventual removal" is a prediction.

### [ACCEPTED] importers-still-use-legacy-path
Seven internal modules under `reasons/` (`ask.py`, `cli.py`, `contradictions.py`, `derive.py`, `mcp_server.py`, `pg.py`, `repair.py`) and 50+ test files still import via `reasons_lib.api`, making the shim load-bearing until every one of those imports is migrated to `reasons.api`.
- Source: summaries/2026/11/01/reasons_lib-api.md
- Rejected: ephemeral: Specific import counts (7 modules, 50+ tests) are a transitional snapshot that changes with every migration commit; not a durable invariant.

### [ACCEPTED] shim-binds-identical-function-objects
Because the star-import in every `reasons_lib` shim binds the same function/class objects (not copies or proxies), any monkey-patch applied to `reasons.api` (or any other canonical module) is immediately visible through the `reasons_lib` alias and vice versa — there is no encapsulation boundary between the two namespaces.
- Source: summaries/2026/11/01/reasons_lib-api.md

### [ACCEPTED] reasons-lib-dependency-is-one-directional
The dependency is strictly `reasons_lib → reasons` (never the reverse), so the shim introduces no circular-import risk; `reasons` modules importing `reasons_lib.api` is a live consideration for potential cycles but is not caused by the shim file itself.
- Source: summaries/2026/11/01/reasons_lib-ask.md
- Rejected: speculative: editorial claim about what is "permitted" and an "unenforced convention," not an observable code invariant; also contradicted by the internal-import evidence in the same batch

### [ACCEPTED] reasons-lib-safe-to-delete-atomically
Since every `reasons_lib` module is a pure star-re-export with zero own executable logic, mutable state, or conditional branching, the entire `reasons_lib/` directory can be deleted in a single atomic commit once all external and internal importers are migrated to `reasons`.
- Source: summaries/2026/11/01/reasons_lib-ask.md
- Rejected: speculative: "Can be deleted in a single atomic commit once all importers are migrated" is a recommendation and prediction about future state, not a grounded code fact.

### [ACCEPTED] reasons-lib-noqa-f403-is-ci-load-bearing
Without the `# noqa: F403` directive, linters enforcing F403 (star import) would flag every `reasons_lib` shim file in CI, blocking merges for a non-issue — the directive is a required part of the shim's operational contract, not a cosmetic annotation.
- Source: summaries/2026/11/01/reasons_lib-ask.md
- Rejected: speculative: claims about CI pipeline behavior and merge-blocking that aren't verifiable from the code itself


---

**Generated:** 2026-11-01 (batch 10/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] reasons-lib-is-uniform-star-import-shim
Every file in `reasons_lib/` consists solely of a docstring and a single `from reasons.<name> import *` statement; no shim file contains any original logic, classes, functions, or mutable state.
- Source: summaries/2026/11/01/reasons_lib-build_wiki.md
- Rejected: duplicate: identical claim to the first reasons-lib-is-uniform-star-import-shim proposal

### [ACCEPTED] star-import-guarantees-object-identity
For any public name `X`, `reasons_lib.<mod>.X is reasons.<mod>.X` evaluates to `True` (same object reference, not a copy), so monkey-patching a function in one namespace is immediately visible through the other.
- Source: summaries/2026/11/01/reasons_lib-build_wiki.md
- Rejected: trivial: standard Python language behavior for star imports, known to any developer

### [ACCEPTED] reasons-internal-modules-still-import-via-reasons-lib
At least three modules inside the canonical `reasons/` package (`api.py`, `pg.py`, `verify.py`) still import symbols through the deprecated `reasons_lib` namespace rather than their own package, making the shim a live production code path.
- Source: summaries/2026/11/01/reasons_lib-check_stale.md
- Rejected: trivial: immediately visible by reading the import statements in api.py, pg.py, verify.py

### [ACCEPTED] reasons-lib-is-strict-subset-of-reasons
`reasons_lib/` is missing several modules that exist in `reasons/` (including `mcp_client.py`, `mcp_server.py`, `hf.py`, `pubsub.py`), so the legacy package does not cover the full canonical API surface.
- Source: summaries/2026/11/01/reasons_lib-check_stale.md
- Rejected: ephemeral: specific enumeration of missing modules (mcp_client, mcp_server, hf, pubsub) is a point-in-time snapshot that changes as modules are added

### [ACCEPTED] no-deprecation-warning-emitted-on-shim-import
Importing any `reasons_lib.*` module does not raise `DeprecationWarning` or `PendingDeprecationWarning`; the sole deprecation signal is the docstring text directing users to the `reasons.*` path.
- Source: summaries/2026/11/01/reasons_lib-build_wiki.md
- Rejected: trivial: the shim file contains only a star import; absence of a warning mechanism is immediately visible from the single-line code

### [ACCEPTED] shim-deletion-blocked-by-test-and-internal-imports
Deleting the `reasons_lib/` package without first rewriting imports in ~20 test files and the internal `reasons/` modules (`api.py`, `pg.py`, `verify.py`, and others) will cause `ImportError` in those consumers.
- Source: summaries/2026/11/01/reasons_lib-cli.md
- Rejected: duplicate: core claim (internal reasons/ modules import via reasons_lib) already captured by reasons-internal-modules-still-import-via-reasons-lib

### [ACCEPTED] star-import-excludes-underscore-prefixed-names
Because all shims use `import *` rather than explicit name lists, any symbol in the canonical module that begins with `_` is not forwarded to `reasons_lib`, automatically preserving the private/public boundary without additional gating.
- Source: summaries/2026/11/01/reasons_lib-cli.md
- Rejected: trivial: standard Python `import *` semantics, known to any developer

### [ACCEPTED] reasons-lib-mirrors-reasons-one-to-one
The claim that "every module in reasons/ has a corresponding shim in reasons_lib/" is contradicted by explicit evidence that `reasons_lib/` is missing `mcp_client.py`, `mcp_server.py`, `hf.py`, `pubsub.py`, and others; the mirror is a strict subset, not isomorphic.
- Source: summaries/2026/11/01/reasons_lib-check_stale.md


---

**Generated:** 2026-11-01 (batch 11/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] reasons-lib-is-pure-reexport-shim
Every file in `reasons_lib/` defines zero original symbols; each module's sole substance is a single `from reasons.<module> import *` statement, making the entire package a transparent namespace alias with no logic, state, or error handling of its own.
- Source: summaries/2026/11/01/reasons_lib-contradictions.md

### [ACCEPTED] reasons-lib-mirrors-reasons-with-exceptions
The `reasons_lib/` directory is a file-for-file mirror of `reasons/`, except that `merkle.py` and `pg.py` exist only under `reasons/` with no `reasons_lib/` counterpart, indicating those modules were added after the shim namespace was frozen.
- Source: summaries/2026/11/01/reasons_lib-contradictions.md
- Rejected: ephemeral: specific module exceptions (merkle.py, pg.py) are a point-in-time directory listing snapshot

### [ACCEPTED] reasons-api-imports-via-reasons-lib
`reasons/api.py` (the canonical package's public API layer) imports modules such as `contradictions`, `derive`, `export_markdown`, and `hf` through the `reasons_lib` shim path rather than directly from `reasons`, creating an intra-project dependency that blocks deletion of the `reasons_lib` package.
- Source: summaries/2026/11/01/reasons_lib-derive.md
- Rejected: duplicate: same claim as reasons-internal-modules-still-import-via-reasons-lib

### [ACCEPTED] star-import-shim-provides-automatic-forward-compat
Because every `reasons_lib/` shim uses `import *` with no local `__all__`, any new public symbol added to a `reasons/` module automatically becomes accessible through the corresponding `reasons_lib/` shim without any code change to the shim file.
- Source: summaries/2026/11/01/reasons_lib-export_markdown.md
- Rejected: trivial: standard Python `import *` behavior, known to any developer

### [ACCEPTED] shim-identity-guarantee-is-object-identity
For every public name `N` in a `reasons` module, `reasons_lib.<module>.N is reasons.<module>.N` evaluates to `True` (object identity, not mere equality); the shim adds no wrapping, proxying, sub-classing, or transformation.
- Source: summaries/2026/11/01/reasons_lib-contradictions.md
- Rejected: duplicate: same claim as star-import-guarantees-object-identity

### [ACCEPTED] reasons-lib-shim-emits-no-deprecation-warning
Importing from any `reasons_lib` module produces no `DeprecationWarning`, no stderr message, and no other signal to consumers that the import path is slated for removal; the only documentation is a docstring string.
- Source: summaries/2026/11/01/reasons_lib-derive.md
- Rejected: duplicate: same claim as no-deprecation-warning-emitted-on-shim-import

### [ACCEPTED] shim-import-failure-is-immediate-and-unwrapped
If the target `reasons.<module>` fails to import (missing dependency, syntax error), the `reasons_lib` shim raises the exception immediately at module-load time with no `try/except`, no `__getattr__` fallback, no deferred-import guard, and no additional error context wrapping.
- Source: summaries/2026/11/01/reasons_lib-export_markdown.md
- Rejected: trivial: the shim file is a single star-import line; absence of try/except is immediately visible

### [ACCEPTED] shim-star-import-respects-target-all
If a `reasons/<module>.py` file defines an `__all__` list, the shim's star import will respect it (re-exporting only listed names), which can silently break `reasons_lib` consumers who previously relied on public symbols not included in `__all__`.
- Source: summaries/2026/11/01/reasons_lib-export_card.md
- Rejected: trivial: standard Python `import *` and `__all__` interaction, known to any developer


---

**Generated:** 2026-11-01 (batch 12/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] reasons-lib-is-uniform-star-import-shim
Every file in `reasons_lib/` is a single-line `from reasons.<module> import *` re-export with no local function, class, or constant definitions, making the package a pure namespace alias for `reasons/` with no independent logic.
- Source: summaries/2026/11/01/reasons_lib-import_agent.md
- Rejected: duplicate: identical claim to the first reasons-lib-is-uniform-star-import-shim proposal

### [ACCEPTED] reasons-internal-modules-still-route-through-reasons-lib
At least three modules inside the canonical `reasons/` package (`api.py`, `import_agent.py`, `pg.py`) still import from `reasons_lib.import_beliefs` rather than `reasons.import_beliefs` directly, making the shim load-bearing for internal code rather than only external consumers.
- Source: summaries/2026/11/01/reasons_lib-import_beliefs.md
- Rejected: duplicate: same claim as reasons-internal-modules-still-import-via-reasons-lib

### [ACCEPTED] reasons-lib-star-import-auto-propagates-new-exports
Because the shims use `import *` without an explicit symbol list or `__all__` override, any new public symbol added to `reasons.<module>` becomes immediately accessible through `reasons_lib.<module>` without editing the shim file, but a renamed or removed symbol silently breaks old callers with a bare `ImportError`.
- Source: summaries/2026/11/01/reasons_lib-mcp_server.md
- Rejected: trivial: standard Python `import *` behavior, known to any developer

### [ACCEPTED] reasons-lib-dependency-is-one-directional
`reasons_lib` imports from `reasons`, never the reverse; no `reasons` module is permitted to import from `reasons_lib`, preventing circular imports by construction (though this is an unenforced convention, not a static check).
- Source: summaries/2026/11/01/reasons_lib-mcp_server.md
- Rejected: speculative: editorial claim about what is "permitted" and an "unenforced convention," not an observable code invariant; also contradicted by the internal-import evidence in the same batch

### [ACCEPTED] reasons-lib-shim-preserves-object-identity
Objects accessed via `reasons_lib.<module>.X` are the exact same Python objects as `reasons.<module>.X` (`is`-identity holds); the shim performs a one-time namespace copy at import time with no runtime proxying, wrapping, or `__getattr__` indirection at call time.
- Source: summaries/2026/11/01/reasons_lib-llm.md
- Rejected: trivial: Python `import *` semantics (reference copy, not proxy) are immediately visible from the one-line shim files; any developer reading `from reasons.pubsub import *` understands identity is preserved.

### [ACCEPTED] reasons-lib-owns-no-unique-code
The `reasons_lib/` package is a deletion candidate: it defines zero functions, classes, or constants of its own, owns no state, and exists solely to preserve old import paths during the `reasons_lib` → `reasons` rename, so it can be removed in one pass once all consumers migrate.
- Source: summaries/2026/11/01/reasons_lib-mcp_client.md
- Rejected: trivial: "defines zero functions, classes, constants" is immediately visible from the single-line `import *` files; the "deletion candidate" framing is speculative editorial.

### [ACCEPTED] reasons-lib-shim-fails-hard-on-missing-canonical
The claim that importing `reasons_lib.<module>` raises `ModuleNotFoundError` if `reasons.<module>` is missing is a trivial consequence of Python's import system and adds no actionable constraint beyond the already-known one-directional dependency.
- Source: summaries/2026/11/01/reasons_lib-llm.md


---

**Generated:** 2026-11-01 (batch 13/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] reasons-lib-is-universal-shim-package
Every file in `reasons_lib/` is a single-line `from reasons.X import *` re-export that defines zero independent classes, functions, constants, or runtime logic; the entire package exists solely to preserve legacy import paths during the `reasons_lib` → `reasons` rename.
- Source: summaries/2026/11/01/reasons_lib-metadata.md
- Rejected: trivial: "every file is a single-line star import" is self-evident from reading the directory; no architectural insight beyond what the files show.

### [ACCEPTED] reasons-internal-modules-still-route-through-shim
At least 9 modules inside the canonical `reasons/` package (api.py, cli.py, storage.py, check_stale.py, compact.py, export_card.py, export_markdown.py, import_agent.py, import_beliefs.py) import from `reasons_lib.*` rather than `reasons.*`, making the shim a hard runtime dependency of active code.
- Source: summaries/2026/11/01/reasons_lib-network.md

### [ACCEPTED] shim-star-import-auto-tracks-canonical-exports
Because every shim uses `import *` (not a curated `__all__` list), any public symbol added to `reasons.X` is automatically visible through `reasons_lib.X` without editing the shim, and any symbol made private (`_`-prefixed) in the canonical module is silently dropped from the shim.
- Source: summaries/2026/11/01/reasons_lib-mind_service.md
- Rejected: trivial: `import *` auto-tracking public names and skipping underscore-prefixed names is standard Python import semantics, not a codebase-specific invariant.

### [ACCEPTED] shim-emits-no-deprecation-warning
Importing from any `reasons_lib.*` module produces no `DeprecationWarning` or `PendingDeprecationWarning`, so downstream consumers receive no programmatic signal that they should migrate to the `reasons.*` import path.
- Source: summaries/2026/11/01/reasons_lib-network.md
- Rejected: trivial: absence of `warnings.warn()` in a one-line `import *` file is immediately visible and requires no deep analysis.

### [ACCEPTED] shim-fails-fast-without-fallback
If the canonical `reasons.X` module fails to import (missing dependency, syntax error, circular import), the `reasons_lib.X` shim propagates the `ImportError` or `ModuleNotFoundError` unmodified with no try/except, no fallback, and no additional diagnostic context about the forwarding layer.
- Source: summaries/2026/11/01/reasons_lib-pg.md
- Rejected: trivial: absence of try/except in a one-line import statement is self-evident from reading the shim files.

### [ACCEPTED] test-suite-validates-shim-import-chain
25+ test files import from `reasons_lib.*` rather than `reasons.*`, meaning the test suite exercises the shim's re-export chain and would not independently detect if a canonical module's `__all__` were misconfigured to exclude expected symbols.
- Source: summaries/2026/11/01/reasons_lib-network.md
- Rejected: trivial: test files importing from `reasons_lib.*` is immediately visible from their import lines; the count is ephemeral.

### [ACCEPTED] reasons-lib-deletion-blocked-by-internal-importers
The `reasons_lib/` package cannot be safely deleted until at least 9 internal `reasons/` modules and 25+ test files are migrated to import directly from `reasons.*`, because all of them currently resolve their symbols through the shim's star-import re-export.
- Source: summaries/2026/11/01/reasons_lib-propose_update.md
- Rejected: speculative: "cannot be safely deleted" is an editorial judgment about future migration, not an observable code fact.


---

**Generated:** 2026-11-01 (batch 14/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] reasons-lib-is-uniform-star-reexport-shim
Every file in `reasons_lib/` is a single-line `from reasons.<module> import *` with `# noqa: F401,F403`, containing zero local definitions, functions, or logic — the entire directory is a uniform namespace-alias layer over `reasons/`.
- Source: summaries/2026/11/01/reasons_lib-pubsub.md
- Rejected: duplicate: same claim as `reasons-lib-is-universal-shim-package` — every file is a one-line star import with no local logic.

### [ACCEPTED] reasons-api-imports-through-reasons-lib-shim
`reasons/api.py` (inside the canonical package) imports pubsub, repair, repair_premises, report_belief, and report_gated symbols via the `reasons_lib` namespace rather than directly from `reasons.*`, creating a dependency from the canonical package to its own compatibility layer.
- Source: summaries/2026/11/01/reasons_lib-report_gated.md

### [ACCEPTED] reasons-lib-names-are-identical-objects-not-copies
For every public name `N` in `reasons.<module>`, `reasons_lib.<module>.N is reasons.<module>.N` evaluates to `True` (object identity, not equality), because the star import copies references rather than creating copies.
- Source: summaries/2026/11/01/reasons_lib-report_belief.md
- Rejected: duplicate: identical claim to `reasons-lib-shim-preserves-object-identity` — star import preserves object identity.

### [ACCEPTED] reasons-lib-has-no-unique-modules
Every module present in `reasons_lib/` has a corresponding file in `reasons/`; the sole module without a `reasons_lib` counterpart is `reasons/merkle.py`, which was added after the package rename and thus has no shim.
- Source: summaries/2026/11/01/reasons_lib-repair.md
- Rejected: trivial: 1:1 directory correspondence is visible from a directory listing; the merkle.py note is an ephemeral point-in-time observation.

### [ACCEPTED] reasons-lib-is-frozen-transitional-layer
The `reasons_lib/` package is a deprecated compatibility shim intended for eventual deletion; its docstrings explicitly direct new code to use `from reasons.<module>` imports, and the directory must never gain new local definitions or logic.
- Source: summaries/2026/11/01/reasons_lib-pubsub.md
- Rejected: trivial: "docstrings direct to use reasons.*" is visible in the files; "must never gain new definitions" is a normative/speculative constraint not grounded in enforced code.

### [ACCEPTED] star-import-is-time-bound-copy-not-live-proxy
The `from reasons.<module> import *` in each shim copies name bindings into the shim's `__dict__` at import time; if a module-level name in `reasons.<module>` is subsequently re-bound (e.g., `reasons.pubsub.some_func = new_ref`), the `reasons_lib.<module>` namespace retains the original reference and does not see the re-binding.
- Source: summaries/2026/11/01/reasons_lib-pubsub.md
- Rejected: trivial: time-bound name-binding copy (vs. live proxy) is a Python language semantic, not a codebase-specific architectural invariant.

### [ACCEPTED] star-import-shim-skips-underscore-names
The star import in each `reasons_lib/` shim does not re-export names prefixed with `_` unless the source module explicitly lists them in `__all__`, so private module-level attributes in `reasons.*` are invisible through the `reasons_lib` alias.
- Source: summaries/2026/11/01/reasons_lib-report_gated.md
- Rejected: trivial: `import *` skipping `_`-prefixed names is standard Python behavior, not a codebase-specific rule.


---

**Generated:** 2026-11-01 (batch 15/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] reasons-lib-is-universal-star-import-shim
Every file in `reasons_lib/` is a one-line `from reasons.<name> import *` re-export with no original logic; the two directories are structurally 1:1 file-for-file, making `reasons/` the sole source of truth for all business logic.
- Source: summaries/2026/11/01/reasons_lib-review.md
- Rejected: duplicate: identical claim to `reasons-lib-is-universal-shim-package` and `reasons-lib-is-uniform-star-reexport-shim`.

### [ACCEPTED] reasons-api-routes-through-reasons-lib-shim
`reasons/api.py` and at least two other first-party modules in `reasons/` (`repair.py`, `review_justifications.py`) import symbols via `reasons_lib.*` paths rather than `reasons.*` directly, making the compatibility shims load-bearing for internal production code, not just external consumers.
- Source: summaries/2026/11/01/reasons_lib-review.md

### [ACCEPTED] shim-preserves-object-identity
For every public name N re-exported by a `reasons_lib/` shim, `reasons_lib.X.N is reasons.X.N` evaluates to True (object identity, not equality), so monkey-patching, `isinstance` checks, and identity comparisons behave identically through either import path.
- Source: summaries/2026/11/01/reasons_lib-review.md
- Rejected: duplicate: identical claim to `reasons-lib-shim-preserves-object-identity` and `reasons-lib-names-are-identical-objects-not-copies`.

### [ACCEPTED] shim-propagates-exceptions-unchanged
The `reasons_lib/` shims contain no try/except, logging, wrapping, or error translation; any exception raised during import of the target `reasons/` module or raised at call time by a re-exported function propagates unchanged to the caller.
- Source: summaries/2026/11/01/reasons_lib-review_justifications.md
- Rejected: trivial: "no try/except, no logging, no wrapping" in a one-line `import *` statement is immediately visible from the file content.

### [ACCEPTED] reasons-lib-star-import-tracks-public-namespace
Because every shim uses `from reasons.X import *`, adding or removing a public name in any `reasons/X.py` module automatically propagates to the corresponding `reasons_lib/X.py` shim at import time with zero edits required to the shim itself.
- Source: summaries/2026/11/01/reasons_lib-review_premises.md

### [ACCEPTED] reasons-lib-removal-blocked-by-internal-importers
Deleting any `reasons_lib/` shim file would break `reasons/api.py` and multiple test modules (e.g., `test_review.py`, `test_storage.py`, `test_verify.py`) that import through the `reasons_lib` path; the shims are not dead code but active load-bearing compatibility layers until all ~15+ import sites are migrated.
- Source: summaries/2026/11/01/reasons_lib-storage.md

### [ACCEPTED] shim-target-must-not-import-its-own-shim
To prevent circular imports, the canonical module that a shim re-exports (e.g., `reasons/review.py`) must not import from its own `reasons_lib` counterpart (e.g., `reasons_lib/review.py`); the one-way dependency `reasons_lib → reasons` must be preserved for every leaf module pair.
- Source: summaries/2026/11/01/reasons_lib-review_premises.md

### [ACCEPTED] reasons-lib-shim-import-is-atomic-eager
Because the shims use `from … import *` (not `__getattr__`-based lazy resolution), the full target `reasons/` module must be importable and fully executed (side-effect-free at module level) at the moment the `reasons_lib` shim is first accessed; there is no deferred or partial resolution.
- Source: summaries/2026/11/01/reasons_lib-verify.md


---

**Generated:** 2026-11-01 (batch 16/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] dual-package-distribution-split
`reasons/` and `reasons_lib/` form a full-versus-library distribution split: `reasons` adds merkle proofs and metadata on top of the shared TMS core modules, while `reasons_lib` ships the lighter variant without those additions.
- Source: summaries/2026/11/01/scan-ftl-reasons.md

### [ACCEPTED] core-propagation-is-synchronous-single-process
TMS truth propagation is synchronous within a single process; the core inference engine has no event bus, no microservice boundary, and no async dispatch — all cascade effects resolve in-process before the calling operation returns.
- Source: summaries/2026/11/01/scan-ftl-reasons.md

### [ACCEPTED] llm-is-pluggable-enhancement-not-core-inference
LLM capabilities (summarization in `compact.py`, derivation assistance in `llm.py`) are a pluggable enhancement layered on the TMS core; the core inference engine (`network.py`, `derive.py`, `repair.py`) has no dependency on any LLM module.
- Source: summaries/2026/11/01/scan-ftl-reasons.md

### [ACCEPTED] pubsub-and-mind-service-are-optional-notification-hooks
`pubsub.py` and `mind_service.py` provide optional external notification hooks outside the core inference path; they are not required for TMS correctness and do not participate in propagation or evaluation.
- Source: summaries/2026/11/01/scan-ftl-reasons.md

### [ACCEPTED] tests-package-importability-required-by-test-pg
`tests/test_pg.py` resolves the `tests` package by name (`from tests import ...`), so the empty `tests/__init__.py` must exist for that import path to work correctly; deleting it makes the import fragile or broken.
- Source: summaries/2026/11/01/tests-__init__.md
- Rejected: trivial: any developer can see `from tests import ...` in test_pg.py and the empty `__init__.py` in the file tree; this is a basic Python packaging fact visible from imports and file structure

### [ACCEPTED] pg-teardown-deletes-in-fk-safe-order
The `pg_api` fixture teardown deletes rows in foreign-key-safe order (children before parents): `rms_propagation_log` → `rms_justifications` → `rms_nogoods` → `rms_network_meta` → `rms_nodes`; reversing this order would violate FK constraints.
- Source: summaries/2026/11/01/tests-conftest.md

### [ACCEPTED] pg-available-is-import-time-immutable
`pg_available` is a module-level boolean evaluated once at `conftest.py` import time as a snapshot of `DATABASE_URL`; setting or unsetting the environment variable after pytest starts has no effect on which tests are skipped.
- Source: summaries/2026/11/01/tests-conftest.md

### [ACCEPTED] nogood-visibility-is-atomic-any-restricted-hides-all
A nogood is hidden from a caller if ANY of its constituent nodes carries an `access_tags` restriction; the nogood is treated as an atomic unit for visibility filtering rather than being partially revealed.
- Source: summaries/2026/11/01/tests-test_access_tags.md

### [ACCEPTED] tag-representation-and-api-boundary-translation
Internally, an untagged node has the `access_tags` key entirely absent from its metadata dict (not an empty list), making it visible to all callers; the `trace_access_tags` API translates this key-absence into an explicit `[]` return value at the public boundary.
- Source: summaries/2026/11/01/tests-test_access_tags.md

### [ACCEPTED] api-add-justification-requires-justification-kwarg
`api.add_justification` requires at least one justification-type keyword argument (`sl=`, `cp=`, or `unless=`); calling it with none raises `ValueError` with a "Must provide" message rather than silently creating a no-op.
- Source: summaries/2026/11/01/tests-test_add_justification.md
- Rejected: duplicate: identical claim already exists as `api-add-justification-requires-justification-arg`


---

**Generated:** 2026-11-01 (batch 17/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] premise-count-reflects-per-justification-width
`premise_count` in API return dicts reflects per-justification antecedent width (1 for any_mode, SL size for normal, 0 for bare premise), not the total number of premises in the SL string.
- Source: summaries/2026/11/01/tests-test_any_mode.md

### [ACCEPTED] any-mode-nodes-never-produce-restoration-hints
`retract_node` does not emit restoration hints for any_mode nodes because each expanded justification is independent and a single retraction never cascades the node to OUT.
- Source: summaries/2026/11/01/tests-test_any_mode.md

### [ACCEPTED] surviving-premises-preserve-original-sl-order
The `surviving_premises` list in a restoration hint preserves the original SL (antecedent) order minus the OUT premises, making list order semantically significant.
- Source: summaries/2026/11/01/tests-test_any_mode.md

### [ACCEPTED] retract-node-always-returns-restoration-hints-key
`api.retract_node()` always returns a dict containing the `restoration_hints` key (an empty list when no hints apply), never omitting the key.
- Source: summaries/2026/11/01/tests-test_any_mode.md

### [ACCEPTED] init-db-raises-on-reinit-unless-force
`api.init_db()` raises `FileExistsError` when called on an existing database path unless `force=True` is passed, which overwrites.
- Source: summaries/2026/11/01/tests-test_api.md

### [ACCEPTED] proposal-accept-drift-detection-marks-stale
Accepting a proposal whose target state changed between propose and accept (node already OUT, node deleted, ID collision) results in `applied=False` and `status="stale"` rather than an exception.
- Source: summaries/2026/11/01/tests-test_api.md

### [ACCEPTED] api-llm-mocking-patches-consumer-namespace
LLM-dependent API tests must patch the imported name in the consumer module's namespace (e.g., `reasons.ask.invoke_model`) rather than the source module (`reasons.llm.invoke_model`) for the mock to take effect.
- Source: summaries/2026/11/01/tests-test_ask.md
- Rejected: trivial: Python mocking implementation detail visible directly in test source; not a system behavioral invariant

### [ACCEPTED] dual-mode-requires-sources-db
`ask(dual=True)` raises `ValueError` if `sources_db` is not provided; dual mode is inoperable without a source-document FTS5 database.
- Source: summaries/2026/11/01/tests-test_ask.md
- Rejected: duplicate: identical claim already exists as `ask-dual-requires-sources-db`

### [ACCEPTED] dual-mode-makes-exactly-three-llm-calls
Dual mode performs exactly three `invoke_model` calls in sequence: TMS-leg answer → FTS-leg answer → merged synthesis answer.
- Source: summaries/2026/11/01/tests-test_ask.md
- Rejected: duplicate: existing `ask-dual-mode-makes-three-llm-calls` already captures the 3-call structure (and is more precise with short-circuit note)

### [ACCEPTED] timeout-after-successful-search-preserves-results
If the LLM tool-call search succeeds before a subsequent `invoke_model` timeout, the fallback returns the already-found beliefs rather than `NO_BELIEFS_MSG`.
- Source: summaries/2026/11/01/tests-test_ask.md

### [ACCEPTED] mcp-tool-json-renders-no-trailing-comma
When a MCP tool has zero parameters, `_build_tools_section` renders the JSON example as `'{"tool": "name"}'` (no trailing comma), preventing LLM JSON-parser failures.
- Source: summaries/2026/11/01/tests-test_ask_mcp.md
- Rejected: trivial: a single string-formatting detail in `_build_tools_section` visible on inspection; no architectural or behavioral significance

### [ACCEPTED] extract-tool-call-is-tool-name-agnostic
`extract_tool_call` parses any valid `{"tool": "<name>", ...}` JSON object regardless of the tool name — there is no allow-list restricting which tools can be dispatched.
- Source: summaries/2026/11/01/tests-test_ask_mcp.md

### [ACCEPTED] mcp-bridge-duck-type-interface
The ask module's MCP dispatch expects any bridge object to implement `list_tools() → list[dict]`, `get_instructions() → str`, and `call_tool(name, arguments) → str`; no concrete class import is required.
- Source: summaries/2026/11/01/tests-test_ask_mcp.md

### [ACCEPTED] trace-assumptions-raises-keyerror-for-missing-node
`Network.trace_assumptions(node_id)` raises `KeyError` when the node does not exist in the network.
- Source: summaries/2026/11/01/tests-test_backtracking.md

### [ACCEPTED] add-nogood-inactive-when-either-node-out
`add_nogood` is a no-op returning `[]` (no log entry, no retraction) if either member of the nogood is already OUT.
- Source: summaries/2026/11/01/tests-test_backtracking.md

### [ACCEPTED] entrenchment-hierarchy-sourced-observation-dependents
The backtracking entrenchment score encodes a strict priority: sourced premises > unsourced, OBSERVATION type > DERIVED type, and fewer dependents = lower entrenchment (retracted first).
- Source: summaries/2026/11/01/tests-test_backtracking.md

### [ACCEPTED] trace-assumptions-returns-only-root-premises
`trace_assumptions` performs a transitive walk through justification antecedents and returns only root premise IDs — intermediate derived nodes never appear in the result.
- Source: summaries/2026/11/01/tests-test_backtracking.md


---

**Generated:** 2026-11-01 (batch 18/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] build-wiki-topic-assignment-is-prefix-based
`_assign_topics` maps each node to the first topic whose key is a prefix of the node ID; nodes matching no topic go to "Other" and are never dropped.
- Source: summaries/2026/11/01/tests-test_build_wiki.md

### [ACCEPTED] build-wiki-llm-failure-falls-back-to-template
When `generate_wiki_page` raises any exception during LLM-based wiki generation, `build_wiki` silently falls back to template rendering and still produces valid output.
- Source: summaries/2026/11/01/tests-test_build_wiki.md

### [ACCEPTED] build-wiki-linkify-longest-first-prevents-partial-match
`_linkify` sorts node IDs by length (longest first) before substitution to prevent a shorter ID like `node-alpha` from consuming the prefix of `node-alpha-extended`, and skips same-page references to avoid self-links.
- Source: summaries/2026/11/01/tests-test_build_wiki.md

### [ACCEPTED] build-wiki-slug-collision-produces-distinct-files
Two topics that slugify to the same filename produce two distinct output files rather than silently overwriting each other; the index page is also always created even for an empty database.
- Source: summaries/2026/11/01/tests-test_build_wiki.md

### [ACCEPTED] hash-sources-additive-unless-forced
`hash_sources` only fills nodes with an empty `source_hash`; it never overwrites an existing non-empty hash unless `force=True` is explicitly passed.
- Source: summaries/2026/11/01/tests-test_check_stale.md

### [ACCEPTED] upgrade-hashes-mutates-network-in-place
When `upgrade_hashes=True`, a stored hash that is a strict prefix of the current file's full SHA-256 is silently upgraded in-place (the node's `source_hash` attribute is mutated to the full hash) and is NOT reported as stale.
- Source: summaries/2026/11/01/tests-test_check_stale.md

### [ACCEPTED] check-stale-operates-per-node-not-per-file
Multiple nodes pointing to the same source file each receive their own independent result entry in the returned list; the function iterates nodes, not files.
- Source: summaries/2026/11/01/tests-test_check_stale_issue25.md
- Rejected: duplicate: `check-stale-no-dedup-by-source-path` already states multiple nodes referencing the same source each get independent entries

### [ACCEPTED] check-stale-skip-rules-precede-file-existence
Retraction, missing `source`, missing `source_hash`, and empty-string `source` all suppress results even when the referenced file is also missing on disk; the eligibility check precedes the filesystem existence check.
- Source: summaries/2026/11/01/tests-test_check_stale_issue25.md

### [ACCEPTED] empty-file-is-content-changed-not-deleted
A zero-byte file that exists on disk is never classified as `source_deleted`; if its hash differs from the stored hash it produces `content_changed`, if it matches it produces no result.
- Source: summaries/2026/11/01/tests-test_check_stale_issue25.md

### [ACCEPTED] classify-defeat-reason-returns-empty-string-on-failure
`classify_defeat_reason` never raises; every failure path (binary not found, subprocess exception, unparseable LLM output) returns the empty string `""`, allowing callers to distinguish "no classification" from an exception.
- Source: summaries/2026/11/01/tests-test_classify_defeaters.md

### [ACCEPTED] defeat-reason-types-is-closed-seven-element-set
`DEFEAT_REASON_TYPES` contains exactly 7 kebab-case strings; the parser performs a membership check against this set, so any new classification type must be added to this constant before the LLM prompt can reference it.
- Source: summaries/2026/11/01/tests-test_classify_defeaters.md
- Rejected: ephemeral: "exactly 7" is a specific count that changes with any edit; the closed-set membership behavior is the invariant but the count is a snapshot

### [ACCEPTED] classify-defeat-dry-run-is-write-free
When `dry_run=True`, `api.classify_defeat_reason_types` reads the database but performs zero writes; the node's `metadata` dict is provably unchanged (verified by re-reading and asserting the key is absent).
- Source: summaries/2026/11/01/tests-test_classify_defeaters.md

### [ACCEPTED] classify-defeat-idempotent-via-metadata-presence
A defeater node that already carries `defeat_reason_type` in its metadata is placed in the `"skipped"` bucket and is never re-sent to the LLM, making repeated batch invocations idempotent.
- Source: summaries/2026/11/01/tests-test_classify_defeaters.md

### [ACCEPTED] llm-mock-targets-subprocess-boundary
All LLM-dependent tests patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` specifically, ensuring the text-parsing and type-matching logic in `classify_defeat_reason` always executes for real while only the external process invocation is replaced.
- Source: summaries/2026/11/01/tests-test_classify_defeaters.md
- Rejected: trivial: test implementation detail (what patches are applied) visible by reading the test file; not a codebase behavioral invariant

### [ACCEPTED] init-requires-force-flag-for-reinitialization
Running `init` on an existing database without `--force` fails with exit code 1 and mentions `--force` in stderr, preventing accidental data loss from re-initialization.
- Source: summaries/2026/11/01/tests-test_cli.md

### [ACCEPTED] access-tags-gate-all-query-commands
No CLI command (show, explain, trace, status, trace-access-tags) leaks access-restricted content to unauthorized viewers; a node tagged `finance` is invisible to `--visible-to hr` queries across all inspection commands.
- Source: summaries/2026/11/01/tests-test_cli.md
- Rejected: duplicate: already covered by `access-control-enforced-at-read-not-write` which states access control is enforced at all read/query boundaries

### [ACCEPTED] names-only-format-outputs-bare-ids
When `--format names-only` is used, the output contains only bare node IDs with no brackets, status words, or labels — a strict contract for machine-parseable scripted consumption across multiple commands.
- Source: summaries/2026/11/01/tests-test_cli.md

### [ACCEPTED] build-wiki-returns-page-count-dict
`build_wiki` returning `{"pages": int, "total_nodes": int}` is a minor API signature detail unlikely to change semantics independently of the broader build_wiki contract already captured by other beliefs.
- Source: summaries/2026/11/01/tests-test_build_wiki.md


---

**Generated:** 2026-11-01 (batch 19/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] auto-k-formula-is-n-div-5-capped-at-20
The `_auto_k` helper computes a default cluster count as `min(n // 5, 20)` with a floor of 2; an explicit `n_clusters` override is capped at `n_beliefs`, and a separate `max_k` parameter provides an additional upper bound.
- Source: summaries/2026/11/01/tests-test_cluster.md
- Rejected: duplicate: already captured by `cluster-auto-k-heuristic` which documents the same `len(beliefs) // 5` clamped between 2 and 20 formula

### [ACCEPTED] list-clusters-guarantees-complete-partition
`list_clusters` returns a complete partition of the input: every belief ID provided as input appears in exactly one cluster across the full result, with no belief duplicated or omitted.
- Source: summaries/2026/11/01/tests-test_cluster.md

### [ACCEPTED] cluster-stats-cluster-sizes-sum-to-input-length
The `stats` dict returned by `cluster_beliefs` always contains `cluster_sizes` as a list of integers whose sum equals `len(beliefs)`, ensuring no beliefs are silently lost during the clustering reduction.
- Source: summaries/2026/11/01/tests-test_cluster.md
- Rejected: duplicate: already covered by `cluster-stats-sizes-sum-to-input` which states cluster_sizes always sums to total input

### [ACCEPTED] compact-in-section-orders-by-dependent-count-desc
Within the IN (active) section, `compact` orders nodes by dependent count descending (most-connected first), so that budget-driven truncation preferentially drops the least-connected nodes while preserving structurally important ones.
- Source: summaries/2026/11/01/tests-test_compact.md
- Rejected: duplicate: already covered by `compact-in-nodes-ordered-by-dependents`

### [ACCEPTED] nogood-ids-follow-deterministic-concatenation
Nogood IDs are deterministically constructed as `"nogood-" + "-".join(member_node_ids)`, providing a stable, content-derived identifier that survives serialization round-trips without requiring a separate counter.
- Source: summaries/2026/11/01/tests-test_compact.md

### [ACCEPTED] compact-budget-exhaustion-appends-omission-notice
When the global token budget is exhausted mid-section, `compact` appends a `"<N> more <section> omitted"` notice to the output rather than raising an exception or silently truncating, preserving the contract that `compact` never raises.
- Source: summaries/2026/11/01/tests-test_compact.md

### [ACCEPTED] contradiction-batch-failure-is-isolated-per-batch
A `RuntimeError` raised during one batch in `detect_contradictions` is caught locally and does not prevent remaining batches from executing; results from the failed batch are simply absent from the aggregated output.
- Source: summaries/2026/11/01/tests-test_contradictions.md

### [ACCEPTED] contradiction-plan-append-does-not-duplicate-header
Calling `api.write_contradiction_plan` with `append=True` must not repeat the `# Contradiction Plan` header; subsequent appends add `[APPLY]`/`[SKIP]` entries only, preserving plan-file integrity across multiple detection runs.
- Source: summaries/2026/11/01/tests-test_contradictions.md

### [ACCEPTED] contradiction-prompt-truncates-belief-text-at-200-chars
`format_beliefs_for_contradiction_check` caps each belief's rendered text to 200 characters followed by `"..."` to keep LLM prompt size bounded regardless of individual belief length, and silently skips IDs not present in the node map.
- Source: summaries/2026/11/01/tests-test_contradictions.md
- Rejected: duplicate: already covered by `belief-text-truncated-at-200-chars` and `contradictions-belief-text-truncated-at-200-chars`

### [ACCEPTED] mark-duplicate-and-superseded-reject-self-reference
Both `mark_duplicate(x, x)` and `mark_superseded(x, x)` raise `ValueError` with a message indicating self-reference is forbidden, leaving the node in its original `IN` truth state unchanged.
- Source: summaries/2026/11/01/tests-test_deduplication.md

### [ACCEPTED] mark-duplicate-overwrites-prior-retract-reason
Calling `mark_duplicate` or `mark_superseded` on a node that is already `OUT` (from a prior retraction) succeeds and overwrites the existing `retract_reason` metadata with the dedup/supersession-specific message (e.g., `"Duplicate of <canonical>"`) rather than failing or appending.
- Source: summaries/2026/11/01/tests-test_deduplication.md

### [ACCEPTED] dedup-supersession-cascade-flips-all-support-linked
After `mark_duplicate` or `mark_superseded` succeeds, every node linked via support-link (`sl=`) to the retracted node MUST appear in the returned `changed` list and MUST independently be in `OUT` truth state, guaranteeing no dependent is left in a dangling `IN` state.
- Source: summaries/2026/11/01/tests-test_deduplication.md
- Rejected: duplicate: already covered by `api-retract-cascade-is-transitive` (propagates OUT to all SL-dependent nodes) and `dedup-rewrites-both-antecedents-and-outlist`


---

**Generated:** 2026-11-01 (batch 20/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] defeater-is-first-class-graph-node
A defeater created by `defeat_justification` is a full first-class node in the belief graph with its own `truth_value` ("IN"), metadata, deterministic ID, and is exportable via `export_network` and retractable via `retract_node` like any other node — not a side-table, annotation, or flag.
- Source: summaries/2026/11/01/tests-test_defeat_justification.md

### [ACCEPTED] defeat-justification-error-contract
`defeat_justification` raises `KeyError` (message contains "not found") for missing nodes, `ValueError` (message contains "no justifications") for nodes with zero justifications, and `IndexError` (message contains "out of range") for out-of-range justification indices — it never returns an error dict for these conditions.
- Source: summaries/2026/11/01/tests-test_defeat_justification.md

### [ACCEPTED] defeater-types-are-enumerated
Exactly four defeater types are recognized and stored in `metadata["defeater_type"]`: `invalid-inference`, `over-generalizes`, `duplicate-of`, and `superseded-by`.
- Source: summaries/2026/11/01/tests-test_defeat_justification.md

### [ACCEPTED] premises-un-defeatable-at-justification-level
A node with no justifications (a premise/axiom) cannot be targeted by `defeat_justification`; the API raises `ValueError` because justification-level defeat requires at least one justification to exist.
- Source: summaries/2026/11/01/tests-test_defeat_justification.md

### [ACCEPTED] defeater-default-id-is-deterministic
When `defeater_id` is not provided, the system generates a default ID following the exact pattern `{defeater_type}-{target_node_id}-j{justification_index}` (e.g., `over-generalizes-belief-j0`).
- Source: summaries/2026/11/01/tests-test_defeat_justification.md

### [ACCEPTED] scope-beliefs-are-atomic-in-premises
Scope beliefs created by `defeat_with_scope` are always `IN` with zero justifications (atomic leaf premises), and carry `metadata.scope_of` (which antecedent they scope) and `metadata.for_defeater` (which defeater consumes them).
- Source: summaries/2026/11/01/tests-test_defeat_with_scope.md

### [ACCEPTED] scope-defeater-justification-links-to-scope-beliefs
The defeater node produced by `defeat_with_scope` has exactly one SL justification whose antecedents are the scope-belief IDs, making the defeater structurally derived from the scope beliefs rather than independently inserted.
- Source: summaries/2026/11/01/tests-test_defeat_with_scope.md

### [ACCEPTED] defeat-reason-type-absent-when-unset
When `defeat_reason_type` is empty or unset in `defeat_with_scope`, the key is deliberately absent from the defeater's metadata dict rather than present with an empty-string value.
- Source: summaries/2026/11/01/tests-test_defeat_with_scope.md

### [ACCEPTED] defeat-with-scope-requires-non-empty-findings
`defeat_with_scope` raises `ValueError` (message matches "scope_findings must not be empty") when the `scope_findings` parameter is an empty list, in addition to the standard KeyError/ValueError/IndexError for missing node, no justifications, and out-of-range index.
- Source: summaries/2026/11/01/tests-test_defeat_with_scope.md

### [ACCEPTED] rebuild-dependents-is-destructive-then-rebuild
`_rebuild_dependents` first clears all existing `dependents` sets on every node, then repopulates them from scratch — making it safe to call at any time as a repair mechanism and guaranteeing no stale entries survive.
- Source: summaries/2026/11/01/tests-test_dependents_integrity.md

### [ACCEPTED] verify-dependents-is-pure-observer
`verify_dependents` is a read-only auditor that re-derives expected dependents from justifications and diffs against stored values, returning a list of error strings without ever mutating `nodes[*].dependents`.
- Source: summaries/2026/11/01/tests-test_dependents_integrity.md

### [ACCEPTED] self-reference-antecedent-is-valid
A node listing itself as its own antecedent in a justification is allowed (degenerate but valid) and passes `verify_dependents` without error — the dependents index handles self-loops without crashing.
- Source: summaries/2026/11/01/tests-test_dependents_integrity.md

### [ACCEPTED] gate-belief-semantics-are-conjunctive-with-outlist
A gated (GATE) belief is IN if and only if all its antecedents are IN AND no unless/outlist node is IN; retracting an outlist node restores the gated belief to IN.
- Source: summaries/2026/11/01/tests-test_derive.md

### [ACCEPTED] prompt-template-validation-raises-valueerror
`build_prompt` raises `ValueError` with message "unknown placeholder" for unrecognized `{placeholder}` tokens in custom templates, and `ValueError` with message "malformed braces" for unclosed braces — these are programmer errors, not runtime data errors.
- Source: summaries/2026/11/01/tests-test_derive.md
- Rejected: duplicate: already captured by existing belief `build-prompt-validates-custom-templates` which states `build_prompt` raises `ValueError("unknown placeholder")` for unrecognized `{fields}` and a second ValueError for malformed braces

### [ACCEPTED] budget-section-graceful-emptiness
This claim (zero beliefs/agents/locals handled without raising) is a trivial consequence of any well-written loop over empty collections and does not represent a non-obvious invariant or architectural constraint a developer needs to know.
- Source: summaries/2026/11/01/tests-test_derive_budget.md


---

**Generated:** 2026-11-01 (batch 21/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] dialectical-errors-use-keyerror-not-custom-exception
`Network.challenge()` and `Network.defend()` raise the built-in `KeyError` (not a domain-specific exception) when the target or challenge node ID does not exist in the network.
- Source: summaries/2026/11/01/tests-test_dialectical.md

### [ACCEPTED] dialectical-naming-convention-is-deterministic
Challenge nodes are named `challenge-{node_id}` (with sequential `-2`, `-3`… suffixes for duplicates on the same target); defense nodes are named `defense-{challenge_id}`; the first challenge always gets the bare suffix.
- Source: summaries/2026/11/01/tests-test_dialectical.md
- Rejected: duplicate: challenge ID pattern with sequential suffixes is already captured by existing belief `challenge-id-auto-generation`

### [ACCEPTED] dialectical-metadata-links-are-bidirectional
A challenge node always records `metadata["challenge_target"]` pointing to the target, and the target always records the challenge ID in its `metadata["challenges"]` list; defenses follow the same pattern with `defense_target` and `defends`.
- Source: summaries/2026/11/01/tests-test_dialectical.md

### [ACCEPTED] dialectical-api-wraps-load-operate-save
The `reasons.api` dialectical functions (`challenge`, `defend`) wrap each call in a load → operate → save cycle against the SQLite file at `db_path`, ensuring the public API is a faithful pass-through to `Network` methods.
- Source: summaries/2026/11/01/tests-test_dialectical.md
- Rejected: duplicate: already captured by existing belief `api-uses-with-network-context-manager` which states the `_with_network` context manager ensures load-operate-save atomicity for all network operations in api.py

### [ACCEPTED] example-key-absent-when-unspecified
Omitting the `example` parameter in `api.add_node` results in the key being entirely absent from `node["metadata"]`, not present as `None` or an empty string.
- Source: summaries/2026/11/01/tests-test_example.md

### [ACCEPTED] show-commands-renders-example-with-2-space-indent
The `show` CLI command prints an `Example:` header line followed by every line of a multiline example indented by exactly two spaces; the `Example:` section is absent entirely when no example was set.
- Source: summaries/2026/11/01/tests-test_example.md

### [ACCEPTED] cli-tests-patch-sys-argv-not-subprocess
CLI integration tests simulate invocations by patching `sys.argv`, `sys.stdout`, and `sys.stderr` in-process (capturing `SystemExit`), testing only the Python entry point and not the installed script's shebang or shell integration.
- Source: summaries/2026/11/01/tests-test_example.md
- Rejected: duplicate: Already captured by `cli-tests-are-black-box-integration` which states tests invoke `main()` through the full argv-parsing pipeline in-process via the `run_cli` harness.

### [ACCEPTED] example-stored-verbatim-in-metadata
The `example` field is stored as the exact string provided (no trimming, escaping, or wrapping) and retrieved via `node["metadata"]["example"]` as a flat string value within the metadata JSON blob.
- Source: summaries/2026/11/01/tests-test_example.md
- Rejected: trivial: Storing a string in a dict and reading it back as the same string is immediately observable from the code; no invariant or contract beyond "it's a dict value."

### [ACCEPTED] export-card-belief-counts-are-exhaustive-partition
In the EEM card frontmatter, `beliefs_in + beliefs_out == beliefs_total` and `premises + derived == beliefs_total`, where a node is a premise if and only if it has zero justifications.
- Source: summaries/2026/11/01/tests-test_export_card.md

### [ACCEPTED] export-card-title-derived-from-project-name
The Markdown H1 title in the card body is produced by applying Python's `.title()` method to the `project_name` value (e.g., `k8s-ops-expert` → `K8S Ops Expert`).
- Source: summaries/2026/11/01/tests-test_export_card.md

### [ACCEPTED] export-card-defaults-are-explicit
`export_card` defaults to `license="mit"`, `domain=[]`, and `base_network=None` when callers omit these parameters; `type` is always the literal string `"eem"` and `schema_version` equals `reasons.metadata.SCHEMA_VERSION`.
- Source: summaries/2026/11/01/tests-test_export_card.md

### [ACCEPTED] export-card-max-depth-is-longest-justification-chain
The "Max derivation depth" reported in the card body equals the length (number of edges) of the longest justification chain in the network; a network with only premises has depth 0.
- Source: summaries/2026/11/01/tests-test_export_card.md

### [ACCEPTED] export-markdown-orders-in-before-out
Within the `## Claims` section of `export_markdown` output, all `[IN]` node entries appear at a lower character offset than any `[OUT]` or `[STALE]` entry.
- Source: summaries/2026/11/01/tests-test_export_markdown.md

### [ACCEPTED] stale-label-takes-precedence-over-out
A retracted node whose metadata contains `stale_reason` is labeled `[STALE]` (not `[OUT]`) in the export header, and the stale metadata fields (`stale_reason`, `superseded_by`) are rendered as additional bullet lines.
- Source: summaries/2026/11/01/tests-test_export_markdown.md

### [ACCEPTED] defeater-vs-regular-outlist-rendering
An outlist entry renders as `- Defeated by: <id> (<type>)` if and only if the referenced node carries `defeater_type` metadata; otherwise it renders as `- Unless: <id>`, and the two formats never appear for the same entry.
- Source: summaries/2026/11/01/tests-test_export_markdown.md

### [ACCEPTED] beliefs-type-metadata-overrides-default-label
If a node's metadata contains `beliefs_type`, that string replaces the default type label (`OBSERVATION` for premises, `DERIVED` for justified nodes) in the `### <id> [<status>] <TYPE>` header.
- Source: summaries/2026/11/01/tests-test_export_markdown.md

### [ACCEPTED] export-markdown-section-ordering-is-fixed
The document structure follows the fixed order: `# Belief Registry` → `## Claims` (all node entries) → `## Nogoods` (if any nogoods exist); an empty network still emits the section headers.
- Source: summaries/2026/11/01/tests-test_export_markdown.md

### [ACCEPTED] hf-401-always-maps-to-authentication-runtimeerror
A 401 HTTP response from any HuggingFace API call (create_repo or upload_file) is mapped to `RuntimeError` with a message matching "Authentication failed"; callers never see a raw `HTTPError`.
- Source: summaries/2026/11/01/tests-test_hf_default_org.py.md

### [ACCEPTED] hf-publish-uploads-exactly-three-files
`api.publish_hf` uploads precisely three files — `network.json`, `beliefs.md`, and `README.md` — to the target HF repo's `main` branch, and performs exactly four HTTP calls (one create + three uploads).
- Source: summaries/2026/11/01/tests-test_hf_default_org.py.md
- Rejected: ephemeral: "Precisely three files" and "exactly four HTTP calls" are point-in-time counts that invalidate with any single added file or retry logic change.

### [ACCEPTED] hf-token-resolution-follows-param-env-file-chain
The HF token is resolved in order: explicit `token` parameter → `HF_TOKEN` environment variable → `~/.huggingface/token` file; if none is found, a `RuntimeError` is raised before any network call is made.
- Source: summaries/2026/11/01/tests-test_hf_default_org.py.md

### [ACCEPTED] hf-create-repo-409-is-idempotent
A 409 HTTP response from the HF "create repo" endpoint is swallowed and treated as success (the repo already exists), making `publish_hf` safe to invoke repeatedly against the same repo name.
- Source: summaries/2026/11/01/tests-test_hf_default_org.py.md

### [ACCEPTED] hf-resolve-repo-id-preserves-explicit-org
`resolve_repo_id` prepends the default org (`EEM-Hub` or `REASONS_HF_ORG` env var) only to bare names without a `/`; an explicit `org/name` string or full URL is passed through unchanged.
- Source: summaries/2026/11/01/tests-test_hf_default_org.py.md


---

**Generated:** 2026-11-01 (batch 22/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] search-lookup-compact-hide-out-by-default
All three model-facing retrieval functions (`api.search`, `api.lookup`, `api.compact`) exclude OUT-status beliefs from their output by default; callers must explicitly pass `include_out=True` to include retracted beliefs.
- Source: summaries/2026/11/01/tests-test_hide_out.md

### [ACCEPTED] zero-result-sentinel-preserved-after-out-filtering
When the only belief matching a search or lookup query has OUT status, the API returns a sentinel string ("No results found" / "No beliefs found") rather than an empty string or list, preserving a distinguishable empty-result contract for callers.
- Source: summaries/2026/11/01/tests-test_hide_out.md

### [ACCEPTED] api-retrieval-functions-return-formatted-text
The retrieval functions `api.search`, `api.lookup`, and `api.compact` return human-readable markdown-formatted strings (not dicts), so callers and tests assert on substring presence/absence rather than structured field access.
- Source: summaries/2026/11/01/tests-test_hide_out.md
- Rejected: trivial: Return types (str vs dict) are immediately visible from function signatures and return statements; `api-functions-return-dicts` already notes str returns for markdown/compact functions.

### [ACCEPTED] import-agent-dual-format-parity
`import_agent` accepts both Markdown and JSON source files and produces structurally and behaviorally equivalent network states, with the JSON path explicitly preserving the full justification array (antecedents, outlist, type, label).
- Source: summaries/2026/11/01/tests-test_import_agent.md

### [ACCEPTED] import-agent-active-premise-carries-role-metadata
The synthetic `<agent>:active` premise node created by `import_agent` carries `role=agent_premise` in its metadata, distinguishing it from user-authored premises and enabling programmatic identification of infrastructure nodes.
- Source: summaries/2026/11/01/tests-test_import_agent.md

### [ACCEPTED] import-beliefs-observation-type-maps-to-premise
During `import_into_network`, claims with type `OBSERVATION` are imported as premises with an empty `justifications` list, making them un-defeatable root nodes whose truth value can only change via explicit retraction.
- Source: summaries/2026/11/01/tests-test_import_beliefs.md

### [ACCEPTED] import-beliefs-derived-type-maps-to-single-sl-justification
During `import_into_network`, claims with type `DERIVED` receive exactly one SL justification whose `antecedents` list matches the source document's "Depends on" field, establishing the dependency relationship in the network graph.
- Source: summaries/2026/11/01/tests-test_import_beliefs.md

### [ACCEPTED] hf-token-resolution-priority-chain
`_resolve_token` follows a strict priority order: explicit argument > `HF_TOKEN` environment variable > `~/.cache/huggingface/token` file > `None`; an explicit token always shadows the environment variable regardless of whether the env var is set.
- Source: summaries/2026/11/01/tests-test_import_hf.md

### [ACCEPTED] hf-download-translates-http-errors-to-runtimeerror
`download_network` converts raw `urllib` `HTTPError` exceptions into higher-level `RuntimeError` with actionable messages (401 → "Authentication required", 404 → "not found"), so callers of `api.import_hf` never encounter `HTTPError` directly.
- Source: summaries/2026/11/01/tests-test_import_hf.md

### [ACCEPTED] export-import-json-roundtrip-preserves-behavioral-equivalence
The `export_network` → `import_json` pipeline preserves not only structural fidelity (node count, justification structure, outlist entries, metadata) but also behavioral equivalence: defeasible propagation (retract, re-assert, cascade) on the imported database produces identical truth-value outcomes to the source database.
- Source: summaries/2026/11/01/tests-test_import_json.md

### [ACCEPTED] import-json-missing-file-raises-filenotfound
`api.import_json` raises `FileNotFoundError` when the specified path does not exist, rather than silently returning an empty import result — the error contract distinguishes "file not found" from "file had no nodes."
- Source: summaries/2026/11/01/tests-test_import_json.md


---

**Generated:** 2026-11-01 (batch 23/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] invoke-model-always-returns-string-or-raises
`invoke_model()` in `reasons/llm.py` always returns a `str` on success and raises an exception on failure — it never returns `None`, a list, or a raw dict.
- Source: summaries/2026/11/01/tests-test_llm.md

### [ACCEPTED] resolve-model-cmd-rejects-api-and-vertex-prefixes
`resolve_model_cmd()` raises `ValueError` when given a model string with `api:` or `vertex:` prefixes, since those are exclusively routed to the langchain-based `_invoke_api()` path.
- Source: summaries/2026/11/01/tests-test_llm.md

### [ACCEPTED] langfuse-handler-is-process-singleton
`_get_langfuse_handler()` creates the Langfuse tracing callback at most once per process (lazy singleton) and returns `None` when `LANGFUSE_PUBLIC_KEY` is unset.
- Source: summaries/2026/11/01/tests-test_llm.md

### [ACCEPTED] cost-tracking-is-module-level-accumulator
LLM cost accounting uses a module-level mutable dict in `reasons/llm.py` with explicit `reset_cost_tracker()` lifecycle — there is no class instance, and all callers share the same accumulator.
- Source: summaries/2026/11/01/tests-test_llm.md

### [ACCEPTED] llm-dispatch-routes-by-model-string-prefix
`invoke_model()` uses the model string prefix as the sole dispatch mechanism: strings without `api:`/`vertex:` go through CLI subprocess (claude, gemini, ollama), while `api:*` and `vertex:*` go through langchain SDK clients.
- Source: summaries/2026/11/01/tests-test_llm.md

---

### [ACCEPTED] mcp-tools-never-raise-return-error-json
Every MCP tool function in `reasons/mcp_server.py` catches internal exceptions and returns a JSON string containing `{"error": "<message>"}` rather than raising — the protocol contract is "JSON-or-text, never an exception."
- Source: summaries/2026/11/01/tests-test_mcp_server.md

### [ACCEPTED] proposal-lifecycle-is-idempotent-state-machine
Once a proposal reaches a terminal state (`accepted`, `rejected`, or `withdrawn`), any further `accept_proposal`, `reject_proposal`, or `withdraw_proposal` call returns an error rather than re-applying or mutating.
- Source: summaries/2026/11/01/tests-test_mcp_server.md

### [ACCEPTED] accept-proposal-detects-drift-before-applying
`accept_proposal()` compares the target node's current truth value against the proposal's expectation before mutating; if the node was retracted out-of-band, it returns `{"applied": false, "status": "stale"}` without modifying the network.
- Source: summaries/2026/11/01/tests-test_mcp_server.md

### [ACCEPTED] what-if-is-read-only-never-persists
The `what_if()` MCP tool computes the full retraction/assertion cascade in memory but never writes to the database — subsequent `show_node` calls reflect the original unmodified state.
- Source: summaries/2026/11/01/tests-test_mcp_server.md
- Rejected: duplicate: existing belief `both-backends-support-safe-hypothetical-reasoning` already states that both storage backends enable what-if reasoning without permanent mutation

### [ACCEPTED] mcp-db-discovery-precedence-env-then-cwd-then-default
`_find_db()` resolves the SQLite path with strict precedence: `REASONS_DB` environment variable, then walking up from `os.getcwd()`, then `api.DEFAULT_DB` — and `_get_db()` caches the result in the module-level `_db` attribute.
- Source: summaries/2026/11/01/tests-test_mcp_server.md

---

### [ACCEPTED] text-hash-is-sha256-of-node-text
A node's `text_hash` field is always the hex digest of `hashlib.sha256(node.text.encode())`, verified independently in tests against directly-computed SHA-256 values.
- Source: summaries/2026/11/01/tests-test_merkle.md

### [ACCEPTED] content-hash-folds-antecedent-text-hashes
A justification's `content_hash` incorporates the `text_hash` of every antecedent node, so changing an antecedent's text (even with a consistently-updated `text_hash`) invalidates the dependent's `content_hash` — this is the Merkle-tree property that distinguishes chain mutations from text mutations.
- Source: summaries/2026/11/01/tests-test_merkle.md

### [ACCEPTED] merkle-mutation-detection-is-transitive-upward
A single leaf-node text mutation cascades up the DAG: `check_integrity` flags all transitive ancestors (not just direct dependents) as chain mutations, because each ancestor's `content_hash` incorporates its child's hash.
- Source: summaries/2026/11/01/tests-test_merkle.md

### [ACCEPTED] verify-all-reports-corruption-never-raises
`verify_all()` / `api.check_integrity()` always returns a well-formed dict with `text_mutations`, `chain_mutations`, and `missing_hashes` keys — it reports corruption as data rather than raising exceptions, even on cyclic or corrupt graphs.
- Source: summaries/2026/11/01/tests-test_merkle.md

### [ACCEPTED] merkle-backfill-is-lossless
After `backfill_hashes()` fills empty `text_hash`/`content_hash` fields, a subsequent `check_integrity()` reports zero missing hashes and zero mutations — the backfill operation is idempotent and produces a clean graph.
- Source: summaries/2026/11/01/tests-test_merkle.md

---

### [ACCEPTED] created-at-is-immutable-after-init
The `created_at` metadata field is written once at `init_db` time and is never overwritten by subsequent `add_node`, `save`, or other mutations — it is a permanent audit timestamp.
- Source: summaries/2026/11/01/tests-test_metadata.md

### [ACCEPTED] project-name-defaults-to-db-filename-stem
When `project_name` is not explicitly provided to `init_db`, it defaults to the database filename minus its extension (e.g., `my-beliefs.db` → `"my-beliefs"`).
- Source: summaries/2026/11/01/tests-test_metadata.md

### [ACCEPTED] import-tolerates-absent-metadata-without-raising
Both JSON import (missing `"meta"` key) and Markdown import (missing YAML frontmatter) succeed with defaults rather than raising — the system must never crash on files written by older versions that lack the metadata envelope.
- Source: summaries/2026/11/01/tests-test_metadata.md

### [ACCEPTED] generator-stamp-contains-ftl-reasons-prefix
The `generator` metadata field always contains the substring `"ftl-reasons/"` followed by a version number — it is a provenance stamp present in all export formats (JSON, Markdown, SQLite).
- Source: summaries/2026/11/01/tests-test_metadata.md

### [ACCEPTED] updated-at-always-refreshed-to-current-time
Unlike `created_at`, the `updated_at` metadata field is always set to the current timestamp on every save operation, providing a "last modified" indicator that diverges from `created_at` after the first write.
- Source: summaries/2026/11/01/tests-test_metadata.md

---

### [ACCEPTED] strip-frontmatter-is-total-never-raises
`strip_frontmatter()` is a total function: on any input (including malformed, unclosed, or empty frontmatter) it returns a `(body, frontmatter_dict)` tuple without raising — malformed input yields `(original_text, {})`.
- Source: summaries/2026/11/01/tests-test_metadata_extended.md

### [ACCEPTED] frontmatter-values-are-strings-even-for-numerics
The `strip_frontmatter()` parser always returns frontmatter values as strings (e.g., `"node_count": "42"` rather than `42`), and single-quoted values have their quotes stripped.
- Source: summaries/2026/11/01/tests-test_metadata_extended.md

### [ACCEPTED] node-count-includes-retracted-nodes
The `node_count` metadata field reflects the total number of nodes in the network at export time, including retracted (OUT) nodes — it is a structural count, not an active-belief count.
- Source: summaries/2026/11/01/tests-test_metadata_extended.md

### [ACCEPTED] legacy-import-neither-crashes-nor-clobbers-metadata
Importing a file without metadata (legacy JSON without `"meta"`, Markdown without frontmatter) succeeds with defaults and does not destroy existing metadata in the target database — both crash-safety and data-preservation are guaranteed.
- Source: summaries/2026/11/01/tests-test_metadata_extended.md

### [ACCEPTED] updated-at-is-iso-8601-with-utc-offset
The `updated_at` field is always formatted as ISO-8601 with a `+00:00` UTC offset suffix (e.g., `"2026-11-01T12:00:00+00:00"`), distinguishing it from a naive local timestamp.
- Source: summaries/2026/11/01/tests-test_metadata_extended.md


---

**Generated:** 2026-11-01 (batch 24/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] migration-clears-both-retraction-metadata-keys
`migrate_retract_to_defeaters` atomically removes both `retract_reason` and `_retracted` from the target node's metadata; clearing only one constitutes a half-migrated state and is a bug.
- Source: summaries/2026/11/01/tests-test_migrate_defeaters.md

### [ACCEPTED] migration-dry-run-is-pure-read
`migrate_retract_to_defeaters(dry_run=True)` must not create any defeater node nor mutate any existing node's metadata; it is a pure read that only reports candidate counts.
- Source: summaries/2026/11/01/tests-test_migrate_defeaters.md

### [ACCEPTED] migration-defeater-id-encodes-justification-index
The migrated defeater node ID is always `f"migrated-retraction-{source_node_id}-j0"` where `-j0` denotes justification index 0, making the ID deterministic and referenceable for reversal.
- Source: summaries/2026/11/01/tests-test_migrate_defeaters.md

### [ACCEPTED] migration-requires-satisfied-justification
An OUT node whose antecedents are themselves OUT (unsatisfied justification) is placed in `result["skipped"]` and not migrated, because its OUT state is explained by the collapsed support chain rather than a retraction reason.
- Source: summaries/2026/11/01/tests-test_migrate_defeaters.md

### [ACCEPTED] migration-reversibility-through-defeater-retraction
After successful migration, the defeater node is the exclusive causal reason for the target's OUT state; retracting the defeater must restore the target to IN, guaranteeing lossless reversibility.
- Source: summaries/2026/11/01/tests-test_migrate_defeaters.md

### [ACCEPTED] push-belief-omits-empty-sl-key
When `sl` is `None`, `push_belief` omits the `sl` key entirely from the JSON POST body rather than sending `"sl": null`, allowing the remote API to distinguish "no antecedents" from "unspecified."
- Source: summaries/2026/11/01/tests-test_mind_service.md

### [ACCEPTED] export-api-fault-tolerant-per-node
`api.export_api` catches per-node HTTP errors internally, increments an `errors` counter, and continues pushing remaining nodes rather than aborting on the first failure; callers inspect `result["errors"]` rather than catching exceptions.
- Source: summaries/2026/11/01/tests-test_mind_service.md

### [ACCEPTED] mind-service-config-precedence-explicit-over-env
`_resolve_config` resolves parameters in strict precedence order (explicit keyword args > `MIND_SERVICE_URL`/`MIND_AGENT_ID`/`MIND_API_KEY` env vars), and raises `RuntimeError` if `url` or `agent_id` is unresolvable after both lookups.
- Source: summaries/2026/11/01/tests-test_mind_service.md

### [ACCEPTED] mind-service-url-strips-trailing-slash
`_resolve_config` always strips a trailing slash from the base URL before it is used for path concatenation, preventing double-slash URLs like `//api/agents/...`.
- Source: summaries/2026/11/01/tests-test_mind_service.md

### [ACCEPTED] fetch-export-maps-http-errors-to-runtimeerror
`fetch_export` raises `RuntimeError` with a domain-specific message for expected HTTP failures (401 → "Authentication failed", 404 → "Agent not found"), making it a "loud" low-level function contrasted with the "quiet" aggregate-level `import_api`/`export_api`.
- Source: summaries/2026/11/01/tests-test_mind_service.md

### [ACCEPTED] network-alternate-justification-keeps-node-in
A derived node with multiple `Justification` entries remains IN as long as at least one justification has all its antecedents IN; retracting antecedents of one justification does not flip the node to OUT if another justification is still satisfied.
- Source: summaries/2026/11/01/tests-test_network.md

### [ACCEPTED] network-log-is-append-only-audit-trail
`net.log` is an append-only list of action entries (add, retract, propagate, warn); no operation removes or reorders entries, and production code must not truncate it, ensuring a complete chronological audit trail.
- Source: summaries/2026/11/01/tests-test_network.md

### [ACCEPTED] network-add-duplicate-node-raises-valueerror
`Network.add_node` called with an ID that already exists in `net.nodes` raises `ValueError` (with "already exists" in the message) rather than overwriting or silently ignoring the duplicate.
- Source: summaries/2026/11/01/tests-test_network.md

### [ACCEPTED] cache-write-exception-preserves-prior-state
If an exception is raised inside a `with _with_network(db, write=True)` block, the context manager's `__exit__` discards the in-progress network and the pre-exception cached `Network` object remains intact and unmutated.
- Source: summaries/2026/11/01/tests-test_network_cache.md

### [ACCEPTED] cache-invalidation-is-mtime-only
The network cache uses the SQLite file's modification time (`st_mtime`) as its sole invalidation signal; it does not use inode numbers, file descriptors, or content hashes, so a same-tick content rewrite could produce a false cache hit.
- Source: summaries/2026/11/01/tests-test_network_cache.md

### [ACCEPTED] what-if-operations-mutate-cache-copy
`what_if_retract` and `what_if_assert` operate on a copy of the cached `Network` object, so their hypothetical mutations never corrupt the shared cache; the cached original retains its pre-hypothetical truth values.
- Source: summaries/2026/11/01/tests-test_network_cache.md

### [ACCEPTED] cache-read-returns-identical-object
Repeated read-mode calls to `_with_network` on the same DB path with unchanged mtime return the exact same Python `Network` object (identity via `is`), meaning any caller that mutates the returned object in-place corrupts the global cache for all subsequent readers.
- Source: summaries/2026/11/01/tests-test_network_cache.md

### [ACCEPTED] nogood-duplicate-receives-incremental-suffix
A second `add_nogood` call with the same node set appends `-2` (then `-3`, etc.) to the base ID rather than overwriting or rejecting the entry, ensuring unique IDs while preserving both nogoods.
- Source: summaries/2026/11/01/tests-test_nogood_id.md

### [ACCEPTED] nogood-legacy-ids-coexist-with-content-based
Nogood IDs imported from legacy JSON (`"nogood-005"`) or beliefs text (`"nogood-010"`) are stored verbatim and do not collide with newly generated content-based IDs (`"nogood-a-b"`); the ID namespace is a free-form string space where the content-based scheme is a convention for new entries only.
- Source: summaries/2026/11/01/tests-test_nogood_id.md

### [ACCEPTED] nogood-id-persists-opaque-through-storage
`Nogood.id` is stored as an opaque string in SQLite and is byte-identical after a `Storage.save` → `Storage.load` round-trip; no re-derivation or normalization occurs at the storage layer.
- Source: summaries/2026/11/01/tests-test_nogood_id.md


---

**Generated:** 2026-11-01 (batch 25/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] explain-trace-includes-violated-outlist
`Network.explain()` returns step dicts that include a `violated_outlist` key (list of IN node IDs) specifically when a justification is defeated by its outlist, in addition to the always-present `outlist` key showing the declared list.
- Source: summaries/2026/11/01/tests-test_outlist.md

### [ACCEPTED] api-unless-param-maps-to-outlist
The `api.add_node(..., unless="y", sl="a")` keyword parameters map to `Justification(type="SL", antecedents=["a"], outlist=["y"])`; `unless` is syntactic sugar for the outlist and `sl` for the inlist/antecedents.
- Source: summaries/2026/11/01/tests-test_outlist.md

### [ACCEPTED] assert-retract-is-idempotent
Retracting an already-OUT node or asserting an already-IN node returns an empty `changed` list with no cascading side effects, ensuring repeated calls are no-ops.
- Source: summaries/2026/11/01/tests-test_pg.md
- Rejected: duplicate: Already covered by existing belief `api-idempotent-retract-assert` which states the same idempotency contract.

### [ACCEPTED] access-tags-inherit-by-union
A derived node's effective access tags are the union of all its antecedents' tags, meaning visibility scope only tightens (grows) as you ascend the justification chain.
- Source: summaries/2026/11/01/tests-test_pg.md
- Rejected: duplicate: Already covered by existing belief `access-tags-union-inheritance` which states the derived node's tags are the union of antecedent tags.

### [ACCEPTED] cli-flags-take-precedence-over-env-vars
`_backend_kwargs` resolves the storage backend with CLI flags (`--pg`, `--domain-id`) overriding environment variables (`REASONS_PG_CONNINFO`, `REASONS_PROJECT_ID`); when neither is set, SQLite is the default.
- Source: summaries/2026/11/01/tests-test_pg_dispatch.md
- Rejected: duplicate: Already covered by existing belief `cli-flags-override-env-vars` which states CLI flags override environment variables.

### [ACCEPTED] pg-mode-requires-domain-id-or-exits
When PG is configured (via CLI flag or `REASONS_PG_CONNINFO` env) but `domain_id` cannot be resolved from either source, `_backend_kwargs` raises `SystemExit` rather than proceeding.
- Source: summaries/2026/11/01/tests-test_pg_dispatch.md

### [ACCEPTED] pgapi-used-as-context-manager-in-dispatch
The `_pg_dispatch` helper instantiates `PgApi(conninfo, domain_id)` inside a `with` block, so `PgApi` must implement `__enter__`/`__exit__`; exceptions inside the block propagate (not suppressed).
- Source: summaries/2026/11/01/tests-test_pg_dispatch.md

### [ACCEPTED] pin-sources-is-idempotent-without-force
`pin_sources` skips nodes that already have `metadata["pinned_sha"]` set, returning zero results unless `force=True` is explicitly passed to re-pin.
- Source: summaries/2026/11/01/tests-test_pin_sources.md

### [ACCEPTED] pin-lines-uses-1-indexed-inclusive-ranges
Line ranges in `pin_lines` are 1-indexed and inclusive (e.g., `"89-93"` covers lines 89 through 93); the function raises `ValueError` for `start < 1` or `start > end`, and `KeyError` for nonexistent node IDs.
- Source: summaries/2026/11/01/tests-test_pin_sources.md

### [ACCEPTED] pin-source-url-rewrites-github-blob-to-sha
`pin_source_url` rewrites GitHub `/blob/<branch>/<path>` URLs to `/blob/<sha>/<path>` using the provided commit SHA, passes through non-GitHub URLs unchanged, and uses a `source_path` hint to handle slashed branch names.
- Source: summaries/2026/11/01/tests-test_pin_sources.md

### [ACCEPTED] parse-update-proposals-is-total-function
`parse_update_proposals` never raises: malformed JSON, non-list input, or missing keys all produce `[]` or a filtered list of validated dicts; items lacking an `id` key are silently dropped rather than defaulted.
- Source: summaries/2026/11/01/tests-test_propose_update.md

### [ACCEPTED] update-proposals-validate-enums-with-safe-defaults
Invalid `action` values default to `"update"`, invalid `basis` to `"prior-knowledge"`, invalid `failure_mode` to `""` — ensuring every parsed proposal remains actionable even when the LLM hallucinates an enum value.
- Source: summaries/2026/11/01/tests-test_propose_update.md

### [ACCEPTED] propose-updates-defaults-to-in-nodes
When `belief_ids` is `None`, `propose_updates` selects all nodes with `truth_value == "IN"` for the LLM prompt; empty or all-nonexistent ID lists short-circuit to `[]` without invoking the model.
- Source: summaries/2026/11/01/tests-test_propose_update.md

### [ACCEPTED] cascade-rendering-is-action-dependent
RETRACT proposals with cascade data render "will go OUT" language (dependents are destroyed), while UPDATE proposals render "need re-review" language (dependents are invalidated, not retracted) — reflecting the semantic difference between the two actions.
- Source: summaries/2026/11/01/tests-test_propose_update.md


---

**Generated:** 2026-11-01 (batch 26/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] pubsub-subscriber-exception-isolation
A subscriber callback that raises an exception is caught and logged without preventing delivery of the same event to subsequent subscribers in the registry.
- Source: summaries/2026/11/01/tests-test_pubsub.md

### [ACCEPTED] pubsub-events-fire-post-persistence
Pubsub events are delivered to subscribers only after the database write has committed, so any side-effect in a subscriber callback (e.g., reading the DB) observes the already-persisted new state.
- Source: summaries/2026/11/01/tests-test_pubsub.md

### [ACCEPTED] pubsub-publish-noop-on-empty-list
If the `events` list passed to `publish` is empty, no subscriber callback is invoked at all — the dispatch loop short-circuits before iterating subscribers.
- Source: summaries/2026/11/01/tests-test_pubsub.md
- Rejected: trivial: "empty list produces no iteration" is the natural consequence of a for-loop over an empty collection, visible immediately from the code structure

### [ACCEPTED] pubsub-failed-mutation-emits-no-events
If an API mutation raises an exception (e.g., `KeyError` on a nonexistent node), the snapshot-diff-publish pipeline is never reached and zero events are delivered to subscribers.
- Source: summaries/2026/11/01/tests-test_pubsub.md

### [ACCEPTED] pubsub-scoped-subscriber-receives-matching-db
Subscriber scoping is inclusive (OR semantics): a subscriber registered without a `db_path` (global) AND a subscriber registered with a specific `db_path` both fire when an event for that database is published.
- Source: summaries/2026/11/01/tests-test_pubsub.md

### [ACCEPTED] remove-justification-cleans-dependents-exactly
When removing a justification, an antecedent is removed from the node's `dependents` set only if it does not appear in any remaining justification — shared antecedents are preserved.
- Source: summaries/2026/11/01/tests-test_remove_justification.md

### [ACCEPTED] remove-justification-outlist-entries-are-dependents
Outlist entries (defeaters) listed in a justification are registered in the antecedent node's `dependents` set and are cleaned up when the justification containing them is removed, exactly like antecedent references.
- Source: summaries/2026/11/01/tests-test_remove_justification.md

### [ACCEPTED] remove-justification-return-dict-contract
`Network.remove_justification` returns a dict with keys `removed` (the justification), `remaining` (int count), `old_truth_value`, `new_truth_value`, and `changed` (collection of node IDs whose truth flipped).
- Source: summaries/2026/11/01/tests-test_remove_justification.md

### [ACCEPTED] remove-justification-blocks-premise-and-last-justification
Removing a justification from a premise (no justifications) or from a node with exactly one justification both raise `ValueError`, enforcing that premises are axiomatic and a node cannot be left with zero justifications via this operation.
- Source: summaries/2026/11/01/tests-test_remove_justification.md

### [ACCEPTED] repair-candidate-filtering-excludes-antecedents-and-derived
In the smuggled-premise repair pipeline, candidate premises are filtered to exclude (a) IDs already present in the justification's antecedent list and (b) nodes that have their own non-empty `justifications` (derived beliefs), yielding `"no_candidates"` if the result is empty.
- Source: summaries/2026/11/01/tests-test_repair.md

### [ACCEPTED] repair-preserves-outlist-on-added-justification
When `repair_smuggled_beliefs` adds a matched premise as a new antecedent, the original justification's `outlist` is passed as the `unless` parameter to `add_justification`, preserving defeat semantics on the repaired node.
- Source: summaries/2026/11/01/tests-test_repair.md

### [ACCEPTED] repair-uses-status-strings-for-domain-failures
The `repair_smuggled_beliefs` function communicates domain-level failures through a `"status"` field in the result dict (values: `"repaired"`, `"no_candidates"`, `"no_match"`, `"extraction_failed"`, `"error"`) rather than raising exceptions.
- Source: summaries/2026/11/01/tests-test_repair.md

### [ACCEPTED] repair-match-ids-silently-filtered-to-candidates
`parse_match_response` validates LLM-returned `matched_ids` against the provided candidate set and silently drops any ID not in that set, returning a default `{"matched_ids": []}` on malformed JSON rather than raising.
- Source: summaries/2026/11/01/tests-test_repair.md

### [ACCEPTED] repair-rewrite-creates-successor-node
A `"rewrite"` action in premise repair never mutates the original node's text in place; it supersedes the old node (truth → OUT) and creates a new node (truth → IN) with the corrected text, preserving the belief-network audit trail.
- Source: summaries/2026/11/01/tests-test_repair_premises.md

### [ACCEPTED] parse-repair-defaults-to-retract
When the LLM omits the `"action"` field in its JSON response, `parse_repair_response` defaults to `"retract"` with `corrected_text=None`, making the safe (non-fabricating) action the default.
- Source: summaries/2026/11/01/tests-test_repair_premises.md

### [ACCEPTED] repair-requires-review-file-or-belief-ids
`api.repair_premises` raises `ValueError` immediately if neither `review_file` nor `belief_ids` is provided, enforcing that at least one target specifier is given before any LLM call or DB mutation.
- Source: summaries/2026/11/01/tests-test_repair_premises.md

### [ACCEPTED] repair-llm-failure-produces-error-dict
If the LLM subprocess raises (binary missing, network error), `repair_premises` catches the exception and returns a single `{"action": "error", ...}` entry for that premise rather than propagating the exception to the caller.
- Source: summaries/2026/11/01/tests-test_repair_premises.md

### [ACCEPTED] report-belief-raises-keyerror-on-missing-node
`api.report_belief` raises `KeyError` for a non-existent node ID — it does not return an empty report, a sentinel dict, or log-and-continue, making missing nodes a hard error at the API boundary.
- Source: summaries/2026/11/01/tests-test_report_belief.md

### [ACCEPTED] report-belief-premise-count-is-leaf-premises
The `premise_count` in `api.report_belief`'s return dict equals the exact number of leaf premises reachable through the `sl` justification chain, and the Markdown report heading `Root Premises (N)` must match this count.
- Source: summaries/2026/11/01/tests-test_report_belief.md

### [ACCEPTED] report-belief-out-nodes-visually-marked
An OUT-status node in a belief report must include `status: OUT` and a `[-]` visual marker, and any outlist blockers must be listed alongside the word `outlist` in the generated Markdown.
- Source: summaries/2026/11/01/tests-test_report_belief.md


---

**Generated:** 2026-11-01 (batch 27/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] api-report-gated-returns-fixed-shape
`api.report_gated` always returns a dict with exactly four keys: `report` (str), `blocker_count` (int), `gated_count` (int), `retracted_count` (int), even for an empty database.
- Source: summaries/2026/11/01/tests-test_report_gated.md

### [ACCEPTED] report-gated-blockers-sorted-descending
In both the structured report and the LLM prompt, active blockers are always sorted in descending order by the number of gated beliefs they control.
- Source: summaries/2026/11/01/tests-test_report_gated.md

### [ACCEPTED] report-gated-llm-invocation-is-opt-in
In the `report_gated` command, `invoke_model` is never called unless an explicit `model` string parameter is supplied; the no-model path produces only the structured Markdown report without any LLM narrative.
- Source: summaries/2026/11/01/tests-test_report_gated.md

### [ACCEPTED] report-gated-pure-formatters-never-raise
`_build_structured_report` and `_format_data_for_prompt` are pure functions that return strings unconditionally; they never raise exceptions even for empty networks, and the report always begins with the unconditional header `# Gated Beliefs Report`.
- Source: summaries/2026/11/01/tests-test_report_gated.md

### [ACCEPTED] repair-pipeline-has-four-triage-patterns
The repair pipeline classifies each invalid belief into exactly one of four LLM-determined patterns — `search_and_link`, `soften`, `abandon`, or `research` — each triggering a distinct mutation action (add justification, supersede with text, retract node, or set metadata only).
- Source: summaries/2026/11/01/tests-test_research.md

### [ACCEPTED] repair-skips-valid-beliefs-before-llm
Beliefs marked `valid: True` in the review results are filtered out before any LLM interaction, producing zero result entries and zero model calls — they never enter the triage/act/record pipeline.
- Source: summaries/2026/11/01/tests-test_research.md

### [ACCEPTED] repair-api-count-invariant
The repair API enforces the invariant `total_invalid == linked + softened + abandoned + failed + errors`, ensuring every invalid belief is accounted for in exactly one outcome bucket.
- Source: summaries/2026/11/01/tests-test_research.md

### [ACCEPTED] repair-aliases-are-identity-not-copies
`research_beliefs is repair_beliefs` and `api.research is api.repair` must hold as identity (`is`) checks, not equality — these are backward-compatibility shims that reference the same function object, not copies.
- Source: summaries/2026/11/01/tests-test_research.md
- Rejected: trivial: Identity of module-level aliases (research_beliefs is repair_beliefs) is immediately visible from the assignment statement in the code; no behavioral insight beyond naming.

### [ACCEPTED] repair-parsers-never-raise-use-status-strings
`parse_triage_response` and `parse_soften_response` never raise exceptions on malformed LLM output; they return dicts with empty-string defaults (e.g., `{"pattern": ""}`), and `repair_beliefs` maps any parse failure to a status string (`"triage_failed"`, `"soften_failed"`, `"extraction_failed"`) rather than propagating an exception.
- Source: summaries/2026/11/01/tests-test_research.md

### [ACCEPTED] review-metadata-classification-priority
Review result classification follows a strict priority: `valid=False` → `"invalid"`, else `sufficient=False` → `"insufficient"`, else `necessary=False` → `"unnecessary"`, else `"pass"` — the first failing axis determines the stored `review_result` metadata.
- Source: summaries/2026/11/01/tests-test_review.md

### [ACCEPTED] review-batch-size-bounds-llm-calls
`batch_size` in `review_beliefs` is a hard cap on LLM subprocess invocations: with N derived beliefs and `batch_size=K`, exactly `ceil(N/K)` calls to `subprocess.run` are made, verified by counting mock invocations.
- Source: summaries/2026/11/01/tests-test_review.md

### [ACCEPTED] namespace-empty-string-means-local-only
In the review pipeline, `namespace=""` filters to local-only nodes (those whose IDs do not contain a colon separator), as opposed to a specific namespace prefix match.
- Source: summaries/2026/11/01/tests-test_review.md

### [ACCEPTED] review-llm-process-failure-returns-empty
If `subprocess.run` raises (LLM binary crash, timeout, etc.), `review_beliefs` catches the exception and returns `[]` — the caller does not crash and no partial batch results are lost.
- Source: summaries/2026/11/01/tests-test_review.md

### [ACCEPTED] justification-review-defaults-to-all
`parse_justification_review` defaults the `classification` field to `"ALL"` (the conservative choice requiring all antecedents) when the LLM omits or nulls it, and normalizes all values to uppercase (`"any"` → `"ANY"`).
- Source: summaries/2026/11/01/tests-test_review_justifications.md

### [ACCEPTED] justification-review-is-read-only
`api.review_justifications` is strictly read-only: it must not mutate stored justifications, and tests explicitly re-read nodes after the call to assert the original SL antecedent lists are unchanged.
- Source: summaries/2026/11/01/tests-test_review_justifications.md

### [ACCEPTED] justification-review-eligibility-requires-multi-antecedent
Only nodes with at least one SL-type justification whose antecedent list length is ≥ `min_antecedents` (default 2) are sent to the model; premises (no justifications) and single-antecedent derived nodes are always excluded.
- Source: summaries/2026/11/01/tests-test_review_justifications.md

### [ACCEPTED] dual-package-layout-reasons-and-reasons-lib
The repo maintains two import trees (`reasons/` and `reasons_lib/`) with identical module names; tests import from `reasons`, and `pyproject.toml` may alias one to the other, so a developer must verify which tree is canonical before editing.
- Source: summaries/2026/11/01/tests-test_review_justifications.md
- Rejected: speculative: "may alias" and "must verify which tree is canonical" are unverified guesses and editorial judgments, not observable code facts

### [ACCEPTED] premise-review-skips-sourceless-nodes
Premise nodes without a `source` field are excluded from LLM review entirely, counted in the `skipped_no_source` summary field, and never sent to the model — the LLM requires source text to verify against.
- Source: summaries/2026/11/01/tests-test_review_premises.md

### [ACCEPTED] premise-review-batch-failures-non-fatal
A subprocess exception in one batch of `review_premises` does not abort the remaining batches; the function returns whatever results were successfully collected (possibly `[]`) rather than propagating the error.
- Source: summaries/2026/11/01/tests-test_review_premises.md

### [ACCEPTED] premise-review-parser-defaults-to-passing
`parse_premise_review_response` defaults missing LLM fields to safe passing values: `accurate=True`, `well_scoped=True`, `error_type=None`, `comment=""` — a vague or partial LLM response cannot trigger a false-positive repair action.
- Source: summaries/2026/11/01/tests-test_review_premises.md

### [ACCEPTED] premise-review-filters-to-source-backed-premises-only
The premise review pipeline filters to nodes that have no `sl` field (true premises) AND have a `source` field; derived nodes (with `sl`) are never entered into the review loop regardless of other attributes.
- Source: summaries/2026/11/01/tests-test_review_premises.md
- Rejected: duplicate: the source requirement is already captured by premise-review-skips-sourceless-nodes; the "no sl field" criterion is merely the definition of a premise (trivial)


---

**Generated:** 2026-11-01 (batch 28/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] roundtrip-omits-empty-fields
Markdown export must omit any `- Field: value` line when the field is None or empty string, preventing ambiguity between "not set" and "set to empty" in the serialized format.
- Source: summaries/2026/11/01/tests-test_roundtrip.md

### [ACCEPTED] double-roundtrip-is-idempotent
`export_markdown(import_into_network(export_markdown(net)))` must produce markdown identical to `export_markdown(net)`, with the sole permitted difference being `updated_at` timestamp lines that change per import.
- Source: summaries/2026/11/01/tests-test_roundtrip.md

### [ACCEPTED] nogood-export-format-mismatch
The nogood exporter writes IDs in `nogood-NNN` format (no colon, no label) but `parse_nogoods` expects `nogood-N: label` format, meaning nogood round-trip is currently broken and only the export side is tested.
- Source: summaries/2026/11/01/tests-test_roundtrip.md
- Rejected: ephemeral: "currently broken" is a point-in-time bug state that will be resolved by a fix; the underlying format facts are stable but the mismatch claim expires

### [ACCEPTED] roundtrip-preserves-retraction-and-stale-reason
A retracted node (truth_value "OUT") with a `stale_reason` must survive export→import unchanged, and the import result must report `claims_retracted == 1` so callers can detect retracted content in the imported markdown.
- Source: summaries/2026/11/01/tests-test_roundtrip.md

### [ACCEPTED] search-source-chunks-return-contract
`search_source_chunks` always returns a `list[dict]` (possibly empty) where each dict contains at minimum the keys `"text"`, `"filename"`, and `"section"`; it never returns None, a bare string, or raises for empty/stopword-only queries.
- Source: summaries/2026/11/01/tests-test_search_sources.md

### [ACCEPTED] fts5-index-is-content-backed
The `chunks_fts` virtual table is created with `content=chunks, content_rowid=id`, meaning FTS5 stores only the inverted index and all row data (filename, section, cluster) must be read back from the `chunks` content table via JOIN.
- Source: summaries/2026/11/01/tests-test_search_sources.md

### [ACCEPTED] search-source-chunks-missing-db-propagates-operational-error
Passing a non-existent or unreadable DB path to `search_source_chunks` raises `sqlite3.OperationalError` directly; the function does not catch, wrap, or convert the exception into a custom type or empty result.
- Source: summaries/2026/11/01/tests-test_search_sources.md

### [ACCEPTED] source-type-is-closed-four-value-enum
The valid `source_type` values are exactly `"code"`, `"document"`, `"self-description"`, and `"derived"`; any other string passed to `api.add_node` raises `ValueError` with a message matching `"Invalid source_type"` before any DB write occurs.
- Source: summaries/2026/11/01/tests-test_source_type.md

### [ACCEPTED] derive-apply-auto-stamps-source-type-derived
`apply_proposals` unconditionally sets `metadata["source_type"] = "derived"` on every newly created derived node, regardless of what the proposing LLM or caller specifies in the proposal dict.
- Source: summaries/2026/11/01/tests-test_source_type.md

### [ACCEPTED] list-nodes-defaults-source-type-to-empty-string
When a node has no `source_type` set, `api.list_nodes` returns `""` (an empty string, not None or a missing key) for the `source_type` field on that node's flat dict representation.
- Source: summaries/2026/11/01/tests-test_source_type.md

### [ACCEPTED] source-type-absent-means-key-missing-in-metadata
The `source_type` key is entirely absent from `node["metadata"]` when never set (not present as null or empty string); downstream consumers must check for key existence rather than truthiness or null-check.
- Source: summaries/2026/11/01/tests-test_source_type.md

### [ACCEPTED] storage-save-is-full-overwrite
Calling `Storage.save()` a second time on the same path completely replaces all prior network state (nodes, nogoods, log); it does not append, merge, or upsert — the result is equivalent to deleting the file and saving fresh.
- Source: summaries/2026/11/01/tests-test_storage.md

### [ACCEPTED] storage-rebuilds-dependents-index-from-justifications
The `dependents` reverse-index on nodes is not stored as a separate table or column; `Storage.load()` recomputes it by scanning justification rows, so after a round-trip the index is always consistent with the actual justification antecedent lists.
- Source: summaries/2026/11/01/tests-test_storage.md

### [ACCEPTED] loaded-network-supports-further-inference
A `Network` returned by `Storage.load()` is a fully operational inference engine (retract, restore, and propagation all work), not a frozen read-only snapshot — the storage layer must reconstruct all internal graph links including SL/justification connections.
- Source: summaries/2026/11/01/tests-test_storage.md
- Rejected: duplicate: `bootstrap-bypasses-incremental-propagation` already establishes that persistence loading constructs the full node graph before truth maintenance, making the network fully operational

### [ACCEPTED] summary-truth-is-conjunctive-over-covered-nodes
A summary node's truth value is "IN" if and only if every node in its `over` list is "IN"; retracting any single covered node immediately flips the summary to "OUT", and re-asserting restores it to "IN".
- Source: summaries/2026/11/01/tests-test_summarize.md

### [ACCEPTED] sl-justification-is-sole-summary-cascade-mechanism
The summary's truth-value propagation is entirely mediated by a single SL (Subsumption Link) justification whose antecedents are the covered node IDs; there is no separate watcher, subscription, or imperative link mechanism.
- Source: summaries/2026/11/01/tests-test_summarize.md

### [ACCEPTED] compact-hides-covered-nodes-only-when-summary-in
`compact()` suppresses covered nodes from its output only when the covering summary's truth value is "IN"; if the summary is "OUT" (due to a retracted covered node), all covered nodes reappear in the output unhidden.
- Source: summaries/2026/11/01/tests-test_summarize.md
- Rejected: duplicate: `compact-summary-hiding-requires-in` already states that a summary node only hides its covered nodes when the summary itself is IN

### [ACCEPTED] summarize-validates-existence-and-uniqueness
`Network.summarize()` raises `KeyError` if any ID in the `over` list does not exist in the network, and raises `ValueError` if the given summary ID is already in use — both are hard preconditions checked before any mutation.
- Source: summaries/2026/11/01/tests-test_summarize.md


---

**Generated:** 2026-11-01 (batch 29/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] sync-agent-first-call-equivalent-to-import
`sync_agent` on a brand-new agent (no prior `import_agent`) behaves identically to `import_agent` and returns `created_premise=True` in the result dict
- Source: summaries/2026/11/01/tests-test_sync_agent.md

### [ACCEPTED] sync-agent-idempotent-on-identical-state
Two consecutive calls to `sync_agent` with identical remote content produce zero mutations (all counters return 0) on the second call
- Source: summaries/2026/11/01/tests-test_sync_agent.md

### [ACCEPTED] sync-agent-must-preserve-agent-inactive-outlist
`sync_agent`'s internal `assert_node` calls must not strip `test-agent:inactive` from any justification's outlist, or the agent-revocation cascade mechanism breaks
- Source: summaries/2026/11/01/tests-test_sync_agent.md

### [ACCEPTED] sync-registers-repo-directory-mapping
After any `sync_agent` call, `api.list_repos` reflects the agent mapped to the parent directory of the source file
- Source: summaries/2026/11/01/tests-test_sync_agent.md

### [ACCEPTED] topics-result-shape-is-exact
`api.topics()` always returns a dict with exactly two top-level keys: `"topics"` (list of `{"topic": str, "count": int}`) and `"total_nodes"` (int); no additional keys are emitted
- Source: summaries/2026/11/01/tests-test_topics.md

### [ACCEPTED] topics-tokenizer-recognizes-four-delimiters
The topic tokenizer splits node IDs on exactly four delimiters: hyphen, dot, colon, and underscore; any new delimiter must be added to both the implementation and the corresponding test
- Source: summaries/2026/11/01/tests-test_topics.md
- Rejected: speculative: "any new delimiter must be added to both the implementation and the corresponding test" is editorial judgment about future maintenance, not a grounded code fact

### [ACCEPTED] topics-filters-short-words-and-stopwords
Topic extraction excludes tokens with fewer than 3 characters and a curated English stop-word list; both filters apply before frequency counting
- Source: summaries/2026/11/01/tests-test_topics.md

### [ACCEPTED] topics-cli-exits-zero-on-empty-graph
The `reasons topics` CLI command exits with code 0 and prints "No topics found" when the graph contains no nodes; empty data is a valid state, not an error
- Source: summaries/2026/11/01/tests-test_topics.md

### [ACCEPTED] parse-verify-response-never-raises
`reasons.verify.parse_verify_response` returns an empty dict `{}` for any malformed JSON, empty string, or missing `"verdict"` key — it never raises an exception
- Source: summaries/2026/11/01/tests-test_verify.md

### [ACCEPTED] verify-belief-raises-keyerror-for-unknown-node
`api.verify_belief("nonexistent-id")` raises `KeyError` (not a dict return or silent skip) when the node does not exist in the database
- Source: summaries/2026/11/01/tests-test_verify.md

### [ACCEPTED] verify-confirmed-stamps-verified-at
A CONFIRMED verdict from the LLM causes `api.verify_belief` to set a non-null `verified_at` timestamp on the node record in the database
- Source: summaries/2026/11/01/tests-test_verify.md

### [ACCEPTED] verify-dry-run-performs-no-llm-call-and-no-mutations
`dry_run=True` in `api.verify_belief` short-circuits before any LLM invocation and returns an empty `results` dict with `dry_run=True`, making the call fully read-only
- Source: summaries/2026/11/01/tests-test_verify.md

### [ACCEPTED] verify-trace-redirects-derived-beliefs-to-premises
When `trace=True` and the target node is derived (has an `sl` field), `api.verify_belief` verifies the premise nodes listed in `sl` rather than the derived node itself, and sets `is_derived=True` in the result
- Source: summaries/2026/11/01/tests-test_verify.md

### [ACCEPTED] contradiction-detection-batches-fifty-in-beliefs
`detect_contradictions` sends IN beliefs to the LLM in batches of 50 (`CONTRADICTION_BATCH_SIZE`), requesting NOGOOD sets of ≥2 mutually incompatible beliefs
- Source: summaries/2026/11/01/topic-dialectical-argumentation.md
- Rejected: duplicate: `contradictions-shuffles-before-batching` already states beliefs are partitioned into batches of 50

### [ACCEPTED] repair-pipeline-uses-three-defense-patterns
`reasons/repair.py` triages challenged beliefs into exactly three LLM-selected patterns: `search-and-link` (find and wire a missing antecedent), `soften` (rewrite the belief text to weaken the claim), and `abandon` (retract the belief)
- Source: summaries/2026/11/01/topic-dialectical-argumentation.md

### [ACCEPTED] defeat-reason-taxonomy-has-four-categories
`classify_defeat_reason` in `reasons/review.py` assigns one of four defeater types: `unsupported-conjunct`, `over-generalizes`, `scope-mismatch`, or `migrated-retraction`
- Source: summaries/2026/11/01/topic-dialectical-argumentation.md

### [ACCEPTED] dialectical-pipeline-is-three-stage
The challenge-and-defense pipeline flows in order: `contradictions.py` discovers NOGOODs → `network.py` executes formal retraction via outlist propagation → `repair.py` attempts defense (link/soften/abandon) before final commitment
- Source: summaries/2026/11/01/topic-dialectical-argumentation.md

### [ACCEPTED] reasons-lib-is-pure-reexport-shim
`reasons_lib/` contains only two star-import files (6 lines total) that re-export from `reasons/`; it has zero implementation code and exists solely as a backwards-compatibility alias for `import reasons_lib`
- Source: summaries/2026/11/01/topic-reasons-vs-reasons_lib.md

### [ACCEPTED] reasons-lib-lacks-functional-submodules
`reasons_lib` does not include `merkle`, `network`, `api`, `cli`, or `mcp_server` sub-modules; any `from reasons_lib.merkle import …` raises `ModuleNotFoundError`, while the equivalent `reasons.merkle` import succeeds
- Source: summaries/2026/11/01/topic-reasons-vs-reasons_lib.md

### [ACCEPTED] pyproject-ships-both-reasons-packages
The `[tool.setuptools] packages` list in `pyproject.toml` explicitly includes both `"reasons"` and `"reasons_lib"`, so `pip install ftl-reasons` installs the shim as a first-party package alongside the full implementation
- Source: summaries/2026/11/01/topic-reasons-vs-reasons_lib.md


---

**Generated:** 2026-11-01 (batch 30/30)
**Model:** ollama:qwen-64k:latest

### [ACCEPTED] node-truth-derives-from-justifications
Duplicates existing beliefs `node-in-if-any-justification-valid`, `truth-is-disjunctive-over-conjunctive-rules`, and `truth-semantics-are-emergent-and-uniform` which already capture that truth values are computed from justifications and premises default to IN.
- Source: summaries/2026/11/01/topic-tms-node-model.md

### [ACCEPTED] dependents-index-covers-outlist
Duplicates existing beliefs `dependency-tracking-is-complete-for-all-reference-types`, `every-network-mutation-maintains-dependents-invariant`, and `rewrite-dependents-updates-both-antecedents-and-outlists` which already establish that both antecedent and outlist edges populate the dependents index.
- Source: summaries/2026/11/01/topic-tms-node-model.md

### [ACCEPTED] retract-preserves-node
Duplicates existing belief `network-retracted-nodes-persist` which already states that retracted nodes remain in the network to enable restoration without rederivation.
- Source: summaries/2026/11/01/topic-tms-node-model.md

### [ACCEPTED] nogood-is-a-set-of-incompatible-inodes
Duplicates existing beliefs `contradictions-only-checks-in-beliefs`, `contradiction-management-is-complete-and-traceable`, and `contradiction-resolution-is-minimal-disruption` which already define nogoods as sets of mutually exclusive IN nodes.
- Source: summaries/2026/11/01/topic-tms-node-model.md

### [ACCEPTED] entrenchment-ranks-premises-above-derived
Duplicates existing beliefs `add-nogood-retraction-prefers-least-entrenched` and `backtracking-retracts-least-entrenched` which already establish that premises (evidence) are preferred over derived nodes (speculation) during retraction selection.
- Source: summaries/2026/11/01/topic-tms-node-model.md

### [ACCEPTED] entrenchment-uses-three-tier-ordering
Entrenchment scoring enforces a strict three-tier preference: premises (highest) > source-backed derived nodes (middle) > pure derived nodes (lowest), with dependent count as a tiebreaker within tiers, ensuring `find_culprits` always retracts speculation before evidence.
- Source: summaries/2026/11/01/topic-tms-node-model.md

### [ACCEPTED] domain-data-must-be-reconstructed-via-import
The belief graph (`network.json`), exported beliefs (`beliefs.md`), and individual entry files (`entries/`) are excluded from version control and must be reconstructed through import pipelines (`import_beliefs`, `import_agent`) rather than read from the repository.
- Source: summaries/2026/11/01/unknown.md

### [ACCEPTED] uv-lock-is-untracked-by-policy
`uv.lock` is deliberately excluded from version control, meaning dependency resolution is not pinned to a single transitive version set across developer and CI environments, introducing potential non-reproducibility unless compensated by exact version specs in `pyproject.toml`.
- Source: summaries/2026/11/01/unknown.md
- Rejected: speculative: The core fact (uv.lock listed in .gitignore) is trivially observable, and "introducing potential non-reproducibility" is an editorial judgment about consequences rather than a grounded behavioral invariant

### [ACCEPTED] gitignore-permits-no-exception-commits
The `.gitignore` contains no negation (`!`) patterns, meaning any file matching an ignore rule (e.g., `*.db`, `network.json`) cannot be committed via normal `git add` without `-f`, and test fixtures must avoid excluded extensions like `.db`.
- Source: summaries/2026/11/01/unknown.md
- Rejected: trivial: The claim (no `!` negation patterns in .gitignore) is directly readable from the file, and the stated consequences are simple logical inferences from that single observable fact

### [ACCEPTED] gitignore-uses-unanchored-patterns
The observation that patterns lack a leading `/` is an implementation detail of the `.gitignore` syntax, not an architectural invariant or behavioral contract a developer needs to know to work in the codebase.
- Source: summaries/2026/11/01/unknown.md

### [ACCEPTED] gitignore-covers-internal-tooling
The presence of `.sdlc-loop/` and `.code-expert/` entries is a trivial listing of internal tooling directories; it does not constrain how a developer writes code, configures the project, or reasons about system behavior.
- Source: summaries/2026/11/01/unknown.md


---

**Generated:** 2026-11-18
**Source:** 149 entries from /home/forge/workspace/entries/
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017


---

**Generated:** 2026-11-18 (parsed from ## Beliefs sections)
**Source:** 2 summary files

### [ACCEPTED] report-is-diart-update-run-ledger
`.forge/report.json` records a single run of the `diary-update` pipeline with exactly two ordered steps: `propose-beliefs` then `accept-beliefs`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] accept-bellefs-gate-accepted-zero
In this run, the `accept-beliefs` step accepted 0 of the 182 proposed beliefs while the belief store remained at `total_beliefs: 123`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] summary-mirrors-step-counts
`summary.entries`/`summary.proposed` equal the propose step's `counts.entries`/`counts.proposed`, and `summary.accepted`/`summary.total_beliefs` equal the accept step's `counts.accepted`/`counts.total_beliefs`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] step-counts-schema-is-step-specific
`counts` keys differ by step (`entries`/`proposed` vs `accepted`/`total_beliefs`), so consumers must select steps by `name` rather than assume a uniform schema.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] timestamps-are-utc-iso8601
All timestamps use ISO-8601 with a `+00:00` UTC offset, and all four recorded timestamps equal `2026-11-18T18:00:09+00:00`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] zero-accepted-is-completed-status
A run where `accepted == 0` is reported as `status: "completed"`, not as an error.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] diary-update-report-is-generated-run-artifact
`.forge/report.json` is a generated run-report artifact for the `diary-update` pipeline; this commit only refreshes run metadata and counts rather than editing domain logic.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md

### [ACCEPTED] diary-update-pipeline-has-propose-and-accept-steps
The `diary-update` pipeline records two ordered steps: `propose-beliefs` followed by `accept-beliefs`.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md

### [ACCEPTED] propose-beliefs-consumes-entries-and-produces-proposals
The `propose-beliefs` step reports its input as `entries` and its output as `proposed` beliefs.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md

### [ACCEPTED] accept-beliefs-tracks-accepted-and-total-beliefs
The `accept-beliefs` step records both the number of beliefs accepted in the run and the total belief count after processing.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md

### [ACCEPTED] report-summary-aggregates-pipeline-counts
The `.forge/report.json` summary aggregates `entries`, `proposed`, `accepted`, and `total_beliefs` from the pipeline steps.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md


---

**Generated:** 2026-11-18 (batch 1/1)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] forge-report-json-is-generated-run-ledger
`.forge/report.json` is a machine-readable run ledger for the `diary-update` pipeline, overwritten per run rather than append-only.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-counts-schema-varies-by-step-name
Each `steps[].counts` object has step-specific keys (`entries`/`proposed` for propose, `accepted`/`total_beliefs` for accept), so consumers must locate steps by `name` rather than assume a global counts schema.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-summary-mirrors-step-counts
The top-level `summary` is a denormalized projection of the per-step counts: `entries`/`proposed` from propose-beliefs and `accepted`/`total_beliefs` from accept-beliefs.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] zero-accepted-beliefs-is-valid-success-outcome
In the `diary-update` pipeline, `accepted == 0` is a valid successful outcome, not an error; the run still reports `status: "completed"`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-completed-status-implies-all-steps-completed
When `.forge/report.json` has top-level `status == "completed"`, `completed_at` is present and every `steps[*].status` is `"completed"`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-timestamps-are-utc-iso8601
All timestamps in `.forge/report.json` are ISO-8601 strings with explicit UTC offset `+00:00`, and `duration_seconds` is the precise timing signal because timestamps are second-resolution.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-shared-across-reasons-and-reasons-lib
Both `reasons/` and `reasons_lib/` contain modules that read `.forge/report.json`, so schema changes must be synchronized across the two trees.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-detail-is-non-authoritative
The `detail` string in step records is human-readable and non-authoritative; machine consumers must use `counts` and `status`.
- Source: summaries/2026/11/18/.forge-report.md

### [ACCEPTED] forge-report-is-generated-run-artifact
`.forge/report.json` is a generated run-report artifact for the `diary-update` pipeline; commits to it refresh run metadata and counts rather than editing domain logic.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md

### [ACCEPTED] diary-update-pipeline-records-two-ordered-steps
The `diary-update` pipeline records exactly two ordered steps: `propose-beliefs` followed by `accept-beliefs`.
- Source: summaries/2026/11/18/diff-0cc147af-.forge-report.md



---

**Generated:** 2026-11-19
**Source:** 149 entries from /home/forge/workspace/entries/
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] justification-type-convention-is-sl-or-cp
Justification.type is conventionally either "SL" or "CP", but the dataclass does not enforce this constraint.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-__init__.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-records-mutually-exclusive-node-set
A Nogood records a set of node IDs that cannot all be IN simultaneously.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-__init__.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-default-db-is-reasons-db
DEFAULT_DB in reasons_lib/api.py is "reasons.db" and is used as the default db_path by API functions.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] init-db-raises-fileexists-without-force
init_db raises FileExistsError when the target SQLite database already exists and force is not True.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] with-network-skips-save-on-exception
_with_network saves the network only when write=True and no exception occurred in the context body.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-justification-requires-type
add_justification raises ValueError when no justification type is specified.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] source-path-fallback-uses-tilde-git-repo
resolve_source_path falls back to ~/git/<repo-name>/... when a node's repo is not present in the repos mapping.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-propagates-file-read-errors
File read errors from hash_file, such as OSError or PermissionError, are not caught by check_stale and propagate to the caller.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] empty-stale-list-cannot-distinguish-fresh-from-untracked
An empty result from check_stale cannot distinguish all source files being fresh from no nodes having source tracking.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] source-path-splits-on-first-slash
resolve_source_path extracts the repo name by splitting a node's source string on the first slash.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-check_stale-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-db-flag-defaults-to-api-default-db
Every CLI subcommand accepts a --db flag defaulting to api.DEFAULT_DB.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-round-return-codes-are-zero-saturated-negative-one-error
_derive_one_round returns 0 when derivation is saturated and -1 on error.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-warns-on-three-premise-sl-without-any
The CLI warns at add time when an SL justification has 3 or more premises and --any was not used.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-default-budget-is-500-words
compact() defaults to budget=500 and truncate=True.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-out-reason-priority-is-retract-stale-supersede
compact annotates OUT nodes by checking metadata retract_reason and stale_reason first, then falling back to superseded_by.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] summary-coverage-uses-metadata-summarizes-key
compact identifies nodes covered by a summary through the metadata key "summarizes".
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-in-line-includes-first-non-summary-antecedents
Each emitted IN node line includes the antecedents of its first non-summary justification.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-detect-agents-skips-active
`_detect_agents` identifies multi-agent networks by colon-namespaced IDs and skips `:active` premise nodes when scanning.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-topic-filter-uses-or-semantics
`_filter_by_topic` matches space-separated terms case-insensitively against node ID and text, including a node if any keyword matches.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-sampling-uses-reservoir
`_sample_beliefs` uses reservoir-style random sampling to stay within token budgets and returns identity when the list is already under budget.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-build-beliefs-groups-by-id-prefix
Without agents, `_build_beliefs_section` groups beliefs by the first ID segment before `-`.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-derived-section-sorts-deepest-first
`_build_derived_section` renders existing derived conclusions sorted by depth deepest first, showing status, antecedents, and outlist.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-parser-supports-two-formats
`parse_proposals` supports a new v0.10+ format and an old v0.9 format, falling back to the old format only when the new format yields no matches.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-imports-unused-subprocess-sys-path
`derive.py` imports `subprocess`, `sys`, and `Path` but does not use them.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-build-prompt-returns-prompt-and-stats
`build_prompt` returns a tuple of prompt text and a stats dict.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-public-api-four-wrappers
`import_agent`, `import_agent_json`, `sync_agent`, and `sync_agent_json` are thin wrappers that normalize input and delegate to `_import_claims` or `_sync_claims`.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-active-is-bare-premise
`_ensure_agent_nodes` creates `agent:active` as a bare premise with no justifications, IN by default.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-inactive-justified-by-outlist-active
`agent:inactive` is justified by an SL justification with empty antecedents and `active` in its outlist, making it IN only when `active` is OUT.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-nogood-requires-two-existing-refs
Nogoods are imported only if at least two of their referenced nodes exist in the network after prefixing.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-has-no-try-except
`import_agent.py` contains no try/except blocks and relies on parsers, `Network.add_node`, dict defaults, and set operations for fault tolerance.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-build-justifications-synthetic-sl
`_build_justifications` gives even beliefs with no other justification a synthetic SL justification containing `agent:inactive` in the outlist.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-returns-stats-dict
All four public import/sync functions return a stats dict with counts such as added, skipped, and retracted.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] empty-outlist-vacuously-valid
An SL or CP justification with an empty outlist is valid whenever all its antecedents are IN, providing pure monotonic support.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] justification-validation-never-raises
`_justification_valid` raises no exceptions; missing antecedents and missing outlist nodes are handled inline.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] justification-validation-called-by-compute-truth-and-explain
`_justification_valid` is called from `_compute_truth` and `explain`.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] tms-node-in-if-any-justification-valid
`_compute_truth` marks a node IN when any one of its justifications is valid.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] justification-validation-assumes-in-out-string
`_justification_valid` assumes `truth_value` is either 'IN' or 'OUT'; any other string silently fails the comparison.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_justification_valid.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-uses-visited-set
`_propagate` uses a `visited` set seeded with `changed_id` to prevent cycles and redundant recomputation.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-traverses-dependents-forward
`_propagate` walks forward along `dependents` edges from the changed node and does not traverse antecedents.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-assumes-changed-id-exists
`_propagate` accesses `self.nodes[changed_id]` without a membership check, so a missing changed ID raises `KeyError`.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-logs-each-changed-node
`_propagate` appends a `propagate` log entry via `_log` for each node whose truth value changes.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-returns-empty-when-no-dependents
If the seed node has no dependents, `_propagate` returns an empty list.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-invoked-by-all-truth-mutators
`_propagate` is called after `retract`, `assert_node`, `supersede`, `add_justification`, `challenge`, and `convert_to_premise`.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagate-returns-changed-excluding-trigger
`_propagate` never includes the seed `changed_id` in its returned changed list; callers prepend it separately.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-_propagate.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-nogood-validates-node-ids-before-record
`add_nogood` iterates `node_ids` and raises `KeyError` before appending a `Nogood` record if any ID is missing.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-add_nogood.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-nogood-inactive-contradiction-returns-empty
If any node in the nogood set is already OUT, `add_nogood` returns an empty list immediately after recording the nogood.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-add_nogood.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-nogood-duplicates-silently-accepted
`add_nogood` silently accepts duplicate nogood records without raising an error.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-add_nogood.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-nogood-has-no-remove-api
There is no `remove_nogood`; once a nogood is recorded, it remains in `self.nogoods` permanently.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-add_nogood.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-nogood-does-not-persist
`add_nogood` mutates only in-memory state; persistence to `reasons.db` or `network.json` is the caller's responsibility.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-add_nogood.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-nogood-single-node-list-accepted
`add_nogood` accepts a single-element `node_ids` list without a validation error.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-add_nogood.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] challenge-node-is-premise
The challenge node itself is always created as a premise (no justifications, IN by default) and can only be driven OUT by explicit retraction or by being challenged/defended.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-challenge.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] challenge-propagation-is-automatic
After creating the challenge, truth value recomputation and BFS propagation through dependents happen within the same call; callers never need to trigger propagation manually.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-challenge.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] challenge-registers-dependency
After a challenge, the challenge node's dependents set includes the target_id so the target gets recomputed when the challenge changes.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-challenge.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] challenge-adds-to-target-metadata
The challenge method appends the new challenge ID to the target's metadata['challenges'] list.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network-challenge.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-class-has-nodes-nogoods-repos-log
The Network class holds four attributes: nodes (dict[str, Node]), nogoods (list[Nogood]), repos (dict[str, str]), and log (list[dict]).
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] retract-sets-retracted-metadata
The retract method forces a node OUT, marks _retracted in its metadata, and then cascades to dependents.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] summarize-node-is-in-when-all-summarized-nodes-are-in
The summarize method creates a derived node that is IN when all summarized nodes are IN.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] convert-to-premise-strips-justifications
The convert_to_premise method strips all justifications from a derived node, turning it into an unconditional premise.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] entrenchment-favors-premises-sources-and-many-dependents
The _entrenchment scoring heuristic ranks premises above derived nodes, sourced nodes above unsourced, and nodes with many dependents above those with few.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-schema-defines-six-tables
The SCHEMA constant defines six tables: nodes, justifications, nogoods, repos, propagation_log, and nodes_fts (an FTS5 virtual table).
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-rebuilds-fts-index-on-every-save
The nodes_fts full-text index is wiped and rebuilt alongside the nodes table in every save() call, so they cannot drift.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-load-assumes-valid-json
load() does not defend against invalid JSON in text columns; it assumes save() is the only writer and always writes valid JSON.
- Source: /home/forge/workspace/entries/2026/04/23/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] test-suite-has-18-files-and-211-tests
The test suite consists of 18 test files and 211+ tests.
- Source: /home/forge/workspace/entries/2026/04/23/scan-ftl-reasons.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-py-is-1500-lines
The api.py module is approximately 1500 lines long.
- Source: /home/forge/workspace/entries/2026/04/23/scan-ftl-reasons.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-has-35-subcommands
The CLI entry point (reasons command) exposes 35+ subcommands.
- Source: /home/forge/workspace/entries/2026/04/23/scan-ftl-reasons.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-uses-namespacing-and-active-relay-nodes
import_agent.py implements multi-agent belief federation using namespacing and active/inactive relay nodes.
- Source: /home/forge/workspace/entries/2026/04/23/scan-ftl-reasons.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-prioritizes-nogoods-then-out-then-in-by-dependents
compact.py orders summaries by priority: nogoods first, then OUT nodes, then IN nodes sorted by number of dependents.
- Source: /home/forge/workspace/entries/2026/04/23/scan-ftl-reasons.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sl-and-cp-stand-for-support-list-and-conditional-proof
In Doyle's TMS terminology, SL stands for Support List and CP stands for Conditional Proof.
- Source: /home/forge/workspace/entries/2026/04/23/topic-doyle-1979-tms.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] tms-separates-reasoning-from-bookkeeping
The TMS separates reasoning (which conclusions follow from which premises) from bookkeeping (which conclusions are currently believed given which premises are currently held).
- Source: /home/forge/workspace/entries/2026/04/23/topic-doyle-1979-tms.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cp-justifications-represent-consistency-based-derivation
CP justifications allow deriving conclusions from hypothetical assumptions that have not been contradicted, based on consistency rather than truth.
- Source: /home/forge/workspace/entries/2026/04/23/topic-doyle-1979-tms.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-defines-set-of-incompatible-nodes
A Nogood represents a recorded contradiction: a set of node IDs that cannot all be IN simultaneously.
- Source: /home/forge/workspace/entries/2026/04/23/topic-doyle-1979-tms.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] namespaces-discovered-by-active-premise-scan
Namespaces are discovered at runtime by scanning for nodes matching `*:active` with `role: agent_premise` metadata.
- Source: /home/forge/workspace/entries/2026/04/23/topic-multi-agent-federation.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-prefixes-antecedents-and-outlist
During agent import, antecedent IDs and remote outlist entries are prefixed with `agent_name:`; the imported belief's outlist also includes the agent's `inactive_id`.
- Source: /home/forge/workspace/entries/2026/04/23/topic-multi-agent-federation.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] inactive-relay-justification-uses-empty-antecedents
The `agent:inactive` node uses an SL justification with empty antecedents and `outlist=[active_id]`.
- Source: /home/forge/workspace/entries/2026/04/23/topic-multi-agent-federation.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] challenge-converts-bare-premise-to-justified
Challenging a bare premise with no justifications converts it into a justified node with `SL(antecedents=[], outlist=[challenge_id])`.
- Source: /home/forge/workspace/entries/2026/04/23/topic-outlist-semantics.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] challenge-premise-starts-in-and-defeats-target
A challenge creates a new premise node that starts IN, and adding it to the target's outlist makes the target go OUT until defended.
- Source: /home/forge/workspace/entries/2026/04/23/topic-outlist-semantics.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ten-defects-resolved-between-2026-04-23-and-2026-04-24
All 10 defect premises identified in the ftl-reasons knowledge base were resolved between 2026-04-23 and 2026-04-24.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagation-guards-dangling-dependent-references
`_propagate` was fixed to skip dangling dependent references with `if dep_id not in self.nodes: continue`, preventing a `KeyError`.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-agent-count-increment-moved-outside-loop
`_build_beliefs_section` was fixed by moving the belief-count increment outside the per-belief loop, preventing count inflation and non-agent budget shrinkage.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-returns-source-deleted-result
`check_stale` returns an explicit `source_deleted` result with `reason: "source_deleted"`, `new_hash: None`, and `source_path: None` when a tracked source file is missing.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-token-estimate-uses-chars-div-4
`estimate_tokens` uses the `len(text) // 4` BPE approximation instead of whitespace-separated word count.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-budget-applies-to-all-output-sections
The compact output budget applies to nogoods, OUT, and IN sections using a running character counter.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-file-stores-full-sha256
`hash_file()` stores and compares full 64-hex-character SHA-256 hashes after removing the `[:16]` truncation.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-merge-skips-gate-beliefs-during-auto-retraction
ftl-merge 0.2.0 added `load_network()` and `has_outlist()` to skip GATE beliefs during `--auto-retract`, preventing `_retracted` metadata from blocking TMS restoration.
- Source: /home/forge/workspace/entries/2026/04/24/defect-resolution-clean-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] setuptools-68-is-build-backend
`pyproject.toml` declares `setuptools >= 68.0` as the PEP 517 build backend.
- Source: /home/forge/workspace/entries/2026/04/24/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pytest-is-only-optional-dependency
The only optional dependency is `pytest` under a test extra; no linters, formatters, or type checkers are declared.
- Source: /home/forge/workspace/entries/2026/04/24/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] package-list-explicit-reasons-lib
`[tool.setuptools]` explicitly lists `reasons_lib` as the only package to include, preventing auto-discovery of tests or scripts.
- Source: /home/forge/workspace/entries/2026/04/24/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pytest-testpaths-tests
`[tool.pytest.ini_options]` points pytest at `tests/` for test discovery.
- Source: /home/forge/workspace/entries/2026/04/24/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-with-network-write-false-discards-changes
`_with_network(db_path, write=False)` loads the network but does not save on exit, so in-memory mutations are discarded.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-add-node-auto-wires-namespace-active-premise
When `namespace` is provided to `add_node`, it creates an `ns:active` premise and injects it as an antecedent of the node's justification.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-search-expands-by-one-hop
`search` expands results by one hop in the dependency graph (neighbors).
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-search-supports-three-output-formats
`search` supports three output formats: markdown, JSON, and minimal.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-missing-node-raises-keyerror
`show_node`, `explain_node`, `trace_assumptions`, and `what_if_*` raise `KeyError` when a node ID is not found.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-hidden-node-raises-permissionerror
When `visible_to` is set and a target node's access tags are not a subset, `show_node`, `explain_node`, `trace_assumptions`, and `trace_access_tags` raise `PermissionError`.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-requires-source-and-source-hash
A node must have both source and source_hash metadata to be eligible for stale-checking.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-source-path-bare-filename-current-directory
When a source string has no slash, resolve_source_path treats it as a bare filename in the current directory.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-sources-manifest-includes-was-empty-flag
hash_sources returns a manifest record with a was_empty flag to distinguish backfills from overwrites.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-classifies-content-changed-and-source-deleted
check_stale categorizes results as content_changed when the hash differs and source_deleted when the resolved path is missing.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-check-stale-exits-nonzero-on-stale
cmd_check_stale calls sys.exit(1) when any stale node is found.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] every-cli-subcommand-has-db-argument
Every CLI subcommand inherits a --db argument defaulting to api.DEFAULT_DB.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-add-justification-warns-multi-premise-and-mode
cmd_add and cmd_add_justification call _warn_multi_premise when an AND justification uses 3 or more antecedents.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-what-if-output-notes-database-not-modified
cmd_what_if explicitly notes 'database NOT modified' in its output.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-summary-hiding
Nodes whose IDs appear in a summary node's metadata['summarizes'] list are excluded from the visible IN node output.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-estimate-tokens-char-div-4-floor-one
estimate_tokens approximates token count as len(text) // 4 and returns at least 1.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-omitted-markers-include-count
When the budget is exhausted, compact emits '... (N more omitted)' markers for omitted nodes.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-header-includes-date-and-counts
compact emits a header containing the date, total node counts, and nogood count.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-summary-nodes-prefixed-and-marked-coverage-count
Summary nodes are emitted with a [summary] prefix and a '(covers N nodes)' suffix.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-retraction-guard
Proposals whose ID has at least 50% Jaccard token overlap with any OUT node ID are rejected.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-parser-falls-back-to-legacy-format
parse_proposals tries the current ### DERIVE format first and only falls back to the legacy bold-field format if zero matches are found.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-depth-is-max-antecedent-depth-plus-one
Premises have depth 0 and derived nodes have depth equal to the maximum antecedent depth plus 1.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-depth-memoizes-and-guards-cycles
_get_depth memoizes computed depths and sets memo[node_id] to 0 before recursion as a cycle guard.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-prompt-detects-multi-agent-networks
build_prompt detects multi-agent networks and adjusts the prompt accordingly.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-creates-active-and-inactive-nodes
_ensure_agent_nodes creates agent:active as a premise node and agent:inactive with SL(antecedents=[], outlist=[agent:active]) for each agent.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-prefixes-ids-with-agent-name-colon
Every node imported from agent foo has an ID starting with foo:.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-nogoods-require-two-valid-nodes
_import_nogoods only creates a nogood if at least two of its referenced nodes exist in the network.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] normalize-markdown-maps-stale-or-out-to-is-out
_normalize_markdown maps STALE and OUT statuses to is_out=True.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-beliefs-truncates-text-to-first-non-metadata-line
`parse_beliefs` sets each claim's `text` to the first non-metadata, non-empty line after the header, truncating multi-line descriptions.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-beliefs-does-not-validate-dependencies
`parse_beliefs` returns a flat list of claim dicts and does not validate that referenced `depends_on` or `unless` IDs exist.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-nogoods-require-two-existing-affected-nodes
`import_into_network` imports a nogood only if at least two of its affected nodes already exist in the network; otherwise it silently drops the nogood.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-into-network-returns-summary-counts
`import_into_network` returns a summary dict containing `claims_imported`, `claims_skipped`, `claims_retracted`, and `nogoods_imported`.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-implements-doyles-1979-tms
`Network` implements Doyle's 1979 Truth Maintenance System algorithm with nodes, justifications, and automatic truth-value propagation.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] defense-is-challenge-to-challenge
`Network.defend()` is implemented as `challenge(challenge_id, ...)`, making a defense a challenge to the original challenge node.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] trace-assumptions-uses-dfs-to-find-premises
`Network.trace_assumptions()` performs a DFS backward through justification chains to collect all premise nodes a conclusion depends on.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-belief-set-returns-in-node-ids
`Network.get_belief_set()` returns the IDs of all nodes whose `truth_value` equals `IN`.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] recompute-all-bounded-by-node-count-plus-one
`Network.recompute_all()` bounds its fixpoint iterations to `len(nodes) + 1` to guarantee termination.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-save-is-delete-rewrite
`Storage.save()` deletes all rows from every table and reinserts the entire network inside a single SQLite transaction; it never diffs or upserts.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-dependent-index-not-persisted
The `dependents` reverse index is not stored in SQLite; `Storage.load()` rebuilds it from justifications by calling `network._rebuild_dependents()`.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-enables-sqlite-foreign-keys
`Storage.__init__` enables SQLite foreign key enforcement when opening the database connection.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-load-reads-tables-in-dependency-order
`Storage.load()` reads nodes first, then justifications, then nogoods, repos, metadata, and propagation log to reconstruct the network.
- Source: /home/forge/workspace/entries/2026/04/24/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] access-tags-not-fabricated-for-untagged-parents
When all parent nodes are untagged, a derived node receives no inherited access tags; the system does not fabricate tags.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] trace-access-tags-raises-keyerror-for-missing-node
`Network.trace_access_tags()` raises `KeyError` when given a nonexistent node ID.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] search-neighbor-expansion-respects-visible-to
`lookup` and `search` exclude forbidden nodes from results, including during neighbor expansion.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-status-counts-only-visible-nodes
`get_status` produces a status summary that counts only nodes visible to the caller's `visible_to` clearance.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-justification-nonexistent-node-raises-keyerror
`Network.add_justification()` raises `KeyError` when the target node ID does not exist.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_add_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-justification-appends-not-replaces
`Network.add_justification()` appends a new justification to the node's existing justification list rather than replacing existing justifications.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_add_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-add-justification-supports-cp-and-unless
`api.add_justification()` supports CP justifications and outlist-only `unless` justifications in addition to SL justifications.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_add_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-add-justification-namespace-prefixes-result-id
`api.add_justification()` applies the configured namespace prefix to the node ID in its result, for example returning `ns:b` rather than `b`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_add_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] any-mode-single-premise-no-expansion
When `any_mode=True` is used with a single premise, no expansion occurs.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_any_mode.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] default-sl-conjunctive-single-justification
Without `any_mode`, `api.add_node` with `sl='a,b,c'` creates a single conjunctive justification with three antecedents.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_any_mode.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] any-mode-add-justification-can-restore-out-node
`api.add_justification` with `any_mode=True` expands premises post-creation and can restore an OUT node to IN.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_any_mode.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] any-mode-expansion-implemented-in-api-layer
`any_mode` expansion is implemented in `reasons_lib.api.add_node` and `reasons_lib.api.add_justification`, not in the `Network` propagation engine.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_any_mode.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-nodes-depth-filter-min-max
`api.list_nodes` depth filtering uses `min_depth` and `max_depth`; `min_depth=1` excludes depth-0 premise nodes and `max_depth=0` excludes derived nodes.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-log-last-n-truncates
`api.get_log(last=N)` returns only the most recent N log entries.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] explain-node-supports-premise-and-chain
`api.explain_node` provides explanations for both premise nodes and derived chain nodes.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-source-path-empty-source-returns-none
`resolve_source_path` returns `None` for an empty source string.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-sources-skips-missing-source-or-file
`hash_sources` reports only successful backfills, silently skipping nodes that lack a `source` field or whose source file is missing.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-requires-in-source-and-hash
`check_stale` only examines nodes that are IN, have a non-empty `source`, and have a non-empty `source_hash`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-source-path-defaults-to-home-git
When `repos` is `None`, `resolve_source_path` falls back to resolving source paths under `~/git/`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] empty-file-not-source-deleted
An empty file on disk is not treated as a deleted source; if its hash does not match, `check_stale` reports `content_changed`, not `source_deleted`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] source-deleted-results-per-node
If multiple IN nodes reference the same deleted source file, `check_stale` returns a separate `source_deleted` result for each node.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-empty-network-reports-zero-nodes
For an empty network, `compact` outputs the text `0 nodes tracked`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-truncate-clips-text-with-ellipsis
When `truncate=True`, `compact` clips long node text and appends `...`; when `truncate=False`, it preserves full node text.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-shows-dependencies-and-dependent-counts
The `compact` output displays each derived node's antecedents, such as `<- a`, and the number of nodes that depend on it.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] rebuild-dependents-repairs-corruption
`_rebuild_dependents()` removes injected phantom entries and reconstructs each node's `dependents` set from justification references.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_dependents_integrity.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-validation-requires-antecedent-existence
`validate_proposals` rejects any proposal whose antecedents or unless-list reference nodes that are not present in the network.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-parse-strips-backticks
`parse_proposals` strips backticks from proposal IDs and antecedent IDs.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-parse-distinguishes-derive-and-gate
`parse_proposals` distinguishes DERIVE proposals from GATE proposals, where GATE proposals supply outlist entries.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dedup-clusters-by-jaccard-on-ids
`deduplicate` clusters beliefs by Jaccard similarity on their IDs.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-build-prompt-computes-stats
`build_prompt` computes summary statistics including `total_in`, `total_derived`, and `max_depth`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-agent-budget-proportional-formula
The agent belief budget in `_build_beliefs_section` is computed as `max(5, int(max_beliefs * len(agent_beliefs) / total_all))`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-ceiling-enforced
In `_build_beliefs_section`, the sum of agent-shown and local-shown beliefs never exceeds `max_beliefs`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-section-handles-empty-inputs
`_build_beliefs_section` returns a valid formatted string for empty nodes, empty agents, or agents pointing to missing nodes.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-remaps-dependencies
Importing an agent rewrites justification dependency references to use the agent's namespace prefix.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-only-in-excludes-out-beliefs
With `only_in=True`, `import_agent` skips STALE/OUT beliefs entirely so they are absent from the database.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-imports-nogoods
`import_agent` imports and counts nogood contradiction records from a co-located nogoods file.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-returns-counts-and-control-id
`import_agent` returns a dict containing the agent name, namespace prefix, active node ID, premise-creation flag, and counts of imported and retracted claims.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-returns-count-summary
`import_into_network` returns a summary dict with counts `claims_imported`, `claims_retracted`, `claims_skipped`, and `nogoods_imported`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-pipeline-is-lenient-not-fail-fast
The beliefs.md import pipeline is designed to skip bad data and return counts rather than raise exceptions for malformed or missing references.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] imported-network-supports-retraction-cascade
After importing SAMPLE_BELIEFS, retracting `premise-a` causes `derived-c` and `chain-e` to become OUT.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-real-registry-smoke-test-skips-missing-file
`TestImportRealRegistry` uses `pytest.skip` when `~/git/beliefs-pi/beliefs.md` does not exist.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-tests-separate-parse-and-import-failures
The import test suite has separate `TestParseBeliefs` and `TestImportIntoNetwork` classes so parse bugs are distinguishable from import bugs.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derived-node-evaluates-on-creation
Adding a derived node evaluates its justification immediately, and the new node starts OUT if its antecedents are OUT.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-missing-node-operations-raise-keyerror
`Network.retract`, `assert_node`, `explain`, and `add_nogood` raise `KeyError("not found")` when referencing a nonexistent node.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-belief-set-returns-in-node-ids
`Network.get_belief_set()` returns the IDs of all nodes whose truth value is IN.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] propagation-log-records-add-retract-propagate
The propagation audit trail records `add`, `retract`, and `propagate` events with their targets.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] explain-out-node-includes-failed-antecedents
`Network.explain()` on an OUT node includes `failed_antecedents` identifying what broke the justification.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-ids-format-sequential-nnn
Nogood IDs are formatted as `nogood-NNN` with a zero-padded sequential number starting at `nogood-001`.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_nogood_id.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-tests-use-tmp-sqlite-db-fixture
The nogood ID tests use a `db` fixture that creates a fresh SQLite database in `tmp_path` for each test.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_nogood_id.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-tests-simulate-schema-migration-by-dropping-meta-table
The backwards-compatibility tests drop the `network_meta` table with raw SQL to simulate databases created before the monotonic counter fix.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_nogood_id.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-tests-use-two-phase-storage-reopen
Persistence tests save a `Storage` instance, close it, and open a new `Storage` on the same path to prove the counter is written to disk.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_nogood_id.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-removes-absent-beliefs
`sync_agent` retracts to OUT any existing agent belief whose node ID is absent from the synced file.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-updates-belief-text
`sync_agent` updates the stored belief text when the corresponding file text changes.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-rewires-justification-antecedents
`sync_agent` updates a node's justification antecedents when its `Depends on:` list changes.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-truth-change-cascades-to-dependents
When `sync_agent` changes a node's truth value, BFS propagation cascades the change to dependent nodes.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-supports-markdown-and-json-input
`sync_agent` supports both beliefs.md markdown and JSON input formats.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-tests-lack-error-path-coverage
The sync agent test suite contains no tests for malformed markdown, missing files, or conflicting agent names.
- Source: /home/forge/workspace/entries/2026/04/24/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-constructor-takes-conninfo-project-id
`PgApi` is instantiated with `(conninfo, project_id)`.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-uses-psycopg-v3
`PgApi` uses psycopg v3 as its PostgreSQL driver, and psycopg is an optional dependency not required by CLI users.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-schema-reuses-expert-service-tables
PgApi uses the tables `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, and `rms_network_meta`.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-tests-require-database-url
The PgApi tests in `tests/test_pg.py` are marked `@pytest.mark.pg` and skipped when `DATABASE_URL` is not set.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-list-nodes-missing-challenged-depth-filters
PgApi `list_nodes()` does not support the `challenged`, `min_depth`, or `max_depth` filter parameters.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-has-no-async-wrapper
`PgApiAsync` using `psycopg.AsyncConnection` is not implemented.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-search-uses-plainto-tsquery
PgApi search uses `plainto_tsquery` instead of manually quoted `to_tsquery` to handle user input containing single quotes.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-no-cli-backend-flag
The CLI has no `--backend pg` flag or `DATABASE_URL` handling for selecting PgApi.
- Source: /home/forge/workspace/entries/2026/04/28/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] changelog-follows-keep-a-changelog-1-1-0
CHANGELOG.md follows the Keep a Changelog 1.1.0 convention with version headers and Added/Fixed/Changed sections.
- Source: /home/forge/workspace/entries/2026/04/29/CHANGELOG.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] all-releases-are-minor-version-bumps
All 15 releases in CHANGELOG.md are minor-version bumps (0.x.0); no patch-level releases are present.
- Source: /home/forge/workspace/entries/2026/04/29/CHANGELOG.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] version-0-11-0-is-missing-from-changelog
The CHANGELOG.md sequence jumps from 0.10.0 to 0.12.0 with no 0.11.0 entry.
- Source: /home/forge/workspace/entries/2026/04/29/CHANGELOG.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] version-0-18-0-introduces-access-tags
Version 0.18.0, dated 2026-04-25, introduced RBAC-style access tags for provenance-based filtering.
- Source: /home/forge/workspace/entries/2026/04/29/CHANGELOG.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] version-0-1-0-shipped-complete-tms
Version 0.1.0 shipped with SL justifications, retraction cascades, nogood detection, dialectical argumentation, SQLite persistence, and a multi-command CLI.
- Source: /home/forge/workspace/entries/2026/04/29/CHANGELOG.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] versions-0-15-0-through-0-17-0-share-date
Versions 0.15.0, 0.16.0, and 0.17.0 all have the release date 2026-04-18.
- Source: /home/forge/workspace/entries/2026/04/29/CHANGELOG.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-exhaust-run-added-119-derived-beliefs
A derive --exhaust run of 10 rounds added 119 new derived beliefs to the network.
- Source: /home/forge/workspace/entries/2026/04/29/gated-and-negative-analysis.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] four-active-blockers-gate-14-derived-beliefs
After the derive --exhaust run, 4 active blockers prevented 14 derived beliefs from going IN.
- Source: /home/forge/workspace/entries/2026/04/29/gated-and-negative-analysis.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-missing-namespace-and-import-export
PgApi is missing namespace support, import/export, and some maintenance operations; PRs #52-56 added challenge, defend, compact, list-gated, and what-if but did not close those gaps.
- Source: /home/forge/workspace/entries/2026/04/29/gated-and-negative-analysis.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-propagate-bypasses-api-layer
The cmd_propagate CLI handler goes directly from Storage to Network instead of through the API layer.
- Source: /home/forge/workspace/entries/2026/04/29/gated-and-negative-analysis.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] negative-belief-classifier-reduced-196-candidates-to-9
The negative belief classifier reduced 196 keyword candidates to 9 genuinely negative beliefs, a 95.4% false-positive rate among candidates.
- Source: /home/forge/workspace/entries/2026/04/29/gated-and-negative-analysis.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cp-justifications-use-sl-evaluation-logic
CP justifications use SL evaluation logic.
- Source: /home/forge/workspace/entries/2026/04/29/gated-and-negative-analysis.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] setuptools-build-backend-floor-68
The build system requires setuptools >=68.0 as the PEP 517 backend.
- Source: /home/forge/workspace/entries/2026/04/29/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] static-version-0-22-0
pyproject.toml declares version 0.22.0 as a static, manually bumped project version.
- Source: /home/forge/workspace/entries/2026/04/29/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pytest-collects-only-tests-directory
pytest is configured with testpaths = ["tests"], restricting collection to the tests directory.
- Source: /home/forge/workspace/entries/2026/04/29/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] test-pg-extra-duplicates-test-and-pg-dependencies
The test-pg optional dependency duplicates the dependencies of the test and pg extras instead of referencing them via cross-extra references.
- Source: /home/forge/workspace/entries/2026/04/29/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] namespace-resolution-preserves-existing-colon
The `_resolve_namespace` function prefixes bare IDs with `namespace:` but leaves IDs already containing `:` untouched.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] add-node-namespace-auto-wiring-creates-active-premise
When a namespace is supplied, `add_node` creates the `ns:active` premise and prepends it to the new node's antecedents.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cascade-depth-uses-bfs-shortest-path
The `_cascade_depth` function performs BFS through dependents edges to find the shortest path from a retracted node to a target.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-gated-finds-out-nodes-with-in-outlist-blockers
`list_gated()` returns OUT nodes whose outlist contains IN nodes, identifying beliefs gated by an IN blocker.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] export-access-filtering-constructs-in-memory-network
`export_network` and `export_markdown` support access-tag filtering by constructing a filtered `Network` in memory.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] deduplicate-clusters-by-jaccard-id-token-similarity
`deduplicate()` clusters IN beliefs by Jaccard similarity on ID tokens and uses union-find to group similar IDs.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dedup-plan-workflow-is-human-in-the-loop
`write_dedup_plan`, `parse_dedup_plan`, and `apply_dedup_plan` implement a human-in-the-loop deduplication workflow via an editable plan file.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] explain-node-delegates-to-network-explain
`explain_node()` delegates to `Network.explain()` for a step-by-step proof trace.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-reports-content-changed-and-source-deleted
check_stale returns problem records with one of two reason values: content_changed when the file hash differs, or source_deleted when the resolved file path no longer exists.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-skips-nodes-without-source-and-hash
check_stale silently skips nodes that do not have both source and source_hash set, emitting no error or report entry.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-sources-force-rehashes-existing-hashes
hash_sources with force=True re-hashes nodes that already have a source_hash and returns a was_empty flag distinguishing forced re-hashes from initial backfill.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-source-path-bare-filename-current-directory
resolve_source_path treats a source string with no slash as a bare filename in the current directory.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-sources-walks-all-truth-values
hash_sources walks all nodes regardless of truth value, unlike check_stale which only examines IN nodes.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-delegates-to-api-except-propagate
Every cmd_* handler delegates to reasons_lib.api except cmd_propagate, which directly instantiates Storage, loads the network, calls recompute_all(), and saves.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-threads-db-path-through-api
Every CLI command handler threads args.db through to the API, with args.db defaulting to api.DEFAULT_DB.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-what-if-never-modifies-database
cmd_what_if is read-only and never modifies the database; its output explicitly states (database NOT modified).
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-derive-shells-out-to-claude-or-gemini
cmd_derive shells out to an external CLI (claude or gemini) via asyncio.create_subprocess_exec to generate belief proposals.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-add-warns-for-three-premises-without-any
cmd_add and cmd_add_justification call _warn_multi_premise to hint about --any mode when 3+ premises form an AND conjunction without --any.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-deduplicate-has-plan-and-apply-phases
cmd_deduplicate supports a two-phase workflow: generating a dedup plan file, or applying one via --accept.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-in-nodes-sorted-by-dependent-count
compact() sorts IN nodes by len(node.dependents) descending, so more-depended-on beliefs appear first and survive budget truncation.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-estimate-tokens-is-length-div-four-min-one
compact.estimate_tokens approximates token count as max(1, len(text) // 4).
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-appends-omitted-sentinel-when-budget-exhausted
When compact() stops adding items because the budget would overflow, it appends a ... (N more omitted) sentinel and ends that section.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-validate-rejects-jaccard-similar-to-out
validate_proposals skips any proposal whose ID has at least 50% Jaccard token similarity to an existing OUT belief, preventing re-derivation of retracted conclusions.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-depth-cycle-guard-returns-zero
_get_depth sets memo[node_id] = 0 before recursing into antecedents, so circular justification chains terminate with depth 0 rather than infinite recursion.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-validate-rejects-missing-antecedents
validate_proposals rejects proposals whose antecedent or outlist references do not exist in the current network.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-parse-returns-empty-list-on-no-match
parse_proposals silently returns an empty list when the LLM response matches neither the new nor old proposal format.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-allocation-is-proportional-by-agent
When agent namespaces are detected, _build_beliefs_section allocates the belief budget proportionally by agent size.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-build-prompt-default-budget-is-300
build_prompt has a default budget of 300 beliefs.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-write-proposals-file-roundtrips-through-parse
write_proposals_file serializes proposals to a markdown file in the same format parse_proposals() can read back, enabling human review before reasons accept.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-creates-active-inactive-kill-switch-pair
_ensure_agent_nodes() creates a premise node agent:active and an SL-justified node agent:inactive with outlist=[agent:active], so retracting agent:active flips agent:inactive IN and cascades all imported beliefs OUT.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-inactive-in-every-outlist
_build_justifications() unconditionally injects agent:inactive into every imported belief's outlist, making bulk distrust work through TMS propagation.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-nogoods-require-at-least-two-valid-nodes
_import_nogoods() filters out nogoods that reference fewer than 2 nodes existing in the network.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-retracts-out-beliefs-after-recompute
_import_claims() calls recompute_all() to propagate truth values and then explicitly retracts nodes that were OUT in the source.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-prefixes-ids-with-agent-name
All imported node IDs are prefixed with agent_name: (e.g., rhel-expert:kernel-modules-loaded) to prevent collisions between agents and with local beliefs.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-beliefs-topo-sort-is-cycle-tolerant
The import_beliefs topological sort uses a multi-pass Kahn-like algorithm that detects zero progress and appends remaining claims without raising an error when circular dependencies exist.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-beliefs-adds-out-nodes-before-retracting
STALE and OUT claims are first added to the network with their full justification structure and then retracted, preserving the dependency graph for possible future restoration.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-beliefs-skips-existing-node-ids
If a claim ID already exists in network.nodes, import_beliefs silently skips it with no upsert or merge behavior.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-beliefs-drops-nogoods-with-fewer-than-two-valid-nodes
Nogoods are imported only when at least 2 of their referenced nodes exist in the network; otherwise they are silently dropped.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-beliefs-parses-repos-until-next-header
parse_repos extracts the ## Repos section into a name-to-path mapping and stops parsing at the next ## header.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-beliefs-first-free-line-is-claim-text
parse_beliefs treats the first non-metadata, non-empty line after a claim header as the claim's human-readable text.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-import_beliefs.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-challenge-converts-premise-to-justified-node
When Network.challenge targets a premise, it converts that target into a justified node with an SL justification containing the challenge in its outlist.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-defend-implemented-via-challenge
Network.defend is implemented by calling self.challenge on the challenge node, creating a premise in the challenge's outlist.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-explain-recurses-through-valid-justification
Network.explain recursively traces why a node is IN by finding its valid justification and recursing into antecedents, and why a node is OUT by reporting failed antecedents and violated outlist nodes.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-trace-assumptions-returns-leaf-premises
Network.trace_assumptions walks backward through justification chains to return all leaf premise nodes with no justifications.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-schema-contains-five-tables
PgApi defines five project-partitioned tables: rms_nodes, rms_justifications, rms_nogoods, rms_propagation_log, and rms_network_meta.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-constructor-accepts-connection-string-or-psycopg-connection
The PgApi constructor accepts either a PostgreSQL connection string or an existing psycopg connection, plus a project_id.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-list-gated-finds-out-nodes-flippable-by-outlist-blockers
PgApi.list_gated finds OUT nodes that would flip IN if their outlist blockers went OUT, implementing the GATE pattern.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-add-node-inherits-access-tags-from-antecedents
PgApi.add_node inherits access tags from a node's antecedents.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-add-node-returns-node-type-and-premise-count
PgApi.add_node returns the node type (premise vs. derived) and the current premise count.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-jsonb-fields-checked-for-string-before-json-loads
PgApi defensively checks every metadata, antecedents, and outlist field with isinstance(x, str) before calling json.loads, because psycopg may or may not auto-deserialize JSONB.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-load-rebuilds-dependents-index
Storage.load assigns nodes directly into network.nodes and then calls network._rebuild_dependents() to reconstruct the reverse dependents index.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-save-persists-propagation-log-and-repos
Storage.save persists the propagation_log and repos tables in addition to nodes, justifications, nogoods, and network_meta.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-load-does-not-validate-json
Storage.load performs no validation on loaded JSON columns; malformed JSON raises json.JSONDecodeError and propagates uncaught.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-init-schema-is-idempotent
Storage._init_schema uses CREATE TABLE IF NOT EXISTS and CREATE VIRTUAL TABLE IF NOT EXISTS, making schema initialization safe to run against an existing database.
- Source: /home/forge/workspace/entries/2026/04/29/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-what-if-tracks-cascade-depth
PgApi what_if_retract and what_if_assert annotate cascade effects with BFS depth by following both antecedents and outlist via _find_dependents.
- Source: /home/forge/workspace/entries/2026/04/29/session-summary.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-propagate-delegates-to-api-propagate
cmd_propagate delegates to api.propagate() using the _with_network pattern, removing the only CLI handler that bypassed the API layer.
- Source: /home/forge/workspace/entries/2026/04/29/session-summary.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pgapi-coverage-gaps-decomposed-into-eight-sub-issues
23 missing PgApi functions were catalogued and filed as 8 focused sub-issues numbered #63 through #70 by category.
- Source: /home/forge/workspace/entries/2026/04/29/session-summary.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] code-expert-update-produced-242-new-beliefs
A code-expert update run produced 242 new beliefs, bringing the total to 542, with 217 derived beliefs.
- Source: /home/forge/workspace/entries/2026/04/29/session-summary.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] project-version-is-0-22-0
The project version recorded in the session summary is 0.22.0.
- Source: /home/forge/workspace/entries/2026/04/29/session-summary.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pytest-configure-registers-pg-marker
pytest_configure registers the `pg` marker at collection time so `@pytest.mark.pg` does not trigger an unknown-marker warning.
- Source: /home/forge/workspace/entries/2026/04/29/tests-conftest.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-api-fixture-initializes-db
The `pg_api` fixture calls `init_db()` on the UUID-scoped `PgApi` instance before yielding it to the test.
- Source: /home/forge/workspace/entries/2026/04/29/tests-conftest.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-cleanup-deletes-in-dependency-order
The `pg_api` teardown deletes rows across the five tracked tables in dependency order, with `rms_propagation_log` first and `rms_nodes` last.
- Source: /home/forge/workspace/entries/2026/04/29/tests-conftest.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] trace-access-tags-raises-keyerror-for-missing-node
`Network.trace_access_tags()` raises `KeyError` for nonexistent node IDs.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derived-node-merges-explicit-and-inherited-access-tags
A derived node's effective `access_tags` merge any explicit tags declared on the node with the inherited tags from its antecedents.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] access-tags-survive-sqlite-roundtrip
`access_tags` survive write-to-SQLite and read-back through `api.add_node` and `api.show_node`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-status-counts-only-visible-nodes
`get_status` summary counts include only nodes visible under the supplied `visible_to` filter.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_access_tags.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-idempotent-operations-return-empty-changed
Retracting an already-OUT node or asserting an already-IN node returns `changed == []`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-get-belief-set-returns-only-in-node-ids
`api.get_belief_set()` returns only the IDs of nodes whose truth value is IN.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-get-log-supports-last-n-truncation
`api.get_log(last=N)` returns at most the last N audit-log entries.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] build-ask-prompt-includes-search-beliefs-tool
`build_ask_prompt()` always includes the `search_beliefs` tool definition so the LLM knows it can request additional searches.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] build-ask-prompt-appends-tool-history-section
When `tool_history` is provided, `build_ask_prompt()` appends an 'Additional search results' section containing prior queries and their results.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-unknown-tool-returns-raw-response
When the LLM emits an unrecognized tool name, `ask()` returns the raw LLM response without crashing.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-no-synth-returns-no-results-when-no-fts5-matches
In `no_synth=True` mode, `ask()` returns the string 'No results' when FTS5 finds no matches.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-reports-content-changed-with-old-new-hashes
When a source file changes, `check_stale()` reports a result with `reason='content_changed'` and includes the old and new hashes.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-file-uses-sha256
`hash_file()` computes a SHA-256 hash of the file's contents.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-sources-mutates-network-in-place
`hash_sources()` writes computed hashes directly into `net.nodes[id].source_hash` for nodes it backfills.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-requires-source-hash
`check_stale()` skips IN nodes that have a `source` but no non-empty `source_hash`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-requires-nonempty-source-and-source-hash
`check_stale` skips nodes that lack `source` metadata, lack `source_hash` metadata, or have an empty-string `source`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-empty-file-with-matching-hash-is-fresh
An existing empty source file whose hash matches the node's `source_hash` is treated as fresh and produces no staleness result.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-content-changed-populates-source-path
A `content_changed` result from `check_stale` includes the resolved filesystem path in its `source_path` field.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-repos-none-still-detects-source-deleted
When `check_stale` is called with `repos=None`, a missing source file still produces a `source_deleted` result.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] run-cli-builds-argv-and-captures-streams
The `run_cli` test helper builds a synthetic `sys.argv` starting with "reasons" (plus optional `--db`), patches stdout/stderr with `StringIO`, calls `main()`, and returns `(stdout, stderr, exit_code)`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-tests-treat-human-readable-output-as-api
CLI tests assert on human-readable output substrings such as `Added a [IN]`, `Went OUT`, and `already IN`, treating CLI output copy as part of the tested API.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-idempotent-retract-assert-return-zero
`retract` on an already-OUT node and `assert` on an already-IN node both return exit code 0 with informational messages rather than an error.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-what-if-does-not-persist-changes
CLI `what-if` tests verify that cascade simulation runs without persisting changes and that output contains `NOT modified`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-tests-set-up-networks-via-chained-cli-calls
CLI tests build belief networks by sequentially calling `run_cli("init", ...)`, `run_cli("add", ...)`, and similar commands rather than constructing synthetic database states.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-nogoods-render-under-heading-with-auto-ids
`compact()` includes nogood contradiction sets under a `## Nogoods` section with auto-generated IDs.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-truncates-long-text-only-when-truncate-true
`compact()` ellipsis-truncates long belief text when `truncate=True` and preserves it when `truncate=False`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-empty-network-reports-zero-nodes
`compact()` on an empty network produces a valid summary containing `0 nodes tracked`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-shows-antecedents-and-dependent-counts
The compact output shows derived-node antecedents as `<- <id>` and displays dependent counts such as `(2 dependents)` when present.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] compact-budget-omits-entries-with-more-omitted-message
Each compact output section independently respects the token budget and emits a `more … omitted` message when entries are dropped.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_compact.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dangling-dependent-warning-once-per-encounter
Every dangling dependent reference encountered during propagation produces exactly one warning entry in `net.log`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_dangling_dependents.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dangling-guard-covers-challenge-and-add-justification
The dangling-dependent propagation guard also fires when propagation is triggered through `challenge()` and `add_justification()`, not only through `retract()` and `assert_node()`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_dangling_dependents.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dangling-warning-includes-timestamp
Dangling-dependent warning entries in `net.log` include a `timestamp` field.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_dangling_dependents.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dangling-dependents-injected-by-direct-set-mutation
Dangling-dependent tests inject ghost IDs by directly mutating a node's `dependents` set, such as `net.nodes["a"].dependents.add("ghost")`, rather than through the public API.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_dangling_dependents.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-ceiling-invariant
`_build_beliefs_section` enforces that the combined count of agent-shown and local-shown beliefs never exceeds the configured `max_beliefs` budget.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-local-remainder-formula
When enough local beliefs exist, `_build_beliefs_section` sets `local_shown` to `max(5, max_beliefs - agent_total)`.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-agent-match-uses-startswith
`_build_beliefs_section` identifies agent beliefs by checking whether node IDs start with the agent name followed by a colon.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-sampling-is-seed-deterministic
In sample mode, `_build_beliefs_section` produces identical output across calls when given the same random seed.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-missing-node-operations-raise-keyerror
`Network.retract`, `assert_node`, `explain`, and `add_nogood` raise `KeyError` with message 'not found' when referencing a nonexistent node ID.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-duplicate-node-add-raises-valueerror
`Network.add_node` raises `ValueError` with message 'already exists' when called with a duplicate node ID.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-multiple-justifications-survive-single-antecedent-retraction
A node with multiple justifications remains IN when only one justification's antecedent is retracted.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-idempotent-retract-assert-return-empty-list
Retracting an already-OUT node or asserting an already-IN node returns an empty change list.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-pre-fix-ids-used-len-plus-one
Before the issue #26 fix, nogood IDs were derived from `len(self.nogoods) + 1`, which allowed ID reuse after deletion.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_nogood_id.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] nogood-old-db-without-meta-defaults-counter-to-one
When loading a pre-fix database with no `network_meta` table and no existing nogoods, `_next_nogood_id` defaults to 1.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_nogood_id.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-api-duplicate-add-node-raises-generic-exception
`PgApi.add_node` with an existing node ID raises a generic `Exception` rather than a specific subclass.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-api-missing-node-operations-raise-keyerror
`PgApi.retract_node`, `show_node`, `what_if_retract`, `what_if_assert`, `challenge`, `defend`, `add_justification`, and `add_nogood` raise `KeyError` for nonexistent node IDs.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-api-defend-id-collision-raises-valueerror
`PgApi.defend` raises `ValueError` when the provided `defense_id` collides with an existing node.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-api-idempotent-retract-assert-return-empty-changed
`PgApi.retract_node` on an already-OUT node and `assert_node` on an already-IN node return an empty changed list rather than raising.
- Source: /home/forge/workspace/entries/2026/04/29/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] apply-proposals-requires-prior-validation
`apply_proposals` trusts its input unconditionally; callers must run `validate_proposals` first to avoid database errors from missing antecedents or duplicate IDs.
- Source: /home/forge/workspace/entries/2026/04/29/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-negative-two-stage-classification
`list_negative` uses keyword pre-filtering against a hardcoded `NEGATIVE_TERMS` list of about 50 words, then LLM classification via `ask._invoke_claude` to eliminate false positives.
- Source: /home/forge/workspace/entries/2026/04/29/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] test-pg-extra-manually-lists-deps
The `test-pg` extra in `pyproject.toml` manually lists `pytest`, `pytest-cov`, and `psycopg[binary]>=3.1` instead of composing the `test` and `pg` extras, requiring updates in two places when test dependencies change.
- Source: /home/forge/workspace/entries/2026/05/05/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] what-if-functions-never-persist
The `what_if_retract` and `what_if_assert` API functions are read-only simulations that load the network, apply the operation in memory, and pass `write=False` to `_with_network`, so no changes are persisted to the database.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-dual-mode-requires-sources-db
In ask.py, dual mode requires a sources_db value and raises ValueError when dual=True is used without one.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-natural-flag-strips-belief-metadata
When natural=True, ask.py uses the natural citation rule without belief IDs and calls _strip_belief_metadata to remove status, justification, and dependency lines from belief context.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-search-source-chunks-uses-stopword-filtered-or-query
_search_source_chunks tokenizes the question, filters stop words, runs an OR query against the rag_fts.db SQLite FTS5 index, and returns markdown with filename and section headers.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-tool-loop-status-messages-use-stderr
Status messages in ask.py are written to stderr so they do not pollute the answer written to stdout.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-source-path-priority-chain
resolve_source_path resolves relative source strings by checking the agent repo, then db_dir, then the repos dict, then ~/git fallback, then the bare path.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-file-returns-full-64-char-sha256
hash_file returns the full 64-character SHA-256 hex digest and does not catch errors for unreadable files.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] hash-sources-force-rehashes-all-source-nodes
hash_sources with force=True re-hashes all source-bearing nodes regardless of whether they already have a source_hash.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-upgrade-hashes-excludes-upgraded-truncated
When upgrade_hashes=True, check_stale silently upgrades matching truncated hashes in place on the node and excludes them from stale results.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-db-option-defaults-to-api-default-db
Every CLI handler passes args.db through, and the --db option defaults to api.DEFAULT_DB, making the database path user-overridable on every command.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-parse-visible-to-returns-none-when-absent
_parse_visible_to returns None rather than an empty list when --visible-to is absent, and the API layer interprets None as no access control.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-derive-exhaust-implies-auto
cmd_derive's --exhaust mode implies --auto because _derive_one_round checks args.auto or args.exhaust before auto-applying proposals.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-output-is-plain-text-not-json
CLI command handlers format returned result dicts as plain text for terminal output rather than JSON.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-init-catches-fileexistserror-suggests-force
cmd_init catches FileExistsError and suggests using --force.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-creates-two-infra-nodes-per-agent
_ensure_agent_nodes creates two infrastructure nodes per agent: agent:active as a bare premise and agent:inactive with an SL justification whose outlist contains agent:active.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-prefixes-ids-at-justification-build-stage
Imported belief IDs get the agent_name prefix during _build_justifications, while raw justifications carry unprefixed IDs.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-sync-marks-removed-nodes-retracted-metadata
In sync, local-only agent beliefs not present in the remote set are retracted or marked _retracted in metadata, excluding infrastructure nodes.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-dual-format-normalization-shared-claim-schema
_normalize_markdown and _normalize_json both produce a common claim dict with id, text, is_out, source, source_hash, date, metadata, and raw_justifications fields.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-model-resolution-supports-four-syntax-forms
resolve_model_cmd supports exact MODEL_COMMANDS names, claude:<submodel>, gemini:<submodel>, and ollama:<model> syntax forms.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-invoke-default-timeout-300-seconds
invoke_model has a default timeout of 300 seconds for LLM subprocess calls.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-checks-binary-exists-before-spawn
invoke_model uses shutil.which to check that the resolved model binary exists in PATH before spawning the subprocess, raising FileNotFoundError if missing.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-nonzero-exit-raises-runtimeerror-with-stderr
A non-zero exit from an LLM subprocess causes invoke_model to raise RuntimeError containing the subprocess stderr output.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-bfs-propagation-uses-deque-and-visited-set
`_propagate()` uses a `deque`-based BFS through the dependents reverse index and a `visited` set to prevent cycles.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-recompute-all-bound-by-node-count-plus-one
`recompute_all()` iterates to a fixpoint over all derived nodes and is bounded by `len(self.nodes) + 1` iterations.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-add-node-raises-valueerror-on-duplicate-id
`add_node()` raises `ValueError` when given an already-existing node ID.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-entrenchment-ranks-premises-above-derived-and-sourced-above-unsourced
`_entrenchment()` scores premises higher than derived nodes, sourced nodes higher than unsourced nodes, and nodes with more dependents higher than nodes with fewer dependents.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-propagate-warns-on-dangling-dependents-without-raising
`_propagate()` logs a warning for a dangling dependent ID that is absent from `self.nodes` but does not raise an exception.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-verify-dependents-detects-index-drift
`verify_dependents()` recomputes the dependents index from justifications and diffs it against the live index to detect drift.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-what-if-rolls-back-unconditionally
`what_if_retract` and `what_if_assert` mutate rows inside a transaction, collect cascade results, and then unconditionally call `self.conn.rollback()` in a `finally` block so simulations are side-effect-free.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-schema-constrains-truth-values-and-justification-types
The PostgreSQL schema constrains `truth_value` to `'IN'` or `'OUT'` and justification `type` to `'SL'` or `'CP'` via CHECK constraints.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-search-combines-full-text-and-ilike-fallback
`PgApi.search` uses PostgreSQL `to_tsvector`/`plainto_tsquery` full-text search plus an ILIKE fallback on node IDs and includes 1-hop neighbor expansion.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-list-gated-finds-out-nodes-by-outlist-blockers
`list_gated` finds OUT nodes that would flip IN if their outlist blocker nodes were retracted (GATE pattern).
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-missing-psycopg-raises-importerror-with-install-instructions
`PgApi.__init__` calls `_require_psycopg()`, which raises `ImportError` with install instructions when the `psycopg` driver is unavailable.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-add-justification-requires-sl-or-cp
`PgApi.add_justification` raises `ValueError` when both the `sl` and `cp` justification inputs are empty.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-batch-size-constant-is-20
`REVIEW_BATCH_SIZE` is set to 20, so `review_beliefs` sends at most 20 beliefs per LLM call by default.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-format-returns-empty-string-for-missing-node
`format_belief_for_review` returns an empty string when the requested node ID is not found in the `nodes` dict.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-parser-scans-for-json-array-with-raw-decode
`parse_review_response` uses `json.JSONDecoder.raw_decode` scanning from each `[` character to extract a JSON array from LLM output that may contain surrounding prose.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-renders-missing-reference-as-not-found-in-network
`format_belief_for_review` renders `(not found in network)` for antecedent or outlist entries missing from the `nodes` dict.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-results-returned-in-batch-order
`review_beliefs` returns results in batch order, not necessarily matching input order within a batch.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-save-deletes-and-reinserts-all-rows
`Storage.save()` deletes all rows from every table and re-inserts the full network state in a single SQLite transaction, with no incremental/diff-based persistence.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-fts-table-rebuilt-on-every-save
The `nodes_fts` FTS5 virtual table is dropped and recreated on every `Storage.save()` rather than incrementally updated.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-load-trusts-stored-state-without-validation
`Storage.load()` does not validate that antecedent node IDs exist, truth values are valid, or justification types are recognized; it trusts the database contents written by `save()`.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-fts-uses-porter-stemming
The `nodes_fts` FTS5 index uses Porter stemming on node ID and text.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] storage-no-connection-pooling-or-retry
`Storage` uses one SQLite connection per instance and has no connection pooling or retry logic for `SQLITE_BUSY`.
- Source: /home/forge/workspace/entries/2026/05/05/reasons_lib-storage.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-reviewed-80-derived-beliefs-in-four-batches
The first live `review-beliefs` run reviewed 80 derived beliefs in 4 batches of 20, using 4 LLM calls total.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-65-of-80-beliefs-passed
In the first live review run, 65 of 80 reviewed derived beliefs passed the LLM review.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-9-of-80-beliefs-invalid
In the first live review run, 9 of 80 reviewed derived beliefs were marked invalid.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-8-retractions-cascaded-about-100-dependents
The first review run retracted 8 invalid beliefs, cascading approximately 100 dependent beliefs OUT and reducing the network from 812 IN to 710 IN.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-two-beliefs-strengthened-with-missing-antecedents
The first review run fixed 2 beliefs by adding missing antecedents that already existed in the network but were not wired into the justification.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-actionable-issue-rate-19-percent
The first review run found actionable issues in 19% of reviewed beliefs (15 of 80).
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-retraction-reduced-IN-by-12-5-percent
After the first review run's retraction cascades, the network's IN count shrank by 12.5%, from 812 to 710.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-practice-llm-call-takes-30-60-seconds
Each review LLM call took approximately 30-60 seconds in the first live run.
- Source: /home/forge/workspace/entries/2026/05/05/review-beliefs-in-practice.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-tests-use-fresh-sqlite-db-per-test
The API test suite uses a `db_path` pytest fixture that creates a fresh SQLite database in `tmp_path` for each test.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-functions-accept-db-path-string
Public functions in `reasons_lib.api` accept a `db_path` string keyword and open and close their own SQLite connection internally.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-retract-already-out-returns-empty-changed
Calling `api.retract_node()` on an already-OUT node returns `changed: []` without raising an error.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-assert-already-in-returns-empty-changed
Calling `api.assert_node()` on an already-IN node returns `changed: []` without raising an error.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] search-depth-expansion-adds-antecedents
`api.search()` depth expansion returns only direct hits at depth 0, adds direct antecedents at depth 1, and adds transitive antecedents at depth 2.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-gated-excludes-retracted-blockers
`api.list_gated()` excludes gated nodes whose blockers have been retracted.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-dual-performs-three-sequential-llm-calls
`ask(..., dual=True)` performs three sequential LLM calls: one for TMS beliefs, one for source documents, and one to merge the two answers.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-source-docs-persist-across-tool-rounds
In the agentic `ask` loop, source document content included in the prompt persists across tool-call rounds.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] strip-belief-metadata-is-idempotent-on-plain-text
`_strip_belief_metadata` leaves plain belief text unchanged and only removes markdown structural elements such as headers, status lines, and separators.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-returns-results-and-upgrade-count-tuple
`check_stale()` returns a tuple `(results, upgraded_count)` where `results` is a list of result dicts and `upgraded_count` counts truncated hashes upgraded in place.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_check_stale.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-existing-empty-file-is-not-deleted
An existing zero-byte source file is not reported as `source_deleted`; it is fresh if its hash matches and `content_changed` if the hash differs.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] check-stale-repos-none-still-detects-missing-files
`check_stale()` with `repos=None` still returns `source_deleted` results for missing source files instead of raising an exception.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_check_stale_issue25.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-registers-source-repo
`import_agent` registers the imported agent's source directory in the repos list, exposed via `api.list_repos()`.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-imports-companion-nogoods
`import_agent` imports nogoods from a companion `nogoods.md` file alongside the agent's beliefs.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-only-in-excludes-stale-out-beliefs
With `only_in=True`, `import_agent` excludes STALE/OUT beliefs entirely so that `api.show_node` on an excluded belief raises `KeyError`.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-model-cmd-claude-bare-argv
resolve_model_cmd('claude') returns the argv list ['claude', '-p'].
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-model-cmd-claude-submodel-argv
resolve_model_cmd('claude:sonnet') returns the argv list ['claude', '-p', '--model', 'sonnet'].
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-model-cmd-gemini-bare-argv
resolve_model_cmd('gemini') returns the argv list ['gemini', '--skip-trust', '-p', ''].
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-model-cmd-gemini-submodel-argv
resolve_model_cmd('gemini:flash') returns the argv list ['gemini', '--skip-trust', '-m', 'flash', '-p', ''].
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-model-cmd-ollama-argv
resolve_model_cmd('ollama:gemma3:4b') returns the argv list ['ollama', 'run', 'gemma3:4b'].
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] resolve-model-cmd-unknown-prefix-raises-valueerror
resolve_model_cmd raises ValueError for an unknown model prefix such as 'gpt-4'.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] invoke-model-forwards-timeout
invoke_model forwards its timeout argument to subprocess.run.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-thinking-strip-requires-both-markers
invoke_model strips Ollama thinking output only when both 'Thinking...' and '...done thinking.' markers are present; partial markers leave output unchanged.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_llm.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-derived-node-inherits-union-of-antecedent-access-tags
Derived nodes in PgApi inherit the union of their antecedents' access tags.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-what-if-operations-are-read-only
PgApi what_if_retract() and what_if_assert() report cascades without mutating database state.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-challenge-creates-outlist-entry-forcing-target-out
A PgApi challenge creates an outlist entry that forces the target node OUT.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-defend-creates-outlist-entry-restoring-target
A PgApi defense creates an outlist entry on the challenge node, neutralizing the challenge and restoring the target.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-assert-restores-dependents-except-independently-retracted
Asserting a PgApi node restores transitive dependents whose justifications are satisfied but does not restore nodes that were independently retracted.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-list-gated-excludes-satisfied-gates
PgApi list_gated excludes satisfied gates, where a gate is satisfied when its blocker is OUT.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-multi-tenancy-allows-node-id-reuse-across-projects
Two PgApi instances with different project_id values sharing one database can reuse the same node ID independently.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-retract-already-out-returns-empty-changed
Retracting an already-OUT node in PgApi returns an empty changed list.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-parse-drops-items-without-id
parse_review_response silently drops response entries that lack an id field.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-parse-extracts-json-array-from-prose
parse_review_response extracts a JSON array from LLM output even when it is surrounded by prose.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-batches-beliefs-by-review-batch-size
review_beliefs batches derived beliefs into groups of REVIEW_BATCH_SIZE before sending them to the LLM.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-cli-displays-invalid-insufficient-unnecessary-flags
The review-beliefs CLI displays INVALID, INSUFFICIENT, and UNNECESSARY(...) flags for each finding.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-cli-auto-retract-retracts-invalid-beliefs
The review-beliefs CLI with --auto-retract retracts beliefs marked invalid unless --dry-run is active.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-cli-output-file-writes-markdown-findings
The review-beliefs CLI with -o writes a Markdown findings report.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-format-numbers-only-multiple-justifications
format_belief_for_review numbers justification headers only when a belief has multiple justifications.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-adds-new-remote-beliefs
sync_agent adds new beliefs present in the remote file to the local database with correct metadata.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-retracts-remote-missing-beliefs
sync_agent retracts local beliefs that are missing from the remote file by setting them OUT.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-updates-changed-description-text
sync_agent updates the local node description when the remote file changes a belief's text.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-rewires-changed-dependencies
sync_agent rewires justification antecedents when the remote file changes a belief's Depends on list.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-first-run-behaves-as-initial-import
Calling sync_agent for a never-imported agent behaves as an initial import.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-truth-value-changes-cascade-to-dependents
When sync_agent changes a belief's truth value, the change cascades to dependent nodes.
- Source: /home/forge/workspace/entries/2026/05/05/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] business-planning-documents-live-in-google-drive
Business planning source documents are stored in Google Drive rather than on local disk.
- Source: /home/forge/workspace/entries/2026/05/06/belief-staleness-in-business-planning.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] business-planning-lacks-hash-based-change-detection
The business planning network has no hash-based source change detection equivalent to check-stale for local files.
- Source: /home/forge/workspace/entries/2026/05/06/belief-staleness-in-business-planning.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] stale-premise-can-support-dozens-of-derived-conclusions
A single stale premise can support dozens of derived conclusions in the belief network.
- Source: /home/forge/workspace/entries/2026/05/06/belief-staleness-in-business-planning.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] business-premise-verification-can-take-hours-to-weeks
Verifying a business premise can take hours to weeks per belief.
- Source: /home/forge/workspace/entries/2026/05/06/belief-staleness-in-business-planning.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-first-run-checked-526-in-beliefs
The first live run of reasons contradictions checked all 526 IN beliefs in the ftl-reasons-expert network.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-first-run-found-8-contradictions
The first live run of reasons contradictions found 8 contradictions.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-first-run-retracted-4-beliefs
The first contradictions run retracted 4 beliefs directly.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-medium-severity-findings-filed-issues-100-101-102
The 4 medium-severity contradictions in the first run were filed as issues #100, #101, and #102.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] reasons-update-command-implemented-pr-104
The reasons update command was implemented in PR #104 during the first contradictions triage session.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-batches-50-beliefs-review-beliefs-batches-20
The contradictions command batches 50 beliefs per LLM call, while review-beliefs batches 20 beliefs with antecedent context.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-scope-all-in-beliefs-review-scope-derived-only
The contradictions command checks all IN beliefs, whereas review-beliefs checks only derived beliefs.
- Source: /home/forge/workspace/entries/2026/05/06/contradictions-first-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-beliefs-first-pass-targeted-448-derived-beliefs
The first full review-beliefs pass targeted all 448 derived beliefs in the ftl-reasons-expert network and reviewed approximately 420 of them.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-beliefs-first-pass-retracted-44-invalid-beliefs
The first full review-beliefs pass retracted 44 invalid derived beliefs.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-beliefs-first-pass-cascade-amplification-was-6-5x
Retracting 44 beliefs in the first review pass caused 286 additional retractions, a 6.5x cascade amplification factor.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] agent-beliefs-retraction-cascaded-to-52-dependents
Retracting agent-beliefs-undergo-full-revision caused a cascade of 52 dependent beliefs.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-beliefs-batches-20-beliefs-with-600s-timeout
review-beliefs evaluates batches of 20 beliefs and required a 600s timeout after two 300s batch failures.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-found-about-20-unnecessary-antecedents
The first review pass found about 20 beliefs with unnecessary antecedents.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-beliefs-command-introduced-pr-97-v0-33-0
The review-beliefs command was introduced in PR #97 as v0.33.0.
- Source: /home/forge/workspace/entries/2026/05/06/full-review-beliefs-results.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-lacks-document-retrieval
ftl-reasons has no built-in document retrieval or source material navigation.
- Source: /home/forge/workspace/entries/2026/05/06/pageindex-and-tms-integration.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pageindex-tms-integration-would-add-section-level-provenance
Integrating PageIndex with ftl-reasons would let beliefs carry section-level provenance such as document UUID, section path, and page range.
- Source: /home/forge/workspace/entries/2026/05/06/pageindex-and-tms-integration.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pageindex-can-automate-staleness-verification-retrieval
PageIndex could automate retrieving the current source section for staleness verification before an LLM checks whether it still supports a belief.
- Source: /home/forge/workspace/entries/2026/05/06/pageindex-and-tms-integration.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pageindex-tree-diff-can-flag-changed-section-beliefs
A section-level diff between old and new PageIndex trees could flag beliefs sourced from changed sections as potentially stale.
- Source: /home/forge/workspace/entries/2026/05/06/pageindex-and-tms-integration.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pageindex-can-ground-derived-beliefs-in-source-evidence
PageIndex could search indexed documents for supporting or contradicting evidence and attach source sections to derived beliefs.
- Source: /home/forge/workspace/entries/2026/05/06/pageindex-and-tms-integration.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pageindex-tms-integration-requires-pageindex-api-access
The proposed PageIndex integration requires PageIndex workspace sharing or API access from other tools.
- Source: /home/forge/workspace/entries/2026/05/06/pageindex-and-tms-integration.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-build-backend-is-setuptools
pyproject.toml declares setuptools >=68.0 as the PEP 517 build backend with build-backend set to setuptools.build_meta.
- Source: /home/forge/workspace/entries/2026/05/08/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-version-is-static-0-38-0
The ftl-reasons package version is statically declared as 0.38.0 in pyproject.toml and bumped manually.
- Source: /home/forge/workspace/entries/2026/05/08/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-extra-requires-psycopg-binary-3-1
The pg optional dependency extra requires psycopg[binary] >=3.1.
- Source: /home/forge/workspace/entries/2026/05/08/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-extra-requires-sentence-transformers-2-2-and-scikit-learn-1-2
The cluster optional dependency extra requires sentence-transformers >=2.2 and scikit-learn >=1.2.
- Source: /home/forge/workspace/entries/2026/05/08/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pytest-testpaths-restricted-to-tests-directory
pytest is configured with testpaths = ["tests"], restricting collection to the tests/ directory.
- Source: /home/forge/workspace/entries/2026/05/08/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] uv-lock-generated-from-pyproject-dependency-declarations
The uv.lock file is generated from pyproject.toml's dependency declarations.
- Source: /home/forge/workspace/entries/2026/05/08/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-with-network-is-class-context-manager
`_with_network` is implemented as a class with `__enter__` and `__exit__` rather than using `@contextmanager`, and it saves the network on clean exit only when `write=True`.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] what-if-operations-never-save
`what_if_retract` and `what_if_assert` load the network, perform the operation in memory, diff truth values, and never save to SQLite.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] namespace-active-premise-gates-namespace
`ensure_namespace` creates a `<ns>:active` premise node that all namespaced beliefs depend on, so retracting that single premise cascades the entire namespace OUT.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-uses-remote-wins
`sync_agent` imports another agent's beliefs with remote-wins merge semantics.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] search-expands-neighbors-via-justification-bfs
`search` performs neighbor expansion by BFS along justification edges after the FTS5 query.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-json-topologically-orders-nodes
`import_json` uses topological sort to add nodes in dependency order, with a fallback pass for cycles.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-gated-finds-out-nodes-blocked-by-in-outlist
`list_gated` finds OUT nodes that are blocked by IN outlist nodes.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] deduplicate-clusters-by-id-token-jaccard
`deduplicate` uses Jaccard similarity on ID tokens to cluster similar beliefs and rewires dependents before retracting duplicates.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-db-flag-available-everywhere
`--db` is added to the top-level parser and is available to every subcommand, defaulting to `api.DEFAULT_DB`.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-exception-types-map-to-failure-modes
The CLI maps `KeyError` to node not found, `ValueError` to invalid input, `FileNotFoundError` to missing file, and `PermissionError` to access-tag violation, exiting with code 1.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cmd-update-prevalidates-one-field
`cmd_update` validates that at least one of `--text`, `--source`, or `--source-url` is provided before calling the API.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-no-op-cases-exit-zero
No-op cases such as already OUT, already IN, no nodes, and no matching nodes print informative messages and return exit code 0 rather than raising errors.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-kmeans-uses-n-init-10
`cluster_beliefs` runs KMeans with `n_init=10` and the user's seed for reproducibility.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-k-clamped-to-valid-range
The final cluster count `k` is clamped to `[1, len(beliefs)]` so KMeans never receives an invalid cluster count.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-beliefs-returns-selected-ids-and-stats
`cluster_beliefs` returns `(selected_ids, cluster_stats)`, where `cluster_stats` includes cluster count, sizes, and model name.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-ml-errors-propagate
Beyond the optional-dependency `ImportError`, embedding and clustering errors in `cluster.py` are not caught and propagate to the caller.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-default-batch-size-50
`CONTRADICTION_BATCH_SIZE` defaults to 50 beliefs per LLM call.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-prompt-defines-four-types
`CONTRADICTION_PROMPT` defines four contradiction types: direct negation, incompatible properties, scope conflict, and quantitative mismatch.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-format-truncates-text-200
`format_beliefs_for_contradiction_check` truncates each belief's text at 200 characters and skips IDs that do not exist in the nodes mapping.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-parser-extracts-nogood-blocks
`parse_contradiction_response` uses `_NOGOOD_PATTERN` to find `### NOGOOD <id>` blocks and line-parses Claims, Analysis, and Severity fields.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-prompt-excludes-tensions
The contradiction prompt instructs the LLM not to flag tensions or tradeoffs, only genuine logical incompatibilities.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-build-prompt-returns-prompt-and-stats
`build_prompt` returns `(prompt_text, stats_dict)`, where `stats` captures prompt inputs for audit trails.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-depth-premises-zero-derived-one-plus-max
`_get_depth` assigns premises depth 0 and derived nodes depth 1 plus the maximum antecedent depth, using memoization.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-validate-requires-existing-antecedents-and-unless
`validate_proposals` rejects a proposal if any antecedent or unless node does not exist in the network.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-proposals-file-roundtrips-through-parser
`write_proposals_file` serializes valid proposals to the same `### DERIVE` and `### GATE` markdown format that `parse_proposals` can re-read for human review.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-topic-filter-ors-keywords
`_filter_by_topic` treats space-separated keywords as OR conditions so any keyword match passes.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-default-budget-groups-by-kebab-prefix
In default budget mode, `_build_beliefs_section` groups beliefs by kebab-case ID prefix and outputs groups largest first, truncating at budget.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-cross-agent-task-injected-when-multi-agent
`CROSS_AGENT_TASK` is injected into the derive prompt only when multi-agent beliefs are detected.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-sample-uses-reservoir-sampling
`_sample_beliefs` uses reservoir-style random sampling to stay within the token budget.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-markdown-additive-skips-existing
`import_agent()` performs additive Markdown import and skips nodes whose IDs already exist.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-json-preserves-full-justifications
`import_agent_json()` imports JSON beliefs while preserving their full justification structure.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-raw-justifications-filtered-before-prefixing
During normalization, `raw_justifications` stores unprefixed IDs and filtering to the import set occurs before agent-name prefixing.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-resurrection-clears-retracted
When sync sees a previously OUT node become IN remotely, it clears `_retracted` metadata and calls `network.assert_node()` to resurrect the node.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-defers-assertions-until-after-dependents
Sync collects OUT-to-IN transitions in `assert_after` and asserts them after `_fixup_dependents()` so dependencies are rebuilt first.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-nogoods-prefixed-on-sync
`_import_nogoods()` adds prefixed nogood records during sync.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-batch-size-default-20
`REVIEW_BATCH_SIZE` defaults to 20 beliefs per LLM review call.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-prompt-requires-six-field-json
`REVIEW_PROMPT` instructs the LLM to return a JSON array with `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, and `comment` fields.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-json-parser-scans-for-array
`parse_review_response` scans for a JSON array with `json.JSONDecoder.raw_decode` and returns `[]` when no valid array is found.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-preserves-batch-order
`review_beliefs` preserves batch ordering so results appear in the same order as input belief IDs except for batches that fail entirely.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-format-returns-empty-for-missing-node
`format_belief_for_review` returns an empty string when the requested node does not exist.
- Source: /home/forge/workspace/entries/2026/05/08/reasons_lib-review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-init-requires-force-to-overwrite
`api.init_db()` refuses to overwrite an existing database unless `force=True` is passed.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-idempotent-operations-return-empty-changed
Retracting an already-OUT node or asserting an already-IN node is a no-op that returns an empty `changed` set.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-get-belief-set-excludes-out-nodes
`api.get_belief_set()` returns only IN node IDs and excludes OUT nodes.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] fts-single-character-only-query-returns-empty
FTS queries composed only of single-character words return empty results.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] api-tests-use-isolated-temp-sqlite
`tests/test_api.py` gives each test a fresh SQLite database created via `api.init_db()` in pytest `tmp_path`.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-add-supports-derived-nodes-with-sl-flag
The CLI `add` subcommand supports derived nodes through the `--sl` flag.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-what-if-outputs-not-modified
The CLI `what-if` command prints `NOT modified` and does not change network state.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-explain-shows-failed-antecedents-and-violated-outlists
The CLI `explain` command displays failed antecedents and violated outlists.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-update-errors-without-text-or-source
The CLI `update` command returns an error when neither a text nor a source flag is provided.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-init-requires-force-to-overwrite-db
The CLI `init` command refuses to overwrite an existing database unless `--force` is supplied.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-beliefs-honors-explicit-n-clusters
When `n_clusters` is passed to `cluster_beliefs`, the returned `stats['n_clusters']` matches the requested value exactly.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-stats-include-embedding-model
The `cluster_beliefs` stats dict includes an `embedding_model` field naming the sentence-transformer model used.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-missing-deps-error-mentions-extra
`_require_cluster_deps` raises an `ImportError` whose message mentions `ftl-reasons[cluster]` when clustering dependencies are missing.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-cache-embed-returns-ids-and-embeddings
`ClusterCache.embed(beliefs)` returns a tuple of belief IDs and their embeddings.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-tests-verify-cross-domain-diversity
`test_cluster_beliefs_cross_domain` creates auth and database belief groups and asserts both are represented in the selected output.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-contradiction-response-uses-nogood-markdown-format
parse_contradiction_response expects nogoods in markdown format with `### NOGOOD <id>` headers and `Claims:`, `Analysis:`, and `Severity:` fields.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradiction-batching-splits-in-beliefs-by-batch-size
detect_contradictions with 3 IN beliefs and batch_size=2 makes exactly 2 LLM calls.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradiction-timeout-parameter-passes-through-to-subprocess
The timeout parameter to detect_contradictions is passed through to the LLM subprocess call.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradiction-parser-returns-empty-on-malformed-output
If an LLM contradiction response lacks `### NOGOOD` headers, parse_contradiction_response returns an empty list.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-cli-output-writes-markdown-report
The `reasons contradictions --output` flag writes a markdown report file with proper headers.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradiction-api-auto-apply-records-nogoods
`api.detect_contradictions(auto_apply=True)` records detected contradictions by calling `add_nogood`.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-proposals-supports-v09-bold-field-format
parse_proposals can parse the older v0.9 proposal format that uses bold field names.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-proposals-parses-gate-blocks-with-unless-outlists
parse_proposals extracts GATE blocks that include `unless` outlists.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] deduplicate-clusters-jaccard-similar-belief-ids
api.deduplicate finds clusters of belief IDs that are Jaccard-similar.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dedup-plan-roundtrips-through-write-and-parse
api.write_dedup_plan output can be parsed by api.parse_dedup_plan with round-trip fidelity.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-proposals-file-roundtrips-through-parse-proposals
write_proposals_file output parses identically through parse_proposals.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] apply-dedup-plan-reports-missing-nodes-as-errors
api.apply_dedup_plan returns missing nodes as error strings in its result dict rather than raising exceptions.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-conserves-total-slots
`_build_beliefs_section` keeps `agent_shown + local_shown <= max_beliefs`.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-count-accumulates-across-agents
Agent belief count in `_build_beliefs_section` accumulates across all agents rather than resetting per agent.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_derive_budget.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-active-premise-has-role-agent-premise
import_agent creates the `{agent}:active` sentinel node as an IN premise with `role=agent_premise` metadata.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-stores-original-id-metadata
Every belief imported by import_agent carries `original_id` metadata.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-only-in-omits-out-beliefs
When import_agent is called with `only_in=True`, STALE or OUT beliefs from the source are skipped and do not exist in the database.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-reimport-returns-zero-imported
Re-importing the same agent and beliefs returns `claims_imported=0` and `claims_skipped=3`.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-registers-source-directory-in-repos
`api.list_repos()` includes an imported agent with its source directory path.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-imports-nogoods
import_agent imports nogood records from the source; the sample source produces one imported nogood.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-api-supports-depth-sample-dependency-visibility-filters
api.review_beliefs supports `min_depth`, `sample`, `depends_on`, and `visible_to` filtering.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-nodes-supports-review-metadata-filters
api.list_nodes supports `never_reviewed` and `not_reviewed_since` filters, `by_impact` sorting, and includes `last_reviewed` and `review_result` fields.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-cli-no-report-suppresses-report-directory
The `review-beliefs --no-report` flag suppresses creation of the report directory.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-cli-output-writes-findings-markdown
The `review-beliefs -o` flag writes findings to a markdown file.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-auto-retract-retracts-invalid-beliefs
The `review-beliefs --auto-retract` flag actually changes truth values by retracting invalid beliefs.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_review.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-namespaces-beliefs-under-agent-id
`sync_agent` stores remote beliefs under namespaced IDs of the form `agent:id` (e.g., `test-agent:alpha-fact`).
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-stale-remote-maps-to-out
Beliefs marked `[STALE]` in the remote markdown are synced as OUT nodes; `TestSyncNoChanges` expects `beliefs_retracted == 1` for `gamma-stale`.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-rewires-dependency-antecedents
When a remote belief's `Depends on:` line changes, `sync_agent` rewires the corresponding justification antecedents to match.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-registers-repo-directory
`sync_agent` records the agent's file directory in the repo registry, observable through `api.list_repos`.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-counts-justification-only-changes-as-updates
`sync_agent` increments the `beliefs_updated` counter for justification-only changes, not only for text or truth-value changes.
- Source: /home/forge/workspace/entries/2026/05/08/tests-test_sync_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-review-response-returns-empty-list-on-malformed-input
`parse_review_response` returns an empty list on malformed input — bad JSON, non-list JSON, or items missing `id` fields — rather than raising an exception.
- Source: /home/forge/workspace/entries/2026/05/08/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-review-response-defaults-missing-booleans-true
`parse_review_response` defaults missing `valid`, `sufficient`, and `necessary` boolean fields to `True`.
- Source: /home/forge/workspace/entries/2026/05/08/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-parse-error-never-generates-validity-warning
A review-response parse error never generates a false-positive validity warning because missing boolean fields default to `True`.
- Source: /home/forge/workspace/entries/2026/05/08/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-prompts-passed-via-stdin-not-argv
LLM prompts are passed to LLM subprocesses through `subprocess.run(input=...)` rather than as command-line arguments.
- Source: /home/forge/workspace/entries/2026/05/08/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-compatible-command-count-is-25
25 CLI commands dispatch to PostgreSQL through `PgApi`.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-export-network-filters-private-metadata
`PgApi.export_network` filters out private metadata keys that start with `_`.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-export-network-respects-visible-to
`PgApi.export_network(visible_to=...)` filters exported nodes by access tags using the `visible_to` argument.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-remove-justification-deletes-nth-by-serial-id
`PgApi.remove_justification(node_id, index)` deletes the Nth justification by serial ID order.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-remove-justification-prevents-zero-justifications-for-derived-node
`PgApi.remove_justification` refuses a removal that would leave a derived node with zero justifications.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-update-node-updates-only-provided-fields
`PgApi.update_node` performs a dynamic UPDATE that only sets provided `text`, `source`, and `source_url` fields.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-convert-to-premise-deletes-justifications-and-sets-in
`PgApi.convert_to_premise` deletes all justifications for a node and sets its truth value to IN.
- Source: /home/forge/workspace/entries/2026/05/10/postgresql-backend-support.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] semantic-contradictions-use-all-minilm-l6-v2
Semantic contradiction detection embeds beliefs using the sentence-transformers model `all-MiniLM-L6-v2`.
- Source: /home/forge/workspace/entries/2026/05/10/semantic-contradiction-detection.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] semantic-contradiction-reuses-existing-prompt-parse-pipeline
After clustering, semantic contradiction detection reuses the existing contradiction prompt/parse pipeline unchanged.
- Source: /home/forge/workspace/entries/2026/05/10/semantic-contradiction-detection.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] semantic-contradiction-first-run-used-575-in-beliefs-and-20-clusters
The first `--semantic` contradiction run processed 575 IN beliefs grouped into 20 clusters.
- Source: /home/forge/workspace/entries/2026/05/10/semantic-contradiction-detection.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] semantic-contradiction-55-belief-cluster-exceeded-300s-timeout
In the first `--semantic` contradiction run, cluster 1 with 55 beliefs exceeded the 300-second LLM timeout.
- Source: /home/forge/workspace/entries/2026/05/10/semantic-contradiction-detection.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-version-is-0-40-0
`pyproject.toml` declares the `ftl-reasons` package version as `0.40.0`.
- Source: /home/forge/workspace/entries/2026/05/11/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-build-backend-is-setuptools-build-meta
The project uses `setuptools>=68.0` as the build requirement and `setuptools.build_meta` as the build backend.
- Source: /home/forge/workspace/entries/2026/05/11/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-pg-extra-depends-on-psycopg-binary
The `[pg]` optional dependency extra requires `psycopg[binary]>=3.1`.
- Source: /home/forge/workspace/entries/2026/05/11/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-cluster-extra-depends-on-sentence-transformers-and-scikit-learn
The `[cluster]` optional dependency extra requires `sentence-transformers` and `scikit-learn`.
- Source: /home/forge/workspace/entries/2026/05/11/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-mcp-extra-depends-on-mcp-1-0
The `[mcp]` optional dependency extra requires `mcp>=1.0`.
- Source: /home/forge/workspace/entries/2026/05/11/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-pytest-testpaths-is-tests
`pyproject.toml` configures pytest with `testpaths = ["tests"]`.
- Source: /home/forge/workspace/entries/2026/05/11/pyproject.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] what-if-simulations-are-read-only-and-report-cascade-depth
`what_if_retract` and `what_if_assert` run the requested operation in memory without `write=True` and use `_cascade_depth()` BFS to report each affected node's distance from the trigger without persisting changes.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-gated-finds-out-nodes-blocked-by-in-outlist
`list_gated` finds OUT nodes that are blocked by IN outlist nodes, implementing the gate pattern where a belief flips IN when its blocker goes OUT.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-creates-namespace-active-kill-switch
`import_agent` and `sync_agent` prefix imported beliefs with `namespace:` and create a `namespace:active` premise whose retraction cascades OUT every belief from that agent.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] lookup-searches-all-terms-substring
`lookup()` performs an all-terms substring search across the full belief block, including ID, text, source, and dependencies.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-dual-mode-requires-sources-db
`ask()` raises `ValueError` when `dual=True` and `sources_db` is not provided.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-dual-short-circuits-if-either-path-empty
`ask()` in dual mode short-circuits the merge step if either the TMS answer or the FTS RAG answer is empty.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-simple-mode-uses-depth-2-markdown-search
`ask()` in `simple=True` mode calls `api.search(format="markdown", depth=2)` for the initial belief context.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-source-chunk-search-default-top-k-10
`_search_source_chunks(question, sources_db, top_k=10)` defaults to returning the top 10 source document chunks.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-command-dispatch-uses-exact-string-dict
`main()` dispatches subcommands through a `commands` dict mapping command-name strings to `cmd_*` functions, so a subcommand registered in argparse but missing from that dict raises `KeyError`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-derive-cluster-and-sample-mutually-exclusive
`cmd_derive` enforces that `--cluster` and `--sample` are mutually exclusive before starting derive work.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-report-timestamps-strip-colons
Report files written by the CLI use ISO timestamps with colons stripped (for example `T105808`) for filesystem safety.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-derive-exhaust-stops-on-saturation
`cmd_derive --exhaust` loops `_derive_one_round` up to `--max-rounds` times and stops when a round adds zero new beliefs (saturation).
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-any-warning-for-three-plus-sl-premises
`_warn_multi_premise` prints a tip when an SL justification has 3 or more premises and `--any` was not passed.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-remainder-favors-largest-clusters
When the sampling budget does not divide evenly across clusters, `cluster_beliefs` allocates remainder slots to the largest clusters first, sorted by descending size.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-auto-k-is-n-beliefs-div-five-clamped
`_auto_k` computes the cluster count as `n_beliefs // 5`, clamped to `[2, max_k]`, and `cluster_beliefs` further constrains `max_k` to `budget // 3`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-embed-sorts-belief-ids
`ClusterCache.embed()` uses `ids = sorted(beliefs.keys())` so the ID list and embedding array have consistent ordering regardless of dict insertion order.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-default-model-is-all-minilm-l6-v2
The default sentence-transformer model used by `cluster.py` is `all-MiniLM-L6-v2`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-list-clusters-short-circuits-at-three-or-fewer
`list_clusters` short-circuits without clustering when the belief set contains 3 or fewer items.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-batch-size-is-50
`CONTRADICTION_BATCH_SIZE` is set to 50, the default number of beliefs sent per LLM contradiction-detection call.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-prompt-defines-four-contradiction-types
`CONTRADICTION_PROMPT` defines four contradiction types: direct negation, incompatible properties, scope conflict, and quantitative mismatch.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-semantic-sub-batches-large-clusters
`detect_contradictions_semantic` further sub-batches large clusters at `CONTRADICTION_BATCH_SIZE` after clustering.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradictions-flush-results-incrementally-to-plan-file
Both contradiction detection strategies flush results after each batch via `api.write_contradiction_plan`, using `header_written` to toggle between write-new-file and append modes.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-budget-proportional-allocation-min-floor-5
When multiple agents contribute beliefs, `build_prompt` divides prompt budget proportionally to each agent's belief count with a minimum floor of 5.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-detect-agents-skips-active-sentinel
`_detect_agents` identifies agent namespaces from colon-separated node IDs and skips `:active` sentinel nodes.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-derived-section-sorts-deepest-first
`_build_derived_section` sorts existing derived conclusions by depth with deepest first.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-build-prompt-wraps-template-format-errors
`build_prompt` catches `KeyError` and `ValueError` from `str.format()` and suppresses the original traceback with `from None`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-cluster-import-is-lazy
`cluster` is imported lazily inside `_build_beliefs_section` only when `cluster=True`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-rebuilds-dependents-before-recompute
After adding imported nodes and nogoods, `_fixup_dependents()` calls `network._rebuild_dependents()` before `network.recompute_all()` because outlist targets may not have existed when referencing nodes were added.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-lets-network-errors-propagate
`import_agent.py` has no explicit try/except blocks; errors from `network.add_node()`, `network.retract()`, or `network.assert_node()` propagate unhandled.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-normalization-uses-unprefixed-ids
Normalized claim dicts use unprefixed belief IDs; namespace prefixing is applied later by import/sync loops and `_build_justifications()`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-returns-stats-dict
All four public import/sync functions return a stats dict with counts of imported, skipped, retracted, and propagated beliefs.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sync-agent-preserves-infrastructure-nodes
`_sync_claims` excludes `agent:active` and `agent:inactive` from `removed_ids`, so sync never deletes infrastructure nodes.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-import_agent.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-bridge-call-tool-joins-content-parts
`call_tool()` returns the MCP result as a single joined string of all content parts.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-mcp_client.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-bridge-get-instructions-returns-server-instructions
`get_instructions()` returns the server-provided instructions string captured from `session.initialize()`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-mcp_client.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-bridge-close-joins-thread-with-5s-timeout
`close()` signals `_shutdown`, joins the background thread with a 5-second timeout, then closes the event loop.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-mcp_client.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-bridge-implements-context-manager
`McpBridge` implements `__enter__` and `__exit__`, enabling `with McpBridge(command) as bridge:` for automatic connect/close.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-mcp_client.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-bridge-command-parsed-with-shlex
`McpBridge` parses its command string with `shlex.split()` into argv for `StdioServerParameters`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-mcp_client.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-bridge-propagates-background-init-errors
If MCP session initialization fails on the background thread, `_session_lifecycle` stores the exception in `self._error` and sets `_ready`; `connect()` then re-raises it in the calling thread.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-mcp_client.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-convert-to-premise-unlinks-dependents
`convert_to_premise()` strips all justifications, unlinks the node from its antecedents' dependents sets, and makes it a premise that defaults to IN.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-remove-justification-cleans-dependents
`remove_justification()` cleans up dependents entries for references no longer used by the remaining justifications.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-challenge-converts-premise-to-justified
If `challenge()` targets a premise, it converts that premise to a justified node with the challenge node in its outlist.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-defend-delegates-to-challenge
`defend(target_id, challenge_id, reason)` is implemented by delegating to `challenge()`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-explain-recurses-valid-justification
For an IN node, `explain()` finds the valid justification and recursively explains its antecedents; for an OUT node, it lists failed antecedents and violated outlist items.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-trace-access-tags-unions-backward-subgraph
`trace_access_tags(node_id)` returns the union of `access_tags` metadata across the entire backward dependency subgraph.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] network-verify-dependents-audits-index-drift
`verify_dependents()` compares the live dependents index against what justifications imply and returns error strings.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-network.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-network-meta-stores-per-project-config
`rms_network_meta` is a key-value store for per-project configuration such as repo paths and the nogood counter.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-api-accepts-connection-string-or-existing-connection
`PgApi` constructor accepts either a PostgreSQL connection string or an existing `psycopg` connection, plus a `project_id`.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-get-status-reports-justification-counts
`PgApi.get_status` lists all nodes with truth values and justification counts.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-lookup-substring-searches-id-text-source-refs
`PgApi.lookup` performs an all-terms substring search over the full belief block: ID, text, source, and references.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-list-gated-finds-out-nodes-blocked-by-in-outlist
`PgApi.list_gated` finds OUT nodes blocked by IN outlist nodes (GATE beliefs).
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-internal-helpers-do-not-commit
Internal PgApi helpers such as `_add_node_raw`, `_propagate`, and `_compute_truth` operate on a passed cursor without committing; the public caller owns the transaction boundary.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-derived-node-inherits-union-access-tags
When PgApi creates a derived node, `_inherit_access_tags` sets its access tags to the union of its antecedents' access tags.
- Source: /home/forge/workspace/entries/2026/05/11/reasons_lib-pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] search-depth-expansion-adds-antecedents
`api.search` with `depth=0` returns only direct FTS matches, `depth=1` adds direct antecedents, and `depth=2` adds transitive antecedents.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] list-nodes-supports-depth-range-filtering
`api.list_nodes()` supports `min_depth` and `max_depth` parameters to filter nodes by justification depth.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-belief-set-returns-only-in-node-ids
`api.get_belief_set()` returns only the IDs of nodes whose truth value is IN.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-log-supports-last-n-filtering
`api.get_log(last=N)` limits audit log retrieval to the most recent N entries.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_api.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-empty-network-returns-no-beliefs-message
When the belief network is empty or no beliefs match the query, `ask()` returns `NO_BELIEFS_MSG` rather than raising an exception.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-dual-mode-uses-three-llm-calls
In `dual=True` mode, `ask()` performs three LLM calls: one for the TMS leg, one for the FTS leg, and one merge call.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] build-tools-section-merges-all-bridges
`_build_tools_section` merges tool descriptors from all provided MCP bridges together with the built-in `search_beliefs` tool.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask_mcp.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] build-tools-section-handles-empty-params-cleanly
For a tool whose input schema has empty `properties`, `_build_tools_section` emits a JSON example without a trailing comma after the tool name.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask_mcp.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-prompt-data-source-instructions-header
`build_ask_prompt` injects MCP instructions under a `Data Source Instructions` header and omits that header when the instructions are empty.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask_mcp.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ask-mcp-dispatch-routes-by-tool-name
When the LLM returns an MCP tool call, `ask()` dispatches it to the matching bridge's `call_tool` method and feeds the returned result back into the tool history.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask_mcp.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] build-mcp-instructions-concatenates-bridges
`_build_mcp_instructions` concatenates instructions from multiple MCP bridges and returns an empty string when no instructions are provided.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_ask_mcp.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-idempotent-operations-report-already-status
The CLI reports `already OUT` when retracting an already-OUT node and `already IN` when asserting an already-IN node.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-empty-state-queries-exit-zero
CLI query commands given an empty database return messages such as `No nodes` or `No matching` with exit code 0, not an error exit code.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] run-cli-harness-catches-systemexit
The `run_cli` test helper catches `SystemExit` raised by the CLI `main()` function and extracts its exit code.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_cli.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-stats-has-fixed-keys-and-sum
`cluster_beliefs` returns a `stats` dictionary containing `n_clusters`, `cluster_sizes`, and `embedding_model`, and the cluster sizes sum to the total number of beliefs.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] auto-k-max-k-caps-cluster-count
`_auto_k` caps the computed cluster count at `max_k` and clamps an explicit `n_clusters` value to the number of beliefs.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cluster-cache-embed-returns-ids-and-embeddings
`ClusterCache.embed(beliefs)` returns a tuple of belief IDs and their embeddings and caches embeddings by text.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_cluster.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] run-cli-harness-invokes-cli-in-process
The run_cli test harness invokes reasons_lib.cli:main in-process by patching sys.argv, sys.stdout, and sys.stderr, returning stdout, stderr, and exit code.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradiction-detection-batches-by-constant-size
detect_contradictions splits IN-only beliefs into batches of CONTRADICTION_BATCH_SIZE and sends each batch as an independent LLM call.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-contradiction-extracts-nogood-id-sections
parse_contradiction_response extracts ### NOGOOD <id> sections from LLM responses, pulling out claims, analysis, and severity.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] contradiction-semantic-variant-clusters-before-llm
detect_contradictions_semantic uses embedding-based clustering to group related beliefs before sending them to the LLM.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_contradictions.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] detect-agents-excludes-active-sentinel
_detect_agents identifies colon-namespaced agent prefixes and excludes :active sentinel nodes from belief counts.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] get-depth-computes-derivation-chain-depth
_get_depth returns 0 for premises, 1 for nodes derived from premises, and 2 for nodes derived from depth-1 nodes.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] parse-proposals-supports-legacy-v09-format
parse_proposals supports the old v0.9 format with backtick-delimited IDs and bold field names, as well as ### DERIVE and ### GATE blocks.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] validate-proposals-rejects-missing-antecedents
validate_proposals rejects any proposal that references an antecedent not present in the current network.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] topic-filter-matches-ids-and-text-with-or-semantics
_filter_by_topic matches space-separated keywords against both node IDs and text fields using OR semantics.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] sample-beliefs-deterministic-budget-bounded
_sample_beliefs returns all beliefs when under budget, caps at the budget when over, and is reproducible with a seeded RNG.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-exhaust-iterates-until-no-new-proposals
The derive --exhaust option iterates until the LLM produces no new proposals.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-auto-writes-json-round-report
derive --auto writes a JSON report with round-by-round stats, and --no-report suppresses the report.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_derive.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-retracted-pin-survives-antecedent-reassertion
In PgApi, a node explicitly retracted remains OUT when its antecedent is re-asserted, even though the justification chain is satisfied.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-namespace-auto-creates-active-premise
Calling PgApi.add_node with a namespace parameter auto-creates an <ns>:active premise node and adds it to the derived node's antecedent list.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-namespace-prefixing-is-idempotent
PgApi.add_node with namespace=agent1 produces agent1:x for input x and does not double-prefix an already-prefixed agent1:x ID.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-challenge-defense-uses-outlist
PgApi challenge and defend operate through the outlist non-monotonic mechanism rather than directly toggling truth values, and defending all challenges restores the target to IN.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] pg-derived-access-tags-union-antecedent-tags
In PgApi, a derived node inherits the union of its antecedents' access tags.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] export-markdown-empty-network-handled
export_markdown accepts PG connection kwargs for an empty belief network and produces markdown without error.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg_dispatch.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-agent-supports-markdown-and-json-input
The API import_agent function supports both markdown and JSON input variants before dispatching parsed data to PgApi.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg_dispatch.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-json-propagates-file-not-found
import_json propagates FileNotFoundError when given a nonexistent path rather than swallowing it.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg_dispatch.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] import-maintenance-repo-negative-cli-now-accept-pg-kwargs
Regression tests confirm that import, maintenance, repo, and negative CLI commands previously gated behind _require_sqlite now accept PG kwargs without error.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_pg_dispatch.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] remove-justification-cleans-dependents-index
After Network.remove_justification, antecedent and outlist nodes' dependents sets are updated to remove the target node unless it remains referenced by another justification.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_remove_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] remove-justification-preserves-shared-antecedent-dependents
If a node appears in both justifications' antecedent lists, removing one justification does not remove it from the dependents index.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_remove_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] remove-justification-cascades-truth-value
If Network.remove_justification causes a node to flip from IN to OUT, the change propagates to all downstream dependents.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_remove_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] remove-justification-rejects-premises
Calling Network.remove_justification on a premise node raises ValueError because premises have no justifications to remove.
- Source: /home/forge/workspace/entries/2026/05/11/tests-test_remove_justification.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] reference-validation-audit-finds-three-uncovered-boundary-gaps
Issue #126 reference validation audit found three specific node ID reference boundaries whose coverage does not establish validation at every boundary.
- Source: /home/forge/workspace/entries/2026/05/11/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-fault-tolerance-audit-finds-three-uncovered-failure-modes
Issue #122 review fault tolerance audit found three specific handlers that do not establish coverage of all failure modes.
- Source: /home/forge/workspace/entries/2026/05/11/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] evolution-tolerance-audit-finds-not-all-boundaries-have-documented-forward-compatibility
Issue #121 evolution tolerance audit found that not all system boundaries have documented forward-compatibility mechanisms.
- Source: /home/forge/workspace/entries/2026/05/11/update.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] ftl-reasons-belief-network-is-model-agnostic
The ftl-reasons belief network is model-agnostic: any LLM that can do tool calling or use CLI commands can query and update it.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] eem-card-is-readme-md-modeled-on-huggingface-model-cards
The EEM_CARD.md format is modeled on HuggingFace model cards and is the README.md in the EEM repo.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] eem-card-has-yaml-frontmatter-with-schema-version-and-type-eem
An EEM card has YAML frontmatter with schema_version "1.0" and type "eem".
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] eem-expert-published-to-huggingface-at-benthomasson-eem-expert
The eem-expert knowledge base was published to HuggingFace at benthomasson/eem-expert on 2026-05-23.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] huggingface-requires-lowercase-license-field
HuggingFace metadata validation requires the license field to be lowercase, such as mit rather than MIT.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] gitignore-excludes-reasons-db-and-wal-files-from-hf-upload
The eem-expert .gitignore excludes reasons.db and its WAL files so that only portable files are uploaded to HuggingFace.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] reasons-import-hf-falls-back-to-raw-http-without-huggingface-hub
The planned reasons import-hf command is designed to work without huggingface_hub installed by falling back to raw HTTP download for public repositories.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] multiple-eems-merge-into-single-reasons-db-with-shared-node-ids-and-nogoods
Multiple EEM repositories can be imported into a single reasons.db, where shared node IDs connect across networks and contradictions are detected as nogoods.
- Source: /home/forge/workspace/entries/2026/05/23/huggingface-eem-distribution.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] eem-builder-runs-on-gce-vms
The EEM builder currently runs on GCE VMs.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mcp-supports-sse-and-streamable-http-transports
MCP supports two HTTP transports: SSE and Streamable HTTP.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cloud-run-mcp-server-uses-streamable-http-transport
The Cloud Run MCP server uses Streamable HTTP transport rather than SSE.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cloud-run-mcp-server-loads-network-json-from-gcs-on-cold-start
The Cloud Run MCP server loads network.json from GCS on cold start and caches it in memory.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cloud-run-mcp-server-exposes-search-show-explain-list-topics-get-topic-tools
The Cloud Run FastAPI MCP server exposes search, show, explain, list_topics, and get_topic tools.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cloud-run-mcp-server-is-read-side-sharing-network-json-via-gcs-with-gce-builder
The Cloud Run MCP server handles the read side and shares network.json with the GCE builder via GCS.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] iap-provides-google-oauth-gating-in-front-of-cloud-run-mcp-server
Identity-Aware Proxy in front of Cloud Run provides Google OAuth gating for the MCP server.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cloud-run-scales-to-zero-with-no-idle-cost
Cloud Run scales to zero and incurs no cost when idle.
- Source: /home/forge/workspace/entries/2026/07/09/mcp-server-cloud-run.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] eem-wiki-generates-static-html-from-network-json
The eem-wiki generates static HTML from network.json belief exports.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] eem-wiki-publicly-deployed-to-cloudflare-pages-at-wiki-llmeem-ai
The eem-wiki is currently deployed publicly to Cloudflare Pages at wiki.llmeem.ai.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] private-wiki-deployment-uses-iap-in-front-of-cloud-run
A private wiki deployment option uses Google Cloud IAP in front of a Cloud Run service to handle Google OAuth.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] private-wiki-build-uses-eem-wiki-build-all
The private wiki build pipeline uses eem-wiki build-all to produce the dist/ directory.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] other-static-hosts-lack-built-in-google-oauth-gating
GitHub Pages, Netlify, Vercel, S3+CloudFront, and nginx do not have built-in Google OAuth gating.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] per-customer-wiki-uses-own-dist-build-behind-iap
Each enterprise customer's private wiki uses its own dist/ build deployed behind IAP and gated to their Google Workspace domain.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] wiki-authentication-is-independent-of-build-process
The authentication question for eem-wiki is independent of the build process because the output is static HTML.
- Source: /home/forge/workspace/entries/2026/07/09/private-wiki-deployment.md
- Source URL: none
- Model: openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017


---

**Generated:** 2026-11-19 (parsed from ## Beliefs sections)
**Source:** 100 summary files

### [ACCEPTED] justification-type-is-a-string-discriminator
`Justification.type` is a plain `str` restricted by convention to `"SL"` or `"CP"`; the module defines no Enum and performs no validation.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] supporting-justification-contains-list-index
`Node.supporting_justification` is an integer index into `Node.justifications`, not an ID or object reference, so the ordering of the `justifications` list is semantically meaningful.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] dependents-is-a-denormalized-reverse-index
`Node.dependents` is a `set[str]` that must mirror the inverse of all `antecedents` and `outlist` edges; this module never populates or checks it.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] module-is-a-validation-free-schema-layer
The file defines no `__post_init__`, no methods, and no validation, so invalid states such as an out-of-range `supporting_justification` or an unknown `truth_value` are constructible without error.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] outlist-enables-non-monotonic-retraction
`Justification.outlist` encodes defeaters: a justification is invalidated whenever any node in `outlist` becomes `IN`, which is what allows beliefs to be retracted without deletion.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] node-defaults-to-in-for-premises
`Node.truth_value` defaults to `"IN"` and a premise is defined as a `Node` with an empty `justifications` list, so freshly constructed premises are `IN` without any propagation pass.
- Source: summaries/2026/11/19/reasons-__init__.md

### [REJECT] network-cache-keyed-by-mtime
`_network_cache` stores `(mtime, Network)` per `db_path`, and read-only calls (`write=False`) reuse the cached `Network` iff the current file mtime equals the cached one; otherwise a fresh `Storage(db_path).load()` is performed.
- Source: summaries/2026/11/19/reasons-api.md
- Rejected: duplicate: Existing `cache-invalidation-is-mtime-only` and `cache-read-returns-identical-object` already capture mtime-keyed cache reuse and fresh load behavior.

### [REJECT] write-context-suppressed-on-exception
`_with_network.__exit__` saves and publishes only when `write=True` and `exc_type is None`; on write exceptions it neither saves the network nor refreshes `_network_cache`, preserving on-disk/cache agreement.
- Source: summaries/2026/11/19/reasons-api.md
- Rejected: duplicate: Existing `cache-write-exception-preserves-prior-state` covers the same no-save-on-exception behavior.

### [REJECT] namespaced-writes-prepend-active-premise
`add_node(namespace=...)` prefixes the id via `_resolve_namespace` and inserts `{ns}:active` at position 0 of the first justification's `antecedents` (creating the premise node if missing), making retraction of `{ns}:active` a cascade kill switch.
- Source: summaries/2026/11/19/reasons-api.md
- Rejected: duplicate: Existing `add-node-namespace-auto-wiring-creates-active-premise` and `api-add-node-auto-wires-namespace-active-premise` capture the same namespace active premise injection.

### [REJECT] accept-proposal-requires-snapshot-match
`accept_proposal` marks a proposal `stale` and returns `{"applied": False}` whenever the live node's `truth_value` or `text_hash` no longer matches the proposal's `snapshot_json`.
- Source: summaries/2026/11/19/reasons-api.md
- Rejected: duplicate: Existing `accept-proposal-detects-drift-before-applying` already covers snapshot/truth/value drift detection.

### [ACCEPTED] what-if-simulations-never-persist
`what_if_retract`, `what_if_assert`, and `what_if_supersede` open `_with_network(..., write=False)` and operate on a `copy.deepcopy` of the network, so the cached `Network` and the database are both left unmodified regardless of outcome.
- Source: summaries/2026/11/19/reasons-api.md

### [REJECT] ask-requires-sources-db-for-dual-mode
`ask` raises `ValueError` at the top if `dual=True` and `sources_db is None`.
- Source: summaries/2026/11/19/reasons-ask.md
- Rejected: duplicate: Existing beliefs `ask-dual-mode-requires-sources-db` and `ask-dual-requires-sources-db` already state this exact ValueError contract.

### [REJECT] extract-tool-call-implements-json-line-protocol
`extract_tool_call` parses the first line beginning with `{` that loads as JSON and contains a `"tool"` key.
- Source: summaries/2026/11/19/reasons-ask.md
- Rejected: duplicate: Existing `ask-tool-call-parsing-is-line-based-json` and related beliefs already capture line-based JSON tool-call extraction with a `"tool"` key.

### [REJECT] ask-prevents-tool-calls-in-final-round
On the last iteration, `build_final_prompt` (no tool schema) is used, and any returned tool call falls back to raw context.
- Source: summaries/2026/11/19/reasons-ask.md
- Rejected: duplicate: Existing `ask-last-iteration-structurally-forces-answer` already states the final iteration omits tool schema and forces an answer.

### [ACCEPTED] tool-search-results-replace-beliefs-context-when-non-empty
A `search_beliefs` result only overwrites `beliefs_context` when it is non-empty and not `"No results found."`.
- Source: summaries/2026/11/19/reasons-ask.md

### [ACCEPTED] dual-mode-detects-emptiness-via-prompt-substrings
Dual mode classifies an empty TMS answer via the exact `NO_BELIEFS_MSG` string or the substring `"don't have enough beliefs"`, coupling it to prompt wording.
- Source: summaries/2026/11/19/reasons-ask.md

### [REJECT] ask-falls-back-to-raw-context-on-llm-errors
Both `subprocess.TimeoutExpired` and generic exceptions from `invoke_model` return `_beliefs_or_no_match(ctx)` instead of raising.
- Source: summaries/2026/11/19/reasons-ask.md
- Rejected: duplicate: Already covered by existing `ask-falls-back-to-raw-search` and `ask-never-raises-on-llm-failure` (and `ask-llm-failure-returns-raw-context`), which capture the same fallback-to-raw-context behavior on LLM errors.

### [ACCEPTED] build-wiki-does-not-perform-grouping
`build_wiki` receives the `groups` dict fully formed; it never calls `_assign_topics` itself.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [REJECT] build-wiki-swallows-llm-errors
LLM generation failures in `build_wiki` are caught, logged to stderr as `WARN`, and the page falls back to raw markdown rendering.
- Source: summaries/2026/11/19/reasons-build_wiki.md
- Rejected: duplicate: Same claim as existing `build-wiki-llm-failure-falls-back-to-template` (errors caught, falls back to template/raw rendering); the stderr WARN detail is a trivial rewording.

### [ACCEPTED] build-wiki-parallel-requires-model
The parallel branch is only entered when **both** `model` is truthy and `parallel > 0`; `parallel > 0` with `model=""` degrades to the static renderer.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] node-to-page-maps-node-ids-to-filenames
`node_to_page` is a `{node_id: "<slug>.md"}` dict built before rendering and reused by `_format_node` and `_linkify` for cross-reference resolution.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [REJECT] linkify-sorts-ids-by-length-descending
`_linkify` processes IDs longest-first so that IDs which are substrings of other IDs are linked after longer matches.
- Source: summaries/2026/11/19/reasons-build_wiki.md
- Rejected: duplicate: existing `build-wiki-linkify-longest-first-prevents-partial-match` already captures longest-first sorting to prevent partial matches

### [ACCEPTED] assign-topics-parses-id-segments
`_assign_topics` splits node IDs on `[-._:]`, drops tokens of length ≤ 2, and matches the first segment against the topic set.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] reserved-slug-index-rewrites-to-index-topic
A group label whose slug is `index` (the reserved slug) is rewritten to `index-topic` before file I/O.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] check-stale-processes-only-in-nodes
`check_stale` and `pin_sources` skip any node whose `truth_value` is not `"IN"`.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] git-errors-yield-changed-true
`file_changed_since` and `file_lines_changed_since` return `True` on subprocess errors or timeouts, forcing content-hash fallback.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] truncated-hashes-are-16-hex-chars
Legacy truncated `source_hash` values are exactly 16 characters and are recognized by prefix match against the full 64-char hash.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] check-stale-mutates-network-in-place
`check_stale`, `hash_sources`, and all `pin_*` functions write to `node.source_hash` / `node.metadata` directly; the caller must persist the network.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] resolve-source-path-prefers-agent-repo
`resolve_source_path` resolution order is agent repo, then `db_dir`, then `repos[first_component]`, then `~/git/...`.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] pin-lines-auto-pins-sha
`pin_lines` sets `pinned_sha` and `source_hash` automatically when the node was not previously pinned.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] cli-handlers-are-thin-api-wrappers
Every `cmd_*` function other than `_derive_one_round` (and its helpers) performs exactly one `reasons.api` call followed by dict-formatted `stdout` output.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] pg-backend-requires-domain-id
`_backend_kwargs` refuses to return PostgreSQL kwargs unless a domain id is supplied via flag or the `REASONS_DOMAIN_ID`/`REASONS_PROJECT_ID` env var, exiting with code 1 otherwise.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] sqlite-only-commands-guarded-by-require-sqlite
Commands that touch the local filesystem, Merkle hashes, embeddings, or LLM-specific workflows call `_require_sqlite` (or the equivalent inline `REASONS_PG_CONNINFO` check) and abort if PostgreSQL is configured.
- Source: summaries/2026/11/19/reasons-cli.md

### [REJECT] main-dispatches-by-command-string-dict
`main()` routes subcommand execution through a `commands: dict[str, Callable]` table rather than `argparse.set_defaults(func=...)`.
- Source: summaries/2026/11/19/reasons-cli.md
- Rejected: trivial: internal dispatch implementation detail (dict table vs set_defaults), obvious from reading main()

### [ACCEPTED] derive-cluster-and-sample-are-mutually-exclusive
`cmd_derive` exits with code 1 if both `--cluster`/`--intra-cluster` and `--sample` are supplied, before making any API call.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] check-stale-exits-nonzero-on-stale-nodes
Unlike the general error-to-exit-1 convention, `cmd_check_stale` deliberately exits 1 when any source is stale or deleted, enabling CI freshness gates.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] cluster-public-functions-require-cluster-extras
Every public clustering function calls `_require_cluster_deps()` before its fast path, so missing sentence-transformers/scikit-learn raises `ImportError` even when clustering would be skipped.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] cluster-cache-keys-on-node-id-and-text-hash
`ClusterCache.embed` invalidates cached embeddings when belief text changes by keying entries on `(node_id, sha256(text)[:16])`.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] cluster-inter-sampling-is-not-budget-guaranteed
`cluster_beliefs` can return fewer than `budget` IDs because per-cluster allocations are capped by cluster size and not redistributed.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] cluster-intra-rotates-size-sorted-clusters
`cluster_beliefs_intra` selects the focus cluster by `round_num % k` after sorting clusters by descending size.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] cluster-auto-k-caps-by-budget-thirds
`cluster_beliefs` and `cluster_beliefs_intra` use `max_k=min(budget // 3, 20)` for auto-k, while `list_clusters` uses the default `max_k=20`.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] token-estimate-uses-chars-div-4
`estimate_tokens` returns `max(1, len(text) // 4)`, the standard BPE approximation.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] compact-prioritizes-nogoods-then-out-then-in
Sections are emitted in that order and share a single `budget`, so nogoods can exhaust the budget before OUT/IN appear.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] budget-check-reserves-footer-space
`_over_budget` adds `footer_tokens` to every line check so the closing token-count line always has reserved room.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] summaries-hide-covering-in-nodes
An IN node whose id appears in another IN node's `metadata["summarizes"]` is omitted from the IN listing and reported via `(N nodes hidden by summaries)`.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] truncate-defaults-to-80-char-node-text
With `truncate=True` (default), `_text` clips node text to 77 chars plus `"..."`, i.e. 80 total.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] omission-and-hidden-lines-bypass-budget-guard
The `... (N more … omitted)` and `(N nodes hidden by summaries)` lines are appended without `_over_budget` checks, so a run can exceed `budget` slightly.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] contradictions-detects-only-in-beliefs
Both `detect_contradictions` and `detect_contradictions_semantic` filter candidate beliefs to nodes whose `truth_value == "IN"` before any LLM call.
- Source: summaries/2026/11/19/reasons-contradictions.md

### [ACCEPTED] parse-contradiction-filters-hallucinated-ids
`parse_contradiction_response` drops any claim not present in `valid_ids` and discards a nogood if fewer than 2 claims survive.
- Source: summaries/2026/11/19/reasons-contradictions.md

### [REJECT] contradictions-requires-2-claims-per-nogood
A nogood is only retained when its parsed `claims` list has length ≥ 2, mirroring the prompt's minimum-2 rule.
- Source: summaries/2026/11/19/reasons-contradictions.md
- Rejected: duplicate: subsumed by parse-contradiction-filters-hallucinated-ids, which already states nogoods with fewer than 2 surviving claims are discarded.

### [ACCEPTED] contradictions-swallows-batch-llm-failures
`invoke_model` errors are caught per batch, logged as `WARN: …` to stderr, and do not abort the loop — the function always returns a list.
- Source: summaries/2026/11/19/reasons-contradictions.md

### [ACCEPTED] write-contradiction-plan-lazy-imported
`write_contradiction_plan` is imported inside `_flush_results` rather than at module top to break the circular dependency with `reasons/api.py`.
- Source: summaries/2026/11/19/reasons-contradictions.md

### [ACCEPTED] beliefs-section-excludes-derived-nodes
`_build_beliefs_section` only lists nodes that are `IN` and not in the derived set; derived nodes are listed separately.
- Source: summaries/2026/11/19/reasons-derive.md

### [ACCEPTED] validate-proposals-requires-at-least-two-antecedents
`validate_proposals` skips proposals with fewer than two antecedent IDs.
- Source: summaries/2026/11/19/reasons-derive.md

### [ACCEPTED] validate-proposals-rejects-similar-retracted-beliefs
`validate_proposals` rejects a proposal whose ID has Jaccard token similarity `>= 0.5` with any `OUT` node.
- Source: summaries/2026/11/19/reasons-derive.md

### [ACCEPTED] apply-proposals-converts-api-errors-to-result-strings
`apply_proposals` catches exceptions from `api.add_node` per proposal and returns them as strings instead of raising.
- Source: summaries/2026/11/19/reasons-derive.md

### [ACCEPTED] gate-proposals-still-use-derived-source-type
`apply_proposals` always passes `source_type="derived"`; GATE behavior is represented by the `unless` argument.
- Source: summaries/2026/11/19/reasons-derive.md

### [ACCEPTED] export-card-returns-string-with-no-io
`export_card` returns a single string and performs no file writes; callers persist the `README.md` themselves.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] out-count-is-not-in
A node is counted as OUT whenever its `truth_value` is anything other than the exact string `"IN"`.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] premises-have-depth-zero
`_node_depth` returns `0` for missing nodes, nodes without justifications, and premise nodes.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] valid-supporting-justification-restricts-depth
When `supporting_justification` is a valid index, only that justification’s antecedents contribute to the node’s derivation depth.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] empty-network-is-safe
`export_card` handles an empty network because retraction percentage guards `total == 0` and `max_depth` uses `default=0`.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-markdown-is-pure-string-export
`export_markdown` returns a markdown string and performs no file I/O or network mutation.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-sorts-in-nodes-before-non-in-nodes
Claims are sorted with `truth_value == "IN"` first, then all non-IN nodes, alphabetically by node ID within each group.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-maps-out-with-stale-or-retract-reason-to-stale
A node with truth value `OUT` and metadata `stale_reason` or `retract_reason` is rendered with status `STALE`.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-classifies-defeaters-via-defeats-node-metadata
An outlist ID is rendered as `Defeated by` only when the referenced node’s metadata has `defeats_node` equal to the current node ID; otherwise it is rendered under `Unless`.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-aggregates-justification-antecedents
`Depends on` is the de-duplicated union of antecedents across all of a node’s justifications, preserving first-seen order.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] hf-uses-stdlib-urllib-not-sdk
The module deliberately uses `urllib.request` and stdlib only; there is no `huggingface_hub` import.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] hf-token-resolution-order
`_resolve_token` resolves in the order: explicit arg → `HF_TOKEN` env → `~/.cache/huggingface/token`.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] default-org-is-eem-hub
Bare repo names are namespaced under `EEM-Hub` unless `REASONS_HF_ORG` overrides it.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] resolve-repo-id-always-returns-org-name
`resolve_repo_id` always returns a single-slash `org/name` string and raises `ValueError` for non-HuggingFace URLs.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] create-repo-swallows-409
`create_repo` treats HTTP 409 as success so it can be called unconditionally before an upload.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] download-network-returns-raw-json-text
`download_network` returns the response body as a `str`; parsing is the caller's responsibility.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] download-network-rethrows-non-401-404
`download_network` only wraps 401 and 404 into `RuntimeError`; other `HTTPError` codes propagate raw, unlike `create_repo` and `upload_file` which wrap all codes.
- Source: summaries/2026/11/19/reasons-hf.md

### [REJECT] inactive-relay-implements-kill-switch
Every imported belief's every outlist includes `<agent>:inactive`, so retracting `<agent>:active` cascades the entire agent's beliefs OUT.
- Source: summaries/2026/11/19/reasons-import_agent.md
- Rejected: duplicate: covered by existing active/inactive relay and kill-switch-outlist beliefs such as agent-beliefs-are-permanently-defeasible-and-updatable.

### [REJECT] active-premise-is-excluded-from-antecedents
`<agent>:active` is never placed in antecedents of imported justifications; only `<agent>:inactive` appears in outlists.
- Source: summaries/2026/11/19/reasons-import_agent.md
- Rejected: duplicate: Same claim already exists as `active-not-in-antecedents`; `active-premise-not-in-antecedents` was also marked covered by it.

### [ACCEPTED] sync-agent-uses-remote-wins-semantics
`_sync_claims` updates existing node fields, adds remote-only beliefs, and retracts local beliefs absent from the remote.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] json-import-preserves-out-node-justifications
`_normalize_json` retains justifications on `truth_value=="OUT"` nodes, enabling resurrection; `_normalize_markdown` strips them.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] import-agent-skips-existing-node-ids
`_import_claims` short-circuits with `skipped += 1` when `agent:id` already exists in the network.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] topological-order-prevents-forward-references
Claims are topologically sorted by `depends_on` before insertion, guaranteeing in-file dependencies are added to the network before the claims that reference them.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] sl-justification-references-only-in-file-deps
The SL justification's `antecedents` are filtered by `claim_by_id` membership, so a claim depending only on pre-existing network nodes gets `justifications=None` and is imported as a premise.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] stale-and-out-claims-are-retracted-after-add
Both `STALE` and `OUT` statuses route through `network.retract(id)` immediately after `add_node`, and both increment `claims_retracted`.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] nogoods-require-at-least-two-live-nodes
A nogood is only appended if `len([a for a in affects if a in network.nodes]) >= 2`; otherwise it is dropped silently.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] frontmatter-whitelist-maps-three-keys
Only `schema_version`, `project_name`, and `created_at` from YAML frontmatter are copied into `network.meta`; other keys are ignored.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] llm-invoke-model-is-single-facade
All LLM calls in the `reasons` package route through `invoke_model()`; no other module imports the SDKs directly.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] llm-requires-lock-for-cost-tracking
Every write to `_cost_tracker` occurs under `_cost_lock`, thread-safety is enforced at the module level.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] openai-prefix-bypasses-resolve-model-cmd
`resolve_model_cmd` deliberately raises for `openai:` models, which are dispatched earlier in `invoke_model` via `_invoke_openai` and the Responses API.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] api-prefix-does-not-record-cost
`_invoke_api` returns content without calling `_record_cost`, so `api:` and `vertex:` calls are invisible to `get_cost_summary()`.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] cli-json-parse-fails-open
`_parse_cli_json` returns raw output on any JSON parse failure, ensuring callers always receive a string response.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] mcpbridge-requires-mcp-package
Constructing `McpBridge` raises `ImportError` when the optional `mcp` package is unavailable.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcpbridge-keeps-session-open-until-shutdown-event
`_session_lifecycle` awaits `_shutdown.wait()` inside the `stdio_client` and `ClientSession` context managers to keep the MCP session alive.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcpbridge-caches-tools-at-connect
Tool metadata is fetched once during `connect()` and returned from `list_tools()` without re-querying the server.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [REJECT] mcpbridge-call-tool-flattens-content
`call_tool()` returns newline-joined text or stringified content from the MCP `CallToolResult`.
- Source: summaries/2026/11/19/reasons-mcp_client.md
- Rejected: duplicate: Same claim already covered by existing belief `call-tool-returns-newline-joined-string` which states call_tool() flattens the MCP result's content list into a newline-joined string.

### [ACCEPTED] mcpbridge-close-uses-thread-safe-signaling
`close()` sets the asyncio shutdown event through `loop.call_soon_threadsafe`.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-server-import-requires-mcp-extra
Importing `reasons.mcp_server` raises `ImportError` if the `mcp` package is missing, directing users to install `ftl-reasons[mcp]`.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-server-delegates-all-mutations-to-api
Every tool body calls a `reasons.api.*` function; no tool accesses the database directly.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] reasons-db-env-overrides-walk-up-discovery
`_find_db` returns `REASONS_DB` if set, otherwise walks up from cwd for `reasons.db`, otherwise falls back to `api.DEFAULT_DB`.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-read-tools-propagate-exceptions
`search`, `list_beliefs`, `compact`, `list_proposals`, `status`, `list_gated`, `export_markdown`, and `topics` have no try/except and let exceptions surface to FastMCP, unlike mutating tools which return `{"error": ...}`.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-names-only-handled-locally-for-lists
`list_beliefs`, `status`, and `list_gated` implement `output_format="names-only"` by post-processing API results, whereas `search` and `compact` forward `output_format` into the API.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] fresh-hash-computation-ignores-stored-hashes
`_fresh_merkle_hash` and `_fresh_content_hash` recompute hashes from current text and never read stored `text_hash` or `content_hash`.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] merkle-hash-follows-supporting-justification-only
A node’s Merkle hash only incorporates its `supporting_justification`; alternate justifications do not affect that node hash.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] verify-all-counts-missing-hashes-separately-from-findings
`verify_all` reports missing text/content hashes in `missing_hashes` and does not include them in `findings`.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] backfill-hashes-only-fills-missing-hashes
`backfill_hashes` computes only missing `text_hash`/`content_hash` values and leaves existing hashes untouched.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] cycle-guard-returns-empty-string-hash
A back-edge encountered during recursive Merkle computation returns `compute_text_hash("")` rather than raising or reporting a cycle.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] schema-version-is-hardcoded
`SCHEMA_VERSION` is a module constant `"1.0"` and is not derived from the installed package version.
- Source: summaries/2026/11/19/reasons-metadata.md

### [REJECT] build-meta-returns-six-keys
`build_meta` returns a dict with `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, and `generator`.
- Source: summaries/2026/11/19/reasons-metadata.md
- Rejected: duplicate: identical to existing `build-meta-returns-fixed-six-key-dict`

### [ACCEPTED] build-meta-preserves-created-at
A truthy `created_at` argument is copied verbatim; falsey values are replaced with the current UTC timestamp.
- Source: summaries/2026/11/19/reasons-metadata.md

### [ACCEPTED] build-meta-refreshes-updated-at
`updated_at` is always regenerated using `datetime.now(timezone.utc)` with seconds precision.
- Source: summaries/2026/11/19/reasons-metadata.md

### [ACCEPTED] generator-version-lookup-swallows-errors
`_get_generator` catches any `Exception` from `importlib.metadata.version("ftl-reasons")` and returns `ftl-reasons/unknown`.
- Source: summaries/2026/11/19/reasons-metadata.md

### [REJECT] mind-service-client-is-pure-stdlib
The module imports only from `json`, `os`, and `urllib.*`; no third-party HTTP library is required.
- Source: summaries/2026/11/19/reasons-mind_service.md
- Rejected: trivial: merely restates the module's import list, observable from imports

### [ACCEPTED] config-resolution-prefers-explicit-params
`_resolve_config` gives explicit function arguments precedence over `MIND_*` environment variables.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] mind-service-missing-config-is-aggregated
`_resolve_config` collects all missing fields and raises a single `RuntimeError` rather than failing on the first.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] resolved-base-url-has-no-trailing-slash
`_resolve_config` always strips trailing `/` from the URL before returning it.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] fetch-export-maps-401-and-404-to-runtimeerror
`fetch_export` converts HTTP 401 and 404 into `RuntimeError`, chained from the original `HTTPError`.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] dependents-index-is-derived-from-justifications
The `dependents` reverse index is reconstructible via `_rebuild_dependents()` and validated by `verify_dependents()`; no independent source of truth exists.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] outlist-nodes-are-dependents
Unlike pure antecedent-only TMSes, this implementation registers outlist nodes in `dependents`, which is what lets retracting a superseder restore the superseded node.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] premises-default-in-and-skip-compute
`add_node` sets `truth_value = "IN"` for zero-justification nodes, and `_compute_truth` returns the current value for premises rather than recomputing.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] node-is-in-if-any-justification-valid
`_compute_truth` returns IN at the first valid justification scanned in reverse order and sets `supporting_justification` to that index.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] retracted-nodes-are-skipped-during-propagation
`_propagate` short-circuits any dependent with `metadata["_retracted"] == True` so explicitly retracted nodes are not silently reasserted by propagation.
- Source: summaries/2026/11/19/reasons-network.md

### [REJECT] nogood-short-circuits-when-not-all-nodes-in
`add_nogood` records the `Nogood` object but returns `[]` and performs no backtracking unless every listed node is currently `IN`.
- Source: summaries/2026/11/19/reasons-network.md
- Rejected: duplicate: Covered by existing `add-nogood-inactive-contradiction-returns-empty` / `add-nogood-inactive-when-either-node-out`.

### [REJECT] supersede-is-reversible-via-outlist
Because `supersede` uses outlist entries rather than direct truth-value mutation, retracting the `new_id` restores the `old_id` through normal propagation.
- Source: summaries/2026/11/19/reasons-network.md
- Rejected: duplicate: existing `all-defeat-mechanisms-are-reversible` already states supersession (outlist-based defeat) is inherently reversible

### [ACCEPTED] removing-last-justification-is-rejected
`remove_justification` raises `ValueError` on the final justification; callers must use `convert_to_premise` or `retract` to change a derived node into a premise.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] pg-requires-optional-psycopg
`psycopg` is imported in a `try/except`; `_require_psycopg` raises `ImportError` with install instructions when it's missing.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] domain-id-enables-multi-tenancy
every SQL statement filters by `self.domain_id`, and `(id, domain_id)` is the nodes/nogoods primary key, so tenant data is isolated at the schema level.
- Source: summaries/2026/11/19/reasons-pg.md

### [REJECT] what-if-rollbacks-are-unconditional
`what_if_retract` and `what_if_assert` both call `self.conn.rollback()` in a `finally`, so they can never persist a state change.
- Source: summaries/2026/11/19/reasons-pg.md
- Rejected: duplicate: existing `both-backends-support-safe-hypothetical-reasoning` already covers what-if operations not persisting state changes

### [ACCEPTED] single-justification-removal-is-blocked
`remove_justification` raises `ValueError` when a node would be left with zero justifications; `convert_to_premise` is the sanctioned alternative.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] propose-updates-defaults-to-in-beliefs
When `belief_ids` is `None`, `propose_updates` selects only nodes with `truth_value == "IN"`, iterating them in sorted order for determinism.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] propose-updates-filters-unknown-ids
When an explicit `belief_ids` list is passed, IDs not present in `nodes` are dropped before any LLM call.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] batch-failures-do-not-abort-propose-updates
Exceptions from `invoke_model`, parsing, or the `on_batch` callback are caught per batch and logged as `WARN` to stderr; remaining batches still execute.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] parse-update-proposals-returns-first-non-empty-array
`parse_update_proposals` returns immediately on the first `[` position that `raw_decode`s into a list with at least one valid `{id: ...}` dict, ignoring any later arrays.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] parse-update-proposals-coerces-enum-fields
Out-of-taxonomy `action`/`failure_mode`/`basis` values are downgraded to `"update"`/`""`/`"prior-knowledge"` respectively rather than rejected.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] change-event-is-frozen-dataclass
`ChangeEvent` is an immutable frozen dataclass with fields `type`, `node_id`, `old_truth_value`, `new_truth_value`, and `db_path`.
- Source: summaries/2026/11/19/reasons-pubsub.md

### [ACCEPTED] publish-isolates-subscriber-exceptions
A subscriber exception is caught and logged by `publish()`, and other subscribers still receive the event batch.
- Source: summaries/2026/11/19/reasons-pubsub.md

### [ACCEPTED] global-subscribers-receive-db-scoped-events
A callback registered with `db_path=None` is included for every database path passed to `publish()`.
- Source: summaries/2026/11/19/reasons-pubsub.md

### [ACCEPTED] compute-changes-emits-deletions-for-missing-after-keys
`compute_changes()` emits a `node_changed` event with `new_truth_value=None` when a node exists in `before` but is absent from `after`.
- Source: summaries/2026/11/19/reasons-pubsub.md

### [ACCEPTED] publish-noops-on-empty-event-list
`publish()` returns immediately when `events` is empty and does not notify subscribers.
- Source: summaries/2026/11/19/reasons-pubsub.md

### [ACCEPTED] repair-beliefs-only-processes-valid-false
`repair_beliefs` and `repair_smuggled_beliefs` process only review results where `valid` is explicitly false; missing `valid` defaults to valid.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] search-and-link-restricts-candidates-to-in-premises
Search-and-link candidate selection excludes the belief itself, existing antecedents, non-`IN` nodes, and nodes with justifications.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-beliefs-triages-with-llm-before-mutation
`repair_beliefs` calls `triage_belief` first and only executes a repair branch when triage returns a pattern in `VALID_PATTERNS`.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] dry-run-suppresses-api-mutations
`dry_run=True` still performs LLM and search calls but skips `add_justification`, `supersede_with_text`, `retract_node`, and `set_metadata`.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] json-parsers-scan-with-raw-decode
The LLM response parsers use `json.JSONDecoder.raw_decode` while scanning each `{` so they tolerate prose around JSON.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] empty-premise-ids-returns-empty-list
`repair_premises` returns `[]` immediately when `premise_ids` is empty, with no LLM calls.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] llm-failures-are-swallowed-as-error-actions
Exceptions inside `_repair_one`'s `try` block are converted to `{"action": "error", "rationale": str(e)}` rather than re-raised.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] parse-failures-downgrade-to-retract
An unparseable LLM response yields `{"action": "retract", "rationale": "Could not parse repair response"}`.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] parallel-mode-preserves-completion-order-not-input-order
Parallel results are appended in `as_completed` order, so `all_results` may not match the order of `premise_ids`.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] repair-one-has-unguarded-node-lookup
`nodes[node_id]` is accessed outside the `try`, so a missing node id raises `KeyError` and, in parallel mode, aborts the `as_completed` loop.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] on-result-receives-accumulating-list
The `on_result` callback is invoked with the full growing `all_results` list after each item, not with a single new result.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] parse-repair-response-scans-for-first-valid-json-array
Parsing iterates over each `[` position and returns the first list containing at least one dict with an `id` key, dropping malformed entries.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] report-belief-returns-markdown
`report_belief` always returns a Markdown string, using deterministic structured rendering when `model` is empty and LLM narrative rendering when `model` is non-empty.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] structured-renderer-requires-premise-id
`_build_structured_report` requires every `premises_data` element to contain an `id` key; missing it raises `KeyError`.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] only-in-truth-value-gets-plus-marker
The structured renderer maps only `truth_value == "IN"` to `[+]`; every other value, including `UNKNOWN`, maps to `[-]`.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] source-chunks-must-be-a-mapping
Both formatting helpers call `source_chunks.get(...)`, so callers must pass a mapping keyed by premise id; `None` raises `AttributeError`.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] llm-failure-propagates-without-structured-fallback
If `model` is set and `invoke_model` raises, `report_belief` propagates the exception instead of falling back to the structured report.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-gated-selects-llm-path-when-model-is-truthy
If `model` is non-empty, `report_gated()` uses `_format_data_for_prompt()` and `generate_report()`; otherwise it uses `_build_structured_report()`.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] llm-path-prepends-title-and-summary
In the LLM path, `report_gated()` prepends `# Gated Beliefs Report` and the generated-date summary before the LLM content.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] structured-report-truncates-retracted-and-gated-text
The structured path truncates retracted premise text to 80 characters and gated belief text to 100 characters; the prompt path does not truncate.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] structured-report-escapes-pipes-only-in-retracted-table
Only retracted premise text and `retract_reason` replace `|` with `\|`; blocker and gated belief text are not escaped.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-has-no-llm-fallback
If `invoke_model()` raises, the exception propagates and the code does not fall back to the structured report.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] review-beliefs-requires-justifications
`review_beliefs` only selects nodes whose `justifications` list is non-empty, whether IDs are auto-selected or caller-provided.
- Source: summaries/2026/11/19/reasons-review.md

### [ACCEPTED] review-beliefs-batches-llm-calls
The function splits work into chunks of `batch_size` (default 20) LLM invocations, one prompt per chunk.
- Source: summaries/2026/11/19/reasons-review.md

### [ACCEPTED] parse-review-response-defaults-to-permissive
Missing `valid`/`sufficient`/`necessary` fields in LLM JSON default to `True`.
- Source: summaries/2026/11/19/reasons-review.md

### [ACCEPTED] review-batch-errors-are-swallowed
Per-batch exceptions in `review_beliefs` are logged to `stderr` and the batch is skipped, not propagated.
- Source: summaries/2026/11/19/reasons-review.md

### [ACCEPTED] review-produces-structured-defeaters
When `valid` is `false`, results carry `scope_findings`, `missing_property`, and `defeat_reason_type` for downstream defeater construction.
- Source: summaries/2026/11/19/reasons-review.md

### [ACCEPTED] classify-defeat-reason-maps-to-enum
`classify_defeat_reason` resolves LLM output to a `DEFEAT_REASON_TYPES` value via exact-then-substring matching, otherwise returns `""`.
- Source: summaries/2026/11/19/reasons-review.md

### [ACCEPTED] review-justifications-filters-to-in-truth-value
Both the default and explicit `belief_ids` branches require `node.get("truth_value") == "IN"` before including a node.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] review-justifications-requires-multi-antecedent-sl
A node is eligible only if it has at least one justification with `type == "SL"` and `len(antecedents) >= min_antecedents` (default 2).
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] parse-justification-review-returns-first-valid-array
`parse_justification_review` scans for the first bracket with a successfully decoded JSON list containing an item with `id`, and ignores subsequent arrays.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] batch-failures-do-not-abort-review
A `_process_batch` exception inside either the serial loop or a parallel `future.result()` is caught, logged to stderr, and does not stop remaining batches.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] on-batch-callback-receives-cumulative-results
`on_batch` is invoked after each batch with the full accumulated `all_results`, not just the newest batch's results.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] parallel-mode-loses-batch-order
When `parallel > 0`, results are appended in `as_completed` order, so the returned list is not necessarily in the original `belief_ids` order.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] review-premises-groups-by-source
Premises are bucketed by `node["source"]` and flattened before batching, so final result order follows first-occurrence source, not `premise_ids` order.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] parse-premise-review-response-scans-every-bracket
`parse_premise_review_response` iterates every `[` in the response and uses `JSONDecoder.raw_decode`, returning the first parseable list with id-bearing dicts.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] review-premises-swallows-batch-failures
Exceptions from `_process_batch` (and from `future.result()` in parallel mode) are caught in `review_premises` and logged as `WARN`, never re-raised.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] on-batch-receives-mutable-cumulative-list
`on_batch` is invoked with the same `all_results` list object, mutated in place, after each batch.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] process-batch-returns-list-not-tuple
Despite its docstring saying `(batch_results, error_or_None)`, `_process_batch` returns a single `list[dict]` from `parse_premise_review_response`.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] missing-premise-nodes-silently-omitted
A `premise_id` absent from `nodes` produces an empty formatted chunk and no corresponding result entry, with no warning emitted.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] save-replaces-whole-network
`save()` deletes every row in every core table and re-inserts from the in-memory `Network` inside a single transaction.
- Source: summaries/2026/11/19/reasons-storage.md

### [ACCEPTED] save-preserves-external-tags-and-sources
`save()` snapshots `node_tags`/`node_sources` before clearing and restores them for nodes still present.
- Source: summaries/2026/11/19/reasons-storage.md

### [ACCEPTED] access-tags-are-prefixed
Access tags are inserted into `node_tags` as `access:<tag>`, while plain `metadata["tags"]` are inserted verbatim.
- Source: summaries/2026/11/19/reasons-storage.md

### [REJECT] load-bypasses-add-node
`load()` constructs `Node` objects directly and calls `network._rebuild_dependents()` instead of using `Network.add_node`.
- Source: summaries/2026/11/19/reasons-storage.md
- Rejected: duplicate: existing `bootstrap-bypasses-incremental-propagation` already covers loading bypassing incremental `add_node` semantics and rebuilding graph state directly.

### [ACCEPTED] next-nogood-id-skipped-on-load
`load()` explicitly skips the `next_nogood_id` key in `network_meta`, leaving the counter to be recomputed from loaded nogoods.
- Source: summaries/2026/11/19/reasons-storage.md

### [REJECT] proposals-schema-unused-here
`PROPOSALS_SCHEMA` is declared in this file but never executed by `Storage._init_schema`.
- Source: summaries/2026/11/19/reasons-storage.md
- Rejected: trivial: unused/dead schema declaration; low-value implementation detail visible from file structure

### [REJECT] build-meta-import-unused
`build_meta` is imported from `reasons.metadata` but never referenced in this file.
- Source: summaries/2026/11/19/reasons-storage.md
- Rejected: trivial: unused import, obvious from imports/naming

### [ACCEPTED] verify-beliefs-returns-empty-dict-on-parse-failure
`verify_beliefs` (via `parse_verify_response`) returns `{}` rather than raising when the model output is empty or not valid JSON.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] read-source-requires-resolve-source-path
`read_source` depends on `check_stale.resolve_source_path` for path resolution and has no independent path-resolution logic.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] source-truncation-caps-at-30k-chars
`read_source` truncates content at `MAX_SOURCE_CHARS` (30,000) and appends a `[... truncated ...]` marker.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] verify-prompt-forces-verbatim-quote
`VERIFY_PROMPT` explicitly instructs the model to copy the quote verbatim and to mark `INCONCLUSIVE` when source is missing, but the code does not verify the quote against the source.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] parse-verify-response-normalizes-verdict-case
`parse_verify_response` uppercases the `verdict` string but does not validate it against the four allowed values.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] verify-beliefs-leaves-read-source-to-caller
`verify_beliefs` accepts pre-read source content; it never calls `read_source` itself, so callers control disk I/O.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] reasons-lib-is-backwards-compatibility-shim
`reasons_lib/__init__.py` exists solely to re-export the canonical `reasons` package for legacy imports.
- Source: summaries/2026/11/19/reasons_lib-__init__.md

### [REJECT] reasons-lib-depends-on-reasons
Importing `reasons_lib` executes `from reasons import *` and therefore imports the `reasons` package.
- Source: summaries/2026/11/19/reasons_lib-__init__.md
- Rejected: trivial: stating that module X imports module Y is the canonical trivial example.

### [REJECT] reasons-lib-reexports-reasons-public-api
Names available through `reasons_lib` are determined by `reasons`’s public namespace and optional `__all__`.
- Source: summaries/2026/11/19/reasons_lib-__init__.md
- Rejected: trivial: restates star-import namespace semantics visible from a single import line.

### [REJECT] reasons-lib-has-no-local-runtime-logic
This file defines no classes, functions, constants, or exception handling.
- Source: summaries/2026/11/19/reasons_lib-__init__.md
- Rejected: trivial: absence of definitions is directly visible from reading the file.

### [REJECT] reasons-lib-does-not-alias-the-reasons-module
`reasons_lib` is a separate module object whose namespace is populated by a star import, not a true module alias.
- Source: summaries/2026/11/19/reasons_lib-__init__.md
- Rejected: trivial: module-identity distinction is derivable from the single star-import line.

### [ACCEPTED] reasons-lib-api-is-compat-shim
`reasons_lib/api.py` contains no API implementation and only re-exports `reasons.api` via a star import.
- Source: summaries/2026/11/19/reasons_lib-api.md

### [ACCEPTED] reasons-lib-api-requires-reasons-api-importability
Importing `reasons_lib.api` fails if `reasons.api` cannot be imported; there is no fallback implementation.
- Source: summaries/2026/11/19/reasons_lib-api.md

### [ACCEPTED] reasons-lib-api-exports-reasons-api-public-names
The names available on `reasons_lib.api` are determined by `reasons.api`’s star-import export set at import time.
- Source: summaries/2026/11/19/reasons_lib-api.md

### [ACCEPTED] reasons-lib-api-has-no-local-all-or-getattr
The shim does not define `__all__`, `__getattr__`, or deprecation warnings.
- Source: summaries/2026/11/19/reasons_lib-api.md

### [REJECT] reasons-lib-api-is-imported-by-canonical-modules
Canonical modules such as `reasons/ask.py` and `reasons/cli.py` import this legacy shim, so it is part of internal runtime imports, not just external compatibility.
- Source: summaries/2026/11/19/reasons_lib-api.md
- Rejected: trivial: merely states module import relations visible from file structure

### [ACCEPTED] reasons-lib-ask-is-alias-of-reasons-ask
`reasons_lib.ask` contains no implementation; every public name it exposes is a re-export of the corresponding name in `reasons.ask`.
- Source: summaries/2026/11/19/reasons_lib-ask.md

### [REJECT] reasons-lib-ask-depends-on-reasons-ask
`reasons/ask.py` must be importable for `reasons_lib/ask.py` to load; the dependency direction is `reasons_lib → reasons`, not the reverse.
- Source: summaries/2026/11/19/reasons_lib-ask.md
- Rejected: trivial: States the obvious import dependency of a wildcard shim (`from reasons.ask import *` implies reasons_lib→reasons); no non-obvious architectural content.

### [REJECT] reasons-lib-ask-uses-star-import
The shim relies on `from reasons.ask import *` with `# noqa: F401,F403`, so its public surface is determined by `reasons.ask.__all__` (or its non-underscore names).
- Source: summaries/2026/11/19/reasons_lib-ask.md
- Rejected: trivial: Describes a wildcard import visible directly from the file; "module X imports Y" style fact, nothing durable or non-obvious.

### [REJECT] reasons-lib-ask-preserves-object-identity
Calling `reasons_lib.ask.<fn>` executes the exact same function object as `reasons.ask.<fn>`; identity (`is`) holds between the two modules' symbols.
- Source: summaries/2026/11/19/reasons_lib-ask.md
- Rejected: trivial: Direct mechanical consequence of `from X import *`; any developer sees this from the one-line shim.

### [REJECT] reasons-lib-ask-adds-no-error-handling
All exceptions raised inside `reasons.ask` (including at import time) propagate unchanged through `reasons_lib.ask`.
- Source: summaries/2026/11/19/reasons_lib-ask.md
- Rejected: trivial: Follows trivially from the file being a bare re-export; no actual error-handling logic exists to reason about.

### [REJECT] reasons-lib-build-wiki-is-a-compatibility-shim
The file contains only a module docstring and a wildcard import from `reasons.build_wiki`, with no local business logic.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md
- Rejected: trivial: Describes the file's structure (docstring + wildcard import) that is obvious from reading the file; not a substantive invariant.

### [REJECT] reasons-lib-build-wiki-requires-reasons-build-wiki
Importing `reasons_lib.build_wiki` requires `reasons.build_wiki` to exist and import successfully.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md
- Rejected: trivial: Obvious from the star-import/forwarding file structure; not a durable invariant.

### [REJECT] reasons-lib-build-wiki-export-surface-follows-star-import
Its re-exported names are determined by star-import semantics from `reasons.build_wiki`, not by an explicit local `__all__`.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md
- Rejected: trivial: Visible directly from the import statement; module re-export mechanics are self-evident.

### [REJECT] reasons-lib-build-wiki-propagates-import-errors
It has no try/except, so canonical-module import failures propagate to callers unchanged.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md
- Rejected: trivial: Obvious consequence of no try/except; anyone can see from the file.

### [REJECT] reasons-api-imports-compat-path
Repository context lists `reasons/api.py` as an importer of `reasons_lib.build_wiki`, so the compatibility path is still used internally.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md
- Rejected: trivial: An import relationship any developer sees from imports; not a durable architectural invariant.

### [REJECT] reasons-lib-check-stale-re-exports-reasons-check-stale
This module forwards the entire public namespace of `reasons.check_stale` via a single `from reasons.check_stale import *`.
- Source: summaries/2026/11/19/reasons_lib-check_stale.md
- Rejected: trivial: A star-import forwarding fact self-evident from the module's import statement.

### [REJECT] shim-adds-no-new-api
The file defines no functions, classes, or constants of its own; its only runtime artifact is the imported namespace.
- Source: summaries/2026/11/19/reasons_lib-check_stale.md
- Rejected: trivial: file-structure observation; no definitions is obvious from the import shim.

### [REJECT] shim-requires-reasons-check-stale
The shim has a one-way dependency on `reasons.check_stale`; any import-time failure in the canonical module propagates to legacy importers.
- Source: summaries/2026/11/19/reasons_lib-check_stale.md
- Rejected: trivial: the one-way import and standard import-failure propagation are obvious from the shim's import statement.

### [ACCEPTED] reasons-is-canonical-and-reasons-lib-is-legacy
The docstring names `reasons.check_stale` as the intended import path, making `reasons_lib.check_stale` a deprecated alias.
- Source: summaries/2026/11/19/reasons_lib-check_stale.md

### [REJECT] shim-uses-noqa-f401-f403
The star-import line carries `# noqa: F401,F403` to suppress unused-import and star-import lint warnings, and removing it would break lint checks without affecting runtime.
- Source: summaries/2026/11/19/reasons_lib-check_stale.md
- Rejected: trivial: lint-comment detail; removing it is a tooling concern, not a durable runtime invariant.

### [REJECT] reasons-lib-cli-is-a-shim
`reasons_lib/cli.py` contains exactly one executable statement and defines no functions, classes, or constants of its own.
- Source: summaries/2026/11/19/reasons_lib-cli.md
- Rejected: ephemeral: exact statement count is a point-in-time file metric, and the no-definitions part is trivial.

### [ACCEPTED] shim-reexport-surface-maps-to-reasons-cli-all
The names available from `reasons_lib.cli` are exactly `reasons.cli.__all__` when defined, otherwise every non-underscore module-global of `reasons.cli`.
- Source: summaries/2026/11/19/reasons_lib-cli.md

### [ACCEPTED] shim-creates-independent-name-bindings
`reasons_lib.cli` and `reasons.cli` are distinct module objects sharing the same initial object references, so rebinding an attribute on one does not affect the other.
- Source: summaries/2026/11/19/reasons_lib-cli.md

### [ACCEPTED] shim-import-is-eager-and-unprotected
Importing `reasons_lib.cli` fully imports `reasons.cli` and propagates any of its import-time exceptions, with no try/except in the shim.
- Source: summaries/2026/11/19/reasons_lib-cli.md

### [REJECT] shim-does-not-forward-private-names-or-submodules
Star-import semantics prevent `reasons_lib.cli` from exposing underscore-prefixed names or submodules such as `reasons.cli.argparse`.
- Source: summaries/2026/11/19/reasons_lib-cli.md
- Rejected: trivial: standard star-import semantics obvious from the import statement

### [REJECT] reasons-lib-cli-imported-by-reasons-cli-raises-circularity-question
The dependency listing claims `reasons/cli.py` imports this shim, which is only safe if that import is guarded or deferred.
- Source: summaries/2026/11/19/reasons_lib-cli.md
- Rejected: speculative: asks whether a circular import is safe rather than stating an observed code fact

### [ACCEPTED] reasons-lib-cluster-is-a-shim
`reasons_lib/cluster.py` contains no implementation of its own; it re-exports `reasons.cluster` via `from reasons.cluster import *`.
- Source: summaries/2026/11/19/reasons_lib-cluster.md

### [REJECT] cluster-shim-requires-reasons-package
Importing `reasons_lib.cluster` hard-fails with `ImportError` if `reasons.cluster` is not importable; there is no try/except fallback.
- Source: summaries/2026/11/19/reasons_lib-cluster.md
- Rejected: trivial: Direct consequence of the wildcard import statement; no fallback is evident from `from reasons.cluster import *` and adds nothing beyond the shim fact.

### [REJECT] cluster-shim-preserves-name-identity
Symbols accessed via `reasons_lib.cluster.X` are the same objects as `reasons.cluster.X`, not copies.
- Source: summaries/2026/11/19/reasons_lib-cluster.md
- Rejected: trivial: Standard Python import semantics — `import *` never copies objects; obvious from module structure.

### [REJECT] cluster-shim-inherits-api-from-reasons-cluster
The set of names importable from `reasons_lib.cluster` is exactly `reasons.cluster`'s public (`__all__` or non-underscore) namespace; the shim performs no curation.
- Source: summaries/2026/11/19/reasons_lib-cluster.md
- Rejected: trivial: Consequence of `from reasons.cluster import *`, which any developer sees from the import line; redundant with the shim belief.

### [REJECT] cluster-shim-suppresses-only-f401-and-f403
The `# noqa: F401,F403` annotation silences only the "unused import" and "wildcard import" flake8 rules, leaving other linting of this file active.
- Source: summaries/2026/11/19/reasons_lib-cluster.md
- Rejected: trivial: Lint-annotation detail about flake8 rule codes, not a durable architectural or behavioral invariant.

### [REJECT] reasons_lib-compact-is-a-shim
`reasons_lib/compact.py` contains no implementation and re-exports everything from `reasons.compact` via `import *`.
- Source: summaries/2026/11/19/reasons_lib-compact.md
- Rejected: trivial: one-line re-export via `import *` is obvious from file/import structure.

### [REJECT] reasons_lib-compact-requires-reasons-compact
Importing the shim necessarily triggers a full import of `reasons.compact`, so failures there propagate directly to the legacy path.
- Source: summaries/2026/11/19/reasons_lib-compact.md
- Rejected: trivial: generic Python import behavior, not a durable architectural invariant.

### [REJECT] reasons_lib-compact-preserves-object-identity
Symbols imported from the shim are the same objects as those in `reasons.compact`, not copies or wrappers.
- Source: summaries/2026/11/19/reasons_lib-compact.md
- Rejected: trivial: standard `import *` object identity, obvious Python semantics.

### [REJECT] reasons-api-imports-reasons_lib-compact
`reasons/api.py` still depends on the legacy `reasons_lib` path for compact functionality.
- Source: summaries/2026/11/19/reasons_lib-compact.md
- Rejected: trivial: an ordinary current import/dependency, visible from imports and not a constraint.

### [REJECT] reasons_lib-compact-produces-no-own-exports
The shim exposes no names that are not also public in `reasons.compact`; its public surface is fully determined by the target module's `__all__` or public naming.
- Source: summaries/2026/11/19/reasons_lib-compact.md
- Rejected: trivial: `import *` public-surface behavior is obvious Python semantics.

### [REJECT] reasons-lib-contradictions-is-a-compat-shim
The module defines no logic and only re-exports `reasons.contradictions` via a star import.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: Obvious from the module’s single star-import/file structure; essentially “module X imports Y.”

### [REJECT] reasons-lib-contradictions-requires-reasons-contradictions
Importing the shim fails with the target module's `ImportError` if `reasons.contradictions` cannot be imported.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: This is standard import propagation visible from the import statement.

### [REJECT] reasons-lib-contradictions-lacks-all
The shim does not define `__all__`, so `reasons_lib.contradictions.__all__` raises `AttributeError` regardless of the target module.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: Absence of `__all__` is a file-structure detail directly visible in the shim.

### [REJECT] reasons-lib-contradictions-exports-identical-objects
Every exported symbol is the same object as its `reasons.contradictions` counterpart (`is` identity holds).
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: Object identity is an immediate consequence of star re-export/import aliasing.

### [REJECT] reasons-api-imports-legacy-contradictions
`reasons/api.py` depends on `reasons_lib.contradictions` rather than importing `reasons.contradictions` directly, routing the canonical package through the shim.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: This is a single import-path fact visible from `api.py`, not a broader architectural invariant.

### [REJECT] reasons-lib-derive-is-a-shim
`reasons_lib/derive.py` contains no logic beyond a docstring and a star import of `reasons.derive`.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: visible file/import structure, not an architectural invariant.

### [REJECT] reasons-lib-derive-reexports-reasons-derive
All public symbols of `reasons.derive` are bound into `reasons_lib.derive` with identical object identity.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: direct consequence of the star import and visible from the module.

### [REJECT] reasons-lib-derive-requires-reasons-derive-importable
Import failures in `reasons.derive` propagate unchanged through the shim; there is no fallback or `try/except`.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: standard Python import behavior with no fallback.

### [REJECT] reasons-api-depends-on-legacy-shim
`reasons/api.py`, `reasons/cli.py`, and three test modules import `reasons_lib.derive` rather than `reasons.derive` directly.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: current import inventory, including a changing count of test modules.

### [REJECT] reasons-lib-derive-excludes-private-names
Because the shim uses `import *` with no `__all__` override, underscore-prefixed names in `reasons.derive` are not forwarded.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: standard `import *` semantics visible from the shim.

### [REJECT] reasons-lib-derive-defines-no-all
The shim does not define `__all__`, so a star import of the shim exposes every non-underscore binding in its namespace.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: standard Python star-import behavior; obvious from absence of `__all__`.

### [REJECT] reasons-lib-export-card-is-a-shim
`reasons_lib/export_card.py` contains no implementation logic; its only statement is `from reasons.export_card import *`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md
- Rejected: trivial: file structure is directly visible from the shim's single import statement.

### [REJECT] reasons-lib-export-card-requires-reasons-export-card
Importing this module raises `ModuleNotFoundError` if `reasons.export_card` cannot be imported.
- Source: summaries/2026/11/19/reasons_lib-export_card.md
- Rejected: trivial: normal import dependency behavior, obvious from `from ... import *`.

### [REJECT] reasons-lib-export-card-reexports-only-public-names
Only names in `reasons.export_card.__all__` (or, if unset, non-underscore names) are exposed through this shim; symbols with leading underscores are not re-exported.
- Source: summaries/2026/11/19/reasons_lib-export_card.md
- Rejected: trivial: standard Python star-import semantics for `__all__` and underscore names.

### [REJECT] reasons-lib-export-card-preserves-identity
Re-exported objects are the same Python objects as their `reasons.export_card` counterparts, not copies or wrappers.
- Source: summaries/2026/11/19/reasons_lib-export_card.md
- Rejected: trivial: Python imports preserve object identity by definition; not a durable code invariant.

### [REJECT] reasons-lib-export-card-has-two-consumers
`reasons/api.py` and `tests/test_export_card.py` both import through this path, so removing the shim requires migrating both.
- Source: summaries/2026/11/19/reasons_lib-export_card.md
- Rejected: ephemeral: specific consumer count/inventory changes with test/API edits and is not a durable invariant.

### [REJECT] reasons-lib-export-markdown-is-a-shim
The file contains no implementation; it re-exports every public name from `reasons.export_markdown`.
- Source: summaries/2026/11/19/reasons_lib-export_markdown.md
- Rejected: trivial: re-export-only shim behavior is directly visible from the file's imports/structure.

### [REJECT] export-markdown-shim-requires-reasons-export-markdown
Importing `reasons_lib.export_markdown` raises immediately if `reasons.export_markdown` cannot be imported; failures are reported at line 2 of the shim.
- Source: summaries/2026/11/19/reasons_lib-export_markdown.md
- Rejected: ephemeral: line-number detail is brittle and the underlying import dependency is obvious from the shim.

### [REJECT] reasons-api-depends-on-reasons-lib-export-markdown
The canonical `reasons` package still imports the exporter via the legacy `reasons_lib` path.
- Source: summaries/2026/11/19/reasons_lib-export_markdown.md
- Rejected: trivial: "module imports module" dependency is visible from imports and not a meaningful invariant.

### [ACCEPTED] rebinding-shim-attributes-does-not-affect-canonical-module
`reasons_lib.export_markdown` and `reasons.export_markdown` are distinct module objects sharing references; assigning to an attribute on the shim leaves the canonical module unchanged.
- Source: summaries/2026/11/19/reasons_lib-export_markdown.md

### [REJECT] export-markdown-shim-emits-no-deprecation-warning
Neither the import nor any access emits a warning; migration is enforced only by the docstring.
- Source: summaries/2026/11/19/reasons_lib-export_markdown.md
- Rejected: trivial: Obvious from reading the shim; "migration enforced only by docstring" is an editorial judgment, not a durable observable invariant.

### [REJECT] reasons-lib-hf-is-a-shim
`reasons_lib/hf.py` contains no logic of its own and exists solely to re-export `reasons.hf`.
- Source: summaries/2026/11/19/reasons_lib-hf.md
- Rejected: trivial: Equivalent to "module re-exports another module" — obvious from file structure/imports.

### [REJECT] reasons-lib-hf-reexports-via-star-import
The entire file body is a single `from reasons.hf import *` statement.
- Source: summaries/2026/11/19/reasons_lib-hf.md
- Rejected: trivial: "module X imports Y" style fact, directly visible from the source.

### [REJECT] reasons-lib-hf-suppresses-flake8-warnings
The import carries `# noqa: F401,F403` to keep flake8 clean for the intentionally-unused wildcard import.
- Source: summaries/2026/11/19/reasons_lib-hf.md
- Rejected: trivial: A code-comment detail any developer sees from reading the file.

### [REJECT] reasons-lib-hf-propagates-all-errors
Any exception during `import reasons.hf` propagates unchanged through `reasons_lib.hf`; no error handling exists locally.
- Source: summaries/2026/11/19/reasons_lib-hf.md
- Rejected: trivial: Trivial consequence of the shim having no logic/try-except; obvious from structure.

### [ACCEPTED] reasons-api-imports-from-reasons-lib-hf
`reasons/api.py` depends on `reasons_lib.hf`, not directly on `reasons.hf`, creating a dependency on the shim even inside the new package.
- Source: summaries/2026/11/19/reasons_lib-hf.md

### [ACCEPTED] reasons-lib-import-agent-is-a-compat-shim
`reasons_lib/import_agent.py` defines no functions or classes; its entire body is a docstring plus one wildcard re-export from `reasons.import_agent`.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md

### [REJECT] reasons-lib-import-agent-requires-reasons-import-agent
Importing `reasons_lib.import_agent` necessarily executes `reasons/import_agent.py` in full, including its transitive imports.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md
- Rejected: trivial: restates standard Python import semantics (importing a module executes it), observable from the import statement

### [ACCEPTED] reasons-api-requires-reasons-lib-import-agent
`reasons/api.py` and `reasons/pg.py` import the legacy shim, creating a `reasons → reasons_lib → reasons.import_agent` import chain.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md

### [REJECT] star-import-drops-private-names
Any `_`-prefixed symbol in `reasons/import_agent.py` is unreachable through `reasons_lib.import_agent`, because the wildcard import filters underscore names.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md
- Rejected: trivial: tautological consequence of Python wildcard-import behavior (underscore filtering), not a codebase-specific invariant

### [REJECT] shim-binds-symbols-by-reference
Exported objects are the same objects as in the canonical module (`is` identity holds), but rebinding an attribute on either module does not affect the other.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md
- Rejected: trivial: standard wildcard-import binding semantics visible from imports

### [ACCEPTED] reasons-lib-import-beliefs-is-a-compatibility-shim
This file exists only to keep old `reasons_lib.import_beliefs` imports working.
- Source: summaries/2026/11/19/reasons_lib-import_beliefs.md

### [REJECT] reasons-lib-import-beliefs-requires-reasons-import-beliefs
Importing this module requires the canonical `reasons.import_beliefs` module to be importable.
- Source: summaries/2026/11/19/reasons_lib-import_beliefs.md
- Rejected: trivial: dependency is evident from the import statement

### [REJECT] reasons-lib-import-beliefs-propagates-canonical-import-errors
It has no exception handling, so canonical import failures propagate unchanged.
- Source: summaries/2026/11/19/reasons_lib-import_beliefs.md
- Rejected: trivial: absence of exception handling and normal import error propagation are obvious implementation details

### [REJECT] reasons-lib-import-beliefs-has-no-explicit-all
The shim does not define `__all__`; its exported names come from the wildcard import on line 2.
- Source: summaries/2026/11/19/reasons_lib-import_beliefs.md
- Rejected: trivial: absence of `__all__` and wildcard import are visible file-structure details; line number is ephemeral

### [REJECT] reasons-lib-import-beliefs-star-imports-canonical-public-api
The only runtime operation is `from reasons.import_beliefs import *`.
- Source: summaries/2026/11/19/reasons_lib-import_beliefs.md
- Rejected: trivial: "module X only does a star import" is directly visible from file structure and imports.

### [REJECT] reasons-lib-llm-is-a-shim
`reasons_lib/llm.py` contains no implementation logic and only re-exports names from `reasons.llm`.
- Source: summaries/2026/11/19/reasons_lib-llm.md
- Rejected: trivial: "re-exports names from reasons.llm" is a trivial module-imports-module fact seen from the file's imports.

### [REJECT] reasons-lib-llm-requires-reasons-llm
Importing `reasons_lib.llm` requires `reasons.llm` to be importable.
- Source: summaries/2026/11/19/reasons_lib-llm.md
- Rejected: trivial: This is a direct and obvious consequence of the import statement, not a durable non-obvious constraint.

### [REJECT] reasons-llm-owns-public-api
The public API exposed through this shim is determined by `reasons.llm` and its `__all__` or public non-underscore names.
- Source: summaries/2026/11/19/reasons_lib-llm.md
- Rejected: trivial: Restates standard Python star-import/`__all__` semantics that any developer sees from the source.

### [REJECT] star-import-excludes-underscore-names
Names beginning with `_` in `reasons.llm` are not re-exported by this shim unless explicitly included in `reasons.llm.__all__`.
- Source: summaries/2026/11/19/reasons_lib-llm.md
- Rejected: trivial: Pure Python language semantics (underscore names excluded from star import unless in `__all__`), not a codebase invariant.

### [REJECT] shim-propagates-import-errors
This module has no error handling, so failures in `reasons.llm` propagate as import exceptions.
- Source: summaries/2026/11/19/reasons_lib-llm.md
- Rejected: trivial: Obvious consequence of a wildcard-import shim having no error handling; visible from file structure.

### [REJECT] reasons-lib-mcp-client-delegates-to-reasons-mcp-client
This file re-exports every public symbol from `reasons.mcp_client` via a single wildcard import and defines nothing of its own.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md
- Rejected: trivial: Wildcard re-export with no own definitions is obvious from the single import line.

### [ACCEPTED] reasons-is-canonical-reasons-lib-is-legacy
The module docstring directs callers to `reasons.mcp_client`, establishing `reasons` as the canonical package and `reasons_lib` as the compatibility layer.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md

### [REJECT] reasons-cli-requires-shim
`reasons/cli.py` imports this shim rather than `reasons.mcp_client` directly, so removing the shim without updating the CLI would break imports.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md
- Rejected: trivial: Import relationship directly visible in the CLI's imports; removing a module breaking its importers is self-evident.

### [REJECT] shim-has-no-error-handling
The file has no `try/except`; `ImportError` from `reasons.mcp_client` propagates unchanged.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md
- Rejected: trivial: Absence of try/except is directly visible from the file and not a durable architectural invariant.

### [REJECT] reasons-lib-mcp-server-is-a-pure-shim
The file defines no functions, classes, or constants; its entire contents are one `from reasons.mcp_server import *` statement.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: Describes file contents ("entire contents are one statement") — equivalent to the "module X imports Y" example; a developer sees this from the file's single import line.

### [REJECT] reasons-lib-mcp-server-requires-reasons-mcp-server
Importing this module fails with the same exception (e.g. `ModuleNotFoundError` for the `mcp` SDK) as importing `reasons.mcp_server` directly, since there is no fallback.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: Direct consequence of the wildcard re-export statement; "importing this module imports that module" is an obvious fact visible from imports.

### [REJECT] wildcard-reexport-omits-underscore-names
Names beginning with `_` in `reasons/mcp_server.py` are not accessible via `reasons_lib.mcp_server`.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: Standard Python `import *` language semantics, not a codebase-specific invariant.

### [REJECT] wildcard-reexport-drops-all-attribute
`reasons_lib.mcp_server` has no `__all__` attribute even if `reasons.mcp_server` declares one.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: Standard Python `import *` semantics (does not propagate `__all__`), not a codebase fact.

### [REJECT] shim-preserves-object-identity-but-not-binding-identity
`reasons_lib.mcp_server.X is reasons.mcp_server.X` holds for public `X`, but `monkeypatch.setattr` on the shim does not affect the real module.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: General Python module/binding semantics (rebinding the shim's name doesn't affect the source module); not a durable codebase invariant.

### [ACCEPTED] importing-shim-triggers-server-registration-side-effects
Any import of `reasons_lib.mcp_server` executes `reasons/mcp_server.py`'s module scope (server construction, tool registration) exactly once per process.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md

### [REJECT] test-mcp-server-depends-on-shim
`tests/test_mcp_server.py` is the only listed importer of this file, so the canonical `reasons/mcp_server.py` is currently exercised through the alias.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: ephemeral: importer inventory/test arrangement is a point-in-time snapshot, not a durable code invariant

### [ACCEPTED] reasons-lib-metadata-is-a-bc-shim
`reasons_lib/metadata.py` contains no implementation and exists solely to re-export the public API of `reasons.metadata` for backward compatibility.
- Source: summaries/2026/11/19/reasons_lib-metadata.md

### [ACCEPTED] reasons-metadata-is-canonical
The module docstring designates `reasons.metadata` as the canonical import path; `reasons_lib.metadata` is the compatibility alias.
- Source: summaries/2026/11/19/reasons_lib-metadata.md

### [ACCEPTED] reasons-lib-metadata-imports-from-reasons-metadata
The single statement `from reasons.metadata import *` creates a one-way compile-time dependency from `reasons_lib` to `reasons`, so `reasons.metadata` must not import `reasons_lib.metadata` (would be circular).
- Source: summaries/2026/11/19/reasons_lib-metadata.md

### [REJECT] noqa-suppresses-f401-and-f403
The trailing `# noqa: F401,F403` suppresses "unused import" and "wildcard import" lint diagnostics, both of which are intentional for this shim pattern.
- Source: summaries/2026/11/19/reasons_lib-metadata.md
- Rejected: trivial: lint suppression is self-evident from the `noqa` comment.

### [REJECT] reasons-lib-mind-service-is-a-shim
`reasons_lib/mind_service.py` contains no logic and only forwards names from `reasons/mind_service.py` via a star import.
- Source: summaries/2026/11/19/reasons_lib-mind_service.md
- Rejected: trivial: being only a star-import shim is visible from file structure.

### [ACCEPTED] mind-service-symbols-are-identity-equal-across-paths
Names imported from `reasons_lib.mind_service` and `reasons.mind_service` resolve to the identical objects, so `isinstance` and equality checks work across both paths.
- Source: summaries/2026/11/19/reasons_lib-mind_service.md

### [ACCEPTED] reasons-api-requires-legacy-shim
`reasons/api.py` imports `reasons_lib.mind_service`, so removing the shim would break the in-repo API module.
- Source: summaries/2026/11/19/reasons_lib-mind_service.md

### [REJECT] reasons-lib-network-is-shim
`reasons_lib/network.py` defines no code of its own; its only statement is `from reasons.network import *`, forwarding all public names from the canonical module.
- Source: summaries/2026/11/19/reasons_lib-network.md
- Rejected: trivial: being only a star-import shim is visible from file structure.

### [ACCEPTED] reasons-internal-requires-reasons-lib-shim
Several `reasons/*` modules (`api.py`, `storage.py`, `compact.py`, `check_stale.py`, `export_card.py`, `export_markdown.py`, `import_agent.py`, `import_beliefs.py`) still import from `reasons_lib.network`, so removing this file would break the canonical package itself, not just tests.
- Source: summaries/2026/11/19/reasons_lib-network.md

### [REJECT] shim-exports-only-public-names
Because `import *` skips underscore-prefixed names and honors `reasons.network.__all__` if defined, private names in `reasons/network.py` are not accessible through the shim.
- Source: summaries/2026/11/19/reasons_lib-network.md
- Rejected: trivial: Describes standard Python `import *`/`__all__` semantics rather than a codebase-specific invariant.

### [ACCEPTED] reasons-lib-pg-is-a-re-export-shim
`reasons_lib/pg.py` contains no logic beyond a docstring and `from reasons.pg import *`.
- Source: summaries/2026/11/19/reasons_lib-pg.md

### [ACCEPTED] reasons-lib-pg-requires-reasons-pg
Importing `reasons_lib.pg` fails at import time if `reasons.pg` is unimportable.
- Source: summaries/2026/11/19/reasons_lib-pg.md

### [REJECT] reasons-lib-pg-honors-reasons-pg-all
Because it uses `import *`, the set of names exposed depends on `reasons.pg.__all__` if defined.
- Source: summaries/2026/11/19/reasons_lib-pg.md
- Rejected: trivial: Restates standard `import *`/`__all__` behavior with no codebase-specific consequence.

### [REJECT] reasons-lib-pg-suppresses-lint-warnings
The `# noqa: F401,F403` comment intentionally disables flake8 unused-import and star-import warnings.
- Source: summaries/2026/11/19/reasons_lib-pg.md
- Rejected: trivial: A lint-suppression comment is an obvious implementation detail visible from the file, not a durable architectural invariant.

### [ACCEPTED] reasons-lib-pg-is-a-backwards-compatibility-alias
The docstring directs new code to `reasons.pg`, meaning `reasons_lib.pg` exists only to preserve older import paths.
- Source: summaries/2026/11/19/reasons_lib-pg.md

### [REJECT] reasons-lib-propose-update-is-a-shim
`reasons_lib/propose_update.py` contains no logic; it is a pure wildcard re-export of `reasons.propose_update`.
- Source: summaries/2026/11/19/reasons_lib-propose_update.md
- Rejected: trivial: "Module wildcard re-exports another module" is exactly the kind of fact visible from imports/file structure.

### [REJECT] propose-update-shim-requires-canonical-module
Importing the shim requires `reasons.propose_update` to be importable, and any failure there propagates with no interception.
- Source: summaries/2026/11/19/reasons_lib-propose_update.md
- Rejected: trivial: That a wildcard re-export imports its source (and propagates import errors) is an obvious consequence of how imports work.

### [REJECT] propose-update-public-surface-maps-to-upstream-all
The shim's public API is exactly the public surface of `reasons.propose_update` (its `__all__` if defined, otherwise all non-underscore names).
- Source: summaries/2026/11/19/reasons_lib-propose_update.md
- Rejected: trivial: Restating that a `from x import *` shim exposes the upstream public surface is evident from the import itself.

### [REJECT] shim-defines-no-all
The shim does not declare `__all__`, so further wildcard re-exports scan its post-wildcard namespace rather than a curated list.
- Source: summaries/2026/11/19/reasons_lib-propose_update.md
- Rejected: trivial: absence of `__all__` is visible directly from the file structure; the downstream namespace-scanning consequence is a minor detail of that obvious fact.

### [ACCEPTED] reasons-lib-pubsub-is-a-compatibility-shim
The module contains no implementation; it exists solely to re-export symbols from `reasons.pubsub` for legacy import paths.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md

### [REJECT] reasons-lib-pubsub-reexports-via-star-import
The entire public API surface is inherited through the single statement `from reasons.pubsub import *`.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md
- Rejected: trivial: "module X imports Y via star" is exactly the kind of fact obvious from imports.

### [REJECT] reasons-lib-pubsub-requires-reasons-pubsub
Importing `reasons_lib.pubsub` fails if `reasons.pubsub` cannot be imported; there is no fallback implementation.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md
- Rejected: trivial: "import fails if the target module can't be imported" is inherent to any import statement and obvious from the star import.

### [REJECT] reasons-api-imports-reasons-lib-pubsub
`reasons/api.py` reaches pubsub functionality through this legacy shim rather than importing `reasons.pubsub` directly.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md
- Rejected: trivial: a module-to-module import fact, directly visible from imports.

### [REJECT] reasons-lib-pubsub-has-no-own-symbols
The module defines no functions, classes, or constants of its own; any future local definition would violate the pure-alias invariant.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md
- Rejected: trivial: Pure-alias/empty-module fact visible directly from imports and file structure; the "future definition would violate invariant" part is editorial/speculative.

### [REJECT] reasons-lib-repair-re-exports-reasons-repair
This module defines no symbols locally; its entire namespace is produced by `from reasons.repair import *`.
- Source: summaries/2026/11/19/reasons_lib-repair.md
- Rejected: trivial: Directly stated by the `from reasons.repair import *` line; an import-shape fact any developer sees.

### [REJECT] reasons-lib-repair-requires-reasons-repair
Importing `reasons_lib.repair` transitively imports `reasons.repair`; an import error there propagates with no fallback or wrapping.
- Source: summaries/2026/11/19/reasons_lib-repair.md
- Rejected: trivial: Bare import propagation is obvious Python behavior visible from the import statement.

### [REJECT] reasons-api-imports-reasons-lib-repair
The canonical `reasons/api.py` consumes the legacy shim rather than importing `reasons.repair` directly.
- Source: summaries/2026/11/19/reasons_lib-repair.md
- Rejected: trivial: "Module X imports Y" dependency fact visible from the import; no durable architectural invariant beyond that.

### [REJECT] reasons-lib-repair-is-a-compat-shim
The docstring identifies the module as a backwards-compatibility shim and directs new code to `reasons.repair`.
- Source: summaries/2026/11/19/reasons_lib-repair.md
- Rejected: trivial: Restates the module's name/docstring; obvious from naming and the file itself.

### [REJECT] reasons-lib-repair-preserves-underscore-names-only-via-all
Names in `reasons.repair` beginning with `_` are re-exported here only if `reasons.repair.__all__` lists them; otherwise they are dropped by the wildcard import.
- Source: summaries/2026/11/19/reasons_lib-repair.md
- Rejected: trivial: Restates standard Python wildcard-import semantics (underscore names skipped unless in `__all__`); any developer knows this from import behavior.

### [ACCEPTED] reasons-lib-repair-premises-is-a-shim
The module defines no functions or classes; its entire body is a docstring plus one star import from `reasons.repair_premises`.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md

### [REJECT] reasons-lib-repair-premises-requires-reasons-package
Importing this module fails with `ModuleNotFoundError` unless the sibling `reasons` package is importable.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md
- Rejected: trivial: Merely restates that the star import depends on the `reasons` package being importable — an obvious import-dependency fact.

### [REJECT] star-import-does-not-forward-private-names
Names beginning with `_` in `reasons/repair_premises.py` are not accessible via `reasons_lib.repair_premises`.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md
- Rejected: trivial: Basic Python fact that wildcard imports skip names beginning with `_`; duplicates the same standard-language behavior claim.

### [ACCEPTED] shim-and-origin-are-distinct-module-objects
Monkeypatching `reasons_lib.repair_premises.<name>` does not affect `reasons.repair_premises.<name>`, so tests must patch the origin module.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md

### [ACCEPTED] reasons-api-imports-legacy-shim
`reasons/api.py` depends on the legacy namespace, creating a back-edge from the canonical package to the compatibility layer.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md

### [REJECT] reasons-lib-report-belief-is-a-compat-shim
The module contains no logic of its own; it is a docstring plus a single star-import re-export.
- Source: summaries/2026/11/19/reasons_lib-report_belief.md
- Rejected: trivial: A module being "a docstring plus a star-import re-export" is directly visible from file structure and imports; no durable invariant beyond that.

### [REJECT] reasons-lib-report-belief-requires-reasons-report-belief
The shim cannot import without `reasons.report_belief` being importable; failure there bubbles up unchanged.
- Source: summaries/2026/11/19/reasons_lib-report_belief.md
- Rejected: trivial: The import dependency and that a shim's import failure propagates is obvious from the single re-export structure.

### [REJECT] reasons-lib-report-belief-reexports-public-names
It forwards only public names (per `__all__` or non-underscore convention) of `reasons.report_belief`, sharing object identity rather than copying.
- Source: summaries/2026/11/19/reasons_lib-report_belief.md
- Rejected: trivial: Re-export behavior (star-import sharing object identity) is inherent and obvious from the import statement; "per __all__ or non-underscore convention" is vague/unverified.

### [REJECT] reasons-lib-report-belief-enables-legacy-imports
It preserves `from reasons_lib.report_belief import X` callers during the `reasons_lib` → `reasons` rename.
- Source: summaries/2026/11/19/reasons_lib-report_belief.md
- Rejected: trivial: The backward-compat purpose of a shim is obvious from naming and file structure (reasons_lib → reasons rename).

### [REJECT] reasons-api-imports-legacy-shim-path
`reasons/api.py` imports from `reasons_lib.report_belief` (the legacy path) rather than `reasons.report_belief`, indicating a partially migrated import.
- Source: summaries/2026/11/19/reasons_lib-report_belief.md
- Rejected: trivial: "module X imports Y" import-path fact observable from file structure; the "partially migrated" framing is editorial.

### [REJECT] reasons-lib-report-gated-is-a-shim
`reasons_lib/report_gated.py` defines no logic and only re-exports symbols from `reasons.report_gated`.
- Source: summaries/2026/11/19/reasons_lib-report_gated.md
- Rejected: trivial: Describes file structure ("defines no logic, re-exports") directly visible from the module's imports.

### [REJECT] reasons-lib-report-gated-requires-reasons-report-gated
This file's import succeeds only if `reasons.report_gated` imports successfully; failures propagate unchanged.
- Source: summaries/2026/11/19/reasons_lib-report_gated.md
- Rejected: trivial: Standard Python import propagation; obvious from the star-import structure, not a deliberate invariant.

### [REJECT] reasons-lib-report-gated-preserves-object-identity
Symbols re-exported via star import are the same objects as those in `reasons.report_gated`, so `is` and `isinstance` checks hold across both paths.
- Source: summaries/2026/11/19/reasons_lib-report_gated.md
- Rejected: trivial: Object identity under star imports is a fundamental Python import semantics fact, obvious to any developer.

### [REJECT] reasons-api-depends-on-reasons-lib-shim
`reasons/api.py` still imports from `reasons_lib.report_gated`, so deleting this shim would break the API layer.
- Source: summaries/2026/11/19/reasons_lib-report_gated.md
- Rejected: trivial: "reasons/api.py imports from reasons_lib.report_gated" is an import-structure fact; the "deleting would break" conclusion is a trivial inference.

### [REJECT] reasons-lib-report-gated-exports-canonical-public-names
The names available here equal the public namespace (or `__all__`) of `reasons.report_gated`; underscore-prefixed names are not exposed.
- Source: summaries/2026/11/19/reasons_lib-report_gated.md
- Rejected: trivial: ordinary wildcard re-export detail, observable from the shim's import.

### [REJECT] review-shim-is-pure-reexport
`reasons_lib/review.py` defines no functions, classes, or constants of its own; its only executable statement is a wildcard import from `reasons.review`.
- Source: summaries/2026/11/19/reasons_lib-review.md
- Rejected: trivial: structural file detail; "only a wildcard import" is obvious from the module body.

### [REJECT] review-shim-requires-reasons-review
Importing `reasons_lib.review` necessarily imports the `reasons` package and the `reasons.review` module first, running their full module bodies.
- Source: summaries/2026/11/19/reasons_lib-review.md
- Rejected: trivial: follows directly from the wildcard import and Python import semantics.

### [REJECT] star-import-binds-snapshot-not-proxy
Names are copied at import time, so rebinding a name in `reasons.review` afterward does not change `reasons_lib.review`'s attribute, though mutation of the shared object is visible through both.
- Source: summaries/2026/11/19/reasons_lib-review.md
- Rejected: trivial: general Python import semantics, not a codebase invariant.

### [ACCEPTED] canonical-package-imports-legacy-shim
`reasons/api.py`, `reasons/repair.py`, and `reasons/review_justifications.py` import `reasons_lib.review`, creating a cross-tree import edge from the new package back to the legacy one.
- Source: summaries/2026/11/19/reasons_lib-review.md

### [REJECT] review-shim-suppresses-f401-f403
The `# noqa: F401,F403` comment is required for linters to accept the otherwise-unused wildcard import.
- Source: summaries/2026/11/19/reasons_lib-review.md
- Rejected: trivial: Lint-comment detail, not a durable code invariant.

### [ACCEPTED] reasons-lib-is-legacy-name
`reasons_lib/review_justifications.py` is a compatibility shim; the canonical module is `reasons/review_justifications.py`.
- Source: summaries/2026/11/19/reasons_lib-review_justifications.md

### [ACCEPTED] review-justifications-shim-re-exports-all-public-names
The shim exposes the full public API of the canonical module via `from reasons.review_justifications import *`, and defines no `__all__` of its own.
- Source: summaries/2026/11/19/reasons_lib-review_justifications.md

### [REJECT] review-justifications-shim-has-no-own-logic
The shim contains no classes, functions, or constants defined in-file; any behavior must live in `reasons/review_justifications.py`.
- Source: summaries/2026/11/19/reasons_lib-review_justifications.md
- Rejected: trivial: Merely restates that it is a compatibility shim; visible from file structure.

### [REJECT] reasons-api-imports-through-shim
`reasons/api.py` depends on the legacy `reasons_lib.review_justifications` path rather than the canonical one.
- Source: summaries/2026/11/19/reasons_lib-review_justifications.md
- Rejected: trivial: A single import/dependency line, obvious from imports.

### [REJECT] shim-import-failure-propagates-as-canonical-failure
Importing the shim raises exactly the exception from `reasons.review_justifications`; there is no `try/except` fallback.
- Source: summaries/2026/11/19/reasons_lib-review_justifications.md
- Rejected: trivial: default import behavior of a one-line re-export shim; no fallback is obvious from the import structure.

### [REJECT] shim-forwards-to-canonical-module
`reasons_lib/review_premises.py` contains exactly one statement, `from reasons.review_premises import *`, re-exporting the canonical module's public names.
- Source: summaries/2026/11/19/reasons_lib-review_premises.md
- Rejected: trivial: a shim file containing `from reasons.review_premises import *` is an obvious import/file-structure fact.

### [REJECT] no-logic-in-shim
The shim defines no functions, classes, or constants of its own; all behavior lives in `reasons.review_premises`.
- Source: summaries/2026/11/19/reasons_lib-review_premises.md
- Rejected: trivial: absence of local definitions follows directly from the shim being a re-export.

### [REJECT] identity-preserved-across-paths
Objects imported via `reasons_lib.review_premises` are the same objects as those imported via `reasons.review_premises`.
- Source: summaries/2026/11/19/reasons_lib-review_premises.md
- Rejected: trivial: Python re-exports via `import *` preserve object identity; obvious from the import mechanism.

### [REJECT] api-depends-on-legacy-path
`reasons/api.py` imports through this leg
- Source: summaries/2026/11/19/reasons_lib-review_premises.md
- Rejected: trivial: module import dependency is directly visible from imports and the claim is incomplete.

### [REJECT] reasons-lib-storage-is-compat-shim
`reasons_lib/storage.py` contains only a docstring and a wildcard import from `reasons.storage`; it defines no local storage logic.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: describes only the shim file’s docstring/import structure, which is obvious from the file.

### [REJECT] reasons-lib-storage-requires-reasons-storage
Importing `reasons_lib.storage` fails if `reasons.storage` cannot be imported.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: import-failure dependence is a direct consequence of the wildcard import.

### [REJECT] reasons-lib-storage-re-exports-public-names
Public names exported by `reasons.storage` are bound into the `reasons_lib.storage` namespace at import time.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: standard wildcard-import behavior, not a durable architectural/API invariant.

### [REJECT] reasons-lib-storage-has-no-all
This shim does not define `__all__`, so downstream star imports are governed by normal public-name filtering.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: a minor module-structure detail with no meaningful invariant value.

### [REJECT] reasons-lib-storage-does-not-catch-import-errors
Errors raised while importing `reasons.storage` propagate unchanged through this module.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: propagation of import errors is inherent to the simple re-export shim.

### [ACCEPTED] reasons-lib-verify-is-a-shim
`reasons_lib/verify.py` defines no functions or classes of its own and consists only of a docstring plus a wildcard import from `reasons.verify`.
- Source: summaries/2026/11/19/reasons_lib-verify.md

### [REJECT] reasons-lib-verify-forwards-to-reasons-verify
`reasons_lib.verify` re-exports every public name of `reasons.verify` at import time via `from reasons.verify import *`.
- Source: summaries/2026/11/19/reasons_lib-verify.md
- Rejected: duplicate: essentially the same wildcard-reexport claim already made by reasons-lib-verify-is-a-shim

### [REJECT] reasons-verify-is-the-canonical-verifier
The implementation of the verification logic lives in `reasons/verify.py`; `reasons_lib/verify.py` is the legacy alias.
- Source: summaries/2026/11/19/reasons_lib-verify.md
- Rejected: duplicate: restates the shim relationship captured by reasons-lib-verify-is-a-shim with interpretive "canonical/legacy" framing

### [REJECT] tests-is-a-python-package
The presence of this empty file makes `tests` an importable regular package.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: trivial: obvious fact any developer can see from the presence of the empty file

### [REJECT] tests-package-marker-contains-no-code
The file defines no functions, classes, or constants, and has no import-time side effects.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: trivial: trivially true for an empty file; no durable insight

### [REJECT] tests-init-enables-dotted-imports
It allows test modules to be imported as `tests.<module>`, avoiding name collisions across the many `test_*.py` files.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: trivial: Standard Python package behavior; obvious from file structure.

### [REJECT] tests-init-requires-conftest-for-fixtures
Shared fixtures are provided by `tests/conftest.py`, not by this file, so `__init__.py` must stay empty to avoid interfering with pytest's fixture loading.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: speculative: Editorial claim about what the file "must" do; not an observable code fact.

### [REJECT] tests-init-is-not-imported-by-production-code
Neither `reasons/` nor `reasons_lib/` depends on the `tests` package.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: trivial: Obvious from imports/file structure that production code doesn't import tests.

### [REJECT] conftest-fixtures-are-auto-injected
The `pg_api` fixture is exposed to all test modules under `tests/` without any explicit `import`, via pytest's conftest auto-discovery.
- Source: summaries/2026/11/19/tests-conftest.md
- Rejected: trivial: Standard, well-known pytest conftest auto-discovery behavior.

### [REJECT] pg-marker-requires-pytest-configure
The `pg` marker is registered by `pytest_configure`; without that hook, `@pytest.mark.pg` would trigger unknown-marker warnings.
- Source: summaries/2026/11/19/tests-conftest.md
- Rejected: trivial: Standard pytest marker-registration mechanics, obvious to any developer.

### [ACCEPTED] pg-api-fixture-requires-database-url
`pg_api` skips (does not fail) when `DATABASE_URL` is unset, via `pytest.skip` inside the fixture body.
- Source: summaries/2026/11/19/tests-conftest.md

### [ACCEPTED] ldomain-isolation-uses-uuid4
Each `pg_api` invocation mints a UUID4 `domain_id`, and all rows created during the test are scoped to that ID, guaranteeing cross-test isolation without table truncation.
- Source: summaries/2026/11/19/tests-conftest.md

### [ACCEPTED] cleanup-order-respects-foreign-keys
The teardown `DELETE` loop orders tables leaves-first (`rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_nodes`) to satisfy foreign-key constraints.
- Source: summaries/2026/11/19/tests-conftest.md

### [ACCEPTED] rollback-prevents-aborted-transaction-errors
`api.conn.rollback()` at the start of teardown resets failed-transaction state left by the test so the subsequent `DELETE`s succeed.
- Source: summaries/2026/11/19/tests-conftest.md

### [REJECT] pg-api-import-is-deferred
`from reasons.pg import PgApi` is executed inside the fixture body (not at module top) to avoid importing DB drivers at collection time when `DATABASE_URL` is absent.
- Source: summaries/2026/11/19/tests-conftest.md
- Rejected: trivial: implementation detail of import placement in the fixture, visible from the file and not a durable behavioral invariant

### [REJECT] derived-nodes-union-access-tags
A derived node’s `access_tags` is the union of its explicit tags and all antecedent tags across its justifications, and adding a justification can propagate new tags to dependents.
- Source: summaries/2026/11/19/tests-test_access_tags.md
- Rejected: duplicate: Same claim already captured by `access-tags-union-inheritance` and `add-justification-propagates-tags-downstream`.

### [REJECT] visible-to-subset-semantics
A tagged node is visible only when all of its tags are contained in `visible_to`; untagged nodes are always visible and omitting `visible_to` disables filtering.
- Source: summaries/2026/11/19/tests-test_access_tags.md
- Rejected: duplicate: Core subset-visibility claim already covered by `access-tags-subset-gate` / `access-tags-require-subset-for-visibility`.

### [REJECT] restricted-show-explain-trace-raise-permissionerror
`show_node`, `explain_node`, `trace_assumptions`, and `api.trace_access_tags` raise `PermissionError` when the requested node is not visible.
- Source: summaries/2026/11/19/tests-test_access_tags.md
- Rejected: duplicate: Already covered by `api-hidden-node-raises-permissionerror`.

### [ACCEPTED] listing-search-export-compact-filter-silently
`list_nodes`, `lookup`, `search`, `get_status`, `export_network`, `export_markdown`, and `compact` exclude invisible data without raising.
- Source: summaries/2026/11/19/tests-test_access_tags.md

### [ACCEPTED] export-network-drops-restricted-nogoods
`export_network` removes a nogood when any referenced node is invisible to the caller.
- Source: summaries/2026/11/19/tests-test_access_tags.md

### [ACCEPTED] network-add-justification-returns-truth-transition-metadata
`Network.add_justification` returns a dict containing `node_id`, `old_truth_value`, `new_truth_value`, and `changed`.
- Source: summaries/2026/11/19/tests-test_add_justification.md

### [REJECT] api-add-justification-requires-a-justification-argument
`api.add_justification` raises `ValueError` matching `"Must provide"` when none of `sl`, `cp`, or `unless` is supplied.
- Source: summaries/2026/11/19/tests-test_add_justification.md
- Rejected: duplicate: Covered by `api-add-justification-requires-justification-arg`/`-kwarg` which already state ValueError is raised when none of sl/cp/unless is provided.

### [REJECT] add-justification-registers-antecedent-and-outlist-dependents
Adding a justification registers the target node in the dependents of each antecedent and outlist node.
- Source: summaries/2026/11/19/tests-test_add_justification.md
- Rejected: duplicate: Covered by existing `add-justification-registers-dependents`, which states dependents are updated for both antecedent and outlist nodes.

### [ACCEPTED] multiple-justifications-are-alternative-support-paths
A node with two valid justifications remains `IN` after one supporting antecedent is retracted.
- Source: summaries/2026/11/19/tests-test_add_justification.md

### [REJECT] namespaced-add-justification-returns-qualified-node-id
With `namespace="ns"`, `api.add_justification` returns `node_id == "ns:b"`.
- Source: summaries/2026/11/19/tests-test_add_justification.md
- Rejected: duplicate: Covered by existing `api-add-justification-namespace-prefixes-result-id`, which captures namespace prefixing of the returned node_id.

### [REJECT] any-mode-expands-conjunction-into-separate-justifications
`add_node(..., any_mode=True)` with an N-premise comma-separated `sl` produces N justifications, each containing exactly one antecedent.
- Source: summaries/2026/11/19/tests-test_any_mode.md
- Rejected: duplicate: Already captured by existing `any-mode-creates-per-premise-justifications`.

### [REJECT] any-mode-copies-outlist-to-each-expanded-justification
When `unless="enemy"` is supplied alongside `any_mode=True`, every expanded justification contains `"enemy"` in its `outlist`.
- Source: summaries/2026/11/19/tests-test_any_mode.md
- Rejected: duplicate: Already captured by `any-mode-outlist-preserved` and `any-mode-preserves-full-outlist-semantics`.

### [ACCEPTED] add-node-returns-premise-count
`api.add_node` returns a dict whose `"premise_count"` key holds the antecedent count for the created SL, and `0` for premise nodes.
- Source: summaries/2026/11/19/tests-test_any_mode.md

### [ACCEPTED] any-mode-premise-count-is-one
With `any_mode=True`, `add_node`'s returned `"premise_count"` is `1` (per unary justification), not the total number of expansions.
- Source: summaries/2026/11/19/tests-test_any_mode.md

### [ACCEPTED] retract-node-returns-restoration-hints
`api.retract_node` returns a dict with a `"restoration_hints"` list whose entries expose `node_id`, `all_premises`, and `surviving_premises`.
- Source: summaries/2026/11/19/tests-test_any_mode.md

### [REJECT] restoration-hints-skip-any-mode-nodes
Nodes whose justifications are all unary (as produced by `any_mode`) never appear in `restoration_hints`, because the hint gate requires a multi-premise SL.
- Source: summaries/2026/11/19/tests-test_any_mode.md
- Rejected: duplicate: Same claim as existing `any-mode-nodes-never-produce-restoration-hints` (any_mode nodes produce no restoration hints).

### [ACCEPTED] restoration-hints-exclude-directly-retracted-node
The node passed to `retract_node` is never included in its own `restoration_hints`, even if it has multiple premises.
- Source: summaries/2026/11/19/tests-test_any_mode.md

### [REJECT] retract-cascades-to-dependents
Retracting a node sets every transitive `sl=` dependent to `OUT`, and asserting it back restores the whole subtree symmetrically.
- Source: summaries/2026/11/19/tests-test_api.md
- Rejected: duplicate: Covered by existing `api-retract-cascade-is-transitive` and `api-retract-cascade-symmetry`.

### [ACCEPTED] accept-proposal-drift-yields-stale-not-error
When the target state changed between proposal and acceptance, `accept_proposal` returns `{"applied": False, "status": "stale"}` and does not mutate the store, rather than raising.
- Source: summaries/2026/11/19/tests-test_api.md

### [REJECT] fts-query-calls-are-bounded
A single `_fts_search` call for a 20-term query invokes `_fts_query` at most 51 times, evidencing a hard cap on progressive-relaxation fallback attempts.
- Source: summaries/2026/11/19/tests-test_api.md
- Rejected: duplicate: Same 51-call bound already captured by `api-fts-search-bounds-query-calls`.

### [ACCEPTED] cascade-does-not-set-retracted-at
`retracted_at` is populated only on nodes the user directly retracted; cascade-affected dependents end up `OUT` with an empty `retracted_at`.
- Source: summaries/2026/11/19/tests-test_api.md

### [ACCEPTED] access-tags-migrate-to-tag-namespace
Passing `access_tags=["internal"]` to `api.add_node` results in a stored tag `access:internal`, not a raw `access_tags` field.
- Source: summaries/2026/11/19/tests-test_api.md

### [REJECT] dual-requires-sources-db
`ask(..., dual=True)` without `sources_db` raises `ValueError` with `--dual requires --full-sources`.
- Source: summaries/2026/11/19/tests-test_ask.md
- Rejected: duplicate: existing `ask-dual-requires-sources-db` and `ask-dual-mode-requires-sources-db` already capture this.

### [REJECT] tool-call-loop-is-bounded
Full mode permits at most three `search_beliefs` tool rounds before a final synthesis call, as verified by call-count tests.
- Source: summaries/2026/11/19/tests-test_ask.md
- Rejected: duplicate: already covered by `ask-tool-call-budget-is-three` / `ask-tool-loop-capped-at-three`.

### [ACCEPTED] final-synthesis-falls-back-to-beliefs
If final synthesis times out or returns another tool call, `ask()` returns accumulated belief context rather than raw `search_beliefs` JSON.
- Source: summaries/2026/11/19/tests-test_ask.md

### [REJECT] natural-mode-strips-belief-metadata
When `natural=True`, prompts omit `**Status:**`, `###` headers, and citation instructions, and include `plain natural language`.
- Source: summaries/2026/11/19/tests-test_ask.md
- Rejected: duplicate: already covered by existing `ask-natural-mode-strips-metadata-and-cite` (metadata and citation stripping when natural=True).

### [REJECT] mcp-servers-bump-max-iterations-to-five
When `mcp_servers` is non-empty, `ask()` allows up to 5 tool rounds, so `invoke_model` is called at most 6 times.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: already covered by existing `ask-mcp-iteration-limit-is-five`.

### [REJECT] mcp-tool-errors-are-swallowed
Exceptions raised by `bridge.call_tool` are caught inside `ask()` (recorded in tool history) and never propagate to the caller.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: already covered by existing `ask-mcp-errors-non-fatal` / `ask-mcp-integration-is-safely-bounded`.

### [ACCEPTED] tool-dispatch-arguments-omit-tool-key
The dict passed as `arguments` to `bridge.call_tool(name, arguments)` is the parsed tool-call minus the `"tool"` key.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md

### [REJECT] tools-section-always-includes-search-beliefs
`_build_tools_section` includes the builtin `search_beliefs` tool even when the bridge list is empty.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: exact match of existing `build-tools-section-always-includes-search-beliefs`.

### [REJECT] empty-mcp-instructions-produce-no-prompt-section
`_build_mcp_instructions` returns `""` for empty instructions, and `build_ask_prompt` then omits the `"Data Source Instructions"` header entirely.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: same claim already covered by `build-mcp-instructions-concatenates-bridges` (returns empty string) and `ask-prompt-data-source-instructions-header` (omits header when no instructions).

### [ACCEPTED] trace-assumptions-returns-deduplicated-premise-ids
`Network.trace_assumptions` returns premise IDs reached by recursively following justifications and returns the node itself when it is a premise.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [ACCEPTED] find-culprits-orders-by-dependent-count
`Network.find_culprits` returns culprits sorted so a premise with fewer dependents appears before one with more dependents.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [REJECT] add-nogood-retracts-premises-not-derived-nodes
`Network.add_nogood` retracts a selected premise and cascades dependent derived nodes `OUT` rather than retracting a derived nogood member directly.
- Source: summaries/2026/11/19/tests-test_backtracking.md
- Rejected: duplicate: covered by `add-nogood-retraction-prefers-least-entrenched` / `backtracking-retracts-least-entrenched`, which already establish that retraction traces back to premises via justification chains.

### [REJECT] inactive-nogood-is-no-op
If any node in a nogood is already `OUT`, `Network.add_nogood` returns an empty changed list and emits no backtrack log entry.
- Source: summaries/2026/11/19/tests-test_backtracking.md
- Rejected: duplicate: identical to existing `add-nogood-inactive-when-either-node-out` and `add-nogood-inactive-contradiction-returns-empty`.

### [ACCEPTED] entrenchment-increases-with-source-observation-and-dependents
`_entrenchment` scores sourced premises, `OBSERVATION` metadata, and nodes with more dependents above their unsourced/derived/less-depended counterparts.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [REJECT] db-fixture-seeds-five-beliefs-through-api
The `db` fixture creates five nodes in a temporary SQLite database via `reasons.api.add_node`, including a support link from `deploy-ci-pipeline` to `testing-unit-coverage`.
- Source: summaries/2026/11/19/tests-test_build_wiki.md
- Rejected: ephemeral: specific fixture node count and test setup detail, not a durable architectural invariant

### [ACCEPTED] format-node-linkifies-known-references-only
`_format_node` renders cross-references as `[id](page.md#id)` only when the target ID exists in `node_to_page`; otherwise it leaves the ID as plain text.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [REJECT] build-wiki-falls-back-on-llm-errors
When `model` is supplied, `build_wiki` uses `generate_wiki_page`, but a `RuntimeError` from generation does not prevent static `_format_node` Markdown from being written.
- Source: summaries/2026/11/19/tests-test_build_wiki.md
- Rejected: duplicate: existing belief `build-wiki-llm-failure-falls-back-to-template` already captures LLM generation failure fallback

### [ACCEPTED] build-wiki-always-writes-index-md
`build_wiki` creates `index.md` even when there are no beliefs, and that page contains `"No beliefs found"` for an empty database.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [REJECT] linkify-replaces-longest-node-ids-first
`_linkify` processes node IDs longest-first so `node-alpha-extended` is linked as a single token rather than partially linking `node-alpha`.
- Source: summaries/2026/11/19/tests-test_build_wiki.md
- Rejected: duplicate: Same claim as existing `build-wiki-linkify-longest-first-prevents-partial-match`.

### [ACCEPTED] check-stale-returns-results-and-upgrade-count
`check_stale` returns a tuple whose first element is a stale-result list and whose second element is an upgraded-hash count.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] source-hash-is-sha256-hexdigest
`hash_file` and stored `source_hash` values are SHA-256 hex digests of file content.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] empty-source-hash-skips-stale-check
`check_stale` skips nodes with an empty `source_hash`, even if their source file is missing.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] missing-source-produces-source-deleted-result
`check_stale` reports a node whose resolved source file is missing with `reason == "source_deleted"`, `new_hash is None`, and `source_path is None`.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] prefix-hash-upgrade-requires-flag
A stored hash that is a prefix of the actual SHA-256 digest is only upgraded when `upgrade_hashes=True`; otherwise it is reported as `truncated_hash`.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] hash-sources-backfills-empty-hashes-unless-force
`hash_sources` updates nodes with empty `source_hash` by default and only rehashes existing non-empty hashes when `force=True`.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] missing-source-produces-source-deleted
A tracked node whose source file is absent produces a `check_stale` result with reason `source_deleted`.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [ACCEPTED] source-deleted-result-nulls-new-hash-and-source-path
`source_deleted` results set `new_hash` and `source_path` to `None` while preserving `old_hash` and `source`.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [ACCEPTED] check-stale-result-schema-is-exact
Every result dict must contain exactly `node_id`, `old_hash`, `new_hash`, `source`, `source_path`, and `reason`.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [ACCEPTED] check-stale-skips-ineligible-nodes
Nodes that are retracted, lack `source`, lack `source_hash`, or have `source == ""` are omitted from results.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [REJECT] defeat-reason-types-has-seven-entries
`DEFEAT_REASON_TYPES` contains exactly 7 members.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md
- Rejected: ephemeral: exact member count of a constant changes with edits and is a point-in-time count

### [ACCEPTED] defeat-reason-types-are-kebab-case
Every entry in `DEFEAT_REASON_TYPES` contains a hyphen.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] classify-defeat-reason-returns-empty-on-failure
The pure classifier returns `""` on subprocess exception and on unrecognized LLM output, never raising.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] dry-run-does-not-write-metadata
`classify_defeat_reason_types(..., dry_run=True)` never adds `defeat_reason_type` to node metadata.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] apply-mode-writes-defeat-reason-type-metadata
With `dry_run=False`, the classifier result is persisted at `node["metadata"]["defeat_reason_type"]`.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] already-classified-defeaters-are-skipped
Re-running classification skips nodes whose metadata already contains `defeat_reason_type`, reporting reason `"already classified"`.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] classification-failures-go-to-errors-list
Unclassifiable defeaters are collected in `result["errors"]` instead of being raised or written.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] defeater-type-filter-matches-existing-metadata
`defeater_type_filter` matches against the defeater node's existing `defeater_type`, not the new `defeat_reason_type`.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] run-cli-catches-systemexit
`run_cli()` catches `SystemExit` and returns its `.code`, so tests assert numeric exit codes instead of expecting process termination.
- Source: summaries/2026/11/19/tests-test_cli.md

### [REJECT] cli-tests-use-real-temporary-sqlite-db
Every DB-backed test uses a real SQLite file under pytest’s `tmp_path`, created either by `run_cli("init")` or by `TestProposalLifecycle`’s fixture.
- Source: summaries/2026/11/19/tests-test_cli.md
- Rejected: trivial: test-infrastructure detail obvious from the test code, and essentially duplicates existing test-DB beliefs like `api-tests-use-isolated-temp-sqlite`

### [REJECT] global-db-option-precedes-subcommand
`run_cli()` always inserts `--db <path>` immediately after `reasons` and before the command tokens.
- Source: summaries/2026/11/19/tests-test_cli.md
- Rejected: trivial: implementation detail of a test helper (`run_cli`) visible from its code, not a durable invariant

### [ACCEPTED] export-dash-emits-json-on-stdout
`export --output=-` is expected to print pure JSON to stdout, as verified by `json.loads(out)` in `test_export_json`.
- Source: summaries/2026/11/19/tests-test_cli.md

### [REJECT] proposal-lifecycle-fixture-preloads-three-nodes
`TestProposalLifecycle.db_path` initializes a DB with `a`, `b`, and `derived-ab` before each test in that class.
- Source: summaries/2026/11/19/tests-test_cli.md
- Rejected: ephemeral: point-in-time fixture setup describing a specific test graph, not a durable invariant

### [REJECT] semantic-dedupe-tests-skip-without-cluster-deps
`TestDeduplicateSemantic` is skipped when `reasons.cluster.HAS_CLUSTER_DEPS` is false.
- Source: summaries/2026/11/19/tests-test_cli.md
- Rejected: trivial: test skip condition obvious from the decorator, no architectural insight

### [REJECT] cluster-numeric-tests-run-without-extras
The three `_auto_k` tests and `test_require_cluster_deps_message` are the only tests without `@skip_no_cluster`; they exercise pure arithmetic or the missing-dep path.
- Source: summaries/2026/11/19/tests-test_cluster.md
- Rejected: ephemeral: test inventory with a count of unskipped tests that changes with test edits

### [ACCEPTED] cluster-beliefs-returns-exactly-budget
When `0 < budget < len(beliefs)`, `cluster_beliefs` returns exactly `budget` ids, not at most `budget`.
- Source: summaries/2026/11/19/tests-test_cluster.md

### [REJECT] cluster-selection-preserves-cross-domain-diversity
With a mixed auth/database corpus and `budget=20`, `cluster_beliefs` selects at least one belief from each domain.
- Source: summaries/2026/11/19/tests-test_cluster.md
- Rejected: ephemeral: fixture-specific test scenario rather than a durable general invariant

### [ACCEPTED] cluster-stats-account-for-all-beliefs
`sum(cluster_beliefs(...)[1]["cluster_sizes"])` equals the number of input beliefs, and stats include `n_clusters`, `cluster_sizes`, and `embedding_model`.
- Source: summaries/2026/11/19/tests-test_cluster.md

### [REJECT] auto-k-is-n-over-5-capped
`_auto_k(n)` defaults to `max(2, n // 5)`, `max_k` caps that default, and an explicit `n_clusters` overrides it but is clamped to `n`.
- Source: summaries/2026/11/19/tests-test_cluster.md
- Rejected: duplicate: covered by existing auto-k-formula-is-n-div-5-capped-at-20, auto-k-floor-is-two, and auto-k-max-k-caps-cluster-count

### [ACCEPTED] estimate-tokens-uses-char-count
The tests require `estimate_tokens(text)` to equal `len(text) // 4` with a minimum return value of `1`.
- Source: summaries/2026/11/19/tests-test_compact.md

### [ACCEPTED] compact-truncates-only-when-enabled
`truncate=True` must insert `...` and suppress a 200-character node text; `truncate=False` must preserve it.
- Source: summaries/2026/11/19/tests-test_compact.md

### [REJECT] compact-budget-is-approximate
A 200-token budget test allows output below 250 estimated tokens and expects omission markers for overflowed sections.
- Source: summaries/2026/11/19/tests-test_compact.md
- Rejected: ephemeral: exact 200/250 test tolerance is a point-in-time test detail, not a durable invariant

### [REJECT] compact-orders-by-dependent-count
In the tested scenario, a node with one dependent must appear before a node with zero dependents.
- Source: summaries/2026/11/19/tests-test_compact.md
- Rejected: ephemeral: describes a single tested scenario rather than a general ordering invariant

### [REJECT] test-targets-reasons-not-reasons-lib
This test imports from `reasons`, so the parallel `reasons_lib/compact.py` implementation is not exercised here.
- Source: summaries/2026/11/19/tests-test_compact.md
- Rejected: trivial: test import/coverage detail obvious from file structure

### [ACCEPTED] detect-contradictions-only-considers-in-truth-values
Both `detect_contradictions` and its semantic variant skip any node whose `truth_value` is not `"IN"`.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] nogood-requires-at-least-two-valid-claim-ids
`parse_contradiction_response` discards any `### NOGOOD` block with fewer than 2 claims after applying the optional `valid_ids` allow-list.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] detect-contradictions-swallows-batch-errors
A `subprocess.run` exception during batching is caught and yields `[]` rather than propagating to the caller.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [REJECT] format-truncation-caps-text-at-200-chars
Truncated belief text including the trailing `...` totals exactly 200 characters.
- Source: summaries/2026/11/19/tests-test_contradictions.md
- Rejected: duplicate: same claim already covered by existing `belief-text-truncated-at-200-chars`

### [ACCEPTED] accept-flow-requires-existing-plan-file
`reasons contradictions --accept <missing>` exits with code 1; an existing but empty plan exits 0 with a `"No APPLY entries"` message.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] semantic-detection-skips-single-belief
`detect_contradictions_semantic` returns `[]` and never calls `subprocess.run` when fewer than 2 `IN` beliefs exist.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] dangling-dependents-do-not-crash-propagation
`Network._propagate()` must skip missing dependent node IDs instead of raising `KeyError`.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [ACCEPTED] dangling-dependent-produces-warning-log
Encountering a dangling dependent emits a `net.log` entry with `action == "warn"` and `target` equal to the missing ID.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [ACCEPTED] warning-value-identifies-parent-and-dangling
Warning text contains both `"dangling"` and the parent node ID whose `dependents` set held the bad reference.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [ACCEPTED] propagation-continues-past-dangling-references
Valid dependents still update truth values and are included in `changed` even when ghost dependents are present.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [ACCEPTED] dangling-ids-not-in-changed-or-visited
Dangling IDs are excluded from `changed` and must not prevent a later real node with the same ID from participating in propagation.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [ACCEPTED] mark-duplicate-requires-existing-source-and-canonical
`api.mark_duplicate` raises `KeyError("not found")` when either ID is missing.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] mark-superseded-requires-existing-old-and-new
`api.mark_superseded` raises `KeyError("not found")` when either ID is missing.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] self-deduplication-validation-prevents-mutation
Self-reference calls raise `ValueError` and leave the node’s truth value unchanged.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] deduplication-and-supersession-cascade-to-dependents
Dependents created with `sl=<source-id>` are listed in `result["changed"]` and become `OUT`.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] mark-operations-overwrite-existing-retract-reason
Marking an already-retracted node replaces its prior `retract_reason` with the duplicate/supersession reason.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] defeat-is-implemented-via-outlist
Defeating a justification works by appending the defeater node ID to that justification's `outlist`, not by deleting the justification.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] default-defeater-id-follows-pattern
When `defeater_id` is omitted, the generated ID is `"{defeater_type}-{node_id}-j{justification_index}"`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] defeater-types-are-four
The known `defeater_type` values exercised are `invalid-inference`, `over-generalizes`, `duplicate-of`, and `superseded-by`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] premises-cannot-be-defeated
Calling `defeat_justification` on a node with no justifications raises `ValueError("... no justifications ...")`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] defeat-cascades-to-dependents
Defeating a justification re-propagates and marks every transitive dependent that loses support as `OUT`, all listed in `result["changed"]`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] multiple-justifications-or-semantics
A node with N justifications stays `IN` while at least one justification remains valid, and flips `OUT` only after the last valid justification is defeated.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] defeat-with-scope-creates-scope-beliefs-as-in-premises
Each scope finding produces one `IN` premise node with no justifications and metadata linking it to the antecedent and defeater.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] retracting-scope-belief-restores-target
In the tested single-justification scenario, retracting a scope belief makes both the scope belief and defeater `OUT` and restores the target to `IN`.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] defeat-with-scope-cascades-to-dependents
Defeating a node through scope beliefs propagates `OUT` to nodes whose justifications depend on the defeated node.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] defeat-with-scope-validates-inputs-with-specific-exceptions
Missing targets raise `KeyError`, no-justification targets and empty findings raise `ValueError`, and invalid indices raise `IndexError`.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] rebuild-dependents-is-canonical-repair
`_rebuild_dependents()` clears and reconstructs all dependents from justifications and is idempotent.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md

### [ACCEPTED] mutation-methods-preserve-dependents-integrity
`add_node`, `retract`/`assert_node`, `add_justification`, `supersede`, `challenge`/`defend`, `convert_to_premise`, `add_nogood`, and `summarize` are covered and leave the index clean.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md

### [ACCEPTED] rewrite-dependents-updates-both-edges
`_rewrite_dependents()` rewrites justification antecedents/outlist references and moves reverse-index entries from the old node ID to the new one.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md

### [ACCEPTED] validate-proposals-requires-two-antecedents
`validate_proposals` moves any proposal with fewer than 2 antecedents into `skipped` with a reason containing `"at least 2 antecedents"`.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] gate-proposals-return-out-when-unless-in
Applying a `kind == "gate"` proposal whose `unless` antecedent is IN yields `truth_value == "OUT"` until the blocker is retracted, at which point it becomes `"IN"`.
- Source: summaries/2026/11/19/tests-test_derive.md

### [REJECT] any-mode-creates-one-justification-per-antecedent
Applying a proposal with `mode == "any"` produces N justifications of length 1 (one per antecedent), and retracting any single antecedent leaves the node IN.
- Source: summaries/2026/11/19/tests-test_derive.md
- Rejected: duplicate: Same claim already covered by existing belief `any-mode-creates-per-premise-justifications`.

### [ACCEPTED] find-similar-out-threshold-is-jaccard-half
`find_similar_out` matches only nodes with `truth_value == "OUT"` whose `_tokenize_id` Jaccard similarity to the proposal id is ≥ 0.5.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] dedup-rewrites-before-retracting
Both `api.deduplicate(auto=True)` and `api.apply_dedup_plan` rewrite justification antecedents and outlist references to the KEEP node before retracting the duplicates, so dependents whose justifications referenced the retracted id stay IN.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] derive-report-json-has-rounds
The `derive` CLI with `--report-dir` writes a JSON file named `derive-<ts>.json` containing keys `status`, `model`, `exhaust`, `rounds`, and `total_added`; `--exhaust` appends a saturation round whose `added == 0`.
- Source: summaries/2026/11/19/tests-test_derive.md

### [REJECT] test-derive-budget-targets-issue-23
The module docstring explicitly states these tests cover `_build_beliefs_section` budget calculation for issue #23.
- Source: summaries/2026/11/19/tests-test_derive_budget.md
- Rejected: trivial: test docstring/issue reference, not a durable code invariant.

### [REJECT] local-budget-floor-is-five
`TestBudgetFloor` asserts the local belief budget never drops below 5 when agents exceed the budget.
- Source: summaries/2026/11/19/tests-test_derive_budget.md
- Rejected: duplicate: same claim as existing budget-floor-is-five.

### [ACCEPTED] sample-mode-is-deterministic-with-same-seed
`test_sample_mode_deterministic` asserts identical output for the same input and `seed=99`.
- Source: summaries/2026/11/19/tests-test_derive_budget.md

### [ACCEPTED] output-header-format-is-contract-for-regex-helpers
`_count_local_shown` and `_count_agent_shown` parse exact header patterns such as `Local beliefs (N beliefs, showing M)` and `Agent: name (N beliefs, showing M)`.
- Source: summaries/2026/11/19/tests-test_derive_budget.md

### [ACCEPTED] challenge-returns-dict-with-challenge-id-and-changed
`Network.challenge` is contractually required to return `{"challenge_id": ..., "changed": [...]}`, and `"changed"` includes every node whose truth value was affected, including cascaded dependents.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] defense-id-is-derived-from-challenge-id
The default defense node ID is `defense-<challenge_id>`, making the defense's identity a deterministic function of the challenge it neutralizes.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [REJECT] challenge-id-collisions-get-numeric-suffix
A second call to `challenge(target)` with the same target produces `challenge-<target>-2`, not an error and not an overwrite.
- Source: summaries/2026/11/19/tests-test_dialectical.md
- Rejected: duplicate: Already captured by `challenge-id-auto-generation`, which specifies `challenge-<target>-2` behavior.

### [REJECT] retract-is-the-symmetric-undo-for-challenge-and-defense
Retracting a challenge restores the target to `IN`; retracting a defense reinstates the challenge and puts the target back to `OUT` — the same `Network.retract` primitive is used for both.
- Source: summaries/2026/11/19/tests-test_dialectical.md
- Rejected: duplicate: Covered by existing `all-defeat-mechanisms-are-reversible` and related challenge/defense reversibility beliefs.

### [ACCEPTED] dialectical-nodes-are-metadata-linked-not-edge-tabled
Challenges and defenses attach to the graph via `metadata["challenge_target"]` / `metadata["challenges"]` / `metadata["defense_target"]` / `metadata["defends"]`, not via a separate edge collection.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] missing-graph-references-raise-keyerror
`challenge` and `defend` both raise `KeyError` (not a custom exception, not a silent no-op) when the target or challenge node is absent.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] example-flag-persists-to-metadata
CLI `add --example` stores the value under `metadata["example"]` and returns exit code 0.
- Source: summaries/2026/11/19/tests-test_example.md

### [ACCEPTED] omitted-example-omits-metadata-key
Adding a node without `example` leaves the metadata without an `"example"` key.
- Source: summaries/2026/11/19/tests-test_example.md

### [ACCEPTED] show-renders-example-section
The `show` command emits an `Example:` section when an example exists and omits it otherwise.
- Source: summaries/2026/11/19/tests-test_example.md

### [ACCEPTED] multiline-example-lines-are-indented
Multiline examples are displayed with each example line prefixed by two spaces.
- Source: summaries/2026/11/19/tests-test_example.md

### [REJECT] run-cli-converts-system-exit-to-code
The `run_cli` helper catches `SystemExit` and returns its code, defaulting `None` to `0`.
- Source: summaries/2026/11/19/tests-test_example.md
- Rejected: trivial: internal test helper behavior is an implementation detail, not a durable product invariant

### [ACCEPTED] export-card-has-yaml-frontmatter
`export_card()` output is a single string with a YAML frontmatter block delimited by `---` lines.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] default-license-is-mit
Calling `export_card()` without `license=` emits `license: mit` in the frontmatter.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] card-type-is-eem
The frontmatter `type` field is always the literal string `"eem"`.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] premises-defined-by-empty-justifications
A node is counted as a premise when its `justifications` list is empty; any node with at least one justification is counted as derived.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] max-derivation-depth-counts-justification-chain-length
`Max derivation depth` equals the longest chain of antecedent edges in the network (0 for premises-only, 1 for `d ← a`, 2 for `c ← b ← a`).
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] empty-network-produces-valid-card
`export_card(Network())` succeeds and emits a card with all counts `0` and a title derived from `project_name`.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [REJECT] export-markdown-tests-target-reasons-package
The test module imports `Justification`, `Network`, and `export_markdown` from the `reasons` package, not from `reasons_lib`.
- Source: summaries/2026/11/19/tests-test_export_markdown.md
- Rejected: reason-category: trivial, test-module import detail obvious from file structure

### [ACCEPTED] export-markdown-heading-encodes-status-and-type
Claim headings are asserted as `### <id> [<STATUS>] <TYPE>` with statuses `IN`, `OUT`, or `STALE` and types such as `OBSERVATION`, `DERIVED`, or `WARNING`.
- Source: summaries/2026/11/19/tests-test_export_markdown.md

### [ACCEPTED] export-markdown-splits-outlist-defeaters-and-blockers
Outlist nodes with defeater metadata render via `- Defeated by:`, while outlist nodes without defeater metadata render via `- Unless:`.
- Source: summaries/2026/11/19/tests-test_export_markdown.md

### [ACCEPTED] export-markdown-includes-optional-node-metadata-lines
The exporter is expected to emit metadata lines such as `- Source:`, `- Source hash:`, `- Date:`, `- Stale reason:`, and `- Superseded by:` when present.
- Source: summaries/2026/11/19/tests-test_export_markdown.md

### [REJECT] defeater-export-tests-call-recompute-all
All three defeater/outlist tests call `Network.recompute_all()` before invoking `export_markdown`.
- Source: summaries/2026/11/19/tests-test_export_markdown.md
- Rejected: trivial: test implementation detail about test ordering, not a durable code invariant

### [ACCEPTED] env-org-override-does-not-apply-to-explicit-orgs
Setting `REASONS_HF_ORG` changes the default org for bare names only; explicit `owner/repo` inputs are returned unchanged.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] resolve-repo-id-rejects-non-hf-urls
`resolve_repo_id` validates that URL inputs start with `https://huggingface.co/` and raises `ValueError("Not a HuggingFace URL")` otherwise.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] create-repo-is-idempotent-on-409
A `HTTPError(..., 409, ...)` from the create endpoint is swallowed and the repo id is returned as if creation succeeded.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] publish-hf-produces-three-files
`publish_hf` uploads exactly `{"network.json", "beliefs.md", "README.md"}` and issues 4 `urlopen` calls (1 create + 3 uploads).
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] upload-file-uses-put-and-bearer-auth
`upload_file` sends a PUT to `/api/models/{repo_id}/upload/main/{filename}` with header `Authorization: Bearer <token>`.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] publish-hf-requires-token
`publish_hf` raises `RuntimeError("token required")` when neither `HF_TOKEN` nor `~/.huggingface/token` supplies a token.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] search-default-excludes-out
`api.search` omits retracted nodes from its results when `include_out` is not specified.
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [ACCEPTED] include-out-enables-out-visibility
Passing `include_out=True` causes all three retrieval APIs (`search`, `lookup`, `compact`) to include retracted nodes.
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [ACCEPTED] empty-result-message-is-api-specific
`api.search` returns text containing `"No results found"` while `api.lookup` returns `"No beliefs found"` when no IN matches exist.
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [ACCEPTED] compact-renders-out-section-header-only-when-requested
`api.compact` emits the literal header `"OUT (retracted)"` if and only if `include_out=True`.
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [ACCEPTED] out-filter-precedes-neighbor-expansion
Retracted beliefs are excluded even when they are dependents of a live node returned by `api.search` (`### derived-out` is absent from output).
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [REJECT] tests-bind-to-reasons-facade
The suite imports `api` from `reasons`, not `reasons_lib`, so the `reasons` package must re-export all six used symbols.
- Source: summaries/2026/11/19/tests-test_hide_out.md
- Rejected: trivial: describes import statements/test structure visible directly from module imports

### [REJECT] active-precondition-is-not-an-antecedent
For every imported belief, `{agent}:active` is excluded from `justifications[0].antecedents`; this is the explicit fix for the issue-#16 resurrection bug.
- Source: summaries/2026/11/19/tests-test_import_agent.md
- Rejected: duplicate: already captured by existing `active-not-in-antecedents`

### [REJECT] every-imported-belief-has-inactive-in-outlist
Namespaced beliefs all carry `{agent}:inactive` in their outlist, which is what makes retracting `{agent}:active` cascade through every belief.
- Source: summaries/2026/11/19/tests-test_import_agent.md
- Rejected: duplicate: already captured by `agent-beliefs-are-permanently-defeasible-and-updatable` and `active-inactive-relay-pair`

### [ACCEPTED] import-agent-registers-repo
`import_agent` records the source file's resolved parent directory under `repos[agent]`, visible via `api.list_repos(db_path=...)["repos"]`.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] out-source-status-survives-recompute
A belief imported with source status `[OUT]` remains `OUT` even after `Storage(db).load().recompute_all()`, while `[STALE]` beliefs can be propagated back to `IN` on recompute.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] duplicate-import-is-a-noop
Re-importing an identical file returns `claims_imported == 0`, `claims_skipped == N`, and `created_premise == False`; no exceptions are raised for the collision.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] import-agent-supports-markdown-and-json
The public API `reasons.api.import_agent` accepts both the Markdown "Beliefs/Nogoods" export and the JSON network-export format, and both preserve the same namespacing/outlist invariants.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] propose-updates-is-read-only
`propose_updates()` returns proposal dicts and never mutates the belief graph.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

### [ACCEPTED] propose-stage-requires-external-accept
`format_proposal_markdown()` emits suggested `reasons update`/`reasons retract` commands but does not execute them.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

### [ACCEPTED] proposal-parse-failures-return-empty-list
`parse_update_proposals()` returns `[]` for malformed LLM output, indistinguishable from an empty proposal set.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

### [ACCEPTED] proposal-parser-does-not-validate-node-ids
`parse_update_proposals()` accepts any item with an `id` key and does not check membership in the graph.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

### [ACCEPTED] propose-batch-errors-are-swallowed
`propose_updates()` catches per-batch exceptions, logs a warning, and returns partial results rather than raising.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md


---

**Generated:** 2026-11-19 (batch 1/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] node-supporting-justification-is-list-index
`Node.supporting_justification` is an index into the node's `justifications` list, not a justification or node ID, so any reordering/insertion/removal must recompute or invalidate it.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] node-truth-default-valid-only-for-premises
`Node.truth_value` defaults to `"IN"`, which is only correct for premises with no justifications; derived nodes must have their truth value recomputed by propagation.
- Source: summaries/2026/11/19/reasons-__init__.md

### [ACCEPTED] node-timestamps-are-unvalidated-strings
`Node` lifecycle timestamps (`created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at`) are plain strings with no format enforcement, and a non-empty `retracted_at` is the retraction signal.
- Source: summaries/2026/11/19/reasons-__init__.md

### [REJECT] namespace-active-premise-gates-beliefs-via-antecedent
`ensure_namespace` creates a normal `{ns}:active` premise and `add_node` inserts that premise as the first antecedent, so retracting it cascades namespace beliefs OUT through ordinary SL propagation.
- Source: summaries/2026/11/19/reasons-api.md
- Rejected: duplicate: Core claim that add_node creates ns:active and prepends it as an antecedent is already captured by add-node-namespace-auto-wiring-creates-active-premise / api-add-node-auto-wires-namespace-active-premise.

### [ACCEPTED] pg-unsupported-features-raise-before-dispatch
API features unsupported by the PostgreSQL backend (e.g. `any_mode`, tag-filtered `search`, `min_depth`) raise `NotImplementedError` before `_pg_dispatch`, rather than silently using SQLite behavior.
- Source: summaries/2026/11/19/reasons-api.md

### [ACCEPTED] api-proposals-stage-in-separate-table-without-truth-mutation
Hive proposals are stored in a separate `proposals` table and `propose_*` functions do not mutate node truth values; truth changes occur only when an accepted proposal is applied.
- Source: summaries/2026/11/19/reasons-api.md

### [ACCEPTED] api-resolve-namespace-prefixes-only-unqualified-ids
`_resolve_namespace` only prefixes an ID when the namespace is truthy and the ID does not already contain `":"`, treating pre-namespaced IDs as cross-namespace references.
- Source: summaries/2026/11/19/reasons-api.md

### [ACCEPTED] ask-dual-empty-detection-coupled-to-sentinels
Dual mode detects TMS emptiness via `NO_BELIEFS_MSG` or the substring `"don't have enough beliefs"` and FTS emptiness via `answer_fts.startswith("No relevant documents")`, so changing those prompt/sentinel strings silently breaks dual short-circuiting.
- Source: summaries/2026/11/19/reasons-ask.md

### [REJECT] ask-final-round-omits-tool-schema
The final ask round uses `FINAL_ASK_PROMPT` with no tool definitions; if the model still emits a tool call, `ask` returns the fallback context instead of looping again.
- Source: summaries/2026/11/19/reasons-ask.md
- Rejected: duplicate: Covered by existing `ask-last-iteration-structurally-forces-answer`, which already states the final iteration switches to FINAL_ASK_PROMPT and omits tools.

### [ACCEPTED] ask-tool-results-replace-context-only-if-meaningful
Tool-loop search results replace `beliefs_context` only when non-empty and not `"No results found."`; empty tool results are recorded in `tool_history` but do not overwrite the current context.
- Source: summaries/2026/11/19/reasons-ask.md

### [REJECT] ask-natural-mode-strips-belief-citations
With `natural=True`, both `_CITE_RULE_NATURAL` and `_strip_belief_metadata` remove belief IDs and structured metadata from the final answer, producing prose-only output.
- Source: summaries/2026/11/19/reasons-ask.md
- Rejected: duplicate: Same claim as existing `ask-natural-mode-strips-metadata-and-cite` (and its variants).

### [ACCEPTED] build-wiki-reserved-index-slug-rewritten
A topic label that slugifies to `index` is rewritten to `index-topic` so it cannot overwrite the generated `index.md`.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] build-wiki-duplicate-node-membership-last-group-wins
If the same node ID appears in multiple groups, `node_to_page` is built by walking groups in order, so the node is linked to the last group's page without error.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] build-wiki-defeater-rendering-reverse-outlist-scan
`_format_node` renders “Defeated by” by scanning each justification's `outlist`, looking up those nodes, and keeping only nodes whose `metadata["defeats_node"]` equals the rendered node ID.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] build-wiki-linkify-skips-current-page
`_linkify` skips IDs whose target page equals the current page, leaving intra-page references as literal text while still linking cross-page IDs.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] build-wiki-prompt-omits-outlist-and-metadata
`_format_beliefs_for_prompt` emits only status, text, dependency, and support summary for the LLM; unlike `_format_node`, it does not walk outlists or metadata.
- Source: summaries/2026/11/19/reasons-build_wiki.md

### [ACCEPTED] check-stale-mutates-in-place-caller-persists
`check_stale`, `hash_sources`, and all `pin_*` functions mutate `Node.source_hash` and/or `Node.metadata` in place; callers are responsible for saving the network.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] pinned-lines-malformed-falls-back-to-hashing
During `check_stale`, malformed `metadata["pinned_lines"]` silently falls through to whole-file hash checking, while `pin_lines` itself validates line ranges on write.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] diff-hunks-use-old-side-and-drop-pure-insertions
`parse_diff_hunks` returns OLD-side line ranges from `git diff` hunks and drops pure-insertion hunks with old count zero, so insertions above a pinned range do not flag it.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] pin-update-reports-per-node-errors
`pin_update` returns per-node error dicts such as `{"error": "not found" | "no source" | "source not found" | "not in git repo"}` instead of silently skipping failures as `pin_sources` does.
- Source: summaries/2026/11/19/reasons-check_stale.md

### [ACCEPTED] hash-sources-backfills-from-source-without-requiring-hash
`hash_sources` sets `source_hash` for any node with a resolvable source and no hash (unless `force`), whereas `check_stale` inspects only nodes that already have both `source` and `source_hash`.
- Source: summaries/2026/11/19/reasons-check_stale.md


---

**Generated:** 2026-11-19 (batch 2/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] cli-handlers-are-api-facade-except-derive
CLI subcommand handlers act as thin facades over `reasons.api`, with `cmd_derive` as the only handler containing real orchestration.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] cli-stream-discipline-separates-payload-and-diagnostics
CLI user payload is written to `stdout`, while diagnostics, warnings, progress, and cost summaries go to `stderr` so commands remain pipeable.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] cli-sqlite-only-commands-guard-before-api
CLI commands marked SQLite-only call `_require_sqlite` and exit with status 1 before any API call when PostgreSQL is configured.
- Source: summaries/2026/11/19/reasons-cli.md

### [ACCEPTED] cluster-budget-cap-can-underselect
`cluster_beliefs` caps each cluster’s allocation at cluster size and does not redistribute unused slots, so it can return fewer than `budget` IDs.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] cluster-intra-focus-is-size-sorted-index
`cluster_beliefs_intra` reports `focus_cluster` as an index into the descending-size-sorted cluster list, not as the original KMeans label.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] cluster-public-functions-require-deps-before-fast-path
All public clustering functions call `_require_cluster_deps()` before their small-input fast paths, so missing optional dependencies raise `ImportError` even when clustering would be skipped.
- Source: summaries/2026/11/19/reasons-cluster.md

### [ACCEPTED] compact-header-and-footer-are-unconditional
`compact` always emits the header and token-count footer regardless of budget; only the nogoods, OUT, and IN content sections are budget-gated.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] compact-summaries-hide-covered-in-nodes
`compact` hides an IN node when its ID appears in another IN node’s `metadata["summarizes"]` list and reports the hidden count.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] compact-out-reason-priority
For OUT nodes, `compact` surfaces the reason using priority order `retract_reason`, then `stale_reason`, then `superseded_by`.
- Source: summaries/2026/11/19/reasons-compact.md

### [ACCEPTED] contradictions-semantic-has-fixed-batch-size
`detect_contradictions_semantic` does not accept a `batch_size` parameter and always chunks clusters at `CONTRADICTION_BATCH_SIZE` (50).
- Source: summaries/2026/11/19/reasons-contradictions.md

### [ACCEPTED] contradiction-valid-id-filter-is-global
`detect_contradictions` builds `valid_ids` from the full candidate set, not from the current batch, so hallucinated-ID filtering is global across all batches.
- Source: summaries/2026/11/19/reasons-contradictions.md

### [ACCEPTED] contradiction-parser-does-not-raise
`parse_contradiction_response` never raises for malformed or non-matching LLM output; it simply yields no proposal entries unless well-formed `### NOGOOD` blocks survive parsing.
- Source: summaries/2026/11/19/reasons-contradictions.md


---

**Generated:** 2026-11-19 (batch 3/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] export-card-out-count-is-exact-in-only
A node is counted as IN only when `truth_value == "IN"` exactly; every other truth value contributes to OUT.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-card-premise-depth-is-zero
`_node_depth` returns `0` for missing nodes and for nodes without justifications.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-card-invalid-supporting-justification-falls-back
When `supporting_justification` is not a valid index into `node.justifications`, depth traversal falls back to considering all justifications.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-card-cycle-sentinel-prevents-infinite-recursion
`_node_depth` assigns `memo[nid] = 0` before recursing, preventing infinite recursion on cyclic networks without validating acyclicity.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-card-empty-network-safe
`export_card` handles empty networks safely by guarding percentage division with `total == 0` and using `default=0` for `max_depth`.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-card-yaml-fields-mostly-unescaped
Only `project_name`, `schema_version`, and `base_network` are quoted in frontmatter; `license`, `domain`, `source_repos`, and `generator` values are inserted unescaped.
- Source: summaries/2026/11/19/reasons-export_card.md

### [ACCEPTED] export-markdown-returns-string-with-no-io-mutation
`export_markdown` returns a Markdown string, performs no file I/O, and does not mutate the passed `Network`.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-flattens-justifications-to-node-level-bullets
Multiple justifications are flattened into node-level `Depends on`, `Unless`, and `Defeated by` bullets, losing per-justification grouping in the Markdown output.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-metadata-serialization-is-whitelisted
Only curated metadata fields are serialized, keeping the Markdown schema stable rather than dumping arbitrary node metadata.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-nogoods-preserve-network-order
Nogoods are emitted in `network.nogoods` order and are not sorted by ID.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] export-markdown-unquoted-values-are-not-escaped
Node text, IDs, reason strings, repo paths, and many metadata strings are not escaped before insertion into Markdown.
- Source: summaries/2026/11/19/reasons-export_markdown.md

### [ACCEPTED] hf-uses-stdlib-urllib-only
`reasons/hf.py` talks to HuggingFace via stdlib `urllib.request` and does not import `huggingface_hub` or `requests`.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] hf-create-repo-swallows-409
`create_repo` treats HTTP `409 Conflict` as success so it can be called unconditionally before upload.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] hf-download-network-returns-unparsed-json-text
`download_network` returns the response body as a decoded JSON string; parsing and schema validation are the caller’s responsibility.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] hf-endpoints-hardwired-to-main-revision
HuggingFace download and upload endpoints are hard-wired to the `main` revision; there is no branch/revision parameter.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] hf-resolve-repo-id-rejects-non-hf-urls
`resolve_repo_id` raises `ValueError` for URLs that do not contain `huggingface.co/` rather than silently mis-parsing them.
- Source: summaries/2026/11/19/reasons-hf.md

### [ACCEPTED] import-agent-sync-updates-existing-nodes-via-field-diff
`_sync_claims` updates existing agent nodes by diffing `text`, source fields, date, metadata, and justifications rather than blindly replacing them.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] import-agent-records-agent-source-path-in-network-repos
`_import_claims` records the importing agent’s resolved parent path into `network.repos[agent_name]`.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] import-agent-removals-iterate-sorted-node-ids
Sync removal processing iterates `sorted(removed_ids)`, making removal order deterministic.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] import-agent-sync-asserts-resurrections-before-recompute
Sync queues `assert_after` resurrections and processes them before `recompute_all`, while retractions are deferred until after recomputation.
- Source: summaries/2026/11/19/reasons-import_agent.md

### [ACCEPTED] import-beliefs-frontmatter-parser-is-minimal-flat
`strip_frontmatter` is a minimal flat `key: value` parser that strips surrounding quotes and does not use a YAML library or nested structures.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-frontmatter-lifts-only-three-keys
`import_into_network` copies only `schema_version`, `project_name`, and `created_at` from frontmatter into `network.meta`; other frontmatter keys are dropped.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-claim-text-captures-first-nonempty-line-only
`parse_beliefs` captures only the first non-empty, non-bullet line after a claim header, silently truncating multi-line claim text.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-metadata-matching-case-inconsistent
Claim metadata parsing is case-inconsistent: `Source URL`, `Source hash`, and `Accepted PR` are case-sensitive, while `Stale reason` and `Superseded by` are matched case-insensitively.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-justifications-filter-to-in-file-claims-only
`import_into_network` filters `depends_on` and `unless` entries against IDs parsed from the same file, not against all existing `network.nodes`, so pre-existing external dependencies are not aggregated.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-duplicate-claim-ids-last-wins
`import_into_network` builds `claim_by_id` as a last-wins dict, so duplicate claim IDs in one input collapse to the final occurrence.
- Source: summaries/2026/11/19/reasons-import_beliefs.md

### [ACCEPTED] import-beliefs-metadata-whitelist-drops-empty-values
Optional import metadata is limited to a whitelist (`beliefs_type`, `source_type`, `stale_reason`, `superseded_by`, `accepted_pr`) and empty values are dropped before insertion.
- Source: summaries/2026/11/19/reasons-import_beliefs.md


---

**Generated:** 2026-11-19 (batch 4/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] llm-cost-tracker-mutations-are-lock-protected
`_cost_tracker` mutations happen under `_cost_lock`; the LLM call itself is not held under the lock.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] llm-cost-summary-returns-deepcopy
`get_cost_summary()` returns a deepcopy of the tracker so callers cannot mutate internal cost state.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] llm-api-path-skips-cost-recording
The `api:` and `vertex:` invocation path does not call `_record_cost`, so those calls do not contribute to cost/token totals.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] llm-cli-json-parser-fails-open
`_parse_cli_json` returns raw CLI output on JSON parse failure or non-dict data instead of raising.
- Source: summaries/2026/11/19/reasons-llm.md

### [ACCEPTED] llm-api-path-normalizes-exceptions
The API invocation path maps timeout-like exceptions to `subprocess.TimeoutExpired` and other exceptions to `RuntimeError`.
- Source: summaries/2026/11/19/reasons-llm.md

### [REJECT] llm-cli-strips-claudecode-env
The CLI subprocess environment unconditionally removes `CLAUDECODE` to avoid the Claude CLI recursive-session guard.
- Source: summaries/2026/11/19/reasons-llm.md
- Rejected: duplicate: same claim already captured by existing belief `ask-strips-claudecode-env` (`_invoke_claude` removes CLAUDECODE to prevent recursive invocation).

### [ACCEPTED] mcp-bridge-requires-mcp-package
Constructing `McpBridge` raises `ImportError` when the optional `mcp` package is unavailable.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-bridge-caches-tools-at-connect
`list_tools()` returns tool metadata cached during `connect()` and does not re-query the server afterward.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-bridge-connect-startup-timeout
`connect()` waits up to 30 seconds for server readiness and raises `TimeoutError` if `_ready` is not set.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-bridge-call-tool-timeout
`call_tool()` blocks on the background future with a 60-second timeout and does not explicitly cancel the coroutine on timeout.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-bridge-command-uses-shlex-no-shell
`McpBridge` parses its command with `shlex.split`; the first token is the executable and no shell is used.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-bridge-pre-try-startup-errors-masked-as-timeout
Startup errors before the lifecycle `try` block are not stored in `_error`, so `connect()` may surface a 30-second timeout instead of the original exception.
- Source: summaries/2026/11/19/reasons-mcp_client.md

### [ACCEPTED] mcp-server-tools-are-tiered-facade
`reasons/mcp_server.py` organizes tools into four tiers: core, reasoning, proposals, and data management.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-server-names-only-formatting-is-local
For `list_beliefs`, `status`, and `list_gated`, the MCP server handles `"names-only"` formatting locally; `search` and `compact` forward `output_format` into the API.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-server-what-if-supersede-requires-text
The `what_if` MCP tool requires non-empty `text` when `action=="supersede"` and otherwise returns a structured error.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-server-propose-update-stores-conditionally
The `propose_update` tool calls `api.propose_update`, then optionally chains `api.store_llm_proposals` when `store` is true and proposals exist.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] mcp-server-docstrings-are-tool-schemas
FastMCP derives MCP tool names from function names and tool schemas/descriptions from signatures and docstring `Args:` blocks, making docstrings part of the public MCP contract.
- Source: summaries/2026/11/19/reasons-mcp_server.md

### [ACCEPTED] merkle-content-hash-is-order-independent
Justification content hashes sort antecedent Merkle hashes before joining them, so antecedent order does not affect the hash.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] merkle-missing-antecedents-are-omitted
`_fresh_content_hash` filters antecedents to those present in `net.nodes`, silently omitting missing antecedent IDs.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] merkle-duplicate-antecedents-not-deduplicated
Duplicate antecedent IDs are not deduplicated when computing a justification content hash; each occurrence contributes its hash.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] merkle-backfill-only-fills-missing-hashes
`backfill_hashes` fills only missing `text_hash` and `content_hash` values; it never overwrites existing or stale hashes.
- Source: summaries/2026/11/19/reasons-merkle.md

### [ACCEPTED] merkle-missing-hashes-are-not-findings
`verify_all` counts missing hashes in `missing_hashes` and does not emit findings for them; missing hashes are treated as unverified, not mismatched.
- Source: summaries/2026/11/19/reasons-merkle.md

### [REJECT] merkle-supporting-justification-assumed-valid-index
`_fresh_merkle_hash` assumes `supporting_justification` is either `None` or a valid index; invalid values can raise `IndexError` or use Python negative indexing unexpectedly.
- Source: summaries/2026/11/19/reasons-merkle.md
- Rejected: speculative: Describes hypothetical invalid-input behavior and risks ("can raise IndexError or use negative indexing unexpectedly") rather than an observable invariant.

### [ACCEPTED] metadata-schema-version-is-hardcoded
`SCHEMA_VERSION` is a hardcoded `"1.0"` module constant and is independent of the installed package version.
- Source: summaries/2026/11/19/reasons-metadata.md

### [ACCEPTED] metadata-new-record-timestamps-are-identical
`build_meta` computes `now` once; for falsey `created_at`, both `created_at` and `updated_at` receive the same value.
- Source: summaries/2026/11/19/reasons-metadata.md

### [ACCEPTED] metadata-updated-at-is-always-regenerated
`build_meta` always regenerates `updated_at` from the current UTC time; callers cannot supply or preserve it.
- Source: summaries/2026/11/19/reasons-metadata.md

### [ACCEPTED] metadata-generator-version-lookup-is-lazy
`_get_generator()` imports `importlib.metadata.version` inside the function, so importing `metadata` does not require installed distribution metadata.
- Source: summaries/2026/11/19/reasons-metadata.md

### [ACCEPTED] metadata-timestamps-are-utc-second-precision
Generated metadata timestamps use `datetime.now(timezone.utc).isoformat(timespec="seconds")`, producing UTC ISO-8601 strings with seconds precision.
- Source: summaries/2026/11/19/reasons-metadata.md


---

**Generated:** 2026-11-19 (batch 5/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] mind-service-client-is-stdlib-only
`reasons/mind_service.py` imports only stdlib modules (`json`, `os`, `urllib.*`) and requires no third-party HTTP library.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] mind-service-config-resolution-prefers-explicit-params
`_resolve_config` gives explicit function arguments precedence over the corresponding `MIND_*` environment variables.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] mind-service-resolved-url-strips-trailing-slash
`_resolve_config` always strips trailing `/` from the resolved URL before returning it.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] fetch-export-returns-raw-utf8-string
`fetch_export` returns the response body as a raw UTF-8 string and leaves JSON parsing to the caller.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] push-belief-404-propagates-as-httperror
`push_belief` special-cases only HTTP 401; HTTP 404 from the push endpoint propagates as an `HTTPError` rather than becoming a `RuntimeError`.
- Source: summaries/2026/11/19/reasons-mind_service.md

### [ACCEPTED] network-is-pure-in-memory-semantic-engine
`Network` owns only the in-memory belief graph and truth maintenance; persistence and API concerns live in `storage.py` and `api.py`.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] network-add-node-rejects-duplicate-ids
`Network.add_node` raises `ValueError` if the supplied node ID already exists.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] network-add-node-no-justification-is-premise-in
`Network.add_node` sets `truth_value` to `"IN"` when no justifications are supplied; otherwise it computes truth from the justifications.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] network-remove-justification-refuses-empty-derived-node
`Network.remove_justification` raises `ValueError` when removing the last justification from a derived node and directs callers to `convert_to_premise` or `retract`.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] access-tag-inheritance-uses-antecedents-only
`Network._inherit_access_tags` unions `access_tags` only from antecedent nodes, not from outlist nodes.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] trace-assumptions-traverses-antecedents-only
`Network.trace_assumptions` collects premises by DFS over antecedents and does not traverse outlist edges.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] defend-delegates-to-challenge
`Network.defend(target_id, challenge_id, ...)` is implemented by calling `challenge(challenge_id, ...)`, treating a defense as a challenge against the challenge node.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] summarize-creates-sl-node-with-antecedents
`Network.summarize` creates an SL node whose antecedents are the summarized nodes and whose metadata records the `summarizes` back-reference.
- Source: summaries/2026/11/19/reasons-network.md

### [ACCEPTED] pgapi-connection-ownership-is-constructor-dependent
`PgApi` closes its connection on `close()` only when constructed from a conninfo string; an injected `psycopg.Connection` is not closed by `PgApi`.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] pgapi-public-mutations-commit-per-operation
`PgApi` public mutation methods commit at the end of each operation; there are no long-lived transactions spanning multiple public calls.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] pgapi-add-node-returns-contract-and-rejects-duplicates
`PgApi.add_node` raises `ValueError` if the node ID already exists and returns a dict containing `node_id`, `truth_value`, `type`, and `premise_count`.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] pgapi-raw-add-node-bypasses-validation-and-tag-inheritance
`PgApi._add_node_raw` is an unsafe insert used by import paths: it does not validate references, inherit access tags, or commit.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] pgapi-defers-api-imports-to-avoid-circular-dependency
`PgApi` defers imports from `reasons.api`, `llm`, `check_stale`, `import_beliefs`, `metadata`, and `import_agent` into function bodies to avoid a circular import with `reasons.api`.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] pgapi-init-db-uses-idempotent-column-probing
`PgApi.init_db` checks `information_schema.columns` and applies `ALTER TABLE ... ADD COLUMN` for missing columns, making migrations idempotent.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] pgapi-justifications-cascade-delete-with-nodes
The `rms_justifications` table has `ON DELETE CASCADE` from `rms_nodes`, so deleting a node deletes its justifications.
- Source: summaries/2026/11/19/reasons-pg.md

### [ACCEPTED] propose-update-module-is-read-only-proposal-generator
`reasons/propose_update.py` only produces proposal dicts and markdown; it does not apply accepted changes to storage.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] propose-updates-default-candidacy-is-in-set
`propose_updates` selects only nodes with `truth_value == "IN"` when `belief_ids` is `None`, and filters explicit `belief_ids` to IDs present in `nodes`.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] propose-update-failure-modes-and-bases-are-closed-sets
`parse_update_proposals` coerces `failure_mode` to one of six `FAILURE_MODES` values or `""`, and `basis` to one of three `BASES` values or `"prior-knowledge"`.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] parse-update-proposals-takes-first-valid-json-array
`parse_update_proposals` returns the first parsed JSON array that yields at least one valid dict item and ignores later candidate arrays in the same response.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] propose-updates-on-batch-receives-cumulative-list
The `propose_updates` `on_batch` callback receives the cumulative results list, not the current batch slice.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] proposal-markdown-escapes-only-double-quotes
`format_proposal_markdown` escapes only double quotes in `proposed_text` when building copy-paste commands; backticks, dollar signs, backslashes, and single quotes pass through unchanged.
- Source: summaries/2026/11/19/reasons-propose_update.md

### [ACCEPTED] propose-updates-empty-input-skips-llm
`propose_updates` returns `[]` without calling the LLM or printing progress when the selected belief list is empty.
- Source: summaries/2026/11/19/reasons-propose_update.md


---

**Generated:** 2026-11-19 (batch 6/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] research-beliefs-is-alias-for-repair-beliefs
`research_beliefs` is a backward-compatible alias equal to `repair_beliefs`.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-dry-run-still-calls-llm-and-search
`dry_run=True` suppresses database mutations in the repair orchestrators but does not suppress LLM calls or search calls.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-lazy-imports-api-to-break-circular-import
`reasons/repair.py` imports `reasons.api` lazily inside functions rather than at module import time to avoid circular-import problems.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-search-is-dependency-injected
Both repair orchestrators accept an injectable `search_fn`, defaulting to `lambda query, **kw: api.search(query, include_out=True, **kw)`, so tests can avoid touching the real database.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-depth-is-max-antecedent-depth-plus-one
`_compute_depth` returns `0` for missing nodes or nodes without justifications, otherwise `max(antecedent_depth) + 1`, using a memo table and cycle guard.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-metadata-audit-trail-is-pattern-specific
Successful non-dry-run repairs write pattern-specific `repair_action` metadata: `"search_and_link"`, `"softened"`, `"abandoned"`, or `"research"`, with research additionally recording `repair_research`.
- Source: summaries/2026/11/19/reasons-repair.md

### [ACCEPTED] repair-premises-produces-verdicts-not-mutations
`reasons/repair_premises.py` produces repair verdicts for higher-level code to execute but does not apply changes to the store itself.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] repair-premises-prompt-domain-is-rewrite-retract
`REPAIR_PREMISES_PROMPT` allows only `action` values `"rewrite"` and `"retract"`, with `"error"` injected only by the worker.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] repair-premises-empty-input-short-circuits
`repair_premises` returns `[]` immediately when `premise_ids` is empty, making no LLM calls and causing no side effects.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] repair-premises-keyerror-aborts-batch
In `_repair_one`, `nodes[node_id].get("source", "")` is outside the `try/except`, so a missing premise ID raises `KeyError` and in parallel mode aborts the entire batch.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] repair-premises-on-result-receives-cumulative-list
The `on_result` callback in `repair_premises` is invoked after every result in both sequential and parallel modes and receives the full growing `all_results` list, not just the newest result.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] parse-repair-response-drops-malformed-items
`parse_repair_response` scans for the first valid JSON array, keeps only dict items containing `id`, normalizes surviving items with defaults, and returns `[]` if none parse.
- Source: summaries/2026/11/19/reasons-repair_premises.md

### [ACCEPTED] report-belief-only-in-gets-plus-marker
In `_build_structured_report`, only `truth_value == "IN"` renders as `[+]`; every other value, including `"OUT"`, `"UNKNOWN"`, or missing, renders as `[-]`.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-belief-requires-source-chunks-mapping
Both `report_belief` formatting helpers call `source_chunks.get(...)`, so callers must pass a mapping keyed by premise id; passing `None` raises `AttributeError`.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-belief-llm-failure-propagates-no-fallback
If `model` is set and `invoke_model` raises, `report_belief` propagates the exception instead of falling back to the deterministic structured report.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-belief-lazy-imports-llm
`generate_report` lazily imports `invoke_model` inside the function, so the structured rendering path in `report_belief` does not import the LLM layer.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-belief-distinguishes-empty-vs-absent-source-chunks
If a premise id exists in `source_chunks` with an empty list, the structured report emits `*No matching source chunks found.*`; if the id is absent from the mapping, no evidence section is emitted for that premise.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-belief-prompt-omits-outlist
`_format_data_for_prompt` does not include each justification step's `outlist`, even though `_build_structured_report` renders the `outlist`.
- Source: summaries/2026/11/19/reasons-report_belief.md

### [ACCEPTED] report-gated-structured-truncates-but-prompt-uses-full-text
The deterministic gated report truncates retracted premise text to 80 characters and gated belief text to 100 characters, while `_format_data_for_prompt` includes full text without truncation.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-escapes-pipes-only-in-retracted-table
Structured gated reporting escapes `|` as `\|` only in the retracted-premise table's `text` and `retract_reason` fields; blocker and gated-belief text are inserted without Markdown escaping.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-trusts-summary-counts
`report_gated` uses `gated_data["blocker_count"]` and `gated_data["gated_count"]` directly in the summary and does not recompute or cross-validate them against the `blockers` mapping.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-order-is-deterministic
Both gated report paths sort retracted premises by `dependent_count` descending, blockers by number of gated beliefs descending, and gated beliefs by `id` ascending.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-missing-blocker-detail-defaults-zero
Missing `blocker_details[bid]` is treated as `dependent_count = 0`.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-structured-always-includes-methodology
The deterministic structured gated report always includes a `## Methodology` section, while the LLM-generated report does not automatically include that section.
- Source: summaries/2026/11/19/reasons-report_gated.md

### [ACCEPTED] report-gated-dual-mode-selects-on-model
`report_gated` uses a truthy `model` to select LLM narrative rendering and a falsy `model` to select deterministic structured rendering, with both paths returning Markdown beginning with `# Gated Beliefs Report`.
- Source: summaries/2026/11/19/reasons-report_gated.md


---

**Generated:** 2026-11-19 (batch 7/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] review-justifications-selection-requires-in-truth-value
Both the default and explicit `belief_ids` selection branches include only nodes with `truth_value == "IN"`; explicit IDs failing this filter are silently dropped.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] review-justifications-on-batch-receives-cumulative-results
`on_batch` is invoked after each batch with the full accumulated `all_results` list, not just the newest batch's results.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] review-justifications-parallel-mode-loses-input-order
When `parallel > 0`, results are appended in `as_completed` order, so the returned list is not guaranteed to follow the original `belief_ids` order.
- Source: summaries/2026/11/19/reasons-review_justifications.md

### [ACCEPTED] review-premises-unknown-ids-silently-dropped
Unknown premise IDs produce empty formatted chunks and no matching LLM result, with no warning; they vanish from the returned list.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] review-premises-on-batch-receives-shared-mutable-cumulative-list
`on_batch` is called with the same mutable `all_results` list after each batch; retaining it without copying will observe later mutations.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] review-premises-callback-exceptions-abort-run
Exceptions raised by `on_batch` are not caught inside `review_premises`; they abort the function and, in parallel mode, can cancel pending futures.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] review-premises-missing-source-uses-placeholder
When a premise's source is absent from `source_contents`, `_process_batch` renders the literal `"(source not available)"` placeholder and still sends the premise to the LLM.
- Source: summaries/2026/11/19/reasons-review_premises.md

### [ACCEPTED] storage-newness-is-file-existence
`Storage` computes `_is_new` from whether the database path exists, not from whether tables are empty; `network_meta` seeding only occurs on a newly created file.
- Source: summaries/2026/11/19/reasons-storage.md

### [ACCEPTED] storage-save-dedups-sources-by-node-and-source
On save, external `node_sources` rows are restored only when no in-memory node already produced a row with the same `(node_id, source)` pair, preventing duplicate source references.
- Source: summaries/2026/11/19/reasons-storage.md

### [ACCEPTED] storage-connection-enforces-foreign-keys
`Storage` opens SQLite with `PRAGMA foreign_keys=ON`, so insertion order must put nodes before their tags/sources/justifications and deletion must clear children first.
- Source: summaries/2026/11/19/reasons-storage.md

### [ACCEPTED] storage-init-schema-backfills-legacy-tags-and-sources
When `node_tags`/`node_sources` are first introduced, `_init_schema` copies `metadata.access_tags` into `node_tags` as `access:<tag>` and copies each node's `source` into `node_sources`.
- Source: summaries/2026/11/19/reasons-storage.md

### [ACCEPTED] read-source-never-raises-returns-none
`read_source` catches `OSError` and `UnicodeDecodeError`, returning `None` instead of raising when the source cannot be resolved or read.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] verify-parser-greedy-brace-regex
`parse_verify_response` extracts JSON using a greedy `r"\{.*\}"` `DOTALL` regex, so extra braces after the JSON object can make parsing fail and return `{}`.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] verify-verdicts-uppercased-not-validated
`parse_verify_response` uppercases verdict strings but does not validate them against `CONFIRMED`/`STALE`/`PARTIAL`/`INCONCLUSIVE`; an unknown verdict like `"MAYBE"` is passed through.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] verify-quote-not-validated-against-source
The verification prompt requests a verbatim quote, but no post-hoc substring check validates the returned quote against the source text.
- Source: summaries/2026/11/19/reasons-verify.md

### [ACCEPTED] verify-read-source-resolves-relative-to-db-directory
`read_source` delegates to `check_stale.resolve_source_path` with the directory containing the database as the base for relative source paths.
- Source: summaries/2026/11/19/reasons-verify.md


---

**Generated:** 2026-11-19 (batch 8/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] reasons-lib-api-circular-import-risk-with-canonical-importers
`reasons.api` must not import modules such as `reasons.ask` or `reasons.cli` at module load, because those modules import `reasons_lib.api` and would complete a cycle with `reasons_lib.api -> reasons.api`.
- Source: summaries/2026/11/19/reasons_lib-api.md

### [ACCEPTED] reasons-lib-build-wiki-is-distinct-module-object
`reasons_lib.build_wiki` is a distinct module object from `reasons.build_wiki`; the shim copies names into its own namespace rather than aliasing the canonical module in `sys.modules`.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md

### [REJECT] reasons-lib-build-wiki-downstream-star-import-differs-from-canonical-all
Because the shim defines no local `__all__`, `from reasons_lib.build_wiki import *` is governed by the shim's namespace and can differ from `reasons.build_wiki.__all__`, such as omitting underscore-prefixed names even if the canonical module lists them in `__all__`.
- Source: summaries/2026/11/19/reasons_lib-build_wiki.md
- Rejected: trivial: standard star-import behavior from the shim namespace; derived implementation detail rather than a durable architecture/API invariant.

### [REJECT] reasons-lib-cli-monkeypatch-does-not-reach-canonical-internals
Monkeypatching a name on `reasons_lib.cli` rebinds only the shim's attribute; internal calls inside `reasons.cli` resolve from `reasons.cli.__dict__` and do not see the patch.
- Source: summaries/2026/11/19/reasons_lib-cli.md
- Rejected: duplicate: same namespace-lookup principle as api-llm-mocking-patches-consumer-namespace; shim-patching behavior is a standard consequence.



---

**Generated:** 2026-11-19 (batch 9/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] cluster-shim-reverse-import-list-is-mirror-artifact
The reported reverse-importer list for `reasons_lib.cluster` includes `reasons/cluster.py`, but a literal reverse import would be circular; treat that list as API-surface mirroring, not a real dependency edge.
- Source: summaries/2026/11/19/reasons_lib-cluster.md
- Rejected: meta: about a reported analysis artifact, not a codebase invariant.

### [REJECT] cluster-shim-is-pure-reexport
Duplicate of existing `reasons-lib-is-uniform-star-reexport-shim` / `reasons-lib-is-pure-reexport-shim`.
- Source: summaries/2026/11/19/reasons_lib-cluster.md

### [REJECT] cluster-shim-requires-target-importable
Duplicate of existing `reasons-lib-requires-reasons-package` / `shim-fails-fast-without-fallback`.
- Source: summaries/2026/11/19/reasons_lib-cluster.md

### [REJECT] cluster-shim-preserves-identity
Duplicate of existing `star-import-guarantees-object-identity` / `shim-preserves-object-identity`.
- Source: summaries/2026/11/19/reasons_lib-cluster.md

### [REJECT] reasons-api-imports-compact-via-legacy-shim
`reasons/api.py` imports compact functionality through the legacy `reasons_lib.compact` path rather than directly from `reasons.compact`.
- Source: summaries/2026/11/19/reasons_lib-compact.md
- Rejected: trivial: module-import-path detail, not a durable architecture invariant.

### [REJECT] compact-shim-is-pure-reexport
Duplicate of existing `reasons-lib-is-uniform-star-reexport-shim` / `reasons-lib-is-pure-reexport-shim`.
- Source: summaries/2026/11/19/reasons_lib-compact.md

### [REJECT] compact-shim-requires-target
Duplicate of existing `reasons-lib-requires-reasons-package` / `shim-fails-fast-without-fallback`.
- Source: summaries/2026/11/19/reasons_lib-compact.md

### [REJECT] compact-shim-preserves-object-identity
Duplicate of existing `star-import-guarantees-object-identity` / `shim-preserves-object-identity`.
- Source: summaries/2026/11/19/reasons_lib-compact.md

### [REJECT] compact-shim-public-surface-is-target-defined
Duplicate of existing `reasons-lib-star-import-tracks-public-namespace`.
- Source: summaries/2026/11/19/reasons_lib-compact.md

### [REJECT] compact-shim-import-triggers-full-target-import
Duplicate of existing `reasons-lib-shim-import-is-atomic-eager`.
- Source: summaries/2026/11/19/reasons_lib-compact.md

### [REJECT] contradictions-shim-lacks-own-all
`reasons_lib.contradictions` does not define `__all__`, so accessing `reasons_lib.contradictions.__all__` raises `AttributeError` even if `reasons.contradictions.__all__` exists.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: absence of `__all__` and the resulting standard `AttributeError` is source-visible implementation detail.

### [REJECT] contradictions-shim-docstring-differs-from-canonical
`reasons_lib.contradictions.__doc__` is the shim migration text, not the canonical `reasons.contradictions` docstring.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md
- Rejected: trivial: docstring wording difference is not a durable code invariant.

### [REJECT] contradictions-shim-is-compat-shim
Duplicate of existing `reasons-lib-is-uniform-star-reexport-shim` / `reasons-lib-is-pure-reexport-shim`.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md

### [REJECT] contradictions-shim-requires-target
Duplicate of existing `reasons-lib-requires-reasons-package` / `shim-fails-fast-without-fallback`.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md

### [REJECT] contradictions-shim-exports-identical-objects
Duplicate of existing `star-import-guarantees-object-identity` / `shim-preserves-object-identity`.
- Source: summaries/2026/11/19/reasons_lib-contradictions.md

### [REJECT] derive-shim-name-rebinding-does-not-affect-canonical-internals
Reassigning a name on `reasons_lib.derive` only changes the shim namespace; functions inside `reasons.derive` that reference their own module global continue to see the original binding.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: generic Python module-namespace semantics, not a project-specific invariant.

### [REJECT] derive-shim-import-executes-target-side-effects-once
Importing `reasons_lib.derive` causes `reasons.derive` module-level initialization/side effects to run once, not duplicated per import path.
- Source: summaries/2026/11/19/reasons_lib-derive.md
- Rejected: trivial: standard Python import-caching behavior, not a durable project invariant.

### [REJECT] derive-shim-is-pure-reexport
Duplicate of existing `reasons-lib-is-uniform-star-reexport-shim` / `reasons-lib-is-pure-reexport-shim`.
- Source: summaries/2026/11/19/reasons_lib-derive.md

### [REJECT] derive-shim-preserves-identity
Duplicate of existing `star-import-guarantees-object-identity` / `shim-preserves-object-identity`.
- Source: summaries/2026/11/19/reasons_lib-derive.md

### [REJECT] derive-shim-requires-target
Duplicate of existing `reasons-lib-requires-reasons-package` / `shim-fails-fast-without-fallback`.
- Source: summaries/2026/11/19/reasons_lib-derive.md

### [REJECT] derive-shim-emits-no-deprecation-warning
Duplicate of existing `reasons-lib-shim-emits-no-deprecation-warning`.
- Source: summaries/2026/11/19/reasons_lib-derive.md

### [REJECT] derive-shim-has-no-all
Already covered by existing `reasons-lib-is-uniform-star-reexport-shim` and related star-import beliefs.
- Source: summaries/2026/11/19/reasons_lib-derive.md

### [REJECT] export-card-shim-consumers-include-api-and-export-card-tests
Known first-party importers of `reasons_lib.export_card` include `reasons/api.py` and `tests/test_export_card.py`, so both must migrate before the shim can be removed.
- Source: summaries/2026/11/19/reasons_lib-export_card.md
- Rejected: trivial: importer lists are directly visible from imports, and the migration/removal requirement is an editorial plan rather than a durable invariant.

### [REJECT] export-card-shim-is-shim
Duplicate of existing `reasons-lib-is-uniform-star-reexport-shim` / `reasons-lib-is-pure-reexport-shim`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md

### [REJECT] export-card-shim-requires-canonical-module
Duplicate of existing `reasons-lib-requires-reasons-package` / `shim-fails-fast-without-fallback`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md

### [REJECT] export-card-shim-reexports-only-public-names
Duplicate of existing `star-import-shim-skips-underscore-names` / `shim-star-import-respects-target-all`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md

### [REJECT] export-card-shim-preserves-identity
Duplicate of existing `star-import-guarantees-object-identity` / `shim-preserves-object-identity`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md

### [REJECT] export-card-shim-has-no-error-handling
Duplicate of existing `shim-propagates-exceptions-unchanged`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md

### [REJECT] export-card-shim-defines-no-own-names
Duplicate of existing `reasons-lib-owns-no-unique-code` / `reasons-lib-is-uniform-star-reexport-shim`.
- Source: summaries/2026/11/19/reasons_lib-export_card.md


---

**Generated:** 2026-11-19 (batch 10/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] export-markdown-shim-is-covered-by-exporter-tests
Several exporter-related test modules load through `reasons_lib.export_markdown`, making the shim part of the regression surface for export, metadata, roundtrip, and source-type behavior.
- Source: summaries/2026/11/19/reasons_lib-export_markdown.md
- Rejected: trivial: test-module import/coverage inventory is directly visible from test imports and is not an architectural contract.

### [REJECT] hf-shim-is-covered-by-hf-import-tests
`tests/test_hf_default_org.py` and `tests/test_import_hf.py` exercise HF behavior through the `reasons_lib.hf` shim, not only through `reasons.hf`.
- Source: summaries/2026/11/19/reasons_lib-hf.md
- Rejected: trivial: which tests import a shim is an import/coverage fact, not a durable codebase invariant.

### [REJECT] hf-shim-noqa-suppresses-f401-and-f403
`reasons_lib/hf.py` uses `# noqa: F401,F403` to suppress both unused-import and star-import lint warnings for its deliberate wildcard re-export.
- Source: summaries/2026/11/19/reasons_lib-hf.md
- Rejected: trivial: a lint-suppression comment is an obvious local implementation detail.

### [REJECT] reasons-api-and-pg-import-reasons-lib-import-agent
`reasons/api.py` and `reasons/pg.py` import agent-import functionality through the legacy `reasons_lib.import_agent` shim, creating a canonical-to-legacy dependency for this module.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md
- Rejected: trivial: an import relationship through a shim is directly visible from imports and is not a durable architectural invariant.

### [ACCEPTED] import-agent-shim-star-import-does-not-forward-all
Because `from reasons.import_agent import *` binds names but not `__all__`, `reasons_lib.import_agent.__all__` is absent unless the canonical `__all__` explicitly includes `"__all__"`, so downstream wildcard imports from the shim use the non-underscore rule instead of the canonical `__all__` contract.
- Source: summaries/2026/11/19/reasons_lib-import_agent.md

### [REJECT] import-beliefs-shim-is-covered-by-six-test-modules
Six test modules (`test_import_beliefs`, `test_metadata`, `test_metadata_extended`, `test_nogood_id`, `test_roundtrip`, `test_source_type`) import through the `reasons_lib.import_beliefs` shim, making it part of their regression surface.
- Source: summaries/2026/11/19/reasons_lib-import_beliefs.md
- Rejected: trivial: test-module import inventory with a specific count, visible directly from imports

### [REJECT] reasons-lib-llm-is-load-bearing-for-canonical-llm-callers
At least fifteen first-party modules under `reasons/` (including `api.py`, `cli.py`, `repair.py`, and `verify.py`) import LLM functionality through `reasons_lib.llm` rather than directly from `reasons.llm`, so the shim is load-bearing for production LLM call sites.
- Source: summaries/2026/11/19/reasons_lib-llm.md
- Rejected: ephemeral: specific module-count inventory that changes with code edits



---

**Generated:** 2026-11-19 (batch 11/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] reasons-cli-to-mcp-client-shim-chain
The canonical `reasons/cli.py` imports MCP client symbols through `reasons_lib.mcp_client`, creating the chain `reasons.cli -> reasons_lib.mcp_client -> reasons.mcp_client` that stays acyclic only while `reasons.mcp_client` does not import `reasons.cli`.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md

### [REJECT] mcp-client-shim-is-pure-star-reexport
This duplicates existing beliefs that every `reasons_lib` shim is a pure `from reasons.X import *` re-export with no local logic.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md

### [REJECT] mcp-client-shim-propagates-import-errors
This duplicates existing beliefs that shims have no error handling and propagate canonical import failures unchanged.
- Source: summaries/2026/11/19/reasons_lib-mcp_client.md

### [REJECT] shim-module-object-is-distinct-from-canonical
A `reasons_lib.<module>` shim is a distinct module object from `reasons.<module>` (separate `sys.modules` entry, `__name__`, `__file__`, and `__doc__`), even though re-exported class/function objects are identical.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: generic Python module-identity behavior, not a project-specific invariant

### [REJECT] shim-star-import-leaks-incidental-imports
Because the shim uses `import *` and defines no local `__all__`, non-underscore names imported at top level by the canonical module (such as `json` or `FastMCP`) also become attributes of the `reasons_lib` shim unless the canonical module defines `__all__`.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md
- Rejected: trivial: standard `import *` behavior, evident from the shim's imports and lack of `__all__`.

### [REJECT] mcp-server-tests-enter-through-shim
This duplicates existing beliefs that deleting `reasons_lib` shims breaks test imports and that the test suite validates the shim import chain.
- Source: summaries/2026/11/19/reasons_lib-mcp_server.md

### [REJECT] shim-does-not-redeclare-all
The `reasons_lib.metadata` shim does not define its own `__all__`, so `from reasons_lib.metadata import *` exports all non-underscore names present in the shim and can diverge from `reasons.metadata.__all__` if that list contains underscore names or omits non-underscore names.
- Source: summaries/2026/11/19/reasons_lib-metadata.md
- Rejected: trivial: absence of `__all__` and resulting star-import semantics are obvious from the shim file.

### [REJECT] metadata-shim-is-backward-compatibility-alias
This duplicates existing beliefs that `reasons_lib/<module>.py` is a pure compatibility alias re-exporting `reasons/<module>.py`.
- Source: summaries/2026/11/19/reasons_lib-metadata.md

### [REJECT] metadata-shim-emits-no-deprecation-signal
This duplicates existing beliefs that importing any `reasons_lib` shim emits no `DeprecationWarning` or runtime migration signal.
- Source: summaries/2026/11/19/reasons_lib-metadata.md

### [ACCEPTED] shim-reexports-canonical-module-path
Symbols re-exported through `reasons_lib.mind_service` retain `__module__ == "reasons.mind_service"`, so pickles created via the legacy import path reference the canonical module and do not depend on the shim surviving.
- Source: summaries/2026/11/19/reasons_lib-mind_service.md

### [REJECT] mind-service-shim-is-load-bearing-for-reasons-api
This duplicates existing beliefs that `reasons/api.py` imports through `reasons_lib` and that removing the shims would break internal production code.
- Source: summaries/2026/11/19/reasons_lib-mind_service.md

### [REJECT] mind-service-symbols-are-identical-across-paths
This duplicates existing beliefs that shim re-exports preserve object identity across `reasons` and `reasons_lib` import paths.
- Source: summaries/2026/11/19/reasons_lib-mind_service.md

### [REJECT] network-internal-modules-use-reasons-lib-shim
This duplicates the existing belief that at least nine canonical `reasons/` modules (including `api.py`, `storage.py`, `check_stale.py`, `compact.py`, `export_card.py`, `export_markdown.py`, `import_agent.py`, and `import_beliefs.py`) import from `reasons_lib.*`.
- Source: summaries/2026/11/19/reasons_lib-network.md

### [REJECT] network-shim-preserves-object-identity
This duplicates existing beliefs that `reasons_lib.<module>.X is reasons.<module>.X` for re-exported names.
- Source: summaries/2026/11/19/reasons_lib-network.md

### [REJECT] network-shim-exports-only-public-names
This duplicates existing beliefs that `import *` skips underscore-prefixed names unless the canonical module explicitly includes them in `__all__`.
- Source: summaries/2026/11/19/reasons_lib-network.md


---

**Generated:** 2026-11-19 (batch 12/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] reasons-lib-pg-is-distinct-module-object
`reasons_lib.pg` and `reasons.pg` are distinct module objects in `sys.modules` (not aliases), while each name re-exported by the shim references the same function/class objects as the canonical module.
- Source: summaries/2026/11/19/reasons_lib-pg.md
- Rejected: trivial: module identity follows directly from having separate shim and canonical modules.

### [REJECT] reasons-api-cli-import-propose-update-via-shim
`reasons/api.py` and `reasons/cli.py` import `propose_update` through `reasons_lib.propose_update` rather than directly from `reasons.propose_update`.
- Source: summaries/2026/11/19/reasons_lib-propose_update.md
- Rejected: trivial: specific import path, exactly the kind of obvious dependency fact excluded.

### [ACCEPTED] shim-namespace-rebinding-does-not-affect-canonical
Rebinding a name in a `reasons_lib.<module>` shim (e.g. `monkeypatch.setattr("reasons_lib.repair_premises.foo", fake)`) affects only the shim namespace; code inside `reasons.<module>` still resolves its original module-level name.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md

### [ACCEPTED] shim-does-not-copy-canonical-all-attribute
The `reasons_lib` shims do not set their own `__all__`; when `reasons.<module>` defines `__all__`, the star import copies the listed names but not the `__all__` attribute itself, so `reasons_lib.<module>.__all__` is normally absent.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md

### [REJECT] reasons-api-imports-pubsub-via-shim
Duplicate of existing `reasons-api-imports-through-reasons-lib-shim`, which already records that `reasons/api.py` imports pubsub through the legacy shim.
- Source: summaries/2026/11/19/reasons_lib-pubsub.md

### [REJECT] reexported-objects-retain-canonical-module-attribute
Functions and classes defined in a canonical `reasons.<module>` and re-exported by its `reasons_lib` shim keep their original `__module__` value (e.g. `reasons.repair`), not the shim module path.
- Source: summaries/2026/11/19/reasons_lib-repair.md
- Rejected: duplicate: same claim as shim-reexports-retain-canonical-module-attribute

### [REJECT] reasons-lib-repair-is-compat-shim
Duplicate of existing shim-package beliefs (`reasons-lib-is-frozen-transitional-layer`, `reasons-lib-is-uniform-star-reexport-shim`).
- Source: summaries/2026/11/19/reasons_lib-repair.md

### [REJECT] reasons-lib-repair-premises-emits-no-deprecation-warning
Duplicate of existing `reasons-lib-shim-emits-no-deprecation-warning` / `no-deprecation-warning-emitted-on-shim-import`.
- Source: summaries/2026/11/19/reasons_lib-repair_premises.md


---

**Generated:** 2026-11-19 (batch 13/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] shim-reexports-retain-canonical-module-attribute
Functions and classes re-exported through a `reasons_lib` shim retain their original `__module__` value from `reasons.<module>`, so introspection and tracebacks show the canonical module path rather than the shim path.
- Source: summaries/2026/11/19/reasons_lib-review.md

### [ACCEPTED] shim-star-import-exposes-transitive-public-names
When a canonical `reasons/<module>.py` lacks `__all__`, its `reasons_lib` shim re-exports every non-underscore name in the canonical module’s namespace, including names imported from other modules, so the shim’s public surface can be broader than the canonical module’s intended API.
- Source: summaries/2026/11/19/reasons_lib-review_premises.md

### [ACCEPTED] shim-cross-tree-edge-constrains-review-imports
Because `reasons/api.py` imports `reasons_lib.review` and `reasons_lib.review` imports `reasons.review`, the effective import graph has a cross-tree edge; `reasons/review.py` must not import names from `reasons/__init__.py` at module level to avoid circular-import failures.
- Source: summaries/2026/11/19/reasons_lib-review.md



---

**Generated:** 2026-11-19 (batch 14/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] reasons-lib-storage-shim-has-no-all
`reasons_lib/storage.py` defines no `__all__`, so `from reasons_lib.storage import *` uses public-name filtering rather than mirroring `reasons.storage.__all__`.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: absence of `__all__` and standard star-import filtering are obvious implementation details

### [REJECT] reasons-lib-storage-star-import-can-expose-canonical-imported-dependencies
Because the shim uses a bare wildcard import and defines no `__all__`, public imported dependencies in `reasons.storage` can also become available through `reasons_lib.storage` when `reasons.storage` lacks `__all__`.
- Source: summaries/2026/11/19/reasons_lib-storage.md
- Rejected: trivial: describes standard wildcard-import leakage rather than a durable architectural invariant

### [REJECT] reasons-lib-storage-import-errors-propagate-unchanged
This duplicates the existing belief that shim import failures propagate unchanged rather than being wrapped or translated.
- Source: summaries/2026/11/19/reasons_lib-storage.md

### [REJECT] reasons-lib-storage-reexports-same-objects
This duplicates the existing belief that shim re-exports bind the same function/class objects rather than copies.
- Source: summaries/2026/11/19/reasons_lib-storage.md

### [REJECT] reasons-lib-verify-shim-has-no-all
`reasons_lib/verify.py` defines no `__all__`, so downstream star imports from the shim use Python public-name filtering instead of mirroring `reasons.verify.__all__`.
- Source: summaries/2026/11/19/reasons_lib-verify.md
- Rejected: trivial: same low-value `__all__` implementation detail for another shim

### [REJECT] reasons-lib-verify-star-export-over-exports-canonical-all
The entry's over-export claim is unsupported: with no shim `__all__`, downstream star import can drop underscore names from a canonical `__all__`, but it cannot bind names the wildcard import did not import.
- Source: summaries/2026/11/19/reasons_lib-verify.md

### [REJECT] reasons-lib-verify-canonical-implementation-is-reasons-verify
This duplicates existing beliefs that `reasons/` is canonical and `reasons_lib/` is the legacy shim.
- Source: summaries/2026/11/19/reasons_lib-verify.md

### [REJECT] reasons-lib-verify-emits-no-deprecation-warning
This duplicates the existing belief that the shim emits no runtime deprecation warning.
- Source: summaries/2026/11/19/reasons_lib-verify.md

### [REJECT] scan-repo-reasons-lib-is-legacy-mirror
This duplicates existing beliefs that `reasons_lib/` is a near-duplicate legacy/compat mirror.
- Source: summaries/2026/11/19/scan-repo.md

### [REJECT] scan-repo-two-entry-points
This duplicates the existing belief about the two console entry points.
- Source: summaries/2026/11/19/scan-repo.md

### [REJECT] scan-repo-core-tms-module-list-is-speculative
The entry presents the core module list as likely rather than as an established invariant, so it is not a reliable codebase claim.
- Source: summaries/2026/11/19/scan-repo.md

### [REJECT] scan-repo-sqlite-default-postgres-optional
This duplicates existing beliefs about interchangeable SQLite/Postgres backends.
- Source: summaries/2026/11/19/scan-repo.md

### [REJECT] tests-init-makes-tests-a-package
The empty `tests/__init__.py` makes `tests` a regular Python package so test modules import as `tests.<module>`.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: trivial: package status from an empty `__init__.py` is obvious from file structure

### [ACCEPTED] tests-init-must-remain-empty
Executable code must not be added to `tests/__init__.py` because it would run on every test-module import before pytest fixtures load.
- Source: summaries/2026/11/19/tests-__init__.md

### [REJECT] tests-fixtures-live-in-conftest-not-init
Shared pytest fixtures are provided by `tests/conftest.py`, not by `tests/__init__.py`.
- Source: summaries/2026/11/19/tests-__init__.md
- Rejected: trivial: standard pytest file organization visible from directory structure

### [ACCEPTED] production-packages-do-not-import-tests
Neither the `reasons/` nor `reasons_lib/` package imports the `tests` package.
- Source: summaries/2026/11/19/tests-__init__.md

### [ACCEPTED] pg-marker-registration-does-not-skip-tests
`pytest_configure` registers the `pg` marker for selection and warning suppression, but the marker itself does not skip tests; skipping requires `skip_no_pg` or `pytest.skip` in the fixture.
- Source: summaries/2026/11/19/tests-conftest.md

### [ACCEPTED] pg-fixture-rolls-back-before-delete-cleanup
The `pg_api` fixture calls `api.conn.rollback()` before its DELETE cleanup to clear aborted transaction state left by failed test SQL.
- Source: summaries/2026/11/19/tests-conftest.md

### [ACCEPTED] pg-fixture-teardown-close-not-guaranteed-on-delete-error
The `pg_api` teardown has no try/finally, so a failure in one of its DELETE statements can prevent `api.close()` from running.
- Source: summaries/2026/11/19/tests-conftest.md

### [REJECT] pg-fixture-lazily-imports-pgapi
This duplicates the existing belief that `PgApi` is imported lazily inside the fixture to avoid collection-time database imports.
- Source: summaries/2026/11/19/tests-conftest.md

### [REJECT] pg-api-fixture-skips-when-database-url-missing
This duplicates the existing belief that PG tests are gated by `DATABASE_URL` availability.
- Source: summaries/2026/11/19/tests-conftest.md

### [REJECT] pg-fixture-cleans-five-tables-in-fk-safe-order
This duplicates the existing beliefs about the five-table cleanup set and FK-safe deletion order.
- Source: summaries/2026/11/19/tests-conftest.md


---

**Generated:** 2026-11-19 (batch 15/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] derived-access-tags-union-antecedent-tags
Derived nodes' `metadata["access_tags"]` is the union of their explicit tags and all tags reachable through antecedent justifications, including transitive chains and diamond graphs.
- Source: summaries/2026/11/19/tests-test_access_tags.md
- Rejected: duplicate: duplicates existing `access-tags-inherit-by-union` / `access-tags-union-inheritance` / `access-tags-are-monotonically-inherited`.

### [REJECT] trace-access-tags-returns-union-or-keyerror
`Network.trace_access_tags(node_id)` returns the union of tags from the node and its antecedents, returns `[]` for an untagged node, and raises `KeyError` for a missing node.
- Source: summaries/2026/11/19/tests-test_access_tags.md
- Rejected: duplicate: union-of-tags semantics already covered by `access-tags-inherit-by-union`; remaining details are trivial.

### [REJECT] api-trace-access-tags-returns-structured-result
`api.trace_access_tags(node_id, db_path=...)` returns a dict with `node_id` and `access_tags` after enforcing `visible_to` visibility.
- Source: summaries/2026/11/19/tests-test_access_tags.md
- Rejected: duplicate: visibility enforcement is covered by `access-tags-gate-all-query-commands` and dict return by `api-functions-return-dicts`.

### [REJECT] add-justification-appends-without-replacing
Adding a justification to a node appends the new justification to the existing list rather than replacing prior justifications, and new justifications are appended in call order.
- Source: summaries/2026/11/19/tests-test_add_justification.md
- Rejected: duplicate: duplicates existing `add-justification-appends-not-replaces`.

### [ACCEPTED] add-justification-returns-truth-transition-and-changed
`Network.add_justification(...)` returns a dict containing `node_id`, `old_truth_value`, `new_truth_value`, and `changed`, where `changed` includes the target and any cascaded dependents whose truth value changed.
- Source: summaries/2026/11/19/tests-test_add_justification.md

### [REJECT] add-justification-missing-node-raises-keyerror
`Network.add_justification` raises `KeyError` with a message matching `"not found"` when the target node does not exist.
- Source: summaries/2026/11/19/tests-test_add_justification.md
- Rejected: duplicate: Same claim as existing `add-justification-nonexistent-node-raises-keyerror`

### [ACCEPTED] premises-accept-added-justifications
Premise nodes are not immutable: `Network.add_justification` can add an SL justification to an existing premise node.
- Source: summaries/2026/11/19/tests-test_add_justification.md

### [ACCEPTED] premise-count-reflects-single-justification-arity
The `premise_count` field in add results is N for a conjunctive SL with N antecedents, 1 for `any_mode` expansions regardless of original N, and 0 for premise nodes.
- Source: summaries/2026/11/19/tests-test_any_mode.md

### [ACCEPTED] any-mode-survives-single-retraction
With `any_mode=True`, a derived node remains IN after any single premise is retracted and flips OUT only when all expanded premises are retracted.
- Source: summaries/2026/11/19/tests-test_any_mode.md

### [REJECT] init-db-refuses-existing-path-unless-force
`api.init_db(db_path)` returns `{"created": True}` for a new path, raises `FileExistsError` when the path already exists, and overwrites only when `force=True`.
- Source: summaries/2026/11/19/tests-test_api.md
- Rejected: duplicate: Existing belief `api-init-requires-force-to-overwrite` already states that `api.init_db()` refuses to overwrite unless `force=True`.

### [ACCEPTED] api-propagate-repairs-external-storage-corruption
`api.propagate()` recomputes truth values and repairs the store even when a node's stored `truth_value` was corrupted by direct `Storage` writes outside the API.
- Source: summaries/2026/11/19/tests-test_api.md

### [ACCEPTED] explain-node-distinguishes-cycles-from-diamonds
`api.explain_node` returns ordered `steps` with reasons such as `"premise"` and uses `"circular …"` only for genuine cycles, never labeling diamond-shaped dependencies as circular.
- Source: summaries/2026/11/19/tests-test_api.md

### [ACCEPTED] add-source-deduplicates-against-node-source
`api.add_source(node_id, source_ref, ...)` deduplicates sources against those already supplied through `add_node(..., source=...)`.
- Source: summaries/2026/11/19/tests-test_api.md

### [ACCEPTED] api-search-regex-bad-pattern-raises-valueerror
`api.search(..., regex=True)` compiles the query with Python `re` and raises `ValueError` for invalid regular expressions.
- Source: summaries/2026/11/19/tests-test_api.md

### [ACCEPTED] search-pipe-is-or-operator
In `api.search`, a `|` in the query is treated as an OR operator at the query layer.
- Source: summaries/2026/11/19/tests-test_api.md

### [REJECT] ask-tool-loop-bounded-to-three-rounds
`ask()` in default full mode performs at most three tool-call rounds before final synthesis; on final-synthesis timeout, error, or another tool call, it falls back to accumulated belief/search context.
- Source: summaries/2026/11/19/tests-test_ask.md
- Rejected: duplicate: already covered by `ask-tool-loop-capped-at-three`, `ask-tool-call-budget-is-three`, and `ask-tool-loop-max-three-rounds`

### [REJECT] ask-simple-mode-skips-llm-when-no-context
`ask(..., simple=True)` returns `NO_BELIEFS_MSG` without invoking the LLM when there are no matching beliefs and no matching source documents.
- Source: summaries/2026/11/19/tests-test_ask.md
- Rejected: duplicate: outcome `NO_BELIEFS_MSG` on no matches is already captured by `ask-empty-network-returns-no-beliefs-message`

### [ACCEPTED] ask-sources-can-satisfy-query-without-beliefs
When `sources_db` is provided, source-document search results alone can produce an answer even when belief search returns no matches; with neither belief nor source matches, `ask()` returns `NO_BELIEFS_MSG`.
- Source: summaries/2026/11/19/tests-test_ask.md

### [REJECT] prompt-builders-gate-tool-definitions-and-citation
Prompt builders gate tool definitions: only `build_ask_prompt` includes `search_beliefs`; simple mode defaults to citation instructions and switches to plain natural language when `natural=True`.
- Source: summaries/2026/11/19/tests-test_ask.md
- Rejected: duplicate: combines existing `build-ask-prompt-includes-search-beliefs-tool` and `ask-natural-mode-strips-metadata-and-cite`


---

**Generated:** 2026-11-19 (batch 16/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] mcp-instructions-concatenated-across-bridges
`_build_mcp_instructions` concatenates `get_instructions()` from all MCP bridges and returns an empty string when every bridge has empty instructions.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: already captured by existing `build-mcp-instructions-concatenates-bridges`

### [REJECT] ask-prompt-data-source-header-conditional
`build_ask_prompt` inserts the `"Data Source Instructions"` header only when `mcp_instructions` is non-empty; empty instructions omit the header.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: already captured by existing `ask-prompt-data-source-instructions-header`

### [ACCEPTED] ask-prompt-tools-section-overrides-default
A caller-supplied `tools_section` fully replaces the default prompt tools block, including removing the default `"one tool available"` wording.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md

### [ACCEPTED] ask-mcp-dispatch-strips-tool-key
When dispatching an MCP tool call, `ask()` passes `bridge.call_tool(name, args)` with the parsed tool-call dict minus the `"tool"` key.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md

### [REJECT] ask-prompt-avoids-raw-template-braces
The rendered ask prompt must not contain raw `{{` template placeholder tokens.
- Source: summaries/2026/11/19/tests-test_ask_mcp.md
- Rejected: duplicate: already captured by existing `ask-prompt-no-template-placeholders`

### [ACCEPTED] trace-assumptions-deduplicates-premise-roots
`Network.trace_assumptions(node_id)` recursively follows justifications to premise IDs, returns the node itself when it is a premise, and de-duplicates repeated premises across paths.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [ACCEPTED] find-culprits-expands-derived-nodes-to-premises
`Network.find_culprits` expands derived nogood members to their supporting premises and returns each culprit dict with `premise` and `would_resolve` keys.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [ACCEPTED] find-culprits-sorts-by-dependent-count
`Network.find_culprits` orders culprits by dependent count ascending so a premise with fewer dependents appears before one with more.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [REJECT] inactive-nogood-returns-empty-and-no-log
If any node in a nogood is already `OUT`, `Network.add_nogood` returns an empty changed list and writes no backtrack log entry.
- Source: summaries/2026/11/19/tests-test_backtracking.md
- Rejected: duplicate: Already covered by `add-nogood-inactive-when-either-node-out`, which states no log entry and no retraction.

### [ACCEPTED] add-nogood-logs-backtrack-with-culprit
A real `Network.add_nogood` backtrack writes a log entry with `action == "backtrack"` and a `"culprit"` field.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [ACCEPTED] api-trace-assumptions-returns-premises-dict
`reasons.api.trace_assumptions` returns a dict with `"node_id"` and a de-duplicated `"premises"` list.
- Source: summaries/2026/11/19/tests-test_backtracking.md

### [ACCEPTED] assign-topics-groups-unmatched-under-other
`_assign_topics` groups node IDs matching a topic under that topic, places all non-matching IDs under `"Other"`, and returns `{}` for an empty node list.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [ACCEPTED] page-name-normalizes-to-lowercase-hyphens
`_page_name` normalizes spaces and special characters to hyphens and falls back to `"other"` when the normalized result is empty.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [ACCEPTED] format-node-links-only-known-cross-references
`_format_node` renders cross-references as `[node-id](page.md#node-id)` only when the target exists in `node_to_page`; unknown node IDs remain plain text.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [ACCEPTED] format-node-renders-defeater-metadata
`_format_node` renders outlist defeaters with `**Defeated by:**`, `defeater_type`, and optional `defeat_reason_type` when `all_details` is supplied.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [ACCEPTED] linkify-skips-same-page-and-existing-links
`_linkify` does not link references that belong to the current page and does not double-link content already represented as a Markdown link.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [ACCEPTED] generate-wiki-page-calls-llm-invoke-model
`generate_wiki_page` calls `reasons.llm.invoke_model` and builds a prompt containing the topic name and node IDs.
- Source: summaries/2026/11/19/tests-test_build_wiki.md

### [ACCEPTED] hash-file-returns-lowercase-sha256-hexdigest
`hash_file(path)` returns a lowercase SHA-256 hex digest of the file bytes.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] check-stale-returns-result-list-and-upgraded-count
`check_stale(net, ...)` returns a tuple whose first element is the list of stale-result dicts and whose second element is the upgraded-hash count.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] hash-sources-skips-missing-source-silently
`hash_sources` silently skips nodes whose source file is missing and emits no result for them.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] hash-sources-results-include-was-empty
`hash_sources` returns result dicts containing at least `node_id` and `was_empty`, where `was_empty` is `True` only when the previous hash was empty.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] hash-sources-force-rehashes-existing
With `force=True`, `hash_sources` rehashes nodes that already have a non-empty hash and sets `was_empty` to `False` in the result.
- Source: summaries/2026/11/19/tests-test_check_stale.md

### [ACCEPTED] source-deleted-result-preserves-old-hash-and-source
A `source_deleted` result preserves the node's `old_hash` and `source` values while setting `new_hash` and `source_path` to `None`.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [ACCEPTED] repos-none-still-reports-deleted-sources
`check_stale` accepts `repos=None` and still reports missing-source nodes as `source_deleted`.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [ACCEPTED] empty-file-with-wrong-hash-is-content-changed
An existing empty source file with a mismatched stored hash produces a `content_changed` result whose `new_hash` is the SHA-256 of empty content.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md

### [ACCEPTED] content-changed-populates-source-path-string
A `content_changed` result sets `source_path` to the resolved file path as a string, unlike `source_deleted` which leaves it `None`.
- Source: summaries/2026/11/19/tests-test_check_stale_issue25.md


---

**Generated:** 2026-11-19 (batch 17/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] classify-defeat-batch-result-has-classified-skipped-errors
`api.classify_defeat_reason_types` returns a dict with `classified`, `skipped`, and `errors`; classified entries contain `id` and `defeat_reason_type`, and skipped entries contain `reason`.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] classify-defeat-batch-empty-result-becomes-error
In `classify_defeat_reason_types`, an empty classifier result is appended to `result["errors"]` rather than raised or silently discarded.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] classify-defeat-skips-non-defeater-nodes
`classify_defeat_reason_types` ignores non-defeater nodes and does not attempt an LLM call when the database contains only non-defeaters.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] classify-defeat-parser-prefers-exact-match
`classify_defeat_reason` first returns an exact whole-output match against `DEFEAT_REASON_TYPES`, then extracts a member token from prose, otherwise returns `""`.
- Source: summaries/2026/11/19/tests-test_classify_defeaters.md

### [ACCEPTED] cli-idempotent-transitions-are-success
Retracting an already-OUT node or asserting an already-IN node exits `0` and prints `"already OUT"` or `"already IN"` respectively.
- Source: summaries/2026/11/19/tests-test_cli.md

### [REJECT] cli-read-commands-enforce-visible-to
The `status`, `show`, `explain`, `trace`, and `trace-access-tags` CLI commands enforce `--visible-to`; denied reads exit `1` with `"Access denied"`.
- Source: summaries/2026/11/19/tests-test_cli.md
- Rejected: duplicate: Covered by existing `access-tags-gate-all-query-commands` which already enumerates the same read commands (show, explain, trace, status, trace-access-tags) and their access-tag gating.

### [ACCEPTED] supersede-requires-exactly-one-input
The `supersede` command requires exactly one of `new_id` or `--text`; supplying both or neither is an error.
- Source: summaries/2026/11/19/tests-test_cli.md

### [ACCEPTED] cli-init-requires-force-to-overwrite
A second `init` without `--force` exits `1` and writes stderr containing `"--force"`.
- Source: summaries/2026/11/19/tests-test_cli.md

### [REJECT] cli-add-rejects-duplicate-ids
The `add` command rejects duplicate IDs with exit code `1` and stderr containing `"Error"`.
- Source: summaries/2026/11/19/tests-test_cli.md
- Rejected: duplicate: The duplicate-ID rejection semantics are already captured by `api-enforces-typed-preconditions`; the generic exit-1/"Error" CLI wrapper is a trivial restatement.

### [ACCEPTED] list-clusters-partitions-all-input-beliefs
`list_clusters` returns every input belief ID exactly once across clusters, and `cluster_beliefs` stats `cluster_sizes` sum to `len(beliefs)`.
- Source: summaries/2026/11/19/tests-test_cluster.md

### [ACCEPTED] list-clusters-result-shape
`list_clusters` returns a dict with `clusters` (each containing `beliefs`), `n_clusters`, and `embedding_model`.
- Source: summaries/2026/11/19/tests-test_cluster.md

### [REJECT] auto-k-default-and-clamp-semantics
`_auto_k(n)` defaults to `max(2, n // 5)`; `max_k` caps that auto-derived value, while explicit `n_clusters` wins and is clamped to `n`.
- Source: summaries/2026/11/19/tests-test_cluster.md
- Rejected: duplicate: existing beliefs `auto-k-floor-is-two`, `auto-k-formula-is-n-div-5-capped-at-20`, and `auto-k-max-k-caps-cluster-count` already capture the auto-k formula and clamp behavior.

### [ACCEPTED] list-clusters-small-corpus-guard
For a two-belief input, `list_clusters` returns one cluster even though `_auto_k(2)` would yield `2`.
- Source: summaries/2026/11/19/tests-test_cluster.md

### [ACCEPTED] estimate-tokens-char-floor-four-min-one
`estimate_tokens(text)` returns `max(1, len(text) // 4)`, using character count rather than word count.
- Source: summaries/2026/11/19/tests-test_compact.md

### [ACCEPTED] compact-orders-nodes-by-dependent-count
In compact output, nodes with more dependents appear before nodes with fewer dependents, e.g. a node with one dependent appears before a node with zero dependents.
- Source: summaries/2026/11/19/tests-test_compact.md

### [ACCEPTED] compact-renders-dependency-arrows-and-dependent-counts
Compact output renders dependencies as `<- <antecedent-id>` and nodes with dependents as `(N dependents)`.
- Source: summaries/2026/11/19/tests-test_compact.md

### [ACCEPTED] api-detect-contradictions-result-has-checked-found-applied
`api.detect_contradictions` returns a result dict containing `checked`, `found`, and `applied` counts.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] contradiction-detection-honors-batch-size
`detect_contradictions` uses `batch_size` to determine the number of LLM calls; three IN beliefs with `batch_size=2` trigger two `subprocess.run` calls.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] contradiction-format-skips-missing-node-ids
`format_beliefs_for_contradiction_check` silently skips requested node IDs that are missing from the node dict.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] semantic-contradiction-detection-skips-single-belief
`detect_contradictions_semantic` returns `[]` without invoking `subprocess.run` when the network is empty or has only a single belief.
- Source: summaries/2026/11/19/tests-test_contradictions.md

### [ACCEPTED] detect-contradictions-forwards-timeout-to-subprocess
`detect_contradictions` forwards its `timeout` argument to `subprocess.run(..., timeout=...)`.
- Source: summaries/2026/11/19/tests-test_contradictions.md


---

**Generated:** 2026-11-19 (batch 18/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] dangling-dependent-tolerance-spans-all-mutation-entry-points
The dangling-dependent regression suite verifies tolerance through `retract()`, `assert_node()`, `challenge()`, and `add_justification()`-then-retract flows, so guarding only one propagation entry point is insufficient.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [ACCEPTED] dangling-warning-value-identifies-parent-node
When a dangling dependent is detected, the warning log entry’s `value` must include the parent node ID whose `dependents` set contained the missing ID.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md

### [REJECT] dangling-dependent-regression-tests-target-reasons-package-only
The dangling-dependent regression tests import from `reasons`, not `reasons_lib`, so they directly guard `reasons.network.Network` but do not directly guard parity with `reasons_lib`.
- Source: summaries/2026/11/19/tests-test_dangling_dependents.md
- Rejected: trivial: import path/test coverage caveat is obvious and not a durable behavioral invariant

### [ACCEPTED] dedup-supersession-require-existing-node-ids
Both `mark_duplicate` and `mark_superseded` require all referenced node IDs to exist; a missing source/canonical or old/new ID raises `KeyError` whose message matches `"not found"`.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] dedup-supersession-self-reference-validates-before-mutation
Self-reference in `mark_duplicate` or `mark_superseded` raises `ValueError` and leaves the node unchanged in `IN`, proving validation occurs before mutation.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] dedup-supersession-retract-reasons-are-deterministic
`mark_duplicate` writes a retract reason of the form `"Duplicate of <canonical-id>"`, and `mark_superseded` writes `"Superseded by <new-id>"`.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] dedup-supersession-metadata-persists-in-export-and-show
The `duplicate_of` / `superseded_by` and `retract_reason` metadata written by deduplication and supersession persist and are visible through both `show_node` and `export_network`.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] supersession-overwrites-prior-retract-reason
Marking an already-`OUT` node as superseded overwrites its previous retract reason with `"Superseded by <new-id>"`.
- Source: summaries/2026/11/19/tests-test_deduplication.md

### [ACCEPTED] defeat-justification-encodes-defeat-via-outlist-append
`defeat_justification` implements defeat by appending the defeater ID to the target justification’s `outlist`; the justification record itself remains present.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] defeater-default-id-pattern
When `defeater_id` is omitted, `defeat_justification` generates the defeater ID as `"{defeater_type}-{node_id}-j{index}"`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] defeater-metadata-keys-identify-target
Every defeater node created by `defeat_justification` carries metadata keys `defeater_type`, `defeats_node`, and `defeats_justification`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] defeater-types-roundtrip-through-metadata
The defeat API accepts defeater types `invalid-inference`, `over-generalizes`, `duplicate-of`, and `superseded-by`, and stores the supplied value verbatim in `metadata.defeater_type`.
- Source: summaries/2026/11/19/tests-test_defeat_justification.md

### [ACCEPTED] scope-finding-creates-in-premise-scope-belief
`defeat_with_scope` creates one scope-belief node per scope finding, and each scope belief is an `IN` premise with an empty `justifications` list.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] scope-belief-metadata-links-antecedent-and-defeater
Scope-belief metadata records `scope_of` as the antecedent and `for_defeater` as the generated defeater ID.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] scope-belief-ids-are-unique-across-defeats
Scope-belief IDs are uniquely generated per defeat; multiple defeats on the same node with different justification indices produce disjoint sets of `scope_belief_ids`.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] defeat-reason-type-omitted-is-empty-and-absent
When `defeat_reason_type` is omitted, `defeat_with_scope` returns `""` and does not store the key in defeater metadata; when supplied, the value is stored in defeater metadata.
- Source: summaries/2026/11/19/tests-test_defeat_with_scope.md

### [ACCEPTED] dependents-integrity-errors-distinguish-extra-vs-missing
`verify_dependents()` error strings distinguish bogus reverse entries containing `"extra"` from absent reverse entries containing `"missing"`.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md

### [REJECT] dependents-set-deduplicates-multiple-justifications
Because `dependents` is a set, multiple justifications referencing the same antecedent produce a single reverse-index entry rather than duplicate entries.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md
- Rejected: trivial: direct consequence of storing dependents in a set

### [ACCEPTED] convert-to-premise-removes-reverse-dependents-edge
After `convert_to_premise("b")`, `"b"` is no longer present in its former antecedents’ `dependents` sets.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md

### [ACCEPTED] summarize-registers-summary-as-dependent-of-covered-nodes
`summarize(..., over=[...])` registers the summary node as a dependent of every covered node.
- Source: summaries/2026/11/19/tests-test_dependents_integrity.md


---

**Generated:** 2026-11-19 (batch 19/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [ACCEPTED] derive-parser-supports-legacy-and-backtick-format
`parse_proposals` accepts both current `### DERIVE`/`### GATE` markdown blocks and legacy `### DERIVE:`/`**Antecedents**:` format, stripping backticks from IDs and lists.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] apply-gate-proposals-adds-unless-outlist
When `apply_proposals` applies a `gate` proposal, its `unless` entries become the applied node’s outlist.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] derive-validation-requires-at-least-two-antecedents
`validate_proposals` rejects any proposal with fewer than two antecedents as skipped.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] derive-topic-filter-uses-or-keyword-match
`_filter_by_topic` keeps nodes whose ID or text matches any whitespace-separated topic keyword, using OR semantics.
- Source: summaries/2026/11/19/tests-test_derive.md

### [ACCEPTED] write-proposals-file-roundtrips-through-parse-proposals
`parse_proposals(write_proposals_file(proposals))` reconstructs the same proposal shape, keeping writer and parser in sync.
- Source: summaries/2026/11/19/tests-test_derive.md

### [REJECT] derive-local-budget-has-floor-of-five
The derive prompt builder gives the local-belief section at least 5 slots even when agent sections consume the nominal budget.
- Source: summaries/2026/11/19/tests-test_derive_budget.md
- Rejected: duplicate: Same claim already exists as `budget-floor-is-five` (currently OUT) describing `_build_beliefs_section` guaranteeing local beliefs at least 5 slots.

### [ACCEPTED] build-beliefs-section-headers-are-regex-contract
`_build_beliefs_section` emits headers matching `Local beliefs (N beliefs, showing M)` and `Agent: <name> (N beliefs, showing M)`, which tests parse as an output contract.
- Source: summaries/2026/11/19/tests-test_derive_budget.md

### [ACCEPTED] beliefs-section-omits-empty-agent-and-local-sections
`_build_beliefs_section` omits the `Agent:` section when `agents=None` and omits `Local beliefs` when there are no local beliefs.
- Source: summaries/2026/11/19/tests-test_derive_budget.md

### [REJECT] build-beliefs-section-safe-on-empty-or-missing-agent-refs
`_build_beliefs_section` returns a string without crashing for an empty network or when the agents mapping references nonexistent node IDs.
- Source: summaries/2026/11/19/tests-test_derive_budget.md
- Rejected: duplicate: covered by existing budget-section-graceful-emptiness; missing-ref handling is only a minor extension.

### [ACCEPTED] dialectical-default-ids-use-prefix-and-collision-suffix
Challenge nodes default to `challenge-<target>` with `-2`, `-3`, … on collision, and defense nodes default to `defense-<challenge_id>`.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] dialectical-edges-are-metadata-encoded
Challenge/defense relationships are encoded in node metadata via keys such as `challenge_target`, `challenges`, `defense_target`, and `defends`, not a dedicated edge table.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] dialectical-operations-raise-keyerror-for-missing-nodes
`Network.challenge` and `Network.defend` raise `KeyError` when the target or challenge node does not exist, rather than silently creating stubs.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] dialectical-state-survives-storage-roundtrip
A challenge+defend sequence and its metadata survive `Storage.save` → `Storage.load`, and retraction behavior remains the same after reload.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] list-nodes-challenged-filters-to-challenged-targets
`api.list_nodes(challenged=True)` returns only nodes that have been challenged, excluding challenge nodes themselves and unchallenged premises.
- Source: summaries/2026/11/19/tests-test_dialectical.md

### [ACCEPTED] show-omits-example-section-when-absent
CLI `show` emits an `Example:` section when metadata contains an example and omits that section entirely when no example exists.
- Source: summaries/2026/11/19/tests-test_example.md

### [REJECT] example-and-access-tags-coexist-in-metadata
An example and access tags can be stored together in node metadata without either field interfering with the other.
- Source: summaries/2026/11/19/tests-test_example.md
- Rejected: trivial: metadata fields coexisting is not a meaningful durable invariant

### [ACCEPTED] export-card-always-emits-yaml-frontmatter
`export_card()` always emits a frontmatter block delimited by `---` lines that `yaml.safe_load` can parse.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] export-card-type-is-eem
The exported card frontmatter always uses the constant type value `"eem"`.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] export-card-body-sections-and-usage-commands-are-contract
The exported card body must contain the stats table, `## How to Use` with `reasons import-json network.json`, `## Quality`, and `## License` sections.
- Source: summaries/2026/11/19/tests-test_export_card.md

### [ACCEPTED] export-card-empty-network-produces-zero-count-card
`export_card` handles an empty network by producing a valid titled card with zero belief, premise, derived, and nogood counts.
- Source: summaries/2026/11/19/tests-test_export_card.md


---

**Generated:** 2026-11-19 (batch 20/20)
**Model:** openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017

### [REJECT] export-markdown-tests-bind-to-reasons-package
The export-Markdown tests import `Justification`, `Network`, and `export_markdown` from the `reasons` package rather than `reasons_lib`, so they cover the `reasons` modules directly.
- Source: summaries/2026/11/19/tests-test_export_markdown.md
- Rejected: trivial: describes test import paths, which is obvious from the imports/file structure and is test-methodology rather than a durable code invariant

### [ACCEPTED] export-markdown-nogood-id-uses-nogood-ids-joined-by-hyphens
For `add_nogood(["a", "b"])`, Markdown export contains a `nogood-a-b` identifier under a `## Nogoods` section.
- Source: summaries/2026/11/19/tests-test_export_markdown.md

### [ACCEPTED] export-markdown-stale-status-precedes-out
A node with `stale_reason` metadata renders as `[STALE]` rather than `[OUT]` in the tested stale/retracted scenario.
- Source: summaries/2026/11/19/tests-test_export_markdown.md

### [ACCEPTED] hf-resolve-repo-id-rejects-non-huggingface-urls
`resolve_repo_id` accepts only `https://huggingface.co/...` URLs and raises `ValueError("Not a HuggingFace URL")` for other URLs.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] hf-create-repo-idempotent-on-409
`create_repo` treats HTTP 409 Conflict as success and silently returns the fully-qualified repo id.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] hf-upload-file-uses-put-with-bearer-auth-and-raw-bytes
`upload_file` issues a PUT to `/api/models/{repo_id}/upload/main/{filename}` with `Authorization: Bearer <token>` and the raw file bytes as request data.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] hf-publish-hf-uploads-exactly-three-artifacts
`publish_hf` uploads exactly `network.json`, `beliefs.md`, and `README.md`.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] hf-publish-hf-returns-repo-id-url-and-files-uploaded
`publish_hf` returns a dict with `repo_id`, `url` (`https://huggingface.co/{repo_id}`), and `files_uploaded`.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] hf-import-hf-returns-resolved-repo-id
`import_hf` resolves bare repo names through the default org and returns the fully resolved `repo_id`, preserving an explicitly supplied `owner/repo`.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [REJECT] hf-default-org-is-eem-hub
Already covered by the existing `hf-repo-id-normalization-with-env-override` belief.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [REJECT] hf-publish-errors-when-token-missing
Already covered by the existing `hf-write-ops-require-caller-provided-token` belief.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [REJECT] hf-create-repo-401-maps-to-authentication-runtimeerror
Already covered by the existing `hf-401-always-maps-to-authentication-runtimeerror` belief.
- Source: summaries/2026/11/19/tests-test_hf_default_org.md

### [ACCEPTED] compact-out-section-header-is-include-out-gated
`api.compact` emits the literal `OUT (retracted)` section header if and only if `include_out=True`.
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [ACCEPTED] include-out-is-additive-not-exclusive
Passing `include_out=True` adds OUT nodes to `search`, `lookup`, and `compact` output without removing IN nodes.
- Source: summaries/2026/11/19/tests-test_hide_out.md

### [ACCEPTED] import-agent-only-in-skips-non-in-claims
Passing `only_in=True` to `import_agent` skips claims not marked `[IN]`, so a STALE claim is absent and `claims_retracted` is zero.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] import-agent-remaps-nogood-ids-with-namespace
`import_agent` remaps imported nogood members through the same agent namespace pass as beliefs and reports `nogoods_imported`.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] import-agent-preserves-original-id-in-metadata
Namespaced beliefs imported by `import_agent` carry the source ID in `metadata.original_id`.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] stale-imported-claims-can-recompute-to-in
A STALE claim imported by `import_agent` as retracted can propagate back to IN during `recompute_all()` because its imported justification is satisfiable while the agent is active.
- Source: summaries/2026/11/19/tests-test_import_agent.md

### [ACCEPTED] proposal-markdown-emits-suggested-cli-commands
`format_proposal_markdown()` emits suggested `reasons update` / `reasons retract` commands but does not execute them.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

### [ACCEPTED] cascade-impact-not-computed-by-propose
`propose_updates()` never computes cascade impact; proposal Markdown shows “not computed” unless an external caller supplies cascade data.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

### [REJECT] proposal-taxonomies-are-closed-enums
Already covered by the existing `update-proposals-validate-enums-with-safe-defaults` belief.
- Source: summaries/2026/11/19/topic-diary-update-pipeline.md

