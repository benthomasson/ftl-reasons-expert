# Code Review Report

**Branch:** benthomasson/ftl-reasons#29
**Models:** claude, gemini
**Gate:** [BLOCK] BLOCK

## claude [CONCERN]

### `reasons_lib/network.py:_compute_next_nogood_id`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Clean implementation. `re.fullmatch(r"nogood-(\d+)")` correctly excludes prefixed IDs. `max_id = 0` default yields counter=1 for empty lists. The method is called at all 3 external ingestion points (storage, JSON import, beliefs import). The counter is initialized to 1 in `__init__` and incremented in `add_nogood` at line 315. This directly solves issue #26.

### `reasons_lib/network.py:add_nogood` (line 314-315)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Replaced `f"nogood-{len(self.nogoods) + 1:03d}"` with `f"nogood-{self._next_nogood_id:03d}"` and added `self._next_nogood_id += 1`. This is the core fix — ID generation is now independent of list length. The increment happens before the append, which is correct (the ID is captured in a local variable first).

### `reasons_lib/network.py:_rebuild_dependents`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Canonical rebuild implementation now lives on `Network` directly. Correctly clears all dependents sets, then rebuilds from both `antecedents` and `outlist`. Used by `storage.py` (replacing inline code), `import_agent.py:_fixup_dependents`, and `import_agent.py:_update_node_justifications`. This is a refactor, not part of issue #26, but clean and correct.

### `reasons_lib/network.py:verify_dependents`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Diagnostic method that builds expected dependents from justifications and compares against live state. Returns error strings. Useful for testing and debugging. Well-tested in `TestDependentsIntegrity` (8 tests).

### `reasons_lib/storage.py:load` (line 183)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

`network._compute_next_nogood_id()` called immediately after the nogood loading loop and before the method returns. Also replaces inline dependents rebuild with `network._rebuild_dependents()`. Both correct.

### `reasons_lib/api.py:import_json` (line 937)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

`net._compute_next_nogood_id()` called after appending all imported nogoods. Placement is correct — after the loop, before repos import.

### `reasons_lib/import_beliefs.py:import_into_network` (line 244)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

This was the missed call site caught in review round 1. The call is inside the `if nogoods_text:` block, after the nogood import loop. Correct placement — no nogoods text means no raw appends, so no counter update needed. Tested in `TestNogoodIdBeliefImport` (3 tests).

### `reasons_lib/import_agent.py:_import_nogoods` (line 251)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** PARTIAL

This function appends raw `Nogood` objects bypassing `add_nogood`. It uses prefixed IDs (`prefix + ng['id']`), which the `fullmatch` regex ignores — so currently safe. However, the prefix is a parameter, and if it were ever called with an empty prefix, the counter would be stale. A defensive `_compute_next_nogood_id()` call after the loop would close this gap. The implementation notes acknowledge this as non-blocking but call it out. No test verifies this path with the new counter logic.

### `reasons_lib/import_agent.py:_fixup_dependents` refactor
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Delegates to `network._rebuild_dependents()` instead of duplicating the logic. Correct and clean.

### `reasons_lib/import_agent.py:_update_node_justifications` refactor
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

The old code surgically removed old dependent registrations and added new ones for just the modified node. The new code replaces justifications and then calls `_rebuild_dependents()` which clears and rebuilds **all** dependents for the entire network. This is correct but changes from O(k) (where k = justification edges on one node) to O(N*J) (all nodes, all justifications). For large networks with frequent justification updates, this is a performance regression. Functionally correct, but the performance tradeoff wasn't acknowledged.

### `reasons_lib/check_stale.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Missing source files are now reported with `reason="source_deleted"` and `new_hash=None` instead of being silently skipped. Result dict shape is uniform across both reasons. Updated docstring accurately describes the return type. Well-tested in both `test_check_stale.py` and `test_check_stale_issue25.py`.

### `reasons_lib/cli.py:cmd_check_stale`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

CLI now shows "DELETED" label for `source_deleted` results and omits the hash line (since there's no new_hash to show). Falls through to "STALE" for content_changed. Tested in `TestCLIOutput`.

### `reasons_lib/derive.py:_build_beliefs_section` (line 214)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Bug fix — `count += len(belief_ids)` was inside the per-belief `for belief_id in belief_ids:` loop, causing quadratic inflation. Now correctly placed after the loop. This is a separate bug fix (looks like issue #23). Extensively tested in `test_derive_budget.py` (15 tests).

### `tests/test_nogood_id.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive test suite — 21 tests across 6 classes covering counter derivation, deletion scenarios, SQLite round-trip, JSON import, markdown import, and formatting edge cases. Tests the core bug scenario (delete middle, add — verify no collision) and all ingestion paths. Well-structured and thorough.

### `tests/test_network.py` additions
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Adds 3 nogood ID tests to the existing test class (deletion survival, empty list, prefixed IDs) and a new `TestDependentsIntegrity` class with 8 tests. The nogood tests overlap with `test_nogood_id.py` — there is **duplicate test coverage** for the core counter behavior (empty list, prefixed IDs, deletion). Not harmful but worth noting.

### `tests/test_check_stale.py` modifications
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Updated existing `test_stale_node` to assert `reason == "content_changed"`. Added `test_reports_missing_source_files` to verify the new deleted-source behavior. Tests match the updated return dict shape.

### Self-Review
**Limitations:** Could not run the test suite (permission was denied) to verify all tests pass. Relied on code reading only. Did not verify the `test_derive.py` diff since only a portion was shown. Could not verify whether `CLAUDE.md` modifications are intentional since it was in the git status but not in the provided diff.

### Feature Requests
- Include the full diff for all modified files in the review request, not just the new files — I had to read every file separately to reconstruct what changed
- Flag when a review request bundles unrelated changes (touching files unrelated to the stated issue) so the reviewer can call it out immediately
- Show test execution results alongside the diff to confirm tests pass

## gemini [BLOCK]

### FINAL_REPORT.md
**Verdict:** BLOCK
**Correctness:** BROKEN
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** MISSING

The issue requests a fix to `network.py` to use a monotonically increasing counter for nogood IDs. While the provided markdown files (like FINAL_REPORT.md, PLAN_1.md, and tester/REVIEW.md) describe the implementation and testing of this fix in detail, the diff itself contains ONLY these agent-generated markdown artifacts and zero Python code changes. The actual fixes to the codebase (`reasons_lib/network.py`, `reasons_lib/storage.py`, `reasons_lib/api.py`, `reasons_lib/import_beliefs.py`, and `tests/test_nogood_id.py`) are entirely missing from the changeset.

### Self-Review
**Limitations:** The diff only contained documentation and agent process artifacts, completely omitting the actual source code changes. I could not review the implementation because the Python files were not provided in the code changes.
