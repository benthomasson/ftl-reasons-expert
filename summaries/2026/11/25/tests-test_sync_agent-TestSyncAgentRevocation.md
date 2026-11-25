---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: TestSyncAgentRevocation in tests/test_sync_agent.py

## Purpose

`TestSyncAgentRevocation` is a pytest test class containing two regression tests for `api.sync_agent`. It verifies that syncing an agent’s beliefs does not break the truth-maintenance links between those beliefs and the agent’s `active`/`inactive` sentinel nodes.

The important regression is: a no-op or restorative sync must not detach existing beliefs from their justifications, because those justifications include the agent’s inactive node in `outlist`. If sync corrupts those justifications, later revoking `test-agent:active` will fail to cascade beliefs to `OUT`.

## Contract

### Preconditions

- The `initial_import` fixture has already:
  - initialized a temporary database via `api.init_db`
  - written `INITIAL_BELIEFS` to `tmp_path / "beliefs.md"`
  - imported those beliefs under agent `"test-agent"` via `api.import_agent`
- The database contains the expected agent-scoped nodes, including:
  - `test-agent:alpha-fact`
  - `test-agent:beta-depends-alpha`
  - `test-agent:gamma-stale`
  - sentinel nodes such as `test-agent:active` and `test-agent:inactive`
- `tmp_path` is writable.
- `INITIAL_BELIEFS` describes `alpha-fact` as `IN`, `beta-depends-alpha` as `IN` depending on `alpha-fact`, and `gamma-stale` as effectively `OUT` because it is `STALE`.

### Postconditions

For `test_synced_beliefs_still_cascade_on_agent_revocation`:

- After a no-op sync, at least one justification on `test-agent:alpha-fact` still contains `"test-agent:inactive"` in its `outlist`.
- After `api.retract_node("test-agent:active", db_path=db)`:
  - `test-agent:alpha-fact` has `truth_value == "OUT"`.
  - `test-agent:beta-depends-alpha` has `truth_value == "OUT"`.

For `test_synced_restored_belief_still_cascades`:

- Locally retracting `test-agent:alpha-fact` makes it `OUT`.
- Syncing with the original remote file restores it to `IN`.
- After retracting `test-agent:active`, the restored `test-agent:alpha-fact` goes back to `OUT`.

### Invariants

- Agent beliefs are linked to the agent lifecycle through justifications involving the agent’s active/inactive nodes.
- Revoking the agent’s active node must cascade through the TMS and invalidate beliefs whose justifications depend on that active node.
- `sync_agent` must preserve existing justification structures when the remote content is unchanged or when it restores a locally retracted belief.
- Node IDs use the `agent-name:belief-name` convention.
- `show_node` returns a mapping with at least `truth_value` and `justifications`, and each justification mapping may contain `outlist`.

### Assumptions not enforced by the type system

- `initial_import` returns exactly a two-tuple `(db_path, tmp_path)`.
- `show_node` always returns a dictionary with `truth_value` and `justifications` keys for existing nodes.
- `justifications` is an iterable of dictionaries, each with an `outlist` key when relevant.
- `test-agent:active` and `test-agent:inactive` exist after import.
- The string `"test-agent:inactive"` is the exact node ID stored in `outlist`.
- `retract_node("test-agent:active")` has the semantic effect of revoking the agent and cascading all agent beliefs out.
- `sync_agent` with the same content exercises the update path but does not change belief state.

## Parameters

Each test method takes:

- `self` — the pytest test instance. It is not used directly.
- `initial_import` — a pytest fixture returning `(db, tmp_path)`:
  - `db`: a string path to the initialized temporary SQLite database.
  - `tmp_path`: a `pathlib.Path` to the test’s temporary directory.

The tests then create/write `tmp_path / "beliefs.md"` and call API functions with `db_path=db`.

Edge cases implied by the tests:

- If `initial_import` does not create the expected nodes, later `show_node` or `retract_node` calls fail.
- If the file content differs from `INITIAL_BELIEFS`, the sync is no longer a no-op/restore scenario.
- If the agent name does not match `"test-agent"`, node IDs will not match.

## Return Value

Test methods return `None`. pytest uses assertions to determine success or failure.

The tests ignore the dictionary returned by `api.sync_agent`; they validate the resulting database state instead.

## Algorithm

### `test_synced_beliefs_still_cascade_on_agent_revocation`

1. Unpack `(db, tmp_path)` from `initial_import`.
2. Write `INITIAL_BELIEFS` to `tmp_path / "beliefs.md"`.
3. Call `api.sync_agent("test-agent", str(p), db_path=db)`.
   - The remote file is unchanged, so this is intended as a no-op sync that still exercises the update path.
