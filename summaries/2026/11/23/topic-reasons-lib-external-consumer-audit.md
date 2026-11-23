---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Grep GitHub/PyPI dependents for `from reasons_lib`/`import reasons_lib` before deleting the shim

# Grepping GitHub/PyPI dependents for `reasons_lib` before deleting the shim

## What the shim is

`reasons_lib/` is a package of backwards-compatibility re-export files that sit next to the real package `reasons/`. Every file is a one-liner:

- `reasons_lib/__init__.py` (3 lines) — `"""Backwards-compatibility shim — use ``import reasons`` instead."""` followed by `from reasons import *`
- `reasons_lib/api.py` (3 lines) — `from reasons.api import *`
- …and 31 more, in total **33 files** (`reasons_lib/api.py`, `ask.py`, `cli.py`, `network.py`, …), each mapping `reasons.X` → `reasons_lib.X`.

It is still declared in packaging:

- `pyproject.toml` line 29 — `[tool.setuptools] packages = ["reasons", "reasons_lib"]`

And still described in docs:

- `CLAUDE.md:24` — "importable as `import reasons` (also aliased as reasons_lib for backwards compat)"
- `CLAUDE.md:35` — "reasons_lib/ # backwards-compat shim — re-exports from reasons/"
- `CHANGELOG.md:136` — historical note that `reasons_lib.api` returns dicts for CLI/tool-call consumers

The public `README.md` never mentions `reasons_lib` — it only documents the `reasons` CLI, so a new user has no reason to touch the shim.

## Why a local grep is not enough

Your local grep result (`find_exact_reasons_lib_imports`, 0 matches) tells you **only that the repo itself no longer uses the shim**. That is not the question the plan is asking. The shim exists *for code outside this repo*: any external project that once did `from reasons_lib import …` or `import reasons_lib.api as api` will keep working solely because `reasons_lib/` is still packaged. Deleting it in a release turns those imports into a hard `ModuleNotFoundError`. The whole point of the instruction "grep GitHub/PyPI dependents" is to measure that external surface **before** you break it.

So you need to query outside the repo. The relevant external identifiers are:

- PyPI distribution name: **`ftl-reasons`** (from `pyproject.toml:7`), version **`0.58.0`** (`:8`)
- Import name: **`reasons_lib`** (and possibly `reasons` — but `reasons` is the canonical name; you care about `reasons_lib` only)

## What "grep GitHub/PyPI dependents" actually looks like

### GitHub code search

Search across all indexed code (not just this repo), because the shim is imported by *other* repos:

- `"from reasons_lib" language:Python`
- `"import reasons_lib" language:Python`
- `"reasons_lib." language:Python` — catches attribute access like `reasons_lib.api.Client(...)` that doesn't start at the top of a line
- `"reasons_lib" path:requirements.txt` / `path:pyproject.toml` / `path:setup.py` / `path:setup.cfg` — catches declared-but-unimported dependencies
- `"reasons_lib" path:*.ipynb` — notebooks are a common consumer of library shims

Filter results by repo, sort by stars, and record the *external* hits — ignore this repo itself and its own forks. Anything non-zero on a public, non-fork repo is a breaking-change signal that needs a deprecation path.

### PyPI / reverse-dependency discovery

PyPI itself has no reverse-dependency API, so use a dependents index:

