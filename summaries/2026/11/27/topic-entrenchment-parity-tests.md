---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Consider whether direct parity tests between `Network._entrenchment` and `PgApi._entrenchment` are needed.

## Assessment

Short answer: **yes, direct parity tests are warranted**, unless you first refactor the entrenchment scoring into one shared pure function used by both backends.

The observations show two separate implementations:

- `Network._entrenchment` is defined at `reasons/network.py:291`.
- `PgApi._entrenchment` is defined at `reasons/pg.py:3328`.
- Both are used in the same conceptual role:
  - `reasons/network.py:368`: `entrenchment = self._entrenchment(premise_id)`
  - `reasons/pg.py:3316`: `entrenchment = self._entrenchment(cur, premise_id)`

Existing tests exercise `Network._entrenchment` behaviorally:

- `tests/test_backtracking.py:230` asserts `evidence > basis`
- `tests/test_backtracking.py:259` asserts `sourced > unsourced`
- `tests/test_backtracking.py:266` asserts `obs > der`
- `tests/test_backtracking.py:275` asserts `popular > lonely`

But the observed tests only call `net._entrenchment(...)`. There is **no observed test calling `PgApi._entrenchment`**, and no observed test comparing the two implementations. That means the PostgreSQL implementation can drift from the in-memory implementation without detection.

One caveat: the observations are incomplete. The `_entrenchment` method bodies are not shown, only their definition lines. Also, `grep_entrenchment_tests` reports zero matches for `entrench`, while `grep_entrenchment_usage` clearly shows `entrench` matches in `tests/test_backtracking.py`. That inconsistency means the test inventory is not fully reliable without rerunning the search. The symbol lookups also return zero definitions despite grep confirming the methods exist, so those symbol results should not be treated as proof of absence.

Still, based on the available evidence, direct parity tests are the safest guard against semantic drift between `Network._entrenchment` and `PgApi._entrenchment`. They are especially valuable because the existing tests only assert relative ordering on the in-memory backend, not exact values or cross-backend equivalence.

A good parity test would:

1. Build the same graph state in both `Network` and `PgApi`.
2. Call both `_entrenchment` methods on the same node IDs.
3. Assert equality for representative nodes.
4. Reuse the cases already covered in `tests/test_backtracking.py`: evidence vs basis, sourced vs unsourced, observation vs derived, popular vs lonely.
5. Run as an integration test against PostgreSQL, likely skipped when Postgres is unavailable.

If maintaining two implementations is intentional, parity tests are necessary. If not, the stronger fix is to extract a shared pure scoring function and make both backends adapters over that function. In that case, parity tests become less critical, though a cross-backend integration smoke test is still useful.

## Topics to Explore

- [function] `reasons/network.py:_entrenchment` — Inspect the exact scoring rules and edge-case behavior of the in-memory implementation.
- [function] `reasons/pg.py:_entrenchment` — Compare the SQL-backed implementation against the in-memory rules.
- [file] `tests/test_backtracking.py` — Review existing entrenchment assertions and determine whether they can be reused for parity tests.
- [general] `postgres-integration-test-fixtures` — Understand how `PgApi` tests are set up, skipped, or isolated in this repo.
- [general] `shared-entrenchment-scoring` — Evaluate whether the scoring logic can be centralized instead of duplicated across backends.

## Beliefs

- `entrenchment-has-two-separate-implementations` — `Network._entrenchment` and `PgApi._entrenchment` are separate methods at `reasons/network.py:291` and `reasons/pg.py:3328`.
- `network-entrenchment-has-behavioral-tests` — `tests/test_backtracking.py` asserts entrenchment ordering at lines 230, 259, 266, and 275.
- `pgapi-entrenchment-lacks-observed-direct-tests` — The observed grep results show no test calling `PgApi._entrenchment`.
- `no-cross-backend-parity-test-observed` — No observed test compares `Network._entrenchment` output directly with `PgApi._entrenchment` output.
- `entrenchment-is-used-for-premise-scoring` — Both backends call their private `_entrenchment` method with a `premise_id` during backtracking-related logic.
