# Code Review Report

**Branch:** benthomasson/ftl-reasons#6
**Models:** claude, gemini
**Gate:** [BLOCK] BLOCK

## claude [BLOCK]

### reasons_lib/api.py:_rewrite_dependents
**Verdict:** PASS
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Logic is sound for the common case. Uses `list()` for safe iteration during set mutation — good. Two edge cases: (1) If `new_id` is already in a justification's antecedents/outlist, the rewrite creates duplicate entries (e.g., `["new_id", "new_id"]`). Unlikely in practice since duplicates have *similar* IDs, not both appearing in the same justification. (2) After rewriting outlist from a retracted duplicate to the kept node, the outlist condition may change validity (outlist node was OUT, is now IN). This is semantically correct — if duplicates are the same belief, "unless A" should become "unless B" — but the truth value change is a side-effect the user might not expect. Tests exist for both antecedent and outlist rewrites.

### reasons_lib/api.py:write_dedup_plan
**Verdict:** BLOCK
**Correctness:** BROKEN
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** PARTIAL

**Bug in the primary new workflow.** When called from the non-auto dedup path (the only path that calls this function — `cli.py:538`), clusters don't have a `"kept"` key set. `cluster.get("kept")` returns `None`, so `b["id"] == None` is always false, and **every belief is marked `[RETRACT]`** with none marked `[KEEP]`. When the user then runs `--accept`, `parse_dedup_plan` requires `current_keep` to be truthy to emit a cluster, so all clusters are silently dropped — "No clusters found in plan file." The test `test_write_and_parse_dedup_plan` doesn't catch this because it manually constructs a cluster dict **with `kept` already set**, which doesn't match the real call path. Fix: either always compute `kept` in `deduplicate()` (move the `max(members, ...)` line outside the `if auto:` block), or add a fallback in `write_dedup_plan`.

### reasons_lib/api.py:parse_dedup_plan
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

The parser logic is correct for well-formed input. However: (1) A cluster with multiple `[KEEP]` entries silently uses the last one — should this warn? (2) A cluster with zero `[KEEP]` entries is silently dropped, which contributes to the write_dedup_plan bug being invisible. (3) No test for malformed input (missing KEEP, extra KEEP, garbled lines). The regex `\S+?` is fine for hyphenated IDs but would break on IDs containing backticks, which is unlikely.

### reasons_lib/api.py:apply_dedup_plan
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Good error handling — missing nodes report errors rather than crashing, already-OUT nodes are skipped. Correctly calls `_rewrite_dependents` before `net.retract`. Uses `_with_network(write=True)` so changes are persisted. Minor: doesn't warn when the `keep` node is already OUT (dependents would be rewritten to point at an OUT node, making their justifications invalid). Tests cover: happy path, missing node errors, and user-edited keep selection.

### reasons_lib/api.py:deduplicate (modified)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The addition of `_rewrite_dependents` before `net.retract` is the right order — rewrite the references while the old node is still in `net.nodes`, then retract. The `cluster["kept"]` assignment is only inside `if auto:`, which is the root cause of the `write_dedup_plan` bug (see above). The `kept` computation should be moved outside the `if auto:` block so it's always available.

### reasons_lib/cli.py:cmd_deduplicate (modified)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The `--accept` path is well-structured: validates file exists, parses, applies, reports errors. The non-auto path now writes a plan file instead of suggesting `--auto`, which is the intended workflow change. However: (1) The behavior change from "suggest --auto" to "write plan file" means `reasons deduplicate` now has a **side effect** (creates `proposed-dedup.md` in CWD) on every non-auto run — users might not expect file creation from a query command. (2) If `--auto` and `--accept` are both passed, `--accept` takes precedence (checked first), which is fine but undocumented. (3) The `←` to `<-` change is a minor encoding fix. (4) No CLI-level tests for either `--accept` or the plan-writing path.

### Self-Review
**Limitations:** Could not run tests to verify they pass (permission denied). Did not see the full `add_node` API to verify how `sl`/`unless` params construct justifications and whether truth values in tests match expectations. Did not see `_propagate` to verify rewrite + retract ordering doesn't cause stale truth values.

### Feature Requests
- Include the full function body for callers of modified functions (e.g., show `add_node` when tests call it with specific params)
- Run the test suite automatically and include pass/fail results
- Show the actual file written by `write_dedup_plan` for a non-auto cluster to make the BLOCK bug immediately visible

## gemini [BLOCK]

### `reasons_lib/api.py`
**Verdict:** BLOCK
**Correctness:** BROKEN
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** PARTIAL

The `deduplicate` function only sets `cluster["kept"] = keep` inside the `if auto:` block (around line 1235). When run in manual mode (`auto=False`), `cluster.get("kept")` evaluates to `None`. This causes `write_dedup_plan` to write `[RETRACT]` for every single node in the proposed plan, failing to propose a default `[KEEP]` node. If a user runs `reasons deduplicate --accept proposed-dedup.md` without manually editing the file to add a `[KEEP]` to each cluster, `parse_dedup_plan` will silently drop all clusters (since `current_keep` remains `None`), resulting in a "No clusters found in plan file" message. The manual deduplication workflow is functionally broken out of the box. To fix this, extract the `keep` calculation in `deduplicate` so it runs unconditionally for every cluster, allowing `write_dedup_plan` to propose a default node to keep. The `_rewrite_dependents` logic itself is sound and correctly handles non-cascading retractions.

### `reasons_lib/cli.py`
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The CLI changes correctly wire up the new `--accept` and `--output` flags and provide clear feedback. However, because of the bug in `api.py` where `cluster.get("kept")` is not populated in manual mode, the CLI will never actually print the `<- kept` indicator when listing clusters in manual mode, which violates the intended UX. Additionally, there appears to be no test coverage for the CLI parsing or formatting logic for the new deduplication plan workflow.

### `tests/test_derive.py`
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The tests are well-structured and successfully verify the dependent rewriting logic and the parsing/application of dedup plans. However, `test_write_and_parse_dedup_plan` manually constructs a mock `clusters` dictionary that already includes `"kept": "gl108-validation-disabled"`. Because it bypasses calling `deduplicate(auto=False)` to generate this payload, the test fails to exercise the integration gap that causes the all-`[RETRACT]` bug. Adding an end-to-end test for the manual workflow (calling `deduplicate` with `auto=False`, passing the result to `write_dedup_plan`, and parsing it back) would ensure the entire feature works as expected.

### Self-Review
**Limitations:** I initially lacked the full context of `deduplicate` in `api.py` outside of the provided diff hunks, which obscured the `if auto:` conditional logic. I was able to retrieve this using the read file and grep tools to confirm the bug.

### Feature Requests
- Include the full file context for modified functions, not just diff hunks. Bugs often hide in the unedited lines of a modified function (like the `if auto:` block that was omitted from the diff).
