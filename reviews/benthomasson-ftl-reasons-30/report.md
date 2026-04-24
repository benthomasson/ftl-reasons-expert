# Code Review Report

**Branch:** benthomasson/ftl-reasons#30
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### reasons_lib/derive.py:_build_beliefs_section
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The fix is exactly one indentation change at line 214. Before: `count += len(belief_ids)` was indented under `for belief_id in belief_ids:`, running N times per agent (accumulating N*N). After: it's dedented to align with the `for` statement, running once per agent (accumulating N). This is the minimal correct fix — `count` now accumulates the actual number of shown beliefs per agent, and `remaining = max(5, max_beliefs - count)` on line 218 computes the non-agent budget correctly. No signature changes, no side effects.

### tests/test_derive.py:test_build_prompt_agent_count_does_not_starve_local
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Well-designed regression test. Uses 5 premise agent beliefs (no `sl=`, so they stay in `in_nodes` where N=5 and N^2=25 diverges), 8 local beliefs (exceeding the `max(5,...)` floor), and `budget=15`. The arithmetic is documented inline: fixed gives `remaining=max(5,15-5)=10` (all 8 locals fit), buggy gives `remaining=max(5,15-25)=5` (only 5 fit). Asserting on `local-belief-07` (the 8th, alphabetically last) is a clean canary. The test uses the existing `db` fixture appropriately.

### tests/test_derive_budget.py (new file, 15 tests)
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive edge-case coverage — multiple agents, single belief, no agents, empty network, derived agent beliefs, sampling mode, budget floor, direct `_build_beliefs_section` call. All tests are logically sound with documented arithmetic. However, this creates **duplicate test coverage** with `test_derive.py` — both files test the same regression through overlapping scenarios. The `test_count_is_linear_not_quadratic` test in this file is essentially a copy of the implementer's test with slightly different numbers. Minor concern: this is a lot of test infrastructure for a one-line indentation fix. Also note that `test_three_agents_count_accumulates` (line 252) has a buggy comment: it says "buggy count=9+9+9=27" but the actual bug would produce 3^2 + 3^2 + 3^2 = 27 — the comment is right by coincidence but describes the wrong calculation (should be "3*3 + 3*3 + 3*3 = 27").

### Self-Review
**Limitations:** Could not run the tests to verify they pass (permission denied on `uv run`). Reviewed correctness by reading the code and tracing the arithmetic manually. Could not verify the other modified files (api.py, network.py, etc.) are truly unrelated — only inferred from git diff --stat showing they have large, unrelated-looking diffs.

### Feature Requests
- Include the actual production code diffs in the provided changeset, not just new files — the `derive.py` and `test_derive.py` diffs were missing from what was supplied
- Flag when git status shows modified files not represented in the provided diff, so the reviewer can explicitly note mixed changesets
- Auto-detect and flag process artifact files (PLAN.md, REVIEW.md, IMPLEMENTATION.md patterns) that shouldn't be committed

## gemini [PASS]

### `reasons_lib/derive.py:_build_beliefs_section`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `count += len(belief_ids)` statement was correctly dedented to be outside the per-belief inner loop and inside the per-agent loop. This strictly implements the suggested fix, preventing quadratic count inflation and accurately allocating proportional token budget to non-agent local beliefs.

### `tests/test_derive.py:test_build_prompt_agent_count_does_not_starve_local`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The regression test correctly sets up a scenario with premise agent beliefs and >5 local beliefs to exceed the `max(5, ...)` floor. By setting a specific budget (15), the test reliably discriminates between buggy and fixed code based on the presence of the alphabetic tail of local beliefs.

### `tests/test_derive_budget.py`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

This file provides exceptionally comprehensive test coverage for budget allocations. Its 15 distinct tests handle multi-agent summing, zero agents, network boundaries, degenerate single-belief cases, derived exclusions, and sampling constraints, validating that the underlying logic in `_build_beliefs_section` remains stable.

### Self-Review
**Limitations:** None directly limiting the review accuracy, though the prompt's provided markdown diff did not include the actual `.py` implementation or test diffs. I used workspace context commands (`grep_search`, `list_directory`) to inspect the real source changes and test files in `/Users/ben/git/ftl-reasons/` to complete the review.

### Feature Requests
- Include actual source code file modifications (e.g., `.py` or `.ts` files) in the provided diff output, rather than just the agent's markdown artifacts and documentation logs.
