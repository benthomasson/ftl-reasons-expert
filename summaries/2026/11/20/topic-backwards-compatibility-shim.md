---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Explore the tradeoffs of wildcard re-exports versus explicit re-exports, especially around `__all__` and type checking.

# Wildcard vs. Explicit Re-exports

## What the codebase actually does

The shim layer under `reasons_lib/` is built entirely on wildcard re-exports:

- `reasons_lib/__init__.py:2` — `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py:2`, `reasons_lib/ask.py:2`, … — `from reasons.<module> import *  # noqa: F401,F403`
- 33 wildcard `import *` sites total (see `grep_wildcard_imports`).

Crucially, `grep_all_exports` shows **zero `__all__` declarations anywhere in the repo**, and `grep_type_checking` shows **zero `TYPE_CHECKING` guards**. `pyproject.toml` is a lean 35-line file with no `[tool.mypy]`, `[tool.pyright]`, or `py.typed` entry — just a setuptools package list at `pyproject.toml:31` that ships both `reasons` and `reasons_lib`.

That combination — wildcard imports, no `__all__`, no type-checker config — is the worst-case combination for the tradeoffs below.

## How `from reasons import *` actually behaves here

Without an `__all__`, star-import pulls in **every module-level binding that doesn't start with `_`**. Look at the top of `reasons/__init__.py:3`:

```python
from dataclasses import dataclass, field
```

