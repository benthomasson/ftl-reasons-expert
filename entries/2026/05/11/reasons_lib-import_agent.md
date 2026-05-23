# File: reasons_lib/import_agent.py

**Date:** 2026-05-11
**Time:** 12:53

## Purpose

`import_agent.py` implements **multi-agent belief federation** — the ability to pull beliefs from one TMS network into another while keeping them isolated and controllable. It owns the entire lifecycle of cross-agent belief import and synchronization: namespacing, kill-switch wiring, format normalization, topological ordering, and remote-wins merge semantics.

This is the mechanism that lets multiple independent expert knowledge bases (e.g., `aap-expert`, `rhel-expert`) contribute beliefs to a shared network without ID collisions or uncontrolled cascading.

## Key Components

### Public API (4 functions)

| Function | Format | Mode | Description |
|---|---|---|---|
| `import_agent()` | Markdown | Insert-only | First-time import; skips existing nodes |
| `import_agent_json()` | JSON | Insert-only | Same, but from `network.json` (lossless justifications) |
| `sync_agent()` | Markdown | Remote-wins merge | Updates, adds, removes to match remote state |
| `sync_agent_json()` | JSON | Remote-wins merge | Same, lossless format |

All four return a stats dict with counts of imported/skipped/retracted/propagated beliefs.

### Agent Infrastructure Nodes (`_ensure_agent_nodes`)

For each agent `foo`, two control nodes are created:

- **`foo:active`** — A premise (no justifications). While IN, the agent's beliefs are live.
- **`foo:inactive`** — An SL justification with `outlist=[foo:active]`. Goes IN exactly when `foo:active` goes OUT.

Every imported belief gets `foo:inactive` in its outlist. This creates a **kill switch**: retracting `foo:active` makes `foo:inactive` go IN, which cascades every `foo:*` belief to OUT. The docstring explicitly warns that `active` is *not* placed in antecedents — that would create an always-valid fallback path that defeats per-belief retraction.

### Normalization Layer

Two parallel normalizers produce identical `claim` dicts regardless of input format:

- **`_normalize_markdown()`** — Delegates to `parse_beliefs()` from `import_beliefs.py`. Strips justifications from OUT beliefs (they become bare premises that get retracted).
- **`_normalize_json()`** — Preserves full justification structure including outlists on OUT nodes, enabling future resurrection when the remote flips a belief back to IN.

Each normalized claim has: `{id, text, is_out, source, source_hash, date, metadata, raw_justifications}` with unprefixed IDs. The same pattern applies to nogoods via `_normalize_nogoods_markdown()` and `_normalize_nogoods_json()`.

### Topological Sort (`_topo_sort_claims`)

Claims are sorted so that dependencies are added before dependents. Uses iterative passes: each pass adds claims whose antecedent dependencies are already placed. If a cycle is detected (no progress in a pass), remaining claims are appended as-is — a pragmatic choice that avoids crashing on circular references.

### Import vs Sync

**`_import_claims`** is additive only — it skips any `node_id` already present in the network. Simple and safe for first-time imports.

**`_sync_claims`** implements remote-wins merge with five operations:
1. **Add** — new beliefs from remote
2. **Update** — changed text, source, justifications, or truth value
3. **Resurrect** — remote says IN but local has `_retracted` flag → clear flag and `assert_node()`
4. **Retract** — remote says OUT → set `_retracted` and force truth_value to OUT
5. **Remove** — beliefs in local but not in remote → retract or mark `_retracted`

## Patterns

**Namespace prefixing**: Every imported belief ID becomes `agent_name:original_id`. This is applied uniformly by `_build_justifications()` which prefixes all antecedent and outlist references, and by the import/sync loops which prefix the node ID itself.

**Two-phase truth management**: Nodes are first added to the network, then `recompute_all()` propagates truth values, and finally explicit retractions are applied for OUT beliefs. This ordering matters — retracting before recompute would be overwritten.

**`_retracted` metadata flag**: A soft-retraction marker that prevents `recompute_all()` from resurrecting OUT beliefs that should stay OUT. This is the mechanism that preserves remote intent across local propagation cycles. The sync path carefully clears this flag when the remote flips a belief back to IN.

**Format-agnostic core**: The normalization layer means `_import_claims` and `_sync_claims` never touch raw markdown or JSON — they work exclusively with normalized claim dicts. Adding a new input format (YAML, SQLite) would only require a new normalizer.

## Dependencies

