---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_remove_justification.py

## Explanation of `tests/test_remove_justification.py`

### Purpose

This file is the behavioral specification for the **"remove a single justification from a node"** operation across the three public layers of the `reasons` project:

1. `Network.remove_justification` — in-memory graph engine
2. `api.remove_justification` — persisted (SQLite-backed) wrapper
3. CLI subcommand `remove-justification` — end-user entry point

It exists because a derived node may carry multiple `Justification` records (multiple independent arguments proving it). Removing one — but not all — must *mutate the justification list*, *recompute truth values*, *cascade to dependents*, and *stay consistent with the reverse-index (`dependents`) bookkeeping*. That last part is subtle enough to warrant dedicated tests.

The file is the counterpart to `tests/test_add_justification.py`; together they define the "incremental justification editing" contract.

---

### Key Components

#### `TestNetworkRemoveJustification` (10 cases)

Exercises `Network.remove_justification(node_id: str, index: int) -> dict`.

**Return contract** (inferred from assertions):

| Key | Meaning |
|---|---|
| `removed` | the `Justification` object that was at `index` (`.label` read in `test_remove_one_of_two`) |
| `remaining` | `int` — count of justifications left on the node |
| `old_truth_value` | truth value before removal (`"IN"`/`"OUT"`/`"UNKNOWN"`) |
| `new_truth_value` | truth value after |
| `changed` | iterable of node IDs whose truth value changed (includes the target and dependents) |

**Individual cases:**

- `test_remove_one_of_two` — happy path: two SL justifications `[a]`, `[b]`; removes index 0; asserts `justifications[0].label == "second"` — i.e. the list is re-indexed/compacted, not tombstoned.
- `test_remove_causes_out` — with `b` retracted (OUT), `c` is `IN` only because of `a`; removing the `a`-justification flips `c` to `OUT` and reports `old_truth_value == "IN"`, `new_truth_value == "OUT"`.
- `test_remove_propagates_cascade` — adds `d` derived from `c`; losing `c`'s justification makes `d` drop to `OUT`, and `d` must appear in `result["changed"]`.
- `test_remove_cleans_dependents` — removing justification 0 (antecedent `a`) removes `c` from `net.nodes["a"].dependents` but leaves `c` in `net.nodes["b"].dependents`.
- `test_remove_keeps_shared_dependent` — justification 0 uses `[a,b]`, justification 1 uses `[a]`; removing index 0 must **not** remove `c` from `a.dependents`, because the surviving justification still references `a`. Enforces reference-counted back-index maintenance.
- `test_remove_outlist_cleans_dependents` — `outlist=["blocker"]` also registers `c` in `blocker.dependents`; removing that justification removes the edge. This confirms `dependents` tracks **both** antecedents and outlist entries.
- `test_error_on_premise` — premises have no justifications to remove → `ValueError` matching `"premise"`.
- `test_error_on_single_justification` — cannot leave a derived node with zero justifications → `ValueError` matching `"only one justification"`.
- `test_error_on_bad_index` — `IndexError` matching `"out of range"` for `index=5`.
- `test_error_on_missing_node` — `KeyError` matching `"not found"`.

#### `TestApiRemoveJustification.test_api_remove_justification`

Uses `tmp_path` to create a real SQLite file. Flow:

```
api.init_db(db_path=db)
api.add_node("a"...), api.add_node("b"...)
api.add_node("c", sl="a", ...)        # first justification
api.add_justification("c", sl="b", ...)  # second justification
```

Asserts `show_node("c")["justifications"]` starts at 2 and drops to 1 after `api.remove_justification("c", 0, db_path=db)`. Shapes the API signature: `(node_id, index, *, db_path)`.

#### `TestCliRemoveJustification.test_cli_remove_justification`

Integration test using `subprocess.run(["uv", "run", "reasons", "--db", db, "remove-justification", "c", "0"])`. Asserts:

- `returncode == 0`
- stdout contains `"Removed justification 0"`
- The remaining justification preserves label `"keep-this"` (added second via `api.add_justification(..., label="keep-this")`).

This pins the **CLI argument order** `remove-justification <node_id> <index>` and the confirmation message format.

---

### Patterns

