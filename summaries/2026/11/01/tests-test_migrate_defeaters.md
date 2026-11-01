---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_migrate_defeaters.py

# `tests/test_migrate_defeaters.py`

## Purpose

This file tests the **backward-migration path** for the defeasible reasoning engine. The system originally stored retraction as a metadata field (`retract_reason`) on an OUT node. It was refactored to a first-class **defeater node** that causally flips a node to OUT. This test file verifies that `api.migrate_retract_to_defeaters` correctly converts legacy retraction records into the new defeater-node model, and that the conversion preserves, clears, and reverses the right state.

In short: it owns the contract that *old retraction data can be losslessly migrated to defeater nodes without side effects*.

## Key Components

All tests exercise a single function: **`api.migrate_retract_to_defeaters`**. Its effective signature (derived from the call sites) is:

```python
def migrate_retract_to_defeaters(
    *,
    dry_run: bool = ...,
    db_path: str,
    node_ids: list[str] | None = None,
) -> dict:
    """Returns {"migrated": [...], "skipped": [...], "errors": [...]}"""
```

| Key in return dict | Contents |
|---|---|
| `migrated` | List of dicts with `id` (source node) and `defeater_id` (new node). Absent in dry-run. |
| `skipped` | List of dicts with `id` (and presumably a `reason`). |
| `errors` | List of dicts with `id` for node_ids that couldn't be resolved. |

The test file defines **8 test functions**, each a single-`assert`-chain scenario:

| Test | What it pins down |
|---|---|
| `test_basic_migration` | Happy path: OUT node + `retract_reason` + satisfied justification → new defeater node created, old metadata removed, defeater metadata set. |
| `test_dry_run` | `dry_run=True` reports candidates but **does not** create a defeater or clear metadata. |
| `test_skip_unsatisfied_justification` | If the antecedent (`base`) is itself OUT, the justification is unsatisfied → node is **skipped**, not migrated. |
| `test_skip_no_retract_reason` | OUT node with *no* `retract_reason` is simply not a candidate. |
| `test_metadata_cleared` | Both `retract_reason` **and** `_retracted` keys are removed from the node's metadata post-migration. |
| `test_reversibility` | Retracting the migrated defeater node restores the target back to IN. |
| `test_node_ids_filter` | Passing `node_ids=["d1"]` migrates only `d1`; `d2` is untouched. |
| `test_error_on_missing_node` | A `node_id` that doesn't exist lands in `result["errors"]`, not an exception. |

## Patterns

