---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Understand `prepend` vs `importlib` vs `append` and how `__init__.py` changes module identity and shadowing.

# `prepend` vs `importlib` vs `append`, and `__init__.py` module identity

## TL;DR — the premise doesn't quite match this repo

The three mechanisms you asked about (sys.path `prepend`, `append`, and dynamic `importlib` loading) are **not used at all** in this codebase:

| Mechanism | Where it would live | What the observations show |
|-----------|---------------------|----------------------------|
| `sys.path.insert(0, …)` (prepend) | anywhere | `grep_sys_path_manipulation` → **0 matches** |
| `sys.path.append(…)` (append) | anywhere | same grep → **0 matches** |
| `importlib.import_module` / `spec_from_file_location` | anywhere | `grep_importlib_usage` → **2 matches, both `importlib.metadata.version`** |
| `__init__.py` identity/shadowing | package dirs | The only real instance is the `reasons_lib` shim |

So the right answer is: (a) explain what each mechanism does, (b) show where this repo *would* exercise it, and (c) explain the one place where `__init__.py` identity genuinely matters here — the `reasons_lib` compatibility shim.

---

## 1. `prepend` vs `append` on `sys.path`

`sys.path` is a list of string paths. `import X` scans it left-to-right and uses the first entry that answers.

```python
sys.path.insert(0, "/some/dir")   # prepend — highest priority
sys.path.append("/some/dir")      # append  — lowest priority
```

- **prepend** shadows anything already importable. If `./vendor/reasons` is prepended and contains `reasons/`, that wins over the pip-installed `reasons`. This is how override-injection works — and how accidental shadowing breaks things (one `sqlite.py` in cwd poisons every import process-wide).
- **append** is the safe fallback: installed packages win; your directory is consulted only as a last resort. Common pattern for optional plugin directories.

**Where this repo would need it — but doesn't:** `tests/conftest.py:22` does `from reasons.pg import PgApi` with no `sys.path` premable. That means `reasons` is resolved either because the package is installed in editable mode (`pip install -e .`) or because the test session starts with the repo root on `sys.path` (e.g. `python -m pytest`, which puts cwd at `sys.path[0]`). Because `tests/` has **no `tests/__init__.py`**, pytest's default `prepend` import mode inserts `tests/`, not the repo root, into `sys.path` — so the repo must be providing `reasons` another way. That is important: **there is no explicit sys.path contract in the code**, only an implicit one supplied by the Python invocation.

---

## 2. `importlib` — when `sys.path` order isn't enough

`importlib` gives you call-level control over imports instead of process-wide `sys.path`:

- `importlib.import_module("reasons.pg")` — behaves exactly like `import reasons.pg` (still obeys `sys.path`), but the name can be computed at runtime. Useful for plugin strings, entry points, config-driven dispatch.
- `importlib.util.spec_from_file_location("x", "/abs/x.py")` + `module_from_spec` + `exec_module` — loads a specific file **without searching `sys.path` at all**. This is the escape hatch when you must guarantee "this exact file, not whatever wins the search".
- `importlib.metadata.version("pkg-name")` — reads the **installed distribution** metadata, not the source tree. That's the only flavour used here.

The two uses in this repo:

- `reasons/cli.py:13` — `from importlib.metadata import version as _pkg_version`
- `reasons/metadata.py:17` — `from importlib.metadata import version`

These are metadata lookups (e.g. `version("ftl-reasons")` → `"0.58.0"` from `pyproject.toml:6`), **not** module loading. Caveat to know: `importlib.metadata.version` raises `PackageNotFoundError` when the package isn't installed. If you clone and run `python -m reasons` from source without `pip install -e .`, that path can blow up. This is the most "importlib"-adjacent thing in the codebase.

Rule of thumb:

| Need | Use |
|------|-----|
| Control search order (add fallback) | `sys.path.append` |
| Override an installed package | `sys.path.insert(0, …)` |
| Dynamic module name, still normal resolution | `importlib.import_module` |
| Load a specific file, ignore resolver | `spec_from_file_location` |
| Read installed version/metadata | `importlib.metadata.version` |

---

## 3. `__init__.py` — module identity

An `__init__.py` makes a directory a **regular package**. This is where module identity gets subtle:

- **`sys.modules` identity**: `import reasons` returns a singleton stored at `sys.modules["reasons"]`. Import it again — same object. But `sys.modules["reasons_lib"]` is a **distinct object** with `__name__ == "reasons_lib"`.
- **Names re-bound, not copied**: `reasons_lib/__init__.py:2` is `from reasons import *  # noqa: F401,F403`. That rebinds names in `reasons_lib`'s namespace. Consequence:
  - `reasons_lib.Node is reasons.Node` → **`True`** (same class object)
  - `isinstance(x, reasons_lib.Node)` works for objects built from `reasons.Node`
  - `reasons_lib.Node.__module__ == "reasons"` (identity is preserved at the class level, not the package level)
  - Pickling a `reasons.Node` yields `reasons.Node` on unpickle — never `reasons_lib.Node`. Any code that serializes across versions of the shim must account for that.
