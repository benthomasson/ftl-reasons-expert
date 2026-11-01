---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_import_agent.py

# `tests/test_import_agent.py`

## Purpose

This file is the integration test suite for the **multi-agent belief import** feature. It validates the `api.import_agent()` function, which takes a serialized belief network (either Markdown or JSON) and ingests it into the reasons database under a unique agent namespace. The core responsibility it owns is verifying that:

- Beliefs are correctly namespaced per agent (e.g., `test-agent:alpha-fact`)
- A synthetic "active premise" node is created as the root dependency for all imported beliefs
- Dependencies and outlists are remapped from raw IDs to namespaced IDs
- RETRACTION cascades properly when the active premise is retracted
- OUT-status beliefs from the source are preserved (not resurrected by recomputation)
- Multiple agents coexist independently in the same database
- Duplicate imports are idempotent

## Key Components

### Fixtures

| Fixture | Contract |
|---------|----------|
| `db(tmp_path)` | Initializes a fresh SQLite database via `api.init_db()`, yields the path. Each test gets isolation. |
| `beliefs_file(tmp_path)` | Writes `SAMPLE_BELIEFS` and `SAMPLE_NOGOODS` to disk, yields the Markdown path. |

### Sample Data

- **`SAMPLE_BELIEFS`** — A 3-belief network: `alpha-fact` (IN, observation), `beta-depends-alpha` (IN, derived from alpha), `gamma-stale` (STALE, observation). Exercises the IN/OUT/STALE truth-value spectrum.
- **`SAMPLE_NOGOODS`** — A single nogood that references `gamma-stale`.

### Tests (grouped by concern)