- **deps.dev** — `https://deps.dev/pypi/ftl-reasons` → "Dependents" tab lists packages whose `requirements` include `ftl-reasons`. This is authoritative for declared dependencies.
- **libraries.io** — `https://libraries.io/pypi/ftl-reasons` → "Dependent Repositories" and "Dependent Packages" lists, backed by GitHub code crawling.
- **pypistats.org** / `google-cloud-bigquery` **PyPI download dataset** — `https://pypistats.org/packages/ftl-reasons` shows whether the package has meaningful downloads at all; a flatline download graph supports the "no consumers" conclusion.
- **static analyzers on the wheel** — after publishing, run a scanner over the sdist/wheel to confirm nothing under `reasons/` still does `import reasons_lib` (the `find_exact_reasons_lib_imports` result already establishes this for the current tree, but it's the packaging-side confirmation).

You should be collecting four numbers before deciding:

1. External GitHub repos importing `reasons_lib` (excluding this repo and forks)
2. External packages on deps.dev declaring `ftl-reasons`
3. `ftl-reasons` weekly downloads (to judge whether the package has any real usage)
4. The last release where `reasons_lib` was a *documented* name — from `CHANGELOG.md` (the note at line 136 refers to a release in the `0.x` line, so the shim has been around since at least that version)

If (1) and (2) are both zero and (3) is negligible, deletion is defensible. If (1) is non-zero — even one repo — you should ship a deprecation release first: keep the shim, but add a `DeprecationWarning` in `reasons_lib/__init__.py` and a note in `CHANGELOG.md`, and delete it one minor version later. (0.58.0 → 0.59.0 shim + warning → 0.60.0 removal.)

## If you decide to delete it, the checklist is wider than `rm -rf reasons_lib/`

The observations show three coupled artifacts that will otherwise be left dangling:

- **Packaging** — `pyproject.toml:29`, `packages = ["reasons", "reasons_lib"]` must become `["reasons"]`. Otherwise setuptools will fail to build the wheel (or, worse, silently drop the shim from the wheel while leaving it in the source tree).
- **Docs** — `CLAUDE.md:24` and `CLAUDE.md:35` describe the shim as alive; change them to "removed in 0.60.0" or delete those lines. `CHANGELOG.md:136` is historical and should stay, but consider appending a "Removed" entry to the current changelog. No `README.md` change needed.
- **`.forge/proposed-entries.json`** — this file contains *per-file entries* for many shim files: `entries/2026/04/23/reasons_lib-__init__.md`, `reasons_lib-api.md`, `reasons_lib-cli.md`, `reasons_lib-compact.md`, `reasons_lib-derive.md`, `reasons_lib-import_agent.md`, several `reasons_lib-network-*.md` entries, `reasons_lib-network.md`, and `reasons_lib-storage.md`. If these entries are hashes of documentation generated from the shim sources, they will become stale when those files disappear. Re-generating that file (or removing the shim entries from it) is part of the deletion, not an afterthought.

One inconsistency worth checking before you commit: the shim list has **33 files**, but the forge list references **`entries/2026/04/23/reasons_lib-storage.md` — yet `reasons_lib/storage.py` does not appear in `find_shim_mentions`**. Either storage was already deleted but its forge entry wasn't, or the file exists and just wasn't in the truncated result set. Verify `ls reasons_lib/storage.py` before removing anything; a missing shim file with a live downstream import is exactly the kind of thing you're trying to avoid.

## What the observations don't give you

The provided JSON contains **zero actual GitHub/PyPI dependent data** — it only shows that the *local* repo no longer imports `reasons_lib`. That is necessary but nowhere near sufficient to justify deletion. Before making the call you still need:

- GitHub code-search hit count and repo list for the four query patterns above
- deps.dev / libraries.io dependent-package counts for `ftl-reasons`
- pypistats weekly downloads for `ftl-reasons`

That missing evidence is the whole deliverable of this step; the rest of the answer is just describing what to feed into it and what to clean up after.

## Topics to Explore

- [file] `pyproject.toml` — The `[tool.setuptools] packages` line is the actual packaging-level gate; deleting `reasons_lib/` without editing this line breaks the wheel build.
- [file] `.forge/proposed-entries.json` — Per-shim-file entry hashes that must be regenerated or pruned when `reasons_lib/` is removed; also flags a `reasons_lib-storage` entry with no matching shim file.
- [general] `deps.dev-reverse-deps` — The authoritative reverse-dependency lookup for `ftl-reasons`; without this data you cannot claim "no PyPI dependents."
- [general] `deprecation-cycle-strategy` — The alternative to hard deletion: ship 0.59.0 with a `DeprecationWarning` in `reasons_lib/__init__.py`, then remove in 0.60.0. Worth exploring before committing to the more disruptive path.
- [file] `CLAUDE.md` — Lines 24 and 35 are the two places in the repo that still document the shim as living API; if they aren't updated, the next agent/engineer will assume it's still present.

## Beliefs

- `reasons-lib-is-a-star-reexport-shim` — Every file under `reasons_lib/` (33 total) is a 1–3 line module whose body is a single `from reasons.X import *` statement preceded by a "Backwards-compatibility shim" docstring.
- `local-grep-is-not-evidence-for-external-deletion` — A zero-match result for `from reasons_lib|import reasons_lib` in this repo proves no *internal* callers exist, and says nothing about external consumers of the published `ftl-reasons` distribution.
- `reasons-lib-is-declared-in-packaging` — `pyproject.toml:29` lists `reasons_lib` in `[tool.setuptools] packages`, so removal requires a packaging edit in addition to deleting the directory.
- `forge-entries-track-shim-files` — `.forge/proposed-entries.json` contains hashed entries keyed by `entries/2026/04/23/reasons_lib-*` paths, coupling the shim files to the forge documentation pipeline.
- `reasons-lib-has-no-readme-exposure` — `README.md` never mentions `reasons_lib`; the only in-repo references to the shim name are `CLAUDE.md:24,35` and the historical `CHANGELOG.md:136`.
- `reasons-lib-storage-mismatch` — `.forge/proposed-entries.json` references `reasons_lib-storage.md` but `reasons_lib/storage.py` is absent from the shim-file list, indicating either a pre-existing stale entry or a truncated search result that must be verified before deletion.
