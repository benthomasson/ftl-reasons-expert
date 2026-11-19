---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_nogood_id.py

## Purpose

This file is a pytest test module that defines the expected behavior for **content-based nogood IDs**. A “nogood” is identified by the sorted IDs of the nodes it contradicts, producing deterministic IDs such as `nogood-a-b`; duplicates of the same generated ID receive a numeric suffix like `nogood-a-b-2`.

The file is effectively an executable specification for three areas:

1. In-memory ID generation via `Network.add_nogood`.
2. Persistence round-tripping via `Storage.save` / `Storage.load`.
3. Backward-compatible import behavior from JSON and Markdown sources, followed by new content-based ID generation through `api.add_nogood`.

It does not implement production behavior; it pins down contracts that the `reasons` package must satisfy.

## Key Components

### Fixture: `db`

```python
@pytest.fixture
def db(tmp_path):
    db_path = str(tmp_path / "reasons.db")
    api.init_db(db_path=db_path)
    return db_path
```

- Uses pytest’s `tmp_path` to create an isolated temporary directory per test.
- Builds a SQLite database path.
- Calls `api.init_db(db_path=db_path)` to initialize the database.
- Returns the DB path string for tests that need `db_path=db`.
- The fixture does not explicitly close or clean up the DB; pytest’s `tmp_path` handles filesystem isolation.

### `TestContentBasedIds`

Tests `Network.add_nogood` directly, without storage.

- `test_id_from_sorted_nodes`
  - Adds nodes `"b"` and `"a"`.
  - Calls `net.add_nogood(["b", "a"])`.
  - Asserts the generated ID is `"nogood-a-b"`.
  - This proves input order is normalized by sorting.

- `test_different_nodes_different_ids`
  - Adds `a`, `b`, `c`, `d`.
  - Adds nogoods for `["a","b"]` and `["c","d"]`.
  - Asserts IDs are `"nogood-a-b"` and `"nogood-c-d"`.
  - This proves different node sets produce different content-derived IDs.

- `test_duplicate_gets_suffix`
  - Adds the same nogood `["a","b"]` twice.
  - Asserts the first ID is `"nogood-a-b"` and the second is `"nogood-a-b-2"`.
  - This establishes the duplicate suffix contract.

- `test_three_node_nogood`
  - Adds nodes `x`, `y`, `z`.
  - Calls `net.add_nogood(["z", "x", "y"])`.
  - Asserts `"nogood-x-y-z"`.
  - Confirms multi-node sorting and ID formatting.

### `TestPersistence`

- `test_id_survives_save_load`
  - Creates a `Storage` instance against a temporary DB path.
  - Creates a `Network`, adds nodes `a` and `b`, and adds nogood `["a","b"]`.
  - Saves the network, closes storage, reopens a new `Storage`, and loads the network.
  - Asserts the loaded nogood still has ID `"nogood-a-b"`.
  - This proves IDs are persisted as data, not recomputed differently on load.

### `TestImportJson`

- `test_import_preserves_old_format_ids`
  - Initializes DB via the `db` fixture.
  - Adds nodes `a` and `b` through `api.add_node`.
  - Writes a JSON payload containing a legacy nogood:

```json
{"id": "nogood-005", "nodes": ["a", "b"], "discovered": "", "resolution": ""}
```

  - Calls `api.import_json(json_file, db_path=db)`.
  - Then calls `api.add_nogood(["a", "b"], db_path=db)`.
  - Asserts the returned `result["nogood_id"] == "nogood-a-b"`.

  The important contract is that legacy IDs such as `"nogood-005"` are preserved/imported in the old namespace, while subsequent new nogoods use content-based IDs. In particular, the legacy ID for nodes `a,b` does not cause the new addition to become `"nogood-a-b-2"`.

### `TestImportBeliefs`

- `test_import_nogoods_preserves_old_format_ids`
  - Initializes DB via `db`.
  - Adds nodes `a` and `b` through `api.add_node`.
  - Passes empty belief text and Markdown nogood text:

```markdown
# Nogoods

### nogood-010: a, b
- Affects: a, b
- Discovered: 2026-01-01
```

  - Loads the network through `Storage(db)`, calls `import_into_network(net, beliefs_text, nogoods_text)`, saves, and closes storage.
  - Calls `api.add_nogood(["a", "b"], db_path=db)`.
  - Asserts the new nogood ID is `"nogood-a-b"`.

  This is the Markdown-import equivalent of the JSON import test. It verifies that old-style `nogood-NNN` IDs do not interfere with content-based ID generation for newly added nogoods.

## Patterns

