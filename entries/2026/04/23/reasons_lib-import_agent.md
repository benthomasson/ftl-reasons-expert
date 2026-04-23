# File: reasons_lib/import_agent.py

**Date:** 2026-04-23
**Time:** 16:40

# `reasons_lib/import_agent.py` — Agent Belief Import & Sync

## Purpose

This file owns the **multi-agent belief federation** capability of the reasons system. It lets you import another agent's belief network into your own, namespaced so beliefs from different agents don't collide, and wired with a kill switch so you can retract everything from a specific agent in one operation.

It provides two modes: **import** (additive, skip duplicates) and **sync** (remote-wins reconciliation). Both work with two input formats: markdown (`beliefs.md`) and JSON (`network.json`).

## Key Components

### Public API (4 functions, all thin wrappers)

| Function | Mode | Format | Delegates to |
|---|---|---|---|
| `import_agent()` | import | markdown | `_import_claims()` |
| `import_agent_json()` | import | JSON | `_import_claims()` |
| `sync_agent()` | sync | markdown | `_sync_claims()` |
| `sync_agent_json()` | sync | JSON | `_sync_claims()` |

All four follow the same contract: normalize input to a common claim format, then delegate to shared logic. They return a stats dict summarizing what happened (counts of added, skipped, retracted, etc.).

### The Kill Switch Pattern (`_ensure_agent_nodes`)

For each imported agent, two infrastructure nodes are created:

