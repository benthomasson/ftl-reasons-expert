# Code Review Report

**Branch:** benthomasson/ftl-reasons#15
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### `reasons_lib/api.py:_node_depth`
**Verdict:** PASS
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The cycle guard (`memo[nid] = 0`) prevents infinite recursion but silently produces incorrect depths for cyclic justification chains — a node in a cycle gets depth 0+1=1 regardless of its actual position. This is acceptable since TMS cycles are pathological, but worth noting. The mutable default argument `memo=None` pattern is correct (guarded by `if memo is None`). One concern: this duplicates logic from `derive.py:_get_depth` — same algorithm, different data structures (Network objects vs dicts). This is a minor maintainability issue but not a blocker.

### `reasons_lib/api.py:list_nodes`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `memo` dict is lazily created only when depth filters are active, which is efficient. Depth filtering is applied after all other filters, so nodes already excluded by status/premises/namespace checks don't incur depth computation cost. The shared `memo` across all nodes in the loop ensures O(n) depth computation total. CLI integration at `cli.py:cmd_list` passes both `min_depth` and `max_depth` correctly.

### `reasons_lib/derive.py:_get_depth` (cycle guard fix)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Adding `memo[node_id] = 0` as a cycle guard before recursing fixes infinite recursion on cyclic graphs. This mirrors the same pattern in `api.py:_node_depth`. No dedicated test for the cycle case exists, but the fix is clearly correct — without it, a cycle would recurse infinitely; with it, a cycle breaks at depth 0 (consistent with the api.py version).

### `reasons_lib/derive.py:build_prompt` (filter additions)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The approach of computing depth from the full graph *before* filtering is correct — this ensures depth values don't shift when nodes are removed. However, the `max_depth` recomputation on line 400 (`max_depth = max((_get_depth(k, nodes, all_derived, memo) for k in derived), default=0)`) passes `nodes` (the filtered set) but `all_derived` (from the pre-filter set). This is fine for depth lookup since `memo` is already populated, but the intent is unclear — it's computing max depth over the *filtered* `derived` set using *pre-filter* derivation data via the memoized values. This works correctly because memo already has all values, but is fragile if someone later clears or modifies memo.

A more substantive concern: `premises_only` and `has_dependents` are new `build_prompt` parameters that are **not documented in the docstring**. The docstring mentions `min_depth` and `max_depth_filter` but omits the two new boolean params.

### `reasons_lib/cli.py:_derive_one_round`
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The depth filter display logic on line 597 (`if stats.get("min_depth") is not None or stats.get("max_depth_filter") is not None`) has a subtle bug: `stats.get("min_depth")` returns `None` when the key is absent, so `is not None` is correct. **But** `stats.get("min_depth")` also returns `None` when `min_depth` was not passed — and `min_depth` is only added to `stats` when it's not None (line 456-457 in derive.py). So this works, but only because the `is not None` check catches both "key absent" and "value is None" — it's coincidentally correct. A more robust pattern would be `if "min_depth" in stats or "max_depth_filter" in stats`. 

Also: the `--premises` and `--has-dependents` flags for `derive` are not tested through the CLI path.

### `reasons_lib/cli.py` (argparse additions)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Both `derive` and `list` subcommands get the new `--min-depth`, `--max-depth`, `--premises`, and `--has-dependents` flags. The `list` subcommand already had `--premises` and `--has-dependents` so no duplication there. The `derive` subcommand correctly maps `args.max_depth` to `max_depth_filter` parameter. `list` correctly maps to `max_depth`.

### `tests/test_api.py:TestListNodesDepth`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Three tests covering min-only, max-only, and range filtering. Each builds a small network and verifies correct inclusion/exclusion. The depth range test verifies exact equality (`ids == ["mid"]`), which is good. Tests use the `db_path` fixture correctly.

### `tests/test_derive.py` (new tests)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Five new tests covering `min_depth`, `max_depth_filter`, depth range, `premises_only`, and `has_dependents` through `build_prompt`. The `has_dependents` test correctly expects 2 nodes (fact-a and fact-b are referenced by derived-ab; fact-c is not). The depth range test uses unique prefixed node IDs (`zz-*`) to avoid collisions with the fixture. Good coverage of the derive path.

### Self-Review
**Limitations:** Could not run the test suite to verify all tests pass. Did not see the full `_derive_one_round` function to verify `args.premises` and `args.has_dependents` are accessible (though argparse additions confirm they're defined). Did not check whether the `--premises` flag for `derive` conflicts with any existing arg in the derive parser.

### Feature Requests
- Include full function bodies for all modified functions in the observations, not just the diff hunks — I had to do manual reads to verify surrounding logic
- Show callers of modified functions to verify integration completeness
- Auto-run the test suite and include results in the review context

## gemini [PASS]

### `reasons_lib/api.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `_node_depth` function correctly calculates node depth from premises (depth 0) and includes a necessary cycle guard to prevent infinite recursion in graph cycles. The `list_nodes` filtering logic handles the new `min_depth` and `max_depth` parameters properly, executing the filters smoothly.

### `reasons_lib/cli.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Successfully wires the new depth, premises, and dependents filtering parameters from CLI arguments through to `derive.build_prompt` and `api.list_nodes`. Adds helpful logging for depth filtering to stderr using bounds (e.g. 0–∞). Missing direct CLI tests in the diff, but the core logic is covered in API and derive tests.

### `reasons_lib/derive.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Filtering logic in `build_prompt` is robust. It properly evaluates depth and dependencies against the *full* original network before filtering out nodes, guaranteeing that node depth is consistently measured regardless of the filter applied. Recomputing `max_depth` for the stats dictionary correctly provides the max depth of the *filtered* network.

### `tests/test_api.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Provides appropriate and direct coverage for `min_depth` and `max_depth` options in `api.list_nodes()`.

### `tests/test_derive.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Provides excellent test coverage mapping to the newly introduced prompt generation filters: depth range, min/max depths, premises only, and dependents. Assertions correctly validate prompt exclusions and metadata stats.

### Self-Review
**Limitations:** I assumed the `db` fixture in `test_derive.py` is properly established in `conftest.py` or the suite structure since it's used instead of `db_path` (which appears in `test_api.py`), though it is standard practice. No major limitations impeded reviewing the logic correctly.
