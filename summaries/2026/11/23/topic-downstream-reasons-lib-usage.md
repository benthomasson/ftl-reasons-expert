---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Check issues, telemetry, or external repos before deleting a published import path.

## Check issues, telemetry, or external repos before deleting a published import path.

### The rule, in this repo's terms

`reasons_lib` is not an internal helper package — it is a **shipped, published import path**. `pyproject.toml` (the `[tool.setuptools]` block, ~line 31) explicitly declares:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

Because the distribution is named `ftl-reasons` at version `0.58.0` (lines 6–7), anything under `reasons_lib/` is installed on every user's machine alongside `reasons/`. Deleting the directory is therefore a **packaging/API change**, not a refactor. The guidance is: before you delete, establish empirically that nobody imports it — and that means looking *outside* the repo (issues, usage telemetry, dependent repos), because a local `grep` cannot see your users.

### What the repository currently tells you

| Evidence | What it shows |
|---|---|
| `reasons_lib/__init__.py:1` | `"""Backwards-compatibility shim — use ``import reasons`` instead."""` |
| `reasons_lib/api.py`, `ask.py`, `build_wiki.py`, `check_stale.py`, `cli.py`, `cluster.py`, `compact.py`, `contradictions.py`, `derive.py`, `export_card.py`, `export_markdown.py`, `hf.py`, `import_agent.py`, … | The shim is **per-submodule**, each doing `from reasons.<module> import *`. The compatibility surface is not just `import reasons_lib` — it is also `from reasons_lib.api import X`, `reasons_lib.cli`, etc. |
| `CLAUDE.md` (Project Structure) | Describes `reasons_lib/` as "backwards-compat shim — re-exports from `reasons/`". |
| `CLAUDE.md` (What This Is) | The package was **renamed**: previously "rms", renamed to "reasons" because a tool-name ablation showed a 5pp accuracy difference. That rename is *why* `reasons_lib` exists — meaning real users may still be on the old spelling. |
| `find_reasons_lib_imports` → 283 matches | The shim mirrors a large module surface. Internal importers are visible (`reasons/mcp_server.py:23` `from reasons import api`; `reasons/pg.py:943` and `:1324` `from reasons.api import ...`) — but these are `reasons`→`reasons` imports, **not** consumers of `reasons_lib`. |

### What the repo does *not* tell you (and why that matters)

The three diagnostic searches came back effectively empty:

- **`find_published_import_path_guidance` → 2 matches**, neither relevant: `CHANGELOG.md:52` ("JSON import path validates outlist references" — a *serialization* path, not a Python import path) and the docstring at `tests/test_import_agent.py:407`. **There is no written deletion/deprecation policy for import paths anywhere in the repo.**
- **`find_telemetry_or_external_repo_references` → 0 matches.** There is no usage reporting, no analytics hook, and no list of downstream repos. You cannot measure `reasons_lib` usage from inside this codebase.
- **`find_compatibility_or_deprecation_notes` → 0 matches**, which is **contradictory**: `reasons_lib/__init__.py:1` literally contains the string "Backwards-compatibility". This result is almost certainly a scoping artifact (search restricted to docs/markdown, or excluding `reasons_lib/`). Treat it as unreliable and re-run before relying on "no deprecation notes exist."

The one adjacent observability dependency is `langfuse = ["langfuse>=2.0"]` in `[project.optional-dependencies]` (~line 22) — but that is *LLM call tracing for the library's users*, not package-usage telemetry about who imports `reasons_lib`. It won't answer the question.

**Conclusion: the repository is blind to its own downstream usage.** The absence of telemetry and external-repo references is precisely why the rule says *check issues and external repos* — those are the only available signals.

### A concrete pre-deletion checklist grounded in these files

