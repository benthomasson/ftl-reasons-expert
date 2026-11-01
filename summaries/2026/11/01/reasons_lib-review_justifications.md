---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/review_justifications.py

# `reasons_lib/review_justifications.py`

## 1. Purpose

This file is a **backwards-compatibility shim**. Its entire job is to make old import paths keep working after the project's canonical package moved from `reasons_lib/` to `reasons/`.

The docstring states the contract explicitly:

> *"Backwards-compatibility shim — use ``from reasons.review_justifications`` instead."*

If you're maintaining this codebase, treat `reasons/review_justifications.py` as the source of truth. This file adds zero logic — it exists solely so that any external consumer (or legacy internal code) still doing `from reasons_lib.review_justifications import review_justifications` doesn't break.

## 2. Key Components

There is exactly **one statement** of substance:

```python
from reasons.review_justifications import *  # noqa: F401,F403
```

| Detail | Meaning |
|---|---|
| `import *` | Re-exports every public name (those not starting with `_`) from the canonical module. If `reasons.review_justifications` defines `__all__`, only those names are forwarded; otherwise all top-level names except private ones. |
| `# noqa: F401` | Suppresses the linter warning "imported but unused." The import *is* used — by downstream consumers of this module — but the linter can't see that. |
| `# noqa: F403` | Suppresses the linter warning "star import makes it unclear what's exported." Intentional here: the shim's whole purpose is to be an opaque pass-through. |

No classes, functions, constants, or logic are defined locally.

## 3. Patterns

- **Shim / façade re-export.** A well-known idiom in Python package restructuring: keep the old dotted path alive by `import *`-ing from the new location. The `reasons_lib/` directory mirrors `reasons/` file-for-file, confirming this is a project-wide rename, not a one-off.
- **Dual-tree layout.** Every module that exists under `reasons/` has a corresponding one-liner under `reasons_lib/`. This is a migration strategy: `reasons/` is the live code; `reasons_lib/` is a frozen compatibility layer that can be deleted in the next major version.
- **No `__all__` override.** The shim inherits whatever `reasons.review_justifications` exports. If that module defines `__all__`, the shim respects it automatically.

## 4. Dependencies

**Upstream (this file imports):**

- `reasons.review_justifications` — the canonical implementation. This is the *only* dependency.

**Downstream (imports this file):**

- `reasons/api.py` — the public API surface. It imports from `reasons_lib.review_justifications`, which means the API layer is still routing through the old namespace. This is worth flagging: the shim is load-bearing for the API.
- `tests/test_review_justifications.py` — the test suite imports via the shim path, so tests exercise the re-export chain rather than the canonical module directly.

## 5. Flow

Execution is trivial and linear:

1. Python's import system resolves `reasons_lib.review_justifications`.
2. The module body executes the single `from reasons.review_justifications import *` statement.
3. Python loads (or finds in `sys.modules`) `reasons.review_justifications` and copies its public names into `reasons_lib.review_justifications`'s namespace.
4. Any downstream `from reasons_lib.review_justifications import X` now resolves `X` against that copied namespace.

No function calls, no conditional logic, no side effects beyond the import itself.

## 6. Invariants

- **Single source of truth.** All real logic must live in `reasons/review_justifications.py`. This shim must never define, monkey-patch, or conditionally alter any name.
- **Name parity.** Every public symbol in `reasons.review_justifications` must be visible through `reasons_lib.review_justifications`. Adding a new public function to the canonical module automatically propagates it here (no manual update needed).
- **No circular imports.** `reasons.review_justifications` must not import anything from `reasons_lib`, or you get a circular import at load time.

## 7. Error Handling

There is none here. If `reasons.review_justifications` fails to import (missing dependency, syntax error, circular import), the exception propagates directly to whoever triggered `import reasons_lib.review_justifications`. This shim neither catches, wraps, or transforms any exception.

The `# noqa` comments are purely a *linter* concern (flake8/pycodestyle), not a runtime concern.

---

## Topics to Explore

- [file] `reasons/review_justifications.py` — The canonical implementation this shim delegates to; all real logic, data structures, and contracts live here.
- [file] `reasons/api.py` — Imports through this shim; understanding *which* symbols it pulls from `review_justifications` tells you what the public API actually exposes.
- [file] `tests/test_review_justifications.py` — Exercises the shim path; check whether tests would still pass if the shim were removed (they'd need their import lines updated).
- [general] `reasons-vs-reasons-lib-migration` — The project-wide rename strategy: every file in `reasons_lib/` is a one-line re-export. Knowing when `reasons_lib/` can be deleted is a key maintenance question.
- [file] `reasons/review.py` — The sibling module; understanding how `review` and `review_justifications` interact reveals the broader belief-justification review workflow.

---

## Beliefs

- `reasons_lib-replaces-reasons-as-canonical-package` — `reasons_lib/` is a backwards-compatibility layer; `reasons/` is the authoritative source of all logic.
- `api-requires-reasons_lib-shim` — `reasons/api.py` imports review-justification symbols via `reasons_lib.review_justifications`, so deleting the shim without updating the API will break the public entry point.
- `shim-has-zero-runtime-logic` — The module body consists of a single star-import and two `# noqa` comments; it defines no functions, classes, constants, or side effects of its own.
- `shim-inherits-__all__-from-canonical` — If `reasons/review_justifications.py` defines `__all__`, the re-exported name set in `reasons_lib` is exactly that list; no names are added or filtered by the shim.
- `reasons_lib-deletion-is-safe-when-api-updated` — The shim can be removed (along with the rest of `reasons_lib/`) in one atomic change once `reasons/api.py` and `tests/test_review_justifications.py` are pointed at `reasons.review_justifications` directly.
