---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Determine how to emit and test `DeprecationWarning` from a wildcard-reexport shim package.

## Current state

`reasons_lib` is currently a pure wildcard-reexport shim, with no warning machinery:

- `reasons_lib/__init__.py:1` is the shim docstring.
- `reasons_lib/__init__.py:2` is `from reasons import *  # noqa: F401,F403`.
- Submodule shims do the same, e.g. `reasons_lib/api.py:2` is `from reasons.api import *`.
- The search results show **0** `DeprecationWarning` matches and **0** `warnings.warn` calls, so no warning is emitted today.
- The real package is `reasons`; `reasons/__init__.py` defines the core dataclasses such as `Justification`, `Node`, and `Nogood`.
- `pyproject.toml` declares both packages in `[tool.setuptools]` as `packages = ["reasons", "reasons_lib"]`, so `reasons_lib` is shipped as a top-level compatibility package.

So the task is to add module-import-time warnings to the shim package, not to the real `reasons` package.

## How to emit the warning

### Direct approach

At the top of each shim module, emit a `DeprecationWarning` before or immediately after the wildcard import. For example:

```python
# reasons_lib/__init__.py
"""Backwards-compatibility shim — use ``import reasons`` instead."""
import warnings

warnings.warn(
    "`reasons_lib` is deprecated; import from `reasons` instead.",
    DeprecationWarning,
    stacklevel=2,
)

from reasons import *  # noqa: F401,F403
```

For a submodule shim:

```python
# reasons_lib/api.py
import warnings

warnings.warn(
    "`reasons_lib.api` is deprecated; use `reasons.api` instead.",
    DeprecationWarning,
    stacklevel=2,
)

from reasons.api import *  # noqa: F401,F403
```

Important placement notes:

- Put the warning in the shim boundary, i.e. `reasons_lib/*`, not in `reasons/*`.
- Emitting it before the wildcard import means the warning is raised even if the real package import later fails, and it makes “warnings as errors” behave like a hard deprecation.
- `stacklevel=2` avoids pointing the warning at the `warnings.warn` line itself. If you wrap the call in a helper, increase `stacklevel` appropriately so the warning does not point at the helper.

### Reusable helper approach

Because there are many shim modules, a helper is cleaner:

```python
# reasons_lib/_deprecation.py
import warnings


def warn_shim(module_name: str, replacement: str) -> None:
    warnings.warn(
        f"{module_name} is deprecated; use {replacement} instead.",
        DeprecationWarning,
        stacklevel=3,
    )
```

Then:

```python
# reasons_lib/__init__.py
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from ._deprecation import warn_shim

warn_shim(__name__, "reasons")

from reasons import *  # noqa: F401,F403
```

```python
# reasons_lib/api.py
from ._deprecation import warn_shim

warn_shim(__name__, "reasons.api")

from reasons.api import *  # noqa: F401,F403
```

There are 33 wildcard-reexport matches in the observations, including `reasons_lib/__init__.py`, `reasons_lib/api.py`, `reasons_lib/cli.py`, `reasons_lib/pg.py`, etc. A helper keeps the message format consistent across all of them.

### Visibility caveat

`DeprecationWarning` is ignored by Python’s default warning filters outside `__main__` in many contexts. That is fine for a library deprecation, but tests must explicitly capture it, and users who want to see it need one of:

```bash
python -Wd your_script.py
```

or:

```bash
PYTHONWARNINGS=always::DeprecationWarning python your_script.py
```

Do not globally change warning filters inside the library.

## How to test it

The main testing issue is Python’s module cache: once `reasons_lib` is imported, a later `import reasons_lib` will not re-execute `__init__.py` and will not re-emit the warning. Tests must either purge `sys.modules`, use `importlib.reload`, or run a fresh subprocess.

### Recommended pytest test using fresh imports

```python
# tests/test_deprecation_warnings.py
import importlib
import sys

import pytest


def fresh_import(module_name: str):
    for name in list(sys.modules):
        if (
            name == module_name
            or name.startswith(module_name + ".")
            or name == "reasons_lib"
            or name.startswith("reasons_lib.")
        ):
            sys.modules.pop(name, None)
    return importlib.import_module(module_name)


def test_reasons_lib_root_emits_deprecation_warning():
    with pytest.warns(
        DeprecationWarning,
        match=r"reasons_lib.*deprecated",
    ):
        fresh_import("reasons_lib")


def test_reasons_lib_api_emits_deprecation_warning():
    with pytest.warns(
        DeprecationWarning,
        match=r"reasons_lib\.api.*deprecated",
    ):
        fresh_import("reasons_lib.api")
```

