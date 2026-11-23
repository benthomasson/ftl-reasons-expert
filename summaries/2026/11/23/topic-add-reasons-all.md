---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Proposed one-line `__all__` in `reasons/__init__.py:1` that would curate wildcard re-exports without editing 33 shim files.

# Proposed one-line `__all__` in `reasons/__init__.py` — What It Would (and Wouldn't) Do

## What the one-liner would be

Given the current contents of `reasons/__init__.py` — a docstring, then `from dataclasses import dataclass, field` on line 3, then three dataclated classes (`Justification`, `Node`, `Nogood`) — the most plausible "one-line `__all__`" is:

```python
__all__ = ["Justification", "Node", "Nogood"]
```

The intent: stop `from reasons import *` from re-exporting the imported helper names `dataclass` and `field` (line 3) alongside the three dataclasses.

## What it actually curates

`__all__` only affects wildcard imports **from the module that declares it**. So `__all__` in `reasons/__init__.py` curates exactly one wildcard consumer in the repo:

- `reasons_lib/__init__.py:2` — `from reasons import *`

That shim currently leaks `dataclass` and `field` into the `reasons_lib` top-level namespace. Adding the `__all__` cleans that up. That's it.

## What it does **not** curate

The other 32 shim files under `reasons_lib/` do **not** import from `reasons`; they import from specific submodules. For example:

- `reasons_lib/api.py:2` — `from reasons.api import *`
- (analogously: `reasons_lib/network.py`, `reasons_lib/storage.py`, `reasons_lib/cli.py`, …)

Each of those wildcard imports is governed by `__all__` in `reasons/api.py`, `reasons/network.py`, `reasons/storage.py`, etc. `reasons/__init__.py` is a *different module* and its `__all__` is irrelevant to `from reasons.api import *`.

This is the crux: **a single `__all__` in `reasons/__init__.py` cannot curate the 32 module-level shims.** Each corresponding `reasons/X.py` needs its own `__all__`, or the shim pattern itself has to change. `existing_all_declarations` confirms no module in the codebase currently declares `__all__` — so every shim is currently re-exporting "everything not starting with `_`" from its source module.

## Why this matters in practice

- **Blast radius is one import.** Only `reasons_lib/__init__.py` benefits. The other 32 shims keep exactly their current wildcard surface.
- **The shims appear to be external-only.** `shim_import_patterns` (searching for `from reasons_lib | import reasons_lib`) returns **0 matches** across the codebase. Nothing in `reasons/`, tests, or scripts imports `reasons_lib`. So the "curate wildcard re-exports" motivation is entirely about downstream users who install `ftl-reasons` and still `from reasons_lib import *` per the docstring in `reasons_lib/__init__.py:1`.
- **The pyproject ships both packages.** `pyproject.toml:33` lists `packages = ["reasons", "reasons_lib"]`, so both are installed and both are part of the public API surface even though only `reasons` has internal consumers.

## What's missing from the observations

To verify the proposal end-to-end I'd need to see:

1. **The actual proposed line.** It wasn't quoted; the JSON only shows the pre-change file. Whether it's a literal list, a comprehension like `[n for n in dir() if not n.startswith("_")]`, or a lazy `__getattr__` + `__all__` combination changes the analysis.
2. **The other 32 shim files verbatim.** Only `reasons_lib/__init__.py`, `reasons_lib/api.py`, and `reasons_lib/check_stale...` (via the proposed-entries manifest) are sampled. If some don't follow the `from reasons.X import *` pattern (e.g. they import a specific object), a central `__all__` would matter differently.
3. **Any `__getattr__` / lazy-loader in `reasons/__init__.py`.** Not visible in the 59-line dump. Without one, `reasons/__init__.py` does not surface submodule names, so no amount of `__all__` there can "curate" `from reasons.api import *`.
4. **Line-1 concern.** `reasons/__init__.py:0` is the module docstring. If the proposal literally puts `__all__` on the *first physical line*, it will demote the docstring (it's no longer the first statement) and break `help(reasons)`. If "line 1" means 0-indexed line 1 (the blank line after the docstring), it's safe. This distinction matters.

## Recommended reading of the proposal

Treat it as a **partial, top-level safeguard**, not a full curation of the 33 shims. If the intent is truly to curate all shims with one change, the missing piece is either (a) per-module `__all__` in each `reasons/X.py`, or (b) a rewrite of the shims from `from reasons.X import *` to something like `from reasons import *` (which the single `__all__` would then cover) — but that would require editing the shims, contradicting the premise.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The only shim actually affected by the proposed `__all__`; check what names currently leak through `from reasons import *`.
- [file] `reasons/api.py` — The largest shim source (5886 lines). Adding `__all__` here — not in `reasons/__init__.py` — is what would curate `reasons_lib/api.py`'s wildcard surface.
- [general] `shim-dead-code-audit` — `shim_import_patterns` shows zero in-repo consumers of `reasons_lib`; worth confirming whether the 33 shims are truly external-compat-only before investing in curation.
- [file] `pyproject.toml` — `[tool.setuptools].packages` at line 33 ships both `reasons` and `reasons_lib`, so shim behavior is user-facing even though it's unused internally.
- [general] `pep-562-lazy-submodule-loading` — The only way a single `__all__` in `reasons/__init__.py` could even *nominally* enumerate submodules is via a `__getattr__` lazy loader; the current file has none.

## Beliefs

- `no-module-declares-all` — `existing_all_declarations` reports 0 `__all__` matches in the codebase, so every shim currently relies on Python's default "export everything not starting with `_`" behavior.
- `reasons-init-defines-three-dataclasses` — `reasons/__init__.py` defines exactly `Justification`, `Node`, and `Nogood` between lines 6 and 59.
- `reasons-init-leaks-imported-helpers` — `reasons/__init__.py:3` imports `dataclass` and `field` without underscore-prefixing or `__all__`, so `from reasons import *` currently leaks both into `reasons_lib/__init__.py`'s namespace.
- `top-level-shim-is-only-wildcard-consumer-of-reasons-init` — `reasons_lib/__init__.py:2` is the sole `from reasons import *` in the repo; the other ~32 shims use `from reasons.X import *` and are governed by `reasons/X.py`, not `reasons/__init__.py`.
- `shims-have-no-internal-consumers` — `shim_import_patterns` finds 0 in-repo `import reasons_lib` / `from reasons_lib` uses, so any change to shim wildcard surface affects only external callers.
- `pyproject-ships-both-packages` — `pyproject.toml:33` lists `packages = ["reasons", "reasons_lib"]`, meaning shim namespace leakage is part of the installable public API.
