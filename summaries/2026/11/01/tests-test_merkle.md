---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_merkle.py

# `tests/test_merkle.py`

## Purpose

This file is the test suite for the **Merkle tree integrity layer** of the `ftl-reasons` belief network. Its job is to verify that the content-addressing and tamper-detection mechanism works correctly across the full lifecycle of a belief network: node creation, justification linking, text mutation, transitive chain invalidation, hash backfilling, export/import round-trips, and edge cases like cycles.

The system under test (`reasons/merkle.py`) implements a layered hash scheme:
- **`text_hash`** — SHA-256 of a node's own text
- **`content_hash`** — a Merkle-like hash on each justification that folds in the text hashes of all antecedent nodes
- **Transitive propagation** — if an antecedent's text changes, every downstream justification that (transitively) depends on it should be flagged as stale

This test file ensures none of those invariants silently break.

## Key Components

The file has no classes of its own. It defines **12 test functions**, each a standalone `pytest` function taking `tmp_path` (the standard pytest fixture for an isolated temp directory). They exercise the following API surface:

| Function Under Test | Contract |
|---|---|
| `compute_text_hash(text)` → `str` | Returns SHA-256 hex of the raw text bytes |
| `compute_content_hash(...)` → `str` | Returns a hash incorporating the justification's own data *and* its antecedents' text hashes |
| `compute_merkle_hash(...)` → `str` | (Imported but used indirectly via content hash) |
| `verify_all(net)` → `dict` | Walks the graph, returns `{"text_mutations": [...], "chain_mutations": [...], "missing_hashes": int}` |
| `backfill_hashes(net)` → `dict` | Fills in empty `text_hash` / `content_hash` fields; returns `{"nodes_updated": int, "justifications_updated": int}` |

The tests also exercise **public API wrappers** in `reasons/api.py`:
- `api.check_integrity(db_path)` — thin wrapper around `verify_all`
- `api.backfill_hashes(db_path)` — thin wrapper around `backfill_hashes`
- `api.add_node`, `api.show_node`, `api.add_justification`, `api.supersede_with_text`, `api.export_network`, `api.import_json`

## Patterns