Caveat: importing `reasons_lib.api` first imports the parent package `reasons_lib`, so the root shim warning may also fire. If you want to assert the exact set of warnings, use `warnings.catch_warnings(record=True)` instead of only `pytest.warns`.

### Stronger assertion with `catch_warnings`

```python
import importlib
import sys
import warnings


def test_reasons_lib_api_emits_specific_warning():
    for name in list(sys.modules):
        if name == "reasons_lib" or name.startswith("reasons_lib."):
            sys.modules.pop(name, None)

    with warnings.catch_warnings(record=True) as caught:
        warnings.simplefilter("always")
        importlib.import_module("reasons_lib.api")

    messages = [
        str(w.message)
        for w in caught
        if issubclass(w.category, DeprecationWarning)
    ]
    assert any("reasons_lib.api" in message for message in messages)
```

### Verify the shim still reexports correctly

The warning should not break compatibility. A useful test asserts both warning and symbol identity:

```python
def test_shim_still_reexports_reasons_symbols():
    with pytest.warns(DeprecationWarning):
        importlib.import_module("reasons_lib")

    import reasons
    import reasons_lib

    assert reasons_lib.Node is reasons.Node
    assert reasons_lib.Justification is reasons.Justification
```

### Subprocess-style test

For a more isolated import test:

```python
import subprocess
import sys


def test_reasons_lib_warning_in_clean_interpreter():
    code = "import reasons_lib"
    result = subprocess.run(
        [sys.executable, "-W", "always::DeprecationWarning", "-c", code],
        capture_output=True,
        text=True,
        check=True,
    )
    assert "DeprecationWarning" in result.stderr
    assert "reasons_lib" in result.stderr
```

Use this when you want to prove the warning appears on a real first import without mutating the current pytest process’s `sys.modules`.

## Summary

To emit and test `DeprecationWarning` from this shim:

1. Add `warnings.warn(..., DeprecationWarning, ...)` to each `reasons_lib` shim module, especially `reasons_lib/__init__.py:2` and the per-module shims such as `reasons_lib/api.py:2`.
2. Prefer a shared `reasons_lib/_deprecation.py` helper for consistent messages and `stacklevel`.
3. Do not modify `reasons/__init__.py`; the real package should remain warning-free.
4. Test with fresh imports via `sys.modules` cleanup, `importlib.import_module`, and `pytest.warns` or `warnings.catch_warnings`.
5. Remember that importing a submodule also executes the parent `reasons_lib/__init__.py`, so submodule tests may observe both root and submodule warnings.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Top-level shim where the first `DeprecationWarning` should be emitted before the wildcard reexport.
- [file] `reasons_lib/api.py` — Representative submodule shim showing the repeated wildcard-reexport pattern that needs a warning.
- [function] `reasons_lib/_deprecation.py:warn_shim` — Suggested helper for consistent warning messages and `stacklevel` handling.
- [general] `pytest-warning-capture-and-module-cache` — How `sys.modules`, `importlib.reload`, and `pytest.warns` interact when testing import-time warnings.
- [general] `python-warning-filters` — Default visibility of `DeprecationWarning` and how `-Wd` / `PYTHONWARNINGS` affect real user imports.

## Beliefs

- `reasons-lib-root-is-a-wildcard-reexport-shim` — `reasons_lib/__init__.py` line 2 is `from reasons import *`.
- `reasons-lib-submodules-are-wildcard-reexport-shims` — Shim modules such as `reasons_lib/api.py` line 2 use `from reasons.api import *`.
- `reasons-lib-currently-emits-no-warnings` — Repository searches found zero `DeprecationWarning` and zero `warnings.warn` occurrences.
- `reasons-lib-is-installed-as-a-top-level-package` — `pyproject.toml` `[tool.setuptools]` lists both `reasons` and `reasons_lib`.
- `reasons-real-package-defines-core-dataclasses` — `reasons/__init__.py` defines `Justification`, `Node`, and `Nogood`.
