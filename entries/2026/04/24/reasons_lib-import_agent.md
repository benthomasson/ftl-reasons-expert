# File: reasons_lib/import_agent.py

**Date:** 2026-04-24
**Time:** 16:45

# `reasons_lib/import_agent.py` — Multi-Agent Belief Federation

## Purpose

This file owns the problem of **importing beliefs from one TMS network into another** without ID collisions or loss of retraction semantics. It's the federation layer — it lets multiple autonomous agents (each with their own `beliefs.md` or `network.json`) contribute beliefs into a single local network while keeping them independently controllable.

The core responsibility: take a foreign agent's beliefs, namespace them, wire them into the local dependency graph with a kill switch, and provide both one-shot import and ongoing sync.

## Key Components

### Public API (4 functions)

| Function | Format | Mode | Description |
|---|---|---|---|
| `import_agent()` | Markdown | Append-only | Adds new beliefs, skips existing |
| `import_agent_json()` | JSON | Append-only | Same, but from JSON export |
| `sync_agent()` | Markdown | Remote-wins | Adds, updates, removes to match remote |
| `sync_agent_json()` | JSON | Remote-wins | Same, but from JSON export |

All four return a stats dict describing what happened (counts of imported, skipped, retracted, etc.).

### The Kill Switch Pattern (`_ensure_agent_nodes`)

For each agent, two infrastructure nodes are created:

- **`agent:active`** — A premise node (no justifications, IN by default). Represents "we trust this agent."
- **`agent:inactive`** — Has justification `SL(antecedents=[], outlist=[agent:active])`. This means it's IN only when `active` is OUT.

Every imported belief gets `agent:inactive` in its **outlist**, not `agent:active` in its antecedents. This is a critical design choice explained in the docstring: if `active` were an antecedent, it would provide a fallback justification that defeats per-belief retraction. With the outlist approach, retracting `active` makes `inactive` go IN, which cascades every imported belief OUT — but individual beliefs can still be retracted independently.

### Normalization Layer

Two format-specific normalizers produce a common intermediate representation:

- `_normalize_markdown()` — Parses `beliefs.md` format via `parse_beliefs()`, strips dependencies referencing unknown IDs, maps STALE/OUT to `is_out=True`
- `_normalize_json()` — Parses JSON export format, preserves full justification structure including outlists

Both produce dicts with shape: `{id, text, is_out, source, source_hash, date, metadata, raw_justifications}` where IDs are **unprefixed**. Prefixing happens later in `_build_justifications`.

Nogoods get similar treatment via `_normalize_nogoods_markdown()` and `_normalize_nogoods_json()`.

### `_import_claims` vs `_sync_claims`

**`_import_claims`**: Append-only. If a node ID already exists in the network, it's skipped. OUT beliefs are imported then explicitly retracted after propagation.

**`_sync_claims`**: Remote-wins merge. It:
1. Identifies existing local nodes for this agent (by prefix)
2. Updates text, source, justifications, and truth values for existing nodes
3. Adds new nodes not yet present
4. Retracts local nodes that are absent from the remote (the "removed" set: `local_agent_ids - remote_ids`)
5. Handles the IN↔OUT transition: nodes that went OUT remotely get retracted; nodes that came back IN get `assert_node` called

## Patterns

**Normalize-then-operate**: Both formats are normalized to the same intermediate structure before any graph operations. This keeps `_import_claims` and `_sync_claims` format-agnostic — a classic adapter pattern.

**Topological sorting** (`_topo_sort_claims`): Claims are ordered so that dependencies are added before dependents. Uses iterative BFS with a cycle-breaker (if no progress, append remaining — handles circular dependencies gracefully rather than failing).

**Deferred propagation**: Nodes are added first, then `_fixup_dependents` rebuilds the dependent index, then `recompute_all()` propagates truth values, and only *then* are OUT nodes explicitly retracted. This ordering avoids intermediate states where a node can't find its dependencies.

**Namespace isolation**: The `prefix = f"{agent_name}:"` convention ensures no collisions between agents or with local beliefs. Infrastructure nodes (`active`, `inactive`) use the same prefix.

## Dependencies

