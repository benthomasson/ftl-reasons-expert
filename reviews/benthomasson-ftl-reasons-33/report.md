# Code Review Report

**Branch:** benthomasson/ftl-reasons#33
**Models:** claude, gemini
**Gate:** [PASS] PASS

## claude [PASS]

### reasons_lib/derive.py:_build_beliefs_section (line 214)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The fix is a one-character indentation change — moving `count += len(belief_ids)` from inside the `for belief_id in belief_ids:` loop (line 211) to after it. Before the fix, each iteration added `len(belief_ids)` to `count`, producing `N * N = N²` accumulation. After the fix, it runs once per agent, accumulating exactly `N`. The downstream effect at line 218 (`remaining = max(5, max_beliefs - count)`) now gets the correct remainder, so local beliefs are no longer starved. The fix is minimal and surgical — no risk of side effects.

### tests/test_derive_budget.py (new file, 281 lines)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Thorough regression test suite covering 7 test classes and 15 tests. Key strengths:

- **Core regression** (`TestCountLinearNotQuadratic`): `test_six_beliefs_one_agent` precisely encodes the bug's arithmetic — with the old code, `count=5*5=25` starves locals to floor 5; with the fix, `count=5`, locals get 15. The docstring math is correct and verified against the function logic at lines 197-198 and 218.
- **Multi-agent** (`TestMultiAgent`): Validates cross-agent accumulation with 2 and 3 agents, asserting `remaining = max(5, max_beliefs - agent_total)`.
- **Floor guarantee** (`TestBudgetFloor`): Confirms `max(5, ...)` floor with 50 and 200 agent beliefs.
- **Edge cases** (`TestEdgeCases`): N=1 (where N²=N, bug invisible), no agents, no locals, empty network, empty agents, derived exclusion.
- **Sampling** (`TestSampling`): Covers the `sample=True` path — both budget correctness and determinism.
- **Integration** (`TestBuildPromptIntegration`): End-to-end through `build_prompt` with a real SQLite database via `api.add_node` and `api.export_network`.

The `_make_nodes` helper correctly avoids creating `:active` premise nodes (which would inflate `agent_beliefs` counts due to `startswith` matching at line 192). The regex helpers `_count_local_shown` and `_count_agent_shown` use `re.escape` for agent names and assert on match failure with diagnostic output.

No overlap with existing `test_derive.py` — the existing file tests `build_prompt` at the API level but doesn't test `_build_beliefs_section` directly or verify budget arithmetic. The new file fills a genuine coverage gap.

One minor note: `test_derived_beliefs_excluded_from_count` (line 213) only asserts `"Agent: agent-a" in output` — it doesn't verify the count is unaffected by derived beliefs. This is more of a smoke test than a budget-accuracy test. Not a concern — the core regression tests are strong.

### Self-Review
**Limitations:** Could not run the tests (permission denied for `uv run pytest`), so pass/fail is verified by code reading only. Also did not see the `build_prompt` function signature to confirm how it passes `budget` through to `_build_beliefs_section` as `max_beliefs`, though the integration test implicitly validates this path.

### Feature Requests
- Include callers of modified functions to verify integration wiring without manual grep
- Allow test execution in review mode to confirm tests actually pass

## gemini [PASS]

### reasons_lib/derive.py:_build_beliefs_section
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `count += len(belief_ids)` statement was unindented so that it executes once per agent rather than once per belief in the agent. This corrects the N² bug to correctly accumulate N beliefs per agent, directly addressing the described issue.

### tests/test_derive_budget.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Adds comprehensive test coverage for `_build_beliefs_section` and its budget constraints, including tests specifically targeting the N² accumulation bug and budget starvation for local beliefs, as well as tests handling multiple agents, budget floors, and random sampling modes.

### Self-Review
**Limitations:** None. The issue description and provided diff directly align with the expected changes and provide complete clarity on the intent and resolution.
