---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The tests assume review LLM calls go through `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`, which is important when refactoring the LLM boundary.

# Why the review tests pin `shutil.which`/`subprocess.run` in `reasons.llm`

## The call chain the tests rely on

1. `reasons/review.py:10` does `from .llm import invoke_model`, so `review_beliefs` and `classify_defeat_reason` hold a *local reference* to `invoke_model`.
2. Inside `review_beliefs` (around `reasons/review.py:~232`) the code runs `response = invoke_model(prompt, model=model, timeout=timeout)`.
3. `invoke_model` (in `reasons/llm.py`) does three things for CLI models:
   - Line 297: `if not shutil.which(binary):` — fails fast if the binary isn't on `PATH`.
   - Line 302: `result = subprocess.run(...)` — the actual spawn.
   - `api:`/`vertex:`/`openai:` models **skip** these two lines and route to `_invoke_api`/`_invoke_openai`.
4. `reasons/llm.py` imports `shutil` and `subprocess` at the top of the module (lines 12–14), so those names *only* exist as module attributes of `reasons.llm`.

## Why the patch path must be `reasons.llm.*`

`patch("reasons.llm.shutil.which", return_value="/usr/bin/claude")` resolves `reasons.llm.shutil` to the shared `shutil` module object and stubs its `which` attribute. Same for `reasons.llm.subprocess.run`.

Because `reasons/review.py` never imports `shutil` or `subprocess`, you *cannot* write `patch("reasons.review.shutil.which", ...)` — the attribute lookup would `AttributeError`. This forces tests to reach all the way down into `reasons/llm.py`'s namespace. That's the coupling the observation is describing.

Examples of that deep patch:
- `tests/test_review.py` — `TestReviewBeliefs.test_reviews_batch`, `test_filters_to_existing_ids`, `test_batch_size_respected`, etc., all combine `patch("reasons.llm.shutil.which", …)` + `patch("reasons.llm.subprocess.run", return_value=mock_result)`.
- `tests/test_review_premises.py` — `TestReviewPremisesBatchLoop` and `TestApiReviewPremises` (e.g. `test_calls_llm_and_parses`, `test_stores_metadata`, `test_dry_run_no_metadata`).
- Same pattern leaks into sibling suites: `tests/test_ask.py:173`, `tests/test_classify_defeaters.py:14+`, `tests/test_contradictions.py:189+`, `tests/test_derive.py:946+`.

The observation counts ~127 occurrences of `shutil.which` and ~148 of `subprocess.run`, overwhelmingly in tests. Only `tests/test_llm.py` (`test_missing_binary_raises`, `test_invokes_subprocess`, `test_strips_claudecode_env`, etc.) *should* be poking at those internals — it is the unit test for `reasons/llm.py` itself.

## Why this matters when refactoring the LLM boundary

These tests treat `invoke_model`'s *implementation* as the contract. Any change to how `invoke_model` reaches the CLI silently invalidates dozens of tests even when behavior is unchanged. Specifically:

| Refactor | What breaks |
|---|---|
| Swap `subprocess.run` → `Popen().communicate()` or `check_output` | Every `mock_run.call_count == N` and `mock_run.call_args` assertion (e.g. `tests/test_review.py:test_batch_size_respected`, `tests/test_llm.py:test_invokes_subprocess`). |
| Move the `shutil.which` preflight into a helper module (e.g. `reasons/llm/discovery.py`) | `patch("reasons.llm.shutil.which", …)` no longer intercepts the call. |
| Make `invoke_model` async or return a coroutine | All `with patch(...)` context managers exercise synchronous text returns. |
| Add a new provider that bypasses subprocess (e.g. direct HTTP for `claude` default) | Batched tests that assert `mock_run.call_count == N` will see `0`. |
| Replace the CLI binary with an in-process SDK | `shutil.which` is never called; `FileNotFoundError` tests (`tests/test_llm.py:test_missing_binary_raises`) silently pass/fail for the wrong reason. |
| Change default `model="claude"` to `model="api:claude"` | Everywhere tests omit `model=`, the `api:` branch is taken and none of the `subprocess.run` stubs fire. |

