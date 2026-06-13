# import

[Back to index](index.md)

The import subsystem is responsible for ingesting external belief networks into a local truth maintenance system. It handles multiple input formats, reconciles truth states across independent sources, and preserves the structural invariants of the TMS through a carefully ordered pipeline. The subsystem supports both one-time bulk loads and ongoing synchronization with remote agents, making it central to ftl-reasons' multi-agent federation capabilities.

## Pipeline Architecture

All API-level import functions — `import_json`, `import_beliefs`, and `import_agent` — follow a common pattern: they read and parse files into structured data in the API layer before delegating to `PgApi`, which never receives raw file paths (`import-functions-parse-before-dispatch`). For agent imports specifically, both the markdown and JSON normalizers produce identical intermediate dictionaries containing id, text, truth state, source metadata, and raw justifications before hitting shared processing logic (`import-agent-normalizers-share-intermediate-schema`). The beliefs.md parser is forward-compatible: unknown metadata lines are silently skipped, allowing the import format to evolve without breaking older consumers (`import-beliefs-parser-is-forward-compatible`).

## Two-Phase Truth Maintenance

Import follows a deliberate three-step ordering discipline: add all nodes first, propagate truth values via `recompute_all()` second, and apply explicit retractions last (`import-two-phase-truth-maintenance`). This sequencing prevents incorrect cascades that would arise from propagating through partially-constructed networks. Deferred retractions correctly override any truth values that propagation would compute, producing a final state that respects both structural dependencies and explicit intent (`import-ordering-ensures-correct-final-state`).

## Topological Sort and Cycle Tolerance

When reconstructing networks, `import_json()` uses a multi-pass topological sort loop that repeatedly adds nodes whose dependencies are already loaded (`import-json-uses-topological-sort`). The sort is best-effort: when progress stalls due to dependency cycles or missing references, remaining nodes are appended in arbitrary order rather than raising an error (`import-topo-sort-tolerates-cycles`, `import-topo-sort-cycle-tolerant`). The cycle-breaking threshold is `max_passes = len(remaining) + 1` iterations (`import-agent-topo-sort-breaks-cycles`). This pragmatic approach ensures import always completes, even when circular references exist in the source data.

## Dual Reconciliation Modes

The subsystem offers two distinct reconciliation strategies (`import-sync-has-dual-reconciliation-modes`). **Import mode** is additive — it performs a one-time load that skips any nodes already present in the local database. **Sync mode** uses remote-wins semantics, overwriting text, justifications, and truth values from the remote source and retracting locally any beliefs that have been removed from the remote (`import-skips-existing-sync-is-remote-wins`). Together these modes support different operational needs: import for initial bootstrap, sync for ongoing federation.

## Handling Heterogeneous Truth States

The import pipeline handles mixed truth states gracefully (`import-handles-heterogeneous-truth-states`). Beliefs that are OUT or STALE in the source are imported with empty justification lists, which prevents `recompute_all` from resurrecting them to IN even when their justification antecedents happen to be IN locally (`import-agent-out-beliefs-get-empty-justifications`, `import-agent-out-beliefs-not-resurrected`). Nodes marked `[STALE]` in beliefs.md are mapped to truth value OUT during both initial import and subsequent syncs (`stale-maps-to-out-on-import`). This invariant — that retractions survive recomputation — was established as a regression guard for issue #16 (`import-agent-retraction-survives-recompute`).

## Agent Namespace Isolation

Every node imported from an agent gets an ID prefix matching the agent's name (e.g., `X:node-id`), including infrastructure nodes like `X:active` and `X:inactive`, ensuring zero collision with local or other-agent beliefs (`import-agent-namespace-prefix`). Every imported belief unconditionally includes `agent:inactive` in its outlist, enforced in `_build_justifications()` with no code path that omits it — this is the mechanism behind the per-agent kill switch (`import-agent-inactive-always-in-outlist`). The `_sync_claims()` function filters out these infrastructure nodes via `infra_ids` before computing the removal set, so the kill-switch pair is never inadvertently retracted by remote-wins sync (`import-agent-infra-nodes-excluded-from-removal`).

## Nogood ID Collision Prevention

Both `import_json` and `import_into_network` advance the `_next_nogood_id` counter to at least `max_imported_id + 1` during import (`import-paths-bump-nogood-counter`, `nogood-id-import-sync`). The high-water-mark update uses `max(current, parsed_id + 1)`, ensuring future auto-generated nogood IDs never collide with imported ones (`import-beliefs-nogood-id-uses-high-water-mark`).

## Reconciliation Completeness

The import subsystem provides complete reconciliation coverage: heterogeneous truth states are handled correctly on initial load, dual modes support both additive import and remote-wins sync, and the colon-based namespace convention with auto-wiring prevents ID collisions across agents (`import-provides-complete-reconciliation`). Two broader convergence claims — that import achieves ordered convergent reconciliation (`import-achieves-ordered-convergent-reconciliation`) and that reconciliation converges deterministically (`import-reconciliation-converges-deterministically`) — are currently OUT, as they depend on upstream beliefs about deterministic convergence that have been retracted.

## Test Infrastructure

The `PgApi` class is imported inside the `pg_api` test fixture body rather than at module top-level, so the test suite loads without Postgres dependencies when running SQLite-only tests (`pg-import-is-deferred`). Several beliefs note unused or vestigial imports in test and production code (`hash-file-import-unused`, `unused-path-import`, `unused-node-import-in-dangling-test`) — these are unstable details likely to be cleaned up and are not architectural invariants.
