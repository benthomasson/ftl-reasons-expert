---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/import_agent.py

# `reasons/import_agent.py`

## Purpose

This file owns the **cross-agent belief import and sync** responsibility. It takes another agent's belief set (in markdown or JSON) and merges it into the local RMS (Reasoning Management System) network under a **namespace** so that beliefs from multiple agents coexist without ID collision.

It implements two distinct operations:
- **Import** (one-shot): Add new beliefs, skip existing ones. Idempotent on re-run.
- **Sync** (remote-wins): Add, update, and *remove* beliefs to mirror the remote agent's state exactly.

A critical architectural feature is the **kill-switch**: each agent gets a premise node (`agent:active`) and a relay node (`agent:inactive`). Retractions cascade through outlists, so retracting `agent:active` flips `agent:inactive` IN, which flips every imported belief OUT in one recompute pass.

## Key Components

### Public API (4 entry points)

| Function | Format | Mode | Returns |
|----------|--------|------|---------|
| `import_agent` | markdown | import (add-only) | `dict` with counts |
| `import_agent_json` | JSON dict | import (add-only) | `dict` with counts |
| `sync_agent` | markdown | sync (remote-wins) | `dict` with counts |
| `sync_agent_json` | JSON dict | sync (remote-wins) | `dict` with counts |

All four are thin wrappers: normalize → delegate to `_import_claims` or `_sync_claims`.

### Normalization Layer

Both formats are reduced to a **normalized claim dict**:
```python
{
    "id": str,
    "text": str,
    "is_out": bool,
    "source": str,
    "source_hash": str,
    "date": str,
    "metadata": dict,
    "raw_justifications": [{"type", "antecedents", "outlist", "label"}],  # unprefixed IDs
}
```

- `_normalize_markdown(beliefs_text, only_in)` — parses markdown via `parse_beliefs`, filters OUT nodes' justifications to empty, filters antecedents/outlist to known claim IDs.
- `_normalize_json(data, only_in)` — reads `data["nodes"]` dict, preserves full justification structure (including outlists).

This separation means the shared logic (`_import_claims`, `_sync_claims`) never cares about the source format.

### Agent Infrastructure Nodes

`_ensure_agent_nodes(network, agent_name, source_path)` creates two nodes if absent:

- **`{agent}:active`** — a bare premise (no justifications). Text: *"Agent 'X' beliefs are trusted."* This is the trust anchor.
- **`{agent}:inactive`** — a relay node with `Justification(type="SL", antecedents=[], outlist=[active_id])`. This is the logical NOT: it goes IN when `active` goes OUT.

Every imported belief has `inactive_id` injected into its outlist (see `_build_justifications`). This is the kill-switch wiring.

### Justification Building

`_build_justifications(claim, prefix, inactive_id, agent_name)` takes the normalized `raw_justifications` and:
1. Prefixes all antecedent and outlist IDs with `{agent_name}:`.
2. **Prepends `inactive_id`** to the outlist of every justification.
3. Falls back to a single `Justification(type="SL", antecedents=[], outlist=[inactive_id])` if none exist.

This ensures every belief is defeasible by the kill-switch, even bare premises.

### Topological Sort

`_topo_sort_claims(claims)` orders claims so that antecedents appear before dependents. It's a simple iterative peeling algorithm (not DFS/Kahn's): repeatedly scan remaining claims, emit those whose in-claim dependencies are all already placed. If a full pass makes no progress (cycle), it dumps the remainder and breaks.

### Import Logic (`_import_claims`)

Flow:
1. Ensure agent nodes exist.
2. Register the source repo path in `network.repos[agent_name]`.
3. Topo-sort claims.
4. For each claim: skip if `{prefix}{id}` already exists; otherwise add node with built justifications and metadata.
5. Collect `retract_after` list for OUT claims with *no* justifications (bare premise path).
6. For OUT claims *with* justifications: set `_retracted=True` and `truth_value="OUT"` directly (preserves justifications for future resurrection).
7. Import nogoods (prefixed, requires ≥2 valid node references).
8. `_fixup_dependents` → `recompute_all()`.
9. Execute deferred retractions.

