---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Decide when the incremental `dependents`-driven path is trustworthy vs. when the full sweep is required; the `verify_dependents` helper is relevant here.

## Decision rule for trusting the incremental path

The relevant helper is `Network.verify_dependents()` in `reasons/network.py:44`. From the observed tests, it is a **read-only consistency checker** for the reverse `dependents` index: it detects “extra” and “missing” reverse entries and does **not** repair them (`tests/test_dependents_integrity.py:95`, `:108`, `:117`, `:125`). The canonical repair is `_rebuild_dependents()`, which rebuilds the index from justifications and clears stale entries (`tests/test_dependents_integrity.py`, `TestRebuildDependents`; corruption/repair tests at `:227` and `:237`).

Use the incremental `dependents`-driven path only when **all** of these are true:

| Condition | Why |
|---|---|
| `verify_dependents()` returns `[]` | Confirms the current reverse index has no observed extra/missing entries. Clean network tests assert this at `tests/test_dependents_integrity.py:102`. |
| The mutation went through a tested `Network` mutation path | Tests cover `add_node` with justifications, `retract`/`assert_node`, `add_justification`, `supersede`, `challenge`/`defend`, `convert_to_premise`, `add_nogood`, and `summarize` in `TestMutationPathIntegrity`. |
| No code directly mutated `node.dependents`, `justifications`, `antecedents`, or `outlist` | Direct mutation can bypass the incremental bookkeeping; tests show injected extra/missing entries are detected. |
| The update logic accounts for both `antecedents` and `outlist` reverse edges | Tests specifically cover `outlist` and combined antecedent/outlist dependencies (`test_outlist_dependency`, `test_both_antecedent_and_outlist`). |
| The DB load/migration path is known to preserve or rebuild the index | Storage round-trip tests assert cleanliness after save/load, but older or migrated DBs should be verified first. |

If any condition fails, require the full sweep:

1. Run `_rebuild_dependents()`.
2. Re-run `verify_dependents()` and require `[]`.
3. Only then trust the incremental path again.

In other words: `verify_dependents()` is a **gate**, not a proof that a future incremental mutation is correct. It proves the current index is clean; the mutation-path tests prove which operations are expected to keep it clean. If the index is dirty, or the mutation path is untested/unknown, do the full `_rebuild_dependents()` sweep.

## What is missing from the observations

The observations do **not** include the implementation body of `verify_dependents()` or `_rebuild_dependents()`, nor any production call site of an incremental “dependents-driven” path. The usage search for `dependents` in `reasons` returned zero matches, which is inconsistent with the observed definition at `reasons/network.py:44`. So the exact runtime trigger for the incremental path cannot be proven from the provided material; the guidance above is the contract implied by the tests.

Also note: `reasons/verify.py` is unrelated to this topic. It verifies beliefs against source documents; `verify_dependents()` is the reverse-index integrity helper on `Network`.

## Topics to Explore

- [function] `reasons/network.py:verify_dependents` — Read the implementation to confirm exactly what invariants it scans and what error strings it returns.
- [function] `reasons/network.py:_rebuild_dependents` — Understand the canonical full sweep and whether it is safe/cheap to run after every risky operation.
- [file] `tests/test_dependents_integrity.py` — Study mutation-path coverage and add tests for any new incremental update path.
- [general] `incremental-dependents-update-path` — Locate the actual production call sites that use `node.dependents` to limit work; they were not present in the observations.
- [file] `reasons/storage.py` — Confirm whether load/save rebuilds or verifies the dependents index, especially for older DBs.

## Beliefs

- `verify-dependents-is-read-only` — `verify_dependents()` detects inconsistencies but does not fix them, per the test docstring at `tests/test_dependents_integrity.py:95` and the state-preservation test around `:125`.
- `verify-dependents-returns-empty-list-for-clean-index` — Clean networks assert `net.verify_dependents() == []` at `tests/test_dependents_integrity.py:102`.
- `rebuild-dependents-is-canonical-repair` — `_rebuild_dependents()` clears stale entries and repairs extra/missing reverse entries, as exercised in `TestRebuildDependents` and the corruption/repair tests at `tests/test_dependents_integrity.py:227` and `:237`.
- `dependents-index-tracks-antecedents-and-outlists` — Tests assert reverse edges for both `antecedents` and `outlist` dependencies (`test_outlist_dependency`, `test_both_antecedent_and_outlist`).
- `incremental-path-not-observed` — The gathered `dependents` usage search found no production matches in `reasons`, so the actual incremental call site is missing from the observations.
