---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How (or whether) the project warns callers off `reasons_lib` before removal, and what the removal criterion is.

## How `reasons_lib` is being deprecated (or not)

**Short answer: it isn't warned off at runtime, and no removal criterion is documented anywhere in the observed code.**

### What `reasons_lib` actually is

`reasons_lib/__init__.py` is a three-line shim:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

It re-exports everything from `reasons` and contains no other logic. Its only signals to callers are:

1. **The module docstring** (`reasons_lib/__init__.py:1`) — tells readers to use `import reasons`.
2. **Documentation** in `CLAUDE.md` (lines 24 and 35), which calls it a "backwards-compat shim — re-exports from reasons/".

### What's missing (i.e., no active deprecation mechanism)

- **No `DeprecationWarning` / `warnings.warn(...)`** at import time. The deprecation-warning search across the repo returned only three unrelated hits:
  - `reasons/api.py:3919` — a string literal `'deprecated'` used as a *content-analysis word*, not a real warning.
  - `reasons/cli.py:3064` — a help string: `"(deprecated alias for --domain-id)"`.
  - `reasons/cli.py:3854` — a ridealong help string: `"Alias for 'repair' (deprecated)"`.

  None of these touch `reasons_lib`.
- **No tests assert a warning** — the grep for `reasons_lib|DeprecationWarning|warnings` in tests returned zero matches.
- **No `reasons_lib` references in Python source anywhere** — the `search_reasons_lib_references_python` pattern found zero matches, meaning no internal code imports it, and no code path exercises it.
- **`reasons_lib` is still shipped** — `pyproject.toml` still lists it in `[tool.setuptools] packages = ["reasons", "reasons_lib"]`, so every wheel still installs it.

### Removal criterion

There is **no removal criterion** in the codebase. A repo-wide grep for `remov|removal|criterion|sunset|migration` returned exactly one hit — an unrelated sentence in `.claude/skills/reasons/SKILL.md:109` about `reasons assert` not removing SL justifications. There is:

- no version cutoff (`"to be removed in v1.0"`),
- no `DeprecationWarning` gate,
- no `pending_removal` marker,
- no changelog entry announcing a sunset,
- no migration checklist.

So the effective policy is: **silent, indefinite backwards compatibility** — the shim's docstring tells humans to migrate, but no machine-visible signal forces them to, and nothing declares when the shim will be dropped.

### Comparison with the project's other deprecations

`reasons/cli.py` handles its deprecated aliases in the same spirit — just help-text labels (`--domain-id` alias at line 3064, `research` subcommand at line 3854) with no runtime `DeprecationWarning`. So the shim's lack of a warning is *consistent* with how this project deprecates other surface area: documentation-only, no enforcement, no declared removal date.

### What the observations don't tell us

- Whether the shim is mentioned in `README.md`, `docs/`, or the CHANGELOG under a "Deprecations" heading (only one `reasons_lib` hit in `CHANGELOG.md:136`, and it's about `reasons_lib.api`, not the shim itself).
- Whether any down-stream consumer (e.g., a dependent package) pins against `reasons_lib` and would justify keeping it.
- The contents of any issue tracker / roadmap that might hold an unwritten sunset plan.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The full shim; note that its docstring is its *only* deprecation signal and there is no `warnings.warn` call.
- [file] `pyproject.toml` — Confirms `reasons_lib` is still a published package in `[tool.setuptools] packages`, so the shim ships with every release.
- [file] `CHANGELOG.md` — Check whether any version entry announces a plan to remove `reasons_lib`, or whether the `reasons_lib.api` mention at line 136 refers to a separate submodule.
- [general] `deprecation-strategy` — Compare how CLI aliases (`reasons/cli.py:3064`, `reasons/cli.py:3854`) and the Python shim are both deprecated via documentation only, suggesting a project-wide pattern.
- [file] `CLAUDE.md` — The only doc explicitly describing `reasons_lib` as a "backwards-compat shim"; useful as the authoritative human-facing description.

## Beliefs

- `reasons-lib-is-a-thin-re-export` — `reasons_lib/__init__.py` imports everything from `reasons` via `from reasons import *` and contains no other code.
- `reasons-lib-has-no-runtime-warning` — Importing `reasons_lib` emits no `DeprecationWarning`; the only migration signal is the module docstring.
- `reasons-lib-is-still-packaged` — `pyproject.toml` lists `reasons_lib` under `[tool.setuptools] packages`, so the shim is installed for every user of `ftl-reasons`.
- `reasons-lib-has-no-removal-criterion` — No sunset version, migration deadline, or deprecation checklist exists for `reasons_lib` anywhere in the observed files.
- `cli-deprecations-use-help-text-only` — The project's other deprecations (`reasons/cli.py:3064`, `reasons/cli.py:3854`) also rely solely on help-text labels rather than runtime warnings, mirroring the shim's approach.
