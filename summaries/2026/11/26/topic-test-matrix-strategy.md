---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Confirm whether runtime versions, OS targets, or feature flags are parameterized via a matrix, and how that fans out runner cost and coverage.

# CI Parameterization: Matrix, Runtime Versions, OS Targets, Feature Flags

## Short answer

**No matrix parameterization is present.** The repository has exactly one GitHub Actions workflow (`.github/workflows/tests.yml`), and a pattern search for `matrix` across `.github/workflows` returns **0 matches**. Likewise, the combined search for `runs-on|python-version|os:|include:|exclude:` returns **0 matches**, and the feature-flag/env search (`FEATURE_|feature_flag|feature-flag|ENV|env:`) returns **0 matches**.

So, per the captured evidence:

| Axis | Parameterized? | Evidence |
|---|---|---|
| Runtime Python versions | Not in CI | `matrix` 0 matches; `python-version` 0 matches |
| OS targets | Not in CI | `os:` / `runs-on` / `include` / `exclude` 0 matches |
| Feature flags | Not in CI | `FEATURE_…` / `env:` 0 matches |
| Dependency capability groups | Declared, but install-time only | `pyproject.toml` optional-dependencies |

## What the files actually show

### `.github/workflows/tests.yml`
- Exists (`.github/workflows/tests.yml`), and it is the **only** workflow file (`total_entries: 1`).
- The pattern searches returned zero hits for both `matrix` and the runtime/OS selectors. There is no evidence of a `strategy.matrix` block, `include`/`exclude` fan-out, or per-axis `python-version`/`os` keys.

### `pyproject.toml`
- `requires-python = ">=3.10"` (around line 10) — a **single floor**, no upper bound and no enumerated list of supported interpreters.
- `[project.optional-dependencies]` (around lines 13–23) defines capability groups: `test`, `pg`, `test-pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`. These are **install-time extras**, not CI matrix axes — nothing in the observations shows them being exercised as a matrix dimension.
- `[tool.pytest.ini_options]` (around lines 32–33) points tests at `tests`, again with no per-environment branching.

### `costs/2026-11-19T180331-update.json`
This file is **not** CI runner cost. It records an `openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017` model invocation: `calls: 4`, `input_tokens: 49387`, `output_tokens: 41591`, `cost_dollars: 0.0`. Treat this as LLM API cost telemetry, not GitHub Actions minutes; it does not tell us anything about runner fan-out.

## Fan-out of runner cost and coverage

Because there is **no matrix**, the fan-out is effectively the trivial `1 × 1` case:

- **Runner cost:** cost scales with the number of jobs, and no dimension multiplies that number. If the workflow contains a single job (the most likely reading of a minimal `tests.yml`), you pay for one runner per push/PR. Any growth in cost would come from adding jobs directly, not from a matrix.
- **Coverage:** coverage is limited to whatever the workflow's single `runs-on` value and single interpreter resolve to. With no `python-version` matrix, the tested interpreter is whatever the runner image defaults to (or whatever a non-matrix `setup-python` step pins — not observable here). With no `os:` axis, only one OS is exercised. With `requires-python = ">=3.10"` and no matrix, the **3.10 boundary is not actually verified** by CI unless the default interpreter happens to be 3.10.
- **Feature flags:** none are wired into the workflow, so optional-dependency groups (`pg`, `cluster`, `mcp`, `api`, `vertex`, `openai`, `llm`, etc.) are not parameterized across jobs either — a single job can only exercise one install profile.

## Caveat — what's missing

The zero hits for `runs-on` are suspicious for a real GitHub Actions file: every runnable job requires `runs-on`. That means one of the following is true, and the observations can't distinguish them:

1. `tests.yml` is a stub/placeholder with no `jobs:` block.
2. The pattern search is case-sensitive or otherwise missed the key (e.g., `runs-on` appears inline/indented in a way the matcher skipped).
3. The workflow delegates to a reusable workflow or external action elsewhere.

The raw contents of `.github/workflows/tests.yml` were **not returned** — only the pattern-search results. To make a definitive statement about OS/runtime defaults, the file itself needs to be read. The conclusion "no matrix parameterization" is well-supported; the conclusion "one fixed OS/Python" is a strong inference, not directly observed.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Read the actual job definitions to confirm `runs-on`, any `setup-python` pin, and whether the workflow is a stub or a reusable-workflow call
- [general] `ci-matrix-expansion` — Decide whether to add `python: ['3.10','3.11','3.12']` and `os: [ubuntu-latest, windows-latest]` axes, and model the resulting runner-minute multiplier
- [file] `pyproject.toml` — Inspect the optional-dependency groups (`pg`, `cluster`, `mcp`, `api`, `vertex`, `llm`) and consider promoting key ones to matrix install profiles
- [file] `costs/2026-11-19T180331-update.json` — Clarify that this is LLM token cost telemetry, separate from CI runner spend, before using it in any cost model
- [general] `feature-flag-strategy` — Determine where runtime behavior is gated (if anywhere), since no `FEATURE_`/`env:` references exist in workflows

## Beliefs

- `ci-has-no-matrix-parameterization` — A search for `matrix` across `.github/workflows` returns 0 matches with only `tests.yml` present.
- `runtime-floor-is-python-3-10` — `pyproject.toml` sets `requires-python = ">=3.10"` with no upper bound and no enumerated version list.
- `ci-runtime-os-selectors-absent` — No `runs-on`, `python-version`, `os:`, `include:`, or `exclude:` matches exist in the workflow directory.
- `feature-flags-not-declared-in-workflows` — No `FEATURE_`/`feature_flag`/`ENV`/`env:` matches exist in `.github/workflows`.
- `optional-dependencies-are-capability-groups` — `pyproject.toml` defines ten extras (`test`, `pg`, `test-pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`) as install-time profiles, not CI axes.
- `cost-log-is-llm-cost-not-runner-cost` — `costs/2026-11-19T180331-update.json` records model token usage and `cost_dollars`, not GitHub Actions minutes.
