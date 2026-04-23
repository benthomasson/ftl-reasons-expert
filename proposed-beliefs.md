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

### [ACCEPT] node-in-if-any-justification-valid
A Node is IN when at least one of its Justifications is valid; a Justification is valid when all antecedents are IN and all outlist nodes are OUT (disjunctive over justifications, conjunctive within each).
- Source: entries/2026/04/23/reasons_lib-__init__.md

### [ACCEPT] premises-have-no-justifications
A premise node is represented by an empty `justifications` list and defaults to `truth_value="IN"`; the system treats the empty-justifications case as a special unconditional belief.
- Source: entries/2026/04/23/reasons_lib-__init__.md

### [ACCEPT] init-is-pure-data-model
`reasons_lib/__init__.py` contains only dataclass definitions (`Node`, `Justification`, `Nogood`) with no behavior, validation, or I/O; it imports nothing from the project and sits at the bottom of the import graph.
- Source: entries/2026/04/23/reasons_lib-__init__.md

### [ACCEPT] outlist-enables-non-monotonic-reasoning
The `outlist` field on `Justification` allows beliefs to be retracted when a defeating node becomes IN — this is the core non-monotonic mechanism powering `supersede`, `challenge`, and default-logic patterns.
- Source: entries/2026/04/23/reasons_lib-__init__.md

### [ACCEPT] dependents-is-manual-reverse-index
`Node.dependents` is a denormalized reverse pointer set that must be kept in sync by external code (primarily `network.py`); nothing in the data model enforces consistency.
- Source: entries/2026/04/23/reasons_lib-__init__.md

---

## From `reasons_lib/api.py`

### [ACCEPT] api-functions-return-dicts
Every public API function returns a `dict` (or `str` for markdown/compact), never a `Network` or `Node` object, ensuring JSON-serializability at the boundary for CLI, HTTP, and tool-call consumers.
- Source: entries/2026/04/23/reasons_lib-api.md

### [ACCEPT] write-false-prevents-persistence
Functions using `_with_network(write=False)` can mutate the in-memory network (as `what_if_retract` does) but changes are never saved to SQLite; write-or-not is declared upfront and never conditional.
- Source: entries/2026/04/23/reasons_lib-api.md

### [ACCEPT] namespace-active-premise-invariant
When `namespace` is set, `add_node` auto-creates a `{namespace}:active` premise node and wires it as an antecedent into every namespaced justification; retracting that single premise cascades OUT every belief from that namespace.
- Source: entries/2026/04/23/reasons_lib-api.md

### [ACCEPT] any-mode-creates-per-premise-justifications
When `any_mode=True` and multiple antecedents are given, each antecedent gets its own SL justification (OR semantics: node is IN if *any* antecedent is IN), rather than the default single multi-antecedent justification (AND semantics).
- Source: entries/2026/04/23/reasons_lib-api.md

### [ACCEPT] transaction-per-function
Every API function opens the database, does its work, and closes — no shared state, no connection pooling, no long-lived sessions; each invocation is fully independent.
- Source: entries/2026/04/23/reasons_lib-api.md

### [ACCEPT] colon-means-already-namespaced
`_resolve_namespace` treats a colon in a node ID as "already namespaced" and never double-prefixes; this is the convention for cross-namespace references.
- Source: entries/2026/04/23/reasons_lib-api.md

---

## From `reasons_lib/check_stale.py`

### [ACCEPT] check-stale-is-read-only
`check_stale` never mutates the network; it returns a list of stale-node dicts and leaves all nodes unchanged — staleness detection is separated from staleness resolution.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

### [ACCEPT] check-stale-skips-out-nodes
Only nodes with `truth_value == "IN"` are checked for staleness; retracted (OUT) nodes are ignored even if their source file has changed.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

### [ACCEPT] check-stale-requires-both-source-fields
A node must have both `source` (non-empty) and `source_hash` (non-empty) to be eligible for staleness checking; nodes missing either field are silently skipped.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

