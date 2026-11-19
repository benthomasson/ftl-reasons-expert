---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_migrate_defeaters.py

# tests/test_migrate_defeaters.py

## Purpose

This is a **pytest integration test module** that validates the `reasons.api.migrate_retract_to_defeaters(...)` migration command. It exercises the transitional path that converts a *legacy* way of defeating a belief — a node that has been set to `OUT` via `retract_node(...)` (stored as `retract_reason` / `_retracted` metadata plus an outlist entry) — into a first-class **defeater node** whose presence undercuts the target belief.

The module's responsibility is purely to verify:
- which nodes are (and are not) candidates,
- what artifacts the migration writes to the graph,
- how state is cleaned up or preserved,
- that the migration is reversible, filterable, and dry-runnable,
- and how missing input is reported.

It does not own any implementation — it is spec-with-fixtures for the API surface.

## Key Components

All exercised behavior lives in `reasons.api`. The test file references:

| Symbol | Usage in these tests |
|---|---|
| `api.init_db(db_path)` | Create the SQLite database at a `tmp_path` location |
| `api.add_node(id, statement, sl=..., db_path=...)` | Create nodes; `sl` (support link) makes a node a *derived* node from a premise; without `sl` it's an axiom that starts `IN` |
| `api.show_node(id, db_path=...)` | Return a dict with at least `truth_value` (`"IN"`/`"OUT"`) and `metadata` (dict) |
| `api.retract_node(id, reason=..., db_path=...)` | Flip a node to `OUT`, storing `retract_reason` and `_retracted` in metadata |
| `api.set_metadata(id, key, value, db_path=...)` | Set arbitrary metadata (used to inject `retract_reason` onto a non-retracted node) |
| `api.migrate_retract_to_defeaters(node_ids=..., dry_run=..., db_path=...)` | The subject under test; returns a structured result dict |

**Return contract of `migrate_retract_to_defeaters`** (inferred from all tests):

```python
{
  "migrated": [   # one entry per successfully migrated node
      {"id": "<original node id>", "defeater_id": "migrated-retraction-<id>-j0"},
      # ...but in dry_run=True: defeater_id is absent
  ],
  "skipped": [{"id": "<node id>"}, ...],   # candidates rejected by policy
  "errors":  [{"id": "<node id>"}, ...],   # unknown node_ids
}
```

**Defeater-ID scheme** is deterministic: `migrated-retraction-{node_id}-j0`. The `-j0` suffix strongly implies a justification index (0-based), so a node with multiple justifications could yield `-j1`, `-j2`, … — even though no test covers that case.

**Defeater-node metadata contract**, verified in `test_basic_migration`:
- `metadata["defeater_type"] == "migrated-retraction"`
- `metadata["defeats_node"] == "<original node id>"`

## Patterns

- **Fixture-isolated integration tests.** Each test creates its own SQLite file via pytest's built-in `tmp_path` fixture, runs a real graph through `reasons.api`, and asserts on the resulting state. **No mocking** — this tests the storage layer end-to-end.
- **Arrange–Act–Assert** structure used consistently in all eight tests.
- **One behavior per test**, named `test_<behavior>` with a one-line docstring that states the invariant (e.g., "OUT node where antecedent is also OUT should be skipped").
- **Black-box API testing** — the tests speak only in `api.*` vocabulary, so if the underlying storage schema changes, the tests stay valid.
- **State-verification via re-read** — after every mutation the test calls `api.show_node(...)` again to prove persistence, rather than trusting the mutation's return value.

## Dependencies

**Imports (this file → others):**
- `pytest` — for the `tmp_path` fixture, though `import pytest` itself is technically unused (no `pytest.raises`, no markers, no parametrize). It is present for convention/expansion.
- `from reasons import api` — the single module under test. Note the repo also has a parallel `reasons_lib/` package; these tests pin behavior to the `reasons` package (presumably the canonical one, with `reasons_lib` being a vendored mirror).

**Imported by (others → this file):**
- Nothing. It's a leaf-level pytest module collected by the test runner. There is no `__init__`-level re-export in `tests/` that pulls it in.

## Flow

A canonical happy-path execution (`test_basic_migration`):

1. `api.init_db(db)` — creates schema at `<tmp>/test.db`.
2. `add_node("base", ...)` — a root belief with no `sl`, therefore satisfiable → `IN`.
3. `add_node("derived", "Derived belief", sl="base", ...)` — justification is `base`, and `base` is `IN`, so `derived` is also `IN`.
4. `retract_node("derived", reason="Invalid reasoning")` — flips `derived` to `OUT`, stores `retract_reason` and `_retracted` metadata (this is the *legacy* representation the migration is meant to replace).
5. `migrate_retract_to_defeaters(dry_run=False)`:
   - **Candidate scan**: find nodes where `truth_value == "OUT"` *and* `"retract_reason"` in metadata *and* the node's justification is otherwise satisfiable (antecedent is `IN`).
   - **For each candidate** (deterministic ID rooted at the original node):
     - create a new node `migrated-retraction-derived-j0`,
     - annotate its metadata with `defeater_type = "migrated-retraction"` and `defeats_node = "derived"`,
     - remove `retract_reason` and `_retracted` from the original node's metadata,
     - leave the original node `OUT` (the defeater now supplies the reason).
   - **Return** `{migrated: [...], skipped: [...], errors: [...]}`.
