---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: The broader `reasons_lib` → `reasons` rename: whether `reasons_lib/` is deprecated wholesale, which modules are still shims vs. real code, and the planned removal timeline.

## Summary

`reasons_lib/` is not the canonical implementation. The real code lives under `reasons/`, and `reasons_lib/` exists as a backwards-compatibility shim for external imports.

There is **no evidence of wholesale deprecation** of `reasons_lib/`. It is still explicitly packaged, has no observed `DeprecationWarning`, and has no documented removal version or date.

## What the rename actually is

- `CLAUDE.md:24` says `reasons/` is importable as `import reasons` and is “also aliased as reasons_lib for backwards compat.”
- `CLAUDE.md:35` says `reasons_lib/` is a “backwards-compat shim — re-exports from reasons/.”
- `CHANGELOG.md` v0.3.0 records the historical rename from `rms` → `reasons` for CLI, library, and package name, based on an LLM accuracy ablation.
- `CHANGELOG.md:136` references the older API path `reasons_lib.api` in v0.1.0, which is consistent with `reasons_lib` being the legacy import namespace.

So the broader migration is: `reasons/` is canonical; `reasons_lib/` is a compatibility alias for older consumers.

## Shim vs. real code

### Real code: `reasons/`

`CLAUDE.md:24-33` lists the actual implementation modules under `reasons/`:

- `reasons/__init__.py`
- `reasons/network.py`
- `reasons/api.py`
- `reasons/storage.py`
- `reasons/cli.py`
- `reasons/import_beliefs.py`
- `reasons/export_markdown.py`
- `reasons/check_stale.py`
- `reasons/merkle.py`
- `reasons/compact.py`

The CLI entry points also target the real package, not the shim:

```toml
[project.scripts]
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

### Shim: `reasons_lib/`

The only observed `reasons_lib` module is:

```python
# reasons_lib/__init__.py
from reasons import *  # noqa: F401,F403
```

That is a wildcard re-export shim. There is no observed real logic in `reasons_lib/`.

There is also no observed internal usage of `reasons_lib`:

- Search for `from reasons_lib|import reasons_lib` returned zero matches.
- So the compatibility surface appears aimed at external users, not internal code.

### Caveat: possible missing shim coverage

The v0.1.0 changelog mentions `reasons_lib.api`, but the observations do not show a `reasons_lib/api.py` file. Only `reasons_lib/__init__.py` is shown. That means I cannot confirm from the observations whether legacy paths like:

```python
from reasons_lib.api import ...
```

still work. The wildcard re-export in `reasons_lib/__init__.py` may or may not expose `api` depending on what `reasons/__init__.py` exports.

Also, the shim-pattern search returned zero matches even though `reasons_lib/__init__.py:1` clearly contains `from reasons import *`. That search result appears inconsistent with the direct file read. Treat the direct file content as authoritative for that file.

## Is `reasons_lib/` deprecated wholesale?

No, not according to the observed docs or packaging.

Evidence:

- `CLAUDE.md:35` calls it a backwards-compat shim, not deprecated code.
- `pyproject.toml` still includes both packages:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

- No `DeprecationWarning`, `warnings.warn`, or equivalent was observed in `reasons_lib`.
- No changelog entry announces `reasons_lib` deprecation or removal.

So the accurate description is:

> `reasons_lib/` is a compatibility-only shim package. It is not documented as deprecated and is still shipped.

## Planned removal timeline

There is **no documented removal timeline** in the observations.

The changelog, README, `CLAUDE.md`, and `pyproject.toml` do not specify:

- a removal version,
- a deprecation window,
- a migration deadline,
- or a warning mechanism.

Because `reasons_lib` is still in `[tool.setuptools] packages`, removing it would currently be a breaking packaging change for any external consumer still importing it.

What is missing to answer the timeline question:

- a roadmap or issue reference,
- release notes after the latest observed changelog entry,
- tests that assert legacy import compatibility,
- and the full contents of `reasons_lib/` to confirm whether only `__init__.py` exists.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm whether this is the only shim file and whether it exposes submodules like `reasons_lib.api`.
- [file] `reasons/__init__.py` — See what `from reasons import *` actually exports and whether it includes `api`.
- [general] `reasons-lib-import-compatibility` — Search external docs, tests, and release notes for `reasons_lib.api` and `reasons_lib` usage.
- [general] `reasons-lib-removal-policy` — Look for roadmap or issue references that mention deprecating or removing the compatibility package.
- [file] `CHANGELOG.md` — Trace the `rms` → `reasons` rename and check whether later versions mention `reasons_lib` migration.

## Beliefs

- `reasons-is-canonical-package` — The CLI entry points target `reasons.cli:main` and `reasons.mcp_server:main`, not `reasons_lib`.
- `reasons-lib-is-a-backwards-compat-shim` — `reasons_lib/__init__.py` re-exports the `reasons` package via `from reasons import *`.
- `reasons-lib-is-still-packaged` — `pyproject.toml` includes both `reasons` and `reasons_lib` in `[tool.setuptools] packages`.
- `reasons-lib-has-no-observed-deprecation-warning` — No `DeprecationWarning` or `warnings.warn` call was observed in `reasons_lib`.
- `reasons-lib-removal-timeline-is-undocumented` — The observed changelog, README, `CLAUDE.md`, and `pyproject.toml` do not specify a removal version or date for `reasons_lib`.