| Test | What it verifies |
|------|-----------------|
| `test_import_agent_basic` | Return-dict shape: agent name, prefix, active node, counts |
| `test_import_agent_creates_premise` | The synthetic `:active` node has correct metadata (`agent`, `role=agent_premise`) |
| `test_import_agent_namespaces_beliefs` | Namespacing and `original_id` metadata preservation |
| `test_import_agent_remaps_dependencies` | Antecedent remapping + inactive-kill-switch in outlist |
| `test_import_agent_retract_premise_cascades` | `what_if_retract` (dry-run) reports correct affected count |
| `test_import_agent_retract_premise_actually_cascades` | `retract_node` actually flips all dependents to OUT |
| `test_import_agent_restore_premise_cascades` | `assert_node` on the premise restores dependents to IN |
| `test_import_agent_skip_duplicates` | Second import is a no-op (0 imported, 3 skipped, no new premise) |
| `test_import_agent_only_in` | `only_in=True` filter excludes non-IN beliefs |
| `test_import_multiple_agents` | Two agents share a DB without interference; retraction is scoped |
| `test_import_agent_preserves_out_status` | OUT beliefs stay OUT after import (not re-derived) |
| `test_import_agent_preserves_outlist` | Unless/outlist relationships are namespaced and enforced |
| `test_import_agent_supersession_preserved` | Superseded (OUT) beliefs keep their inactive-gate justification |
| `test_import_agent_json` | JSON import path preserves full justification structure |
| `test_import_agent_json_outlist_blocks` | JSON path: IN blocker correctly gates a belief to OUT |
| `test_retracted_belief_survives_propagate` | **Regression (issue #16)**: retracted beliefs don't resurrect on `recompute_all` |
| `test_retracted_json_belief_survives_propagate` | Same regression via JSON path |
| `test_out_with_satisfied_justifications_stays_out` | OUT node with satisfied antecedents must not flip to IN |
| `test_import_agent_registers_repo` | Agent name is registered in the repos table with file path |
| `test_import_agent_nogoods` | Nogoods from the source are imported |

## Patterns

1. **Namespace-as-isolation**: The fundamental pattern tested is that `import_agent` creates a self-contained sub-graph rooted at `<agent>:active`. All beliefs depend on this premise (or on sibling beliefs), and retracting the premise kills the entire sub-graph atomically. The outlist on every justification includes `<agent>:inactive` as a kill switch.

2. **Dry-run vs. mutation**: `what_if_retract` is tested as a pure query (returns affected count), while `retract_node` is tested as a mutation. This separation is verified in separate tests.

3. **Idempotency**: `test_import_agent_skip_duplicates` ensures calling import twice doesn't create duplicate nodes or a second premise.

4. **Dual-format parity**: Several scenarios are tested via both Markdown (`beliefs_file` fixture) and JSON (inline `tmp_path` files). The JSON tests explicitly verify that the `justifications` array (with `antecedents`, `outlist`, `type`, `label`) round-trips correctly.

5. **Regression-as-test**: `test_retracted_belief_survives_propagate` and `test_out_with_satisfied_justifications_stays_out` encode specific bug fixes (issue #16) as permanent guards. They call `recompute_all()` directly via `Storage` to simulate the propagation engine.

6. **Isolated databases**: Every test gets a fresh `tmp_path` database. No shared state between tests. The `db` fixture is function-scoped by default in pytest.

## Dependencies

**This file imports:**
- `reasons.api` — the public API surface (`init_db`, `import_agent`, `show_node`, `retract_node`, `assert_node`, `what_if_retract`, `get_status`, `list_repos`)
- `reasons.storage.Storage` — used in regression tests to trigger `recompute_all()`
- `pytest` — fixture and assertion framework
- `json`, `os`, `tempfile`, `pathlib.Path` — file I/O and path manipulation

**Depends on (indirectly, via `api`):**
- `reasons.import_agent` — the implementation under test
- `reasons.storage` — persistence layer
- `reasons.network` — the belief network data structure
- `reasons.merkle` — likely involved in identity/dedup

**Imported by:** Nothing in the repo imports this test file (it's a leaf test module).

## Flow

```
import_agent(agent_name, file_path, db_path, only_in?)
  │
  ├─ Parse file (Markdown or JSON based on extension/content)
  │
  ├─ For each belief:
  │    ├─ Namespace the ID: f"{agent}:{original_id}"
  │    ├─ Remap antecedents: f"{agent}:{ant}" for each
  │    ├─ Remap outlist entries similarly
  │    ├─ Add f"{agent}:inactive" to every outlist (kill switch)
  │    ├─ Add f"{agent}:active" as an antecedent (root dependency)
  │    └─ Preserve original truth_value (IN stays IN, OUT stays OUT)
  │
  ├─ Create `<agent>:active` premise node (role=agent_premise)
  ├─ Create `<agent>:inactive` node (used in outlists)
  │
  ├─ Import nogoods (if present)
  ├─ Register repo path
  │
  └─ Return summary dict
```

The tests verify each step of this pipeline by inspecting the resulting database state via `api.show_node()`.

## Invariants

1. **Namespace uniqueness**: `<agent>:<id>` is globally unique. Two agents can both have `alpha-fact` without collision (verified in `test_import_multiple_agents`).

2. **Kill-switch completeness**: Every justification's outlist must contain `<agent>:inactive`. This ensures that retracting the active premise (which asserts inactive) cascades to ALL beliefs, regardless of their individual antecedent satisfaction.

3. **OUT is sticky**: If a belief is OUT in the source, it remains OUT after import. The recomputation engine (`recompute_all`) must not resurrect it, even if its antecedents are all IN. This is the issue #16 invariant.

4. **Idempotency**: Re-importing the same file for the same agent results in 0 new nodes. The premise is not recreated.

5. **Scoping**: Retraction of one agent's premise has zero effect on another agent's beliefs. The database is shared but the sub-graphs are logically isolated.

## Error Handling

This test file doesn't test error *production* (e.g., malformed input, missing files) — it assumes valid inputs. However, it does verify:

- **`pytest.raises(KeyError)`** in `test_import_agent_only_in`: asserting that a node *doesn't exist* after a filtered import. This is the negative case.
- **Return-dict contract**: The tests rely on `import_agent` always returning a dict with specific keys (`agent`, `prefix`, `active_node`, `created_premise`, `claims_imported`, `claims_retracted`, `claims_skipped`, `nogoods_imported`). If a key is missing, the test fails with `KeyError`.
- **`retract_node` / `assert_node` return shape**: Both return a dict with a `"changed"` list of node IDs. The tests assert on list membership and length.

There's no explicit error-path testing (e.g., what happens with a non-existent file, a malformed Markdown structure, or a JSON with missing `nodes` key). These would likely be covered elsewhere or are implicit preconditions of the API.

## Topics to Explore

- [function] `reasons/import_agent.py:import_agent` — The implementation under test; understand how it parses, namespaces, and wires the active/inactive premise
- [file] `reasons/network.py` — The belief network data structure; needed to understand how `recompute_all` decides IN vs. OUT and why the kill-switch outlist is sufficient
- [file] `reasons/storage.py` — The `Storage` class used in regression tests; understand the `load()` → `recompute_all()` → `save()` cycle and what "propagate" means here
- [diff] `tests/test_sync_agent.py` vs `tests/test_import_agent.py` — How does "sync" differ from "import"? Both deal with agent belief management but likely have different semantics (update-in-place vs. one-shot ingest)
- [general] `issue-16-regression` — The specific bug where `active` as an antecedent created an always-valid fallback justification; understanding the fix helps prevent similar issues in other propagation paths

## Beliefs

- `import-agent-creates-kill-switch` — Every justification imported by `import_agent` includes `<agent>:inactive` in its outlist, making the entire sub-graph retractable via a single premise retraction
- `out-status-survives-recompute` — A belief marked OUT in the source snapshot remains OUT after `recompute_all()`, even if all its antecedents are IN and its outlist is empty
- `import-agent-is-idempotent` — Calling `import_agent` twice with the same agent name and file produces zero new nodes on the second call and does not recreate the active premise
- `agent-namespace-isolates-retraction` — Retracting `<agent-a>:active` changes no nodes belonging to `<agent-b>`, even when both agents imported the identical belief set
- `import-agent-registers-repo-path` — `import_agent` writes the agent name and resolved parent directory of the source file into the repos table, linking the agent to its origin repository
