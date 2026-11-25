---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Grep the test suite for `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` to enumerate every test that would break under a boundary refactor; prioritize those for migration.

## Grep conclusion: the reported zero is not trustworthy

The grep observations are internally inconsistent:

- `grep_tests_for_reasons_llm_shutil_which` reports `total_count: 0`.
- `grep_tests_for_reasons_llm_subprocess_run` reports `total_count: 0`.
- The broader `(shutil\.which|subprocess\.run)` grep also reports `0`.
- But the read of `tests/test_llm.py` clearly contains literal patches such as:
  - `patch("reasons.llm.shutil.which", ...)`
  - `patch("reasons.llm.subprocess.run", ...)`

So the correct interpretation is not “there are no tests to migrate.” It is: **the grep run was scoped incorrectly, over-escaped, or otherwise failed to see `tests/test_llm.py`.** Re-run with a fixed-string or simpler regex, for example:

```bash
rg -n -F 'reasons.llm.shutil.which' tests
rg -n -F 'reasons.llm.subprocess.run' tests
rg -n 'reasons\.llm\.(shutil\.which|subprocess\.run)' tests
```

Also note: `tests/test_llm.py` has `470` total lines, but the read returned only lines `1–300`. So even the file read is truncated and cannot prove an exhaustive list. Within the visible segment, there are at least **10 tests patching `reasons.llm.shutil.which`** and **9 patching `reasons.llm.subprocess.run`**.

## Why these tests break under a boundary refactor

The tests patch module attributes on `reasons.llm`:

```python
patch("reasons.llm.shutil.which", ...)
patch("reasons.llm.subprocess.run", ...)
```

That means they depend on:

1. `reasons.llm` importing `shutil` and `subprocess`.
2. `reasons.llm.invoke_model` calling those exact attributes directly.
3. The old call signature and return shape continuing to exist.

If the refactor extracts CLI execution into a new boundary, such as `reasons/llm_cli.py` or a function like `reasons.llm._run_cli(...)`, then:

- `patch("reasons.llm.subprocess.run")` will patch an unused object or fail if `reasons.llm` no longer imports `subprocess`.
- `patch("reasons.llm.shutil.which")` will no longer intercept the actual lookup.
- Tests that assert `mock_run.call_args` will be tied to the old internal call contract instead of the new boundary contract.

## Visible tests to migrate, prioritized

| Priority | Test | Patches | Why it is at risk |
|---|---|---|---|
| P0 | `tests/test_llm.py::TestInvokeModel::test_invokes_subprocess` | `reasons.llm.shutil.which`, `reasons.llm.subprocess.run` | Asserts exact command list, `input`, and `timeout` passed to `subprocess.run`; highest coupling to the old internal call site. |
| P0 | `tests/test_llm.py::TestInvokeModel::test_strips_claudecode_env` | both | Asserts the `env` dict passed to `subprocess.run`; boundary refactor may change how env is supplied. |
| P0 | `tests/test_llm.py::TestInvokeModel::test_missing_binary_raises` | `reasons.llm.shutil.which` | If `shutil.which` lookup moves out of `reasons.llm`, this patch will not control the path; if `import shutil` is removed, the patch target disappears. |
| P0 | `tests/test_llm.py::TestInvokeModel::test_nonzero_exit_raises` | both | Relies on a mocked subprocess result reaching the old error-mapping code in `invoke_model`. |
| P1 | `tests/test_llm.py::TestInvokeModel::test_ollama_command` | both | Asserts Ollama command construction and return path through subprocess. |
| P1 | `tests/test_llm.py::TestInvokeModel::test_claude_does_not_strip_thinking` | both | Uses the mocked subprocess output path and asserts Claude post-processing behavior. |
| P1 | `tests/test_llm.py::TestInvokeModelApiDispatch::test_claude_still_uses_subprocess` | both | Specifically verifies that Claude remains on the CLI/subprocess path. |
| P2 | `tests/test_llm.py::TestInvokeModel::test_ollama_strips_thinking_output` | both | Mostly tests stdout post-processing, but still relies on the old subprocess patch target. |
| P2 | `tests/test_llm.py::TestInvokeModel::test_ollama_no_thinking_markers_unchanged` | both | Same: behavior-focused, but coupled through the old patch target. |
| P2 | `tests/test_llm.py::TestInvokeModel::test_ollama_incomplete_thinking_unchanged` | both | Same. |

## Adjacent risk not caught by the exact grep

`tests/test_llm.py::TestInvokeApi::test_timeout_reraises_as_timeout_expired` does not patch `reasons.llm.subprocess.run`, but it asserts:

```python
with pytest.raises(subprocess.TimeoutExpired):
    _invoke_api(...)
```

So if the boundary refactor changes how CLI/API timeout exceptions are represented, this test may also need migration even though it is not enumerated by the two target grep patterns.

## Recommended migration direction

Do not migrate by merely changing the patch string from:

```python
patch("reasons.llm.subprocess.run")
```

to another stdlib target. That still couples tests to implementation details. Prefer migrating to a stable boundary seam:

- `reasons.llm._run_cli(cmd, prompt, timeout, env)` or a small `CliRunner` object.
- A dedicated `which`/binary-resolution boundary.
- Tests should patch the **new boundary**, not `shutil.which` or `subprocess.run` directly.

Priority should go to the tests that assert low-level subprocess call arguments first: `test_invokes_subprocess`, `test_strips_claudecode_env`, `test_missing_binary_raises`, and `test_nonzero_exit_raises`.

## Topics to Explore

- [file] `tests/test_llm.py` — Lines 301–470 were not returned; inspect the remainder for additional `reasons.llm.shutil.which` or `reasons.llm.subprocess.run` patches.
- [function] `reasons/llm.py:invoke_model` — The CLI path that checks `shutil.which(binary)` and then invokes the subprocess; identify the exact seam to extract.
- [general] `boundary-refactor-design` — Decide whether to extract a CLI runner function, a runner class, or dependency-injected callables.
- [function] `reasons/llm.py:_parse_cli_json` — Parses CLI JSON and records cost; CLI-output tests may need to preserve this contract after runner extraction.
- [general] `subprocess-timeout-semantics` — Ensure `subprocess.TimeoutExpired` behavior remains stable for both CLI and API timeout paths.

## Beliefs

- `grep-results-contradict-test-file` — The grep observations report zero matches, but `tests/test_llm.py` contains literal `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` patch strings.
- `visible-test-file-patches-llm-module-internals` — The boundary mocks target `reasons.llm` module attributes, not a dedicated CLI-runner boundary.
- `visible-test-file-has-ten-which-patches-and-nine-run-patches` — In the returned 1–300 line segment, 10 tests patch `reasons.llm.shutil.which` and 9 patch `reasons.llm.subprocess.run`.
- `invoke-model-requires-shutil-which` — `reasons/llm.py:invoke_model` calls `shutil.which(binary)` before invoking CLI models.
- `test-read-is-truncated` — `tests/test_llm.py` has 470 lines, but only lines 1–300 were supplied, so the visible enumeration is not exhaustive.
