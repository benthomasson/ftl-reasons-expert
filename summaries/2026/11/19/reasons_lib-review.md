---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/review.py

# `reasons_lib/review.py` — Backwards-Compatibility Shim

```python
"""Backwards-compatibility shim — use ``from reasons.review`` instead."""
from reasons.review import *  # noqa: F401,F403
```

That is the entire file: a docstring and one wildcard import. Its job is purely **symbolic forwarding** — it does not implement, wrap, or reinterpret any behaviour.

---

## 1. Purpose

`reasons_lib/review.py` exists so that older code written against the `reasons_lib` package layout keeps working after the project settled on `reasons` as the canonical package name.

The repository contains **two parallel trees with identical filenames** (`reasons/` and `reasons_lib/`, both containing `api.py`, `ask.py`, `review.py`, …). The canonical implementations live in `reasons/`; `reasons_lib/review.py` is one of the modules where the old path is kept alive as an alias.

It owns **no responsibility** of its own:

- no validation, no state, no logging, no defaults
- no `__all__` — so its exported surface is whatever `reasons.review` exposes
- no versioning logic, no deprecation warning emission (notably, it does *not* warn on import)

The docstring on line 1 is the only in-code guidance: new code should `from reasons.review import …`.

---

## 2. Key Components

There are no functions, classes, or constants defined here. The "components" are the two module-level statements and the identity relationship they create.

| Line | Statement | Contract |
|---|---|---|
| 1 | `"""Backwards-compatibility shim — use ``from reasons.review`` instead."""` | Module docstring; the only documentation of intent. |
| 2 | `from reasons.review import *` | Binds every public name from `reasons.review` into `reasons_lib.review.__dict__` at import time. |

### Resulting contract of `reasons_lib.review`

- **Reading attributes**: `reasons_lib.review.<name>` resolves to whatever `reasons.review.<name>` was bound to *at the moment `reasons_lib.review` was first imported*.
- **Calling functions**: for any public function `f` in `reasons.review`, `reasons_lib.review.f is reasons.review.f` — it is the *same function object*, not a wrapper. Overrides, signatures, docstrings, and `__module__` are unchanged (note `f.__module__` will say `"reasons.review"`, not `"reasons_lib.review"` — relevant when debugging tracebacks).
- **Classes**: `reasons_lib.review.SomeClass is reasons.review.SomeClass`, so `isinstance` checks are interchangeable between the two import paths. This is the main reason the shim is safe for type-based dispatch.

### What is *not* re-exported

- Any name starting with `_` (private helpers, module-private constants, `_cache`, etc.).
- Module dunders such as `__name__`, `__file__`, `__doc__` — `reasons_lib.review.__name__` is `"reasons_lib.review"`, and `__doc__` is the shim docstring above, **not** the docstring of `reasons.review`.
- Anything that `reasons.review` adds to itself *after* the shim has been imported (see Invariants).

---

## 3. Patterns

**Wildcard re-export shim / alias module.** A module whose entire body is `from X import *`. It is the cheapest form of "rename without a breaking change"; the alternative would be explicit per-symbol imports or a module-level `__getattr__` forwarding hook.

**Parallel package trees.** `reasons/` and `reasons_lib/` mirror each other filename-for-filename. Most `reasons_lib/*.py` files are presumably thin re-exports like this one, with `reasons_lib/__init__.py` gluing the legacy namespace together.

**Lint suppression as documentation of intent.** `# noqa: F401,F403` is doing real work here:
- `F401` = "imported but unused" (the wildcard must not be "used" by definition)
- `F403` = "`from … import *` used; unable to detect undefined names"

The `noqa` is the mechanical assertion that this file's *purpose* is to import something it never references.

**Canonical-vs-legacy naming.** The docstring documents the direction of migration ("use `from reasons.review`"), so the shim is intentionally write-once and never the place to add logic.

---

## 4. Dependencies

### Depends on (outgoing)

