---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Document the `MIND_SERVICE_URL` / `MIND_AGENT_ID` / `MIND_API_KEY` variables and who sets them (CI, shell profile, docs).

## Summary

The three variables are read only when `reasons.mind_service._resolve_config()` is called. That function resolves configuration in this order:

1. Explicit function/CLI arguments
2. Environment variables:
   - `MIND_SERVICE_URL`
   - `MIND_AGENT_ID`
   - `MIND_API_KEY`

The environment lookup happens in `reasons/mind_service.py:15-17`:

```python
url = url or os.environ.get("MIND_SERVICE_URL", "")
agent_id = agent_id or os.environ.get("MIND_AGENT_ID", "")
api_key = api_key or os.environ.get("MIND_API_KEY", "")
```

There is no non-empty built-in fallback. If any value is still empty, `_resolve_config()` raises `RuntimeError` listing the missing fields at `reasons/mind_service.py:21-25`.

## Variable reference

| Variable | Meaning | CLI override | Runtime use |
|---|---|---|---|
| `MIND_SERVICE_URL` | Base URL for the Mind service. Trailing slashes are stripped by `_resolve_config()`. | `--url` | Used to build endpoints like `/api/agents/{agent_id}/export` and `/api/agents/{agent_id}/beliefs`. |
| `MIND_AGENT_ID` | Agent UUID / agent identifier. CLI help calls it “Agent UUID” at `reasons/cli.py:3480` and `reasons/cli.py:3488`. | `--agent-id` | Embedded in service API paths. |
| `MIND_API_KEY` | Bearer token / API key for the Mind service. | `--api-key` | Sent as `Authorization: Bearer <api_key>`. Auth failures mention checking this variable at `reasons/mind_service.py:42` and `reasons/mind_service.py:79`. |

The CLI help text advertises the environment fallback at `reasons/cli.py:3479-3481` and again at `reasons/cli.py:3487-3489`:

```text
--url       Service URL (default: MIND_SERVICE_URL env)
--agent-id  Agent UUID (default: MIND_AGENT_ID env)
--api-key   API key (default: MIND_API_KEY env)
```

Despite the word “default”, the code’s fallback is an empty string, followed by a missing-config error. So in practice these variables are required unless explicit CLI arguments are supplied.

## Who sets them?

Based on the checked-in repository:

- **Production/runtime code does not set them.** It only reads them via `os.environ`.
- **CLI users can override them** with `--url`, `--agent-id`, and `--api-key`.
- **Tests set and clear them with `monkeypatch`.** Examples:
  - `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY` are set in `tests/test_mind_service.py:46-48`.
  - They are cleared in `tests/test_mind_service.py:60-62`, `:67-68`, `:182-184`, and `:239-241`.
- **CI does not appear to set them.** The only checked-in workflow is `.github/workflows/tests.yml`, and a grep for these variables in CI configs found zero matches.
- **Docs do not mention them.** A grep for the three variable names in docs found zero matches.

So the practical answer is: **the runtime environment or the user must set them** — for example via a shell profile, deployment environment, secret manager, or CLI flags. However, this repository snapshot does not show any shell profile, `.env` file, README section, deployment manifest, or CI secret wiring that actually sets them.

Also note that `reasons_lib/mind_service.py` simply re-exports `reasons.mind_service`, so the same environment-variable behavior applies whether code imports `reasons.mind_service` or `reasons_lib.mind_service`.

## What is missing from the observations

The repository evidence is enough to document how the variables are consumed, but not enough to identify who sets them in real deployments. Missing information includes:

- README or user-facing docs that describe setup
- `.env.example` or sample shell configuration
- CI/CD secret definitions outside checked-in workflow files
- Deployment manifests or service configuration
- Developer shell profile files

So the current codebase has a documentation gap: the variables are discoverable from `reasons/mind_service.py`, CLI help, and tests, but not from project documentation or checked-in CI configuration.

## Topics to Explore

- [function] `reasons/mind_service.py:_resolve_config` — Central precedence and missing-config validation for all three environment variables.
- [file] `reasons/cli.py` — Shows the CLI help text and argument definitions for `--url`, `--agent-id`, and `--api-key`.
- [function] `reasons/api.py:import_api` — Likely wires `_resolve_config` into the import workflow; useful to see how missing env vars surface to users.
- [file] `.github/workflows/tests.yml` — Confirms the checked-in CI does not set these variables and relies on mocked tests.
- [general] `mind-service-secret-management` — Decide where `MIND_API_KEY` should live for local development, CI, and production.

## Beliefs

- `env-vars-read-at-resolve-time` — `_resolve_config` reads `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY` from `os.environ` when called, not at import time.
- `explicit-config-overrides-env` — Explicit `url`, `agent_id`, and `api_key` arguments take precedence over their environment-variable counterparts.
- `missing-mind-config-raises-runtimeerror` — `_resolve_config` raises `RuntimeError` naming each missing url/agent-id/api-key before network calls proceed.
- `mind-api-key-is-bearer-token` — `fetch_export` and `push_belief` send `MIND_API_KEY` as an `Authorization: Bearer <key>` header.
- `repo-ci-and-docs-do-not-set-mind-env-vars` — The checked-in docs and `.github/workflows/tests.yml` contain no references to the three variables; tests set and clear them only with `monkeypatch`.
