# File: tests/test_import_beliefs.py

**Date:** 2026-04-24
**Time:** 16:50

# `tests/test_import_beliefs.py` — Explanation

## Purpose

This file is the test suite for the **beliefs.md import pipeline** — the bridge between a human-authored markdown document (a "belief registry") and the internal `Network` dependency graph. It validates three layers of that pipeline:

1. **Parsing** markdown into structured claim dicts (`parse_beliefs`, `parse_nogoods`)
2. **Importing** those parsed claims into a live `Network` with correct justifications, truth values, and metadata
3. **Post-import invariants** — that the imported network actually behaves correctly under retraction and cascading, not just that it loaded data

This is a critical integration boundary: beliefs.md is the user-facing format, and any parse error or import bug silently corrupts the dependency graph.

## Key Components

### Test Fixtures (module-level constants)

- **`SAMPLE_BELIEFS`** — A synthetic beliefs.md fragment with five nodes covering every structural variant: premises (no dependencies), derived nodes (with `Depends on`), stale nodes (with `Stale reason` / `Superseded by`), and a two-level dependency chain (`chain-e → derived-c → premise-a, premise-b`). This is the canonical test document.

- **`SAMPLE_NOGOODS`** — A synthetic nogoods.md fragment with two contradiction records. Each has an ID, label, resolution text, and an `Affects` list referencing node IDs from `SAMPLE_BELIEFS`.

### Test Classes

**`TestParseBeliefs`** — Unit tests for `parse_beliefs()`. Validates that the parser correctly extracts from the markdown heading format `### id [STATUS] TYPE`:
- Node count, ID, status (`IN`/`STALE`), type (`OBSERVATION`/`DERIVED`)
- Free-text description (the line after the heading)
- Metadata fields: `Source`, `Source hash`, `Date`, `Depends on`, `Unless`, `Stale reason`, `Superseded by`
- Default values: empty lists for `depends_on` and `unless` when absent

**`TestParseNogoods`** — Unit tests for `parse_nogoods()`. Validates extraction of nogood ID, label, `Affects` list, and `Resolution` text from the markdown format.

**`TestImportIntoNetwork`** — Integration tests for `import_into_network()`. This is the most important class — it verifies that parsed claims produce a correct `Network`:
- Premises get no justifications; derived nodes get SL justifications with correct antecedents
- STALE claims are retracted to `OUT` with metadata preserved
- The reverse index (`dependents`) is populated
- Retraction cascading works post-import (retract `premise-a` → `derived-c` and `chain-e` go OUT)
- Duplicate imports are idempotent (second import skips all, imports zero)
- Nogoods are imported and nogoods referencing missing nodes are silently skipped
- Import returns a summary dict with counts: `claims_imported`, `claims_retracted`, `claims_skipped`, `nogoods_imported`

**`TestImportRealRegistry`** — An optional smoke test against a real `~/git/beliefs-pi/beliefs.md` file. Uses `pytest.skip` if the file doesn't exist, so it only runs on machines that have the real registry. Validates basic sanity: at least one claim imported, more IN than OUT.

## Patterns

- **Fixture-as-constant**: The test documents are module-level string constants rather than file fixtures, making the tests self-contained and fast — no disk I/O for the core suite.
- **`by_id` dict pattern**: Multiple tests parse the same document and index by ID (`{c["id"]: c for c in claims}`) to assert on specific nodes without depending on parse order.
- **Layered testing**: Parse tests validate the parser in isolation, then import tests validate the full pipeline including network construction and propagation. This means a parse bug vs. an import bug are distinguishable from test failures.
- **Idempotency test** (`test_skip_duplicates`): Imports twice and asserts the second import produces zero new nodes — guards against accidental duplicate creation.
- **Post-import behavioral tests**: `test_cascading_works_after_import` goes beyond "did it load" to "does the loaded graph actually propagate correctly" — this catches subtle bugs where nodes load with correct truth values but broken justification wiring.
- **Graceful skip for optional fixtures**: `TestImportRealRegistry` uses `pytest.skip` for the real file, not `pytest.mark.skipif` — the check happens at fixture resolution time.

## Dependencies