- **`agent:active`** — A bare premise (no justifications). IN by default. Represents trust in that agent.
- **`agent:inactive`** — Justified by `SL(antecedents=[], outlist=[active])`. This is IN only when `active` is OUT (it's the logical negation via outlist).

Every imported belief gets `agent:inactive` in its **outlist** (not antecedents). This means: if `inactive` goes IN (because you retracted `active`), every imported belief cascades to OUT. The docstring is explicit about why `active` isn't placed in antecedents — that would provide a fallback justification path, defeating per-belief retraction.

### Normalization Layer (`_normalize_markdown`, `_normalize_json`)

Both formats are converted to a common dict shape:

```python
{"id", "text", "is_out", "source", "source_hash", "date", "metadata", "raw_justifications"}
```

where `raw_justifications` contains unprefixed IDs. Filtering (removing references to nodes not in the import set) happens here, so the shared logic only needs to prefix everything.

Key difference: markdown format only preserves `depends_on` and `unless` (lossy), while JSON preserves full justification structure including outlist (lossless).

OUT/STALE beliefs get `raw_justifications = []` — they're imported as bare premises with no justification, preventing `recompute_all` from accidentally resurrecting them.

### Import Logic (`_import_claims`)

Additive import — skips nodes that already exist:

1. Ensures agent infrastructure nodes exist
2. Topologically sorts claims so antecedents are added before dependents
3. Adds each new node with prefixed IDs and `inactive` in every outlist
4. Fixes up dependent registrations (outlist nodes may not have existed at add time)
5. Runs `recompute_all()` to propagate truth values
6. Explicitly retracts nodes that were OUT in the source

### Sync Logic (`_sync_claims`)

Remote-wins reconciliation — more complex:

1. Same setup as import
2. Computes `removed_ids = local_agent_ids - remote_ids` to find beliefs the remote no longer holds
3. For existing nodes: updates text, source, metadata, justifications; tracks whether anything actually changed (for accurate `beliefs_updated` count)
4. For new nodes: adds them (same as import)
5. Retracts removed beliefs, marks them `_retracted`
6. Re-asserts nodes that were locally OUT but remotely IN
7. Runs `recompute_all()`, then handles retraction of remotely-OUT beliefs

The ordering is deliberate: assertions happen before `recompute_all`, retractions happen after. This ensures the truth maintenance system sees the intended state.

## Patterns

**Normalize-then-process**: The format-specific code (markdown vs JSON) is isolated in `_normalize_*` functions. The core logic (`_import_claims`, `_sync_claims`) is format-agnostic. This is the Strategy pattern applied to input parsing.

**Deferred fixup**: `_fixup_dependents` exists because node addition order may not match dependency order. Rather than requiring sorted input, the system does a bulk fixup pass after all nodes are added.

**Topological sort with cycle tolerance**: `_topo_sort_claims` attempts a topological ordering but appends remaining nodes if progress stalls (handles cycles gracefully rather than erroring).

**Two-phase truth maintenance**: Truth values aren't set during node creation. Instead, nodes are added, then `recompute_all()` propagates, then explicit retractions run. This avoids intermediate states where a retraction would cascade incorrectly because dependent nodes haven't been added yet.

## Dependencies

**Imports from:**
- `Justification`, `Nogood` from `reasons_lib.__init__` — data classes for justification records and contradictions
- `parse_beliefs`, `parse_nogoods` from `reasons_lib.import_beliefs` — markdown parsing
- `Network` from `reasons_lib.network` — the truth maintenance network (provides `add_node`, `retract`, `assert_node`, `recompute_all`)

**Imported by:**
- `reasons_lib/api.py` — the CLI/API layer that wires user commands to these functions

## Flow

A typical `sync_agent` call:

```
sync_agent(network, "aap-expert", beliefs_text, nogoods_text)
  → _normalize_markdown(beliefs_text)        # parse + filter + normalize
  → _normalize_nogoods_markdown(nogoods_text) # parse nogoods
  → _sync_claims(network, ...)
      → _ensure_agent_nodes(...)              # create active/inactive if needed
      → compute removed_ids                   # local - remote
      → _topo_sort_claims(claims)             # order by dependencies
      → for each claim:
          if exists: diff & update fields, queue retractions/assertions
          if new: add_node with prefixed IDs
      → retract removed beliefs
      → _fixup_dependents(network)            # repair dependent registrations
      → assert queued nodes                   # re-assert previously-OUT nodes
      → network.recompute_all()               # propagate truth values
      → retract remotely-OUT nodes            # final retraction pass
      → _import_nogoods(...)                  # import contradictions
      → return stats dict
```

## Invariants

1. **Namespace isolation**: Every imported node ID is prefixed with `agent_name:`. Infrastructure nodes (`active`, `inactive`) use the same prefix. No unprefixed IDs are ever written to the network.

2. **Kill switch wiring**: Every imported belief has `agent:inactive` in its outlist. This is enforced in `_build_justifications` — even beliefs with no other justification get a synthetic `SL` justification containing the inactive node in the outlist.

3. **OUT beliefs have no justifications**: When a source belief is OUT/STALE, it's imported with `justifications=[]`. This prevents `recompute_all` from flipping it back to IN.

4. **Idempotent import**: `_import_claims` skips nodes that already exist (by prefixed ID). Running import twice produces the same network state.

5. **Topological ordering**: Claims are sorted so antecedents are added before the nodes that depend on them. Cycles are tolerated — remaining nodes are appended at the end.

6. **Nogood validity**: Nogoods are only imported if at least 2 of their referenced nodes exist in the network after prefixing.

## Error Handling

This module is notably **thin on error handling**. There are no try/except blocks, no input validation beyond what the normalization functions do implicitly, and no explicit error returns. It relies on:

- `parse_beliefs` and `parse_nogoods` handling malformed input
- `Network.add_node` handling duplicate or invalid node additions
- Dict `.get()` calls with defaults for missing JSON fields
- Set operations silently handling missing keys (e.g., `discard` instead of `remove`)

If a referenced antecedent doesn't exist in the import set, it's silently filtered out during normalization. If a nogood references nodes that don't exist post-import, it's silently skipped. This is a deliberate design choice — partial imports are preferred over failures.

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network` class that `add_node`, `retract`, `assert_node`, and `recompute_all` operate on — understanding its truth maintenance algorithm is essential to understanding why the ordering in this file matters
- [file] `reasons_lib/import_beliefs.py` — The `parse_beliefs` and `parse_nogoods` parsers that convert markdown to the intermediate dict format consumed by `_normalize_markdown`
- [file] `tests/test_sync_agent.py` — Sync has complex edge cases (remote retraction, local re-assertion, `_retracted` flag lifecycle) — the tests document the expected behavior
- [function] `reasons_lib/network.py:recompute_all` — The propagation engine invoked after bulk node changes — understanding what it does vs. what explicit retract/assert calls do clarifies the two-phase design
- [general] `outlist-vs-antecedent-semantics` — The distinction between placing a node in antecedents (positive support) vs. outlist (defeater) is central to why the kill switch works correctly

## Beliefs

- `import-agent-namespace-prefix` — Every node imported from agent X gets the ID prefix `X:`, including infrastructure nodes `X:active` and `X:inactive`, ensuring zero collision with local or other-agent beliefs
- `kill-switch-uses-outlist-not-antecedent` — The `agent:inactive` node is placed in each imported belief's outlist (not antecedents) so that per-belief retraction works independently of the agent-level kill switch
- `out-beliefs-imported-without-justifications` — Beliefs that are OUT or STALE in the source are imported with an empty justification list, preventing `recompute_all` from resurrecting them
- `sync-is-remote-wins` — `_sync_claims` treats the remote source as authoritative: remote additions are added, remote removals are retracted locally, and changed fields are overwritten from remote
- `import-is-additive-skip-duplicates` — `_import_claims` never modifies existing nodes; if a prefixed node ID already exists in the network, it increments `skipped` and moves on

