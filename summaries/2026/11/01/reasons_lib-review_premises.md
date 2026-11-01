---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/review_premises.py

# `reasons_lib/review_premises.py`

## Purpose

This file is a **backwards-compatibility shim**. It exists solely so that any external code (or legacy internal code) that still does `from reasons_lib.review_premises import ...` continues to work after the canonical module was relocated (or is considered to be) under the `reasons` package. It re-exports the entire public namespace of `reasons.review_premises` into the `reasons_lib.review_premises` namespace. No logic lives here — it's a one-line redirect.

The docstring makes the intent explicit: *"use ``from reasons.review_premises`` instead."* This is a deprecation signal for maintainers of downstream packages.

## Key Components

There are **no original classes, functions, or constants** defined in this file. The single executable statement is:

```python
from reasons.review_premises import *  # noqa: F401,F403
```

- `import *` pulls every name not prefixed with `_` from `reasons.review_premises` into this module's `__dict__`.
- `# noqa: F401` suppresses the linter warning "imported but unused."
- `# noqa: F403` suppresses the linter warning "star import is not allowed."

The effective contract is: **whatever `reasons.review_premises` exposes at import time, `reasons_lib.review_premises` also exposes.** If the canonical module adds or removes a public name, this shim automatically tracks it (no manual list to keep in sync).

## Patterns

| Pattern | Detail |
|---|---|
| **Delegation / Facade** | The module delegates 100 % of its namespace to a single upstream module. Callers of `reasons_lib.review_premises` get identical behavior to callers of `reasons.review_premises`. |
| **Star-import re-export** | A common Python idiom for "re-export everything." It avoids an explicit `__all__` list in the shim, but also means the shim's API is *only* as stable as the upstream's `__all__` (or the absence of underscore-prefixed names). |
| **Parallel package tree** | The repo maintains two top-level packages — `reasons/` and `reasons_lib/` — with near-identical file sets. Every file in `reasons_lib/` is (presumably) a shim of the corresponding `reasons/` file. This lets the project rename its canonical package without breaking `pip install`-ed consumers that import from the old name. |
| **Lint suppression** | The two `noqa` codes are the standard pair for star-import shims; their presence signals "this is intentional, not a mistake." |

## Dependencies

**Upstream (this file imports from):**

- `reasons.review_premises` — the canonical implementation. This is the *only* runtime dependency. If that module is missing, this shim raises `ModuleNotFoundError` at import time.

**Downstream (these import this file):**

| Importer | Why it matters |
|---|---|
| `reasons/api.py` | The public API surface; it pulls review-premises helpers through the `reasons_lib` path (legacy or convenience). |
| `tests/test_repair_premises.py` | Test for premise-repair logic; imports via the shim, meaning the test exercises the same code path a legacy consumer would. |
| `tests/test_review_premises.py` | Direct test for the review-premises feature. |

Because the shim uses `import *`, a name collision in `reasons.review_premises` (e.g., two modules defining the same symbol) would be silently resolved by the last-imported name in the upstream's `__dict__` — the shim does no additional disambiguation.

## Flow

Execution is trivial:

1. Python imports `reasons_lib.review_premises`.
2. The single statement `from reasons.review_premises import *` executes.
   a. Python imports `reasons.review_premises` (triggering *its* full import chain).
   b. Python collects all public names from that module's namespace.
   c. Python injects them into `reasons_lib.review_premises.__dict__`.
3. The module is now fully loaded. Any subsequent `from reasons_lib.review_premises import some_func` resolves `some_func` in this module's namespace, which is a reference to the same object that lives in `reasons.review_premises`.

There is no branching, no side effects, no I/O beyond the upstream import.

## Invariants

- **Namespace parity**: At any point in time, `set(dir(reasons_lib.review_premises)) ⊇ set of public names in reasons.review_premises)`. (It also includes `__builtins__`, `__name__`, etc., but those are standard.)
- **Single source of truth**: All business logic must live in `reasons/review_premises.py`. Adding logic to the shim would silently shadow the canonical implementation for callers of the `reasons_lib` path, creating a two-headed implementation.
- **No circular import risk in the shim itself**: The shim only imports *forward* (into `reasons`). If `reasons.review_premises` were to import back from `reasons_lib`, you'd get a circular import. The current one-way direction keeps the graph acyclic.

## Error Handling

There is **no explicit error handling** in this file. Error propagation is entirely delegated:

- **Import-time failure**: If `reasons.review_premises` cannot be imported (missing dependency, syntax error, missing `pg` driver, etc.), the `ModuleNotFoundError` or whatever exception the upstream raises propagates unchanged to the caller of `import reasons_lib.review_premises`. The shim adds no wrapper, no retry, no logging.
- **Runtime failure**: Any exception raised by a function in `reasons.review_premises` propagates through the shim unchanged. The shim introduces no `try/except`.
- **Silent degradation risk**: Because `import *` only pulls *public* names, if the canonical module accidentally prefixes a function with `_`, it vanishes from the shim's namespace. Callers using the `reasons_lib` path would get an `ImportError` at their call site, while callers using the `reasons` path (who access the `_`-prefixed name directly) would still work. This asymmetry is the main hidden risk of the star-import pattern here.

---

## Topics to Explore

- [file] `reasons/review_premises.py` — The canonical implementation this shim re-exports; understanding its public API is prerequisite to understanding what the shim actually provides.
- [file] `reasons/review.py` — Sibling module in the same package; comparing it against `review_premises` clarifies the boundary between general review logic and premise-specific review.
- [file] `reasons_lib/repair_premises.py` — Another shim in the same directory; confirms (or contradicts) whether the entire `reasons_lib/` tree follows the same one-line re-export pattern.
- [repo] `pyproject.toml` — Check how `reasons` and `reasons_lib` are declared as packages (or a single package with two sub-packages) to understand the install-time layout and whether both are importable after `pip install`.
- [general] `package-rename-migration` — The broader strategy behind maintaining two parallel package trees; understanding the migration timeline (check `CHANGELOG.md`) reveals whether `reasons_lib` is slated for removal or is a permanent dual-support surface.

## Beliefs

- `reasons-lib-review-premises-is-a-reason-review-premises` — `reasons_lib.review_premises` is a pure re-export of `reasons.review_premises`; it defines zero original symbols and contains no business logic.
- `shim-star-import-tracks-upstream-public-names` — The shim's namespace is derived via `import *`, so any public name added to `reasons.review_premises` is automatically available through `reasons_lib.review_premises` without modifying the shim.
- `reasons-api-depends-on-reasons-lib-shim` — `reasons/api.py` imports review-premises functionality through the `reasons_lib` path rather than importing `reasons.review_premises` directly, coupling the public API to the compatibility layer.
- `removing-reasons-lib-breaks-legacy-callers` — Deleting the `reasons_lib/` tree would break `reasons/api.py` and the two test files that import via the `reasons_lib` path, making it a load-bearing compatibility layer rather than dead code.
