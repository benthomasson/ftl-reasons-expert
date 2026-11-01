---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_deduplication.py

# `tests/test_deduplication.py` — Explanation

## 1. Purpose

This file is the **dedicated test suite for the deduplication and supersession semantics** of the `reasons` belief-network API. It owns the contract for two related API methods:

- `api.mark_duplicate(source_id, canonical_id)` — collapses one belief node into another, declaring it a redundant copy.
- `api.mark_superseded(old_id, new_id)` — marks an older belief as invalidated by a newer, updated belief.

Both operations are **retraction-with-provenance**: they flip a node's truth value to `OUT` while recording *why* and *what replaced it* in structured metadata. The tests here pin down the exact metadata keys, the cascade behavior to dependent nodes, error semantics, and idempotency guarantees.

## 2. Key Components

All test functions follow the same shape: **setup → action → assert**. There are no helper classes; the file is a flat collection of 12 test functions.

| Function | Contract under test |
|---|---|
| `test_mark_duplicate` | `mark_duplicate` returns `{"source_id", "canonical_id", "changed"}`; the source node becomes `OUT` with `metadata.duplicate_of` and `metadata.retract_reason == "Duplicate of <canonical>"`. |
| `test_mark_superseded` | `mark_superseded` returns `{"old_id", "new_id", "changed"}`; the old node becomes `OUT` with `metadata.superseded_by` and `metadata.retract_reason == "Superseded by <new>"`. |
| `test_mark_duplicate_with_cascade` | Any node linked via `sl=` (support-link) to the duplicated node **also** flips to `OUT` and appears in the `changed` list. |
| `test_mark_superseded_with_cascade` | Same cascade guarantee for the superseded path. |
| `test_mark_duplicate_errors` | Passing a non-existent `source_id` or `canonical_id` raises `KeyError("… not found")`. |
| `test_mark_superseded_errors` | Same `KeyError` contract for `mark_superseded`. |
| `test_metadata_in_export` | `api.export_network()` serializes the `duplicate_of` / `superseded_by` keys and the `OUT` truth value into the JSON export. |
| `test_metadata_in_show_output` | `api.show_node()` surfaces the same metadata keys inline. |
| `test_mark_duplicate_self_reference` | `mark_duplicate("a", "a")` raises `ValueError("… cannot be marked as a duplicate of itself")`; the node stays `IN`. |
| `test_mark_superseded_self_reference` | Same guard for `mark_superseded`. |
| `test_mark_duplicate_already_out` | Calling `mark_duplicate` on a node that was *already* retracted (via `retract_node`) still succeeds, **overwrites** the `retract_reason` to `"Duplicate of <canonical>"`, and preserves `OUT`. |
| `test_mark_superseded_already_out` | Same idempotency / overwrite behavior for `mark_superseded`. |

### API surface exercised

The tests collectively lock in the following `reasons.api` signatures and return shapes:

```python
api.init_db(db_path)                                  # → None
api.add_node(id, text, db_path, sl=None)              # → None; sl creates a support-link
api.show_node(node_id, db_path)                       # → {"truth_value", "metadata", …}
api.mark_duplicate(source_id, canonical_id, db_path)  # → {"source_id", "canonical_id", "changed": [ids]}
api.mark_superseded(old_id, new_id, db_path)          # → {"old_id", "new_id", "changed": [ids]}
api.retract_node(id, reason, db_path)                 # → None
api.export_network(db_path)                           # → {"nodes": {id: {…}}, …}
```

## 3. Patterns & Conventions

- **Isolation via `tmp_path`.** Every test builds a throwaway SQLite file under pytest's `tmp_path` fixture. No shared state, no network. This is the repo-wide convention (you can see the same pattern in `test_contradictions.py`, `test_dangling_dependents.py`, etc.).
- **Arrange-Act-Assert, no helpers.** Each test is self-contained. There are no fixtures beyond `tmp_path`, no shared `conftest` setup specific to this file. The `conftest.py` at `tests/` presumably provides no additional fixtures for these tests.
- **Symmetric test pairs.** Every `mark_duplicate` test has a `mark_superseded` counterpart (`_errors`, `_self_reference`, `_already_out`, `_with_cascade`). This is an intentional mirroring: the two operations are semantically parallel, and the suite guarantees they behave identically across all edge cases.
- **Cascade verification is explicit.** The tests don't just check the target node; they verify that *dependents* (created with `sl=`) are included in the `changed` list and independently flipped to `OUT`. This guards against a regression where the cascade logic is silently dropped.
- **Metadata assertions are string-exact.** The tests pin the *literal* values of `retract_reason` (`"Duplicate of canonical-node"`, `"Superseded by new-belief"`), not just the presence of the key. This means the implementation must format these strings deterministically.

## 4. Dependencies

### What this file depends on

| Dependency | Role |
|---|---|
| `pytest` | Test runner; provides `tmp_path` fixture and `pytest.raises` context manager. |
| `reasons.api` | The public API under test. The file imports only `api` from the `reasons` package. |
| `reasons.storage` (transitive) | `api` delegates to the storage layer for SQLite I/O; tests don't import this directly but the `tmp_path` SQLite file exercises it. |

### What depends on this file

Nothing imports a test file. However, this suite is the **regression net** for the deduplication/supersession code paths in `reasons/api.py` (and the underlying storage/metadata logic in `reasons/storage.py` and `reasons/metadata.py`). If those modules change their metadata keys, return shapes, or cascade behavior, these tests will fail first.

## 5. Flow (per-test execution)

Every test follows this sequence:

1. **Create isolated DB.**
   ```python
   db = tmp_path / "test.db"
   api.init_db(str(db))
   ```
   This creates (or opens) a SQLite database and runs any DDL migrations.

