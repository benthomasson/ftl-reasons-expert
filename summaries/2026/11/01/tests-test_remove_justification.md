---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_remove_justification.py

# `tests/test_remove_justification.py`

## 1. Purpose

This file is the test suite for the **remove-justification** operation in the `ftl-reasons` defeasible logic engine. It validates that removing a single justification (by index) from a multi-justification node correctly:

- Updates the node's truth value (IN/OUT)
- Propagates truth-value changes downstream through the network
- Cleans up the `dependents` sets on antecedent and outlist nodes
- Preserves shared dependents that are still referenced by remaining justifications
- Surfaces clear errors on invalid operations

It tests this behavior at **three architectural layers**: the in-memory `Network` class, the persistent `api` module (SQLite-backed), and the CLI subprocess.

## 2. Key Components

### `TestNetworkRemoveJustification`
Tests the core `Network.remove_justification(node_id, index)` method directly in memory.

| Test | What it verifies |
|---|---|
| `test_remove_one_of_two` | Basic removal: correct justification is removed, `remaining` count is right, the surviving justification is intact |
| `test_remove_causes_out` | Removing the *only* support for an IN node (where the alternative justification depends on a retracted node) flips truth to OUT |
| `test_remove_propagates_cascade` | Truth-value change propagates to downstream nodes (`d` depends on `c`); `result["changed"]` includes `"d"` |
| `test_remove_cleans_dependents` | After removal, the antecedent of the removed justification (`"a"`) is no longer in `c`'s dependents; the antecedent of the surviving justification (`"b"`) still is |
| `test_remove_keeps_shared_dependent` | If `"a"` appears in *both* justifications, removing one does **not** remove `"a"` from `dependents` |
| `test_remove_outlist_cleans_dependents` | Outlist entries (defeaters) also register in `dependents`; removing the justification that listed a defeater cleans that up |
| `test_error_on_premise` | `ValueError` if the target is a premise (no justifications) |
| `test_error_on_single_justification` | `ValueError` if the node has only one justification (can't remove the last) |
| `test_error_on_bad_index` | `IndexError` for out-of-range index |
| `test_error_on_missing_node` | `KeyError` for a non-existent node ID |

### `TestApiRemoveJustification`
One integration test that exercises `api.remove_justification(node, index, db_path=...)` against a real SQLite database (via the `tmp_path` fixture). Verifies the justification count drops from 2 to 1 and the remaining justification is correct.

### `TestCliRemoveJustification`
An end-to-end test that shells out to `uv run reasons --db <path> remove-justification c 0` via `subprocess.run`, asserting:
- Exit code is 0
- stdout contains `"Removed justification 0"`
- The surviving justification retains its label (`"keep-this"`)

### Contracts

- **`Network.remove_justification(node_id: str, index: int) -> dict`** — Returns a dict with keys: `removed` (the justification), `remaining` (int), `old_truth_value`, `new_truth_value`, `changed` (set/list of node IDs whose truth flipped).
- **`api.remove_justification(node: str, index: int, db_path: str) -> dict`** — Same semantic contract but persisted to SQLite.
- **CLI `remove-justification <node> <index>`** — Prints a confirmation message, exits 0 on success.

## 3. Patterns

- **Three-tier test pyramid**: Unit (in-memory `Network`), integration (API + SQLite), e2e (CLI subprocess). Each tier increases setup cost and decreases test count, which is the standard pattern here.
- **`tmp_path` isolation**: DB-backed tests use pytest's `tmp_path` fixture so every test gets a fresh database directory. No shared state between tests.
- **`pytest.raises` for error contracts**: Each error path gets its own test with a `match=` regex, so the *message* is part of the contract (e.g., `match="premise"`, `match="only one justification"`, `match="out of range"`, `match="not found"`).
- **Subprocess CLI testing**: The CLI test uses `subprocess.run(["uv", "run", "reasons", ...])` rather than invoking the CLI function directly. This tests the actual user-facing entry point including argument parsing.
- **Assertion of side effects, not just return values**: Most tests assert on the *state of the network* after the operation (e.g., `net.nodes["c"].truth_value`, `net.nodes["a"].dependents`), not just the return dict. This catches bugs where the return value looks correct but the internal state is inconsistent.

## 4. Dependencies

### This file imports
| Import | Role |
|---|---|
| `pytest` | Test framework, fixtures (`tmp_path`), assertions |
| `reasons.Justification` | Dataclass for a single justification (type, antecedents, outlist, label) |
| `reasons.api` | Persistent API layer (init_db, add_node, add_justification, show_node, remove_justification) |
| `reasons.network.Network` | Core in-memory defeasible logic network |
| `subprocess` | Shelling out to the CLI (CLI test only) |

### What depends on this file
Nothing imports a test file. However, this test file guards the `Network.remove_justification` and `api.remove_justification` code paths in `reasons/network.py` and `reasons/api.py` (or `reasons_lib/`). Breaking those functions without updating this file will cause CI failures.

## 5. Flow

### Network-level flow (e.g., `test_remove_propagates_cascade`)
```
1. net.add_node("a", ...)          → node "a" is IN (premise)
2. net.add_node("b", ...)          → node "b" is IN
3. net.retract("b")               → node "b" is OUT
4. net.add_node("c", justifications=[SL(a), SL(b)])
                                   → "c" is IN (SL(a) fires; SL(b) doesn't because b is OUT)
5. net.add_node("d", justifications=[SL(c)])
                                   → "d" is IN (c is IN)
6. net.remove_justification("c", 0)
   a. Validates: "c" exists, is not a premise, has >1 justification, index in range
   b. Removes justification[0] (the SL(a) one)
   c. Re-evaluates "c": only SL(b) remains, but b is OUT → "c" flips to OUT
   d. Cascade: "d" depends on "c" → "d" flips to OUT
   e. Cleans dependents: "a" was only in the removed justification → remove "c" from a.dependents
   f. Returns {removed, remaining, old_truth_value, new_truth_value, changed: {"d"}}
7. Assertions verify each of the above
```

### API-level flow
```
1. api.init_db(db_path)            → creates fresh SQLite schema
2. api.add_node / api.add_justification  → populates DB
3. api.remove_justification("c", 0, db_path)
   → loads network from DB, calls Network.remove_justification, persists back
4. api.show_node("c", db_path)     → re-reads to confirm persisted state
```

### CLI-level flow
```
1. Setup via api.* (same as API test)
2. subprocess.run(["uv", "run", "reasons", "--db", db, "remove-justification", "c", "0"])
   → CLI parses args, loads DB, calls api.remove_justification, prints confirmation
3. Verify via api.show_node (separate process reads the same DB file)
```

## 6. Invariants Enforced

These are the rules the tests pin down — if any of them breaks, a test fails:

1. **A premise cannot have justifications removed.** Premises are axioms; they have no justifications by construction. (`test_error_on_premise`)
2. **The last justification cannot be removed.** A node with a single justification is irreducible — removing it would make the node unjustifiable, which is a different operation (retraction). (`test_error_on_single_justification`)
3. **Index bounds are validated.** Passing an index ≥ len(justifications) raises `IndexError`, not silently wrapping or returning a wrong justification. (`test_error_on_bad_index`)
4. **Dependent cleanup is exact.** An antecedent is removed from `dependents` only if it does *not* appear in any remaining justification. Shared antecedents are preserved. (`test_remove_cleans_dependents`, `test_remove_keeps_shared_dependent`)
5. **Outlist entries participate in dependents just like antecedents.** A defeater listed in an outlist is a dependent of that node and is cleaned up when the justification is removed. (`test_remove_outlist_cleans_dependents`)
6. **Truth-value cascades are transitive.** If removing a justification flips node C from IN→OUT, and node D depends on C, then D must also flip. The `changed` set reports all affected downstream nodes. (`test_remove_propagates_cascade`)

## 7. Error Handling

All error cases are **raised, not returned**. The tests verify specific exception types and message substrings:

| Condition | Exception | Message match |
|---|---|---|
| Target is a premise | `ValueError` | `"premise"` |
| Only one justification exists | `ValueError` | `"only one justification"` |
| Index out of range | `IndexError` | `"out of range"` |
| Node ID not found | `KeyError` | `"not found"` |

This is a deliberate choice: `remove_justification` is a mutation operation with no "soft-fail" mode. The caller must handle the exception. The CLI layer presumably catches these and prints a user-friendly error (the CLI test only checks the success path).

The `result` dict on success carries `old_truth_value` / `new_truth_value` as a diagnostic aid — callers can log or display what changed without having to snapshot the network beforehand.

---

## Topics to Explore

- [function] `reasons/network.py:remove_justification` — The core implementation under test; understanding the re-evaluation, cascade, and dependent-cleaning logic here is essential for modifying any of the invariants above.
- [file] `reasons/network.py` — The full `Network` class: how `add_node`, `retract`, `truth_value` computation, and the `dependents` bookkeeping interact. The tests in this file exercise only one operation, but the invariants depend on the broader network model.
- [file] `tests/test_defeat_justification.py` — The complementary "defeat" operation (marking a justification as defeated without removing it). Comparing the two reveals the distinction between *removing* and *defeating* a justification and why both exist.
- [file] `reasons/api.py` — The persistence layer: how `remove_justification` serializes/deserializes the network to/from SQLite, and whether the API adds any validation beyond what `Network` does.
- [general] `defeasible-logic-semantics` — The broader FCL/FTL model: what SL (strictly-literal) justification type means, how IN/OUT truth values are computed, and what "outlist" (defeater) semantics imply for dependency tracking.

---

## Beliefs

- `remove-justification-preserves-shared-dependents` — An antecedent node is only removed from `dependents` if it does not appear in any *remaining* justification of the target node; if it appears in both the removed and a surviving justification, it stays in `dependents`.
- `remove-justification-requires-multiple-justifications` — `Network.remove_justification` raises `ValueError` when the target node has exactly one justification, making the "remove last justification" case a hard error rather than a retraction.
- `remove-justification-propagates-truth-cascade` — Removing a justification that causes a node's truth value to flip from IN to OUT triggers a transitive cascade, and the `changed` key in the return dict lists all downstream nodes whose truth value also flipped.
- `outlist-entries-register-in-dependents` — Nodes listed in a justification's `outlist` (defeaters) are tracked in the same `dependents` sets as antecedents, and are cleaned up identically when the justification is removed.
- `cli-remove-justification-uses-uv-run` — The CLI end-to-end test invokes the tool via `uv run reasons` rather than a direct `python -m` call, meaning the test depends on the `uv` toolchain and the package being resolvable in the project environment.