- **Namespace-package alternative**: if `reasons_lib/__init__.py` were deleted, `import reasons_lib` would still succeed (PEP 420), but it would be an **empty namespace package** — the `from reasons import *` body would not execute, and `reasons_lib.Node` would raise `AttributeError`. So this 3-line `__init__.py` is doing real work, not just marking a directory.

The docstring at `reasons_lib/__init__.py:1` — `"""Backwards-compatibility shim — use ``import reasons`` instead."""` — confirms it's an intentional alias package published for old downstreams that still do `import reasons_lib`. `pyproject.toml` line ~30 ships it: `packages = ["reasons", "reasons_lib"]`.

Note: `grep_reasons_lib_references` returns **0 matches inside this repo**. The shim has **no internal consumers** — it exists purely for external users. That means there is no test here exercising `import reasons_lib` or asserting `reasons_lib.Node is reasons.Node`.

---

## 4. Shadowing — the rules that matter (even if unused)

Shadowing = one entry in the search sequence wins over another. Three rules worth internalising:

1. **`sys.path` order**: first match wins. A `prepend`'d dir beats site-packages; an `append`'d dir loses to it.
2. **Package vs module**: if both `reasons.py` and `reasons/__init__.py` sit in the same dir, Python 3 picks the package. In this repo that's not a live risk (`pyproject.toml:30` ships `reasons` as a package; there's no `reasons.py`).
3. **Regular package vs namespace package**: a regular package (has `__init__.py`) on an earlier `sys.path` entry shadows a namespace package of the same name on a later one. This is why an editable checkout of `reasons/` can shadow a pip-installed `reasons` if the checkout path is discovered first — which is exactly what pytest does via its `prepend` import mode.

**Crucial non-shadow**: `reasons` and `reasons_lib` are *not* shadowing each other. Different names, different `sys.modules` entries, different `__name__`. The shim `import`s the real package at import time; there's no aliasing of one to the other at the resolver level. Calling it "shadowing" would be a category error.

---

## What's missing from the observations

- **Why `reasons_lib` exists at this commit.** Likely a rename (`reasons_lib` → `reasons`), but nothing in the snippet proves that. Check `git log -- reasons_lib/`.
- **How `reasons` is importable from `tests/conftest.py` without `sys.path` manipulation.** Depends on the test runner invocation (editable install vs `python -m pytest`). Not encoded in `pyproject.toml` — no `[tool.pytest.ini_options] pythonpath` is set (only `testpaths`, line ~32).
- **Any test for `reasons_lib`.** None found by grep.
- **Where `importlib.metadata.version` failures are handled** — `PackageNotFoundError` isn't covered in the snippet.

---

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The 3-line shim; understand why `from reasons import *` preserves class identity but not package identity
- [file] `pyproject.toml` — `[tool.setuptools].packages` and `[project.scripts]`; how two top-level packages ship together and why `testpaths` doesn't add `pythonpath`
- [function] `reasons/cli.py:main` — Where `_pkg_version("ftl-reasons")` is called; observe `PackageNotFoundError` semantics under non-editable installs
- [file] `tests/conftest.py` — Fixture wiring and the implicit `sys.path` assumption behind `from reasons.pg import PgApi`
- [general] `namespace-package-vs-regular-package` — Try deleting `reasons_lib/__init__.py` locally; note the shim still imports but star-re-export breaks (PEP 420 behavior)

---

## Beliefs

- `reasons-lib-is-a-star-import-shim` — `reasons_lib/__init__.py:2` re-exports every public `reasons` name via `from reasons import *`, so class objects are shared but `__module__` stays `"reasons"`
- `no-sys-path-manipulation-in-repo` — `grep "sys\.path"` returns zero matches; import resolution relies on the runtime invocation or editable install
- `importlib-is-used-only-for-metadata` — Both `importlib` imports (`reasons/cli.py:13`, `reasons/metadata.py:17`) are `importlib.metadata.version`, not dynamic module loading
- `setuptools-ships-both-top-level-packages` — `pyproject.toml` lists `packages = ["reasons", "reasons_lib"]`, so both are installed as top-level importable packages
- `reasons-lib-has-no-internal-consumers` — `grep "reasons_lib"` outside its own file returns zero matches; the shim targets external downstreams only