**Imports from:**
- `reasons_lib.__init__` — `Justification`, `Nogood` dataclasses
- `reasons_lib.import_beliefs` — `parse_beliefs`, `parse_nogoods` (markdown parsing)
- `reasons_lib.network` — `Network` class (the dependency graph)

**Imported by:**
- `reasons_lib/api.py` — The functional API layer that wraps these functions for CLI use

## Flow

A typical `sync_agent()` call:

1. **Normalize**: `_normalize_markdown()` parses raw text → list of claim dicts with unprefixed IDs
2. **Ensure infra**: `_ensure_agent_nodes()` creates `agent:active` and `agent:inactive` if missing
3. **Compute diff**: Compare `remote_ids` against `local_agent_ids` (nodes with matching prefix, excluding infra)
4. **Topo sort**: `_topo_sort_claims()` orders claims so antecedents come first
5. **Apply changes**: Loop through ordered claims — add new, update existing, track retract/assert lists
6. **Remove stale**: Retract local nodes absent from remote
7. **Rebuild graph**: `_fixup_dependents()` → `recompute_all()` → apply deferred retractions/assertions
8. **Import nogoods**: `_import_nogoods()` adds contradiction records with prefixed IDs
9. **Return stats**: Dict with counts of every operation performed

## Invariants

- **Prefix invariant**: Every node from agent `foo` has ID starting with `foo:`. Infrastructure nodes are `foo:active` and `foo:inactive`.
- **Outlist wiring**: Every non-OUT imported belief has `agent:inactive` in its outlist. Never `agent:active` in antecedents.
- **Dangling reference filtering**: Antecedents and outlist entries referencing IDs not in the import set are silently dropped during normalization (`if d in claim_ids` / `if o in claim_ids`).
- **Idempotent infra creation**: `_ensure_agent_nodes` is safe to call multiple times — it checks existence before creating.
- **Topo sort termination**: Guaranteed by the cycle-breaker: if a pass makes no progress, remaining nodes are appended and the loop exits.
- **Nogoods require ≥2 valid nodes**: `_import_nogoods` only creates a nogood if at least 2 of its referenced nodes exist in the network.

## Error Handling

This module does **no explicit error handling** — no try/except, no validation errors raised. It relies on:
- The normalization layer silently dropping unknown references
- `Network.add_node` and `Network.retract` handling invalid states
- The caller (API layer) to catch exceptions from the Network

This is a deliberate choice: the module is tolerant of partial/inconsistent input (drops what it can't resolve) rather than failing fast. A belief referencing a nonexistent dependency simply gets that dependency removed from its justification rather than raising an error.

## Topics to Explore

- [function] `reasons_lib/network.py:_rebuild_dependents` — The dependent index rebuild that `_fixup_dependents` delegates to; understanding this is key to knowing why deferred fixup is needed
- [file] `reasons_lib/import_beliefs.py` — The markdown parser that `_normalize_markdown` depends on; defines the `beliefs.md` grammar
- [function] `reasons_lib/network.py:recompute_all` — The BFS propagation engine that determines truth values after import
- [file] `tests/test_sync_agent.py` — Shows the sync semantics in action: what happens on add, remove, update, and truth-value transitions
- [general] `kill-switch-pattern` — The `active`/`inactive` relay pattern is reusable anywhere you need a single retraction to cascade across a group of nodes; worth understanding as an RMS idiom

## Beliefs

- `import-agent-outlist-not-antecedent` — Imported beliefs wire `agent:inactive` into their outlist, never `agent:active` into antecedents; this preserves per-belief retraction independence
- `sync-is-remote-wins` — `sync_agent` and `sync_agent_json` implement remote-wins semantics: remote truth values override local, and nodes absent from remote are retracted locally
- `normalization-drops-unknown-refs` — Both `_normalize_markdown` and `_normalize_json` silently drop antecedent/outlist references to IDs not present in the import set, preventing dangling edges
- `topo-sort-breaks-cycles` — `_topo_sort_claims` handles circular dependencies by appending remaining nodes after detecting no progress, guaranteeing termination
- `deferred-retraction-ordering` — Nodes are added and propagated before explicit retractions are applied; this ensures the dependency graph is fully wired before truth values are finalized

