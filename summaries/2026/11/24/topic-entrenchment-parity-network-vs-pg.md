---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Verify whether the two `_entrenchment` implementations produce identical scores for the same node data.

# Verification Result

The observations are **not sufficient to verify** that the two `_entrenchment` implementations produce identical scores for the same node data. The codebase snapshot identifies both definitions and their call sites, but it does **not include the function bodies**, so the scoring formulas cannot be compared.

What the observations do show:

- `reasons/network.py:291` defines:
  ```python
  def _entrenchment(self, node_id: str) -> int:
  ```
- `reasons/pg.py:3328` defines:
  ```python
  def _entrenchment(self, cur, node_id):
  ```
- `reasons/network.py:368` calls the network version as:
  ```python
  entrenchment = self._entrenchment(premise_id)
  ```
- `reasons/pg.py:3316` calls the Postgres version as:
  ```python
  entrenchment = self._entrenchment(cur, premise_id)
  ```
- Both call sites feed the result into a candidate record and sort by `"entrenchment"`:
  - `reasons/network.py:368-377`
  - `reasons/pg.py:3316-3325`
- The only observed direct tests of `_entrenchment` call the network implementation through `net._entrenchment(...)`:
  - `tests/test_backtracking.py:230`
  - `tests/test_backtracking.py:259`
  - `tests/test_backtracking.py:266`
  - `tests/test_backtracking.py:275`

There are **no observed tests calling the Postgres `_entrenchment` implementation**, and there is no observed parity test comparing both implementations on the same node data.

The different signatures suggest different data-access strategies: the network implementation takes only `node_id`, while the Postgres implementation also requires a database cursor `cur`. That does not by itself prove the scores differ, but it means parity depends on whether the Postgres query/loading logic supplies exactly the same node attributes, belief types, premise relationships, and scoring rules as the in-memory network model.

To actually verify parity, you would need to compare:

1. The full source of `reasons/network.py:_entrenchment`.
2. The full source of `reasons/pg.py:_entrenchment`.
3. The schema/query used by the Postgres version.
4. The in-memory node data structure used by the network version.
5. The type/category mappings and weight constants used by both.
6. Behavior for missing or null node fields.
7. A test fixture that runs the same logical node through both backends and asserts equal scores.

## Topics to Explore

- [function] `reasons/network.py:_entrenchment` — Inspect the exact scoring formula, type mappings, defaults, and precedence rules used by the in-memory implementation.
- [function] `reasons/pg.py:_entrenchment` — Inspect the SQL/cursor-based data loading and scoring logic to identify parity risks with the network implementation.
- [file] `tests/test_backtracking.py` — Review existing entrenchment tests; they currently appear to exercise only the network backend.
- [general] `entrenchment-parity-test` — Add a shared fixture that supplies identical node data to both implementations and asserts equal scores.
- [general] `node-data-model` — Compare the in-memory node representation in `reasons/network.py` with the persisted Postgres representation in `reasons/pg.py`.

## Beliefs

- `two-entrenchment-definitions-exist` — Both `reasons/network.py` and `reasons/pg.py` define `_entrenchment`; the Postgres version additionally requires a `cur` argument.
- `network-entrenchment-is-directly-tested` — `tests/test_backtracking.py` directly calls `net._entrenchment(...)` at lines 230, 259, 266, and 275.
- `pg-entrenchment-has-no-observed-direct-tests` — The gathered observations show no test calling the Postgres `_entrenchment` implementation.
- `entrenchment-controls-resolution-candidate-order` — Both implementations store an `"entrenchment"` value on resolution candidates and sort candidates by that field.
- `entrenchment-parity-is-unproven` — The observations contain no function bodies or cross-backend assertions, so identical scores for the same node data are not demonstrated.
