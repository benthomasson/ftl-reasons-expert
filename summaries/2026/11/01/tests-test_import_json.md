---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_import_json.py

# `tests/test_import_json.py`

## Purpose

This file validates the **JSON serialization round-trip** of the entire reasoning network. It ensures that the `api.export_network()` → `api.import_json()` pipeline is lossless: a network's full state (premises, justifications, outlists, retracted/IN/OUT truth values, nogoods, and metadata like challenges/defenses) can be serialized to JSON and deserialized into a completely fresh database with identical behavior.

It owns the responsibility of proving that the JSON format is a **complete state representation** — not just a partial snapshot. If any field is dropped during export or silently ignored during import, one of these tests will catch it.

## Key Components

### `TestImportJson` class

Contains 9 test methods, each targeting a specific aspect of the network state:

| Test | What it verifies |
|------|-----------------|
| `test_round_trip_premises` | Basic node creation, `source` metadata, node count, `in_count` |
| `test_round_trip_justifications` | SL (support links) reconstruct with correct antecedents and labels |
| `test_round_trip_outlist` | `unless` clauses survive export/import **and** remain functionally active (retraction propagation still fires) |
| `test_round_trip_retracted_state` | OUT nodes stay OUT after import; re-asserting a retracted premise reactivates dependents |
| `test_round_trip_nogoods` | Explicit nogood constraints are imported (checked via `result["nogoods_imported"]`) |
| `test_round_trip_metadata` | Challenge/defense metadata is preserved; node remains IN despite challenges |
| `test_round_trip_full_network` | Integration test against a real-world `beliefs.md` registry (skipped if absent) |
| `test_missing_file_raises` | Error contract: importing a nonexistent path raises `FileNotFoundError` |
| `test_skip_duplicates` | Idempotency: re-importing into a DB that already has the nodes results in `nodes_imported == 0` |

### API surface under test

All tests interact exclusively through the `reasons.api` module. Key functions exercised:

- `api.init_db(db_path)` — creates a fresh SQLite (or similar) database
- `api.add_node(id, content, sl=, label=, unless=, source=)` — creates premise or derived nodes
- `api.export_network(db_path)` → `dict` — serializes the entire network
- `api.import_json(path, db_path)` → `dict` with keys `nodes_imported`, `nogoods_imported`
- `api.show_node(id, db_path)` → `dict` with `truth_value`, `justifications`, `source`, `metadata`
- `api.get_status(db_path)` → `dict` with `total`, `in_count`
- `api.retract_node(id, db_path)` / `api.assert_node(id, db_path)` — flip truth values
- `api.add_nogood(list, db_path)` — register an explicit inconsistency
- `api.challenge(id, reason, db_path)` / `api.defend(id, cid, defense, db_path)` — attach metadata

## Patterns

1. **Round-trip / mirror-image testing**: Every core test follows the identical skeleton:
   ```
   init src_db → mutate state → export to JSON → init dst_db → import JSON → assert equivalence
   ```
   This is the canonical pattern for testing serialization fidelity. The `tmp_path` fixture guarantees full isolation between tests (separate `src.db` and `dst.db` per test).

2. **Behavioral (not just structural) verification**: The tests don't merely check that JSON fields are present. For example, `test_round_trip_outlist` goes beyond asserting the outlist exists — it *re-asserts* the counter-node after import and verifies that `x` flips to OUT. Similarly, `test_round_trip_retracted_state` re-asserts the retracted premise and checks the dependent reactivates. This proves the imported state is **functionally equivalent**, not just structurally similar.

3. **Guarded integration test**: `test_round_trip_full_network` uses `pytest.skip` when the external `physics-pi-meta` repo isn't checked out. This keeps CI green in isolated environments while still exercising the full pipeline in developer machines that have the dependency.

4. **Stateless API with explicit DB path**: Every API call takes `db_path` as a parameter. There is no global connection or singleton. This makes the tests trivially parallelizable and avoids cross-test contamination.

## Dependencies

### This file imports (and uses)

| Import | Role |
|--------|------|
| `reasons.api` | The entire test surface — all state mutations and queries go through here |
| `json` | Serializing `export_network()` output to a file |
| `pathlib.Path` | Writing the JSON file, checking existence of `beliefs.md` |
| `pytest` | Test framework, `tmp_path` fixture, `pytest.skip`, `pytest.raises` |
| `reasons.Justification` | **Imported but never referenced** in test bodies — likely a leftover or used for type annotations in a prior revision |
| `reasons.network.Network` | **Imported but never referenced** in test bodies — same note |

### What depends on this file

This is a leaf test file. No other module imports it. It is discovered and executed by `pytest` via the standard `tests/` package (note `tests/__init__.py` exists, making it a package).

## Flow

