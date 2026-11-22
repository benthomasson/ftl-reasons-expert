---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How the two `_entrenchment` implementations are kept in sync, and whether any test asserts the Postgres path's ordering.

## How the two `_entrenchment` implementations are related

There is **no visible mechanism that keeps the two `_entrenchment` implementations in sync**. They are separate methods:

- In-memory implementation: `reasons/network.py:291`
  ```python
  def _entrenchment(self, node_id: str) -> int:
  ```
- Postgres implementation: `reasons/pg.py:3328`
  ```python
  def _entrenchment(self, cur, node_id):
  ```

The Postgres method takes an active DB cursor, while the in-memory method only takes a node ID. That difference is expected because the PG path must query node and justification data from Postgres via `cur`.

However, `reasons/pg.py` does **not** import a shared entrenchment helper from `reasons/network.py` or another module. Its visible imports are only stdlib modules: `json`, `re`, `collections.deque`, `datetime`, and `itertools.combinations`. So the two implementations appear to be **parallel, manually maintained logic**, not generated from or delegated to a common function.

The exact scoring bodies are not included in the observations, so we cannot verify whether the rules currently match. The network tests describe expected ordering rules, but the PG method’s body is outside the returned context.

## Test coverage for Postgres ordering

Direct tests of `Network._entrenchment` exist in `tests/test_backtracking.py`:

- `tests/test_backtracking.py:230` asserts `evidence > basis`
- `tests/test_backtracking.py:259` asserts `sourced > unsourced`
- `tests/test_backtracking.py:266` asserts `obs > der`
- `tests/test_backtracking.py:275` asserts `popular > lonely`

Those tests call `net._entrenchment(...)` on an in-memory `Network`, not on the PG API.

For Postgres, there is **no direct test** shown that calls `pg_api._entrenchment(...)` or compares PG entrenchment values. There is, however, one **indirect** assertion in `tests/test_pg.py`:

- `TestFindCulprits.test_find_culprits` creates `premise-a` with `source="code.py"` and `premise-b` without a source.
- It asserts:
  ```python
  assert result["culprits"][0]["premise"] == "premise-b"
  ```
- `find_culprits` calls:
  ```python
  entrenchment = self._entrenchment(cur, premise_id)
  ```
  at `reasons/pg.py:3316`.

So that test **indirectly exercises the Postgres ordering rule** that an unsourced premise is less entrenched than a sourced premise. It does not cover the other ordering rules exercised for the in-memory path, such as `evidence > basis`, `obs > der`, or `popular > lonely`.

One observation is contradictory: `search_tests_for_entrenchment_ordering` returned zero matches, but `find_entrenchment_definitions` lists direct `_entrenchment` assertions in `tests/test_backtracking.py`. The raw test excerpt for `tests/test_pg.py` is more reliable for the PG finding above.

## Summary

- The two `_entrenchment` implementations are **not kept in sync by any visible shared helper or import**; they appear to be duplicated logic, likely maintained manually.
- The in-memory implementation is tested directly in `tests/test_backtracking.py`.
- The Postgres implementation is only tested **indirectly** through `TestFindCulprits.test_find_culprits`, which verifies one source-based ordering outcome via `find_culprits`.
- The full PG `_entrenchment` scoring body and the full network `_entrenchment` scoring body are missing from the observations, so exact parity cannot be confirmed.

## Topics to Explore

- [function] `reasons/network.py:_entrenchment` — Inspect the exact scoring rules for the in-memory implementation.
- [function] `reasons/pg.py:_entrenchment` — Inspect the SQL-backed scoring logic and compare it rule-by-rule with the in-memory version.
- [function] `reasons/pg.py:find_culprits` — Understand how PG entrenchment is consumed and whether ordering affects culprit selection.
- [file] `tests/test_pg.py` — Search for additional indirect assertions involving entrenchment-sensitive Postgres operations.
- [general] `entrenchment-parity-tests` — Consider whether direct parity tests between `Network._entrenchment` and `PgApi._entrenchment` are needed.

## Beliefs

- `pg-entrenchment-takes-cursor` — `PgApi._entrenchment` requires a Postgres cursor and node ID, unlike `Network._entrenchment`, which only takes a node ID.
- `no-shared-entrenchment-helper-visible` — `reasons/pg.py` imports no shared entrenchment helper from `reasons/network.py`, so the two methods are separate implementations.
- `network-entrenchment-tested-directly` — `tests/test_backtracking.py` calls `Network._entrenchment` directly and asserts multiple ordering relationships.
- `pg-entrenchment-tested-indirectly` — `tests/test_pg.py` `TestFindCulprits.test_find_culprits` indirectly asserts that an unsourced premise is less entrenched than a sourced premise.
- `pg-entrenchment-rule-coverage-limited` — The shown PG tests do not directly assert the full set of entrenchment ordering rules covered by the network tests.
