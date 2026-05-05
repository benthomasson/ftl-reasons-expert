# File: reasons_lib/import_agent.py

**Date:** 2026-05-05
**Time:** 15:25

## Purpose

`import_agent.py` is the **multi-agent federation layer** for the TMS network. It lets one `reasons` knowledge base ingest beliefs from another agent's exported beliefs (markdown or JSON), namespacing them so multiple agents' belief sets coexist without ID collisions. It owns two operations:

- **Import** (append-only): adds new beliefs from a remote agent, skipping duplicates.
- **Sync** (remote-wins merge): adds, updates, retracts, and removes beliefs to match the remote agent's current state.

## Key Components

### Public API (4 functions)

| Function | Format | Operation | Returns |
|---|---|---|---|
| `import_agent()` | Markdown | Import | stats dict |
| `import_agent_json()` | JSON | Import | stats dict |
| `sync_agent()` | Markdown | Sync | stats dict |
| `sync_agent_json()` | JSON | Sync | stats dict |

All four follow the same pattern: normalize input → delegate to shared logic (`_import_claims` or `_sync_claims`).

### The Agent Kill Switch (`_ensure_agent_nodes`)

For each agent, two infrastructure nodes are created:

- **`agent:active`** — a bare premise (no justification). While IN, the agent's beliefs are trusted.
- **`agent:inactive`** — has an SL justification with `outlist=[agent:active]`. This means `inactive` is IN precisely when `active` is OUT (Doyle's closed-world assumption on the outlist).

Every imported belief gets `agent:inactive` in its **outlist**, not `agent:active` in its antecedents. This is a critical design choice documented in the module docstring: if `active` were an antecedent, it would provide a second valid justification path, meaning you could never retract individual beliefs — they'd always have a fallback. Instead, placing `inactive` in the outlist means: "this belief is valid *unless* the agent is deactivated." Retracting `active` flips `inactive` to IN, which cascades every imported belief to OUT.

### Normalization Layer

`_normalize_markdown()` and `_normalize_json()` convert both input formats into a common claim structure:

```python
{"id", "text", "is_out", "source", "source_hash", "date", "metadata", "raw_justifications"}
```

Where `raw_justifications` contains unprefixed IDs. Key filtering: antecedents and outlist entries that reference nodes *not present in the import set* are dropped during normalization, preventing dangling references. OUT/STALE beliefs get empty justification lists — they're imported as bare premises so `recompute_all` can't resurrect them.

### `_topo_sort_claims`

A simple iterative topological sort over antecedent dependencies. Claims with no unmet dependencies are emitted first. If a cycle is detected (no progress in a pass), remaining claims are appended in original order. This ensures that when `add_node` is called, antecedent nodes already exist.

### `_import_claims` (Import Logic)

1. Ensures agent infrastructure nodes exist.
2. Topologically sorts claims.
3. For each claim: skips if the node ID already exists, otherwise creates it with prefixed justifications.
4. Imports nogoods.
5. Calls `_fixup_dependents()` → `recompute_all()` to propagate truth values.
6. Retracts nodes that were OUT in the source.

### `_sync_claims` (Sync Logic)

More complex — implements a three-way diff:

1. **Existing nodes**: compares text, source, justifications, truth value. Updates changed fields. Nodes that were retracted locally but are IN remotely get re-asserted (`assert_after` list).
2. **New nodes**: added as in import.
3. **Removed nodes** (local but not in remote): retracted or marked `_retracted` in metadata.
4. Ordering matters: `_fixup_dependents` → assert → `recompute_all` → retract. Assertions happen before recompute so the propagation engine sees the restored justifications.

## Patterns

- **Normalize-then-operate**: Two format parsers feed a single shared logic path. This is the [Strategy pattern](https://en.wikipedia.org/wiki/Strategy_pattern) applied at the data layer — format-specific concerns are isolated in normalization.
- **Namespace prefixing**: All imported IDs get `agent_name:` prepended. This is applied at the justification-building stage (`_build_justifications`), not during normalization — raw justifications carry unprefixed IDs.
- **Deferred retraction**: OUT beliefs are imported first (so they exist as nodes), then retracted after `recompute_all`. This prevents the propagation engine from incorrectly computing truth values for nodes that depend on the retracted ones.
- **Metadata flags**: `_retracted` (underscore-prefixed, indicating internal) is used as a soft-delete marker in sync, distinguishing "removed by remote" from "never existed."

## Dependencies

**Imports:**
- `Justification`, `Nogood` from `reasons_lib.__init__` — data classes for TMS justifications and contradictions.
- `parse_beliefs`, `parse_nogoods` from `.import_beliefs` — markdown parsers that extract structured belief/nogood data.
- `Network` from `.network` — the core TMS graph; owns nodes, justifications, truth propagation, retraction.

**Imported by:**
- `reasons_lib/api.py` — the public API layer that CLI commands call. The four public functions here are the interface `api.py` delegates to.

## Flow

A typical `import_agent` call:

```
import_agent(network, "aap-expert", markdown_text, nogoods_text)
  │
  ├─ _normalize_markdown(markdown_text)     → list of claim dicts
  ├─ _normalize_nogoods_markdown(nogoods)   → list of nogood dicts
  │
  └─ _import_claims(network, "aap-expert", claims, source_path, nogoods)
       │
       ├─ _ensure_agent_nodes()             → creates aap-expert:active, aap-expert:inactive
       ├─ _topo_sort_claims()               → dependency-ordered claims
       │
       ├─ for each claim:
       │    ├─ skip if aap-expert:<id> exists
       │    ├─ _build_justifications()      → prefixed Justification objects
       │    └─ network.add_node()
       │
       ├─ _import_nogoods()
       ├─ _fixup_dependents()               → network._rebuild_dependents()
       ├─ network.recompute_all()           → propagate truth values
       │
       └─ retract nodes that were OUT in source
```

## Invariants

1. **Every imported belief has `agent:inactive` in its outlist.** This is the mechanism for the kill switch — there are no exceptions, including for premises.
2. **`agent:active` is never an antecedent of imported beliefs.** Violating this would defeat per-belief retraction (the docstring explicitly warns about this).
3. **OUT beliefs are imported with empty justification lists.** This prevents `recompute_all` from flipping them back to IN.
4. **Nodes are added in topological order of antecedent dependencies.** The sort guarantees that `add_node` can register dependents correctly.
5. **Import is idempotent on node IDs** — if a prefixed node already exists, it's skipped (import) or updated (sync).
6. **Sync removes beliefs not in the remote set.** Local-only agent beliefs (excluding infrastructure nodes) are retracted.
7. **Nogoods require at least 2 valid nodes** to be imported (`len(valid_nodes) >= 2` in `_import_nogoods`).

## Error Handling

Essentially none — this module trusts its callers and the network API. There are no try/except blocks, no input validation, and no error returns. If `network.add_node()` or `network.retract()` raises, it propagates unhandled. Invalid belief IDs in justifications are silently dropped during normalization (the `if d in claim_ids` / `if o in node_ids` filters). This is intentional: dangling references in exports are expected and should be quietly ignored rather than failing the import.

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network` class that owns `add_node`, `retract`, `recompute_all`, and `_rebuild_dependents` — understanding those methods is essential for understanding what this module's calls actually do.
- [function] `reasons_lib/import_beliefs.py:parse_beliefs` — The markdown parser that converts `beliefs.md` format into structured claim dicts; controls what fields are available during normalization.
- [file] `tests/test_import_agent.py` — Test suite that exercises import/sync for both formats, the kill switch mechanism, idempotency, and edge cases like circular dependencies.
- [general] `outlist-vs-antecedent-kill-switch` — Why `inactive` is placed in the outlist rather than `active` in the antecedents. This is the core design decision that makes per-belief retraction work alongside the global kill switch.
- [function] `reasons_lib/network.py:recompute_all` — The truth propagation engine; understanding when and how it re-evaluates nodes explains why the ordering of operations (add → fixup → recompute → retract) matters.

## Beliefs

- `import-agent-inactive-in-outlist-not-antecedent` — Every imported belief has `agent:inactive` in its outlist; `agent:active` is never placed in antecedents, preserving per-belief retraction capability.
- `import-agent-out-beliefs-get-empty-justifications` — Beliefs that are OUT or STALE in the source are imported with empty justification lists, preventing `recompute_all` from resurrecting them.
- `import-agent-topo-sort-before-add` — Claims are topologically sorted by antecedent dependencies before being added to the network, so `add_node` can find antecedent nodes and register dependents.
- `sync-agent-remote-wins-semantics` — `sync_agent` implements remote-wins merge: new remote beliefs are added, changed beliefs are updated, and beliefs absent from the remote are retracted locally.
- `import-agent-dangling-refs-silently-dropped` — Justification antecedents and outlist entries referencing nodes not in the import set are silently filtered during normalization rather than raising errors.

