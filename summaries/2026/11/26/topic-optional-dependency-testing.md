---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Explore patterns for testing missing-extra error messages without requiring package uninstalls.

The observations show that missing-extra errors are spread across several optional modules and that the current tests likely depend on the environment rather than simulating a missing dependency. The main testability issue is that optional imports are handled ad hoc in each module, with no central helper and no existing import-monkeypatching pattern.

## What the code currently does

`pyproject.toml` declares optional dependency groups such as `cluster`, `mcp`, `api`, `vertex`, `openai`, and `pg` under `[project.optional-dependencies]`.

The missing-dependency handling is repeated in each module:

| Module | Import guard | Raise/message site | Extra referenced |
|---|---|---|---|
| `reasons/cluster.py` | `reasons/cluster.py:17` | `reasons/cluster.py:28` | `ftl-reasons[cluster]` |
| `reasons/llm.py` | `reasons/llm.py:23`, `:30`, `:37`, `:44` | `reasons/llm.py:208`, `:215`, `:252` | `ftl-reasons[api]`, `ftl-reasons[vertex]`, `ftl-reasons[openai]` |
| `reasons/mcp_client.py` | `reasons/mcp_client.py:17` | `reasons/mcp_client.py:23` | `mcp>=1.0` |
| `reasons/mcp_server.py` | `reasons/mcp_server.py:17` | `reasons/mcp_server.py:18` | `ftl-reasons[mcp]` |
| `reasons/pg.py` | `reasons/pg.py:18` | `reasons/pg.py:92` | `psycopg[binary]>=3.1` |

Some of these raises appear to be import-time failures, while others may be guarded by module-level sentinels and only raise when a specific function is called. The observations do not show the exact function names around `reasons/llm.py:208`, `reasons/llm.py:215`, `reasons/llm.py:252`, or `reasons/pg.py:92`, so you need to inspect those modules to know which entry point to invoke in tests.

## What the current tests cover

There is already some direct assertion of these messages:

- `tests/test_cluster.py:28` uses `pytest.raises(ImportError, match="ftl-reasons\\[cluster\\]")`.
- `tests/test_llm.py:238` matches `langchain-anthropic`.
- `tests/test_llm.py:243` matches `langchain-google-vertexai`.
- `tests/test_cluster.py:14` catches `ImportError`, which suggests the test may skip or branch when the cluster extra is unavailable.

However, the observations show no existing use of `monkeypatch`, `sys.modules`, or `builtins.__import__` in tests (`find_tests_monkeypatching_imports` returned zero matches). That means the current tests likely only exercise the missing-extra branch when the optional package happens to be absent from the environment. If the package is installed, those tests may not exercise the error path at all.

The `find_tests_checking_missing_messages` query returned zero matches even though `tests/test_cluster.py:28` and `tests/test_llm.py:238/243` clearly contain related assertions. Treat that query result as incomplete or mis-scoped rather than as evidence that no such tests exist.

## Patterns for testing missing-extra messages without uninstalling

### 1. Block imports with `sys.modules` and reload the module

For module-level `try/except ImportError` guards, you can temporarily poison `sys.modules` so an import fails, then reload the module under test.

```python
import importlib
import sys
import pytest

def test_cluster_missing_extra(monkeypatch):
    import reasons.cluster as cluster

    monkeypatch.setitem(sys.modules, "sentence_transformers", None)
    monkeypatch.setitem(sys.modules, "sklearn", None)

    with pytest.raises(ImportError, match=r"ftl-reasons\[cluster\]"):
        importlib.reload(cluster)
```

If the module catches the import error and defers the raise to a function call, wrap the actual entry point instead of `importlib.reload`. For example, if `reasons/cluster.py:28` is only reached from a cluster function, call that function after reloading.

This pattern is fast and works without uninstalling anything. The main caveat is module state: `importlib.reload` mutates the existing module object, so use `monkeypatch` and consider reloading again after the test or isolating the reload in a subprocess.

### 2. Monkeypatch the dependency sentinel for call-time raises

`reasons/llm.py` has several `except ImportError` blocks at lines 23, 30, 37, and 44, followed by raises at lines 208, 215, and 252. That structure suggests the module may store a sentinel such as `ChatAnthropic = None` when the import fails, then check it later.

For that style, monkeypatch the sentinel directly:

```python
def test_llm_missing_anthropic(monkeypatch):
    import reasons.llm as llm

    monkeypatch.setattr(llm, "ChatAnthropic", None, raising=False)

    with pytest.raises(ImportError, match="langchain-anthropic"):
        llm.some_entrypoint_that_checks_the_sentinel()
```

Replace `some_entrypoint_that_checks_the_sentinel` with the actual function or class method that guards the raise at `reasons/llm.py:208`. This is the cleanest pattern when the code already converts a missing import into a `None` sentinel.