**Imports:**
- `reasons_lib.network.Network` — the dependency graph engine
- `reasons_lib.import_beliefs.parse_beliefs` — markdown → list of claim dicts
- `reasons_lib.import_beliefs.parse_nogoods` — markdown → list of nogood dicts
- `reasons_lib.import_beliefs.import_into_network` — claim dicts + Network → populated graph

**Imported by:** Nothing — this is a leaf test file.

## Flow

1. Each `TestParseBeliefs` test calls `parse_beliefs(SAMPLE_BELIEFS)` → gets back a `list[dict]` where each dict has keys `id`, `status`, `type`, `text`, `source`, `source_hash`, `depends_on`, `unless`, `stale_reason`, `superseded_by`.
2. Each `TestImportIntoNetwork` test creates a fresh `Network()`, calls `import_into_network(net, SAMPLE_BELIEFS)`, then inspects `net.nodes` for correct structure. The import function internally calls `parse_beliefs`, creates nodes, wires justifications, runs propagation, and retracts stale nodes.
3. `test_cascading_works_after_import` then mutates the network (`net.retract("premise-a")`) and checks that BFS propagation correctly cascades through the imported dependency chain.

## Invariants

- **Parse completeness**: `parse_beliefs` must extract exactly one dict per `### heading` block. Count is tested explicitly.
- **Idempotency**: Importing the same document twice must not create duplicates — second import yields `claims_imported == 0`, `claims_skipped == 5`.
- **STALE → OUT mapping**: The beliefs.md format uses `STALE` as a status, but the network only has `IN`/`OUT`. The import maps `STALE` → `OUT` and preserves the stale metadata separately.
- **Dependency ordering**: The importer must handle forward references — `derived-c` depends on `premise-a` and `premise-b`, which appear before it in the document, but `chain-e` depends on `derived-c` which must already exist. The test fixture is ordered correctly, but the parser and importer must handle this.
- **Nogood node validation**: Nogoods referencing non-existent node IDs are skipped, not errored (`test_nogoods_without_valid_nodes_skipped`).
- **Reverse index consistency**: After import, `net.nodes["premise-a"].dependents` must include `"derived-c"`.

## Error Handling

- **No exception tests**: The test suite does not test for raised exceptions — the import pipeline is designed to be lenient (skip bad data, return counts) rather than fail-fast.
- **Missing optional fields**: The parser defaults `depends_on` and `unless` to empty lists, `source_hash` and `stale_reason` to `None`/empty. Tests verify these defaults.
- **Bad nogoods**: Nogoods referencing missing nodes are silently skipped with `nogoods_imported == 0`, not raised as errors.
- **Missing real file**: `TestImportRealRegistry` uses `pytest.skip`, so missing files produce a skip, not a failure.

## Topics to Explore

- [file] `reasons_lib/import_beliefs.py` — The production code under test; see how the markdown heading regex works and how forward references in `Depends on` are resolved
- [function] `reasons_lib/network.py:Network.retract` — The BFS retraction cascade that `test_cascading_works_after_import` exercises post-import
- [file] `reasons_lib/export_markdown.py` — The inverse of import; understanding the round-trip (import → export → re-import) is essential for format evolution
- [file] `tests/test_import_json.py` — The JSON import path; compare with beliefs.md import to understand which format is lossless vs. lossy
- [general] `STALE-to-OUT-mapping` — The semantic gap between the three-state beliefs.md format (IN/STALE/OUT) and the two-state network (IN/OUT), and how metadata bridges it

## Beliefs

- `parse-beliefs-returns-dicts` — `parse_beliefs()` returns a `list[dict]` with keys `id`, `status`, `type`, `text`, `source`, `source_hash`, `depends_on`, `unless`, `stale_reason`, `superseded_by`; missing optional fields default to empty list or None
- `stale-maps-to-out` — The import pipeline maps beliefs.md `STALE` status to network `OUT` truth value, preserving stale metadata (`stale_reason`, `superseded_by`) in the node's metadata dict
- `import-is-idempotent` — Calling `import_into_network` twice with the same document imports zero new claims on the second call and skips all existing ones
- `nogoods-require-valid-nodes` — Nogoods whose `Affects` list references node IDs not present in the network are silently skipped rather than raising an error
- `import-wires-reverse-index` — After import, each dependency target's `dependents` set includes the dependent node, enabling retraction cascades to propagate correctly