6. Assertions confirm the "before/during/after" states.

**`test_reversibility`** demonstrates a second-order flow: after migration, calling `retract_node("migrated-retraction-derived-j0")` again on the defeater causes `derived` to flip back to `IN`. That means:
- a defeater node obeys the same `retract_node` semantics as any other node, and
- once the defeater is `OUT`, its effect on `derived` is neutralized (the defeater no longer defeats).

**Dry-run branch** short-circuits step 5's writes: candidates are returned without `defeater_id` and *without* mutating the original `retract_reason`.

**`node_ids` filter branch** narrows candidates to the intersection of the scan set and the provided list; non-listed candidates (e.g., `d2`) keep their `retract_reason`.

**Missing node branch** (`test_error_on_missing_node`) sends `node_ids=["nonexistent"]` against an empty DB. The function does not raise; it returns `errors=[{"id": "nonexistent"}]` with `migrated=[]` and presumably `skipped=[]`.

## Invariants

These are the rules the tests collectively pin down:

1. **Candidate precondition (the hard one)** — A node is migratable only if *all* of:
   - `truth_value == "OUT"`,
   - `metadata["retract_reason"]` is present,
   - the node's justification antecedent is `IN` (i.e., the node would be `IN` absent the retraction).

   Violating any one of these puts the node in `skipped` (or excludes it entirely, if there is no `retract_reason` at all).

   The distinction between the two skip cases:
   - `retract_reason` present but antecedent `OUT` → row in `skipped` (`test_skip_unsatisfied_justification`).
   - no `retract_reason` at all → simply not a candidate, *nothing* in `skipped` (`test_skip_no_retract_reason`).

2. **Deterministic defeater ID** — `migrated-retraction-{node_id}-j{index}`; index `0` for the first justification.

3. **Metadata cleanup contract** — After a successful non-dry-run migration:
   - `"retract_reason"` is removed from the target node,
   - `"_retracted"` is removed from the target node.

4. **Post-migration truth value preserved** — The target stays `OUT` immediately after migration; the ground truth doesn't change, only the representation.

5. **Reversibility** — Retracting the migrated defeater restores the target to `IN`. This is tested explicitly and is the core "no data lost" property of the migration.

6. **Dry-run purity** — `dry_run=True` must not write anything: no defeater node, no metadata cleanup, and no `defeater_id` key in each candidate entry.

7. **Filter scoping** — When `node_ids` is supplied, only those IDs may be migrated; others keep their `retract_reason`.

8. **Missing-node isolation** — Unknown IDs are non-fatal; they appear under `errors` and do not prevent other migrations.

## Error Handling

The migration is deliberately **non-throwing** for user-input errors:

- Unknown `node_ids` → entry in `result["errors"]`, not an exception (`test_error_on_missing_node`). Callers can iterate `errors` to build a user-facing report — consistent with the repo's general convention (see `reasons/verify.py`, `reasons/repair.py`) of returning structured result dicts instead of exceptions.
- Skip conditions (unsatisfied antecedent, missing retract reason) are **informational**, surfaced in `result["skipped"]` only when the node was arguable-but-rejected.
- True environment failures (DB connection, schema errors) are **not covered by these tests** — they're outside the module's contract and would presumably propagate from `init_db` / `add_node` / `show_node`.

The tests themselves have **no negative-path assertions using `pytest.raises`**, which is itself a signal: the API is expected to always return a value here.

## Topics to Explore

- [function] `reasons/api.py:migrate_retract_to_defeaters` — Read the actual candidate-scan and defeater-creation logic to see how the `j0` index is computed for multi-justification nodes and how `_retracted` is set by `retract_node`.
- [function] `reasons/api.py:retract_node` — Understand the second-order effect used in `test_reversibility`: why does retracting a defeater restore the target to `IN`?
- [file] `tests/test_classify_defeaters.py` — Sibling test that likely defines the `defeater_type` taxonomy this migration outputs into (`"migrated-retraction"`).
- [file] `tests/test_defeat_justification.py` — Shows the *target* end-state that migration produces (a real defeater node undercutting a justification), so you can compare legacy vs. native representation.
- [general] `outlist-semantics` — The `retract_reason` / `_retracted` legacy representation and the `outlist` mechanism implied by the test docstrings; understanding both clarifies why the migration exists.

## Beliefs

- `migrate-returns-structured-result` — `migrate_retract_to_defeaters` always returns a dict with `migrated`, `skipped`, and `errors` keys and never raises on unknown `node_ids`.
- `migration-preserves-truth-value` — After migration, the target node's `truth_value` remains `"OUT"`; only its metadata and a new defeater node change.
- `migration-requires-satisfied-justification` — A node is only migrated if its justification antecedent is `IN`, enforced by `test_skip_unsatisfied_justification`.
- `migrated-defeater-id-format-is-deterministic` — Defeater IDs follow `migrated-retraction-{node_id}-j{index}`; `-j0` for the first justification.
- `dry-run-prevents-all-writes` — `dry_run=True` causes `migrate_retract_to_defeaters` to compute candidates without creating defeater nodes, omitting `defeater_id` from results and leaving `retract_reason` intact.
- `migration-is-reversible` — Retracting a migrated defeater (`retract_node("migrated-retraction-...")`) flips the original target back to `IN`.
- `migration-clears-legacy-metadata` — Both `retract_reason` and `_retracted` are removed from the target on successful migration.
