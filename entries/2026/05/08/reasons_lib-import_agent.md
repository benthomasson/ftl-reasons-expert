# File: reasons_lib/import_agent.py

**Date:** 2026-05-08
**Time:** 14:10

# `reasons_lib/import_agent.py` — Multi-Agent Belief Federation

## Purpose

This file implements the multi-agent federation layer of the TMS. It allows one agent's belief network to ingest beliefs from another agent's exported knowledge base (markdown or JSON), creating a namespaced overlay that coexists with local beliefs. It owns two operations: **import** (additive, skip-if-exists) and **sync** (remote-wins, full reconciliation including removals and resurrections).

The core problem it solves: how do you merge two independently maintained truth maintenance networks without ID collisions, while preserving the ability to retract an entire agent's contributions atomically?

## Key Components

### Public API (4 functions)

| Function | Format | Operation | Key behavior |
|---|---|---|---|
| `import_agent()` | Markdown | Import | Additive; skips existing nodes |
| `import_agent_json()` | JSON | Import | Same, but preserves full justification structure |
| `sync_agent()` | Markdown | Sync | Remote-wins; adds, updates, removes, resurrects |
| `sync_agent_json()` | JSON | Sync | Same with lossless justification preservation |

All four follow the same pattern: normalize input → delegate to `_import_claims` or `_sync_claims` → return a stats dict.

### Infrastructure Nodes: `_ensure_agent_nodes()`

For each agent, two control nodes are created:

- **`agent:active`** — A premise node (no justifications). While IN, the agent's beliefs are trusted.
- **`agent:inactive`** — An SL node with `outlist=[agent:active]`. This is IN precisely when `active` is OUT, functioning as a kill switch.

Every imported belief gets `agent:inactive` in its outlist. Retracting `agent:active` makes `inactive` go IN, which cascades every imported belief to OUT.

### Normalization Layer

`_normalize_markdown()` and `_normalize_json()` convert both input formats into a common claim structure:

```python
{"id", "text", "is_out", "source", "source_hash", "date", "metadata", "raw_justifications"}
```

`raw_justifications` contains unprefixed IDs. Filtering of antecedents/outlist references to only include IDs present in the import set happens here, before prefixing.

### Core Operations

**`_import_claims()`** — Topologically sorts claims, creates nodes with namespaced IDs (`agent:belief-id`), skips existing nodes, and handles OUT beliefs by either retracting (no justifications) or setting `_retracted` metadata (with justifications).

**`_sync_claims()`** — The more complex operation. For each remote claim:
- **New**: creates the node (same as import)
- **Existing, changed**: updates text, source, metadata, justifications; manages `_retracted` flag transitions
- **Existing, unchanged**: counts as `beliefs_unchanged`
- **Local-only (not in remote)**: retracted or marked `_retracted`

Resurrection: when a previously OUT node flips to IN in the remote, sync clears `_retracted` and calls `network.assert_node()` to bring it back.

## Patterns

**Normalize-then-process**: Both formats are normalized into a common dict structure before the shared import/sync logic runs. This keeps format-specific parsing isolated from TMS operations.

**Namespace-as-convention**: Agent beliefs are prefixed with `agent_name:` by string concatenation, not a first-class namespace type. The prefix is applied during `_build_justifications()` and node creation.

**Deferred retraction**: Nodes that should be OUT are first created as IN (so they participate in dependency registration), then retracted after `recompute_all()`. The `retract_after` list accumulates these.

**Deferred assertion**: In sync, nodes transitioning from OUT→IN are collected in `assert_after` and asserted after `_fixup_dependents()` so the dependency graph is complete before propagation.

**`_retracted` metadata flag**: A soft-lock mechanism. Nodes with `_retracted=True` stay OUT even through `recompute_all()`. This prevents the local TMS from resurrecting beliefs that the remote agent has explicitly retracted. Only the sync path can clear this flag.

## Dependencies

**Imports from:**
- `Justification`, `Nogood` from `reasons_lib.__init__` — data structures for TMS justifications and nogood records
- `parse_beliefs`, `parse_nogoods` from `.import_beliefs` — markdown parsing (the heavy lifting of understanding belief markdown format)
- `Network` from `.network` — the TMS network with `add_node`, `retract`, `assert_node`, `recompute_all`, `_rebuild_dependents`

**Imported by:**
- `reasons_lib/api.py` — the public API layer that exposes these operations to the CLI

