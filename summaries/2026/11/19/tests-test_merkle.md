---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_merkle.py

## Purpose

`tests/test_merkle.py` is the integration test suite for the project’s Merkle-style integrity hashing over the belief network. It verifies that:

- nodes receive a deterministic `text_hash` when created;
- justifications receive a non-empty `content_hash`;
- justification hashes bind to antecedent state, so changes propagate through dependents;
- `api.check_integrity()` detects both local text tampering and stale dependency-chain hashes;
- missing hashes can be backfilled for legacy/raw data;
- hashes survive export/import;
- superseding and adding justifications produce correctly hashed data;
- integrity verification does not infinite-loop on malformed cyclic antecedents.

This file is not production code and is not imported by application modules. It is collected and run by pytest, and it exercises `reasons.api`, `reasons.merkle`, and low-level `reasons.storage` behavior.

## Key Components

### Test cases

- `test_text_hash_set_on_creation`
  - Creates a node through `api.add_node`.
  - Reads it back with `api.show_node`.
  - Asserts `node["text_hash"] == hashlib.sha256(b"Safety is established").hexdigest()`.
  - Contract: API-created nodes have a SHA-256 text hash.

- `test_content_hash_set_on_creation`
  - Adds `p1`, then `d1` with `sl="p1"`.
  - Asserts the created justification has a non-empty `content_hash`.
  - Contract: justification hashes are populated during normal creation.

- `test_content_hash_incorporates_antecedents`
  - Creates `d1` citing `p1,p2`.
  - Mutates `p1.text` and updates `p1.text_hash` directly via `Storage`.
  - Calls `api.check_integrity`.
  - Asserts `d1` appears in `chain_mutations`.
  - Contract: a justification’s content hash depends on antecedent state, not just the dependent node’s own text.

- `test_merkle_transitive`
  - Builds `c <- b <- a`: `b` cites `c`, `a` cites `b`.
  - Mutates `c.text` and updates `c.text_hash`.
  - Asserts both `b` and `a` appear in `chain_mutations`.
  - Contract: hash invalidation propagates transitively through the dependency graph.

- `test_verify_clean`
  - Builds a small clean graph and checks integrity.
  - Asserts zero `text_mutations`, zero `chain_mutations`, and `missing_hashes == 0`.
  - Contract: a correctly hashed network verifies cleanly.

- `test_verify_text_mutation`
  - Directly mutates `p1.text` through `_with_network(..., write=True)` without updating `text_hash`.
  - Asserts exactly one `text_mutation` for `p1`.
  - Contract: `check_integrity` recomputes the text hash from text and compares it to stored `text_hash`.

- `test_verify_chain_mutation`
  - Mutates `p1.text` and updates `p1.text_hash` consistently.
  - Asserts no `text_mutations`, but one `chain_mutation` for `d1`.
  - Contract: even if a node’s own text hash is valid, dependents with stale justification hashes are detected.

- `test_backfill_hashes`
  - Strips `text_hash` from two nodes and `content_hash` from one justification.
  - `check_integrity` reports missing hashes.
  - `api.backfill_hashes` reports `nodes_updated == 2` and `justifications_updated == 1`.
  - After backfill, integrity is clean.
  - Contract: backfill repairs legacy/raw rows without leaving integrity errors.

- `test_export_import_preserves_hashes`
  - Exports a graph via `api.export_network`.
  - Asserts exported node and justification hashes are present.
  - Re-imports via JSON using `api.import_json` into a fresh DB.
  - Asserts the imported DB verifies cleanly.
  - Contract: serialization preserves hash fields and integrity.

- `test_cycle_safety`
  - Artificially appends a justification to `a` pointing at `b`, creating a cycle `a -> b -> a`.
  - Calls `api.check_integrity` and only asserts that a dict is returned.
  - Contract: integrity verification must terminate on cyclic antecedent graphs.

- `test_supersede_preserves_hashes`
  - Calls `api.supersede_with_text`.
  - Asserts the successor’s `text_hash` equals SHA-256 of the new text.
  - Asserts integrity is clean afterward.
  - Contract: superseding creates properly hashed successor state.

- `test_multiple_justifications_each_hashed`
  - Creates a node with one justification, then adds another with `api.add_justification`.
  - Asserts both justifications have non-empty and distinct `content_hash` values.
  - Contract: every justification is individually hashed and different antecedents produce different hashes.

### Imported production symbols

- `compute_text_hash`
  - Directly used to recompute the expected `text_hash` after simulated tampering.
  - The tests treat it as deterministic SHA-256 over text.

- `compute_merkle_hash`, `compute_content_hash`, `verify_all`, `backfill_hashes`
  - Imported but not directly called in most tests.
  - They are exercised indirectly through `reasons.api`.

- `Storage`
  - Used for low-level `load()`/`save()` mutation that bypasses normal API hash maintenance.
  - This simulates raw DB edits or bugs.

- `api._with_network(..., write=True)`
  - Private context manager used to mutate the loaded network directly in tests.
  - Also used to append a malformed `Justification` in the cycle-safety test.

- `Justification`
  - Imported from `reasons` for constructing an artificial cyclic justification.

## Patterns

- Integration-first testing: most tests use `api.*` public entry points rather than isolated unit tests of `merkle.py`.
- Arrange–act–assert structure with `tmp_path`-isolated SQLite DBs.
- Explicit tampering simulations:
  - mutate `text` without updating `text_hash` to test local text mutation;
  - mutate `text` and update `text_hash` to test dependency-chain mutation;
  - strip hashes to test backfill.