- **Behavioral test grouping**: Test classes group related contracts by concern: ID generation, persistence, JSON import, and Markdown import.
- **Arrange-Act-Assert**: Each test sets up nodes/data, performs one operation, and asserts on the resulting ID.
- **Layered testing**: Some tests exercise the lower-level `Network` directly; others exercise the public `api` and `Storage` layers.
- **Round-trip testing**: `test_id_survives_save_load` verifies save/load persistence.
- **Compatibility testing**: Import tests verify that old-format IDs like `nogood-005` and `nogood-010` coexist with new content-derived IDs.
- **Fixture reuse**: The `db` fixture centralizes temporary database initialization.
- **Black-box assertions**: Tests assert on public attributes such as `net.nogoods[0].id` and API return dictionaries, not internal helper methods.

## Dependencies

This module imports:

- `json` for writing the JSON import fixture.
- `pytest` for fixtures and test discovery.
- `from reasons import api, Nogood`
  - `api` is used for `init_db`, `add_node`, `import_json`, and `add_nogood`.
  - `Nogood` is imported but not referenced in this file. It appears unused.
- `from reasons.network import Network`
  - Used directly for in-memory ID-generation tests.
- `from reasons.storage import Storage`
  - Used directly for persistence and import-beliefs tests.
- `from reasons.import_beliefs import import_into_network`
  - Used inside `test_import_nogoods_preserves_old_format_ids`.

Production code does not depend on this file. It is discovered by pytest and likely exercised by CI workflows.

## Flow

The test execution flow is straightforward:

1. Pytest discovers test classes and methods.
2. For tests requiring `db`, the fixture creates a fresh temporary SQLite DB via `api.init_db`.
3. ID-generation tests create an in-memory `Network`, add nodes, call `add_nogood`, and inspect `net.nogoods`.
4. Persistence tests create a `Storage`, save a `Network`, close it, reopen storage, load the network, and inspect `loaded.nogoods[0].id`.
5. JSON import tests:
   - Add prerequisite nodes through `api.add_node`.
   - Write a temporary JSON file.
   - Call `api.import_json`.
   - Call `api.add_nogood` for the same node set.
   - Assert that the new addition receives the content-derived base ID.
6. Markdown import tests:
   - Add prerequisite nodes.
   - Load a network from storage.
   - Mutate it with `import_into_network`.
   - Save it back.
   - Call `api.add_nogood` and assert on its returned `nogood_id`.

No test validates malformed input or expected exceptions.

## Invariants

These are the rules enforced by the tests:

- Nogood IDs are content-derived from sorted node IDs.
- Input node order is irrelevant: `["b","a"]` becomes `"nogood-a-b"`.
- Multi-node IDs are joined with hyphens after lexicographic sorting: `["z","x","y"]` becomes `"nogood-x-y-z"`.
- Different node sets produce different base IDs.
- Re-adding the same generated content ID produces a numeric suffix starting at `-2`.
- `Network.nogoods` preserves insertion order, as tests access `net.nogoods[0]` and `net.nogoods[1]`.
- `Nogood.id` survives a `Storage.save` / `Storage.load` round trip.
- Legacy imported IDs such as `"nogood-005"` and `"nogood-010"` remain in the old ID format and do not prevent a subsequent `api.add_nogood(["a","b"])` from returning `"nogood-a-b"`.
- `api.add_nogood` returns a dictionary containing the key `"nogood_id"`.

## Error Handling

There is no explicit error handling in this test file.

- No `pytest.raises` assertions are present.
- Failures occur naturally as assertion failures or uncaught exceptions from `reasons` APIs.
- The tests assume valid node IDs, valid JSON structure, valid Markdown nogood syntax, and successful database initialization.
- If ID generation changes incorrectly, import compatibility breaks, or persistence loses IDs, these tests fail directly.

## Topics to Explore

- [function] `reasons/network.py:Network.add_nogood` — Implements the sorting, ID formatting, duplicate detection, and suffix generation asserted by `TestContentBasedIds`.
- [function] `reasons/api.py:import_json` — Handles legacy JSON nogood import and determines how old `nogood-NNN` IDs coexist with content-based IDs.
- [function] `reasons/import_beliefs.py:import_into_network` — Implements Markdown nogood import exercised by `TestImportBeliefs`.
- [file] `reasons/storage.py` — Owns persistence of `Nogood.id` and the save/load round-trip behavior.
- [general] `legacy-nogood-id-compatibility` — Explore how old-format IDs are stored, displayed, and deduplicated alongside content-derived IDs.

## Beliefs

- `nogood-ids-are-content-derived` — `Network.add_nogood` assigns IDs of the form `nogood-<sorted-node-ids>`.
- `nogood-id-node-order-is-normalized` — Supplying nodes as `["b", "a"]` produces the same ID as `["a", "b"]`.
- `duplicate-content-based-ids-get-numeric-suffix` — Adding the same generated nogood twice yields `nogood-a-b` followed by `nogood-a-b-2`.
- `nogood-id-survives-storage-round-trip` — `Storage.save` followed by `Storage.load` preserves a nogood’s content-derived ID.
- `legacy-import-ids-do-not-block-content-ids` — After importing legacy `nogood-005` or `nogood-010` entries for nodes `a,b`, `api.add_nogood(["a","b"])` still returns `nogood-a-b`.
