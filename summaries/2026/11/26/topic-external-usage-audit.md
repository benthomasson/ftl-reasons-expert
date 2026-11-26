---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Search downstream repositories and package consumers for `import reasons_lib` and `from reasons_lib.api` to quantify breakage risk.

## Summary

The observations show a **repo-local search**, not an actual downstream/package-consumer search. Within this checkout, the exact import search for `from reasons_lib|import reasons_lib` returned **0 matches**, so no first-party Python source in this repository directly imports the shim.

However, this does **not** quantify external breakage risk. The package is configured to ship `reasons_lib` (`pyproject.toml:31`), and `reasons_lib` exists as a backwards-compatibility shim:

- `reasons_lib/__init__.py:2` — `from reasons import *`
- `reasons_lib/api.py:2` — `from reasons.api import *`

So the breakage surface is real for any external consumer that imports `reasons_lib` or `reasons_lib.api`, but the provided observations do not include any external repository scan, PyPI dependents data, or installed-environment import scan.

Verdict:

- **In-repo breakage risk:** low; exact import search found 0 matches.
- **Downstream/package-consumer breakage risk:** unknown/unquantified; external consumer data is missing.

## What the evidence shows

The exact search `from reasons_lib|import reasons_lib` returned `total_count: 0`. Because `from reasons_lib` is a prefix of `from reasons_lib.api`, this pattern would catch direct static imports of both the root package and `reasons_lib.api`. It would still miss dynamic imports such as `importlib.import_module("reasons_lib.api")`.

The broader `reasons_lib` mention search found 17 matches, but they are documentation, packaging, and generated-entry metadata rather than consumer imports:

- `CHANGELOG.md:136` describes `reasons_lib.api` as the Python API layer returning dicts for CLI and tool-call consumers.
- `CLAUDE.md:24` says the package is importable as `reasons` and aliased as `reasons_lib` for backwards compatibility.
- `CLAUDE.md:35` calls `reasons_lib/` a backwards-compat shim re-exporting from `reasons/`.
- `pyproject.toml:31` declares `packages = ["reasons", "reasons_lib"]`.
- `.forge/proposed-entries.json:2-14` references proposed documentation entries such as `reasons_lib-api.md`; these are not import sites.

The downstream/breakage search was also not useful for consumer risk. It returned 386 matches, but they are mostly TMS-domain references to “dependents” in the truth-maintenance graph, e.g.:

- `README.md:11` — retraction cascade to dependents.
- `reasons/__init__.py:38` — `dependents: set[str]` reverse index.
- `.claude/skills/reasons/SKILL.md:35` — retraction cascade description.

These are false positives for “downstream package consumers.” There is only one visible CI workflow, `.github/workflows/tests.yml`, and the observations do not show any downstream compatibility or package-consumer verification job.

## Breakage-risk interpretation

If `reasons_lib` were removed or changed, any external code doing these forms would break at import time:

```python
import reasons_lib
from reasons_lib import something
from reasons_lib.api import something
import reasons_lib.api
```

The shim files are minimal star re-exports, so they depend on `reasons` and `reasons.api` continuing to export the expected public names. If a symbol is removed from `reasons` or omitted from its star export, `reasons_lib` consumers may fail at attribute access even if the shim files still exist.

The README’s Usage section shows CLI commands rather than Python imports, and `pyproject.toml:6-7` defines the distribution as `ftl-reasons` version `0.58.0`. That suggests the documented primary interface is the CLI, but `CHANGELOG.md:136` explicitly mentions `reasons_lib.api` for tool-call consumers, so external Python usage cannot be ruled out.

To actually quantify risk, we would need:

- an organization-wide or public code search for `reasons_lib`, `reasons_lib.api`, and dynamic import variants;
- PyPI dependents or package download/usage data for `ftl-reasons`;
- an installed-package smoke test for `import reasons_lib` and `from reasons_lib.api import ...`;
- a comparison of exported names between `reasons`/`reasons.api` and their `reasons_lib` shims.

## Topics to Explore

- [file] `pyproject.toml` — Confirms `reasons_lib` is shipped in the `ftl-reasons` distribution and defines the package version/name used for PyPI consumer searches.
- [file] `reasons_lib/api.py` — The exact compatibility surface for `from reasons_lib.api` imports and the file whose removal/change would cause import-time breakage.
- [file] `reasons_lib/__init__.py` — The root compatibility shim for `import reasons_lib`; its star re-export behavior determines which symbols consumers can access.
- [general] `external-code-search` — Needed to search downstream repositories and public/private code for `import reasons_lib` and `from reasons_lib.api`.
- [general] `pypi-dependents-and-downloads` — Needed to estimate installed consumer exposure for the `ftl-reasons` distribution.

## Beliefs

- `reasons-lib-is-star-reexport-shim` — `reasons_lib/__init__.py:2` and `reasons_lib/api.py:2` are one-line star re-exports from `reasons` and `reasons.api`.
- `reasons-lib-is-packaged` — `pyproject.toml:31` includes both `reasons` and `reasons_lib` in the setuptools package list.
- `reasons-lib-exact-imports-zero-in-repo` — The repo-local exact search for `from reasons_lib|import reasons_lib` returned 0 matches.
- `package-name-is-ftl-reasons` — `pyproject.toml:6` sets the distribution name to `ftl-reasons`, with version `0.58.0` at `pyproject.toml:7`.
- `changelog-identifies-reasons-lib-api-consumer-surface` — `CHANGELOG.md:136` describes `reasons_lib.api` as the Python API layer for CLI and tool-call consumers.
