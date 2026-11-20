---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Repo-wide pattern of dual module trees; understand whether this is permanent or scheduled for removal.

## Explanation

The repo currently has **two top-level Python packages**, but they are not two independent implementations. The pattern is:

- **Canonical package:** `reasons`
- **Compatibility shim package:** `reasons_lib`

The evidence is in packaging and module content:

1. **Both packages are explicitly packaged for distribution.**
   - `pyproject.toml:31`:
     ```toml
     packages = ["reasons", "reasons_lib"]
     ```
   - This means `reasons_lib` is not just a development artifact; it is intentionally included in the built distribution.

2. **The canonical package contains the actual data model.**
   - `reasons/__init__.py` defines the core dataclasses:
     - `Justification` at `reasons/__init__.py:7`
     - `Node` at `reasons/__init__.py:27`
     - `Nogood` at `reasons/__init__.py:53`
   - These are real definitions, not aliases.

3. **The `reasons_lib` package is only a backwards-compatibility alias.**
   - `reasons_lib/__init__.py:1` says:
     ```python
     """Backwards-compatibility shim — use ``import reasons`` instead."""
     ```
   - `reasons_lib/__init__.py:2` then does:
     ```python
     from reasons import *  # noqa: F401,F403
     ```
   - So `reasons_lib` re-exports public names from `reasons`; it does not implement a separate module tree. In practice this preserves old import paths like `import reasons_lib` for consumers that used the old package name.

4. **The application entry points use the canonical package.**
   - `pyproject.toml:27-28`:
     ```toml
     reasons = "reasons.cli:main"
     reasons-mcp = "reasons.mcp_server:main"
     ```
   - This reinforces that `reasons` is the primary package and `reasons_lib` exists for compatibility only.

### Is this permanent or scheduled for removal?

The observations indicate **compatibility intent but do not establish a removal schedule**.

Supporting points:

- The shim explicitly calls itself a “Backwards-compatibility shim,” which strongly suggests it exists to avoid breaking old imports after a rename/migration.
- No deprecation warning, removal deadline, changelog entry, TODO, or migration plan appears in the provided observations.
- The reported `deprecation_markers_in_reasons_lib` count is `0`, but that is internally inconsistent because `reasons_lib/__init__.py:1` literally contains the words “compatibility” and “shim.” So that search result is unreliable as presented.
- Similarly, `references_to_reasons_lib` reports `0`, but `pyproject.toml:31` and `reasons_lib/__init__.py` itself contain `reasons_lib`. Therefore those zero counts should not be interpreted as proof that there are no internal references; the search scope or filtering is suspect.
- The project version is `0.58.0` in `pyproject.toml:7`, but version alone does not tell us whether `reasons_lib` will be removed at `1.0` or any other milestone.

So the best interpretation is:

- **`reasons_lib` is currently an intentionally packaged compatibility alias.**
- **It is not the canonical implementation.**
- **There is no evidence in the observations that removal is scheduled.**
- **Whether it is “permanent” cannot be determined without release notes, changelog entries, deprecation policy, or issue-tracker context.**

One important limitation: `from reasons import *` only aliases public top-level names from `reasons/__init__.py`. It does not create compatibility submodules such as `reasons_lib.cli` or `reasons_lib.mcp_server`. If old consumers imported submodules rather than top-level names, this shim would not cover them.

## Topics to Explore

- [file] `CHANGELOG.md` — Check whether removal or deprecation of `reasons_lib` is documented for a future release.
- [file] `pyproject.toml` — Confirm packaging policy, entry points, and whether both packages are intentionally shipped.
- [file] `reasons/__init__.py` — Inspect the canonical public API that `reasons_lib` re-exports.
- [general] `external-consumer-compat` — Determine whether downstream users still import `reasons_lib`, since internal references are inconclusive in the provided observations.
- [general] `release-deprecation-policy` — Look for project policy on backward compatibility, deprecation windows, and major-version removals.

## Beliefs

- `reasons-is-canonical-package` — The real data model and entry points live under `reasons`, as shown by `reasons/__init__.py` and `pyproject.toml:27-28`.
- `reasons-lib-reexports-reasons` — `reasons_lib/__init__.py:2` executes `from reasons import *`, so `reasons_lib` aliases public top-level names from `reasons`.
- `packaging-includes-both-packages` — `pyproject.toml:31` explicitly lists both `reasons` and `reasons_lib` in `[tool.setuptools].packages`.
- `no-removal-schedule-observed` — The provided observations contain no deprecation warning, removal date, changelog entry, or TODO indicating when `reasons_lib` will be removed.
- `reported-zero-reference-counts-are-unreliable` — `references_to_reasons_lib` reports `0`, but `pyproject.toml:31` and `reasons_lib/__init__.py` contain `reasons_lib`, so the search scope needs verification.