### [ACCEPT] hash-truncation-is-16-hex
Source hashes are SHA-256 truncated to the first 16 hex characters (64 bits), reducing collision resistance to ~32 bits for birthday attacks compared to the full 256-bit hash.
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

### [ACCEPT] missing-source-file-is-silent
If a node's source file no longer exists on disk, `check_stale` silently skips it; callers cannot distinguish "file deleted" from "file never tracked."
- Source: entries/2026/04/23/reasons_lib-check_stale-check_stale.md

---

## From `reasons_lib/cli.py`

### [ACCEPT] cli-is-pure-formatter
Every `cmd_*` function delegates to `api.*` and only formats the returned dict for terminal output; no business logic lives in the CLI layer (sole exception: `cmd_propagate` touches `Storage` directly).
- Source: entries/2026/04/23/reasons_lib-cli.md

### [ACCEPT] check-stale-exits-nonzero
`cmd_check_stale` calls `sys.exit(1)` when any stale nodes are found, making it usable as a CI or pre-commit gate.
- Source: entries/2026/04/23/reasons_lib-cli.md

### [ACCEPT] derive-strips-claudecode-env
`_derive_one_round` explicitly removes the `CLAUDECODE` environment variable before spawning the model subprocess, preventing recursive Claude Code invocation.
- Source: entries/2026/04/23/reasons_lib-cli.md

### [ACCEPT] exhaust-implies-auto
In `_derive_one_round`, proposals are auto-applied when either `args.auto` or `args.exhaust` is true; the `--exhaust` flag does not require the user to also pass `--auto`.
- Source: entries/2026/04/23/reasons_lib-cli.md

---

## From `reasons_lib/compact.py`

### [ACCEPT] compact-budget-only-limits-in-nodes
The token budget only constrains the IN nodes section; nogoods and OUT nodes are always emitted regardless of budget, so compact output can exceed the specified budget value.
- Source: entries/2026/04/23/reasons_lib-compact.md

### [ACCEPT] compact-summary-hiding-requires-in
A summary node only hides its covered nodes when the summary itself is IN; if the summary goes OUT, covered nodes reappear in the compact output.
- Source: entries/2026/04/23/reasons_lib-compact.md

### [ACCEPT] compact-token-estimate-is-word-count
`estimate_tokens` counts whitespace-separated words, not BPE tokens; the budget parameter throughout the compact module is measured in this unit.
- Source: entries/2026/04/23/reasons_lib-compact.md

### [ACCEPT] compact-in-nodes-ordered-by-dependents
IN nodes are sorted by descending dependent count so structurally important nodes (those depended on by many others) are emitted first and survive budget truncation.
- Source: entries/2026/04/23/reasons_lib-compact.md


I'll review the entries and extract architectural/behavioral beliefs from each.

### [ACCEPT] derive-retraction-guard-uses-jaccard
`validate_proposals` rejects any proposed belief ID with Jaccard similarity >= 0.5 to an existing OUT node, preventing re-derivation of retracted beliefs
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPT] derive-prompt-roundtrips-through-parser
The `### DERIVE` / `### GATE` format is a shared contract between `DERIVE_PROMPT` LLM output, `parse_proposals()` input, and `write_proposals_file()` output, forming a closed serialization loop
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPT] derive-depth-cycle-guard
`_get_depth` sets `memo[node_id] = 0` before recursing to prevent infinite recursion on cyclic justification chains; cycles resolve to depth 0
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPT] derive-agent-budget-proportional
When agents are present, `_build_beliefs_section` allocates prompt token budget proportionally to each agent's belief count, with a floor of 5 beliefs per agent
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPT] derive-fail-soft-validation
`validate_proposals` filters invalid proposals into a skipped list rather than raising; `apply_proposals` catches per-item exceptions so one bad proposal never blocks others
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPT] derive-agent-count-bug
`_build_beliefs_section` has a bug: `count += len(belief_ids)` is inside the per-belief loop instead of outside it, inflating the count and shrinking the non-agent budget below intended size
- Source: entries/2026/04/23/reasons_lib-derive.md

