# File: reasons_lib/import_beliefs.py

**Date:** 2026-04-24
**Time:** 16:49

# `reasons_lib/import_beliefs.py` — Markdown-to-Network Importer

## Purpose

This file is the bridge between the human-readable `beliefs.md` / `nogoods.md` markdown format and the internal TMS dependency network. It owns the responsibility of **parsing structured markdown into claim dicts, then materializing those claims as `Node` objects with correct `SL` justifications inside a `Network`**. It's the primary ingestion path for bulk-loading a belief system from files.

## Key Components

### `parse_repos(text: str) -> dict[str, str]`
Extracts the `## Repos` section — a simple `name: path` mapping. Stops at the next `##` heading. Used to associate beliefs with source repositories.

### `parse_beliefs(text: str) -> list[dict]`
The core parser. Walks line-by-line through markdown, keying on `### claim-id [STATUS] TYPE` headers. For each claim, it extracts:
- `id`, `status` (IN/OUT/STALE), `type`
- `text` — first non-metadata, non-empty line after the header
- `source`, `source_hash`, `date` — provenance metadata
- `depends_on`, `unless` — dependency/outlist references (comma-separated)
- `stale_reason`, `superseded_by` — retraction context

**Contract**: Returns a flat list of dicts. Does not validate that referenced dependencies exist. The `text` field captures only the *first* non-metadata line — multi-line descriptions are truncated.

### `parse_nogoods(text: str) -> list[dict]`
Parses `### nogood-N: label` entries with `Discovered`, `Resolution`, and `Affects` fields. Same line-by-line state-machine pattern as `parse_beliefs`.

### `import_into_network(network, beliefs_text, nogoods_text=None) -> dict`
The orchestrator. Does three things in order:
1. **Parses repos** and merges into `network.repos`
2. **Topologically sorts claims** so dependencies are added before dependents, then adds nodes
3. **Imports nogoods** if provided, filtering to only those where at least 2 affected nodes exist

Returns a summary dict: `claims_imported`, `claims_skipped`, `claims_retracted`, `nogoods_imported`.

## Patterns

**Line-by-line state machine**: Both `parse_beliefs` and `parse_nogoods` use the same idiom — a `current` accumulator dict that's `None` until a header line is hit, then populated by subsequent lines, then flushed when the next header arrives (plus a final flush after the loop). This is a common pattern for parsing sectioned text without a formal grammar.

**Iterative topological sort**: Rather than building a full DAG and using Kahn's algorithm, `import_into_network` does repeated passes over the remaining claims. A claim is "ready" when all its in-registry dependencies are already added. External dependencies (IDs not in the file) are treated as satisfied — the claim will simply compute as OUT if they're missing from the network. Circular dependencies are handled by a progress check: if a full pass adds nothing, the remaining claims are appended as-is and will be OUT.

**Parse-then-act separation**: Parsing (`parse_beliefs`, `parse_nogoods`) is pure — no side effects. Network mutation happens only in `import_into_network`. This makes the parsers independently testable.

## Dependencies

**Imports**: `Justification`, `Node`, `Nogood` from `reasons_lib/__init__.py` (dataclasses); `Network` from `reasons_lib/network.py`. `Path` is imported but unused — likely vestigial from when the functions took file paths directly.

**Imported by**: `reasons_lib/api.py` (the functional API layer exposes `import_beliefs` as a CLI-callable operation), `reasons_lib/import_agent.py` (LangGraph agent integration), and their corresponding test files.

## Flow

1. Caller passes raw markdown text to `import_into_network`
2. `parse_repos` extracts repo mappings → merged into network
3. `parse_beliefs` produces a list of claim dicts
4. Claims are topologically sorted by `depends_on` references
5. For each claim (in order):
   - Skip if node ID already exists in network
   - If `depends_on` or `unless` lists reference in-file claims, build an `SL` justification
   - Call `network.add_node(...)` with text, justifications, source metadata
   - If status was STALE or OUT, immediately call `network.retract()`
6. If `nogoods_text` provided, parse and import nogoods, updating `_next_nogood_id` to avoid collisions

## Invariants

- **Topological ordering**: Dependency nodes are always added to the network before their dependents (within a single import). This ensures `network.add_node` can resolve justification antecedents.
- **Idempotent on existing nodes**: Nodes already in `network.nodes` are skipped, not overwritten. This means re-importing the same file is safe but won't update changed text.
- **Only in-file references become justifications**: If a `depends_on` reference points to a claim ID not present in the beliefs file, it's silently ignored for justification construction. The claim becomes a premise (no justifications) rather than failing.
- **Nogoods require ≥2 valid nodes**: A nogood with fewer than 2 existing nodes in the network is silently dropped.
- **Nogood ID counter**: After importing a nogood, `_next_nogood_id` is bumped to `max(current, parsed_id + 1)` to prevent future ID collisions.

## Error Handling

Minimal — this code is **lenient by design**. No exceptions are raised for:
- Malformed lines (silently skipped via the `elif` chain)
- Missing dependencies (treated as external, claim becomes a premise)
- Circular dependencies (detected via the no-progress check, added anyway)
- Duplicate node IDs (skipped, counted in `skipped`)
- Nogoods referencing nonexistent nodes (dropped if < 2 valid)

The only way this code raises is if `network.add_node` or `network.retract` raises — errors propagate up from the Network layer, not from the parser.

## Topics to Explore

- [file] `reasons_lib/export_markdown.py` — The inverse operation: serializing a Network back to beliefs.md format. Understanding both directions reveals the round-trip contract and any lossy transformations.
- [function] `reasons_lib/network.py:add_node` — The downstream consumer of parsed claims. Understanding its signature and side effects (propagation triggers) clarifies what happens during import.
- [file] `tests/test_import_beliefs.py` — Test cases document the expected markdown format more precisely than the parser code, including edge cases like missing deps and circular references.
- [file] `reasons_lib/import_agent.py` — The LangGraph agent wrapper that uses this module. Shows how import fits into the larger agentic workflow.
- [general] `topological-sort-fallback` — The circular-dependency fallback (adding remaining claims without their deps met) means those nodes will be OUT. Worth understanding whether this is intentional graceful degradation or a potential source of silent data loss.

## Beliefs

- `import-skips-existing-nodes` — `import_into_network` silently skips any claim whose ID already exists in the network; it never updates or merges into existing nodes.
- `stale-maps-to-out` — STALE status in beliefs.md is mapped to OUT (retracted) in the network; there is no distinct STALE state in the TMS.
- `external-deps-become-premises` — A claim whose `depends_on` references are all absent from the beliefs file gets no justifications and is added as a premise (IN by default).
- `nogood-id-collision-prevention` — After importing nogoods, `_next_nogood_id` is set to one past the highest imported ID, preventing `network.record_nogood` from generating a conflicting ID later.
- `unused-path-import` — `Path` is imported but never used in this module; it's dead code.