## Flow

A typical `sync_agent_json()` call proceeds:

1. **Normalize**: `_normalize_json()` converts JSON nodes to claim dicts, filtering outlist references to only include known node IDs
2. **Ensure infrastructure**: `_ensure_agent_nodes()` creates `agent:active` and `agent:inactive` if absent
3. **Topological sort**: `_topo_sort_claims()` orders claims so antecedents are created before dependents (with cycle-breaking fallback — dumps remaining if no progress after a full pass)
4. **Per-claim processing**: For each claim, build prefixed justifications via `_build_justifications()`, then either create or update the node
5. **Remove stale**: Any local agent nodes not in the remote set are retracted
6. **Fix dependents**: `_fixup_dependents()` calls `_rebuild_dependents()` to re-register all outlist dependencies
7. **Assert resurrections**: Nodes transitioning OUT→IN get `assert_node()`
8. **Propagate**: `recompute_all()` propagates truth values through the full network
9. **Deferred retractions**: Nodes in `retract_after` are retracted
10. **Import nogoods**: `_import_nogoods()` adds prefixed nogood records

## Invariants

- **Namespace isolation**: Every imported node ID is prefixed with `agent_name:`. No imported node can collide with a local node or another agent's nodes.
- **Kill switch wiring**: Every imported belief has `agent:inactive` in its outlist. There are no exceptions — even bare premise imports (empty `raw_justifications`) get a fallback justification with `outlist=[inactive_id]`.
- **`inactive` is NOT in antecedents**: The docstring explicitly calls this out. Putting `inactive` in antecedents would create a fallback justification that defeats per-belief retraction.
- **Topological ordering**: Claims are sorted so all antecedents exist before nodes that depend on them. The cycle-breaker appends remaining nodes if no progress is made in a full pass.
- **Remote-wins in sync**: Sync always trusts the remote state. Local-only nodes are retracted. Remote truth value overrides local truth value.
- **Outlist filtering**: Only outlist/antecedent references that exist in the import set are preserved. Dangling references to nodes outside the import are dropped during normalization.

## Error Handling

This module does almost no explicit error handling. It relies on:
- `Network.add_node()` to validate node creation
- `Network.retract()` / `Network.assert_node()` for TMS operations
- The normalization layer silently drops references to unknown node IDs (both antecedents in markdown and outlist entries in JSON)
- `_topo_sort_claims()` has a bounded loop (`max_passes = len(remaining) + 1`) that breaks cycles by appending unsorted nodes rather than raising

If a source file is malformed, the failure will surface in `parse_beliefs()` or JSON deserialization upstream — this module assumes its inputs are already parsed.

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network` class this module operates on: `add_node`, `retract`, `assert_node`, `recompute_all`, and `_rebuild_dependents` are all critical to understanding propagation behavior
- [function] `reasons_lib/import_beliefs.py:parse_beliefs` — The markdown parser that extracts belief ID, text, status, depends_on, and unless fields from the markdown format
- [file] `tests/test_import_agent.py` — Test cases for import and sync, particularly around OUT→IN resurrection and the `_retracted` flag lifecycle
- [general] `outlist-propagation-gap` — The known issue where outlist nodes aren't tracked in the dependents index, causing GATE beliefs to not auto-evaluate when outlist nodes change truth value — directly relevant to the `_fixup_dependents` call here
- [function] `reasons_lib/network.py:recompute_all` — Understanding how `_retracted` metadata interacts with recomputation is essential to understanding why sync marks OUT nodes with `_retracted` rather than just retracting them

## Beliefs

- `import-agent-inactive-always-in-outlist` — Every imported belief node has `agent:inactive` in its outlist, ensuring atomic agent-wide retraction via the kill switch
- `import-agent-inactive-never-in-antecedents` — `agent:inactive` is never placed in antecedents; doing so would create a fallback justification that defeats per-belief retraction
- `sync-agent-remote-wins` — `sync_agent` and `sync_agent_json` implement remote-wins semantics: remote truth value overrides local, and local-only nodes are retracted
- `retracted-metadata-blocks-recompute` — Nodes with `_retracted=True` in metadata stay OUT through `recompute_all()` and can only be resurrected when the remote explicitly sends IN during sync
- `import-agent-topo-sort-breaks-cycles` — `_topo_sort_claims` breaks dependency cycles by appending unsorted nodes after `max_passes` iterations rather than raising an error

