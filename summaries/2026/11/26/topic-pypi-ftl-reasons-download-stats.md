---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Needed to determine whether external users install the package and which versions they use.

The codebase as observed cannot answer whether external users install `ftl-reasons` or which versions they use. It contains distribution intent, not usage data.

### What the observations show

- `pyproject.toml` `[project]` declares `name = "ftl-reasons"` and `version = "0.58.0"`. Its `[project.scripts]` block exposes `reasons = "reasons.cli:main"` and `reasons-mcp = "reasons.mcp_server:main"`. That defines the package artifact, not who installed it.
- `README.md` `## Install` documents `pip install ftl-reasons`, `uv tool install ftl-reasons`, and `uvx ftl-reasons <command>`. That is installation guidance, not evidence of actual installs.
- `CHANGELOG.md` 0.3.0 says “Published to PyPI as `ftl-reasons`”, so publishing was at least intended/happened historically. But the returned 120 changelog lines start at `## [0.18.0]`, while `pyproject.toml` says `0.58.0`; 20 changelog lines were not returned.
- `.github/workflows` contains only `.github/workflows/tests.yml`. The `publish_configuration` search for `pypi|publish|release|twine|upload` has `total_count: 0`. No release/publish automation is observed.
- The `usage_tracking_code` search has 156 matches for a broad pattern, but the observed matches are unrelated to install/version telemetry:
  - `reasons/api.py:1776` — `backtracked_to` is truth-maintenance-system backtracking.
  - `reasons/cli.py:1666-1667` and many similar lines — `reset_cost_tracker` is LLM cost tracking.
  - `reasons/api.py:2311-2313` — `download_network` imports/downloads a Hugging Face network.
- `package_version_symbols` has `total_count: 0`, so no `__version__` runtime symbol was observed. The only observed version source is `pyproject.toml`.

So the repo does not implement telemetry, analytics, version pings, or install reporting. To determine actual external installs and version adoption, you need external data: PyPI metadata for released versions, PyPI download statistics or BigQuery package-download data for version-level counts, GitHub release/tag history, package-manager logs, or a deliberately added opt-in version-check/telemetry mechanism. Without one of those, any claim about external installs or versions is speculation.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — The only observed workflow; check whether publishing is handled elsewhere or missing from the repository.
- [file] `CHANGELOG.md` — Reconcile the returned `0.18.0` top entry with `pyproject.toml`’s `0.58.0`; inspect lines 121-140 and git tags.
- [general] `pypi-download-statistics` — External source needed for install counts and release adoption.
- [general] `version-reporting-design` — If client-side version data is required, design an opt-in version ping/check with privacy considerations.
- [file] `pyproject.toml` — Confirm package name, version, entry points, and optional extras used by external installers.

## Beliefs

- `ftl-reasons-declares-version-0-58-0` — `pyproject.toml` `[project]` declares package name `ftl-reasons` and version `0.58.0`.
- `readme-documents-pip-uv-uvx-install-paths` — `README.md` `## Install` documents `pip install ftl-reasons`, `uv tool install ftl-reasons`, and `uvx ftl-reasons <command>`.
- `no-publish-workflow-observed` — The only observed workflow is `.github/workflows/tests.yml`, and `publish_configuration.total_count` is 0.
- `usage-tracking-search-matches-are-not-telemetry` — Observed matches include `backtracked_to` at `reasons/api.py:1776`, `reset_cost_tracker` at `reasons/cli.py:1666-1667`, and `download_network` at `reasons/api.py:2311-2313`, none of which report installs or versions.
- `changelog-returned-content-starts-at-0-18-0` — The returned 120 lines of `CHANGELOG.md` begin at `## [0.18.0]`, while `pyproject.toml` declares `0.58.0`; the remaining 20 changelog lines were not observed.
