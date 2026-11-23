---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Try deleting `reasons_lib/__init__.py` locally; note the shim still imports but star-re-export breaks (PEP 420 behavior)

## Explaining: Deleting `reasons_lib/__init__.py` and the star re-export

### What the shim does today

`reasons_lib/__init__.py` is a 2‑line compatibility shim (grep reports the import on **line 2**, with the docstring on line 1):

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

Everything else in `reasons_lib/` is a per-module shim: 32 sibling files (`api.py`, `ask.py`, `cli.py`, …) each containing:

```python
from reasons.<mod> import *  # noqa: F401,F403
```

`pyproject.toml` declares both packages in `[tool.setuptools] packages = ["reasons", "reasons_lib"]`, so both are expected to be importable top-level packages.

### What PEP 420 changes

Before PEP 420, a directory needed `__init__.py` to be importable. After PEP 420, a directory **without** `__init__.py` is still importable as an implicit **namespace package**. So:

- `import reasons_lib` **succeeds** even after deleting `reasons_lib/__init__.py`.
- Because there is no `__init__.py`, its body **never executes** — meaning the line `from reasons import *` on line 2 is never run.

That single missing execution is the whole failure:

- The `reasons_lib` **module object has no top-level attributes** coming from `reasons` — so `reasons_lib.Justification`, `reasons_lib.Node`, `reasons_lib.Nogood` (and the leaked `dataclass` / `field`, see below) all raise `AttributeError`.
- `from reasons_lib import Justification` raises `ImportError: cannot import name 'Justification' from 'reasons_lib'`.
- `from reasons_lib import *` yields essentially nothing from `reasons`.

`reasons/__init__.py` (59 lines) defines `Justification` (line ~8), `Node` (line ~30), and `Nogood` (line ~54) and has **no `__all__`**, so its `import *` surface currently leaks `dataclass` and `field` too. Those all disappear from the `reasons_lib` top level once `__init__.py` is gone.

### What still works

The 32 submodule shims are **real modules** with a `__init__.py`-free path available via namespace packaging, so:

- `import reasons_lib.api` → finds `reasons_lib/api.py`, runs `from reasons.api import *`, works.
- `from reasons_lib.cli import main` → works.
- `from reasons_lib import api` → works (import machinery falls back to the submodule).

So "the shim still imports but star-re-export breaks" = "submodule shims survive, but the top-level rebinding of `Justification` / `Node` / `Nogood` into `reasons_lib`'s namespace does not."

### Practical footgun to watch for

After deleting `reasons_lib/__init__.py`, clear `reasons_lib/__pycache__/` before experimenting. A stale `__pycache__/__init__.<tag>.pyc` may look like it "should" load, but Python 3.10+ won't treat a source-less `__init__.pyc` as a package initializer — so the directory falls through to namespace-package behavior and nothing runs. If results look inconsistent between shells, that cache is the usual suspect.

### Why this is a real risk

- `tests/` currently has zero references to `reasons_lib` (`grep_reasons_lib_usage_in_tests` → 0 hits), so no test will catch the regression.
- The only `from reasons import *` in the codebase is the shim itself (`grep_star_reexports` → 1 hit, in `reasons_lib/__init__.py:2`). Nothing else relies on the top-level re-export — but downstream users of `reasons_lib.Justification` (which is the intended use of this "use `import reasons` instead" shim) would suddenly break.
- All 33 wildcard imports (`grep_wildcard_imports`) are the shims themselves; none are in the canonical `reasons` package — good, meaning no internal code path depends on the shim.

### Summary mental model

| Action | Result |
|---|---|
| `rm reasons_lib/__init__.py` | Directory becomes PEP 420 namespace package |
| `import reasons_lib` | ✅ succeeds (empty namespace) |
| `reasons_lib.Justification` | ❌ AttributeError (`from reasons import *` never ran) |
| `from reasons_lib import Node` | ❌ ImportError |
| `from reasons_lib import api` | ✅ imports `reasons_lib/api.py` |
| `from reasons_lib.api import X` | ✅ works via `reasons.api` star |
| `import reasons_lib.cli` | ✅ works via `reasons.cli` star |

The point of the exercise: **`__init__.py` is what makes the `reasons_lib` top level a *re-export surface* rather than just a *namespace*.** Deleting it demonstrates that PEP 420 turns a "package with side effects at import" into "an importable directory with no `on_import` hook" — silently, without an ImportError at `import reasons_lib` time.

## Topics to Explore

- [file] `reasons/__init__.py` — The source of truth for what `from reasons import *` exposes; add an `__all__` here to stop `dataclass` and `field` leaking through the shim.
- [file] `reasons_lib/api.py` — A representative of the 32 submodule shims; confirms the `from reasons.<mod> import *` pattern survives `__init__.py` deletion.
- [general] `pep-420-namespace-packages` — The mechanism behind "directory without `__init__.py` still imports"; explains why the failure is silent at import time.
- [general] `__all__-and-wildcard-exports` — Why every top-level star import is fragile without an explicit `__all__` and what changes if you add one to `reasons/__init__.py`.
- [diff] `remove-reasons_lib_init-shim` — The concrete experiment: delete the file, clear `__pycache__`, and verify which of the 32 submodule shims still resolve.

## Beliefs

- `reasons-lib-init-is-a-star-shim` — `reasons_lib/__init__.py` line 2 is exactly `from reasons import *  # noqa: F401,F403`, re-exporting `reasons`' top-level names into the `reasons_lib` namespace.
- `reasons-exports-three-dataclasses-without-all` — `reasons/__init__.py` defines `Justification`, `Node`, and `Nogood` and declares no `__all__`, so `from reasons import *` also leaks `dataclass` and `field`.
- `reasons-lib-requires-init-for-reexport` — The top-level re-export of `Justification`/`Node`/`Nogood` into `reasons_lib` requires `reasons_lib/__init__.py` to exist; without it, PEP 420 makes `reasons_lib` a namespace package with no such attributes.
- `thirty-two-submodule-shims-survive-init-deletion` — The 32 `reasons_lib/<mod>.py` shims (e.g. `reasons_lib/api.py:2` → `from reasons.api import *`) remain importable after deleting `__init__.py` because they are real modules, not namespace attributes.
- `pyproject-binds-release-to-both-packages` — `[tool.setuptools] packages = ["reasons", "reasons_lib"]` in `pyproject.toml` means the distribution contract treats both names as importable packages, even though only `reasons` is required by tests.