### [ACCEPT] import-agent-namespace-prefix
Every node imported from agent X gets the ID prefix `X:`, including infrastructure nodes `X:active` and `X:inactive`, ensuring zero collision with local or other-agent beliefs
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] kill-switch-uses-outlist-not-antecedent
The `agent:inactive` node is placed in each imported belief's outlist (not antecedents) so that retracting `agent:active` cascades all imported beliefs to OUT, while per-belief retraction still works independently
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] out-beliefs-imported-without-justifications
Beliefs that are OUT or STALE in the source are imported with an empty justification list, preventing `recompute_all` from resurrecting them to IN
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] sync-is-remote-wins
`_sync_claims` implements remote-wins reconciliation: remote text/metadata overwrites local, beliefs removed from remote are retracted locally, and beliefs remotely IN but locally OUT are re-asserted
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] import-two-phase-truth-maintenance
Import/sync adds all nodes first, then runs `recompute_all()` to propagate truth values, then performs explicit retractions — this ordering prevents incorrect cascades from partially-constructed networks
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] import-topo-sort-tolerates-cycles
`_topo_sort_claims` attempts topological ordering but appends remaining nodes when progress stalls, gracefully handling dependency cycles instead of erroring
- Source: entries/2026/04/23/reasons_lib-import_agent.md

### [ACCEPT] sl-outlist-asymmetry
Missing antecedents invalidate a justification, but missing outlist nodes do not — this asymmetry enables "believe X unless Y" where Y may not yet exist in the network
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [ACCEPT] cp-equals-sl
CP (conditional-proof) justifications are evaluated with the exact same logic as SL justifications, despite being a distinct type in Doyle's TMS — either an intentional simplification or incomplete implementation
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [ACCEPT] justification-valid-is-pure
`_justification_valid` is a pure query with no side effects, no logging, and no mutations to network state
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [ACCEPT] empty-antecedents-vacuously-valid
An SL justification with an empty antecedent list is valid (vacuous truth via `all([])`), allowing outlist-only justifications to function as "IN unless Y" — used by `challenge` and `supersede` for converted premises
- Source: entries/2026/04/23/reasons_lib-network-_justification_valid.md

### [ACCEPT] propagate-skips-retracted-nodes
`_propagate` never recomputes truth values for nodes with `_retracted` in metadata, even if their justifications would support IN; only `assert_node` can restore them
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPT] propagate-does-not-change-trigger
The seed node (`changed_id`) is added to `visited` immediately and never has its own truth value recomputed; callers must update it before calling `_propagate`
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPT] propagate-cascade-stops-on-unchanged
If a dependent's recomputed truth value equals its current value, it is not enqueued — the cascade terminates along that path, making propagation selective rather than exhaustive
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPT] propagate-assumes-dependents-exist
Every ID in `node.dependents` is accessed via `self.nodes[dep_id]` without a membership check; a dangling dependent reference will raise `KeyError` — this is intentional (broken invariant = bug)
- Source: entries/2026/04/23/reasons_lib-network-_propagate.md

### [ACCEPT] add-nogood-always-records
`add_nogood` appends a `Nogood` record unconditionally before checking whether the contradiction is active, so nogoods are preserved even when not all member nodes are currently IN
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [ACCEPT] add-nogood-retraction-prefers-least-entrenched
The primary retraction path traces back through justification chains to premises and chooses the one with the lowest entrenchment score (speculative assumptions before evidence-backed observations)
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [ACCEPT] add-nogood-fallback-uses-dependent-count
When `find_culprits` returns no candidates (all nogood nodes are premises with no justification chains), the fallback retracts the nogood member with the fewest direct dependents
- Source: entries/2026/04/23/reasons_lib-network-add_nogood.md

### [ACCEPT] nogood-ids-assume-append-only
Nogood IDs are derived from `len(self.nogoods) + 1`, so deleting a nogood from the list would cause ID collisions on subsequent calls
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

### [ACCEPT] three-layer-architecture
The codebase is a three-layer stack: data model (`__init__.py`), TMS engine (`network.py`), and persistence (`storage.py`), with `api.py` providing functional API and `cli.py` as a thin argparse wrapper.
- Source: entries/2026/04/23/scan-ftl-reasons.md