2. **Seed nodes.**
   ```python
   api.add_node("canonical-node", "…", db_path=str(db))
   api.add_node("duplicate-node", "…", db_path=str(db))
   ```
   The optional `sl=` parameter creates a **support-link** (a directed edge meaning "this node's truth depends on that node"). This is how the cascade tests set up dependent nodes.

3. **Act.**
   ```python
   result = api.mark_duplicate("duplicate-node", "canonical-node", db_path=str(db))
   ```
   Internally (in the implementation), this:
   - Validates both node IDs exist (else `KeyError`).
   - Rejects self-references (else `ValueError`).
   - Sets the source node's `truth_value` to `OUT`.
   - Writes `metadata.duplicate_of` (or `.superseded_by`) and `metadata.retract_reason`.
   - Walks all nodes linked to the source via support-links and flips them to `OUT` as well.
   - Collects all touched node IDs into the `changed` list.
   - Returns the result dict.

4. **Assert the result dict** (keys, values, membership in `changed`).

5. **Assert the persisted state** by calling `api.show_node()` or `api.export_network()` and checking `truth_value`, `metadata.*` keys, and the exact `retract_reason` string.

For the **already-OUT** tests, step 2 additionally calls `api.retract_node("duplicate", reason="Initially retracted", …)` to pre-flip the node, and step 5 asserts that the `retract_reason` was **overwritten** to the dedup-specific message.

## 6. Invariants Enforced

These are the rules the test suite locks in as non-negotiable:

- **Self-reference is forbidden.** `mark_duplicate(x, x)` and `mark_superseded(x, x)` must raise `ValueError` and leave the node in its original `IN` state.
- **Both endpoints must exist.** A missing source *or* target must raise `KeyError` with the substring `"not found"`.
- **Retraction is irreversible within the operation.** Once `mark_duplicate`/`mark_superseded` succeeds, the source node is `OUT`. There is no "un-duplicate" tested here (that would be a separate operation).
- **Cascade is all-or-nothing in `changed`.** If a dependent node exists, it *must* appear in `result["changed"]` and *must* be `OUT` afterward. The tests don't test partial cascades, implying the contract is binary.
- **Metadata is the source of truth for provenance.** The `duplicate_of` / `superseded_by` keys and the human-readable `retract_reason` string are the only recorded explanation. Downstream consumers (`export_network`, `show_node`, CLI output) must surface them.
- **Overwrite semantics for already-OUT nodes.** If a node was retracted for a different reason, `mark_duplicate` *replaces* the `retract_reason` rather than appending or failing.

## 7. Error Handling

The tests define **three distinct error classes**:

| Condition | Exception | Match pattern | Example |
|---|---|---|---|
| Source or target node ID doesn't exist in the DB | `KeyError` | `"not found"` | `mark_duplicate("missing", "exists")` |
| Source and target are the same node | `ValueError` | `"cannot be marked as a duplicate of itself"` / `"cannot be marked as superseded by itself"` | `mark_duplicate("a", "a")` |
| *(not tested here)* | — | — | Cycles, circular supersession chains, etc. |

All assertions use `pytest.raises(..., match=…)`, meaning the **message content** is part of the contract. If the implementation changes the error message wording, these tests break.

There is no error swallowing: the tests expect exceptions to propagate to the caller. The API does not return error dicts or `None` on failure.

---

## Topics to Explore

- [function] `reasons/api.py:mark_duplicate` — The implementation under test. Understanding how it resolves support-links, writes metadata, and builds the `changed` list will clarify the cascade mechanism and why `retract_reason` gets overwritten on already-OUT nodes.
- [function] `reasons/api.py:mark_superseded` — Its sibling operation. Comparing it side-by-side with `mark_duplicate` reveals whether they share an internal helper (DRY) or are fully independent code paths (duplication risk).
- [file] `reasons/metadata.py` — Likely defines the metadata schema, key names (`duplicate_of`, `superseded_by`, `retract_reason`), and validation. Understanding this file explains *why* the tests assert on specific string values.
- [file] `tests/test_dangling_dependents.py` — Tests the inverse scenario: what happens when a node is removed and its dependents become "dangling." Reading both files together gives the full picture of referential integrity in the belief graph.
- [general] `support-link-semantics` — The `sl=` parameter on `add_node` creates a directed dependency edge. Understanding its full semantics (direction, transitivity, what "OUT" propagation means for multi-hop chains) is critical for reasoning about cascade behavior beyond the single-hop cases tested here.

## Beliefs

- `mark-duplicate-overwrites-retract-reason` — Calling `mark_duplicate` on a node that was already retracted via `retract_node` replaces the existing `retract_reason` metadata with `"Duplicate of <canonical>"`; it does not append, merge, or raise.
- `mark-duplicate-cascades-via-support-links` — The `changed` list in `mark_duplicate`'s return value includes not only the source node but every node reachable via a single-hop support-link (`sl=`), and each is independently set to `OUT`.
- `mark-duplicate-and-mark-superseded-share-error-contract` — Both operations raise `KeyError` (substring `"not found"`) for missing node IDs and `ValueError` (substring `"itself"`) for self-references; the test suite enforces identical error semantics across the pair.
- `export-network-serializes-dedup-metadata` — `api.export_network()` includes `duplicate_of` / `superseded_by` in each node's `metadata` dict, making deduplication provenance available to external consumers (CLI, MCP server, Hugging Face export) without a separate lookup.
- `self-reference-guard-preserves-state` — When `mark_duplicate` or `mark_superseded` rejects a self-reference, the target node's `truth_value` remains `"IN"` and no metadata is written; the operation is fully atomic in its failure.
