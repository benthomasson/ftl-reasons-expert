---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Enumerate any PyPI packages or internal services that `import reasons_lib`; this is the sole gate on deletion.

# Enumeration: Who imports `reasons_lib`?

## Short answer

**No PyPI package or internal service in this repository imports `reasons_lib`.**

The dedicated search for actual Python import statements — pattern `(^|\s)(import|from)\s+reasons_lib\b` — returned **0 matches** (`python_import_statements.total_count: 0`). The CI scan for the same token also returned **0 matches** (`ci_references.total_count: 0`).

Therefore, on the "sole gate on deletion" criterion, **nothing inside this repo blocks deletion**. However, see the caveat below about external consumers.

## What the 17 "references" actually are

The broader `all_repo_references` scan found 17 hits, but **none of them are imports**. They break down as:

| Category | Files / lines | What it is |
|---|---|---|
| Documentation prose | `CHANGELOG.md:136`, `CLAUDE.md:24`, `CLAUDE.md:35` | Mentions of `reasons_lib` as a historical API path / backwards-compat shim, not an import |
| Packaging config | `pyproject.toml:31` | `packages = ["reasons", "reasons_lib"]` — declares `reasons_lib` as a *shipped* package, not an importer |
| Forge doc entries | `.forge/proposed-entries.json:2-14` | Entries like `reasons_lib-api.md`, `reasons_lib-cli.md` — doc-generation artifacts mirroring module names, not imports |

The one place `reasons_lib` is actually *used* as code is its own shim:

```python
# reasons_lib/__init__.py (line 1)
from reasons import *  # noqa: F401,F403
```

This imports from `reasons`, not `reasons_lib`, so it doesn't count as an importer of itself.

## Why this matters for deletion

Because `reasons_lib` appears in `pyproject.toml:31`, it **is currently shipped in the `ftl-reasons` wheel**. Anyone who has ever done `pip install ftl-reasons` and then `import reasons_lib` is a consumer — but that consumer class is invisible in this repo. `CLAUDE.md:35` explicitly documents it as a *"backwards-compat shim"*, which implies historical external consumers were the entire reason it exists.

## What's missing from the observations

The instruction is "sole gate on deletion," so the gap is important:

1. **External PyPI dependents** — no data on who has `ftl-reasons` installed and imports `reasons_lib`. Enumerable only via PyPI download stats, published-wheel inspection, or an org-wide code search across other repos.
2. **Private/internal package index usage** — if a mirror or internal index hosts `ftl-reasons`, dependents there aren't covered by these scans.
3. **Downstream service manifests** — the observations only cover this repo; other services in the org aren't scanned.

If the gate is interpreted strictly as "internal imports in this repo," deletion is unblocked. If it includes external consumers, this snapshot alone can't confirm that.

## Topics to Explore

- [file] `pyproject.toml` — Removing `reasons_lib` from `[tool.setuptools] packages` (line 31) is the packaging step required for deletion
- [file] `reasons_lib/__init__.py` — The entire shim is a single line; confirms `reasons_lib` is a pure re-export of `reasons`
- [file] `CLAUDE.md` — Lines 24 and 35 document the "aliased for backwards compat" contract that motivated the shim
- [general] `pypi-dependents-of-ftl-reasons` — External consumers of `reasons_lib` are the real deletion risk, not this repo
- [general] `org-wide-code-search` — A cross-repo search for `import reasons_lib` in sibling services would close the internal-service gap

## Beliefs

- `no-internal-imports-of-reasons-lib` — The repository contains zero Python `import reasons_lib` or `from reasons_lib import ...` statements (regex scan returned 0 matches; CI scan also 0).
- `reasons-lib-is-a-re-export-shim` — `reasons_lib/__init__.py` line 1 is `from reasons import *`, making `reasons_lib` a namespace alias of `reasons`, not an independent module.
- `reasons-lib-is-shipped-as-a-package` — `pyproject.toml:31` lists `reasons_lib` in `[tool.setuptools] packages`, so it is included in the `ftl-reasons` distribution wheel.
- `reasons-lib-is-documented-as-backwards-compat` — `CLAUDE.md:35` labels the directory as a "backwards-compat shim — re-exports from `reasons/`", indicating its only purpose is external API stability.
- `reasons-lib-has-no-ci-coverage` — CI references to the token `reasons_lib` are 0, so deleting the package will not break any CI jobs defined in this repo.
