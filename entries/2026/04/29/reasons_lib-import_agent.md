# File: reasons_lib/import_agent.py

**Date:** 2026-04-29
**Time:** 16:55

# `reasons_lib/import_agent.py` — Multi-Agent Belief Federation

## Purpose

This module handles **importing and synchronizing beliefs between autonomous agents** that each maintain their own Truth Maintenance System (TMS) network. It solves the problem of merging belief sets from different knowledge bases (e.g., `aap-expert`, `rhel-expert`) into a single local network without ID collisions and with a clean mechanism for bulk trust/distrust of an entire agent's contributions.

It owns two distinct operations:
- **Import** (additive, skip-on-conflict): brings in new beliefs, ignores already-imported ones.
- **Sync** (remote-wins merge): updates, adds, removes, and retracts beliefs to match the remote agent's current state.

## Key Components

### Public API (4 functions)

| Function | Format | Operation | Key Behavior |
|---|---|---|---|
| `import_agent()` | Markdown | Import | Skip existing nodes |
| `import_agent_json()` | JSON | Import | Skip existing nodes |
| `sync_agent()` | Markdown | Sync | Remote wins — updates, removes, retracts |
| `sync_agent_json()` | JSON | Sync | Remote wins — updates, removes, retracts |

All four return a stats dict summarizing what changed (counts of imported, skipped, retracted, etc.).

### Agent Infrastructure Nodes — `_ensure_agent_nodes()`

For each agent, two special nodes are created:

- **`agent:active`** — a premise node (no justifications, manually asserted). Represents "we trust this agent."
- **`agent:inactive`** — an SL-justified node with `outlist=[agent:active]`. This is IN only when `active` is OUT. It's a relay/kill-switch.

Every imported belief gets `agent:inactive` in its outlist. This means: retract `agent:active` → `agent:inactive` flips IN → every imported belief cascades OUT. This is a clean bulk-distrust mechanism using the TMS's own propagation, not imperative loops.

The docstring explicitly warns that `active` is placed in the outlist (via `inactive`), **not** in antecedents — putting it in antecedents would make every belief trivially supported regardless of its own justification chain.

### Normalization Layer

Two format-specific normalizers produce a common intermediate representation:

```python
{"id", "text", "is_out", "source", "source_hash", "date", "metadata", "raw_justifications"}
```

- `_normalize_markdown()` — parses markdown via `parse_beliefs()`, builds SL justifications from `depends_on`/`unless` fields, filters antecedents/outlist to only reference IDs present in the import set.
- `_normalize_json()` — reads the JSON export format which already has full justification structure (including outlists), so it's a more lossless path.

Both drop justifications for OUT beliefs (setting `raw_justifications=[]`) so the TMS cannot accidentally resurrect them during `recompute_all`.

### Core Logic

**`_import_claims()`**: Adds new nodes, skips existing. After adding all nodes, calls `recompute_all()` to propagate truth values, then explicitly retracts nodes that were OUT in the source.

**`_sync_claims()`**: The more complex remote-wins merge:
1. Updates text, source, metadata, and justifications on existing nodes if changed.
2. Adds new nodes not yet present.
3. Retracts local nodes that no longer exist in the remote set (`removed_ids = local_agent_ids - remote_ids`).
4. Re-asserts nodes that the remote says are IN but are locally OUT.
5. Marks OUT beliefs with `_retracted` metadata.

### Supporting Functions

- **`_topo_sort_claims()`** — orders claims so dependencies come before dependents, ensuring `add_node` sees antecedents that already exist. Falls back to appending remaining claims if cycles exist.
- **`_build_justifications()`** — prefixes all antecedent/outlist IDs with `agent:` and injects `inactive_id` into every outlist. If a non-OUT claim has no justifications at all, it creates a bare SL justification with just the inactive outlist entry.
- **`_import_nogoods()`** — imports contradictions (nogood sets), prefixing node references, filtering to nodes that actually exist, and deduplicating by ID.

## Patterns

**Namespace prefixing**: All imported IDs are prefixed with `agent_name:` (e.g., `rhel-expert:kernel-modules-loaded`). This is applied consistently in `_build_justifications()` and `_import_claims()`/`_sync_claims()`, ensuring zero collision between agents or with local beliefs.

**Normalize-then-process**: Both markdown and JSON inputs are normalized into identical intermediate dicts before hitting shared `_import_claims`/`_sync_claims` logic. This is a classic adapter pattern that keeps format concerns out of the merge logic.

**Two-phase truth assignment**: Nodes are first added (with justifications), then `recompute_all()` propagates truth values, then explicit retractions run for OUT beliefs. This ordering matters — you can't retract a node before it exists, and you want propagation to settle before applying overrides.

