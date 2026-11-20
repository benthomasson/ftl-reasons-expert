---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How widely `reasons_lib` is still referenced across the repo, and what a removal plan (deprecation warning → grep → delete) would look like.

# How widely `reasons_lib` is still referenced

The repo is **not widely dependent on `reasons_lib` at the code level**, but it is still present as a shipped compatibility package and in docs/config metadata.

| Area | Matches | Where | Assessment |
|---|---:|---|---|
| Python source | **0** | none | No Python file currently imports or names `reasons_lib` |
| Packaging | **1** | `pyproject.toml:31` | `"reasons_lib"` is explicitly included in `packages` |
| Docs | **3** | `CHANGELOG.md:136`, `CLAUDE.md:24`, `CLAUDE.md:35` | Mostly compatibility/history; `CLAUDE.md` needs updating on removal |
| Config | **13** | `.forge/proposed-entries.json:2-14` | Generated/proposed doc-entry names; verify whether they are stale/generated |
| Workflows/YAML | **0** | none | No CI/workflow dependency found |

Total observed references: **17**, but only **1 packaging reference** and **0 Python source references** affect runtime/import behavior.

Key details:

- `reasons_lib/__init__.py` is only a 3-line shim:
  ```python
  """Backwards-compatibility shim — use ``import reasons`` instead."""
  from reasons import *  # noqa: F401,F403
  ```
  It does not currently emit any warning. The observation for `DeprecationWarning|warnings.warn|deprecated` in `reasons_lib` returns **0 matches**.

- `pyproject.toml:31` still ships it:
  ```toml
  packages = ["reasons", "reasons_lib"]
  ```
  So even though internal Python code does not import it, downstream consumers can still install it. That is the main reason to deprecate before deletion.

- `CLAUDE.md:24` says `reasons` is “also aliased as reasons_lib for backwards compat”, and `CLAUDE.md:35` calls `reasons_lib/` a “backwards-compat shim”. Both are actionable docs references.

- `CHANGELOG.md:136` mentions `reasons_lib.api` historically. This should usually be treated as history, not as an active dependency, unless the removal policy requires rewriting old changelog entries.

- `.forge/proposed-entries.json:2-14` contains 13 filenames such as `reasons_lib-api.md`, `reasons_lib-cli.md`, `reasons_lib-network.md`, etc. These look like generated/proposed documentation entries. Since the only shown `reasons_lib` code is a 3-line shim, verify whether these entries are stale or generated before editing them manually.

**Bottom line:** internal source usage is effectively zero. The package is mainly kept alive by packaging metadata plus docs/config. Removal risk is external consumers, not repo-internal imports.

## Removal plan: deprecation warning → grep → delete

### 1. Add a deprecation warning first

Edit `reasons_lib/__init__.py` while keeping the re-export behavior:

```python
"""Deprecated backwards-compatibility shim — use ``import reasons`` instead."""
import warnings

warnings.warn(
    "`reasons_lib` is deprecated; import `reasons` instead.",
    DeprecationWarning,
    stacklevel=2,
)

from reasons import *  # noqa: F401,F403
```

Add a test such as:

```python
import importlib
import pytest

def test_reasons_lib_deprecation_warning():
    with pytest.warns(DeprecationWarning):
        importlib.import_module("reasons_lib")
```

This is the only expected new Python reference during the deprecation window. Release that version and let it sit for your normal deprecation period. Since internal Python references are already **0**, the main purpose is warning external consumers.

### 2. Grep/audit before deletion

Run repo-wide checks:

```bash
rg -n --hidden -g '!.git' '\breasons_lib\b'
rg -n --glob '*.py' 'import\s+reasons_lib|from\s+reasons_lib|reasons_lib\.'
git grep -n reasons_lib
```

Expected triage before deletion:

- **Python**: no matches aside from any temporary deprecation test.
- **Packaging**: `pyproject.toml:31` — remove `"reasons_lib"` from `packages` during deletion.
- **Docs**: `CLAUDE.md:24` and `CLAUDE.md:35` — update to stop describing the alias/shim.
- **Config**: `.forge/proposed-entries.json:2-14` — regenerate or remove those entries if they are generated artifacts.
- **CHANGELOG**: `CHANGELOG.md:136` — normally leave as historical record; add a new changelog entry for the removal instead of rewriting old history.

### 3. Delete and clean packaging/docs/config

After the deprecation window:

1. Delete the `reasons_lib/` directory.
2. Remove `"reasons_lib"` from `pyproject.toml:31`.
3. Update `CLAUDE.md:24` and `CLAUDE.md:35`.
4. Regenerate or clean `.forge/proposed-entries.json:2-14`.
5. Add a new changelog entry for the removal; leave `CHANGELOG.md:136` as history unless your policy says otherwise.
6. Remove the temporary deprecation test.

### 4. Validate

Run the test suite, build the package, and verify the wheel no longer contains `reasons_lib`:

```bash
pytest
python -m build
unzip -l dist/*.whl | grep reasons_lib   # should be empty
python -c 'import reasons'               # should work
python -c 'import reasons_lib'           # should fail with ModuleNotFoundError
```

If external usage telemetry is unavailable, extend the deprecation window before deleting, because repo grep alone only proves internal usage is zero.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The 3-line shim where the `DeprecationWarning` should be added and later deleted.
- [file] `pyproject.toml` — Line 31 explicitly packages `reasons_lib`; this controls whether the shim is distributed to consumers.
- [file] `.forge/proposed-entries.json` — Lines 2-14 contain 13 `reasons_lib`-named entries; determine whether these are generated or stale before cleanup.
- [general] `external-consumer-usage` — Internal Python references are zero, so downstream usage is the primary risk for removal.
- [file] `CLAUDE.md` — Lines 24 and 35 document the alias/shim and must be updated in the removal diff.

## Beliefs

- `reasons-lib-has-no-python-source-references` — A repo grep for `reasons_lib` in Python returns 0 matches, so no Python source file currently imports or names the package.
- `reasons-lib-is-explicitly-packaged` — `pyproject.toml:31` includes `"reasons_lib"` in the `packages` list, so it is part of the distributed package set.
- `reasons-lib-reexports-reasons` — `reasons_lib/__init__.py` is a 3-line shim whose only code is `from reasons import *`, re-exporting the `reasons` package.
- `reasons-lib-emits-no-deprecation-warning` — There are no `DeprecationWarning`, `warnings.warn`, or `deprecated` matches in `reasons_lib`, so importing it currently gives no deprecation signal.
- `reasons-lib-references-concentrate-in-forge-config` — 13 of the 17 observed `reasons_lib` matches are in `.forge/proposed-entries.json`, not source code.
