---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# File: .gitignore

# `.gitignore` — Explanation

## Purpose

This is the repository's Git ignore policy. It is not read by any Python code; it is a declarative filter consumed by Git (and by anything that shells out to Git, e.g. `git check-ignore`, `pre-commit`, and the GitHub Actions workflows under `.github/workflows`).

Its job is to keep three classes of files out of version control:

1. **Standard Python toolchain noise** — bytecode, virtualenvs, build outputs, caches.
2. **Local/ephemeral runtime state produced by the `reasons` CLI** — the `entries/` store, `network.json`, generated `beliefs.md`, proposal drafts, review output, and logs.
3. **AI-agent scratch dirs** — `.sdlc-loop/`, `.code-expert/`, and the `reviews` output directory.

It is a 19-rule, negation-free policy (no `!` re-inclusion lines), which is important: nothing here can be selectively un-ignored without editing the file.

## Key Components

The rules, grouped by intent (line numbers refer to the file as given):

### Python toolchain (lines 1–8)

| Rule | Effect | Notes |
|---|---|---|
| `__pycache__/` | Ignore every `__pycache__` directory at any depth | Trailing `/` = directory-only; no leading `/` = unanchored, so `reasons/__pycache__/` and `tests/__pycache__/` are both caught |
| `*.pyc` | Ignore compiled bytecode | Redundant with the above but catches stray `.pyc` files |
| `*.egg-info/` | Ignore setuptools metadata dirs | Unanchored; matches `reasons.egg-info/` |
| `.venv/` | Ignore the conventional virtualenv | Does **not** match `.env` (see gaps below) |
| `*.db` | Ignore SQLite/local database files anywhere | Matches both the app's runtime DB and any test-created `*.db` |
| `.pytest_cache/` | Ignore pytest's cache | |
| `dist/` | Ignore wheel/sdist output | |
| `build/` | Ignore build staging dir | |

### Package-management policy (line 9)

| Rule | Effect | Notes |
|---|---|---|
| `uv.lock` | **Deliberately not committed** | Unusual — most `uv` projects commit the lock. Here the resolved dependency graph is treated as a local artifact. Consequence: CI must re-resolve from `pyproject.toml` ranges on every run, and reproducibility depends entirely on `pyproject.toml` constraints. |

### Agent / process scaffolding (lines 10–12)

| Rule | Effect | Notes |
|---|---|---|
| `.sdlc-loop/` | Ignore the SDLC-loop working directory | Anchored-looking but actually unanchored; matches at any depth |
| `reviews` | Ignore any file **or directory** named `reviews` | No trailing slash, so it matches both. This is the output dir of the `reasons review*` family (`review.py`, `review_justifications.py`, `review_premises.py`). Because it is unanchored, a future `tests/reviews` fixture would also be silently ignored. |
| `.code-expert/` | Ignore local tool state | |

### Generated data & outputs (lines 13–19)

| Rule | Effect | Notes |
|---|---|---|
| `.coverage` | Ignore coverage.py data file | Note: `htmlcov/` and `coverage.xml` are **not** ignored |
| `network.json` | Ignore the network/citation cache | Produced by `reasons/network.py`; exercised by `tests/test_network.py` and `tests/test_network_cache.py`. Not committed, so it is a pure derived cache. |
| `beliefs.md` | Ignore the generated human-readable belief export | Output of `reasons/export_markdown.py` / `build_wiki.py`. Because the rule is unanchored, **any** file named `beliefs.md` anywhere is ignored. |
| `entries/` | Ignore the on-disk entry store | The primary persistent dataset for the belief system (`storage.py`, `import_beliefs.py`, `derive.py` all read/write it). Ignoring it means the repo ships **code and schema, not data**. |
| `logs/` | Ignore the log directory | Unanchored; matches `logs/` at any depth |
| `*.log` | Ignore individual log files | Partially redundant with `logs/`, but catches `foo.log` outside that dir |
| `proposed-beliefs.md` | Ignore draft output of `propose_update.py` | |
| `proposed-dedup.md` | Ignore dedup proposal output | Pairs with `test_deduplication.py` / `compact.py` |

### What is *intentionally* not ignored

This is as important as what is ignored. `.forge/proposed-entries.json`, `.forge/report.json`, `.claude/skills/`, `costs/2026-11-19T180331-update.json`, `CLAUDE.md`, and `CHANGELOG.md` all remain tracked — i.e. agent configuration, Forge reports, and cost telemetry are treated as first-class repo content.

## Patterns

- **Unanchored, name-based ignores.** Every rule except the trailing-slash forms is unanchored. `beliefs.md`, `network.json`, `proposed-*.md`, `*.db`, `*.log` will match at *any* directory depth. This is convenient but makes name collisions (`tests/reviews`, a fixture named `beliefs.md`) silently uncommittable.
- **Directory-only rules use a trailing slash** (`__pycache__/`, `.venv/`, `.pytest_cache/`, `dist/`, `build/`, `entries/`, `logs/`, `.sdlc-loop/`). This prevents accidentally ignoring a *file* that happens to share the name.
- **No negation (`!`) rules.** There is no way to re-include a file without editing the file itself. If someone needs to commit `entries/fixture.json`, they must use `git add -f`.
- **Ignore-the-outputs-of-the-CLI convention.** Every generator in the `reasons` package that writes to CWD has a corresponding ignore entry: `export_markdown.py` → `beliefs.md`; `propose_update.py` → `proposed-*.md`; `network.py` → `network.json`; `review*.py` → `reviews`; `build_wiki.py` → (wiki output, presumably under `entries/` or `beliefs.md`). This makes the ignore file a *de facto* manifest of side-effecting commands.
- **Code/data separation.** `entries/`, `*.db`, and the markdown exports are all ignored, so the repo is a library/tool repo with no embedded belief corpus.

