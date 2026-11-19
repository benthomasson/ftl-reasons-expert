---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/ask.py

# `reasons_lib/ask.py` — Backwards-Compatibility Shim

## Purpose

This file is a **thin re-export shim**. It contains no logic of its own. Its sole job is to make the historical import path `reasons_lib.ask` continue to work after the code was canonicalized under the `reasons` package. The docstring is explicit about the intended migration target:

```python
"""Backwards-compatibility shim — use ``from reasons.ask`` instead."""
from reasons.ask import *  # noqa: F401,F403
```

So the responsibility here is **namespace aliasing**, not behavior. Any caller that still does `from reasons_lib.ask import ask` (or similar) gets the exact same symbols as calling `reasons.ask` directly, because Python's star-import copies each public name from `reasons.ask`'s module dict into `reasons_lib.ask`'s module dict at import time.

## Key Components

There are only two meaningful syntactic elements:

1. **Module docstring** — serves as the deprecation notice. Tools that print `module.__doc__` (help(), pydoc, IDE hover) will surface this to anyone still importing the old path.
2. **`from reasons.ask import *`** — the actual shim. It imports every name listed in `reasons.ask.__all__` (or, absent `__all__`, every non-underscore top-level name) into this module's namespace.

The `# noqa: F401,F403` comment suppresses two flake8 diagnostics:
- **F401** — "imported but unused" (the names are technically unused *within this module*, though they are re-exported).
- **F403** — "star imports are discouraged" (they hide provenance and can shadow names).

There are **no classes, functions, or constants defined here** — every public symbol you can reach through `reasons_lib.ask` is defined in `reasons/ask.py`.

## Patterns

- **Facade / Alias shim**: the canonical implementation lives in `reasons/ask.py`; `reasons_lib/ask.py` is a façade whose contract is "behave identically to `reasons.ask`."
- **Wildcard re-export with `# noqa`**: the standard idiom for maintaining a duplicate import path without duplicating code or triggering linters. The `F401` + `F403` pair is required precisely because a shim intentionally imports things it does not reference.
- **Paired-package layout**: the repo ships both `reasons/` and `reasons_lib/` side-by-side with the same module names. In this shim direction, `reasons.ask` is authoritative and `reasons_lib.ask` is the legacy surface being kept alive for `reasons/api.py`, `reasons/cli.py`, and three test modules.

## Dependencies

**Depends on (imports):**
- `reasons.ask` — the real implementation module. This shim cannot import without it; if `reasons.ask` is missing or has an import-time error, `reasons_lib.ask` fails identically.

**Depended on by (imported by):**
- `reasons/api.py` — the public API likely re-exports `ask` upward.
- `reasons/cli.py` — the CLI command that invokes `ask`.
- `tests/test_ask.py`, `tests/test_ask_mcp.py`, `tests/test_search_sources.py` — the test suite exercises `ask` through this legacy path (or through an alias that routes here).

Because the shim depends **on** `reasons` while being at the same package-name level, the effective dependency direction is `reasons_lib.* → reasons.*`, never the reverse. Reversing it (making `reasons.ask` import from `reasons_lib.ask`) would create a circular import.

## Flow

Execution is trivial and purely structural:

1. Python imports `reasons_lib.ask`, evaluating the module body top-to-bottom.
2. Line 2 triggers a normal import of `reasons.ask`. Python imports (or reuses) that module, executing `reasons/ask.py` fully.
3. `import *` copies each public name from `reasons.ask`'s module `__dict__` into `reasons_lib.ask`'s module `__dict__`.
4. The module object is returned to the importer. From that point on, `reasons_lib.ask.ask(...)` and `reasons.ask.ask(...)` are the **same function object** (identity holds: `reasons_lib.ask.ask is reasons.ask.ask`).

There are no data transformations, no branching, and no runtime state of its own.

## Invariants

- **Behavioral identity**: every public callable reachable via `reasons_lib.ask` must be the *same object* as in `reasons.ask`. Tests depending on this path are really testing `reasons.ask`.
- **`__all__` is inherited, not redeclared**: if `reasons.ask` defines `__all__`, the star-import honors it; if not, the effective surface is "all non-underscore names." This shim does **not** curate its own `__all__`, so the public API of `reasons_lib.ask` is coupled 1:1 to the current contents of `reasons.ask`.
- **No local state**: you cannot add `reasons_lib.ask`-specific symbols here without breaking the "identical surface" contract — any local definition would not exist on `reasons.ask`.
- **Import order**: `reasons.ask` must be importable from the interpreter's `sys.path` at the time `reasons_lib.ask` is imported. Since both packages coexist in the repo root, this is satisfied as long as the repo root is on the path (which it is for tests via `conftest.py` and for the CLI via the installed package).
- **Wildcard semantics**: names starting with `_` in `reasons.ask` are *not* re-exported unless included in its `__all__`. This is the standard Python behavior and is relevant if any downstream code expected to reach a private helper through the legacy path.

## Error Handling

- The shim **adds no error handling**. Any exception raised while importing `reasons.ask` (e.g., a missing optional dependency like an LLM client, an MCP transport, or a network library) propagates unchanged, but with a traceback that now includes `reasons_lib/ask.py` in the chain. In practice this is a net win for debugging — the failure points at `reasons.ask` even when you imported the legacy path.
- Any `AttributeError`, `TypeError`, or raise inside a function accessed via this module is raised by the **original** function in `reasons.ask`; there is no wrapping, catching, or translation.
- Because the shim uses `import *`, a `SyntaxError` or top-level runtime error in `reasons/ask.py` will surface as an `ImportError` / traceback on the `reasons_lib.ask` import — helpful for detecting when the canonical module breaks during refactors.

---

## Topics to Explore

- [file] `reasons/ask.py` — The canonical implementation; everything this shim exposes lives here. Read this to understand the actual `ask` contract (`__all__`, function signatures, exception behavior).
- [function] `reasons/ask.py:ask` — The primary entry point re-exported by this shim; confirms that this module is a pure alias and not a custom behavior.
- [file] `reasons/api.py` — Downstream consumer listed in the imports; shows how `ask` is wired into the higher-level public API.
- [file] `reasons_lib/api.py` — Its sibling shim; comparing them confirms whether the whole `reasons_lib` package is a legacy alias to `reasons`, or only selected modules.
- [general] `package-alias-migration` — The mechanism (and its limits) by which `reasons_lib.*` mirrors `reasons.*`; useful before doing any rename/refactor to know which paths to keep alive.

## Beliefs

- `reasons-lib-ask-is-alias-of-reasons-ask` — `reasons_lib.ask` contains no implementation; every public name it exposes is a re-export of the corresponding name in `reasons.ask`.
- `reasons-lib-ask-depends-on-reasons-ask` — `reasons/ask.py` must be importable for `reasons_lib/ask.py` to load; the dependency direction is `reasons_lib → reasons`, not the reverse.
- `reasons-lib-ask-uses-star-import` — The shim relies on `from reasons.ask import *` with `# noqa: F401,F403`, so its public surface is determined by `reasons.ask.__all__` (or its non-underscore names).
- `reasons-lib-ask-preserves-object-identity` — Calling `reasons_lib.ask.<fn>` executes the exact same function object as `reasons.ask.<fn>`; identity (`is`) holds between the two modules' symbols.
- `reasons-lib-ask-adds-no-error-handling` — All exceptions raised inside `reasons.ask` (including at import time) propagate unchanged through `reasons_lib.ask`.
