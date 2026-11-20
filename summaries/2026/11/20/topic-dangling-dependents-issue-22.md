---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Regression history and intended semantics for stale/dangling dependent references.

## Regression history

The observed tests preserve two distinct regression stories:

1. **Issue #22 — dangling dependent references during propagation.**  
   `tests/test_dangling_dependents.py` starts with: “Tests for dangling dependent guard in `_propagate` (issue #22)” and says the guard “skips dangling dependent references with a warning instead of crashing with `KeyError`” (observed lines 1–5).  
   The regression class `TestDanglingDependentNoCrash` covers `retract`, `assert_node`, multiple ghost IDs, and a ghost alongside a valid dependent. `TestDanglingDependentWarningLogs` then fixes the expected warning contract: `action == "warn"`, `target` is the ghost ID, `value` contains `"dangling"` and the parent node ID, and the warning has a timestamp. `TestDanglingDependentPropagationContinues` verifies valid dependents still update through chains and diamonds.

2. **Issue #25 — missing source files in `check_stale`.**  
   `tests/test_check_stale_issue25.py` says: “check_stale reports missing source files” and “returns source_deleted results instead of silently skipping nodes whose source files no longer exist on disk” (observed lines 1–4). This is **source-file staleness**, not graph-dependent-reference staleness. `test_missing_source_returns_source_deleted` (around observed line 23) asserts `reason == "source_deleted"`, `new_hash is None`, `source_path is None`, and the old hash/source are preserved.

3. **Dependents index integrity tests.**  
   `tests/test_dependents_integrity.py` describes `_rebuild_dependents()` and `verify_dependents()`. `TestRebuildDependents.test_rebuild_clears_stale_entries` injects a fake dependent `"phantom"`, verifies that `verify_dependents()` reports corruption, then confirms `_rebuild_dependents()` removes it. `TestVerifyDependents.test_detects_extra_dependent` and `test_detects_missing_dependent` define “extra” and “missing” as corruption, and `test_does_not_modify_state` confirms verification is read-only.

The supplied `CHANGELOG.md` grep matches do not show a direct issue #22 or issue #25 entry. It does mention related dependent behavior at `CHANGELOG.md:41` (`--has-dependents`), `CHANGELOG.md:52` (JSON import validates `outlist` references), and `CHANGELOG.md:64` (deduplicating dependents’ justifications), but not the dangling guard itself.

## Intended semantics

### Dangling dependent references

A node’s `dependents` set is a reverse index of node IDs that depend on it. A **dangling dependent** is an ID stored in `node.dependents` that does not exist in `network.nodes`.

Based on `tests/test_dangling_dependents.py`, propagation through `_propagate` should:

- Not crash with `KeyError` when a dependent ID is missing.
- Log a warning event in `net.log` with `action == "warn"`, `target` equal to the dangling ID, `value` containing `"dangling"` and the parent node ID, and a timestamp.
- Emit one warning per dangling reference in the multiple-ghost case: `ghost1` and `ghost2` produce two warnings.
- Continue processing valid dependents. A ghost reference must not block valid cascade/restoration.
- Exclude dangling IDs from the `changed` list returned by `retract()`/`assert_node()`.
- Apply on retract, assert, challenge, and `add_justification` propagation paths, per the edge-case tests.

### Dependents index integrity

`tests/test_dependents_integrity.py` defines the repair/detection semantics:

- `_rebuild_dependents()` is the canonical repair. It clears the reverse index and reconstructs it from justifications (`antecedents` and `outlist`). It removes stale/phantom entries and restores missing entries.
- `verify_dependents()` is read-only. It detects “extra” dependent entries, including IDs that are not real nodes, and “missing” dependent entries, without fixing them.
- Every mutation path must leave the index consistent: `add_node` with justifications, retract/assert, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, etc. The tests use `assert_clean(net)` around these operations.

### “Stale” source references are separate

`reasons/check_stale.py:check_stale` (around observed lines 63–150) checks IN nodes with a `source` and `source_hash`. If `resolve_source_path()` returns `None`, it appends a result with `reason: "source_deleted"` (observed lines 95–104). This is file-source staleness, not reverse-index dependent staleness. It still skips OUT nodes, nodes without source, nodes without hash, and empty source strings, as covered by `TestSkipBehavior` in `tests/test_check_stale_issue25.py`.

## What is missing

The actual `reasons/network.py:_propagate` implementation is not included in the observations; `grep_source_for_dependents_logic` returned zero matches. So the exact guard code, warning message string, traversal order, and whether warnings are deduplicated cannot be verified from the supplied material. The semantics above are inferred from the regression tests.

## Topics to Explore

- [function] `reasons/network.py:_propagate` — Verify the exact dangling-dependent guard and warning emission logic.
- [file] `reasons/network.py` — Inspect `_rebuild_dependents`, `verify_dependents`, and every mutation path that touches `dependents`.
- [file] `tests/test_dependents_integrity.py` — Read the remaining mutation-path tests beyond the first 200 observed lines.
- [diff] `issue-22-dangling-dependent-guard` — Review the regression diff that changed the `KeyError` behavior into warn-and-skip behavior.
- [general] `check-stale-vs-dependents-staleness` — Keep source-file staleness separate from reverse-index stale/dangling dependents.

## Beliefs

- `dangling-dependent-guard-prevents-keyerror` — `_propagate` must skip missing dependent IDs instead of indexing `network.nodes` directly, per issue #22 tests.
- `propagation-warns-and-continues-on-dangling-dependents` — Dangling dependents produce warning log entries and do not block valid dependent propagation.
- `dangling-ids-are-excluded-from-changed` — `retract()` does not include dangling dependent IDs in its returned `changed` list.
- `rebuild-dependents-removes-phantom-entries` — `_rebuild_dependents()` clears and reconstructs the reverse index, removing injected phantom/extra dependents.
- `verify-dependents-is-read-only` — `verify_dependents()` detects extra/missing dependent entries without modifying `node.dependents`.
- `check-stale-reports-missing-sources-as-source-deleted` — `check_stale()` returns `reason="source_deleted"` for IN nodes whose source path resolves to `None`.