4. Fetch `test-agent:alpha-fact` via `api.show_node`.
5. Assert that at least one justification for `alpha-fact` has `"test-agent:inactive"` in its `outlist`.
   - This checks that sync did not strip or replace the justification that links the belief to the agent lifecycle.
6. Call `api.retract_node("test-agent:active", db_path=db)` to revoke the agent.
7. Fetch `test-agent:alpha-fact` and assert its truth value is `OUT`.
8. Fetch `test-agent:beta-depends-alpha` and assert its truth value is `OUT`.
   - Since `beta-depends-alpha` depends on `alpha-fact`, it should cascade out after `alpha-fact` becomes `OUT`.

### `test_synced_restored_belief_still_cascades`

1. Unpack `(db, tmp_path)` from `initial_import`.
2. Locally retract `test-agent:alpha-fact` via `api.retract_node`.
3. Write `INITIAL_BELIEFS` back to `tmp_path / "beliefs.md"`.
4. Call `api.sync_agent("test-agent", str(p), db_path=db)`.
   - The remote file says `alpha-fact` is `IN`, so sync should restore the locally retracted belief.
5. Fetch `test-agent:alpha-fact` and assert it is `IN`.
6. Call `api.retract_node("test-agent:active", db_path=db)` to revoke the agent.
7. Fetch `test-agent:alpha-fact` and assert it is `OUT`.
   - This proves that the restored belief still has the correct justification linkage and still cascades when the agent is revoked.

## Side Effects

- The fixture initializes a temporary database and imports the initial beliefs.
- Each test writes `INITIAL_BELIEFS` to a temporary Markdown file.
- `api.sync_agent` reads that file and mutates database state.
- `api.retract_node` mutates truth values and triggers TMS cascade behavior.
- `api.show_node` is read-only from the test’s perspective.
- All mutations are confined to pytest’s temporary database and temporary directory.

## Error Handling

There is no explicit error handling.

- Assertion failures raise `AssertionError` and fail the test.
- Missing nodes, malformed returned dictionaries, or unexpected API responses can cause `KeyError`, `TypeError`, or API-specific exceptions.
- File write failures can raise `OSError`.
- Database/API failures propagate and fail the test rather than being swallowed.

## Usage Patterns

These tests are run through pytest, not called directly by application code. They rely on fixtures for isolation and should be extended by adding new test methods inside the class or nearby test classes.

When adding similar tests:

- Use `initial_import` to get a fresh imported agent/database.
- Rewrite or modify the remote file under `tmp_path`.
- Call `api.sync_agent`.
- Inspect nodes with `api.show_node`.
- Use `api.retract_node` to trigger cascades.
- Assert final `truth_value` and justification structure.

## Dependencies

- `pytest`
- `reasons.api`
- Fixtures:
  - `db`
  - `initial_import`
- Constant:
  - `INITIAL_BELIEFS`
- API functions used:
  - `api.sync_agent`
  - `api.show_node`
  - `api.retract_node`
- Temporary filesystem via pytest `tmp_path`

## Topics to Explore

- [function] `reasons/api.py:sync_agent` — The sync update path, especially whether it calls `assert_node` and how it preserves or replaces justification lists.
- [function] `reasons/api.py:retract_node` — How retraction triggers truth-maintenance cascade behavior and how active/inactive sentinel nodes participate.
- [function] `reasons/api.py:import_agent` — How initial import creates `test-agent:active`, `test-agent:inactive`, and attaches them to belief justifications.
- [general] `jtms-justification-inlist-outlist` — The semantics of antecedents/inlist and outlist in justifications and why `inactive` appears in `outlist`.
- [file] `tests/test_sync_agent.py` — The broader sync test suite covering truth transitions, dependency updates, idempotency, and JSON sync.

## Beliefs

- `sync-preserves-inactive-outlist` — A no-op `sync_agent` on unchanged `INITIAL_BELIEFS` leaves at least one `alpha-fact` justification with `test-agent:inactive` in its `outlist`.
- `retracting-active-cascades-alpha-out` — Retracting `test-agent:active` sets `test-agent:alpha-fact` to `OUT`.
- `retracting-active-cascades-beta-out` — Retracting `test-agent:active` sets `test-agent:beta-depends-alpha` to `OUT` because it depends on `alpha-fact`.
- `sync-restores-locally-retracted-alpha` — When `alpha-fact` is locally retracted and `sync_agent` reads `INITIAL_BELIEFS` where it is `IN`, `alpha-fact` becomes `IN`.
- `restored-alpha-still-cascades-on-revocation` — After sync restores `alpha-fact`, retracting `test-agent:active` sets `alpha-fact` to `OUT`.
