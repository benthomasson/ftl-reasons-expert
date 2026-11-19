---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_import_json.py

## Purpose

`tests/test_import_json.py` is the regression/acceptance suite for the JSON serialization format used by the `reasons` package. It owns one responsibility: **proving that `api.export_network` → JSON → `api.import_json` reproduces the semantic state of a `Network`, not just the raw rows.**

Every test follows the same "round-trip" discipline: build a network in a source SQLite DB, serialize it to a JSON file, initialize a *fresh* destination DB, reimport the JSON, and then assert that the destination behaves identically to the source — including dynamic behaviors (outlist propagation, retraction restoration) that only manifest when the network is re-evaluated after import.

It is a black-box test of the `reasons.api` public surface; it never imports or manipulates `storage.py`/`pg.py` internals directly.

## Key Components

All tests live in the `TestImportJson` class (used purely for grouping; no fixtures or shared state).

| Test | What it protects |
|---|---|
| `test_round_trip_premises` | Node identity, labels, and `source` field survive export/import. Asserts `result["nodes_imported"] == 2`, `get_status` totals, and `show_node("a")["source"] == "repo/a.md"`. |
| `test_round_trip_justifications` | The `Justification` graph (slist → antecedents + label) survives. Asserts `node_b["justifications"][0]["antecedents"] == ["a"]` and `label == "test"`. |
| `test_round_trip_outlist` | `unless="y"` is stored as a justification `outlist` and — crucially — **still propagates dynamically** after reimport (`api.assert_node("y")` flips `x` to `OUT`). |
| `test_round_trip_retracted_state` | `retract_node("a")` cascades to dependent `b`, and asserting `a` afterwards restores `b` to `IN`. Verifies the retracted flag is part of the serialized state, not just a cached truth value. |
| `test_round_trip_nogoods` | `add_nogood(["a","b"])` serializes and rehydrates; asserts `result["nogoods_imported"] == 1`. |
| `test_round_trip_metadata` | Challenge/defense metadata (`api.challenge`, `api.defend`) survives in `node_a["metadata"]["challenges"]`, and the node remains `IN`. |
| `test_round_trip_full_network` | End-to-end test against the real `~/git/physics-pi-meta/beliefs.md` registry (skipped if absent). Compares `total` and `in_count` between source and reimported DB. |
| `test_missing_file_raises` | `FileNotFoundError` is raised for a nonexistent import path. |
| `test_skip_duplicates` | Re-importing the same JSON into a populated DB is idempotent: `nodes_imported == 0`. |

Notable API contracts exercised:

- `api.init_db(db_path=...)` — creates schema on a fresh DB.
- `api.add_node(id, label, source=?, sl=?, label=?, unless=?, db_path=?)` — `sl` supplies a support list (antecedents), `unless` adds an outlist entry.
- `api.export_network(db_path=...)` → JSON-serializable `dict` (must survive `json.dumps`/`json.loads` verbatim).
- `api.import_json(path, db_path=...)` → `dict` with at least `nodes_imported` and `nogoods_imported`.
- `api.get_status(db_path=...)` → at least `total` and `in_count`.
- `api.show_node(id, db_path=...)` → dict with `source`, `truth_value`, `justifications` (list of `{antecedents, label, outlist}`), `metadata`.
- `api.import_beliefs(path, db_path=...)` — markdown-based bulk loader used only in the external-network test.

## Patterns

- **Round-trip / property-test idiom.** Each test treats `export → import` as the identity function on the semantic network. Tests never assert on intermediate JSON structure — they assert on the *post-import* observable state, which leaves `import_json` free to evolve its format (as long as it reimports).
- **Fresh-DB-per-test with `tmp_path`.** No cross-test pollution; both `src.db` and `dst.db` live under pytest's `tmp_path` and are torn down afterwards. The dest DB is always a *new* path so residual schema cannot mask missing-import bugs.
- **Arrange–Act–Assert.** Every method is three blocks separated by blank lines/comments.
- **Skip-if-missing for external data.** `test_round_trip_full_network` uses `pytest.skip` when the `physics-pi-meta` checkout is absent, keeping the suite green on machines without that repo mounted.
- **Behavioral (not structural) assertions.** `test_round_trip_outlist` and `test_round_trip_retracted_state` invoke `api.assert_node` *after* import and assert the truth-value changes, which is the only way to prove the engine re-attached justifications/outlists to the live inference loop rather than merely dumping rows.
- **Unused imports** (`Justification`, `Network`) suggest earlier iterations verified against the object model directly; they are now dead imports but hint at the intended abstraction boundary.

## Dependencies

**This file imports:**

- `json` — for `json.dumps` when writing the export file.
- `pathlib.Path` — for `write_text` on the temp JSON file; `Path.home()` in the external test.
- `pytest` — `pytest.raises(FileNotFoundError)` and `pytest.skip`.
- `reasons.api` — the only production symbol actually used (via `api.*`).
- `reasons.Justification`, `reasons.network.Network` — imported but unused in the current file body.

**What depends on this file:** Nothing — it is a leaf test module. It is collected by pytest from the `tests/` package (note the empty `tests/__init__.py`).

**Production code being tested** (from the repo layout): `reasons/api.py` (`import_json`, `export_network`, `import_beliefs`, `init_db`, `add_node`, `add_nogood`, `retract_node`, `assert_node`, `challenge`, `defend`, `show_node`, `get_status`) and, transitively, `reasons/storage.py`, `reasons/network.py`, `reasons/import_beliefs.py`. The mirror `reasons_lib/` tree is not exercised here.

## Flow

The typical control flow (identical shape for six of the nine tests):