### [ACCEPT] network-is-central-dependency
`network.py` is imported by essentially every other module — api, storage, import, export, compact, check_stale, and all test files — making it the central data structure of the project.
- Source: entries/2026/04/23/scan-ftl-reasons.md

### [ACCEPT] api-uses-with-network-context-manager
`api.py` uses a `_with_network` context manager to ensure load-operate-save atomicity for all network mutations.
- Source: entries/2026/04/23/scan-ftl-reasons.md

---

## From `reasons_lib-network.md`

### [ACCEPT] sl-justification-semantics
An SL justification is valid iff ALL antecedents are IN AND ALL outlist nodes are OUT; a node is IN iff ANY of its justifications is valid (conjunction within a justification, disjunction across justifications).
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] propagation-is-bfs
Truth value propagation in `_propagate` uses `deque`-based BFS through the `dependents` graph, not DFS, ensuring breadth-first wavefront expansion.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] premise-defaults-to-in
A node with no justifications (a premise) defaults to IN; `_compute_truth` preserves its current truth value rather than recomputing it.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] dependents-bidirectional-index
Each node maintains a `dependents` set (reverse of antecedent/outlist edges), eagerly maintained by `add_node`, `add_justification`, `supersede`, `challenge`, and `convert_to_premise`.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] recompute-all-uses-fixpoint
`recompute_all` iterates until no truth values change, bounded by `len(nodes) + 1` iterations, handling cascading dependencies from arbitrary node ordering.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] retracted-nodes-skipped-in-propagation
Retracted nodes (marked with `_retracted` metadata) are skipped during BFS propagation but remain in the network for potential restoration.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] missing-outlist-nodes-pass-validation
In `_justification_valid`, missing antecedent nodes cause the check to fail (node goes OUT), but missing outlist nodes pass (don't block) — an open-world default.
- Source: entries/2026/04/23/reasons_lib-network.md

### [ACCEPT] backtracking-retracts-least-entrenched
`add_nogood` resolves contradictions via dependency-directed backtracking: `find_culprits` traces to premises, scores by `_entrenchment`, and retracts the least-entrenched premise to minimize disruption.
- Source: entries/2026/04/23/reasons_lib-network.md

---

## From `reasons_lib-network-challenge.md`

### [ACCEPT] challenge-uses-outlist-mechanism
`challenge` works by creating an IN premise node and adding it to the target's outlist in every justification, reusing the same non-monotonic mechanism as `supersede`.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md

### [ACCEPT] challenge-converts-premises-to-justified
When a premise (node with no justifications) is challenged, it is converted to a justified node with an SL justification containing empty antecedents and the challenge in the outlist.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md

### [ACCEPT] challenge-modifies-all-justifications
When the target has multiple justifications, the challenge node is added to the outlist of every justification, ensuring no single justification can independently keep the target IN.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md

### [ACCEPT] challenge-id-auto-generation
Auto-generated challenge IDs follow the pattern `challenge-{target}`, then `challenge-{target}-2`, `-3`, etc.; explicit IDs that collide raise `ValueError` rather than auto-deduplicating.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md

### [ACCEPT] defend-is-challenge-of-challenge
`defend` works by calling `challenge` on the challenge node itself, creating a recursive dialectical structure where truth values resolve automatically through the same outlist mechanism.
- Source: entries/2026/04/23/reasons_lib-network-challenge.md

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

### [ACCEPT] dependents-index-derived-on-load
The `node.dependents` set is never persisted to SQLite; it is rebuilt by walking all justification antecedents and outlists during `load()`.
- Source: entries/2026/04/23/reasons_lib-storage.md

### [ACCEPT] storage-trusts-stored-truth-values
`load()` trusts the stored `truth_value` without re-running propagation, making the database the source of truth for node status.
- Source: entries/2026/04/23/reasons_lib-storage.md

---

## From `topic-doyle-1979-tms.md`

### [ACCEPT] cp-and-sl-evaluated-identically
CP and SL justifications use the same validity check in `_justification_valid`; the distinction is semantic (support vs. consistency), not computational.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md

### [REJECT] sl-justification-is-conjunction-of-inlist-disjunction-of-outlist
Redundant with `sl-justification-semantics` proposed above, which captures the same claim more concisely.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md

### [REJECT] retracted-nodes-stay-in-network
Redundant with `retracted-nodes-skipped-in-propagation` which captures the same fact plus the propagation behavior.
- Source: entries/2026/04/23/topic-doyle-1979-tms.md


No existing beliefs. Here are the proposed beliefs from both entries:

---

## From `entries/2026/04/23/topic-multi-agent-federation.md`

### [ACCEPT] namespace-prefix-is-colon-separated
Agent namespacing uses the format `agent_name:belief_id`; `_resolve_namespace()` skips prefixing any ID that already contains a colon, preventing double-prefixing of cross-namespace references
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPT] active-inactive-relay-pair
Each imported agent gets exactly two infrastructure nodes: `agent:active` (premise, starts IN) and `agent:inactive` (derived via SL with `outlist=[active_id]`, starts OUT); every imported belief includes `inactive_id` in its outlist
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPT] active-not-in-antecedents
The `active` premise is deliberately excluded from imported beliefs' antecedents; if it were an antecedent, it would provide a second always-valid justification path that defeats per-belief retraction semantics
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPT] kill-switch-cascade-is-reversible
Retracting `agent:active` cascades all agent beliefs to OUT via the inactive relay flipping IN; re-asserting `agent:active` reverses the cascade, restoring all beliefs to IN via the same BFS propagation
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPT] agent-cascades-are-isolated-by-namespace
Retracting one agent's active premise does not affect other agents' beliefs, because each agent's imported beliefs reference only their own `inactive` node in their outlist
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPT] justification-validity-requires-inlist-in-and-outlist-out
A justification is valid iff all antecedents are IN and all outlist nodes are OUT; this single rule drives retraction cascades, kill-switch behavior, challenges, and supersession
- Source: entries/2026/04/23/topic-multi-agent-federation.md