- `reasons.review` — the canonical implementation module. Importing this shim therefore also imports the `reasons` package (`reasons/__init__.py` runs first), and then `reasons/review.py` and *its* transitive imports (any storage/network modules that `review` needs).

There are no third-party dependencies at all — no `typing`, no `warnings`, nothing that could fail independently of `reasons.review`.

### Depended on by (incoming)

From the module graph supplied:

| Importer | Note |
|---|---|
| `reasons/api.py` | The **canonical** package imports the **legacy** shim. |
| `reasons/repair.py` | Same — canonical → legacy direction. |
| `reasons/review_justifications.py` | Same. |
| `reasons_lib/review_justifications.py` | Legacy → legacy, consistent. |
| `reasons_lib/review_premises.py` | Legacy → legacy, consistent. |
| `tests/test_review.py`, `tests/test_review_justifications.py`, `tests/test_review_premises.py`, `tests/test_repair_premises.py`, `tests/test_classify_defeaters.py` | The **test suite exercises the shim**, so the shim is load-bearing for CI — it is not dead code. |

**The important consequence**: the `reasons/` package (`api.py`, `repair.py`, `review_justifications.py`) importing `reasons_lib.review` means `reasons_lib` cannot be deleted without first editing the canonical package. It also creates a *cross-tree* import edge — the "new" tree depends on the "old" tree, which is the reverse of what the docstring suggests. And because `reasons_lib.review` imports back into `reasons.review`, you get a near-cycle:

```
reasons.api  →  reasons_lib.review  →  reasons.review  →  (reasons/__init__ already in progress)
```

Python tolerates this because `reasons.review` is a *submodule* import and `reasons/__init__.py` may be partially initialized; but any code in `reasons/review.py` that does `from reasons import <name defined later in __init__>` would break depending on which entry point is imported first. This ordering constraint is the single most fragile thing about this file.

---

## 5. Flow

Importing `reasons_lib.review` is a straight line, no branching:

1. **Module is located and created** with an empty `__dict__`; the docstring is assigned to `__doc__`.
2. **Line 2 executes** `from reasons.review import *`, which means:
   a. Import machinery imports `reasons` (running `reasons/__init__.py`).
   b. Imports `reasons.review` (running its module body, including all of *its* imports).
   c. If `reasons.review` defines `__all__`, the names in `__all__` are copied. Otherwise (the common case) every name in `reasons.review.__dict__` that does not start with `_` is copied.
   d. Each name is bound as a **new pointer to the same object** in `reasons_lib.review.__dict__`.
3. **No further code runs.** There are no `if __name__ == "__main__"` blocks, no side-effecting comprehension, no `warnings.warn`.

Downstream, at call time, there is no indirection cost: `reasons_lib.review.some_fn(...)` calls the very same function object as `reasons.review.some_fn(...)`. Dispatch is direct; there is no `__getattr__` hop.

The one flow subtlety: because the names are **copied** rather than forwarded, the shim is a *snapshot*. If `reasons.review` later rebinds `some_fn` (e.g. a monkeypatch, or a `set_storage()`-style module-level mutator that does `global CURRENT_STORAGE`), then:

- `reasons.review.some_fn` → new object
- `reasons_lib.review.some_fn` → stale, old object

But mutating a **mutable object** (e.g. appending to a module-level list or dict that both names point to) is visible through both paths, because both names refer to the same object.

---

## 6. Invariants

