---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Since both `reasons/` and `reasons_lib/` exist side-by-side, understanding the migration state (which consumers still import the legacy path and when the shim can be removed) is critical for any refactor of `reasons.check_stale`.

## Migration state for `reasons.check_stale`

The migration direction is the opposite of what the phrase “legacy path” might suggest: **`reasons/` is the canonical implementation, and `reasons_lib/` is the backwards-compatibility shim**.

### Canonical vs. shim

| Path | Role | Evidence |
|---|---|---|
| `reasons/check_stale.py` | Canonical implementation | 473 lines total; `def check_stale(...)` starts at line 62 |
| `reasons_lib/check_stale.py` | Compatibility shim | 3 lines; line 2 is `from reasons.check_stale import *` |
| `reasons_lib/__init__.py` | Package-level shim | line 2 is `from reasons import *` |
| `pyproject.toml` | Installs both packages | `[tool.setuptools] packages = ["reasons", "reasons_lib"]` |

The shim file itself confirms the target path:

```python
# reasons_lib/check_stale.py:1-2
"""Backwards-compatibility shim — use ``from reasons.check_stale`` instead."""
from reasons.check_stale import *  # noqa: F401,F403
```

So for any refactor of `reasons.check_stale`, the source of truth is `reasons/check_stale.py`. `reasons_lib/check_stale.py` simply forwards whatever `reasons.check_stale` exports.

### Who imports what

Observed internal consumers use the canonical `reasons.check_stale` path, either directly or relatively:

- `reasons/api.py:2703` — `from .check_stale import check_stale as _check`
- `reasons/api.py:2769` — `from .check_stale import hash_sources as _hash`
- `reasons/api.py:2797` — `from .check_stale import pin_sources as _pin`
- `reasons/api.py:2823` — `from .check_stale import pin_update as _update`
- `reasons/api.py:2854` — `from .check_stale import pin_lines as _pin_lines`
- `reasons/api.py:4253` and `4368` — `from .check_stale import resolve_source_path`
- `reasons/pg.py:801` and `858` — `from .check_stale import hash_file, resolve_source_path`
- `reasons/verify.py:57` — `from .check_stale import resolve_source_path`
- `tests/test_check_stale.py:9` — `from reasons.check_stale import check_stale, hash_file, hash_sources, resolve_source_path`
- `reasons/cli.py:1118-1119` — defines `cmd_check_stale`, then calls `api.check_stale(...)` indirectly

In the returned matches, **no application or test consumer imports `reasons_lib.check_stale`**. The only observed file that imports from `reasons.check_stale` on behalf of `reasons_lib` is the shim itself at `reasons_lib/check_stale.py:2`.

However, `find_check_stale_references` reports `total_count: 78` while only a subset is shown, and `find_legacy_reasons_imports` reports `total_count: 250` while only a subset is shown. So the safe statement is: **within the displayed matches, no consumer imports the legacy `reasons_lib.check_stale` path**, but a full, non-truncated search is required before removal.

### Important shim behavior

Because `reasons_lib/check_stale.py` uses `from reasons.check_stale import *`, it exports all non-underscore names from `reasons.check_stale`, not just the intended public API. In the returned portion of `reasons/check_stale.py`, I do not see an `__all__` declaration. That means the shim may currently expose names like `hashlib`, `re`, `subprocess`, `date`, `Path`, and `Network` in addition to functions such as `check_stale`, `hash_file`, and `resolve_source_path`.

This matters for refactoring:

- Adding an underscore-prefixed helper is safe for shim surface area.
- Adding a new non-underscore helper accidentally becomes part of the shim’s exported surface.
- Renaming or removing an existing public symbol breaks both `reasons.check_stale` and `reasons_lib.check_stale` consumers.
- Adding `__all__` later would intentionally tighten the shim surface, but could break downstream code that accidentally relies on star-imported names.

### What this means for a `reasons.check_stale` refactor

You can refactor the canonical module without touching the shim, as long as the public names and behavior remain compatible. The shim will automatically forward updates because it star-imports the canonical module.

