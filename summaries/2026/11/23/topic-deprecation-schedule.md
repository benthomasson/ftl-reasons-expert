---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Check whether `reasons_lib/__init__.py` emits a `DeprecationWarning` (it does not, per line 1-3 of the observation) and decide whether one should be added before deletion.

## What the observations show

`reasons_lib/__init__.py` does **not** emit a `DeprecationWarning`.

- The observed shim content in `read_reasons_lib_init` is only:
  ```python
  from reasons import *  # noqa: F401,F403
  ```
  It reports `total_lines: 3` and returns only this import line. There is no `import warnings`, no `warnings.warn(...)`, and no `DeprecationWarning`.
- The repo-wide search for `DeprecationWarning` returned `total_count: 0`, so there is no existing warning mechanism elsewhere that this shim triggers.
- `reasons/__init__.py` defines the core dataclasses `Justification`, `Node`, and `Nogood`; the shim simply re-exports them via the wildcard import.
- Documentation still describes `reasons_lib` as a compatibility layer:
  - `CLAUDE.md:24` says `reasons/` is aliased as `reasons_lib` for backwards compatibility.
  - `CLAUDE.md:35` says `reasons_lib/` is a backwards-compat shim re-exporting from `reasons/`.
- `pyproject.toml` still packages it: `[tool.setuptools] packages = ["reasons", "reasons_lib"]`.
- `.forge/report.json` is about a diary-update pipeline and contains no deletion plan or `reasons_lib` usage evidence, so it does not affect the decision.

## Should a `DeprecationWarning` be added before deletion?

**Recommendation: yes, if `reasons_lib` has ever been part of a public release and deletion is planned for a later release. Otherwise, if deletion is immediate and no external users are expected, skip the warning and delete it directly.**

Reasoning:

- Because `pyproject.toml` still lists `reasons_lib` as a packaged module, and `CLAUDE.md` explicitly documents it as a backwards-compatibility shim, it is currently presented as supported. Deleting it without a warning gives downstream importers no migration signal.
- A warning release gives users a chance to switch from:
  ```python
  import reasons_lib
  ```
  to:
  ```python
  import reasons
  ```
- A typical change would be:
  ```python
  import warnings

  warnings.warn(
      "reasons_lib is deprecated; import reasons instead",
      DeprecationWarning,
      stacklevel=2,
  )

  from reasons import *  # noqa: F401,F403
  ```
  Using `stacklevel=2` points the warning at the caller's import site. Note that Python ignores `DeprecationWarning` by default in many contexts; if the goal is to alert end users rather than library developers, `FutureWarning` may be more visible, but that would not literally be a `DeprecationWarning`.

Before actual deletion, also:

- Remove `"reasons_lib"` from `[tool.setuptools] packages` in `pyproject.toml`.
- Update or remove references in `CLAUDE.md:24` and `CLAUDE.md:35`.
- Check `CHANGELOG.md:136`, which mentions `reasons_lib.api`; that may be stale documentation or may indicate a broader deletion scope than just `reasons_lib/__init__.py`.
- Add or adjust a test using `pytest.warns(DeprecationWarning)` if you add the warning.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the exact final contents and add the warning there if the deprecation-release strategy is chosen.
- [file] `pyproject.toml` — The `[tool.setuptools] packages` list must remove `reasons_lib` when the shim is actually deleted.
- [file] `CLAUDE.md` — Lines 24 and 35 document `reasons_lib` as backwards-compatible; these need updating during deprecation or removal.
- [file] `CHANGELOG.md` — Line 136 mentions `reasons_lib.api`; verify whether that is stale docs or evidence of a submodule that also needs migration handling.
- [general] `external-usage-audit` — Search published-package consumers or release history to decide whether a warning release is needed before deletion.

## Beliefs

- `reasons-lib-init-is-wildcard-reexport` — `reasons_lib/__init__.py` contains only `from reasons import *  # noqa: F401,F403` and no warning call.
- `reasons-lib-init-does-not-emit-deprecation-warning` — The shim does not import `warnings` or invoke `DeprecationWarning`, and the repo-wide `DeprecationWarning` search returned zero matches.
- `reasons-lib-is-packaged-as-backwards-compat-shim` — `pyproject.toml` lists both `reasons` and `reasons_lib` in `[tool.setuptools] packages`, and `CLAUDE.md:35` calls `reasons_lib/` a backwards-compat shim.
- `repo-python-code-does-not-reference-reasons-lib` — The `reasons_lib` Python reference search returned zero matches.
- `reasons-package-defines-core-dataclasses` — `reasons/__init__.py` defines `Justification`, `Node`, and `Nogood` dataclasses that `reasons_lib` re-exports.
