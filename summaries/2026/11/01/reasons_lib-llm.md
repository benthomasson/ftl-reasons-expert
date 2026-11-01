---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/llm.py

# `reasons_lib/llm.py`

## Purpose

This file is a **backwards-compatibility shim** — a one-line re-export that exists solely so that code written against the old `reasons_lib` package namespace continues to work. The canonical implementation lives in `reasons/llm.py`. The docstring on line 1 makes the intent explicit:

> `"""Backwards-compatibility shim — use ``from reasons.llm`` instead."""`

In other words, `reasons_lib/` is the **legacy** package layout, and `reasons/` is the **current** one. This file is the glue that lets old import paths (`from reasons_lib.llm import ...`) resolve to the same objects as the new path (`from reasons.llm import ...`).

## Key Components

There are no classes, functions, or constants defined here. The entire functional content is a single statement:

```python
from reasons.llm import *  # noqa: F401,F403
```

| Token | Meaning |
|---|---|
| `from reasons.llm import *` | Pulls every public name (no leading-underscore) defined in `reasons/llm.py` into the `reasons_lib.llm` module's namespace. |
| `# noqa: F401` | Suppresses the flake8 *"imported but unused"* warning — the star import *is* the point of the file. |
| `# noqa: F403` | Suppresses the flake8 *"star imports make it unclear what names are defined"* warning. |

Any name a caller would grab from `reasons_lib.llm` (e.g. an `LLMClient` class, a `generate()` helper, a `MODEL_TIMEOUT` constant — whatever `reasons/llm.py` actually defines) is resolved at import time from `reasons.llm`.

## Patterns

- **Facade / re-export shim.** A well-known Python idiom for package renames or restructures. You keep the old module path alive with a star import so existing `import` statements don't break, while new code migrates to the canonical path.
- **Dual-package layout.** The repo ships two parallel trees — `reasons/` (source of truth) and `reasons_lib/` (compatibility layer). Every file in `reasons_lib/` is either a real implementation *or* a shim like this one. The pattern is consistent across the tree (e.g. `reasons_lib/cluster.py`, `reasons_lib/contradictions.py`, etc. mirror their `reasons/` counterparts).
- **`__all__`-independent star import.** Because `reasons.llm` doesn't (or may not) define `__all__`, the star import picks up every top-level name that doesn't start with `_`. If `reasons/llm.py` ever adds a new public symbol, it automatically becomes available through `reasons_lib.llm` without touching this file.

## Dependencies

### Upstream (what this file imports)

- **`reasons.llm`** — the single hard dependency. If that module is missing or fails to import, `import reasons_lib.llm` will raise an `ImportError` before any of the 16 downstream consumers can load.

### Downstream (what imports this file)

The "Imported By" list includes 15 modules under `reasons/` plus `tests/test_llm.py`:

`reasons/api.py`, `reasons/ask.py`, `reasons/build_wiki.py`, `reasons/cli.py`, `reasons/contradictions.py`, `reasons/pg.py`, `reasons/propose_update.py`, `reasons/repair.py`, `reasons/repair_premises.py`, `reasons/report_belief.py`, `reasons/report_gated.py`, `reasons/review.py`, `reasons/review_justifications.py`, `reasons/review_premises.py`, `reasons/verify.py`, `tests/test_llm.py`

> **Note:** Given the docstring says *"use `from reasons.llm` instead,"* it's worth verifying whether those `reasons/` modules actually import from `reasons_lib.llm` or from `reasons.llm` directly. If they import from `reasons_lib.llm`, the shim is load-bearing for the entire application, not just for external consumers.

## Flow

Execution is trivial:

1. Python's import machinery encounters `import reasons_lib.llm`.
2. It loads this 3-line module.
3. The `from reasons.llm import *` statement triggers import of `reasons/llm.py` (if not already cached in `sys.modules`), executes it, then copies every public name into `reasons_lib.llm`'s namespace.
4. The importing module now holds references to those objects. No further indirection occurs at call time — the function objects, classes, and constants are the *same* objects defined in `reasons/llm.py`.

There is no runtime proxying, no `__getattr__` trick, no lazy loading. It's a one-time name copy at import time.

## Invariants

- **Single source of truth.** All LLM-related logic (prompt construction, model selection, API calls, retries) lives in `reasons/llm.py`. This shim defines nothing. If you find yourself adding code here, you're in the wrong file.
- **Name parity.** Every public name in `reasons/llm.py` is guaranteed to be reachable via `reasons_lib.llm`. Conversely, names that are private (`_foo`) in `reasons.llm` will *not* appear in `reasons_lib.llm`.
- **No circular import risk introduced.** Because this file only imports *from* `reasons`, it cannot create a cycle as long as `reasons/llm.py` does not import from `reasons_lib`.

## Error Handling

There is no explicit error handling in this file. The only failure mode is:

- **`ImportError` / `ModuleNotFoundError`** at import time if `reasons.llm` cannot be resolved (e.g. the `reasons` package isn't installed, or `reasons/llm.py` has a syntax error). This propagates up to whichever module first does `import reasons_lib.llm`.

No exceptions are caught, no fallbacks exist, and no logging occurs. The shim is transparent — it neither masks nor adds to the error surface of the real implementation.

---

## Topics to Explore

- [file] `reasons/llm.py` — The actual implementation this shim forwards to; understanding its public API (classes, functions, constants) is essential before touching either the shim or its consumers.
- [file] `reasons_lib/__init__.py` — Check whether the package-level `__init__` also re-exports submodules or sets up any package metadata that interacts with this shim.
- [repo] `reasons/` vs `reasons_lib/` — Determine which files in `reasons_lib/` are real implementations vs. shims, and whether there's a migration plan to remove the `reasons_lib/` tree entirely.
- [file] `tests/test_llm.py` — See which import path the tests use (`reasons.llm` or `reasons_lib.llm`), which reveals the intended "primary" namespace for the test suite.
- [general] `pyproject.toml` packaging — Check how both `reasons` and `reasons_lib` are declared in `[project]` / `[tool.setuptools]` to understand which is the published package and whether both are shipped.

## Beliefs

- `reasons_lib-llm-is-a-reasons-llm-alias` — `reasons_lib/llm.py` defines zero symbols of its own; every public name in its namespace is the identical object imported from `reasons/llm.py`.
- `reasons_lib-llm-star-import-respects-underscores` — Because the shim uses `import *` without an explicit `__all__` override, private names (prefixed with `_`) in `reasons/llm.py` are *not* re-exported through `reasons_lib.llm`.
- `shim-breaks-if-reasons-llm-missing` — Importing `reasons_lib.llm` will raise `ModuleNotFoundError` if the `reasons` package is not installed or `reasons/llm.py` does not exist; there is no fallback or stub.
- `no-runtime-indirection` — After import time, calling a function "from `reasons_lib.llm`" executes the exact same code object defined in `reasons/llm.py`; there is no wrapper, proxy, or `__getattr__` layer at call time.
