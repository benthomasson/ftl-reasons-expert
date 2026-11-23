---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Reconcile `pyproject.toml` version `0.58.0` with `CHANGELOG.md` topping out at `0.18.0`, and document the `reasons_lib` rename retroactively.

## Version reconciliation

The observed version declarations do not currently agree:

- `pyproject.toml` line 6 declares `version = "0.58.0"` under `[project]`, with distribution name `ftl-reasons` on line 5.
- `CHANGELOG.md` top entry is `## [0.18.0] - 2026-04-25` at line 6, and no later heading appears in the returned history.
- No code-level `__version__` assignment was found, so there is no independent Python package version to cross-check.

This does not necessarily mean `pyproject.toml` is wrong. The changelog is already not a one-to-one release ledger: the `0.10.0` section folds in features labeled `(0.9.0)` and `(0.8.0)` without separate headings. So the most defensible interpretation is that `CHANGELOG.md` stopped being maintained after `0.18.0`, while `pyproject.toml` continued to carry the packaging version.

To reconcile it properly:

1. Treat `pyproject.toml` line 6 as packaging truth until git tags or PyPI metadata prove otherwise.
2. Check git tags and release history for versions `0.19.0` through `0.58.0`.
3. If those releases exist, backfill `CHANGELOG.md` above `0.18.0`, at minimum adding a `## [0.58.0]` section and ideally summarizing `0.19.0`–`0.58.0`.
4. If those releases do not exist, correct `pyproject.toml` to `0.18.0` or the next intended version instead of inventing a `0.58.0` changelog entry.
5. Check `reasons/cli.py:main` to confirm how the `--version` flag derives its value, since no `__version__` was observed in `reasons/__init__.py`.

What is missing from these observations: git tags, PyPI release history, full `git log -- pyproject.toml CHANGELOG.md`, and the implementation of the CLI `--version` flag. Those are required to know whether `0.58.0` is real or stale.

## Retroactive `reasons_lib` rename documentation

The rename history is partly documented but not stated cleanly:

- `CHANGELOG.md` line 136, under `0.1.0`, says the Python API layer was `reasons_lib.api`.
- The `0.3.0` `### Changed` section says the project was renamed from `rms` to `reasons` for CLI, library, and package.
- `CLAUDE.md` line 24 says `reasons/` is importable as `import reasons` and “also aliased as `reasons_lib` for backwards compat.”
- `CLAUDE.md` line 35 describes `reasons_lib/` as a backwards-compat shim that re-exports from `reasons/`.
- `reasons_lib/__init__.py` line 1 is exactly `from reasons import *  # noqa: F401,F403`.
- `pyproject.toml` line 30 still packages both: `packages = ["reasons", "reasons_lib"]`.
- `pyproject.toml` scripts point at `reasons.cli:main` and `reasons.mcp_server:main`, not `reasons_lib`.
- The Python-reference search for `reasons_lib` found `0` matches, so no internal Python code depends on the shim.

A retroactive changelog note should be added under `0.3.0`, or split between `0.1.0` and `0.3.0`, with wording like:

```markdown
- **Retroactive note:** The pre-0.3.0 Python API was importable as `reasons_lib.api` (see 0.1.0). The 0.3.0 rename made `reasons` the canonical import package. `reasons_lib` remains as a backwards-compatibility shim (`reasons_lib/__init__.py` re-exports from `reasons`) and is still included in `pyproject.toml`'s package list so existing `import reasons_lib` users do not break.
```

One important caveat: `reasons_lib/__init__.py` only re-exports top-level names from `reasons`. That alone does not prove `import reasons_lib.api` still works. Verify whether there is a `reasons_lib/api.py` or an explicit module alias before claiming full submodule compatibility.

Also, `.forge/proposed-entries.json` still lists many `entries/2026/04/23/reasons_lib-*.md` paths. That generated metadata may also need a rename or migration if `reasons` is now the canonical package name.

## Topics to Explore

- [file] `pyproject.toml` — Confirm the version source at line 6 and the dual package list at line 30.
- [file] `CHANGELOG.md` — Inspect the `0.1.0` and `0.3.0` sections; line 136 contains the old `reasons_lib.api` reference.
- [general] `git-release-tags` — Compare git tags and PyPI releases to determine whether `0.58.0` is a real release or stale metadata.
- [function] `reasons/cli.py:main` — Check how the `--version` flag derives its value, since no `__version__` was observed.
- [general] `reasons-lib-submodule-compatibility` — Verify whether `import reasons_lib.api` resolves through a real shim file, not just `from reasons import *`.

## Beliefs

- `pyproject-declares-version-0-58-0` — `pyproject.toml` line 6 declares `version = "0.58.0"` for distribution `ftl-reasons`.
- `changelog-top-entry-is-0-18-0` — `CHANGELOG.md` line 6 is `## [0.18.0] - 2026-04-25`, with no later version heading observed.
- `reasons-lib-is-a-compatibility-shim` — `reasons_lib/__init__.py` line 1 re-exports from `reasons`, and `CLAUDE.md` lines 24/35 describe it as a backwards-compat shim.
- `canonical-package-is-reasons` — `pyproject.toml` scripts call `reasons.cli:main` and `reasons.mcp_server:main`, and no Python references to `reasons_lib` were found.
- `version-source-is-pyproject-not-dunder-version` — No `__version__` assignment was found, so the observed version declaration is `[project].version` in `pyproject.toml`.