- **Fixture-free isolation.** Every test builds its own SQLite DB under `tmp_path` (pytest's built-in temp-dir fixture). There is no shared `conftest.py` state for these tests—each one calls `api.init_db(db)` explicitly. This means the tests are fully order-independent.
- **Arrange → Act → Assert** with a single "act" call (`migrate_retract_to_defeaters`). No mocking, no patching—the real DB and real API are exercised end-to-end.
- **Idempotent ID derivation.** The defeater node ID is deterministic: `f"migrated-retraction-{node_id}-j0"`. Tests assert on this exact string, coupling the test to the ID scheme.
- **Dict-based result contract.** The API never raises for business-logic failures; it returns a structured dict. Tests assert on list lengths and key presence rather than catching exceptions (the sole exception is `test_error_on_missing_node`, which still checks the `errors` list, not an exception).
- **`sl=` (support-list) parameter.** `add_node("derived", ..., sl="base")` creates a directed edge: `base` supports `derived`. The defeater model inverts this—a defeater node *attacks* rather than supports.

## Dependencies

```
tests/test_migrate_defeaters.py
  ├── pytest          (test runner, tmp_path fixture)
  └── reasons.api     (the module under test: init_db, add_node, retract_node,
                        show_node, set_metadata, migrate_retract_to_defeaters)
```

Nothing else imports this file (it's a leaf test). Its **sibling** `tests/test_classify_defeaters.py` and `tests/test_defeat_justification.py` likely exercise the *new* defeater API directly; this file is specifically the *migration* bridge test.

## Flow

Each test follows the same control flow:

```
1. api.init_db(db)                          # fresh SQLite file
2. api.add_node("base", ...)                # create antecedent (IN)
3. api.add_node("derived", ..., sl="base")  # create dependent (IN, supported by base)
4. api.retract_node("derived", reason=...)  # flip to OUT, attach retract_reason
5. [optional] api.set_metadata / api.retract_node("base")   # extra setup
6. result = api.migrate_retract_to_defeaters(...)           # THE ACT
7. assert on result dict + api.show_node(...)              # verify post-state
```

The data transformation at step 6, for the happy path:

```
BEFORE:
  derived.truth_value = "OUT"
  derived.metadata["retract_reason"] = "Invalid reasoning"
  derived.metadata["_retracted"] = True

AFTER:
  derived.truth_value = "OUT"          (unchanged, but now *caused by* defeater)
  derived.metadata["retract_reason"]   (deleted)
  derived.metadata["_retracted"]       (deleted)

  NEW NODE:
    id = "migrated-retraction-derived-j0"
    truth_value = "IN"
    metadata["defeater_type"] = "migrated-retraction"
    metadata["defeats_node"] = "derived"
    (has an attack edge toward "derived")
```

## Invariants

The tests collectively enforce these rules:

1. **Candidate eligibility**: A node must be (a) OUT, (b) have `retract_reason` in metadata, AND (c) have a *satisfied* justification (its antecedents must be IN). Violating any one condition → skipped or ignored.
2. **Atomicity of migration**: When a node is migrated, *both* `retract_reason` and `_retracted` are cleared. You can't end up in a half-migrated state where the old metadata lingers.
3. **Reversibility**: The migration must not destroy information. Retracting the defeater node must restore the target to IN. This means the defeater is the *sole* cause of the OUT state post-migration.
4. **Dry-run purity**: `dry_run=True` must be a pure read. No node creation, no metadata mutation.
5. **Filter scoping**: When `node_ids` is provided, only those nodes are considered; the rest of the DB is untouched.
6. **Deterministic IDs**: The defeater ID is `migrated-retraction-{id}-j0`. The `-j0` suffix suggests it's tied to justification index 0 (the first/satisfying justification).

## Error Handling

The API uses a **result-object pattern** rather than exceptions for expected failure modes:

- **Unknown node in `node_ids`** → appended to `result["errors"]` with the offending `id`. No exception is raised. (`test_error_on_missing_node` confirms: `len(result["errors"]) == 1`.)
- **Unqualified candidates** (no `retract_reason`, unsatisfied justification) → appended to `result["skipped"]`. Not an error; a business-logic filter.
- **Actual I/O or schema errors** (e.g., corrupt DB) are not tested here and presumably would raise, since there's no `pytest.raises` anywhere in this file.

The `pytest` import is technically unused (no `pytest.raises`, no `pytest.mark`), likely a leftover or kept for future use.

---

## Topics to Explore

- [function] `reasons/api.py:migrate_retract_to_defeaters` — The implementation under test; understanding its internal traversal of the justification graph will clarify the "satisfied justification" check and the `-j0` ID suffix.
- [file] `tests/test_classify_defeaters.py` — Sibling test that exercises the *new* defeater classification logic; comparing the two files shows the old-vs-new model side by side.
- [file] `reasons/repair.py` — Likely contains the logic that *consumes* defeater nodes to restore IN state; `test_reversibility` in this file depends on that machinery.
- [general] `defeater-id-scheme` — The `migrated-retraction-{id}-j0` pattern encodes justification index; understanding whether `-j1`, `-j2` are possible (multiple justifications) affects migration correctness for multi-support nodes.
- [file] `tests/conftest.py` — Shared fixtures (if any) that set up common DB schemas or API stubs; relevant if you want to refactor these 8 tests into parameterized form.

## Beliefs

- `migrate-retract-clears-both-metadata-keys` — After a successful migration, both `retract_reason` and `_retracted` are removed from the target node's metadata; clearing only one is a bug.
- `dry-run-produces-no-writes` — `migrate_retract_to_defeaters(dry_run=True)` must not create any defeater node nor mutate any existing node's metadata; it is a pure read operation.
- `defeater-id-is-deterministic` — The defeater node ID for a migrated retraction is always `f"migrated-retraction-{source_id}-j0"`, making it safe to reference in `show_node` calls and in reversal logic.
- `satisfied-justification-required-for-migration` — A node whose antecedents are themselves OUT is skipped (not migrated), because its OUT state is already explained by the collapsed support chain, not by a retraction reason.
- `migration-preserves-reversibility` — Retracting the newly created defeater node must restore the target node to IN, meaning the defeater is the *exclusive* causal reason for the target's OUT state after migration.
