---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_nogood_id.py

# `tests/test_nogood_id.py`

## Purpose

This file validates the **content-based nogood ID generation** system. A "nogood" in this reasoning framework represents a set of node IDs that cannot all be true simultaneously (a contradiction set). The ID scheme encodes the nogood's *content* (which nodes it contradicts) directly into the ID string, making IDs deterministic, human-readable, and stable across sessions.

This test file owns the contract that:
- IDs are derived from the **sorted** list of node IDs
- Duplicate nogoods (same node set) receive a numeric suffix
- IDs survive persistence (save/load round-trip)
- Legacy IDs (e.g., `nogood-005`) imported from JSON or beliefs text are preserved as-is, while *new* nogoods created via the API still get the content-based format

## Key Components

### `TestContentBasedIds`
Tests the core ID derivation logic against an in-memory `Network` object:

| Test | Contract |
|------|----------|
| `test_id_from_sorted_nodes` | `add_nogood(["b","a"])` → `"nogood-a-b"` (input order doesn't matter; IDs are sorted) |
| `test_different_nodes_different_ids` | Distinct node sets produce distinct IDs |
| `test_duplicate_gets_suffix` | Second call with same nodes → `"nogood-a-b-2"` |
| `test_three_node_nogood` | Works for N>2 nodes: `["z","x","y"]` → `"nogood-x-y-z"` |

### `TestPersistence`
| Test | Contract |
|------|----------|
| `test_id_survives_save_load` | After `Storage.save(net)` → `Storage.load()`, the nogood retains its `"nogood-a-b"` ID |

### `TestImportJson`
| Test | Contract |
|------|----------|
| `test_import_preserves_old_format_ids` | A JSON file containing `"id": "nogood-005"` is imported verbatim. A *subsequent* `api.add_nogood(["a","b"])` still produces `"nogood-a-b"` (the new scheme) rather than colliding with the legacy ID. |

### `TestImportBeliefs`
| Test | Contract |
|------|----------|
| `test_import_nogoods_preserves_old_format_ids` | Markdown-formatted nogoods (e.g., `### nogood-010: a, b`) are imported via `import_into_network`. Again, a fresh `api.add_nogood(["a","b"])` yields `"nogood-a-b"`. |

### `db` fixture
Initializes a fresh SQLite database at a temp path via `api.init_db(db_path=...)`. Used by the import tests that go through the full API layer.

## Patterns

- **Arrange-Act-Assert in isolation**: Each test builds a minimal `Network` with exactly the nodes needed, adds nogoods, and asserts on the `net.nogoods[i].id` field. No shared mutable state between tests.
- **In-memory vs. disk-backed testing**: The `TestContentBasedIds` class exercises the `Network` object directly (fast, no I/O). `TestPersistence` and the import classes go through `Storage` (SQLite) to verify serialization fidelity.
- **Legacy compatibility tests**: The import tests specifically guard against a migration regression — old data with sequential IDs (`nogood-005`, `nogood-010`) must coexist with new content-based IDs without collision.
- **Fixture-based temp isolation**: The `db` fixture and `tmp_path` (pytest built-in) ensure no cross-test contamination.

## Dependencies

| Direction | Module | Usage |
|-----------|--------|-------|
| imports | `reasons.api` | `init_db`, `add_node`, `add_nogood`, `import_json` — the high-level API layer |
| imports | `reasons.Nogood` | Data class (imported but not directly instantiated in tests) |
| imports | `reasons.network.Network` | In-memory graph; provides `add_node`, `add_nogood`, `.nogoods` list |
| imports | `reasons.storage.Storage` | SQLite persistence; `save`, `load`, `close` |
| imports | `reasons.import_beliefs.import_into_network` | Parses markdown nogood text into a `Network` |
| imports | `pytest` | Fixtures (`tmp_path`), test runner |

Nothing in the repository imports this file (it's a leaf test module).

## Flow

1. **Content-based ID tests** (`TestContentBasedIds`):
   - Construct `Network()` → call `add_node(id, label)` for each required node → call `add_nogood([node_ids])` → assert on `net.nogoods[0].id`.
   - The `Network.add_nogood` method (in `reasons/network.py`) internally sorts the node IDs, formats `"nogood-{a}-{b}-..."`, checks for existing duplicates, and appends `-2`, `-3`, etc. if needed.

2. **Persistence test** (`TestPersistence`):
   - Build network → `Storage(db_path).save(net)` → `Storage(db_path).load()` → assert ID is unchanged. This verifies the SQLite schema stores the `id` column verbatim and the `Nogood` deserialization reconstructs it.

3. **JSON import test** (`TestImportJson`):
   - Seed DB with nodes via API → write a JSON file with a legacy nogood → `api.import_json(...)` merges it → call `api.add_nogood(["a","b"])` → assert the *new* nogood gets the content-based ID (not a collision with `nogood-005`).

4. **Beliefs import test** (`TestImportBeliefs`):
   - Same shape but uses `import_into_network(net, beliefs_text, nogoods_text)` to parse markdown, then `storage.save(net)` to persist, then `api.add_nogood` to verify new ID generation.

## Invariants

- **Determinism**: Given the same set of node IDs, `add_nogood` always produces the same base ID regardless of input order. Sorting is the mechanism.
- **Uniqueness**: The full ID (base + optional suffix) is unique within a network. The `-2`, `-3` suffix scheme enforces this.
- **Legacy coexistence**: Imported IDs of any format (sequential, content-based, custom) are stored as-is. The content-based generator must not collide with them. This implies the ID space is effectively a free-form string namespace, with the content-based scheme being a *convention* for new nogoods.
- **Persistence fidelity**: The `id` field is an opaque string in storage; save/load must be a lossless round-trip for this field.

## Error Handling

This test file doesn't exercise error paths directly. It assumes:
- `Network.add_node` raises (or logs) if a duplicate node ID is added — tests use unique IDs so this isn't triggered.
- `Storage.save` / `Storage.load` raise on I/O failures — covered by other test files (e.g., `test_storage.py`).
- `api.import_json` handles malformed JSON — covered elsewhere.

The only implicit error-relevant behavior tested is the **suffix assignment** in `test_duplicate_gets_suffix`: if the duplicate-detection logic were broken, the second nogood would either overwrite the first or produce a non-unique ID, and the assertion would fail.

---

## Topics to Explore

- [function] `reasons/network.py:Network.add_nogood` — The actual ID generation, sorting, and suffix logic that these tests are pinning down. Understanding the exact algorithm (how it detects "same content") is essential before modifying the scheme.
- [file] `reasons/import_json.py` — How legacy JSON nogoods are merged into the network. The import tests here only verify the *outcome* (new IDs don't collide); reading the import code reveals how it decides whether to skip, deduplicate, or append.
- [file] `reasons/import_beliefs.py` — The markdown parser (`import_into_network`) that turns `### nogood-010: a, b` lines into `Nogood` objects. Useful if you're extending the text format.
- [general] `nogood-id-migration` — The transition from sequential IDs (`nogood-001`) to content-based IDs (`nogood-a-b`) is a one-way migration. Understanding whether a migration script exists (or whether old IDs are simply grandfathered in) affects any future schema changes.
- [file] `reasons/storage.py` — The SQLite schema and serialization for `Nogood`. If you change the ID format, you need to know whether the DB has an index or uniqueness constraint on the `id` column.

---

## Beliefs

- `nogood-id-is-deterministic-from-node-set` — `Network.add_nogood(["b","a"])` and `Network.add_nogood(["a","b"])` always produce the same base ID `"nogood-a-b"`, because the implementation sorts node IDs before formatting.
- `duplicate-nogood-receives-incremental-suffix` — A second `add_nogood` call with the same node set appends `-2` (then `-3`, etc.) to the base ID rather than overwriting or rejecting the entry.
- `import-preserved-ids-coexist-with-content-based-ids` — Legacy nogood IDs (e.g., `"nogood-005"`) imported via JSON or beliefs text do not collide with newly generated content-based IDs (`"nogood-a-b"`); the ID namespace treats them as distinct strings.
- `nogood-id-survives-storage-roundtrip` — The `Nogood.id` field is stored as an opaque string in SQLite and is byte-identical after a `Storage.save` → `Storage.load` cycle.
- `test-file-covers-three-layers` — This single test file exercises the ID logic at the `Network` (in-memory), `Storage` (SQLite), and `api`/import (persistence + integration) layers, meaning a regression in any layer's ID handling will be caught here.
