---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/compact.py

## Overview

`reasons_lib/compact.py` is a **one-line backwards-compatibility shim** that re-exports everything from `reasons.compact`. It exists so that legacy imports like `from reasons_lib.compact import compact_thoughts` (or `import reasons_lib.compact`) continue to work while the canonical implementation lives in the `reasons/` package.

```python
"""Backwards-compatibility shim — use ``from reasons.compact`` instead."""
from reasons.compact import *  # noqa: F401,F403
```

## 1. Purpose

The repository has two parallel package trees:

- `reasons/` — the canonical, actively maintained package.
- `reasons_lib/` — a legacy alias tree preserved for downstream consumers.

This file's **only responsibility** is to forward the public API of `reasons.compact` to anyone who imports `reasons_lib.compact`. It owns no logic of its own — no classes, no functions, no constants, no state. All behavior is delegated via wildcard import.

The module docstring explicitly tells maintainers: *"Backwards-compatibility shim — use `from reasons.compact` instead."* That is the contract — use the new path in new code; this path is for old code.

## 2. Key Components

- **Module docstring** — declares intent and provides the migration hint.
- **`from reasons.compact import *`** — the entire implementation. Everything that `reasons/compact.py` exposes via `__all__` (or, if `__all__` is absent, every public name not starting with `_`) becomes a name on `reasons_lib.compact`.
- **`# noqa: F401,F403`** — silences flake8 for wildcard re-export (`F401` = imported-but-unused, `F403` = wildcard import). This is standard idiom for shim files.

There are no classes, functions, or constants defined here. The "components" are inherited entirely from the target module.

## 3. Patterns

- **Shim / Facade / Alias module pattern.** The canonical implementation lives in one place (`reasons/compact.py`); the legacy module is a thin proxy. This is the classic "module relocation with an `import *` redirect" pattern used when renaming packages without breaking downstream imports.
- **Wildcard re-export.** Rather than manually re-listing every public symbol (`from reasons.compact import compact_thoughts, CompactResult, ...`), the wildcard form auto-propagates new public names to the legacy path. Trade-off: `__all__` in `reasons/compact.py` becomes the de facto contract (new public symbols added there automatically become available here).
- **Dual-tree convention.** Every module in `reasons_lib/` mirrors a module in `reasons/` this way (see the parallel directory listings in the repo tree). It is a project-wide convention, not a one-off.

## 4. Dependencies

**Depends on:**
- `reasons.compact` — the sole import. Whatever that module exports is what this file exposes.

**Depending on it (`imported_by`):**
- `reasons/api.py` — the top-level API layer still reaches for `reasons_lib.compact` (or its symbols).
- `tests/test_compact.py` — exercises the compact entry points through this path.
- `tests/test_summarize.py` — also exercises the legacy path.

Note the asymmetry: `reasons/api.py` (canonical tree) importing from `reasons_lib` (legacy tree) is unusual but works because `reasons_lib.compact` simply re-exports `reasons.compact`. A follow-up cleanup would likely move `reasons/api.py` onto `reasons.compact` directly.

## 5. Flow

Execution is trivially short:

1. Python imports `reasons_lib.compact`.
2. The `from reasons.compact import *` line executes, causing a full import of `reasons.compact`.
3. Any public symbol from `reasons.compact` is bound into the `reasons_lib.compact` namespace.
4. Callers see the exact same objects (functions/classes — not copies), so identity checks and isinstance work interchangeably with `reasons.compact`.

There is no control flow, branching, or transformation — this is a pure import-time binding.

## 6. Invariants

- **Single source of truth:** the real implementation must live in `reasons/compact.py`. Any behavior change must be made there, not here.
- **No `__all__` restriction in the shim:** because `reasons.compact` may or may not define `__all__`, the set of exported names is determined solely by that module's public surface. If `reasons.compact` grows a new public function, it automatically appears through this shim.
- **Object identity preserved:** callers importing `compact_thoughts` from either `reasons.compact` or `reasons_lib.compact` receive the same function object, so `isinstance` checks and monkeypatching against one affect the other.
- **Docstring is the migration contract:** the shim is intended as deprecated-but-supported surface.

## 7. Error Handling

- **No error handling of its own.** If `reasons.compact` fails to import (missing dependency, syntax error, ImportError inside it), that exception propagates unwrapped through this file. The traceback's first frame will point to `reasons/compact.py`, not here.
- **`# noqa` is not error handling** — it suppresses linter warnings only.
- **Wildcard import can raise `NameError`-like surprises** only in the case where `reasons.compact` defines `__all__` listing a nonexistent name (`AttributeError` at import time); that too propagates unchanged.

## Topics to Explore

- [file] `reasons/compact.py` — The canonical implementation this shim delegates to; examine its `__all__` (or lack thereof) to understand exactly what is re-exported here.
- [file] `reasons/api.py` — Check how and why the canonical API layer imports compact from the legacy `reasons_lib` path; likely a candidate for cleanup.
- [file] `tests/test_compact.py` — Shows the expected call sites of `compact_thoughts` / related symbols and confirms which public names must remain reachable.
- [general] `reasons-vs-reasons_lib-migration` — Repo-wide pattern of dual module trees; understand whether this is permanent or scheduled for removal.
- [function] `reasons/compact.py:compact_thoughts` — The primary entry point (assuming that naming) that the shim ultimately exposes.

## Beliefs

- `reasons_lib-compact-is-a-shim` — `reasons_lib/compact.py` contains no implementation and re-exports everything from `reasons.compact` via `import *`.
- `reasons_lib-compact-requires-reasons-compact` — Importing the shim necessarily triggers a full import of `reasons.compact`, so failures there propagate directly to the legacy path.
- `reasons_lib-compact-preserves-object-identity` — Symbols imported from the shim are the same objects as those in `reasons.compact`, not copies or wrappers.
- `reasons-api-imports-reasons_lib-compact` — `reasons/api.py` still depends on the legacy `reasons_lib` path for compact functionality.
- `reasons_lib-compact-produces-no-own-exports` — The shim exposes no names that are not also public in `reasons.compact`; its public surface is fully determined by the target module's `__all__` or public naming.
