# import

[Back to index](index.md)

The import subsystem is responsible for ingesting external belief networks into a local TMS database — whether from serialized files (JSON, markdown) or from remote agents in a federated deployment. It provides complete reconciliation coverage, handling heterogeneous truth states, dependency cycles, and namespace isolation to ensure that all external inputs are safely integrated (import-provides-complete-reconciliation).

## Architecture

Import functions follow a parse-before-dispatch pattern: the API layer (`import_json`, `import_beliefs`, `import_agent`) reads and parses files into structured data before delegating to `PgApi`, which never receives raw file paths (import-functions-parse-before-dispatch). For agent imports, both `_normalize_markdown()` and `_normalize_json()` produce identical intermediate dictionaries — containing id, text, truth state, source metadata, and raw justifications — before feeding into shared `_import_claims`/`_sync_claims` logic (import-agent-normalizers-share-intermediate-schema). This normalization layer means the downstream reconciliation code is format-agnostic.

The parser is deliberately forward-compatible: unknown metadata lines in belief markdown are silently skipped rather than rejected, so older importers can consume exports from newer versions without error (import-beliefs-parser-is-forward-compatible).

## Dual Reconciliation Modes

The import subsystem offers two distinct reconciliation strategies to support different operational needs (import-sync-has-dual-reconciliation-modes).

**Import mode** is additive: when `import_agent` runs as a one-time load, it skips any nodes that already exist in the local database (import-skips-existing-sync-is-remote-wins). This makes import safe to re-run — it will not overwrite local modifications to previously imported beliefs.

**Sync mode** uses remote-wins semantics: it overwrites text, justifications, and truth values from the remote source, and retracts locally any beliefs that have been removed from the remote (import-skips-existing-sync-is-remote-wins). During sync, infrastructure nodes (the `active`/`inactive` pair) are filtered out via `infra_ids` before computing the removal set, so the per-agent kill switch is never accidentally retracted by remote-wins logic (import-agent-infra-nodes-excluded-from-removal).

## Ordering and Truth Maintenance

Import follows a deliberate three-phase ordering discipline to ensure correctness (import-ordering-ensures-correct-final-state):

1. **Add all nodes** — the full network structure is constructed first.
2. **Propagate truth values** — `recompute_all()` runs over the complete graph.
3. **Apply explicit retractions** — deferred retractions override any truth values that propagation would have computed.

This two-phase truth maintenance approach prevents incorrect cascades that would arise from propagating truth values over a partially-constructed network (import-two-phase-truth-maintenance). The ordering ensures that deferred retractions correctly override propagated states, producing a final result that respects both structural dependencies and explicit intent (import-ordering-ensures-correct-final-state).

A critical invariant: after retracting an imported belief, `recompute_all()` must not resurrect it. The justification structure must not provide alternative paths back to IN — this was established as a regression invariant after issue #16 (import-agent-retraction-survives-recompute).

## Topological Sort and Cycle Tolerance

When reconstructing a belief network from serialized form, `import_json()` uses a multi-pass topological sort loop: it repeatedly adds nodes whose dependencies are already loaded, with a force-add fallback for cycles or missing dependencies (import-json-uses-topological-sort). The `_topo_sort_claims` function takes a pragmatic approach to cycles — after `max_passes = len(remaining) + 1` iterations without progress, all remaining unsorted nodes are appended in arbitrary order rather than raising an error (import-agent-topo-sort-breaks-cycles, import-topo-sort-tolerates-cycles). This ensures import always completes, even when circular references exist in the source data (import-topo-sort-cycle-tolerant).

## Handling OUT and STALE Beliefs

The import pipeline correctly handles mixed truth states arriving from external sources (import-handles-heterogeneous-truth-states). Beliefs that are OUT or STALE in the source are imported with empty justification lists, which prevents `recompute_all` from resurrecting them to IN even when their justification antecedents happen to be all-IN in the local database (import-agent-out-beliefs-get-empty-justifications, import-agent-out-beliefs-not-resurrected). Nodes marked `[STALE]` in beliefs.md are mapped to truth value OUT during both initial import and subsequent syncs (stale-maps-to-out-on-import).

## Agent Namespace Isolation

Every node imported from agent X receives the ID prefix `X:`, including the infrastructure nodes `X:active` and `X:inactive` (import-agent-namespace-prefix). This colon-based namespace convention ensures zero collision with local beliefs or beliefs from other agents.

Every imported belief unconditionally includes `agent:inactive` in its outlist, enforced in `_build_justifications()` with no code path that omits it (import-agent-inactive-always-in-outlist). This is the mechanism behind the per-agent kill switch: asserting `X:inactive` immediately retracts all of agent X's beliefs via outlist propagation.

## Nogood ID Management

Both `import_json` and `import_into_network` advance the `_next_nogood_id` counter to at least `max_imported_id + 1` during import (import-paths-bump-nogood-counter, nogood-id-import-sync). The high-water-mark strategy — `max(current, parsed_id + 1)` — prevents future auto-generated nogood IDs from colliding with imported ones (import-beliefs-nogood-id-uses-high-water-mark).

## Test Infrastructure

The `PgApi` class is imported inside the `pg_api` test fixture body rather than at module top-level, so the test suite loads without Postgres dependencies when running SQLite-only tests (pg-import-is-deferred). Several minor observations about unused imports in test and utility files have been noted but are considered unstable trivia rather than architectural invariants (hash-file-import-unused, unused-path-import, unused-node-import-in-dangling-test).