### 3. Use a scoped import blocker for import-time failures

For modules like `reasons/mcp_server.py`, where the `except ImportError` block at line 17 immediately raises at line 18, a temporary import blocker is useful. It also avoids relying on the package being absent globally.

```python
import sys

class BlockImport:
    def __init__(self, *names):
        self.names = set(names)

    def find_spec(self, fullname, path=None, target=None):
        if fullname.split(".")[0] in self.names:
            raise ModuleNotFoundError(f"No module named {fullname!r}")
        return None

def test_mcp_server_missing_extra(monkeypatch):
    import importlib

    monkeypatch.delitem(sys.modules, "mcp", raising=False)
    blocker = BlockImport("mcp")
    monkeypatch.setattr(sys, "meta_path", [blocker, *sys.meta_path])

    with pytest.raises(ImportError, match=r"ftl-reasons\[mcp\]"):
        importlib.import_module("reasons.mcp_server")
```

This pattern is closer to a real missing package and works well for import-time failures. For call-time failures, combine it with `importlib.reload` or invoke the guarded entry point.

### 4. Parameterize across extras and messages

Because the messages follow a consistent pattern, a parameterized test matrix is a good fit:

```python
@pytest.mark.parametrize(
    "module_name,blocked_name,expected_extra",
    [
        ("reasons.cluster", "sentence_transformers", r"ftl-reasons\[cluster\]"),
        ("reasons.mcp_server", "mcp", r"ftl-reasons\[mcp\]"),
        ("reasons.pg", "psycopg", r"psycopg\[binary\]>=3\.1"),
    ],
)
def test_missing_extra_message(module_name, blocked_name, expected_extra, monkeypatch):
    ...
```

This keeps the test intent visible and makes it easy to add coverage for `api`, `vertex`, and `openai` once the exact guard points in `reasons/llm.py` are identified.

### 5. Consider a small optional-dependency helper

The observations show no existing helper matching `optional|extras|require|missing` (`find_optional_dependency_helper` returned zero matches). If you want to make these tests much simpler, introduce a helper such as `_require_extra("cluster", "sentence_transformers", ...)` and test the helper directly as well as one real call site. That reduces the need to reload whole modules and makes the error text consistent.

## What is missing from the observations

The observations do not include the exact function bodies around the raise sites, so we cannot yet say whether each error is raised at import time or only when a specific function is called. We also do not know whether `tests/test_cluster.py` skips when the extra is installed or whether it always reaches the assertion at `tests/test_cluster.py:28`. To implement the patterns above, inspect `reasons/cluster.py`, `reasons/llm.py`, `reasons/mcp_client.py`, `reasons/mcp_server.py`, `reasons/pg.py`, `tests/test_cluster.py`, and `tests/test_llm.py` directly.

## Topics to Explore

- [file] `reasons/cluster.py` — Determine whether the `ImportError` at line 28 is raised during import or only from a cluster entry point.
- [file] `tests/test_cluster.py` — See how the current test at line 28 handles environments where the cluster extra is already installed.
- [file] `tests/test_llm.py` — Compare the assertions at lines 238 and 243 with the actual guard symbols and raise sites in `reasons/llm.py`.
- [general] `import-blocker-fixture` — Design a reusable pytest fixture that temporarily blocks optional modules and restores `sys.modules` and `sys.meta_path`.
- [file] `pyproject.toml` — Map each optional extra name to the import names that should trigger the corresponding missing-extra message.

## Beliefs

- `optional-extras-are-declared-in-pyproject` — `pyproject.toml` declares optional dependency groups including `cluster`, `mcp`, `api`, `vertex`, `openai`, `pg`, and `llm`.
- `missing-extra-errors-embed-pip-install-commands` — Raise sites in `reasons/cluster.py:28`, `reasons/llm.py:208/215/252`, `reasons/mcp_client.py:23`, `reasons/mcp_server.py:18`, and `reasons/pg.py:92` include `pip install` instructions in their `ImportError` messages.
- `optional-imports-are-guarded-with-try-except` — Optional imports are caught with `except ImportError` in `reasons/cluster.py:17`, `reasons/llm.py:23-44`, `reasons/mcp_client.py:17`, `reasons/mcp_server.py:17`, and `reasons/pg.py:18`.
- `existing-tests-assert-some-missing-extra-messages` — `tests/test_cluster.py:28` matches `ftl-reasons[cluster]`, while `tests/test_llm.py:238` and `tests/test_llm.py:243` match `langchain-anthropic` and `langchain-google-vertexai`.
- `no-import-monkeypatch-helper-exists` — The observations found no tests using `monkeypatch`, `sys.modules`, or `builtins.__import__`, and no optional-dependency helper matching `optional|extras|require|missing`.
