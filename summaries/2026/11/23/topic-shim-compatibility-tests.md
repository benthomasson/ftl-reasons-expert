---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Add or inspect tests that import `reasons_lib.check_stale` to catch star-import and public-API regressions before removing the shim.

## What the task is actually asking

Before deleting `reasons_lib/check_stale.py`, you need to be sure of two things:

1. **Nothing currently depends on the shim** — so removal won't break callers (imports, star imports, re-exports).
2. **Any tests you add or inspect actually exercise the shim path** — so a behavior change on either side (`reasons_lib.check_stale` or `reasons.check_stale`) fails the test suite *before* the shim disappears.

The observed state is that (1) is basically confirmed, but (2) is not: existing tests bypass the shim entirely, and the shim's own exported surface has never been asserted.

## Current state of the code

### The shim: `reasons_lib/check_stale.py`

Three lines total:

```
"Backwards-compatibility shim — use ``from reasons.check_stale`` instead."
from reasons.check_stale import *  # noqa: F401,F403
```

- Line 2 is the only `from reasons.check_stale import *` in the repo (per `find_star_imports_from_check_stale_modules`).
- No explicit `__all__` is present in this file, so the exported surface is determined **entirely** by the canonical module's star-import semantics.
- Its sibling `reasons_lib/__init__.py` also uses the same pattern (`from reasons import *`), so `reasons_lib.check_stale` can be reached either as a submodule or (potentially) shadowed by a `check_stale` symbol re-exported from `reasons`.

### The canonical module: `reasons/check_stale.py` (473 lines)

The visible first 200 lines expose these public callables:

- `hash_file(path)` — full SHA-256 of file content.
- `resolve_source_path(source, repos, db_dir, agent)` — source-string → absolute path resolver with agent → db_dir → repos → `~/git` fallback order.
- `check_stale(network, repos, db_dir, upgrade_hashes, git_aware)` — returns `(results, upgraded, sha_bumped)`.
- `hash_sources(network, repos, force, db_dir)` — backfill source hashes.

It also imports `hashlib`, `re`, `subprocess`, `date`, `Path`, and `Network` from `.network`, plus references private helpers `file_lines_changed_since` and `get_file_commit_sha`. Since no `__all__` appears in the first 200 lines, the star import almost certainly leaks the stdlib imports and `Network` through the shim — worth pinning explicitly.

### Existing tests bypass the shim

- `tests/test_check_stale.py` line ~9: `from reasons.check_stale import check_stale, hash_file, hash_sources, resolve_source_path`
- `tests/test_check_stale_issue25.py` line ~11: `from reasons.check_stale import check_stale, hash_file`

Both import the **canonical** module directly. So today:

- If a name is renamed in `reasons/check_stale.py`, these tests catch it — but the shim's re-export surface is never touched.
- If the shim itself breaks (e.g., someone deletes the `*` import, or introduces an `__all__` that omits a symbol), **no test fails**. Removal would then appear safe even if it isn't.
- `find_imports_of_check_stale_shim` returns **0 matches**, so no test currently references the shim.

That means the current test suite does not prove the shim is safe to remove; the grep result only proves *nothing visible* uses it.

## What tests to add or inspect

To catch star-import and public-API regressions before removal, add a new test file (e.g., `tests/test_reasons_lib_check_stale_shim.py`) that imports the shim path explicitly. Concrete cases worth writing:

1. **Shim is importable.**
   ```python
   import importlib
   mod = importlib.import_module("reasons_lib.check_stale")
   assert mod is not None
   ```
   Ensures the submodule path still resolves through `reasons_lib/__init__.py`.

2. **Identity re-export test for each public name.**
   ```python
   import reasons.check_stale as canonical
   import reasons_lib.check_stale as shim
   for name in ("check_stale", "hash_file", "hash_sources", "resolve_source_path"):
       assert getattr(shim, name) is getattr(canonical, name)
   ```
   Catches a shim that re-implements instead of re-exporting, or that lost a name after a canonical rename.

3. **Star-import surface lock.**
   ```python
   def test_shim_surface_matches_canonical():
       import reasons.check_stale as canonical
       import reasons_lib.check_stale as shim
       expected = {n for n in dir(canonical) if not n.startswith("_")}
       actual   = {n for n in dir(shim)      if not n.startswith("_")}
       # expected - actual == set() if shim is faithful; both directions if __all__ should narrow
       assert expected <= actual
   ```
   This is the test that directly encodes "star-import regression": if `reasons.check_stale` gains or loses a public symbol, this fails until the shim is updated or removed.

