---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/review_premises.py

# `reasons_lib/review_premises.py` — Compatibility Shim

## Purpose

This file is a **one-line backwards-compatibility shim**. It does *not* contain the actual review-of-premises logic; it simply re-exports the public surface of `reasons.review_premises` so that code written against the older `reasons_lib.*` import path continues to work after the package was renamed/refactored into `reasons.*`.

The sibling tree in the repo confirms the convention: nearly every module under `reasons/` has a mirror at `reasons_lib/` whose body is this same renaming shim. `review_premises` is one of those mirrors.

**Responsibility owned:** translating the legacy `reasons_lib.review_premises` namespace into the canonical `reasons.review_premises` namespace. All behavior, validation, and error semantics live in the target module — none in this file.

## Key Components

The file contains **no classes, functions, or constants of its own**. Its only component is:

```python
from reasons.review_premises import *  # noqa: F401,F403
```

- `# noqa: F401` suppresses the "imported but unused" lint warning (the whole point is to re-export, not to reference locally).
- `# noqa: F403` suppresses the "`import *` used; unable to detect undefined names" warning (star-import is the intended re-export mechanism here).

**Contract exposed by this module:** whatever `reasons.review_premises` exposes publicly.

- If `reasons.review_premises` defines `__all__`, then *exactly* those names are re-exported.
- If it does not define `__all__`, then every name in its module namespace that does not start with `_` is re-exported (including any transitively imported names, which is why a downstream `dir(reasons_lib.review_premises)` can be a superset of the "public API" you'd expect).

## Patterns

- **Facade / forwarding shim (module-level re-export).** This is the standard Python idiom for maintaining an old import path while the implementation moves. It is alias-by-reference, not subclassing or wrapping — the objects you import from `reasons_lib.review_premises` are *the exact same objects* (same class identity, same function `__module__`) as those in `reasons.review_premises`. `isinstance` and `is` comparisons work across both paths.
- **Consistent mirror layout.** The `reasons/` ↔ `reasons_lib/` duplication in the repo tree is deliberate; each legacy module is a minimal proxy. This keeps the maintenance surface tiny: change behavior only in `reasons/`, never in `reasons_lib/`.
- **Lint-suppression annotations** (`F401`, `F403`) — a standard marker pair used on star-import shims so CI linters don't flag the file as dead code.

## Dependencies

**Depends on:**
- `reasons.review_premises` — the real implementation (source of truth). At import time, Python resolves `reasons.review_premises`; if it raises on import, importing this shim raises identically.

**Depended on by (via `imported_by`):**
- `reasons/api.py` — the public API layer imports from this legacy path, meaning the shim is still on a live code path in production.
- `tests/test_repair_premises.py` — tests exercise premise-repair behavior through this path.
- `tests/test_review_premises.py` — primary test coverage for the underlying functionality, reached via this shim.

Because `reasons/api.py` depends on the shim, `reasons_lib/review_premises.py` cannot be deleted without first repointing `api.py` to `reasons.review_premises`.

## Flow

1. A caller executes `from reasons_lib.review_premises import review_something`.
2. Python's import machinery loads `reasons_lib/review_premises.py`.
3. That module executes its single statement, which triggers the import of `reasons.review_premises` (executing that module's top-level code once, cached in `sys.modules`).
4. The `import *` binds the public names of `reasons.review_premises` into the `reasons_lib.review_premises` module's namespace.
5. The caller receives the bound names — which are references to the *same* objects as in the canonical module.

There is no control flow, no data transformation, and no runtime state of this module's own. Ordering constraint: `reasons.review_premises` must be importable without error *before* this shim can be imported; any circular import involving `reasons.review_premises` and `reasons_lib.review_premises` would surface here as an `ImportError`.

## Invariants

- **Single source of truth.** No logic may be added to this file; any behavioral change belongs in `reasons/review_premises.py`. Duplicating logic here would create drift between the two packages.
- **Public API parity.** The set of names importable from `reasons_lib.review_premises` should equal the intended public set of `reasons.review_premises` (subject to `__all__` presence — see below).
- **Object identity preserved.** `reasons_lib.review_premises.Foo is reasons.review_premises.Foo` for any re-exported `Foo` (assuming `Foo` is a module-level binding, not e.g. a property).
- **Star-import is `__all__`-sensitive.** If `reasons.review_premises` gains an `__all__`, this shim automatically narrows to it; if `__all__` is added later but omits a name that callers previously used from the shim, those callers break. This is an implicit contract to check when adding `__all__` upstream.

## Error Handling

- The shim itself **raises no errors and swallows none**. It is purely declarative.
- Any error raised during import of `reasons.review_premises` (e.g., `ImportError`, a `SyntaxError` in the implementation, a failing top-level assertion) propagates unchanged through this module — the traceback will show `reasons/review_premises.py` as the origin, not this file.
- If a caller references a name not exported (because it starts with `_`, or was excluded from `__all__`), they get a normal `ImportError`/`AttributeError` at *their* import site, not here.
- The `# noqa` comments mean the file will not be flagged by linters for unused imports; this is intentional and should be preserved if the file is edited.

## Topics to Explore

- [file] `reasons/review_premises.py` — The canonical implementation this shim forwards to; study its functions and `__all__` to know the true public surface.
- [file] `reasons/api.py` — Imports through this legacy path; inspect those imports to see which shimmed names are on the live API surface and whether they should be repointed.
- [file] `reasons_lib/__init__.py` — Understand the package-level conventions (e.g., whether `reasons_lib` itself re-exports anything) that this shim follows.
- [general] `shim-migration-policy` — The project-level policy for retiring `reasons_lib.*` mirrors: are they deprecated with warnings, silently supported, or slated for deletion?
- [function] `tests/test_review_premises.py` — Concrete examples of the re-exported API being exercised end-to-end through this shim.

## Beliefs

- `shim-forwards-to-canonical-module` — `reasons_lib/review_premises.py` contains exactly one statement, `from reasons.review_premises import *`, re-exporting the canonical module's public names.
- `no-logic-in-shim` — The shim defines no functions, classes, or constants of its own; all behavior lives in `reasons.review_premises`.
- `identity-preserved-across-paths` — Objects imported via `reasons_lib.review_premises` are the same objects as those imported via `reasons.review_premises`.
- `api-depends-on-legacy-path` — `reasons/api.py` imports through this leg
