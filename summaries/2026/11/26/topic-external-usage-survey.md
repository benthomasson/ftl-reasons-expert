---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: PyPI stats and GitHub code search for `from reasons_lib` / `import reasons_lib` would replace the "unknown external consumers" caveat with concrete data before committing to 0.60.0.

## What the observations establish

The `reasons_lib` package is a **3-line compatibility shim**:

- `reasons_lib/__init__.py:1` — docstring: `"""Backwards-compatibility shim — use ``import reasons`` instead."""`
- `reasons_lib/__init__.py:2` — `from reasons import *  # noqa: F401,F403`

It is shipped as a first-class installable package. `pyproject.toml` line 30-31:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

So `pip install ftl-reasons` installs `reasons_lib` on disk alongside `reasons`. It costs no build configuration, no runtime code, no tests. But it is also *unused inside the repo*:

- `find_reasons_lib_imports_in_code` searched `(from|import) reasons_lib` and returned **0 matches**. No test, module, CLI, or script in the repo imports it.
- The only references to `reasons_lib` anywhere are docs: `CLAUDE.md:24`, `CLAUDE.md:35`, and `CHANGELOG.md:136`.

In other words: **the shim exists purely to serve consumers outside the repo**. Its entire raison d'être is the population the caveat says is "unknown."

## Where the caveat actually comes from

- `locate_external_consumers_caveat` searched `external consumers|unknown external` and found **0 matches**. The caveat is *not written down* in the repo yet — it exists only as a mental blocker in the release conversation.
- `find_060_release_notes` searched for `0.60.0` and found **0 matches**. There is no release plan committed, no changelog entry, no `CHANGELOG` section announcing what 0.60.0 will do.
- Current version is `0.58.0` (`pyproject.toml:7`), and `0.60.0` would skip `0.59.0` — i.e., it is envisioned as a *milestone / breaking* release, not a routine patch.

So the logic chain for the caveat is:

1. Internal usage of `reasons_lib` = 0 (proven by `find_reasons_lib_imports_in_code`).
2. Therefore the shim's purpose is external backwards-compatibility.
3. We have no telemetry on external imports (no import hook, no `PyPI` name-by-submodule breakdown, no issue reports).
4. Therefore we can't say whether removing or reshaping it in 0.60.0 breaks anyone.
5. Therefore: "unknown external consumers."

The proposed remedy — PyPI stats + GitHub code search — attacks step 3 by replacing *absence of data* with *two independent, quantified data sources*.

## What PyPI stats would tell you

Querying `pypistats.org` (or the BigQuery `pypi.file_downloads` dataset) for `ftl-reasons` yields:

| Signal | What it answers |
|---|---|
| Total downloads per day/month | Order-of-magnitude size of the install base |
| Per-version downloads (`0.58.0`, `0.57.x`, …) | Whether *current* versions are being installed, or whether stale installs dominate |
| Ratio of downloads to GitHub stars / watchers | Whether usage is real or incidental (CI matrices, mirrors) |

This does **not** tell you how many of those downloads import `reasons_lib` specifically — PyPI only publishes per-project, per-file stats, not per-module. But it *does* bound the risk: if `ftl-reasons` gets 12 downloads/month total, the ceiling on `reasons_lib` breakage is trivially small regardless of what the shim does.

## What GitHub code search would tell you

Searching public GitHub for `from reasons_lib` and `import reasons_lib` gives a literal **count and list of public importers**. That is the only data source that can produce an actual "N repos will break" number.

Important limitations to note in the write-up:

- **Private repos are invisible.** A sizable fraction of real consumers may be private.
- **Dynamic imports** (`__import__("reasons_lib")`, `importlib.import_module("reasons_lib")`, plugin loaders) won't match the literal patterns.
- **Vendored / forked shims** copied into other projects won't match either.
- **Index lag**: GitHub code search trails by hours-to-days.
- **`reasons_lib.api` is already broken.** `CHANGELOG.md:136` references a historical `reasons_lib.api` module ("Python API layer (`reasons_lib.api`) returning dicts…"). The current shim (`from reasons import *`) only re-exports names from `reasons/__init__.py`, which — per `inspect_reasons_init` — is just `Justification`, `Node`, `Nogood` and nothing named `api`. So *anyone* still doing `from reasons_lib.api import …` is already broken and does **not** depend on the current shim. That significantly narrows the population the shim actually protects.

## How the two data sources compose

PyPI stats and GitHub search are complementary, not redundant:

