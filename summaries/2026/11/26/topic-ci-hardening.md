---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Whether `fail-fast: false`, `concurrency`, `permissions: contents: read`, `--frozen`, and a `timeout-minutes` should be added.

## Recommendation

Short version: **add `permissions: contents: read` and a job-level `timeout-minutes`; consider `concurrency` if this workflow runs on both push and PR; do not add `fail-fast: false` unless/until a matrix exists; do not add `--frozen` while `uv.lock` is ignored.**

The main caveat is that the observations do **not include the full contents of `.github/workflows/tests.yml`**, and `find_workflow_controls` reports zero matches even for `runs-on`, which is suspicious for a valid Actions workflow. Treat the absence of these controls as “not confirmed present” rather than proven absent. The only visible workflow command is `.github/workflows/tests.yml:26`:

```yaml
run: uv run --extra test pytest tests/ -v
```

| Control | Should it be added? | Reason |
|---|---:|---|
| `fail-fast: false` | **Not yet** | Only meaningful with a `strategy.matrix`. The observations show no `matrix`/`strategy` matches. With one job/one Python version, it does nothing. Add it if the workflow later tests multiple Python versions and you want all matrix jobs to report failures. |
| `concurrency` | **Conditionally yes** | Good for test-only PR/push workflows to cancel superseded runs. The full trigger list is missing, so confirm whether it runs on `push`/`pull_request`. Use a ref-scoped group, and avoid canceling main-branch deploys if those are ever added. |
| `permissions: contents: read` | **Yes** | The workflow appears to only check out code and run tests. `find_workflow_token_usage` found no `GITHUB_TOKEN`, `gh`, release, comment, upload, Pages, or Codecov usage. Least privilege is appropriate. `actions/checkout` needs `contents: read`. |
| `--frozen` | **No, not currently** | `.gitignore:9` ignores `uv.lock`, and there is no evidence of a tracked lockfile. `uv run --frozen` is only useful once the lockfile is committed and treated as the source of truth. Without that, it does not buy reproducible CI. First decide whether to track `uv.lock`; then consider `--frozen` or `--locked`. |
| `timeout-minutes` | **Yes** | This is a pure test workflow using `.github/workflows/tests.yml:26`, and `pyproject.toml:15` defines a relatively small test extra: `pytest`, `pytest-cov`, `pyyaml`. A stuck install or test run should not consume GitHub-hosted runner minutes until the default timeout. Add a conservative job-level value after measuring runtime, e.g. `10` or `15` minutes. |

### Suggested shape

If the full workflow is still minimal, something like this is reasonable:

```yaml
permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tests:
    timeout-minutes: 15
    # ...
```

Do **not** add this yet unless `uv.lock` is committed:

```yaml
run: uv run --frozen --extra test pytest tests/ -v
```

The project currently has `dependencies = []` in `pyproject.toml:14`, test dependencies in `pyproject.toml:15`, and the documented local command uses `uv run --extra test pytest tests/ -v` in `README.md:191` and `CLAUDE.md:86`. That supports using `uv`, but not enforcing `--frozen` until the lockfile policy changes.

### Missing information

To make a final call, inspect the complete `.github/workflows/tests.yml`: its triggers, whether it uses a matrix, whether it checks out code, and whether it already declares permissions/concurrency/timeout. Also confirm whether `uv.lock` is intentionally ignored or should be committed.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Full workflow contents are needed to confirm triggers, matrix usage, checkout/setup actions, and whether any of these controls already exist.
- [file] `.gitignore` — Line 9 ignores `uv.lock`; this directly determines whether `--frozen` or `--locked` is appropriate.
- [file] `pyproject.toml` — Lines 14–15 show empty runtime dependencies and the test extra, which helps size `timeout-minutes` and decide whether a matrix is needed.
- [general] `uv-lockfile-policy` — Decide whether `uv.lock` should be committed; this is the prerequisite for reproducible `uv` CI and `--frozen`.
- [general] `ci-runtime-baseline` — Measure actual pytest runtime to choose a safe `timeout-minutes` and judge the benefit of `concurrency`.

## Beliefs

- `workflow-tests-uses-uv-run-pytest` — `.github/workflows/tests.yml:26` runs `uv run --extra test pytest tests/ -v`.
- `uv-lock-is-ignored` — `.gitignore:9` ignores `uv.lock`, so the repository does not evidence a tracked lockfile.
- `project-has-empty-runtime-dependencies` — `pyproject.toml:14` sets `dependencies = []`, with test dependencies declared separately at `pyproject.toml:15`.
- `workflow-controls-not-evidenced` — The `find_workflow_controls` observation reports zero matches for `fail-fast`, `concurrency`, `permissions`, `timeout-minutes`, `--frozen`, `strategy`, `matrix`, and `runs-on`.
- `workflow-does-not-evidence-token-write-usage` — The `find_workflow_token_usage` observation reports zero matches for `GITHUB_TOKEN`, `GH_TOKEN`, `github-script`, `gh`, comments, releases, Pages, uploads, and Codecov.