The refactor surface is larger than the first 300 lines show. `reasons/api.py` imports `pin_sources`, `pin_update`, and `pin_lines` from `.check_stale` at lines 2797, 2823, and 2854, but those definitions are not in the returned portion of `reasons/check_stale.py` because only lines 0-300 of 473 were returned. Any full API inventory must read the rest of the file.

The CLI and API layers also mean that changing `check_stale()`’s signature or return contract should be checked against:

- `reasons/api.py:2680` — `def check_stale(...)`
- `reasons/api.py:2699` — `_pg_dispatch(..., "check_stale", ...)`
- `reasons/cli.py:1118-1119` — `cmd_check_stale` and `api.check_stale(...)`

### When can the shim be removed?

`reasons_lib` is a package-wide shim, not a `check_stale`-specific shim. Removing `reasons_lib/check_stale.py` alone is only safe if no one imports `reasons_lib.check_stale`. Removing the whole shim requires a package-level migration.

A safe removal checklist:

1. Run a full repository and downstream audit for `reasons_lib`, including `from reasons_lib.check_stale`, `import reasons_lib`, and dynamic imports such as `importlib.import_module("reasons_lib.check_stale")`.
2. Confirm no external consumers still depend on the shim, ideally after a deprecation release with warnings.
3. Update `pyproject.toml` `[tool.setuptools] packages` to remove `"reasons_lib"`.
4. Update migration docs: `CLAUDE.md:24` says `reasons/` is aliased as `reasons_lib` for backwards compat, and `CLAUDE.md:35` describes `reasons_lib/` as a backwards-compat shim.
5. Keep entry points as canonical: `reasons = "reasons.cli:main"` and `reasons-mcp = "reasons.mcp_server:main"` already point at `reasons`, not `reasons_lib`.
6. Before tightening exports, add compatibility tests that import `reasons_lib.check_stale` and verify the expected public names.

### Gaps in the observations

The observations are not enough to declare the shim removable:

- Only 300 of 473 lines of `reasons/check_stale.py` were returned, so the full public API is not visible.
- `find_check_stale_references` reports 78 matches but only shows a subset.
- `find_legacy_reasons_imports` reports 250 matches but only shows a subset.
- External/downstream consumers are not represented.
- No deprecation policy or non-repo consumer telemetry is shown.

## Topics to Explore

- [file] `reasons/check_stale.py` — Read the remaining 173 lines, especially the definitions of `pin_sources`, `pin_update`, and `pin_lines` imported by `reasons/api.py`.
- [general] `full-reasons-lib-import-audit` — Run a complete search for `reasons_lib`, including dynamic imports, docs, scripts, and downstream consumers; the displayed searches are truncated.
- [function] `reasons/api.py:check_stale` — Understand the API wrapper and `_pg_dispatch` behavior before changing `check_stale`’s signature or return shape.
- [file] `pyproject.toml` — Confirm packaging and entry-point implications of removing `reasons_lib` from `packages = ["reasons", "reasons_lib"]`.
- [general] `shim-compatibility-tests` — Add or inspect tests that import `reasons_lib.check_stale` to catch star-import and public-API regressions before removing the shim.

## Beliefs

- `reasons-lib-check-stale-is-shim` — `reasons_lib/check_stale.py` is a 3-line compatibility shim that star-imports from `reasons.check_stale`.
- `reasons-check-stale-is-canonical` — The implementation lives in `reasons/check_stale.py`, and the shim docstring directs users to `from reasons.check_stale`.
- `internal-check-stale-consumers-use-canonical-path` — Observed internal consumers import `.check_stale` or `reasons.check_stale`, not `reasons_lib.check_stale`.
- `reasons-lib-shim-leaks-star-imported-names` — Because the shim uses `from reasons.check_stale import *` and no `__all__` is observed in the returned canonical code, the shim exposes all non-underscore names from `reasons.check_stale`.
- `reasons-lib-removal-requires-packaging-change` — `pyproject.toml` installs both `reasons` and `reasons_lib`, so removing the shim requires updating packaging as well as imports.
