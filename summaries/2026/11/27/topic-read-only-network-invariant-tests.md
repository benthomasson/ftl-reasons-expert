---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Search the full test suite for reuse of a `Network` across `what_if_*` calls or assertions that the original object is unchanged.

## What the search shows

The search is effectively negative: the test suite does **not** contain any literal references to `what_if` and does **not** instantiate `Network` directly.

Key evidence:

- `find_what_if_usages_in_tests` searched for `what_if` in tests and returned `0` matches.
- `find_network_instantiation_in_tests` searched for `Network(` in tests and returned `0` matches.
- `grep_unchanged_or_original_assertions` searched for phrases like `unchanged`, `original`, `pristine`, `not mutated`, and `mutat`, and returned `0` matches.
- `tests/conftest.py` only defines a `pg_api` fixture. It yields a `PgApi` instance and cleans up PostgreSQL tables, but it does not provide or construct a `Network`.

The `what_if_*` implementations that do exist are in production code:

- `reasons/api.py:512` — `what_if_retract`
- `reasons/api.py:580` — `what_if_assert`
- `reasons/api.py:647` — `what_if_supersede`
- `reasons/mcp_server.py:176` — `what_if`
- `reasons/pg.py:439` — `PgApi.what_if_retract`
- `reasons/pg.py:484` — `PgApi.what_if_assert`

So, based on the literal searches performed, there is **no evidence in the test suite of reusing a `Network` across multiple `what_if_*` calls**, and **no explicit assertions that an original object remains unchanged**. In fact, there are no test references to `what_if` at all in the observed results.

## Important caveats

These are negative literal searches, so they do not prove that no such behavior is tested indirectly. The following would be needed to make a stronger claim:

- A search for `Network` as an identifier, not just `Network(`.
- A search for test fixtures or factories named `network`, `net`, `api`, `client`, etc.
- A search for imported aliases of `what_if_*` functions.
- A search for state assertions using equality, node counts, edge counts, versions, or snapshots rather than words like `unchanged`.
- The actual contents of `read_network_what_if_implementation`, which returned no matches and therefore does not let us verify the intended immutability behavior of `Network`.

## Topics to Explore

- [file] `tests/conftest.py` — Confirms the only shared fixture is `pg_api`; check whether other test modules define local `Network` fixtures or factories.
- [function] `reasons/api.py:what_if_retract` — Shows the public `what_if_*` API surface and how it loads or mutates network state.
- [function] `reasons/pg.py:what_if_retract` — Shows the PostgreSQL-backed `what_if` implementation and whether it mutates stored state or works transactionally.
- [general] `network-state-immutability-tests` — Search for assertions on node/edge counts, equality, snapshots, or transaction rollback after `what_if_*` calls.
- [general] `test-suite-alias-and-fixture-search` — Search for `Network`, `network`, `net`, and aliases of `what_if_*` to catch indirect usage not found by the literal patterns.

## Beliefs

- `test-suite-has-no-literal-what-if-references` — The observed test search for `what_if` returned zero matches.
- `test-suite-has-no-network-instantiation` — The observed test search for `Network(` returned zero matches.
- `tests-conftest-provides-pg-api-fixture` — `tests/conftest.py` defines a `pg_api` fixture that yields `PgApi` and cleans up PostgreSQL tables.
- `test-suite-has-no-explicit-unchanged-assertions` — The observed search for `unchanged`, `original`, `pristine`, `not mutated`, and `mutat` returned zero matches.
- `what-if-definitions-exist-outside-tests` — `what_if_*` functions are defined in `reasons/api.py`, `reasons/mcp_server.py`, and `reasons/pg.py`, not in the observed test files.
