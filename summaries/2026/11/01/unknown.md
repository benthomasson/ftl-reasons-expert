---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: .gitignore

# `.gitignore` — Explanation

## Purpose

This file defines the set of files and directories that Git should exclude from version control in the `ftl-reasons` project. Its sole responsibility is to keep the repository clean by ensuring that generated artifacts, local environment state, sensitive data files, and tool-specific caches never get committed. It acts as the boundary between "source of truth" (tracked code, tests, config) and "ephemeral local state" (build outputs, databases, per-developer caches).

## Key Components

The file contains no classes, functions, or constants — it's a pure declarative list of Git ignore patterns. However, the patterns can be grouped into logical categories:

| Pattern | Category | Rationale |
|---------|----------|-----------|
| `__pycache__/`, `*.pyc`, `*.egg-info/` | Python toolchain | Bytecode caches and packaging metadata |
| `.venv/` | Local environment | Virtual environment is per-machine |
| `*.db` | Local data | SQLite or similar local databases |
| `.pytest_cache/` | Test tooling | pytest's internal state |
| `dist/`, `build/` | Packaging | Artifacts from `uv`/`setuptools` builds |
| `uv.lock` | Lockfile | Treated as disposable (unusual — see Invariants) |
| `.sdlc-loop/`, `.code-expert/` | Internal tooling | Proprietary SDLC / code-review agent state |
| `.coverage` | Coverage | `pytest-cov` output |
| `network.json` | Domain data | The belief network graph (runtime state) |
| `beliefs.md` | Domain data | Exported belief store |
| `entries/` | Domain data | Individual belief entry files |
| `logs/`, `*.log` | Diagnostics | Runtime log output |
| `proposed-beliefs.md`, `proposed-dedup.md` | Working files | Draft outputs from the repair/dedup pipeline |
| `reviews` | Tooling output | Review artifacts (note: no slash, matches a file *or* dir named `reviews`) |

## Patterns

- **No negation patterns (`!`)**: Every entry is a straightforward exclusion. There is no "ignore everything *except* X" logic here.
- **No anchor prefixes (`/`)**: All patterns are relative-to-any-directory, so `__pycache__/` will match at any depth. This is the standard idiom for Python projects.
- **Mix of extensions and directories**: `*.pyc` catches files; `__pycache__/` catches the container. Both are present because some tools write `.pyc` files outside `__pycache__` (e.g., `sys.path` hacks in older setups).
- **Bare names without leading slash**: `reviews`, `network.json`, `beliefs.md`, `entries/` will match at *any* depth in the tree, not just the root. This is a deliberate choice — it means sub-project or test-fixtures producing these names are also excluded.

## Dependencies

- **Depends on**: None. This is a leaf configuration file consumed by Git itself.
- **Depended on by**: The entire repository's `git add` / `git status` / `git diff` behavior. CI workflows (`.github/workflows/`) that run `git archive` or check clean worktrees are implicitly affected. The `pyproject.toml` packaging step must not accidentally include `dist/` or `build/` in its `sdist` (the `build/` and `dist/` entries protect against that).

## Flow

There is no execution flow. Git reads this file (or the equivalent `core.excludesFile`) on every `git status`, `git add`, `git diff`, and `git commit`. The matching algorithm is:

1. Git walks the worktree.
2. For each path, it checks against the pattern list (bottom-up priority: `.gitignore` in the nearest ancestor wins over `~/.gitignore_global`).
3. Patterns ending in `/` match directories only; patterns with `*` are glob-matched; patterns without `/` match at any depth.
4. If a match is found, the path is excluded from the index.

## Invariants

- **`uv.lock` is intentionally untracked.** This is an unusual and consequential choice. Most Python projects *do* commit their lockfile for reproducible builds. Here it's ignored, meaning each developer/CI environment resolves dependencies independently. This implies either:
  - The team accepts non-reproducible resolution (dangerous), **or**
  - There's a separate pinning mechanism (e.g., `pyproject.toml` uses exact version specs), **or**
  - The lockfile is generated in CI and the ignore entry is a mistake/legacy.
  - A maintainer should verify which of these is true before assuming reproducibility.

- **Domain data files (`network.json`, `beliefs.md`, `entries/`) are local state.** This enforces that the belief graph is *reconstructed* (via imports, migrations, or LLM derivation) rather than shipped in the repo. The `reasons/import_beliefs.py` and `reasons/import_agent.py` modules are presumably the canonical way to populate them.

- **`*.db` is a blanket exclusion.** Any developer who needs to check in a *fixture* database for tests must either rename it (e.g., `test_fixture.sqlite`) or add a negation rule `!tests/fixtures/test_fixture.db`. Currently no such exception exists, so test fixtures must avoid the `.db` extension.

- **`reviews` (no trailing slash)** matches both a file *and* a directory named `reviews`. If a legitimate source file called `reviews.py` existed in the repo, it would **not** be affected (the pattern is `reviews`, not `reviews*`). But a directory `reviews/` at any level would be.

## Error Handling

There is no error handling per se, but there are failure modes a maintainer should be aware of:

- **Forced add**: `git add -f <path>` bypasses `.gitignore`. A developer who accidentally runs this on `network.json` (potentially large or sensitive) will commit it. There's no pre-commit hook in this repo (none visible in the tree) to catch that.
- **Pattern drift**: If a new module writes to `artifacts/` or `.cache/`, it won't be covered. The list is not auto-generated; it's manually maintained.
- **`uv.lock` confusion**: A new contributor who expects the lockfile to be committed will be surprised. This is a documentation gap — `README.md` or `CLAUDE.md` should clarify the policy.

---

## Topics to Explore

- [file] `pyproject.toml` — Defines the dependency specs and build backend; understanding whether versions are pinned here explains *why* `uv.lock` can be safely ignored
- [file] `reasons/storage.py` — Implements the persistence layer that writes `network.json` and `entries/`; shows the data contract behind the ignored files
- [file] `reasons/repair_premises.py` — Likely produces `proposed-beliefs.md` and `proposed-dedup.md`; clarifies the pipeline that generates those working files
- [repo] `.github/workflows/` — CI pipelines that consume this repo; reveals whether the build step depends on `uv.lock` or re-resolves, and whether test fixtures avoid the `.db` extension
- [general] `lockfile-policy` — Whether the team has an explicit policy on dependency pinning that compensates for the ignored `uv.lock`, or whether this is an unaddressed reproducibility gap

---

## Beliefs

- `gitignore-excludes-domain-data` — The belief graph (`network.json`), exported beliefs (`beliefs.md`), and individual entries (`entries/`) are runtime state that must be reconstructed via import pipelines, not stored in the repository
- `gitignore-excludes-uv-lock` — `uv.lock` is deliberately untracked, meaning dependency resolution is not locked to a single set of transitive versions across environments
- `gitignore-uses-unanchored-patterns` — All directory patterns (`entries/`, `logs/`, `reviews`) are relative-to-any-depth (no leading `/`), so they suppress matches in nested subdirectories as well as the project root
- `gitignore-covers-internal-tooling` — The presence of `.sdlc-loop/` and `.code-expert/` entries indicates the team uses internal SDLC and code-review agents whose local state directories must not pollute the repo
- `gitignore-has-no-negation-rules` — There are no `!` (re-include) patterns in this file, meaning any file matching an ignore pattern is unconditionally excluded with no exceptions for test fixtures or documentation