- Low-level white-box mutation through `Storage.load()`/`save()` and `_with_network()` to simulate unauthorized DB changes.
- Compatibility/regression testing for old data via `test_backfill_hashes`.
- Serialization round-trip testing via export/import.
- Cycle-safety test is defensive but minimal: it only checks termination, not exact output shape beyond being a dict.

The `pytest` import is present but unused in this file; there are no `pytest.raises`, fixtures, parameterization, or marks in the test module itself.

## Dependencies

### Imports used by this test file

- `hashlib` — constructs expected SHA-256 text hashes.
- `pytest` — imported but not used directly.
- `reasons.api` — primary public API under test:
  - `init_db`, `add_node`, `show_node`, `check_integrity`, `backfill_hashes`,
    `export_network`, `import_json`, `supersede_with_text`, `add_justification`,
    `_with_network`.
- `reasons.merkle` — hash and verification helpers:
  - `compute_text_hash`, `compute_merkle_hash`, `compute_content_hash`,
    `verify_all`, `backfill_hashes`.
- `reasons.storage.Storage` — low-level load/save used to simulate tampering.
- `reasons.Justification` — used to create artificial cycle state.
- `json` — serializes export payload for re-import.

### What depends on this file

- pytest test discovery/runs this module.
- No production module imports this test file.
- Coverage for `reasons/merkle.py`, `reasons/api.py` integrity methods, and parts of `reasons/storage.py` depends on this file.

## Flow

A typical test follows this sequence:

1. Create a temporary SQLite path with `tmp_path / "test.db"`.
2. Initialize the database with `api.init_db(str(db))`.
3. Create nodes/justifications through `api.add_node` or `api.add_justification`.
4. Optionally simulate corruption:
   - mutate node text via `_with_network`;
   - mutate `Storage.load()` result and save it back;
   - strip hashes;
   - append a cyclic justification.
5. Call `api.check_integrity(db_path=...)`.
6. Inspect result fields:
   - `text_mutations`
   - `chain_mutations`
   - `missing_hashes`
7. Optionally call `api.backfill_hashes`.
8. Assert expected mutation lists/counts and clean state.

The key data transformation is:

- `text_hash` is computed from node text.
- `content_hash` is computed for each justification from justification metadata and antecedent Merkle state.
- Changing an antecedent changes its Merkle state.
- Dependent justifications’ stored `content_hash` values become stale.
- `check_integrity` reports those stale justifications as `chain_mutations`.
- Transitive chains are checked deeply enough that `c <- b <- a` flags both `b` and `a`.

## Invariants

- Every node created through the normal API has `text_hash == SHA-256(node.text)`.
- Every justification created through the normal API has a non-empty `content_hash`.
- Different antecedent sets produce different justification hashes.
- A justification’s `content_hash` incorporates antecedent state, so changing an antecedent can invalidate dependents.
- A clean network must report:
  - `len(text_mutations) == 0`
  - `len(chain_mutations) == 0`
  - `missing_hashes == 0`
- A raw text mutation that does not update `text_hash` must be reported as a `text_mutation`.
- A text mutation that does update `text_hash` must still be reported as a `chain_mutation` for dependents.
- Missing hashes must be repairable with `api.backfill_hashes`.
- Export/import must preserve hash fields sufficiently for integrity verification to pass.
- Superseding a node must produce a successor with a correct new `text_hash`.
- Integrity verification must terminate on cyclic antecedent graphs.

## Error Handling

- The tests do not use `pytest.raises`; they assert on returned values instead.
- `api.check_integrity` is expected to return a dict rather than raise for the tested corruption cases.
- The cycle-safety test specifically checks that verification returns a dict and does not hang.
- Assertion failures are the primary error signal in this test file.
- Low-level tampering is performed deliberately without normal validation, so any exceptions from `Storage` or `_with_network` would fail the test directly.
- The tests do not assert exact error messages or exception types because integrity failures are represented as data in the returned dict, not exceptions.

## Topics to Explore

- [file] `reasons/merkle.py` — Defines `compute_text_hash`, `compute_content_hash`, `compute_merkle_hash`, `verify_all`, and `backfill_hashes`; this is where cycle-safe recursive hashing and propagation semantics live.
- [function] `reasons/api.py:check_integrity` — Maps verifier results into the `text_mutations`, `chain_mutations`, and `missing_hashes` contract used throughout these tests.
- [function] `reasons/api.py:_with_network` — Write-enabled context manager that tests use to simulate raw database mutations and malformed network state.
- [file] `reasons/storage.py` — `Storage.load()`/`save()` behavior and transaction semantics that allow tests to bypass API-level hash maintenance.
- [general] `merkle-cycle-detection` — How the verifier prevents infinite recursion on malformed antecedent graphs such as the artificial `a -> b -> a` cycle.

## Beliefs

- `text-hash-is-sha256-hexdigest` — Node `text_hash` is the SHA-256 hexdigest of the node text, as asserted directly against `hashlib.sha256(...).hexdigest()`.
- `justification-content-hash-incorporates-antecedents` — A justification’s `content_hash` changes when an antecedent node’s Merkle/text state changes, causing dependent nodes to appear in `chain_mutations`.
- `check-integrity-returns-result-dict-not-raise` — For the corruption and cycle cases covered here, `api.check_integrity()` returns a dict and tests assert on `text_mutations`, `chain_mutations`, and `missing_hashes` rather than expecting exceptions.
- `backfill-requires-missing-hashes` — `api.backfill_hashes()` fills empty `text_hash`/`content_hash` fields and reports `nodes_updated` and `justifications_updated`; afterward `check_integrity` reports zero missing hashes.
- `export-import-preserves-hashes` — `api.export_network()` includes existing hashes, and `api.import_json()` restores them so the imported database verifies with no text or chain mutations.