**Kill-switch via TMS semantics**: Rather than maintaining an imperative "disable agent" flag, trust control is implemented purely through the TMS's own outlist mechanism. This is elegant — it piggybacks on existing propagation infrastructure.

## Dependencies

**Imports:**
- `Justification`, `Nogood` from `reasons_lib.__init__` — data classes for TMS justifications and contradiction records
- `parse_beliefs`, `parse_nogoods` from `reasons_lib.import_beliefs` — markdown parsing for the belief/nogood format
- `Network` from `reasons_lib.network` — the TMS graph that holds nodes, justifications, and truth values

**Imported by:**
- `reasons_lib/api.py` — the public API layer that CLI commands call into

## Flow

A typical `import_agent()` call:

1. `_normalize_markdown()` parses beliefs text → list of normalized claim dicts
2. `_normalize_nogoods_markdown()` parses nogood text → list of normalized nogood dicts
3. `_import_claims()`:
   - `_ensure_agent_nodes()` creates `active`/`inactive` infrastructure if needed
   - `_topo_sort_claims()` orders claims by dependency
   - Loop: for each claim, prefix its ID, build justifications (with `inactive` in outlist), add node to network
   - `_import_nogoods()` adds prefixed contradiction records
   - `_fixup_dependents()` rebuilds the dependent index (outlist nodes may have been added out of order)
   - `network.recompute_all()` settles truth values
   - Explicit retraction loop for beliefs that were OUT in the source
4. Return stats dict

`sync_agent()` follows the same shape but `_sync_claims()` replaces step 3 with the remote-wins merge logic described above.

## Invariants

1. **Every imported belief has `agent:inactive` in its outlist.** This is enforced in `_build_justifications()` — it unconditionally prepends `inactive_id` to every outlist.
2. **OUT beliefs get empty justifications.** Both normalizers set `raw_justifications=[]` for OUT beliefs, and `_build_justifications()` returns `[]` for `is_out=True` claims. This prevents `recompute_all` from flipping them back IN.
3. **Topological ordering before insertion.** `_topo_sort_claims()` ensures antecedents exist before dependents. If cycles exist, remaining nodes are appended at the end (not dropped).
4. **Nogoods require at least 2 valid nodes.** `_import_nogoods()` filters out nogoods where fewer than 2 referenced nodes exist in the network.
5. **Sync removes beliefs not in remote.** `_sync_claims()` computes `removed_ids = local_agent_ids - remote_ids` and retracts/marks them, enforcing remote-wins semantics.
6. **Infrastructure nodes (`active`/`inactive`) are excluded from removal.** The sync logic filters them out via `infra_ids` before computing the removal set.

## Error Handling

This module does essentially no error handling — it trusts its inputs. `parse_beliefs()` and `parse_nogoods()` in `import_beliefs.py` handle parsing errors. If a referenced node doesn't exist in the network, the outlist/antecedent filtering in the normalizers silently drops the reference. `_import_nogoods()` silently skips nogoods with fewer than 2 valid nodes. There are no try/except blocks, no raised exceptions, and no validation of the `agent_name` parameter.

---

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network` class that this module manipulates — understanding `add_node`, `retract`, `assert_node`, `recompute_all`, and `_rebuild_dependents` is essential to understanding the import/sync semantics
- [file] `reasons_lib/import_beliefs.py` — The `parse_beliefs()` and `parse_nogoods()` parsers that produce the structured dicts consumed by `_normalize_markdown()`
- [function] `reasons_lib/network.py:recompute_all` — The truth propagation engine that runs after import — understanding when and why beliefs flip is key to debugging import results
- [file] `tests/test_import_agent.py` — Test cases that exercise import, sync, kill-switch cascading, and edge cases like duplicate imports and OUT-belief handling
- [general] `outlist-vs-antecedent-semantics` — The distinction between placing the agent premise in antecedents vs. outlist is the architectural crux of the kill-switch design — misunderstanding it breaks the entire federation model

## Beliefs

- `import-agent-inactive-in-every-outlist` — Every belief imported by `import_agent` or `sync_agent` has `agent:inactive` in its outlist, ensuring bulk retraction via the kill-switch mechanism
- `import-agent-out-beliefs-get-empty-justifications` — Beliefs that are OUT/STALE in the source are imported with no justifications, preventing `recompute_all` from resurrecting them
- `import-agent-topo-sort-before-add` — Claims are topologically sorted by antecedent dependencies before being added to the network, so `add_node` always sees its antecedents
- `import-agent-sync-remote-wins` — `sync_agent` implements remote-wins semantics: local beliefs absent from the remote set are retracted, and remote truth values override local ones
- `import-agent-namespace-prefix-isolation` — All imported node IDs are prefixed with `agent_name:` to prevent collisions between agents and with local beliefs