**Imports:**
- `Justification`, `Nogood` from `reasons_lib.__init__` — data classes for the TMS
- `parse_beliefs`, `parse_nogoods` from `import_beliefs` — markdown parsing
- `Network` from `network` — the core TMS graph

**Imported by:**
- `api.py` — exposes import/sync as CLI commands
- `pg.py` — PostgreSQL backend, likely delegates to these functions for cross-agent operations

## Flow

A typical import flows as:

1. **Normalize** — Raw markdown/JSON → list of uniform claim dicts
2. **Ensure infrastructure** — Create `agent:active` and `agent:inactive` if missing
3. **Topological sort** — Order claims so antecedents come first
4. **Add nodes** — Iterate sorted claims, prefix IDs, build justifications with `inactive` in outlist, call `network.add_node()`
5. **Import nogoods** — Prefix nogood node references, skip invalid (< 2 valid nodes) or duplicate IDs
6. **Rebuild dependents** — `_fixup_dependents()` calls `network._rebuild_dependents()` because outlist targets may not have existed when their referencing nodes were added
7. **Propagate** — `network.recompute_all()` settles truth values
8. **Retract** — Explicitly retract nodes that were OUT in the source with no justifications

Sync follows the same structure but adds update/remove logic between steps 4 and 5.

## Invariants

- **Every imported belief has `agent:inactive` in its outlist.** This is enforced in `_build_justifications()` which unconditionally prepends `inactive_id` to the outlist. There is no code path that skips this.
- **`agent:active` is never in antecedents.** The docstring calls this out as a deliberate design choice — antecedent placement would provide an always-valid fallback.
- **Topological order is best-effort, not strict.** Cycles cause remaining nodes to be appended in arbitrary order rather than raising an error.
- **`_retracted` flag takes precedence over justification validity.** A node with valid justifications but `_retracted=True` stays OUT through recompute cycles.
- **Nogoods require at least 2 valid nodes.** `_import_nogoods()` silently drops nogoods where fewer than 2 referenced nodes exist in the network.
- **Sync never deletes infrastructure nodes.** The `removed_ids` computation explicitly excludes `active_id` and `inactive_id`.

## Error Handling

Minimal — this module follows a **silent skip** strategy rather than raising exceptions:
- Duplicate node IDs during import → silently skipped (counted in `claims_skipped`)
- Outlist references to non-existent nodes → filtered out during normalization
- Nogoods with insufficient valid nodes → silently dropped
- Cyclic dependencies in topological sort → appended without ordering guarantee

No explicit try/except blocks. Errors from `network.add_node()`, `network.retract()`, or `network.assert_node()` would propagate unhandled to the caller.

## Topics to Explore

- [file] `reasons_lib/network.py` — The core TMS graph that this module writes to; understanding `add_node`, `retract`, `assert_node`, `recompute_all`, and `_rebuild_dependents` is essential
- [file] `reasons_lib/import_beliefs.py` — `parse_beliefs` and `parse_nogoods` define the markdown format contract that `_normalize_markdown` depends on
- [file] `tests/test_import_agent.py` — Test cases reveal edge cases: cyclic imports, OUT→IN resurrection, kill-switch cascades, sync removals
- [function] `reasons_lib/network.py:recompute_all` — The propagation engine that settles truth values after import; the `_retracted` flag interaction is critical
- [general] `outlist-vs-antecedent-semantics` — Understanding why `inactive` goes in outlists (not antecedents) requires grasping Doyle's SL justification semantics: antecedents must all be IN for the node to be IN, outlists must all be OUT

## Beliefs

- `import-agent-inactive-always-in-outlist` — Every imported belief unconditionally has `agent:inactive` in its outlist; there is no code path that omits it
- `import-agent-active-never-in-antecedents` — The `agent:active` premise is deliberately excluded from belief antecedents to prevent an always-valid fallback that would defeat per-belief retraction
- `sync-agent-remote-wins-semantics` — `sync_agent` implements remote-wins merge: remote IN overrides local OUT (clears `_retracted`), remote OUT overrides local IN (sets `_retracted`), and beliefs absent from remote are retracted locally
- `retracted-flag-survives-recompute` — The `_retracted` metadata flag prevents `recompute_all()` from resurrecting OUT nodes that the remote explicitly marked OUT, even when their justifications are structurally valid
- `import-topo-sort-cycle-tolerant` — Topological sorting of imported claims is best-effort; cycles cause remaining nodes to be appended in arbitrary order rather than raising an error