- **Layered test pyramid** — all three layers (in-memory `Network`, `api` DB facade, CLI subprocess) are exercised in one file, in the same order as the abstraction stack.
- **Fixture via pytest's `tmp_path`** — isolated SQLite files, no cleanup needed.
- **Arrange / Act / Assert with explicit state probes** — tests build tiny graphs (`a`, `b`, `c`, `d`) and inspect both the *return value* (`result["..."]) and the *side effects* (`net.nodes[...].justifications`, `.dependents`, `.truth_value`). This guards against "returns the right dict but mutates the graph wrong (or vice-versa)".
- **Error-contract testing with `pytest.raises(..., match=...)`** — messages are part of the public contract; assertions match lowercase substrings.
- **Reference-counted back-index** — the `test_remove_keeps_shared_dependent` test codifies that `dependents` sets are maintained like a reverse index over the *union* of justifications, not replaced wholesale.
- **Subprocess integration** — only one test hits the CLI; keeps subprocess overhead proportional to what needs end-to-end verification (argument parsing + message formatting).

---

### Dependencies

**Imports (this file → project):**

- `from reasons import Justification, api` — the DTO/record type and the DB-backed facade.
- `from reasons.network import Network` — the in-memory engine under test.
- `pytest` — fixtures and assertions.
- `subprocess` — imported *inside* the CLI test method (avoids polluting module import for the pure Python tests).

**Imported by:** nothing — this is a leaf test module. Pytest discovers it via the `test_*.py` convention. `tests/__init__.py` exists, so tests are a package; the file is picked up by `pytest tests/`.

**External dependency of the CLI test:** the `reasons` console script (declared in `pyproject.toml`), run through `uv run`. If either `uv` is absent or the script isn't installed in the environment, this test fails with `FileNotFoundError` — a real environmental coupling to note.

---

### Flow

For the canonical scenario (`test_remove_causes_out`):

1. Build a graph where a derived node `c` has two justifications — one from an `IN` antecedent (`a`) and one from an `OUT` antecedent (`b`). Because at least one justification is acceptable, `c` is `IN`.
2. Call `net.remove_justification("c", 0)`.
3. Inside `Network`, this must:
   a. Validate the node exists and has > 1 justification (else raise).
   b. Pop `justifications[index]` from `c`.
   c. For every antecedent (and outlist entry) in the removed `Justification`, drop `c` from that node's `dependents` **only if no remaining justification on `c` still references it**.
   d. Re-evaluate truth of `c` (`IN` → `OUT`).
   e. Recursively re-evaluate dependents of `c` (`d` → `OUT` in the cascade test), collecting the IDs into `result["changed"]`.
4. Return a summary dict.

The API layer wraps this: load node → delegate to `Network.remove_justification` (or equivalent serialized logic) → persist updated justifications + truth values → return the same summary shape. The CLI wraps the API and formats `"Removed justification {index}"`.

---

### Invariants (what the tests enforce as rules)

1. **Premises cannot have justifications removed** — `ValueError` mentioning `"premise"`.
2. **A derived node must retain ≥ 1 justification** — `ValueError` mentioning `"only one justification"`.
3. **Index must be a valid position in the current `justifications` list** — `IndexError` mentioning `"out of range"`.
4. **Node must exist** — `KeyError` mentioning `"not found"`.
5. **Truth values are consequences of the surviving justification set**, not stored state — removing the last "acceptable" justification flips `IN → OUT`, and this change propagates transitively to dependents.
6. **`dependents` is a precise reverse index** — a dependent edge from antecedent `X` to node `Y` exists **iff some justification of `Y` references `X`** (in `antecedents` or `outlist`). This is what `test_remove_keeps_shared_dependent` and `test_remove_outlist_cleans_dependents` pin down.
7. **The returned summary is a superset of the mutation** — it exposes `removed`, `remaining`, `old_truth_value`, `new_truth_value`, and `changed` so callers (API/CLI) don't need to re-query.

---

### Error Handling

- **Explicit, typed exceptions** from `Network.remove_justification`: `ValueError` (premise guard, last-justification guard), `IndexError` (bad index), `KeyError` (missing node). Each carries a human-readable message that the tests treat as part of the API.
- **No swallowing** — these propagate to the caller (`api`, then CLI). The CLI test only asserts `returncode == 0` on the happy path; the file does not test the CLI's error rendering for these error types (that's presumably in `test_cli.py` or a broader CLI test).
- **`pytest.raises(match=...)`** uses regex matching on the message; the tests only match short lowercase substrings (`"premise"`, `"out of range"`, `"not found"`, `"only one justification"`), which keeps them robust to message prefix changes but requires those substrings to remain.

---

## Topics to Explore

- [function] `reasons/network.py:Network.remove_justification` — the implementation behind all ten network tests; inspect how truth propagation and `dependents` recomputation actually work.
- [function] `reasons/api.py:remove_justification` — the DB facade; confirm it shares the `Network` summary contract and how it persists re-indexing.
- [file] `reasons/cli.py` — locate the `remove-justification` subparser to verify argument order and the `"Removed justification {index}"` message string.
- [file] `tests/test_add_justification.py` — the mirror contract; understanding it sharpens the invariants around dependents/outlists on the *insert* side.
- [general] `truth-value-propagation` — how `IN`/`OUT`/`UNKNOWN` is recomputed across dependents during any mutation, which underpins the cascade tests.

---

## Beliefs

- `remove-justification-returns-summary-dict` — `Network.remove_justification` returns a dict with `removed`, `remaining`, `old_truth_value`, `new_truth_value`, and `changed`.
- `justification-removal-requires-multiple-justifications` — `Network.remove_justification` raises `ValueError` when the target node has only one justification.
- `justification-removal-rejects-premises` — `Network.remove_justification` raises `ValueError` on premise nodes.
- `justification-removal-causes-truth-cascade` — removing a justification re-evaluates the node's truth value and propagates changes to dependents listed in `result["changed"]`.
- `dependents-index-is-reference-counted` — a node remains in an antecedent's `dependents` set as long as *any* surviving justification references it.
- `dependents-tracks-outlist-entries` — `outlist` entries are added to and removed from `dependents` the same way as `antecedents`.
- `cli-remove-justification-tested-via-subprocess` — the CLI test invokes `uv run reasons ... remove-justification <node> <index>` and asserts stdout contains `"Removed justification 0"`.
