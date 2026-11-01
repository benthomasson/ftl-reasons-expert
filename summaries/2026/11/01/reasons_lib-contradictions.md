---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/contradictions.py

# `reasons_lib/contradictions.py`

## Purpose

This file is a **backwards-compatibility shim**. It exists solely so that older code importing from the `reasons_lib` namespace (e.g., `from reasons_lib.contradictions import find_contradictions`) continues to work without modification. The actual implementation lives in `reasons/contradictions.py`. The `reasons_lib` package is a legacy namespace that mirrors the canonical `reasons` package one-to-one, and this file is that mirror for the contradictions module.

## Key Components

There are **no** original classes, functions, or constants defined here. The file has exactly one statement of substance:

```python
from reasons.contradictions import *  # noqa: F401,F403
```

This star-import pulls every public name (anything not starting with `_`) from `reasons.contradictions` into the `reasons_lib.contradictions` module namespace. After this import executes, `reasons_lib.contradictions.X` is the exact same object as `reasons.contradictions.X` for every public `X`.

The `# noqa: F401,F403` directive tells linters (flake8, ruff) to suppress:
- **F401** — "imported but unused" (the `*` names aren't explicitly used in this file)
- **F403** — "star imports are not allowed"

Both are expected and intentional for a re-export shim.

## Patterns

- **Re-export shim / compatibility alias.** This is a well-known Python pattern for package renames or namespace migrations. The `reasons` package is the canonical home; `reasons_lib` is a frozen compatibility layer. Every file in `reasons_lib/` follows this same one-line pattern (you can see the parallel structure: `reasons_lib/api.py` ↔ `reasons/api.py`, `reasons_lib/cluster.py` ↔ `reasons/cluster.py`, etc.).
- **Star import for bulk re-export.** Rather than listing each symbol explicitly, `import *` ensures that if new public names are added to `reasons.contradictions` in the future, they automatically become available through `reasons_lib.contradictions` without touching this shim. This is a deliberate tradeoff: it avoids a maintenance tax on every API addition, at the cost of a less explicit interface.
- **Flat parallel package structure.** The repo maintains two top-level packages (`reasons/` and `reasons_lib/`) with identical file layouts. This suggests a migration in progress: the team is moving (or has moved) the canonical code from `reasons_lib` → `reasons`, and the old namespace is kept alive for downstream consumers.

## Dependencies

| Direction | Module | Relationship |
|-----------|--------|--------------|
| **This file imports** | `reasons.contradictions` | The sole and real source of all symbols |
| **Imported by** | `reasons/api.py` | Consumes contradiction-detection utilities via the `reasons_lib` path (likely a legacy import that hasn't been rewritten yet) |
| **Imported by** | `tests/test_contradictions.py` | Tests may import through either namespace |

Notably, `reasons/api.py` importing from `reasons_lib.contradictions` is a sign that the migration to the `reasons` namespace is not fully complete — at least one internal consumer still goes through the compat layer.

## Flow

There is no control flow to speak of. At import time:

1. Python executes `from reasons.contradictions import *`.
2. The import machinery resolves `reasons.contradictions`, executes that module (and all of its own imports), and collects its `__all__` (or all public names).
3. Each name is bound into the `reasons_lib.contradictions` module's `__dict__`.
4. Any subsequent `from reasons_lib.contradictions import <symbol>` or `reasons_lib.contradictions.<symbol>` access resolves to the same object in `reasons.contradictions`.

No function is called, no state is mutated. The file is a pure re-binding.

## Invariants

- **Identity guarantee:** For every public name `N` in `reasons.contradictions`, `reasons_lib.contradictions.N is reasons.contradictions.N` evaluates to `True`. There is no copying, wrapping, or sub-classing.
- **No private leakage:** Because `import *` only pulls names not starting with `_` (or those listed in `__all__`), internal helpers of `reasons.contradictions` stay hidden from the `reasons_lib` namespace.
- **Deletion risk:** If `reasons.contradictions` removes a public symbol, the `reasons_lib` shim silently loses it too. There is no compile-time or import-time check that the two namespaces stay in sync. The only guard is the `noqa` comment acknowledging the star import.

## Error Handling

This file defines no error handling. All error semantics are inherited from `reasons.contradictions`:

- **Import-time failure:** If `reasons.contradictions` itself raises (e.g., a missing dependency, a syntax error in that module), the `import *` statement will propagate that `ImportError`/`Exception` upward. The `reasons_lib.contradictions` module will simply fail to load.
- **Runtime failures:** Any function called through `reasons_lib.contradictions` behaves identically to calling it through `reasons.contradictions` — same exceptions, same return types, same side effects. The shim adds zero error-handling logic.
- **`ImportError` edge case:** If `reasons.contradictions` has an `__all__` that references a name not actually defined in the module, the star import will raise an `ImportError` at import time. This is the only scenario where the shim itself could produce a distinct error (though the root cause is still in `reasons.contradictions`).

## Topics to Explore

- [file] `reasons/contradictions.py` — The actual implementation this shim re-exports; understanding the contradiction-detection logic, its public API surface, and its internal data model is essential before touching the shim.
- [file] `reasons/api.py` — The primary consumer that still imports via `reasons_lib.contradictions`; understanding why it hasn't been migrated reveals the constraints on when the `reasons_lib` package can be retired.
- [general] `reasons-lib-migration` — The broader `reasons_lib` → `reasons` namespace migration: how many files are pure shims, whether any have diverged, and what the plan is for removing the compat layer.
- [file] `tests/test_contradictions.py` — The test suite that exercises this code path; shows the expected contract of the public API and which import style (canonical vs. shim) the tests use.
- [file] `reasons_lib/__init__.py` — Whether the `reasons_lib` package does any additional re-exporting or deprecation warnings at the package level, and whether it mirrors `reasons/__init__.py` exactly.

## Beliefs

- `reasons-lib-contradictions-is-a-pure-reexport` — `reasons_lib/contradictions.py` defines zero original symbols; every public name in that module is the same object as the corresponding name in `reasons/contradictions.py` (identity, not equality).
- `reasons-lib-mirrors-reasons-one-to-one` — For every `.py` file in `reasons/` (excluding `__init__.py` and `merkle.py`, `pg.py` which appear only in `reasons/`), there is a corresponding `reasons_lib/` file that star-exports from the `reasons` version.
- `reasons-api-still-depends-on-reasons-lib` — `reasons/api.py` imports contradiction utilities through the `reasons_lib` namespace rather than directly from `reasons`, meaning the compat layer cannot be deleted until that import is rewritten.
- `star-import-guarantees-automatic-forward-compat` — Adding a new public function to `reasons/contradictions.py` automatically makes it available via `reasons_lib/contradictions.py` without any change to the shim file.
- `no-merkle-or-pg-in-reasons-lib` — The files `merkle.py` and `pg.py` exist only under `reasons/` and have no `reasons_lib/` counterpart, meaning those modules were added after the `reasons_lib` namespace was frozen.
