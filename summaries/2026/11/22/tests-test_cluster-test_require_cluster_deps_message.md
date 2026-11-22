---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: test_require_cluster_deps_message in tests/test_cluster.py

`test_require_cluster_deps_message` is a pytest test function in `tests/test_cluster.py`. It is not production logic; it verifies the missing-optional-dependency error path for `reasons.cluster._require_cluster_deps`.

### Purpose

The test exists to ensure that when the optional clustering dependencies are not installed, calling `reasons.cluster._require_cluster_deps()` raises an `ImportError` whose message points the user at the install extra:

```text
ftl-reasons[cluster]
```

Most clustering tests in this file are skipped when cluster dependencies are missing, via `skip_no_cluster`. This test is intentionally different: it is the test that exercises the missing-dependency path. Therefore it skips when dependencies are installed and only runs when they are absent.

### Contract

Preconditions:

- `HAS_CLUSTER_DEPS` must accurately indicate whether the optional clustering dependencies are available.
- `reasons.cluster._require_cluster_deps` must be importable even when the optional dependencies are absent.
- When `HAS_CLUSTER_DEPS` is false, `_require_cluster_deps()` must raise `ImportError`.

Postconditions:

- If `HAS_CLUSTER_DEPS` is true, the test is skipped.
- If `HAS_CLUSTER_DEPS` is false, the test passes only if `_require_cluster_deps()` raises an `ImportError` whose string representation matches the regex `ftl-reasons\[cluster\]`.
- No production state is mutated.

Invariants:

- The expected error message contains the package extra name `ftl-reasons[cluster]`.
- The helper function is the dependency-checking seam under test.
- The test does not require the actual optional packages to be installed; in fact, it requires them not to be installed for the meaningful path.

### Parameters

This function takes no parameters.

Its behavior depends on environment state through the module-level `HAS_CLUSTER_DEPS` flag and on the importability of `reasons.cluster._require_cluster_deps`.

Edge cases:

- Dependencies installed: test skips.
- Dependencies missing: test expects `ImportError`.
- `_require_cluster_deps` import fails: test errors before entering `pytest.raises`.
- `_require_cluster_deps()` raises no exception: test fails.
- `_require_cluster_deps()` raises an `ImportError` without the expected substring: test fails.
- `_require_cluster_deps()` raises a non-`ImportError`: test errors/fails because only `ImportError` is caught.

### Return Value

It returns `None` when it passes. As a pytest test, its meaningful outcomes are represented by pytest exceptions:

- `pytest.skip(...)` raises `Skipped`.
- `pytest.raises(...)` fails the test if the expected exception is missing or mismatched.
- Unexpected exceptions propagate as test errors.

Callers should not call this function directly expecting a useful return value; pytest discovers and executes it.

### Algorithm

1. Check `HAS_CLUSTER_DEPS`.
2. If it is true, call `pytest.skip("deps are installed, can't test missing-dep path")`. The rest of the function is not executed.
3. If it is false, import `_require_cluster_deps` from `reasons.cluster`.
4. Enter a `pytest.raises(ImportError, match="ftl-reasons\\[cluster\\]")` context.
5. Call `_require_cluster_deps()`.
6. `pytest.raises` verifies that an `ImportError` was raised and that its string representation matches the regex. The regex matches the literal substring `ftl-reasons[cluster]`.
7. If the expected exception and message are seen, the test passes.

Note: the docstring says the error message mentions the install command, but the assertion only checks for `ftl-reasons[cluster]`. It does not assert that the message contains `pip install` or the exact full command.

### Side Effects

- May call `pytest.skip`, which records a skipped test.
- Imports `reasons.cluster._require_cluster_deps` inside the test. This can initialize module state and populate `sys.modules`, but it does not intentionally perform I/O, network calls, or filesystem mutation.
- Uses a pytest exception-catching context manager.
- Does not mutate beliefs, caches, or clustering data.

### Error Handling

- `pytest.skip(...)` handles the “dependencies are installed” case as a skip, not a failure.
- `pytest.raises(ImportError, ...)` catches only `ImportError` from `_require_cluster_deps()`.
- If `_require_cluster_deps()` does not raise, `pytest.raises` fails the test.
- If it raises an `ImportError` with a non-matching message, the test fails.
- If it raises another exception type, that exception propagates as a test error.
- If `from reasons.cluster import _require_cluster_deps` itself fails, that import error occurs outside the `pytest.raises` block and is not handled by this test.

### Usage Patterns

Typical execution is through pytest:

```bash
pytest tests/test_cluster.py::test_require_cluster_deps_message
```

In a fully installed environment, this test skips. To exercise it, run in an environment where the cluster extra is not installed. The caller should not rely on this test to validate the installed dependency path; other tests in the file handle that path when `HAS_CLUSTER_DEPS` is true.

### Dependencies

- `pytest` for test discovery, skipping, and exception assertion.
- `reasons.cluster._require_cluster_deps`, the helper under test.
- The module-level `HAS_CLUSTER_DEPS` flag defined/imported in `tests/test_cluster.py`.
- Indirectly, the optional `ftl-reasons[cluster]` extra and the packages it installs, such as `sentence-transformers` and `scikit-learn`.
- Full-file context also depends on `skip_no_cluster`, which skips the other clustering tests when dependencies are unavailable.

### Assumptions Not Enforced by the Type System

- `HAS_CLUSTER_DEPS` must be truthful. The full file catches any `ImportError` around the grouped import and sets `HAS_CLUSTER_DEPS = False`, so an unrelated import failure could be misclassified as “cluster deps missing.”
- `_require_cluster_deps` must remain importable even when optional dependencies are absent.
- The helper must raise `ImportError` synchronously, not some other exception type.
- The error message must contain the literal extra name `ftl-reasons[cluster]`; the test does not validate the exact install command or the full message.
- The missing-dependency path is only meaningful if the dependencies are actually absent, not merely if some import failed.

## Topics to Explore

- [file] `reasons/cluster.py` — Inspect how optional clustering dependencies are guarded and how `HAS_CLUSTER_DEPS` is defined.
- [function] `reasons/cluster.py:_require_cluster_deps` — See the actual `ImportError` message and dependency-checking logic under test.
- [general] `pytest.raises match semantics` — Understand that `match` uses `re.search` against the exception string and why the brackets are escaped.
- [file] `tests/test_cluster.py` — Review how `skip_no_cluster` and `HAS_CLUSTER_DEPS` gate the rest of the clustering suite.
- [general] `optional-dependency-testing` — Explore patterns for testing missing-extra error messages without requiring package uninstalls.

## Beliefs

- `test-skips-when-cluster-deps-installed` — If `HAS_CLUSTER_DEPS` is true, `test_require_cluster_deps_message` calls `pytest.skip` and does not exercise `_require_cluster_deps()`.
- `missing-cluster-deps-test-requires-import-error` — When `HAS_CLUSTER_DEPS` is false, the test fails unless `_require_cluster_deps()` raises `ImportError`.
- `cluster-import-error-message-requires-extra-name` — The expected `ImportError` string must contain the literal text `ftl-reasons[cluster]`.
- `cluster-missing-deps-path-requires-importable-helper` — `_require_cluster_deps` must remain importable from `reasons.cluster` even when optional cluster dependencies are absent.
- `cluster-deps-flag-is-set-false-by-any-initial-import-error` — In `tests/test_cluster.py`, the top-level `except ImportError` assigns `HAS_CLUSTER_DEPS = False` for any `ImportError` during the grouped import, not only for a missing cluster dependency.
