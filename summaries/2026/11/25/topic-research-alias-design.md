---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Why a bare alias was chosen instead of a wrapper (`__name__`/`__doc__`/signature preservation, import-time cost, test surface).

## Why a bare alias instead of a wrapper

**What was chosen.** `reasons_lib/api.py` is a 3-line module whose entire body is `from reasons.api import *  # noqa: F401,F403`. The observations confirm this is an intentional backwards-compat shim, not an accident of construction: `CLAUDE.md` line 35 lists `reasons_lib/` as "backwards-compat shim — re-exports from reasons/", and `CHANGELOG.md` line 136 shows the API layer originally lived at `reasons_lib.api` before the package rename. So the alias direction is "old path → new module."

**1. `__name__` / `__doc__` / signature preservation come for free — and are more correct than a wrapper.** With `import *`, every exported callable is literally the same object (`reasons_lib.api.foo is reasons.api.foo`). That means `foo.__name__`, `foo.__doc__`, `foo.__module__`, `inspect.signature(foo)`, and `pickle` behavior are all self-consistent with `reasons.api` without any decoration. A hand-written wrapper would need `functools.wraps` on every function and would still get `inspect.signature` wrong for any function that uses `*args, **kwargs` forwarding — or expose a misleading `__module__`. Consider the scale: `reasons/api.py` is 5886 lines with many public functions (only the first 200 lines are shown, and those alone define `_is_visible`, `_get_node_tags`, `_filter_by_tags`, `_resolve_namespace`, `_with_network`, `clear_network_cache`, `init_db`, `_pg_dispatch`). Wrapping each of those is a large, drift-prone file that would need manual maintenance every time a function is added to `reasons.api`. The bare alias is structurally immune to that drift.

**2. Import-time cost is minimized rather than paid twice.** Any wrapper implementation must still `import reasons.api` to delegate, so it bears the full import cost of the 5886-line module plus the cost of executing `def foo(...): return _orig_foo(...)` for every name. `import *` imports `reasons.api` once and binds names by reference — the module-level work in `reasons_lib/api.py` is exactly one import. For a pure compatibility shim, that is the minimum possible execution surface.

**3. There is zero dedicated test surface, and the alias design keeps it that way.** The observation `tests_covering_module_alias_surface.total_count = 0` — no test matches `reasons_lib`, `reasons\.api`, `import reasons`, `__name__`, `__doc__`, `signature`, etc. — is consistent with the pass-through design: because the shim exposes the *same* function objects, any test that exercises `reasons.api.init_db` transitively exercises `reasons_lib.api.init_db`. A wrapper, by contrast, would introduce new runtime behavior (argument forwarding, default handling, exception pass-through, correct `**kwargs` propagation) that would require tests to reach parity — tests that would not be exercised by `reasons.api`'s own suite. Adding a shim that needs its own tests to be safe is a net loss for a package whose Python dependency list is `[]` (per `pyproject.toml`).

**4. Identity semantics survive.** Anything that compares `getattr(reasons.api, name)` against `reasons_lib.api.<name>` — plugin registries keyed by function object, dispatch tables, `is` checks, `functools.partial` reuse — sees the same object. A wrapper would silently break this. Similarly, `from reasons_lib.api import Storage` in older code still binds the *real* `Storage` class from `reasons.storage` (re-exported through `reasons/api.py`'s own `from .storage import ... Storage` at lines 240-ish in the shown snippet), so `isinstance` checks across the two import paths keep working.

**5. The cost accepted: namespace leakage.** With `import *` and no `__all__` visible in the first 200 lines of `reasons/api.py`, `reasons_lib.api` also exposes the module-level imports — `copy`, `json`, `sqlite3`, `datetime`, `Path`, `Callable`, `Justification`, `Network`, `Storage`, `PROPOSALS_SCHEMA`, `DEFAULT_DB`, `_network_cache`, `logger`. A wrapper would have avoided this by explicitly re-exporting a curated list. The design trade-off is: keep the shim to one line and accept incidental leakage, because the only consumers who would ever notice are those doing `from reasons_lib.api import *` themselves — and named imports (`from reasons_lib.api import init_db`), the actual backwards-compat scenario, are unaffected.

**What's missing to fully validate this story.** The observations do not contain (a) an explicit rationale comment in the code, (b) the content of `reasons_lib/__init__.py` (only `api.py` is shown), (c) whether `reasons/api.py` defines an `__all__` at the bottom of its 5886 lines, (d) any evidence of external consumers still importing `reasons_lib.api` (which would confirm the compat requirement is live rather than aspirational), or (e) a CHANGELOG entry explaining the rename. The "bare alias over wrapper" claim is well-supported by the code, but the "why specifically this shim at this time" is inferred from `CLAUDE.md` and `CHANGELOG.md` rather than stated.

## Topics to Explore

- [file] `reasons/api.py` — Scan the full 5886 lines, especially the tail, to check whether `__all__` is defined and which names it exports (this determines exactly what `import *` leaks into `reasons_lib.api`).
- [file] `reasons_lib/__init__.py` — Not shown; determines whether the top-level `reasons_lib` package also shims `reasons/__init__.py`'s dataclasses (`Justification`, `Node`, `Nogood`) or only the `api` submodule.
- [file] `CHANGELOG.md` — Line 136 references the original `reasons_lib.api` layer; look for the rename entry explaining when `reasons` superseded `reasons_lib`.
- [file] `CLAUDE.md` — Lines 24 and 35 describe the two-package layout; useful for confirming whether any removal of `reasons_lib` is planned.
- [general] `backwards-compat-import-strategy` — Determine whether external downstream packages still import `from reasons_lib.api import ...`, which would explain why the alias still exists and why wrapper-level tests were deliberately not added.

## Beliefs

- `reasons-lib-api-is-bare-reexport` — `reasons_lib/api.py` is a 3-line module whose only executable statement is `from reasons.api import *`, re-exporting every public name of `reasons/api.py` unchanged.
- `reasons-lib-shim-exists-for-backwards-compat` — `CLAUDE.md` line 35 and `CHANGELOG.md` line 136 document that `reasons_lib.api` was the original API-layer path and `reasons_lib/` is retained only as a compat shim over `reasons/`.
- `bare-alias-preserves-function-identity` — Because the shim uses `import *`, `reasons_lib.api.foo is reasons.api.foo`, so `__name__`, `__doc__`, `__module__`, `inspect.signature`, and pickling match the canonical module without `functools.wraps`.
- `alias-shim-has-no-dedicated-tests` — Zero tests match `reasons_lib`, `reasons\.api`, `__name__`, `__doc__`, `signature`, or `import reasons`; the shim relies on `reasons.api`'s own test suite for coverage.
- `import-star-leaks-module-level-imports` — `reasons/api.py` defines no `__all__` in its first 200 lines, so `from reasons.api import *` also exposes `sqlite3`, `copy`, `json`, `Path`, `Storage`, `Network`, `Justification`, and `PROPOSALS_SCHEMA` through `reasons_lib.api`.
