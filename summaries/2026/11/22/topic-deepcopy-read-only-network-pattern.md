---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The `what_if_*` functions depend on this pattern; investigate whether the invariant "never mutate read-only `Network`" is checked anywhere or is a purely social contract.

## Explanation

The `what_if_*` API functions rely on a **copy-before-mutate** pattern, not on a checked read-only invariant.

In `reasons/api.py`:

- `what_if_retract` starts at line 512, documents simulation “without mutating the database” at line 514, and does `net = copy.deepcopy(net)` at line 542.
- `what_if_assert` starts at line 580, has the same docstring intent at line 582, and deep-copies at line 609.
- `what_if_supersede` starts at line 647, has the same docstring intent at line 650, and deep-copies at line 677.

For the Postgres path, these functions call `_pg_dispatch` (`reasons/api.py:524`, `:591`, `:659`) rather than mutating a Python `Network` directly. Those dispatches reach methods such as `what_if_retract` and `what_if_assert` in `reasons/pg.py:439` and `reasons/pg.py:484`.

### Is the “never mutate read-only `Network`” invariant checked?

Based on the observations, **no**. It appears to be a social/developer contract plus defensive copying.

Evidence:

1. **`Network` is mutable by design.**  
   `Network` is defined at `reasons/network.py:17`. The tests in `tests/test_network.py` exercise mutating operations such as `add_node`, `retract`, `assert_node`, `add_nogood`, and `recompute_all`. There is no observed read-only wrapper or freeze mode.

2. **No immutability mechanism was found.**  
   The search for terms like `read-only`, `readonly`, `frozen`, `immutable`, `MappingProxyType`, `__setattr__`, and `__slots__` did not reveal any enforcement on `Network`. The only relevant matches are the defensive `copy.deepcopy(net)` calls in the three `what_if_*` API functions.

3. **The protection is a copy, not a check.**  
   The current `what_if_*` implementations avoid mutating the loaded `net` by working on a deep copy. That is not the same as verifying “this `Network` is read-only and must not be mutated.” If a future `what_if_*` function forgot the deep copy, or if another caller expects a `Network` to be read-only, nothing in the observed code would detect the violation.

4. **Tests do not appear to assert the invariant.**  
   The observation `search_what_if_in_tests` reports zero matches, but `locate_what_if_functions` found actual `what_if_supersede` tests in `tests/test_api.py:662-704`. That inconsistency means the search result is unreliable. Still, the visible tests exercise `what_if_supersede` behavior, not “original `Network` remained unchanged after a read-only operation.” The `search_network_mutation_in_tests` result also returns zero despite `tests/test_network.py` clearly containing `Network` and mutation-related tests, so that search observation is contradictory and should not be trusted alone.

### Conclusion

The invariant is **not checked anywhere in the observed code**. The practical rule is:

- `what_if_*` currently avoids mutating the original in-memory `Network` by deep-copying it first.
- There is no class-level immutability, runtime assertion, decorator, type guard, or test in the observations that enforces “never mutate a read-only `Network`.”
- Therefore, “never mutate read-only `Network`” is primarily a **social contract / implementation discipline**, with defensive copying used locally in the current `what_if_*` implementations.

To be completely certain, the full bodies of `_load_network`, `_pg_dispatch`, and the `what_if_*` functions would need inspection. The provided observations do not include those full bodies.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the full bodies around `what_if_retract`, `what_if_assert`, and `what_if_supersede` for any assertions or guards outside the snippets.
- [function] `reasons/api.py:_load_network` — Determine whether `what_if_*` loads a fresh `Network` or a shared/cached object, which affects how important the deep copy is.
- [file] `reasons/network.py` — Review the `Network` class for hidden copy semantics, mutability controls, or cached-state behavior.
- [function] `reasons/pg.py:what_if_retract` — Confirm whether the Postgres what-if path uses transaction rollback/CTE simulation and whether it has any read-only guard.
- [general] `read-only-network-invariant-tests` — Search the full test suite for reuse of a `Network` across `what_if_*` calls or assertions that the original object is unchanged.

## Beliefs

- `what-if-api-functions-use-deepcopy` — `what_if_retract`, `what_if_assert`, and `what_if_supersede` each deep-copy `net` before simulation at `reasons/api.py:542`, `:609`, and `:677`.
- `network-class-is-mutable` — `Network` at `reasons/network.py:17` exposes mutating operations such as `add_node`, `retract`, `assert_node`, and `add_nogood` in the observed tests.
- `no-readonly-network-enforcement-found` — No observed code implements frozen/read-only `Network` enforcement via `__setattr__`, `__slots__`, `MappingProxyType`, or explicit read-only checks.
- `pg-what-if-path-dispatches-to-pg-methods` — `api.what_if_retract/assert/supersede` call `_pg_dispatch` for Postgres, reaching methods like `reasons/pg.py:439` and `reasons/pg.py:484`.
- `read-only-network-invariant-is-social` — In the observations, the “never mutate read-only `Network`” invariant is maintained by developer convention and local defensive copying, not by runtime validation.