1. **Tamper simulation via `Storage` layer bypass.** Several tests (`test_content_hash_incorporates_antecedents`, `test_merkle_transitive`, `test_verify_chain_mutation`) deliberately bypass the high-level API and mutate `net.nodes[...].text` directly through `Storage.load()` / `Storage.save()`. This simulates the "adversary or bug edited the DB" scenario — exactly what the Merkle layer is designed to catch. The key trick: they also update `text_hash` to match the new text, so the *text* check passes but the *chain* check (which compares the justification's stored `content_hash` against a recomputed one) catches the staleness.

2. **Two-flavor mutation detection.** The tests distinguish between:
   - **Text mutations** (`text_mutations`): node's `text` no longer matches its stored `text_hash`. Caught by comparing `sha256(text)` to the stored hash.
   - **Chain mutations** (`chain_mutations`): text and `text_hash` agree, but a justification's `content_hash` is stale because an antecedent's hash changed. This is the Merkle-tree-specific check.

3. **`_with_network` context manager for raw writes.** Tests that need to mutate internal state without going through the API use `from reasons.api import _with_network` with `write=True`. This is a semi-private helper that acquires a write lock on the DB and yields the network object.

4. **Cycle-safety as a non-hanging guarantee.** `test_cycle_safety` explicitly constructs a cycle (a → b → a) and asserts the function *returns* (i.e., doesn't infinite-loop). It doesn't assert correctness of the result — just termination.

5. **`tmp_path` isolation.** Every test gets its own SQLite DB. No shared state, no ordering dependencies.

## Dependencies

**This file imports:**
- `hashlib` — to independently compute the expected SHA-256 (avoids trusting the implementation)
- `pytest` — test framework (only for the `tmp_path` fixture; no `pytest.raises` or parametrize used here)
- `reasons.api` — the public API surface (`init_db`, `add_node`, `show_node`, `check_integrity`, `backfill_hashes`, `export_network`, `import_json`, `supersede_with_text`, `add_justification`, `_with_network`)
- `reasons.merkle` — the functions under test (`compute_text_hash`, `compute_merkle_hash`, `compute_content_hash`, `verify_all`, `backfill_hashes`)
- `reasons.storage` (`Storage`) — for direct DB-level mutation in tamper tests
- `reasons` (`Justification`) — to manually construct a justification in the cycle test
- `json` — for the export/import round-trip test

**Nothing imports this file** — it is a leaf test module, discovered by pytest's `conftest.py` and `pyproject.toml` test configuration.

## Flow

Each test follows the same shape:

```
1. Create a temp DB:        db = tmp_path / "test.db"
2. Initialize:              api.init_db(str(db))
3. Build the belief graph:  api.add_node(...) / api.add_justification(...)
4. (Optional) Mutate state: Storage.load() → edit → Storage.save()
5. Invoke the function:     api.check_integrity(...) / api.backfill_hashes(...)
6. Assert on the result dict
```

The most instructive flow is `test_merkle_transitive`:

```
Graph:  c  →  b  →  a
         (leaf)  (justified by c)  (justified by b)

Step 1: check_integrity → all clean (0 text, 0 chain, 0 missing)
Step 2: Mutate c.text and c.text_hash (simulating a "legit" edit that skips re-hashing justifications)
Step 3: check_integrity →
         - text_mutations = 0  (c's text matches its text_hash)
         - chain_mutations = ["b", "a"]
           * b is flagged because b's justification cites c, and c's hash changed
           * a is flagged because a's justification cites b, and b's merkle hash
             transitively incorporates c's hash → b's hash is now stale → a's
             content_hash no longer matches
```

This is the transitive Merkle property: a single leaf mutation cascades up the DAG.

## Invariants

The tests encode these rules that the implementation **must** uphold:

1. **`text_hash` is always `sha256(node.text)`** — verified in `test_text_hash_set_on_creation` and `test_supersede_preserves_hashes`.
2. **`content_hash` is non-empty and unique per justification** — `test_content_hash_set_on_creation`, `test_multiple_justifications_each_hashed`.
3. **`content_hash` depends on antecedent text hashes** — `test_content_hash_incorporates_antecedents` proves that changing an antecedent's text (even with a matching text_hash) invalidates the dependent's content hash.
4. **Transitivity** — a mutation at depth *n* flags all ancestors at depths *n+1 … max* (`test_merkle_transitive`).
5. **Idempotent verification** — running `check_integrity` on a clean graph always yields zero findings (`test_verify_clean`).
6. **Backfill is lossless** — after `backfill_hashes`, a subsequent `check_integrity` reports zero missing hashes and zero mutations (`test_backfill_hashes`).
7. **Export/import is hash-preserving** — serialized hashes survive a JSON round-trip (`test_export_import_preserves_hashes`).
8. **Cycle termination** — `verify_all` must terminate even on cyclic antecedent references (`test_cycle_safety`).

## Error Handling

This test file **asserts** rather than **catches**. There are no `pytest.raises` blocks. The expectation is:

- `check_integrity` always returns a well-formed dict with keys `text_mutations`, `chain_mutations`, and `missing_hashes`. It never raises on a corrupt graph — it *reports* corruption.
- `backfill_hashes` always returns a dict with `nodes_updated` and `justifications_updated` counts.
- The cycle test (`test_cycle_safety`) is implicitly testing that the function does **not** raise `RecursionError` or hang. The `assert isinstance(result, dict)` is a liveness check.

The only "error path" exercised is the **detection** path: the system is expected to *find* tampering, not crash on it.

---

## Topics to Explore

- [file] `reasons/merkle.py` — The implementation under test: how `compute_content_hash` folds antecedent hashes, how `verify_all` walks the DAG with cycle detection, and how `backfill_hashes` traverses the graph to fill gaps.
- [file] `reasons/storage.py` — The `Storage` class used to bypass the API for tamper simulation. Understanding its `load()`/`save()` contract and how it serializes/deserializes the network is critical for interpreting the mutation tests.
- [function] `reasons/api.py:check_integrity` — The public wrapper that `verify_all` plugs into. Worth reading to see how it loads the network, calls the merkle functions, and shapes the result dict.
- [general] `merkle-hash-design` — The layered hash scheme (text_hash → content_hash → transitive propagation) is the core security/integrity model of the project. Understanding *why* a simple content hash per node isn't sufficient (it isn't — you need antecedent-aware hashing) clarifies the whole architecture.
- [file] `tests/conftest.py` — May define fixtures or autouse setup (e.g., environment variables, DB isolation) that affect how these tests run in CI.

## Beliefs

- `text-hash-is-sha256-of-raw-bytes` — A node's `text_hash` is always the hex digest of `hashlib.sha256(text.encode())`; tests verify this against an independently computed `hashlib.sha256(b"...")` value.
- `content-hash-depends-on-antecedent-text-hashes` — A justification's `content_hash` changes when any antecedent node's `text_hash` changes, even if the antecedent's own `text` and `text_hash` are internally consistent; this is what distinguishes a "chain mutation" from a "text mutation."
- `check-integrity-distinguishes-text-vs-chain` — `check_integrity` returns `text_mutations` (text ≠ stored text_hash) and `chain_mutations` (justification's content_hash is stale relative to current antecedent hashes) as separate categories; a correctly-updated text_hash suppresses the former but not the latter.
- `backfill-hashes-is-idempotent-and-complete` — After `backfill_hashes` runs, `check_integrity` reports zero `missing_hashes`, zero `text_mutations`, and zero `chain_mutations`; running it twice produces no further changes.
- `verify-all-terminates-on-cycles` — `verify_all` (and by extension `api.check_integrity`) must terminate in finite time even when antecedent references form a cycle; it does not raise `RecursionError`.