### Sync Logic (`_sync_claims`)

Flow:
1. Ensure agent nodes, register repo path.
2. Compute `remote_ids` and `local_agent_ids` (excluding infra nodes).
3. Topo-sort.
4. For each claim:
   - **Exists locally**: compare text, source fields, metadata, justifications. Track `changed`. Handle OUT/IN transitions via `_retracted` flag and `assert_after`/`retract_after` deferral lists.
   - **New**: add node, handle OUT state.
5. **Remove** local nodes not in remote: retract if IN, set `_retracted` if already OUT.
6. `_fixup_dependents`.
7. Process `assert_after` (clear `_retracted`, re-assert).
8. `recompute_all()`.
9. Process `retract_after` (retract or set `_retracted`).
10. Import nogoods.

The `_retracted` metadata flag is the key mechanism for **locking a node OUT** even when its justifications would otherwise make it IN. Both `recompute_all()` and `_propagate()` check this flag.

### Nogood Import

`_import_nogoods(network, prefix, nogoods)` prefixes node references, validates ≥2 nodes exist in the network, and appends `Nogood` objects. Deduplicates by prefixed ID.

## Patterns

- **Normalization/Adapter**: Two source formats → one canonical claim structure → shared logic. The `_normalize_*` functions are the adapters.
- **Kill-switch via outlist cascade**: Rather than using antecedents (which would make `agent:active` an always-valid support defeating per-belief retraction), the design puts `agent:inactive` in the **outlist**. This is a deliberate non-monotonic reasoning pattern.
- **Deferred operations**: `retract_after` and `assert_after` lists defer truth-value changes until after `recompute_all()`, preventing intermediate state from corrupting the propagation pass.
- **Idempotent import / convergent sync**: `import` skips existing nodes; `sync` converges local to remote state.
- **Flag-based locking**: `_retracted` in metadata prevents `recompute_all` from resurrecting a node that the remote explicitly set to OUT but still carries justifications.

## Dependencies

### Imports (this file depends on)
- `pathlib.Path` — for resolving `source_path` to a repo directory.
- `reasons.Justification`, `reasons.Nogood` — core domain types.
- `reasons.import_beliefs.parse_beliefs`, `parse_nogoods` — markdown parsers.
- `reasons.network.Network` — the graph structure being mutated.

### Imported by (depends on this)
- `reasons/api.py` — exposes the HTTP/API surface that calls these functions.
- `reasons/pg.py` — likely the Postgres-backed persistence layer that dispatches import/sync.
- `reasons_lib/import_agent.py` — the library mirror (same code, different package).

## Flow (End-to-End Example: `sync_agent`)

```
sync_agent(network, "aap-expert", md_text, ...)
  │
  ├─ _normalize_markdown(md_text, only_in=False)
  │    → [claim1, claim2, ...]  (unprefixed IDs, filtered outlist)
  │
  ├─ _normalize_nogoods_markdown(nogoods_text)
  │    → [ng1, ng2, ...]
  │
  └─ _sync_claims(network, "aap-expert", claims, source_path, nogoods)
       │
       ├─ _ensure_agent_nodes → creates "aap-expert:active" + "aap-expert:inactive"
       ├─ network.repos["aap-expert"] = resolved_parent
       ├─ _topo_sort_claims(claims) → ordered list
       │
       ├─ for claim in ordered:
       │    ├─ exists? → diff text/source/metadata/justs, set flags
       │    └─ new? → add_node with prefixed justifications
       │
       ├─ remove (local_agent_ids - remote_ids)
       ├─ _fixup_dependents(network)
       ├─ process assert_after (clear _retracted, re-assert)
       ├─ network.recompute_all() → returns propagated nodes
       ├─ process retract_after (retract or set _retracted)
       ├─ _import_nogoods(network, "aap-expert:", nogoods)
       │
       └─ return {beliefs_added, beliefs_updated, ...}
```

## Invariants