### [ACCEPT] import-skips-existing-sync-is-remote-wins
Import mode (`import_agent`) is a one-time load that skips existing nodes; sync mode updates text/justifications/truth values with remote-wins semantics and retracts locally any beliefs removed from the remote
- Source: entries/2026/04/23/topic-multi-agent-federation.md

---

## From `entries/2026/04/23/topic-outlist-semantics.md`

### [ACCEPT] outlist-absent-means-out
An outlist node that doesn't exist in the network is treated as OUT (justification satisfied); absent antecedent nodes fail validation — this asymmetry makes missing counter-evidence permissive while missing supporting evidence is strict
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] node-in-if-any-justification-valid
A node's truth value is IN if any single justification is valid (disjunctive semantics); a node can survive an outlist violation if it has a backup justification through a different path
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] challenge-is-outlist-injection
The challenge mechanism creates a new premise node and adds it to the target's outlist; all truth-value changes flow through normal BFS propagation, not direct mutation
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] defend-is-recursive-challenge
Defense is implemented by calling `challenge()` on the challenge node itself, enabling arbitrarily deep dialectical chains using the same outlist mechanism recursively with no special-case code
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] supersession-is-reversible
`supersede()` adds the new node's ID to the old node's outlist rather than deleting the old node; retracting the new belief automatically restores the old one through normal propagation
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] multiple-outlist-is-conjunction
When a justification has multiple outlist entries, ALL must be OUT for the justification to be valid; any single outlist node going IN defeats the entire justification
- Source: entries/2026/04/23/topic-outlist-semantics.md

### [ACCEPT] outlist-relationships-survive-persistence
Outlists are stored as `outlist_json` in the SQLite `justifications` table; on load, the dependent index is rebuilt for both antecedents and outlist nodes, preserving propagation behavior across save/load cycles
- Source: entries/2026/04/23/topic-outlist-semantics.md