## The contrasting, more resilient pattern already in the codebase

`tests/test_review_justifications.py` (e.g. `TestReviewJustifications.test_filters_to_multi_antecedent`, `TestApiReviewJustifications.test_returns_summary`) uses:

```python
patch("reasons.review_justifications.invoke_model", return_value=mock_response)
```

That works because `reasons/review_justifications.py` uses `from .llm import invoke_model` — patching the caller's namespace symbol is a smaller, intent-level seam. It does not care whether `invoke_model` uses `subprocess.run`, an SDK, or a network call. The review tests could adopt the same style (`patch("reasons.review.invoke_model", …)` and `patch("reasons.review_premises.invoke_model", …)`), keeping the `shutil`/`subprocess` stubs localized to `tests/test_llm.py`.

## Recommended refactor direction

1. **Pick one public seam** — either `reasons.llm.invoke_model` (keep as-is) or introduce a small `LLMClient` protocol/class with an `invoke(prompt, model, timeout)` method.
2. **Migrate the deep patches** in `tests/test_review.py`, `tests/test_review_premises.py`, `tests/test_classify_defeaters.py`, `tests/test_contradictions.py`, `tests/test_derive.py`, `tests/test_ask.py` to target that seam (`reasons.review.invoke_model`, etc.).
3. **Keep `tests/test_llm.py`** as the only place that patches `reasons.llm.shutil.which` / `reasons.llm.subprocess.run` — that's its job.
4. **Add a contract test** that pins the public behavior: `invoke_model(prompt, model=..., timeout=...)` returns `str` for CLI + API providers, raises `FileNotFoundError` when the binary is missing, and raises `TimeoutExpired` on timeout. That way downstream modules never need to know which branch was taken.

## Topics to Explore

- [file] `reasons/llm.py` — Full `invoke_model` implementation; trace the `api:`/`vertex:`/`openai:` dispatch vs. the `shutil.which`/`subprocess.run` CLI path to see exactly where a refactor can hide.
- [function] `reasons/review.py:review_beliefs` — The caller that currently only appears to use `invoke_model`, but whose tests are coupled to `reasons.llm` internals.
- [file] `tests/test_review_justifications.py` — Reference implementation of the *correct* patch pattern (`reasons.review_justifications.invoke_model`), which the other review tests should copy.
- [general] `mock-seam-audit` — Grep the test suite for `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` to enumerate every test that would break under a boundary refactor; prioritize those for migration.
- [function] `reasons/llm.py:invoke_model` — Design a contract test here (return type, exception types) so downstream suites can mock at the caller's namespace without losing coverage of failure modes.

## Beliefs

- `review-beliefs-calls-invoke-model` — `reasons/review.py:review_beliefs` invokes the LLM exclusively through the `invoke_model` symbol imported from `reasons.llm` (line 10), never directly via subprocess.
- `cli-path-requires-shutil-which` — `reasons.llm.invoke_model` calls `shutil.which(binary)` (line 297) before `subprocess.run` (line 302) only for CLI models; `api:`, `vertex:`, and `openai:` prefixes bypass both.
- `review-tests-patch-llm-module-namespace` — Tests in `tests/test_review.py` and `tests/test_review_premises.py` patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` because `reasons/review.py` does not import `shutil` or `subprocess` itself.
- `subprocess-run-is-implicit-contract` — Roughly 30+ review/classify/derive tests assert on `mock_run.call_count` or `mock_run.call_args`, making `subprocess.run` an undocumented contract of `invoke_model`.
- `review-justifications-uses-shallow-seam` — `tests/test_review_justifications.py` patches `reasons.review_justifications.invoke_model` instead, demonstrating the more refactor-resilient boundary pattern.
- `test-llm-is-legitimate-deep-patcher` — `tests/test_llm.py` is the only test file where patching `reasons.llm.shutil.which` / `reasons.llm.subprocess.run` is the intended boundary, because it unit-tests `llm.py` itself.