## Dependencies

- **Imports:** none — the file has no executable content.
- **Consumed by:** Git (`git status`, `git add`, `git check-ignore -v`), the GitHub Actions workflows in `.github/workflows` (a CI job that needs a runtime DB or cache must create it fresh), and any editor tooling that respects `.gitignore`.
- **Implicitly coupled to:** every module that writes to the working tree — `reasons/storage.py` (`entries/`, `*.db`), `reasons/network.py` (`network.json`), `reasons/export_markdown.py` (`beliefs.md`), `reasons/propose_update.py` (`proposed-*.md`), `reasons/review*.py` (`reviews`). Renaming any of those output paths without updating this file will leak generated data into commits.
- **Notably not depended on:** `uv` — because `uv.lock` is ignored, dependency resolution is recomputed on each environment.

## Flow

There is no control flow. Conceptually, for every path Git considers staging:

1. Git walks rules top-to-bottom.
2. The **last matching rule** wins. With no negations, the first match that hits a positive pattern decides exclusion.
3. Directory-only patterns (trailing `/`) apply only if the path is a directory; once a directory is excluded, Git does not descend into it (a performance shortcut that makes `__pycache__/` cheap).
4. Unanchored patterns are matched against every path component suffix, so `foo/bar/network.json` matches `network.json`.

The practical flow for a developer is: run `reasons export-markdown` → `beliefs.md` appears in CWD → `git status` is clean → the artifact is never accidentally committed.

## Invariants

- **Generated artifacts must never be staged.** `entries/`, `network.json`, `beliefs.md`, `proposed-*.md`, `logs/`, `*.log`, `*.db`, `reviews` are all assumed reproducible from commands.
- **No negative patterns exist**, so exclusion is absolute for anything matching a rule; re-inclusion requires `git add -f` or an edit to this file.
- **`uv.lock` is not a source of truth.** Anyone reproducing the environment gets a freshly resolved graph from `pyproject.toml`. If `pyproject.toml` has loose ranges, two developers may run different dependency versions.
- **Unanchored name matches are a coin-flip on intent.** A file literally named `beliefs.md` or a dir literally named `reviews` anywhere in the tree — including under `tests/` or `.claude/` — cannot be committed without `-f`.
- **The repo tracks configuration but not state.** `.forge/`, `.claude/skills/`, and `costs/` are tracked; `entries/` and the markdown exports are not.

### Gaps worth knowing

- `.env` is **not** ignored (only `.venv/` is). Any real `.env` secret would be stageable.
- `.coverage` is ignored but `htmlcov/`, `coverage.xml`, `.ruff_cache/`, `.mypy_cache/`, `*.so`, and `.tox/` are not — adding those linters later will require editing this file.
- `*.log` and `logs/` overlap; neither catches a log written to a path that is neither `logs/` nor `*.log` (e.g. `run.out`).

## Error Handling

None in the code sense. The failure modes are silent:

- **Silent over-ignoring.** Git prints no warning when a file is excluded. A developer who adds `tests/fixtures/beliefs.md` will see it missing from `git status` and may spend time debugging the commit rather than the ignore rule.
- **Silent staleness of `uv.lock` policy.** If someone commits an environment-dependent change and CI resolves differently, the failure surfaces as a distant CI error, not as an ignore-rule error.
- **Recovery is manual**: `git check-ignore -v <path>` to discover which line matched, then `git add -f <path>` or amend the rule. There is no automated linting of this file.

## Topics to Explore

- [file] `pyproject.toml` — Since `uv.lock` is ignored, this file's dependency ranges are the *entire* reproducibility guarantee for the project; check whether ranges are pinned.
- [general] `output-path-contract` — Audit every writer (`storage.py`, `network.py`, `export_markdown.py`, `propose_update.py`, `review*.py`) to confirm its default output path matches a rule here, so generated data cannot leak.
- [file] `.github/workflows` — Verify CI regenerates `entries/`, `network.json`, and the DB from scratch rather than relying on ignored artifacts being present.
- [half] `reasons/cli.py` — The CLI is where all these side-effecting commands are registered; useful for mapping each subcommand to the file it writes.
- [general] `gitignore-anchoring` — Practice `git check-ignore -v` against candidate paths (e.g. `tests/reviews/`, `tests/beliefs.md`) to see how unanchored rules behave before adding fixtures.

## Beliefs

- `gitignore-has-no-negation-rules` — The file contains zero `!` patterns, so no excluded path can be re-included without `git add -f` or editing the file.
- `uv-lock-is-intentionally-uncommitted` — `uv.lock` is listed at line 9, so environment resolution always runs from `pyproject.toml` ranges.
- `entries-directory-is-ignored` — `entries/` (line 16) excludes the belief store, meaning the repo tracks code and schema but no belief data.
- `generated-markdown-outputs-are-ignored` — `beliefs.md`, `proposed-beliefs.md`, and `proposed-dedup.md` are all ignored, covering the outputs of `export_markdown.py` and `propose_update.py`.
- `review-output-directory-is-ignored-unanchored` — `reviews` (line 11) has no trailing slash and no leading slash, so it matches both files and directories named `reviews` at any depth.
- `dotenv-is-not-ignored` — Only `.venv/` is listed; a literal `.env` file is not excluded and would be stageable.
- `forge-and-costs-artifacts-are-tracked` — `.forge/` and `costs/` do not appear in this file, so `.forge/report.json`, `.forge/proposed-entries.json`, and the cost JSON are committed content.