4. **Star-import from the shim.**
   ```python
   from reasons_lib.check_stale import *  # noqa
   # verify check_stale, hash_file, hash_sources, resolve_source_path are all present
   ```
   Covers callers who may still be doing `from reasons_lib.check_stale import ...` or `*`.

5. **Explicit note on leaked imports.**
   ```python
   def test_shim_documents_leaked_stdlib_names():
       import reasons_lib.check_stale as shim
       for leaked in ("hashlib", "re", "subprocess", "Path", "Network", "date"):
           assert hasattr(shim, leaked)  # pin current behavior; adjust to `not hasattr` after `__all__` is added
   ```
   This makes the "star import leaks stdlib names" behavior explicit so a future `__all__` in the canonical module doesn't silently change the shim's public surface.

6. **Removal readiness test (optional but useful).**
   Once verification is done, replace the shim with a `pytest.warns`/`DeprecationWarning` and a test asserting that importing `reasons_lib.check_stale` emits the warning. That gives you a two-step removal path.

**Inspection side:** since `find_imports_of_check_stale_shim` returns 0, the existing `tests/test_check_stale.py` and `tests/test_check_stale_issue25.py` only need to remain pointed at `reasons.check_stale` after the shim is deleted — no migration required. The test file above is purely additive.

## What the observations do not tell us

- **Lines 200–473 of `reasons/check_stale.py`** — we don't see whether `__all__` is defined there, which determines whether stdlib names are actually leaked or excluded. This changes the shape of test #5.
- **Lines 200–377 of `tests/test_check_stale.py`** — hidden fixtures/classes may already reference the shim path indirectly.
- **`reasons/__init__.py`** — we don't see whether it imports `check_stale`, which affects whether `reasons_lib.check_stale` resolves to the submodule or to a re-exported function via `from reasons import *`.
- **Broader grep patterns** — the current `find_imports_of_check_stale_shim` pattern only matches `from reasons_lib.check_stale import` / `import reasons_lib.check_stale`. It will not catch `from reasons_lib import check_stale` or `importlib.import_module("reasons_lib.check_stale")`. Re-run with those patterns before declaring the shim unused.

## Topics to Explore

- [file] `reasons/check_stale.py` — Read lines 200–473 to confirm whether `__all__` is defined and to enumerate private helpers (`file_lines_changed_since`, `get_file_commit_sha`) that may be leaked through the shim's star import.
- [file] `reasons/__init__.py` — Determines whether `check_stale` is re-exported as a callable and whether it shadows the submodule path when `import reasons_lib.check_stale` is executed.
- [file] `reasons_lib/__init__.py` — The `from reasons import *` line interacts with submodule resolution; verify that `reasons_lib.check_stale` still points at the shim after this import runs.
- [general] wider-grep-for-shim-imports — Expand the search to `from reasons_lib import check_stale`, `importlib.import_module("reasons_lib.check_stale")`, and bare string references before removal.
- [diff] tests/test_reasons_lib_check_stale_shim.py — New file proposed above; the actual diff to write and merge before deleting `reasons_lib/check_stale.py`.

## Beliefs

- `shim-reexports-canonical-check-stale` — `reasons_lib/check_stale.py` re-exports all public symbols of `reasons.check_stale` through a single `from reasons.check_stale import *` statement on line 2.
- `existing-tests-bypass-shim` — Both `tests/test_check_stale.py` and `tests/test_check_stale_issue25.py` import from `reasons.check_stale` directly, so no existing test exercises the shim's re-export surface.
- `shim-has-no-discovered-callers` — A grep for `from reasons_lib.check_stale import` / `import reasons_lib.check_stale` across the repo returns zero matches; no visible code depends on the shim today.
- `shim-surface-is-defined-by-canonical-star-import` — Because the shim does not declare `__all__`, its public surface (including any leaked stdlib names like `hashlib`, `Path`, `Network`) is inherited verbatim from `reasons/check_stale.py`'s star-import semantics.
- `shim-removal-is-safe-only-after-explicit-surface-tests` — Deleting `reasons_lib/check_stale.py` is only provably safe once a test asserts each expected symbol is identical to the canonical module's, since neither the current test suite nor the existing grep pattern covers alternate import forms.