`Justification`, `Node`, and `Nogood` are the intended public surface, but `dataclass` and `field` are also module-level names here (they're used directly to decorate and build the dataclasses). Because neither starts with `_` and no `__all__` exists, they get re-exported too. That means `reasons_lib.dataclass` and `reasons_lib.field` exist as accidental public API — a classic wildcard-leak symptom.

Compare with the internal cross-package imports:

- `reasons/mcp_server.py:23` — `from reasons import api` (submodule, fine).
- `reasons/pg.py:943` — `from reasons.api import _STOP_WORDS`
- `reasons/pg.py:1324` — `from reasons.api import _TOPIC_STOP_WORDS`

These underscore-prefixed internals are reached by explicit import path, not via wildcard. That is the *intended* mechanism working correctly — but it also means `reasons_lib.pg` never re-exports `_STOP_WORDS` to a downstream user who reaches through the shim (`from reasons_lib.pg import ...`). Wildcard turns "private by underscore convention" into "invisible from the shim," which is usually what you want but is a silent asymmetry that isn't documented anywhere.

## Wall of tradeoffs

### Wildcard re-export (`from reasons import *`)

**Upsides**
- Zero maintenance. New public class added to `reasons/__init__.py` → automatically surfaces in `reasons_lib`, which is exactly the goal of a backwards-compat shim (`reasons_lib/__init__.py:1` docstring says as much).
- Small diff footprint for a whole compatibility package (33 shim files × ~2 lines).

**Downsides**
- **Namespace pollution, demonstrated live.** `dataclass` and `field` leak into `reasons_lib` because `reasons/__init__.py:3` imports them and there's no `__all__`.
- **No contract.** Nothing in the code declares what a shim is *supposed* to expose. Renaming `Justification → JustificationRecord` in `reasons/__init__.py` will silently break every shim consumer without any signal at the shim layer.
- **Wildcard-import unavailability of dunders/underscore-only.** `_STOP_WORDS` and `_TOPIC_STOP_WORDS`, used internally at `reasons/pg.py:943` and `reasons/pg.py:1324`, aren't exposed via `reasons_lib.api`. That may be deliberate, but it means any accidental external dependency on them is silently unshimmable.

### Explicit re-export (`from reasons import Justification, Node, Nogood` + `__all__`)

**Upsides**
- `__all__ = ["Justification", "Node", "Nogood"]` in `reasons/__init__.py` would make wildcard re-export **safe** — `from reasons import *` then forces `dataclass`/`field` to stay behind.
- Type checkers (mypy, pyright) treat `__all__` as the authoritative public surface. Under `--no-implicit-reexport` (mypy) or `reportPrivateImportUsage` (pyright strict), `from reasons_lib import Node` resolves correctly only if the shim was explicit **or** the origin has `__all__`.
- Static analyzers can warn when a downstream file imports a name the shim no longer re-exports — a real refactor safety net.

**Downsides**
- Each `reasons_lib/*.py` shim needs to enumerate names. With 33 shim modules, that's substantial churn the moment a public symbol is added, and the shim can silently drift from the "real" package if a maintainer forgets.
- Duplicated source of truth: `reasons/__init__.py`'s class list and `reasons_lib/__init__.py`'s re-export list must stay in lockstep.
- Tests must be written against the shim explicitly (see "Topics to Explore") to catch drift.

### The `__all__` middle ground

`__all__` in `reasons/__init__.py` gives most of the explicit-reexport benefits while keeping the wildcard shims short:

```python
__all__ = ["Justification", "Node", "Nogood"]
```

Then `from reasons import *` in `reasons_lib/__init__.py` automatically becomes a curated re-export and `dataclass`/`field` stop leaking. No changes required to the 33 shim files. This is the highest-leverage single change in the current design.

## Type-checking consequences in this specific repo

Because there is:
- no `__all__`,
- no `py.typed` marker,
- no `[tool.mypy]` / `[tool.pyright]` section in `pyproject.toml`,
- no `TYPE_CHECKING` guards anywhere,

…the shim package `reasons_lib` is effectively invisible to strict type analysis:

1. **Downstream strictness.** A user on pyright `strict` importing `from reasons_lib import Node` gets a `reportUnknownVariableType` or falls back to `Any` unless the origin package is typed with `py.typed` and a resolvable `__all__`. Neither is present.
2. **In-package analysis.** Since `reasons/pg.py:943` and `reasons/pg.py:1324` import internals via explicit paths, those resolve fine — mypy would actually *prefer* this style over wildcard.
3. **No signal on renames.** A rename inside `reasons/__init__.py` (e.g., removing `Nogood`) produces zero type errors in `reasons_lib/*.py` — the wildcard just stops producing it, and external users discover it at runtime.

The safest posture with the current layout is: explicit imports *inside the `reasons` package* (which this codebase already does, at `reasons/mcp_server.py:23`, `reasons/pg.py:943`, `reasons/pg.py:1324`), plus adding `__all__ = [...]` at `reasons/__init__.py` so the wildcard shims become curated without needing to be touched.

## What the observations do *not* cover

- Contents of `reasons/api.py`, `reasons/ask.py`, etc., so we can't judge whether their module-level namespaces leak other helpers the way `reasons/__init__.py` leaks `dataclass`/`field`.
- Whether any user code actually still imports from `reasons_lib` (would need `grep_public_package_imports` filtered for `reasons_lib` **consumers**, not the shim itself).
- Whether the shim package has compatibility tests.

## Topics to Explore

- [file] `reasons/api.py` — Check whether its module namespace leaks helper imports the same way `reasons/__init__.py:3` leaks `dataclass`/`field` through `reasons_lib/api.py:2`.
- [file] `reasons_lib/__init__.py` — The 3-line shim that currently exposes the entire `reasons` module surface plus `dataclass`/`field`.
- [general] `compat-shim-test-coverage` — Look for pytest cases that import every name through `reasons_lib` to detect wildcard-re-export regressions.
- [general] `add-reasons-all` — Proposed one-line `__all__` in `reasons/__init__.py:1` that would curate wildcard re-exports without editing 33 shim files.
- [general] `pyright-strict-vs-shim` — Run pyright under `strict` against a consumer that imports from `reasons_lib` to see what falls back to `Any`.

## Beliefs

- `reasons-lib-reexports-leak-dataclass-field` — Because `reasons/__init__.py:3` imports `dataclass` and `field` and no `__all__` exists, `from reasons import *` at `reasons_lib/__init__.py:2` leaks both names to consumers.
- `no-all-anywhere-forces-name-based-filtering` — With zero `__all__` declarations repo-wide (`grep_all_exports` total 0), every wildcard re-export in `reasons_lib/*.py` filters only on the underscore prefix rule.
- `underscore-internals-require-explicit-import` — `_STOP_WORDS` and `_TOPIC_STOP_WORDS` used at `reasons/pg.py:943` and `reasons/pg.py:1324` are reachable only via explicit `from reasons.api import _STOP_WORDS`, never through the wildcard shims.
- `no-type-tooling-config-blocks-static-verification` — `pyproject.toml` has no `[tool.mypy]`, `[tool.pyright]`, or `py.typed` marker, so the `reasons_lib` shim cannot be statically verified as faithfully re-exporting `reasons`.
- `shim-package-is-shipped` — `[tool.setuptools] packages = ["reasons", "reasons_lib"]` at `pyproject.toml:31` guarantees the wildcard shims ship in the wheel, so their leakage is a public API property, not an internal detail.