1. **Search the issue tracker and changelog for announced deprecation.** `CHANGELOG.md` (the file containing line 52) is where a deprecation notice would live. Look for a release that warns "`reasons_lib` will be removed in X". None is visible in the observations.
2. **Check PyPI download stats and dependents** for `ftl-reasons` — pypistats, libraries.io, or the PyPI "Used by" listing. Downloads ≠ imports, so treat this as directional only.
3. **GitHub code search for `reasons_lib` and `from reasons_lib`** across public repos. This is the direct substitute for the missing external-repo list.
4. **Decide the version boundary.** At `0.58.0` (pre-1.0) SemVer permits breaking changes on a minor bump, but users pin loosely, so the safe path is: add a `DeprecationWarning` to `reasons_lib/__init__.py`, announce in `CHANGELOG.md`, ship at least one release with the warning, *then* delete.
5. **Delete atomically with the build config.** Removing `reasons_lib/` while `packages = ["reasons", "reasons_lib"]` still lists it leaves `pyproject.toml` pointing at a nonexistent package directory, which breaks the build. The `packages` list and the directory must change in the same commit.
6. **Delete the whole shim, not one file.** Because `reasons_lib/__init__.py` only does `from reasons import *`, it re-exports `Justification`, `Node`, `Nogood` (plus incidental names imported in `reasons/__init__.py`). Removing it while leaving `reasons_lib/api.py` etc. yields a package with no `__init__` guarantees — an inconsistent, harder-to-explain state than either full presence or full absence.
7. **Confirm no tests or internal code import `reasons_lib`.** The 283 matches include `reasons/`-internal imports; verify none of them (or anything under `tests/`) resolve through `reasons_lib`, or the deletion breaks your own suite before users are even affected.

### What's missing from these observations

- The actual `CHANGELOG.md` contents around the rename (rms → reasons) and any past deprecation.
- Git history of `reasons_lib/` — when it was added, whether it was ever annotated for removal.
- Any `DeprecationWarning` currently emitted (the shim files shown contain none).
- Whether `tests/` imports `reasons_lib` (the 283-match list is truncated before reaching test files).
- CI configuration / issue templates, which would tell you whether a deprecation gate exists.

## Topics to Explore

- [file] `pyproject.toml` — The `[tool.setuptools] packages` list is the single point that makes `reasons_lib` a published path; any deletion must edit it in the same commit.
- [file] `reasons_lib/__init__.py` — The top-level shim; the natural place to add a `DeprecationWarning` and the smallest unit of the compatibility contract.
- [file] `CHANGELOG.md` — Where a removal announcement would have to appear; verify no past notice already committed the project to keeping the shim.
- [general] `pypi-dependents-and-download-stats` — The only practical substitute for the absent telemetry; answers "is anyone still installing/importing the old path?"
- [general] `github-code-search-for-reasons_lib` — External-repo check that the codebase itself cannot perform; find `from reasons_lib.* import` in downstream projects before breaking them.

## Beliefs

- `reasons-lib-is-a-published-import-path` — `pyproject.toml` lists `reasons_lib` in `[tool.setuptools] packages`, so it ships in the `ftl-reasons` 0.58.0 distribution rather than existing only in the repo.
- `reasons-lib-requires-reasons-package` — Every `reasons_lib/*.py` module re-exports via `from reasons.<module> import *`, so the shim depends on `reasons`, while `reasons/` has no import of `reasons_lib`.
- `reasons-lib-removal-requires-pyproject-edit` — Deleting the `reasons_lib/` directory without removing it from `packages = ["reasons", "reasons_lib"]` leaves the build configuration referencing a nonexistent package.
- `repo-lacks-telemetry-for-import-usage` — A search for `telemetry|external repo|external repositories` returns zero matches, so downstream import-path usage cannot be measured from repository contents.
- `import-path-deletion-guidance-is-absent` — The only "import path" hits are `CHANGELOG.md:52` (JSON serialization validation) and `tests/test_import_agent.py:407` (a docstring); neither documents a deletion or deprecation policy.
- `reasons-lib-shim-is-per-submodule` — The compatibility surface spans `reasons_lib/api.py`, `ask.py`, `cli.py`, `cluster.py`, `compact.py`, `contradictions.py`, `derive.py`, `export_card.py`, `export_markdown.py`, `hf.py`, and `import_agent.py`, so consumers may import submodules even if they never `import reasons_lib` directly.