- **PyPI** answers *"how many people installed the artifact at all?"* — a denominator / risk ceiling.
- **GitHub** answers *"how many public repos literally import the legacy name?"* — a numerator / concrete blast radius.

Together they let you write a sentence in the 0.60.0 release notes like:

> `reasons_lib` has X,XXX downloads across 0.58.x, and Y public GitHub repositories import it directly as of DATE. The shim remains in 0.60.0 (or is removed, or keeps a deprecation warning) accordingly.

That sentence replaces the current binary "unknown external consumers" with a **bounded, dated, and defensible statement**.

## What the observations are missing

A few gaps you should flag before running the analysis:

1. **No statement of what 0.60.0 changes.** `find_060_release_notes` returned 0 matches, so we don't know if 0.60.0 will (a) drop `reasons_lib`, (b) keep it but deprecate via `warnings.warn`, (c) keep it untouched. The data is only *preparation*; the actual policy decision is undocumented.
2. **No README evidence of `reasons_lib` being recommended.** All 199 lines of `README.md` advocate `import reasons` / `pip install ftl-reasons`; `reasons_lib` never appears. That means the shim is *undocumented in user-facing docs*, which biases against its real external usage — most users would copy the README and never learn `reasons_lib` exists.
3. **`pyproject.toml:30-31` still ships it.** If 0.60.0 keeps the shim, no change is required; if it removes it, `packages = ["reasons_lib"]` must be dropped and a changelog entry added.
4. **No deprecation-warning test.** There is no `tests/` file asserting `import reasons_lib` emits a `DeprecationWarning`, so adding one is a cheap way to warm external consumers in 0.59.x before removing in 0.60.x.

## Recommendation as you write the release decision

Run both queries, then classify:

| PyPI downloads/mo | GitHub importers | Recommended 0.60.0 action |
|---|---|---|
| ~0–50 | 0 | Remove `reasons_lib` from `pyproject.toml:31` and delete the dir |
| ~50–500 | 0–3 | Keep shim, add `DeprecationWarning` in 0.60, plan removal in 1.0 |
| >500 | >3 | Keep shim indefinitely; document it at the top of `README.md` |

Either way, the output is a **dated one-line fact in `CHANGELOG.md`** replacing the verbal caveat — and that is the entire point of the exercise.

## Topics to Explore

- [file] `CHANGELOG.md` — Line 136 references a historical `reasons_lib.api` module that no longer exists; worth understanding what the old public surface was before you decide the shim's fate
- [file] `reasons/__init__.py` — The full public API re-exported by the shim is only `Justification`, `Node`, `Nogood`; confirms what `reasons_lib` actually exposes today
- [general] `pypi-download-stats-query` — Design a concrete query against `pypistats.org` or BigQuery `pypi.file_downloads` for `ftl-reasons` filtered to `0.58.x`
- [general] `github-code-search-query` — Formulate the exact search strings (`from reasons_lib`, `import reasons_lib`, `reasons_lib.api`) and note index lag / private-repo blind spot
- [general] `deprecation-warning-test` — Sketch a `pytest` test that asserts `import reasons_lib` emits a `DeprecationWarning`, as the lowest-cost mitigation if the data says "keep the shim for one more minor"

## Beliefs

- `reasons-lib-is-a-pure-backcompat-shim` — `reasons_lib/__init__.py` contains only a docstring and `from reasons import *`; it has no independent logic.
- `no-internal-code-imports-reasons-lib` — `find_reasons_lib_imports_in_code` returned 0 matches for `(from|import) reasons_lib` across the entire repo.
- `reasons-lib-is-shipped-as-a-package` — `pyproject.toml:31` lists `reasons_lib` in `[tool.setuptools] packages`, so it is installed by every `pip install ftl-reasons`.
- `reasons-lib-api-submodule-is-already-absent` — `CHANGELOG.md:136` mentions `reasons_lib.api`, but `reasons/__init__.py` exports no `api` symbol, so historical `from reasons_lib.api import …` consumers are already broken.
- `unknown-external-consumers-caveat-is-unwritten` — `locate_external_consumers_caveat` found 0 matches for "external consumers|unknown external", meaning the caveat currently lives only in conversation, not in repo files.
- `0-60-0-has-no-committed-release-notes` — `find_060_release_notes` returned 0 matches for `0.60.0`; the release plan is undocumented, so the purpose of the data-gathering exercise is to inform an unwritten decision.