1. `tmp_path` → three paths: `db1` (source SQLite), `db2` (destination SQLite), `json_file`.
2. `api.init_db(db_path=db1)` creates the schema.
3. Build the scenario using `api.add_node` / `api.add_nogood` / `api.challenge` / `api.defend` / `api.retract_node`.
4. `data = api.export_network(db_path=db1)` returns a dict.
5. `Path(json_file).write_text(json.dumps(data))` — serializes to disk. This step **is the contract check for JSON-serializability**; any non-primitive type reachable from the export will raise here.
6. `api.init_db(db_path=db2)` — fresh schema, empty.
7. `api.import_json(json_file, db_path=db2)` — returns a summary dict.
8. Assertions on `result`, `api.get_status`, `api.show_node`, and (in two tests) post-import engine operations.

`test_skip_duplicates` deviates: the same JSON file is imported twice into `db` and the second `import_json` must report `nodes_imported == 0`.

`test_missing_file_raises` is the only test that does not perform a round trip; it just confirms the error path.

Data transformations along the way:

- In-memory `Network` graph → `export_network` dict (nodes, justifications, nogoods, metadata) → `json.dumps` text → file → `import_json` parses → inserts rows → `Network` object lazily hydrated by `show_node` / `get_status` calls → inference engine reevaluates truth values when `assert_node`/`retract_node` fires.

## Invariants

Rules this test file enforces on production code:

1. **JSON round-trip fidelity.** `export_network` output must be `json.dumps`-able without custom encoders for every field stored in a `Network` (source strings, slists, outlists, nogoods, challenge metadata).
2. **Identity is preserved.** Node IDs (`"a"`, `"b"`, …) survive unchanged; `ile`/`source`/`truth_value` are keyable by ID after import.
3. **Justifications are direction-preserving.** A derived node's `antecedents` list equals the exported slist exactly (`== ["a"]`), and the `label` is preserved.
4. **Outlists are live.** The `unless` relationship is not a static annotation — after import the engine must still be capable of *propagating* the outlist (`assert_node("y")` ⇒ `x` becomes `OUT`). This requires the importer to wire justifications into the network, not just insert rows.
5. **Retracted state is serialized.** `retract_node` before export must survive, and proagation-on-behavior must still work: asserting `a` must restore `b` to `IN`. If the importer only copied `truth_value = OUT` as a cached field, `assert_node("a")` would leave `b` stuck OUT.
6. **Import is idempotent per node ID.** Re-importing the same file into the same DB must not create duplicates; `nodes_imported == 0` on the second call.
7. **Fresh-DB isolation.** Importing into a newly initialized DB must yield `status["total"] == status["nodes_imported"]` for the networks built in these tests.
8. **Error on missing input.** `import_json` on a nonexistent path must raise `FileNotFoundError` (not return a partial dict or `None`).

## Error Handling

- **The only explicit error assertion** is `test_missing_file_raises`, which wraps `api.import_json("/nonexistent.json", db_path=db)` in `pytest.raises(FileNotFoundError)`. This fixes the contract: bad paths raise, they do not silently no-op.
- **Implicit failure surfaces** (a regression here would show as a `Test` failure, not an exception):
  - `json.dumps(data)` in every round-trip test fails (with `TypeError`) if `export_network` ever introduces a non-serializable type (e.g. a dataclass, `datetime`, `Path`). This is the *de facto* serializability guard.
  - `api.show_node(...)[...]` will `KeyError` if the importer drops a field (e.g. removes `outlist` from justifications).
- **The duplicate-import path is deliberately not an error**: `test_skip_duplicates` treats reimport as a benign no-op rather than a conflict. `import_json` must **not** raise on existing IDs; it must silently skip and report zero.
- **External-integration test is defensive**: `test_round_trip_full_network` calls `pytest.skip("physics-pi-meta not found")` when the beliefs file is absent, so a missing external checkout is not a test failure — this is the only place external state can perturb the suite.
- No `try/except` inside the tests; failures are expected to propagate as normal pytest failures with assertion diffs.

## Topics to Explore

- [function] `reasons/api.py:import_json` — The implementation being validated; inspect how it translates the parsed JSON into `storage` inserts and how it reports `nodes_imported`/`nogoods_imported`.
- [file] `reasons/import_beliefs.py` — Backs the `api.import_beliefs` call in `test_round_trip_full_network`; shows how markdown is turned into nodes/justifications.
- [file] `tests/test_roundtrip.py` — Likely a sibling round-trip suite (possibly for a different serialization target); comparing it to this file reveals shared conventions and any divergence in what is considered "state".
- [general] `export-format-compatibility` — Investigate whether `export_network` output between the mirror packages `reasons/` and `reasons_lib/` is identical, since the repo tracks both trees.
- [function] `reasons/api.py:retract_node` — Understanding its cascade semantics explains why `test_round_trip_retracted_state` expects `b` to follow `a` and why `assert_node` restores it after import.

## Beliefs

- `import-json-returns-nodes-imported-count` — `api.import_json` returns a dict containing at minimum `nodes_imported` and `nogoods_imported`, both integer counts.
- `import-json-is-idempotent-on-node-id` — A second `import_json` of the same file into a populated DB returns `nodes_imported == 0` and does not raise.
- `import-json-raises-on-missing-file` — `api.import_json` on a nonexistent path raises `FileNotFoundError` rather than returning a partial summary.
- `export-network-is-json-serializable` — `api.export_network` returns a plain `dict` whose every field survives `json.dumps` without a custom encoder.
- `outlist-behavior-requires-live-justifications` — The outlist test only passes if import wires justifications back into the inference engine, not merely into storage; `assert_node("y")` must propagate to `x`.
- `test-suite-imports-from-reasons-not-reasons-lib` — This file imports from `reasons` (not the mirrored `reasons_lib`), so regressions in the `reasons_lib` copy will not be caught here.