1. **Namespace isolation**: Every node, nogood, and justification reference for agent `X` is prefixed `X:`. No two agents can collide on node IDs.
2. **Kill-switch wiring**: Every imported belief's outlist contains `X:inactive`. The `X:active` premise is *never* placed in any antecedents list.
3. **`X:inactive` truth semantics**: `X:inactive` is IN ⟺ `X:active` is OUT. (It has `outlist=[active_id]`, `antecedents=[]`.)
4. **OUT + no justifications → hard retract**: These nodes lose their justification entirely and are retracted via `network.retract()`.
5. **OUT + justifications → soft lock**: Justifications are preserved; `_retracted=True` prevents recompute from flipping them back IN. Sync can clear this flag when remote sends IN.
6. **Topological ordering**: Antecedents are always added before dependents, so `add_node` can register dependents at creation time (except for outlist additions, handled by `_fixup_dependents`).
7. **Nogood minimum arity**: A nogood must reference ≥2 nodes that exist in the network, or it's skipped.

## Error Handling

This file is **notably quiet on error handling**:

- **No exceptions are raised for "normal" operational issues.** Missing nodes in outlist references are silently filtered (`_normalize_markdown` checks `d in claim_ids`; `_normalize_json` checks `o in node_ids`). `_import_nogoods` filters to `valid_nodes` and skips if <2.
- **Topo-sort cycle safety**: If a full pass makes zero progress, the remaining claims are appended unsorted rather than raising. This means circular dependencies in an agent's beliefs will import but may not propagate correctly on first recompute.
- **`network.add_node`** is assumed to validate ID uniqueness and other constraints. This file doesn't wrap it in try/except.
- **`_justifications_match`** does a structural deep comparison (type, antecedents, outlist, label) — no exceptions, just returns `False` on any mismatch.
- **No input validation** on `agent_name` (could be empty string, producing `:active`), `source_path`, or the claim dicts. Callers (API layer, CLI) are expected to validate.

The design philosophy: this module assumes well-formed input from its parsers (`parse_beliefs`, `parse_nogoods`) and trusts the `Network` class for structural integrity.

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class: how `add_node`, `retract`, `assert_node`, `recompute_all`, and `_rebuild_dependents` interact. Understanding the propagation engine is essential to understanding why the kill-switch outlist pattern works.
- [file] `reasons/import_beliefs.py` — The `parse_beliefs` and `parse_nogoods` functions. This file's normalization layer depends entirely on their output contract (keys like `status`, `depends_on`, `unless`).
- [function] `reasons/import_agent.py:_sync_claims` — The sync path is the most complex function here. Trace how `_retracted` interacts with `recompute_all` to understand the lock/resurrect lifecycle of OUT nodes with justifications.
- [file] `tests/test_sync_agent.py` — The test suite for sync reveals the expected edge cases: belief addition, removal, text changes, OUT→IN flips, IN→OUT transitions, and the interaction with the kill-switch.
- [general] `rms-outlist-semantics` — The broader reasoning system's outlist/non-monotonic semantics. Understanding why `agent:active` is deliberately *not* an antecedent (per the module docstring) requires knowledge of how antecedents provide "always-valid fallback" in this system.

## Beliefs

- `agent-inactive-outlist-wiring-enables-cascade-retraction` — Retractions of `agent:active` propagate OUT to all imported beliefs exclusively through the `agent:inactive` outlist entry, never through antecedent support.
- `retracted-metadata-flag-prevents-recompute-resurrection` — A node with `_retracted=True` in its metadata is skipped by both `recompute_all()` and `_propagate()`, keeping it OUT even when its justifications would otherwise evaluate to IN.
- `import-agent-uses-outlist-not-antecedents-for-trust` — The `agent:active` premise is deliberately excluded from antecedent lists because antecedents provide an always-valid fallback that would defeat per-belief retraction; the trust gate is enforced through outlist instead.
- `sync-agent-remote-wins-semantics` — `sync_agent` removes any locally-present agent-prefixed node whose ID is absent from the remote claim set, making the remote the single source of truth for that agent's belief set.
- `normalize-layers-produce-identical-claim-structure` — Both `_normalize_markdown` and `_normalize_json` emit the same dict schema (id, text, is_out, raw_justifications with unprefixed IDs), allowing `_import_claims` and `_sync_claims` to be format-agnostic.
