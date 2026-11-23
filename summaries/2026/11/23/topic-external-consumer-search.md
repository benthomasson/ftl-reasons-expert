---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search outside the repo, such as downstream packages or documentation, for `import reasons_lib` and `from reasons_lib`.

# Searching Outside the Repo for `import reasons_lib` / `from reasons_lib`

## What the search is asking

The question is whether **downstream consumers of `ftl-reasons`** (other packages, notebooks, docs outside this repository) still import the legacy module name `reasons_lib` instead of the canonical `reasons`. This matters because `reasons_lib/` exists *only* as a compatibility shim (see below), and any external importer is a reason to keep it — or a migration target if the shim is to be dropped.

## What the observations actually cover

Every observation in the provided JSON is **internal to the repo**. The explicit search for Python imports (`python_imports_of_reasons_lib`, pattern `(^|\s)(from\s+reasons_lib|import\s+reasons_lib)`) returns **`total_count: 0`**, and the same is true for `reasons_lib_mentions_in_tests` (`total_count: 0`).

What exists inside the repo is definitional, not consumptive:

| File | Line | What it says |
|---|---|---|
| `reasons_lib/__init__.py` | 1–3 | Docstring `"""Backwards-compatibility shim — use import reasons instead."""` followed by `from reasons import *` |
| `pyproject.toml` | 31 | `packages = ["reasons", "reasons_lib"]` — the shim is packaged and distributed |
| `CLAUDE.md` | 24 | `reasons/` "importable as `import reasons` (also aliased as `reasons_lib` for backwards compat)" |
| `CLAUDE.md` | 35 | `reasons_lib/` is a "backwards-compat shim — re-exports from `reasons/`" |
| `CHANGELOG.md` | 136 | References `reasons_lib.api` as the "Python API layer ... for CLI and tool-call consumers" |

So structurally: **the repo defines `reasons_lib` and ships it, but no in-repo code imports it.** By construction, any `import reasons_lib` must therefore live outside the tree.

## Why "outside" is the interesting question

Because `reasons_lib/__init__.py` does `from reasons import *`, it is purely an alias layer. The package metadata in `pyproject.toml:5` names the distribution `ftl-reasons`, so downstream users installing from PyPI will get both `reasons` and `reasons_lib` on their path. The compatibility shim only earns its keep if some **external** consumer — a downstream library, a notebook tutorial, or third-party documentation — actually does `from reasons_lib import ...` or `import reasons_lib`.

The internal data can't answer that. The `CHANGELOG.md:136` reference to `reasons_lib.api` is a historical note about the public surface, not evidence of an active external importer.

## What the observations do not include (and would be needed)

To actually answer the search question, the following external sources would need to be consulted — none of which are present in the JSON:

1. **PyPI metadata for `ftl-reasons`** — reverse dependencies (the `requires_dist` of packages depending on it) and download counts per version. If no package lists `ftl-reasons` as a requirement, the shim has no known consumers.
2. **GitHub code search** (`org:` scoped or global) for `from reasons_lib import`, `import reasons_lib`, and `reasons_lib.api` — this would catch downstream repos, forks, and example notebooks.
3. **External documentation** — website docs, blog posts, or tutorials demonstrating usage of the library. `CHANGELOG.md:136` implies `reasons_lib.api` was once the documented API name, so old docs may still reference it.
4. **Issue trackers / PRs referencing the rename** — the commit that introduced the shim (implied by the "backwards compat" framing in `CLAUDE.md:24,35`) would show whether any external user reported breakage.
5. **The actual old package name** — `reasons_lib` may have been the distribution name in an earlier release before it became `ftl-reasons`. Without that version history, we can't judge how much exposure exists.

## Interpretation

Within the repo, the shim is **dead weight by design** — a hedge against external breakage. The internal evidence (zero imports, zero tests, all mentions being descriptive) is consistent with two very different worlds:

- **World A:** no external consumer ever depended on `reasons_lib`, so the shim could be removed safely.
- **World B:** some external notebook, downstream package, or old tutorial does `from reasons_lib import Node`, and removing it would be a breaking change.

The provided observations cannot distinguish these. Determining which world we are in requires exactly the outside-repo search the query names.

---

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The 3-line shim whose entire purpose is to satisfy external imports; read it to confirm the re-export is the only thing it does
- [file] `pyproject.toml` — Lines 5 and 31 determine the PyPI distribution name (`ftl-reasons`) and confirm `reasons_lib` is shipped; this is what downstream dependency resolution will key off
- [general] `pypi-reverse-dependencies` — Query `ftl-reasons` on PyPI / libraries.io to enumerate downstream packages that require it, which is the only way to find real external importers
- [general] `github-code-search-reasons-lib` — Run `from reasons_lib import` / `import reasons_lib` across GitHub to catch consumers not registered as PyPI dependencies
- [general] `rename-commit-history` — Find the commit that introduced the `reasons_lib` shim and any issue/PR that motivated it; the motivation likely names specific external users

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — `reasons_lib/__init__.py` contains only a docstring and `from reasons import *`, making it a re-export layer with no behavior of its own.
- `reasons-lib-has-no-internal-importers` — Zero in-repo files (including tests) match `import reasons_lib` or `from reasons_lib`; the only mentions are in `CHANGELOG.md` and `CLAUDE.md`.
- `pyproject-ships-reasons-lib-alongside-reasons` — `pyproject.toml:31` lists both `reasons` and `reasons_lib` under `[tool.setuptools] packages`, so the shim is distributed in the `ftl-reasons` wheel.
- `reasons-is-the-canonical-import-name` — `CLAUDE.md:24` and the shim's docstring both designate `import reasons` as the correct import path, with `reasons_lib` retained only for compatibility.
- `external-search-is-required-to-justify-the-shim` — Because no internal code imports `reasons_lib`, the shim's continued existence can only be justified by evidence from outside the repository (PyPI reverse deps, GitHub search, external docs).