Each test follows this sequence (using `test_round_trip_outlist` as the representative):

```
1. api.init_db(db1)                          ← create empty source database
2. api.add_node("y", "Counter Y", db1)       ← add a premise
3. api.retract_node("y", db1)                ← put it OUT
4. api.add_node("x", "Default X", unless="y", db1)  ← add default that depends on y being OUT
   ── at this point: x is IN, y is OUT ──

5. data = api.export_network(db1)            ← serialize full state to Python dict
6. Path(json_file).write_text(json.dumps(data))  ← persist to JSON file

7. api.init_db(db2)                          ← create empty destination database
8. api.import_json(json_file, db2)           ← deserialize and rebuild

9. node_x = api.show_node("x", db2)          ← verify structural fidelity
   assert node_x["truth_value"] == "IN"
   assert node_x["justifications"][0]["outlist"] == ["y"]

10. api.assert_node("y", db2)                ← behavioral check: reactivate counter
11. node_x = api.show_node("x", db2)         ← x must now be OUT
    assert node_x["truth_value"] == "OUT"
```

The critical data transformation is **step 5→8**: the in-memory/DB state becomes a JSON document, then is re-instantiated in a different database. The tests assert both structural equivalence (step 9) and behavioral equivalence (steps 10-11).

## Invariants

- **Lossless round-trip**: `import(export(S))` produces a state S′ where `status(S′) == status(S)` for all observable properties (node count, IN count, individual truth values, justification structure, outlists, nogoods, metadata).
- **Functional equivalence**: The imported network doesn't just *look* correct — defeasible propagation still works. Retracting/re-asserting nodes after import triggers the same cascade as in the original.
- **Idempotent import**: Importing a JSON file into a database that already contains those node IDs is a no-op (`nodes_imported == 0`), not an error and not a duplicate.
- **Explicit failure for bad input**: A nonexistent file path raises `FileNotFoundError` (not a silent empty import).
- **`db_path` is the state container**: All state lives in the database file identified by `db_path`. The API layer holds no in-memory state between calls.

## Error Handling

The file exercises exactly two error paths:

1. **`test_missing_file_raises`** — asserts that `api.import_json("/nonexistent.json", db)` raises `FileNotFoundError`. This is the contract: the import function does *not* silently return an empty result for a missing file. The test uses `pytest.raises` to confirm the exception type.

2. **`test_round_trip_full_network`** — uses `pytest.skip()` when the external `physics-pi-meta/beliefs.md` file is absent. This is not an error path per se, but a graceful degradation that prevents a hard failure in environments without that optional dependency.

There is no error handling *within* the test code itself beyond these. The tests assume the `api` module raises or returns expected values; they do not try to catch unexpected exceptions.

---

## Topics to Explore

- [function] `reasons/api.py:import_json` — The function under test; understanding its internal logic (how it reconstructs nodes, justifications, nogoods, and metadata from the JSON dict) is essential for debugging round-trip failures
- [function] `reasons/api.py:export_network` — The other half of the round-trip; its output schema (the dict structure) defines the JSON contract that `import_json` consumes. Reading both together reveals which fields are guaranteed to round-trip
- [file] `reasons/storage.py` — The persistence layer beneath the API; understanding how nodes, justifications, and nogoods are stored (and how `init_db` creates the schema) explains what "full state" means at the storage level
- [file] `tests/test_roundtrip.py` — Likely tests a different round-trip path (e.g., `export_markdown` → `import_beliefs`); comparing it with this file clarifies the distinction between JSON and Markdown serialization fidelity
- [general] `defeasible-logic-propagation` — The outlist/SL semantics (how `unless` and `sl` interact during retraction cascades) are the core domain logic that makes the behavioral assertions in `test_round_trip_outlist` and `test_round_trip_retracted_state` meaningful

---

## Beliefs

- `import-json-preserves-outlist-functionality` — After `import_json`, outlist (`unless`) constraints remain active: asserting a counter-node on the imported DB causes the default node to flip from IN to OUT, identical to the pre-export behavior.
- `import-json-skips-existing-nodes` — Calling `import_json` on a database that already contains a node with the same ID results in `nodes_imported == 0` for that node (no duplicate, no error); the import is idempotent per-node.
- `import-json-raises-on-missing-file` — `api.import_json` raises `FileNotFoundError` (not a `ValueError` or silent empty import) when the JSON file path does not exist on disk.
- `test-file-uses-stateless-db-path-api` — Every `api.*` call in this file passes an explicit `db_path` string; there is no module-level database handle, making each test fully isolated via `tmp_path`.
- `justification-and-network-imports-are-unused` — The top-level imports of `reasons.Justification` and `reasons.network.Network` are never referenced in any test method; they are vestigial or exist for IDE type-resolution only.
