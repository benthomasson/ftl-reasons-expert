# File: reasons_lib/import_beliefs.py

**Date:** 2026-04-29
**Time:** 17:05

# `reasons_lib/import_beliefs.py` — Markdown-to-Network Importer

## Purpose

This module is the **deserialization bridge** between the human-readable markdown format (`beliefs.md`, `nogoods.md`) and the in-memory RMS (Reason Maintenance System) network. It owns the responsibility of parsing structured markdown into claim/nogood dicts, then materializing them as `Node` objects with proper `SL` justifications inside a `Network` instance.

It's the inverse of `export_markdown.py` — together they form the round-trip serialization layer for the belief system.

## Key Components

### `parse_repos(text: str) -> dict[str, str]`
Extracts the `## Repos` section from beliefs markdown. Returns a `name → path` mapping. Stops parsing at the next `##` header. Used to carry repo metadata through import/export cycles.

### `parse_beliefs(text: str) -> list[dict]`
The core parser. Scans line-by-line for `### claim-id [STATUS] TYPE` headers and accumulates metadata fields (`Source`, `Depends on`, `Unless`, `Stale reason`, etc.) into structured dicts. Each claim dict has a fixed schema with these keys: `id`, `status`, `type`, `text`, `source`, `source_hash`, `date`, `depends_on`, `unless`, `stale_reason`, `superseded_by`.

The first non-metadata, non-empty line after a header becomes the claim's `text` (the human-readable description). Unrecognized `- ` lines are silently skipped — this makes the parser forward-compatible with new metadata fields.

### `parse_nogoods(text: str) -> list[dict]`
Parses `nogoods.md` format. Expects headers like `### nogood-7: label text` and extracts `Discovered`, `Resolution`, and `Affects` fields. The `Affects` field is a comma-separated list of node IDs.

### `import_into_network(network, beliefs_text, nogoods_text=None) -> dict`
The orchestrator. Takes a `Network` instance and raw markdown strings, then:
1. Parses repos and merges them into the network
2. Parses claims and topologically sorts them
3. Creates nodes with appropriate justifications
4. Retracts STALE/OUT nodes after creation
5. Imports nogoods if provided
6. Returns a summary dict with counts

## Patterns

**Iterative topological sort**: Rather than a standard DFS topo-sort, it uses a multi-pass Kahn-like algorithm. Each pass picks claims whose in-registry dependencies are all satisfied. This handles three cases gracefully:
- Claims with no dependencies (premises) go first
- Derived claims wait until their antecedents are added
- Circular or missing dependencies break the loop after `max_passes` — remaining claims are appended as-is and will compute as OUT naturally

**Add-then-retract**: STALE and OUT claims are first added to the network with their full justification structure, *then* retracted. This preserves the dependency graph even for beliefs that aren't currently held — critical for the TMS, which needs to know what *would* be believed if a retracted premise were restored.

**Idempotent skip**: Nodes already present in the network are silently skipped (counted as `skipped`). This makes re-importing safe.

**Forward-compatible parsing**: Unknown `- ` metadata lines are ignored via the `pass` branch, so adding new fields to the export format won't break the importer.

## Dependencies

**Imports from:**
- `.network.Network` — the target data structure being populated
- `. (Justification, Node, Nogood)` — domain types from `__init__.py`
- `re` — regex for header parsing
- `pathlib.Path` — imported but unused (likely vestigial)

**Imported by:**
- `reasons_lib/api.py` — the CLI-facing API layer calls this for `import-json` / `import-markdown` commands
- `reasons_lib/import_agent.py` — agent-based import workflows
- `tests/test_import_beliefs.py`, `tests/test_nogood_id.py` — test coverage

## Flow

```
beliefs.md text
    │
    ├── parse_repos() ──→ dict of repo name → path
    │
    └── parse_beliefs() ──→ list[dict] (flat claim records)
            │
            ├── topological sort (iterative, cycle-tolerant)
            │
            └── for each claim (in topo order):
                    ├── skip if already in network
                    ├── build Justification if depends_on/unless exist
                    ├── network.add_node(...)
                    └── network.retract(...) if STALE or OUT

nogoods.md text (optional)
    │
    └── parse_nogoods() ──→ list[dict]
            │
            └── for each nogood:
                    ├── filter to nodes that exist in network
                    ├── skip if < 2 valid nodes
                    └── network.nogoods.append(...)
                        + update _next_nogood_id high-water mark
```

## Invariants

1. **Dependency ordering**: Nodes are added to the network in topological order. A derived node's antecedents are guaranteed to exist (or be external) before it is added.
2. **No duplicate nodes**: If `claim["id"]` already exists in `network.nodes`, it is skipped entirely — no upsert, no merge.
3. **Retraction is post-creation**: A STALE/OUT node is always added first, then retracted. The node and its justification structure exist in the network regardless of truth value.
4. **Nogood minimum cardinality**: Nogoods are only imported if at least 2 of their affected nodes exist in the network. A single-node "nogood" is meaningless.
5. **Nogood ID monotonicity**: `_next_nogood_id` is set to `max(current, parsed_id + 1)`, ensuring future auto-generated IDs won't collide with imported ones.

## Error Handling

This module is **lenient by design** — it swallows rather than raises:
- Missing dependencies in the topo-sort are treated as external and don't block the claim
- Circular dependencies trigger a fallback append (no error raised)
- Unknown metadata lines are silently skipped
- Nogoods referencing nonexistent nodes are filtered down; if fewer than 2 remain, the nogood is silently dropped
- The only hard failures would come from `network.add_node()` or `network.retract()` — those propagate unhandled

There is no input validation on claim IDs, dates, or source hashes. The parser trusts the markdown was produced by the corresponding exporter.

## Topics to Explore

- [file] `reasons_lib/export_markdown.py` — The inverse of this module; understanding both reveals the full round-trip serialization contract
- [function] `reasons_lib/network.py:Network.add_node` — What actually happens when a node with SL justifications is added — truth value computation, dependent tracking
- [function] `reasons_lib/network.py:Network.retract` — How retraction cascades work, and why add-then-retract preserves structure
- [file] `tests/test_import_beliefs.py` — Test cases that document expected parsing behavior and edge cases (circular deps, missing deps, re-import)
- [general] `outlist-gate-pattern` — How `unless` fields create GATE beliefs that flip truth value when blockers are retracted — connects to the known issue in CLAUDE.md about outlist tracking

## Beliefs

- `import-beliefs-topo-sort-is-cycle-tolerant` — The topological sort detects circular dependencies by checking for zero progress across a pass and appends remaining claims without raising an error
- `import-beliefs-add-then-retract-for-out-nodes` — STALE and OUT claims are first added to the network with full justification structure, then retracted, preserving the dependency graph for potential future restoration
- `import-beliefs-skips-existing-nodes` — If a claim ID already exists in `network.nodes`, it is silently skipped with no upsert or merge behavior
- `import-beliefs-nogood-minimum-two-nodes` — Nogoods are only imported when at least 2 of their referenced nodes exist in the network; otherwise they are silently dropped
- `import-beliefs-path-import-unused` — `pathlib.Path` is imported but never referenced in the module, suggesting a vestigial import from an earlier version