1. **No symbols originate here.** `reasons_lib.review`'s public API is a subset of `reasons.review`'s public API; it can never be a superset.
2. **Identity, not equality, for shared objects.** `reasons_lib.review.X is reasons.review.X` holds for every non-underscore name `X` present in `reasons.review` at shim-import time. Classes and exceptions defined there can be caught and `isinstance`-checked interchangeably.
3. **Underscore names are absent.** `reasons_lib.review._helper` raises `AttributeError` even if `reasons.review._helper` exists. Tests that reach for private helpers must import the canonical module.
4. **Rebinding after import is not propagated** (snapshot semantics, described above).
5. **The shim defines no `__all__`,** so `from reasons_lib.review import *` re-imports exactly the same non-underscore set — good for chained legacy imports, but it means linters will not catch a typo in a legacy `from reasons_lib.review import foo` until runtime.
6. **Import-order coupling.** `reasons_lib.review` must not be imported in a way that shadows a partially-initialized `reasons.review`; because three canonical modules (`reasons/api.py`, `reasons/repair.py`, `reasons/review_justifications.py`) route through the shim, the effective import graph contains a cycle that only works because `*`-copy happens *after* `reasons.review`'s body finishes. Refactoring `reasons/review.py` to lazily import something from `reasons/__init__.py` risks `ImportError: cannot import name … (most likely due to a circular import)`.

---

## 7. Error Handling

There is **no error handling at all** in this file — no `try`/`except`, no `ImportError` fallback, no `warnings.warn(DeprecationWarning)`. Failure modes are therefore Python's default, raised at import time:

| Situation | Result |
|---|---|
| `reasons/review.py` missing or broken | `ModuleNotFoundError` / the original exception from that module, surfaced at `import reasons_lib.review`. Traceback points into `reasons/review.py`, not the shim — mildly confusing but accurate. |
| `reasons.review.__all__` lists a name that doesn't exist | `AttributeError: module 'reasons.review' has no attribute '…'` during the `*` import. |
| Circular import (see Invariant 6) | `ImportError` naming the symbol, at the point `reasons_lib.review` is first imported. |
| Downstream `from reasons_lib.review import typo` | `ImportError`, at *call site* — not caught by any static checker, since `F403` is suppressed and the module has no `__all__` or `TYPE_CHECKING` stub. |

Because the shim is a pure re-export, **any exception raised by `reasons.review` logic at call time propagates unchanged** — the traceback frames show `reasons/review.py`, not `reasons_lib/review.py`, which is a deliberate (and generally desirable) consequence of not wrapping.

---

## Topics to Explore

- [file] `reasons/review.py` — The canonical implementation; everything this shim exports is defined or re-exported there. Read it to know the actual public surface, whether it declares `__all__`, and whether it holds module-level mutable state that the snapshot semantics would make stale.
- [file] `reasons/__init__.py` — Determines whether the `reasons.api → reasons_lib.review → reasons.review` cycle is safe, since it runs before `reasons.review` during the shim's wildcard import.
- [file] `reasons/api.py` — One of the three canonical modules that imports the *legacy* shim; check whether that edge can be retargeted to `reasons.review` to break the cross-tree dependency.
- [file] `tests/test_review.py` — Shows exactly which names the test suite expects from `reasons_lib.review`, i.e. the de-facto contract the shim must keep satisfying.
- [general] `reasons-vs-reasons-lib-migration` — The dual-tree layout and why canonical code still routes through legacy shims; the answer determines whether `reasons_lib/` can ever be deleted.

## Beliefs

- `review-shim-is-pure-reexport` — `reasons_lib/review.py` defines no functions, classes, or constants of its own; its only executable statement is a wildcard import from `reasons.review`.
- `review-shim-requires-reasons-review` — Importing `reasons_lib.review` necessarily imports the `reasons` package and the `reasons.review` module first, running their full module bodies.
- `star-import-excludes-underscore-names` — `reasons_lib.review` lacks every name in `reasons.review` that begins with an underscore, regardless of whether that name exists there.
- `star-import-binds-snapshot-not-proxy` — Names are copied at import time, so rebinding a name in `reasons.review` afterward does not change `reasons_lib.review`'s attribute, though mutation of the shared object is visible through both.
- `canonical-package-imports-legacy-shim` — `reasons/api.py`, `reasons/repair.py`, and `reasons/review_justifications.py` import `reasons_lib.review`, creating a cross-tree import edge from the new package back to the legacy one.
- `review-shim-suppresses-f401-f403` — The `# noqa: F401,F403` comment is required for linters to accept the otherwise-unused wildcard import.
