# import

[Back to index](index.md)

### hash-file-import-unused
**Status:** IN

Trivial/unstable detail about an unused import; not a meaningful architectural invariant.


### import-achieves-ordered-convergent-reconciliation
**Status:** OUT

Import achieves correct final state through two reinforcing mechanisms: deliberate ordering discipline (add → propagate → retract) ensures deferred retractions don't corrupt intermediate states, while deterministic convergence (dual reconciliation modes reaching stable states, propagation terminating via fixpoint) ensures the result is unique and reproducible regardless of input ordering within each phase.

**Depends on:** [import-ordering-ensures-correct-final-state](import.md#import-ordering-ensures-correct-final-state), [import-reconciliation-converges-deterministically](import.md#import-reconciliation-converges-deterministically)
**Supports:** [system-reaches-equilibrium-from-all-modification-paths](system.md#system-reaches-equilibrium-from-all-modification-paths)

### import-agent-inactive-always-in-outlist
**Status:** IN

Every imported belief unconditionally has `agent:inactive` in its outlist, enforced in `_build_justifications()` with no code path that omits it — this is the mechanism behind the per-agent kill switch.


### import-agent-infra-nodes-excluded-from-removal
**Status:** IN

`_sync_claims()` filters out infrastructure nodes (active/inactive) via `infra_ids` before computing the removal set, so the kill-switch pair is never retracted by remote-wins sync.


### import-agent-namespace-prefix
**Status:** IN

Every node imported from agent X gets the ID prefix `X:`, including infrastructure nodes `X:active` and `X:inactive`, ensuring zero collision with local or other-agent beliefs

**Supports:** [agent-isolation-through-namespace-and-relay](agent.md#agent-isolation-through-namespace-and-relay)

### import-agent-normalizers-share-intermediate-schema
**Status:** IN

Both `_normalize_markdown()` and `_normalize_json()` produce identical intermediate dicts (id, text, is_out, source, source_hash, date, metadata, raw_justifications) before hitting shared `_import_claims`/`_sync_claims` logic.


### import-agent-out-beliefs-get-empty-justifications
**Status:** IN

Beliefs that are OUT or STALE in the source are imported with empty justification lists, preventing `recompute_all` from resurrecting them to IN.


### import-agent-out-beliefs-not-resurrected
**Status:** IN

Beliefs marked OUT or STALE in the source are imported as OUT and are never resurrected by `recompute_all`, even when their justification antecedents are all IN in the local database.


### import-agent-outlist-not-antecedent
**Status:** IN

Duplicates existing belief `kill-switch-uses-outlist-not-antecedent`.


### import-agent-retraction-survives-recompute
**Status:** IN

After retracting an imported belief, `recompute_all()` must not resurrect it — the justification structure must not provide alternative paths to IN (issue #16 regression invariant).


### import-agent-topo-sort-breaks-cycles
**Status:** IN

`_topo_sort_claims` breaks dependency cycles by appending all remaining unsorted nodes after `max_passes = len(remaining) + 1` iterations rather than raising an error, ensuring import always completes.


### import-beliefs-nogood-id-uses-high-water-mark
**Status:** IN

`_next_nogood_id` is set to `max(current, parsed_id + 1)` during import, preventing future auto-generated nogood IDs from colliding with imported ones.


### import-beliefs-parser-is-forward-compatible
**Status:** IN

Unknown `- ` metadata lines in belief markdown are silently skipped via a `pass` branch, making the parser forward-compatible with new export format fields.

**Supports:** [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)

### import-beliefs-path-import-unused
**Status:** IN

Vestigial import detail — too trivial and unstable to track as a belief; it could be cleaned up at any time.


### import-functions-parse-before-dispatch
**Status:** IN

API import functions (`import_json`, `import_beliefs`, `import_agent`) read and parse files into structured data in the API layer before delegating to `PgApi` — PgApi never receives raw file paths


### import-handles-heterogeneous-truth-states
**Status:** IN

The import pipeline handles mixed truth states: OUT/STALE beliefs arrive without justifications, topological sort tolerates cycles, and two-phase truth maintenance reconciles everything post-import

**Depends on:** [import-topo-sort-tolerates-cycles](import.md#import-topo-sort-tolerates-cycles), [import-two-phase-truth-maintenance](import.md#import-two-phase-truth-maintenance), [out-beliefs-imported-without-justifications](beliefs.md#out-beliefs-imported-without-justifications)
**Supports:** [agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained), [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation)

### import-is-idempotent
**Status:** IN

Covered by existing `import-skips-existing-sync-is-remote-wins`


### import-json-uses-topological-sort
**Status:** IN

`api.import_json()` reconstructs networks via a multi-pass topological sort loop — repeatedly adding nodes whose dependencies are already loaded — with a force-add fallback for cycles or missing deps.


### import-ordering-ensures-correct-final-state
**Status:** IN

Import follows a deliberate ordering discipline — add nodes first, propagate truth values second, apply explicit retractions last — ensuring deferred retractions correctly override any truth values that propagation would compute, producing a final state that respects both structural dependencies and explicit intent.

**Depends on:** [deferred-retraction-ordering](other.md#deferred-retraction-ordering), [import-two-phase-truth-maintenance](import.md#import-two-phase-truth-maintenance)
**Supports:** [import-achieves-ordered-convergent-reconciliation](import.md#import-achieves-ordered-convergent-reconciliation)

### import-paths-bump-nogood-counter
**Status:** IN

Both `import_json` and `import_into_network` update `_next_nogood_id` to at least `max_imported_id + 1`, ensuring imported nogoods with explicit IDs don't create future collisions


### import-provides-complete-reconciliation
**Status:** IN

The import subsystem provides complete reconciliation coverage: heterogeneous truth states are handled correctly on initial load, dual modes support additive import and remote-wins sync for different operational needs, and the colon-based namespace convention with auto-wiring prevents ID collisions across agents.

**Depends on:** [import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states), [import-sync-has-dual-reconciliation-modes](import.md#import-sync-has-dual-reconciliation-modes), [namespace-is-colon-convention-with-auto-wiring](other.md#namespace-is-colon-convention-with-auto-wiring)
**Supports:** [all-external-inputs-safely-integrated](external.md#all-external-inputs-safely-integrated), [bulk-operations-preserve-topology-and-reconcile](topology.md#bulk-operations-preserve-topology-and-reconcile), [external-belief-lifecycle-is-complete](external.md#external-belief-lifecycle-is-complete), [import-reconciliation-converges-deterministically](import.md#import-reconciliation-converges-deterministically), [initialization-and-reconciliation-converge-equivalently](other.md#initialization-and-reconciliation-converge-equivalently)

### import-reconciliation-converges-deterministically
**Status:** OUT

Import reconciliation achieves both completeness and convergence: dual import/sync modes handle heterogeneous truth states with namespace isolation and topological cycle tolerance (completeness), while all reconciliation operations — individual propagation, agent sync, and global recompute — converge to deterministic fixed points on repeated application (convergence), ensuring that import never introduces oscillation or nondeterminism.

**Depends on:** [all-reconciliation-converges-deterministically](other.md#all-reconciliation-converges-deterministically), [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation)
**Supports:** [import-achieves-ordered-convergent-reconciliation](import.md#import-achieves-ordered-convergent-reconciliation), [system-converges-from-addition-and-removal](system.md#system-converges-from-addition-and-removal)

### import-skips-existing-nodes
**Status:** IN

Duplicates existing belief `import-skips-existing-sync-is-remote-wins`.


### import-skips-existing-sync-is-remote-wins
**Status:** IN

Import mode (`import_agent`) is a one-time load that skips existing nodes; sync mode updates text/justifications/truth values with remote-wins semantics and retracts locally any beliefs removed from the remote

**Supports:** [import-sync-has-dual-reconciliation-modes](import.md#import-sync-has-dual-reconciliation-modes)

### import-sync-has-dual-reconciliation-modes
**Status:** IN

The import/sync subsystem offers two distinct reconciliation strategies: import is additive (skips existing nodes), while sync is remote-wins (overwrites text, justifications, and truth values from the remote source).

**Depends on:** [import-skips-existing-sync-is-remote-wins](import.md#import-skips-existing-sync-is-remote-wins), [sync-is-remote-wins](other.md#sync-is-remote-wins)
**Supports:** [external-belief-ingestion-is-defensively-layered](external.md#external-belief-ingestion-is-defensively-layered), [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation)

### import-topo-sort-cycle-tolerant
**Status:** IN

Topological sorting of imported claims is best-effort: cycles cause remaining nodes to be appended in arbitrary order rather than raising an error, a pragmatic choice that avoids crashing on circular references.


### import-topo-sort-tolerates-cycles
**Status:** IN

`_topo_sort_claims` attempts topological ordering but appends remaining nodes when progress stalls, gracefully handling dependency cycles instead of erroring

**Supports:** [import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states)

### import-two-phase-truth-maintenance
**Status:** IN

Import/sync adds all nodes first, then runs `recompute_all()` to propagate truth values, then performs explicit retractions — this ordering prevents incorrect cascades from partially-constructed networks

**Supports:** [bootstrap-bypasses-incremental-propagation](other.md#bootstrap-bypasses-incremental-propagation), [import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states), [import-ordering-ensures-correct-final-state](import.md#import-ordering-ensures-correct-final-state)

### import-wires-reverse-index
**Status:** IN

Covered by existing `dependents-is-manual-reverse-index` which captures that the reverse index is manually maintained


### nogood-id-import-sync
**Status:** IN

Both `import_json` and `import_into_network` must advance `_next_nogood_id` past the highest imported nogood ID to prevent collisions with pre-existing IDs


### pg-import-is-deferred
**Status:** IN

`PgApi` is imported inside the `pg_api` fixture body, not at module top-level, so the test suite loads without Postgres dependencies when running SQLite-only tests.


### stale-maps-to-out-on-import
**Status:** IN

Nodes marked `[STALE]` in beliefs.md are stored with truth value OUT during both initial import and subsequent syncs

**Supports:** [staleness-information-survives-binary-truth-model](other.md#staleness-information-survives-binary-truth-model)

### unused-node-import-in-dangling-test
**Status:** IN

Unstable trivia — an unused import is likely to be cleaned up and is not an architectural invariant.


### unused-path-import
**Status:** IN

Dead code observation (`Path` imported but unused) — unstable detail that will change when cleaned up.

