---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Decide where `MIND_API_KEY` should live for local development, CI, and production.

## Recommendation: keep `MIND_API_KEY` in the runtime environment, with environment-specific secret sources

The code already defines the correct shape: `MIND_API_KEY` is a runtime environment variable, optionally overridden by an explicit argument.

| Environment | Where `MIND_API_KEY` should live | Why |
|---|---|---|
| **Local development** | In the developer’s shell/OS secret store, e.g. `export MIND_API_KEY=...`, or via an untracked tool like `direnv`/1Password CLI. A one-off `--api-key` is acceptable for manual commands. Do **not** commit it. | `reasons/mind_service.py:_resolve_config` reads `api_key = api_key or os.environ.get("MIND_API_KEY", "")` at line 17. The CLI also exposes `--api-key` and documents `MIND_API_KEY` as the default env source at `reasons/cli.py:3481` and `reasons/cli.py:3489`. |
| **CI** | In the CI secret store, e.g. GitHub Actions repository/environment secret, injected as `MIND_API_KEY` only into jobs that actually need a live Mind service. Unit tests do not need a real key. | `tests/test_mind_service.py` sets and clears `MIND_API_KEY` with `monkeypatch` at lines 48, 62, 68, 184, and 241. Those tests are self-contained. The only observed workflow is `.github/workflows/tests.yml`, but its contents were not provided, so the exact CI wiring is unknown. |
| **Production** | In a production secret manager, injected into the process environment as `MIND_API_KEY` at deploy/runtime. Do not bake it into images, source, config files in the repo, or command-line arguments. | `_resolve_config` is the single observed resolution point and only reads `os.environ` at `reasons/mind_service.py:15-17`. If missing, it raises a `RuntimeError` listing `api-key (--api-key or MIND_API_KEY)` at line 25. Auth failures also explicitly tell operators to check `MIND_API_KEY` at lines 42 and 79. |

### Important implementation detail

`reasons/mind_service.py:_resolve_config` uses this precedence:

1. Explicit `api_key` parameter
2. `MIND_API_KEY` environment variable

That means `--api-key` works for local/one-off usage, but CI and production should prefer the environment variable so the secret does not appear in shell history or process listings.

### `.env` files are not currently supported by the application

There is no `python-dotenv` dependency in `pyproject.toml`, and the code only reads `os.environ`. So a local `.env` file will not be loaded automatically by this code path. If the team introduces one, it must be loaded externally and added to `.gitignore`; the observed `.gitignore` currently has no `.env` pattern.

### What is missing

I cannot determine the exact CI secret wiring or production secret manager from these observations alone. Missing inputs include:

- the contents of `.github/workflows/tests.yml`
- deployment manifests or platform configuration
- whether any CI tests are live integration tests vs. fully mocked tests
- whether any wrapper/tool loads `.env` files outside this package

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Check whether any CI job needs a real `MIND_API_KEY` or whether all Mind-service tests are mocked.
- [function] `reasons/mind_service.py:_resolve_config` — Confirm config precedence and the missing-config failure behavior.
- [general] `secret-management` — Identify the production platform’s secret manager and how it injects environment variables.
- [file] `.gitignore` — Add `.env` and related local secret-file patterns if developers introduce dotenv-style files.
- [general] `integration-test-boundary` — Determine whether CI has live-service tests that require secrets versus unit tests that only use monkeypatched env vars.

## Beliefs

- `mind-api-key-resolution-order` — `_resolve_config` uses an explicit `api_key` argument before falling back to `MIND_API_KEY` from the environment.
- `mind-api-key-is-runtime-env-config` — `reasons/mind_service.py` reads `MIND_API_KEY` from `os.environ` only; no file-based key loading is shown in the observed code.
- `unit-tests-isolate-mind-api-key` — `tests/test_mind_service.py` sets and deletes `MIND_API_KEY` via `monkeypatch` rather than requiring a real secret.
- `cli-api-key-defaults-to-env` — The CLI documents `MIND_API_KEY` as the default environment source for `--api-key` at `reasons/cli.py:3481` and `reasons/cli.py:3489`.
- `gitignore-does-not-ignore-dotenv` — The observed `.gitignore` contains no `.env` or secret-file pattern.
