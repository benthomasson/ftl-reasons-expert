# Code Review Report

**Branch:** benthomasson/ftl-reasons#32
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### reasons_lib/check_stale.py:check_stale
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The core fix is clean and minimal. When `resolve_source_path()` returns `None`, instead of silently skipping, a result dict with `reason: "source_deleted"`, `new_hash: None`, `source_path: None` is appended. The `content_changed` path gains a matching `reason` and `source_path` field, making both result shapes uniform. The iteration is `sorted(network.nodes.items())`, preserving deterministic order. The docstring is updated to document the new return shape. No behavioral regression — nodes without `source` or `source_hash`, and OUT nodes, are still skipped before the `resolve_source_path` call.

### reasons_lib/cli.py:cmd_check_stale
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The CLI handler correctly branches on `item.get("reason") == "source_deleted"` to print `DELETED` instead of `STALE`, and omits the hash line (which would show `None -> None`). Uses `.get()` defensively. The exit-code-1 behavior is unchanged — any stale results (including deleted sources) cause a non-zero exit. The summary line (`fresh, STALE (of checked)`) still counts deleted sources as stale, which is the right behavior since they represent nodes that need attention.

### tests/test_check_stale.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Two targeted changes: (1) `test_stale_node` gains `assert results[0]["reason"] == "content_changed"` to validate the new field on the existing path. (2) `test_skips_missing_source_files` is renamed to `test_reports_missing_source_files` and its assertions are flipped from `assert results == []` to verifying the full `source_deleted` result dict. Both changes align the existing test file with the new behavior.

### tests/test_check_stale_issue25.py (new file)
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive test file covering core detection, result shape uniformity, mixed scenarios, exclusions, API passthrough, and CLI output. Two concerns:

1. **Significant test duplication** with `test_check_stale.py`. The core behavior tested in `TestDeletedSourceDetection` (deleted source returns result, reason field, new_hash is None, source_path is None, preserves old_hash) directly duplicates `TestCheckStale.test_reports_missing_source_files`. The skip/exclusion tests (`TestExclusions`) also duplicate tests in the original file. This creates maintenance burden — a change to behavior requires updating assertions in two files.

2. **Note on diff vs. actual file**: The diff provided shows a 204-line file with classes `TestSourceDeletedResult`, `TestMixedResults`, `TestSkipBehavior`, `TestEdgeCases`. The file on disk is 345 lines with classes `TestDeletedSourceDetection`, `TestResultShape`, `TestMixedScenarios`, `TestExclusions`, `TestAPIPassthrough`, `TestCLIOutput`. The on-disk version is more comprehensive and includes API-layer and CLI-output tests not shown in the diff. This review covers both versions.

3. **Minor fragility**: The `test_deleted_source_then_recreated` and `test_deleted_source_then_recreated_same_content` tests are good edge cases that validate the transition from deleted→recreated, confirming the function is stateless per invocation.

### reasons_lib/api.py:check_stale (not modified)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The API layer requires no changes — it passes through the `results` list from `_check()` directly, and `stale_count` is `len(results)`. The `checked` count (nodes with `source` and `source_hash` that are IN) correctly includes nodes whose source files are deleted, since "checked" means "examined for staleness," not "found on disk." The API passthrough tests in `test_check_stale_issue25.py` (on-disk version) verify this end-to-end.

### Self-Review
**Limitations:** Could not run the test suite to verify all tests pass (permission denied for pytest). The diff provided for `test_check_stale_issue25.py` does not match the file on disk — the on-disk version is more comprehensive. Reviewed both versions but the diff-based review may miss intermediate changes. Did not verify whether other callers of `check_stale` beyond `api.py` and `cli.py` exist that might break due to the new result shape.

### Feature Requests
- Show all callers of modified functions (reverse call graph) to verify integration completeness
- Flag when a diff-provided file doesn't match the working tree, indicating the diff may be stale
- Auto-detect duplicate test coverage across test files and highlight overlapping assertions

## gemini [PASS]

### reasons_lib/check_stale.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `check_stale` function correctly detects when `path` is `None` (missing source) and appends a `source_deleted` dictionary rather than skipping silently. The shape of the return dicts aligns perfectly with the issue description, including a new `reason` key for both cases.

### reasons_lib/cli.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The CLI correctly utilizes the new `reason` field, using `.get("reason")` defensively, and appropriately changes output to "DELETED" without accessing `new_hash` (which is `None` in the `source_deleted` case). No tests for this CLI module change were included in the provided diff.

### tests/test_check_stale.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Existing tests were correctly updated to assert `reason: content_changed` and `reason: source_deleted` when verifying stale reports or missing sources.

### tests/test_check_stale_issue25.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Extensive new tests thoroughly cover permutations of missing sources, normal modifications, correct result dictionary structure, and skip behaviors. Excellent isolation and validation of edge cases.

### Self-Review
**Limitations:** The diff for `tests/test_check_stale_issue25.py` appeared truncated at line 204. I was unable to verify any potential CLI tests that may exist past that line, leading to an UNTESTED rating for `reasons_lib/cli.py` based strictly on the provided diff.

### Feature Requests
- Ensure complete file diffs or appended files are provided without truncation, especially for new test files, so that all coverage (such as CLI assertions) can be properly evaluated.
