---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Confirm how the `reasons` package re-exports `reasons_lib`; these tests bind to the facade, so a broken re-export would break them.

## What the observations actually show (and a direction correction)

The prompt asks how the `reasons` package re-exports `reasons_lib`. The observations show the **opposite direction**:

- `reasons/__init__.py` (59 lines, full contents available) does **not** import, alias, or reference `reasons_lib` anywhere. It only defines the three dataclasses `Justification`, `Node`, and `Nogood`, and imports `dataclass`/`field` from the stdlib.
- `reasons_lib/__init__.py` (3 lines, full contents) is the shim: `"""Backwards-compatibility shim — use ``import reasons`` instead."""` followed by `from reasons import *  # noqa: F401,F403` (line 2 of that file).

So `reasons_lib` is a **legacy alias package that re-exports `reasons`**, not the other way around. That is the correct mental model for these tests and the packaging.

## How the re-export works mechanically

`reasons_lib/__init__.py:2` runs `from reasons import *`, which does two things at import time:

1. **Executes `reasons/__init__.py`**, binding every non-underscore name in that module's namespace.
2. **Copies those names into `reasons_lib`'s namespace** — nothing more, nothing lazily forwarded.

Because `reasons/__init__.py` has **no `__all__`** and only imports `dataclass` and `field` from `dataclasses`, the transitive surface of `reasons_lib` is:

- `reasons_lib.Justification` (the dataclass defined at the top of `reasons/__init__.py`)
- `reasons_lib.Node`
- `reasons_lib.Nogood`
- Leaked implementation detail: `reasons_lib.dataclass` and `reasons_lib.field` (aliases to `dataclasses.dataclass` / `dataclasses.field`). Harmless but a sign the facade isn't curated.

## What is **not** re-exported

`from reasons import *` only binds names already in `reasons.__dict__` at the moment `reasons_lib` is imported. `reasons/__init__.py` imports **no submodules**, so:

- `reasons_lib.api`, `reasons_lib.pg`, `reasons_lib.cli`, `reasons_lib.network`, etc. **do not exist** as attributes.
- `from reasons_lib.api import init_db` will fail: there is no `reasons_lib/api.py` file, and `from reasons import *` cannot synthesize a submodule alias.
- `reasons/api.py` depends on the facade for one symbol only: `from . import Justification` (near the top of the file). So a broken `reasons/__init__.py` would break `reasons.api` via that import — but that failure path runs through `reasons`, not through `reasons_lib`.

## The test-binding claim is **not verified** by the observations

Two searches returned zero matches:

- `find_tests_importing_facade` (pattern `(from reasons import|import reasons\b)`) → **0 matches**.
- `find_facade_modules_referencing_reasons_lib` (pattern `reasons_lib`) → **0 matches**.

The only test file shown, `tests/conftest.py:29-30`, imports `from reasons.pg import PgApi` **inside the `pg_api` fixture** — a submodule import, not the facade. So the premise "these tests bind to the facade" cannot be confirmed from the provided data. What is confirmed is:

- `reasons/api.py` binds to the facade (`from . import Justification`), so a broken facade breaks `reasons.api`.
- The `reasons_lib` shim binds to `reasons` via a single `from reasons import *`, so a broken facade (e.g., removing `Justification`) would also break `reasons_lib.Justification`.
- The grep patterns used are narrow: `from reasons.api import X` matches neither `from reasons import` nor `import reasons\b`, so zero hits does **not** prove no test touches the facade.

## Packaging ties it together

`pyproject.toml` declares `packages = ["reasons", "reasons_lib"]` under `[tool.setuptools]`, so both packages are installed and importable. That is the only thing keeping the backwards-compat shim alive — no test in the observations requires it, but downstream users may.

## Summary

- Direction: `reasons_lib` → `reasons`, **not** `reasons` → `reasons_lib`.
- Mechanism: `from reasons import *` at `reasons_lib/__init__.py:2`.
- Exposed API: `Justification`, `Node`, `Nogood` (and leaked `dataclass`/`field`); **submodules are not exposed**.
- Fragility: any removal/rename in `reasons/__init__.py` silently narrows `reasons_lib`'s surface.
- Evidence gap: no observation shows a test importing `from reasons import …`; the only visible test import is `from reasons.pg import PgApi`.

## Topics to Explore

- [file] `reasons/__init__.py` — Root of the facade; adding `__all__` here would make `reasons_lib`'s star-import surface explicit and stop leaking `dataclass`/`field`.
- [file] `reasons_lib/__init__.py` — The entire backwards-compat shim; check if any external caller still depends on it before retiring.
- [general] `test-imports-audit` — Re-run a broader grep (e.g., `from reasons\.` across `tests/`) to verify which submodules or facade symbols the tests actually bind to, since the observed patterns missed dotted imports.
- [file] `pyproject.toml` — Confirms `reasons_lib` is shipped as a package; if removed, downstream `import reasons_lib` breaks even though no test in the repo uses it.
- [file] `reasons/api.py` — Representative facade consumer (`from . import Justification`); shows the only confirmed internal binding to the facade.

## Beliefs

- `reasons-lib-re-exports-reasons` — `reasons_lib/__init__.py` performs `from reasons import *`, so it is a shim over `reasons`, not the reverse.
- `reasons-facade-exports-justification-node-nogood` — `reasons/__init__.py` defines exactly `Justification`, `Node`, and `Nogood` as its public surface (no `__all__`).
- `star-import-leaks-dataclass-field` — Because `reasons/__init__.py` lacks `__all__`, `reasons_lib.dataclass` and `reasons_lib.field` resolve to the stdlib imports.
- `reasons-lib-does-not-expose-submodules` — `reasons_lib.api` / `.pg` / `.cli` are absent; only names already bound in `reasons.__dict__` at shim-import time propagate.
- `api-requires-facade-justification` — `reasons/api.py` does `from . import Justification`, so removing that symbol from `reasons/__init__.py` breaks `reasons.api` at import time.
- `packaging-ships-both-packages` — `pyproject.toml` lists `packages = ["reasons", "reasons_lib"]`, so both are installable even though only the shim's `from reasons import *` connects them.
