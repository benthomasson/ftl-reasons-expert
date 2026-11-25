---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Understand deployment expectations for `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY`.

## Deployment expectations for Mind service config

The three variables are **required runtime configuration** for Mind service integration. They are resolved in `reasons/mind_service.py:_resolve_config` at `reasons/mind_service.py:15-17`:

```python
url = url or os.environ.get("MIND_SERVICE_URL", "")
agent_id = agent_id or os.environ.get("MIND_AGENT_ID", "")
api_key = api_key or os.environ.get("MIND_API_KEY", "")
```

Precedence is: **explicit argument > environment variable > empty string**. After resolution, all three must be non-empty. If any are missing, `_resolve_config` raises a `RuntimeError` listing the missing fields (`reasons/mind_service.py:19-27`). There is no built-in default service URL, agent ID, or API key.

| Variable | Deployment expectation | Evidence |
|---|---|---|
| `MIND_SERVICE_URL` | Base URL of the Mind service, e.g. `https://mind.example.com`. Used to build `/api/agents/{agent_id}/export` and `/api/agents/{agent_id}/beliefs`. Trailing slashes are stripped after resolution. | `reasons/mind_service.py:15`, `_resolve_config`; endpoint construction in `fetch_export` and `push_belief` |
| `MIND_AGENT_ID` | Agent identifier used as a path segment. CLI help describes it as an “Agent UUID” (`reasons/cli.py:3480`, `reasons/cli.py:3488`). The observed code does not validate UUID format. | `reasons/mind_service.py:16`; `fetch_export` / `push_belief` endpoint strings |
| `MIND_API_KEY` | Bearer token sent in `Authorization: Bearer <api_key>`. A `401` response is translated into `Authentication failed. Check your MIND_API_KEY.` | `reasons/mind_service.py:17`, auth error handling at `reasons/mind_service.py:42` and `reasons/mind_service.py:79`; tests at `tests/test_mind_service.py` |

### CLI and library usage

- The CLI exposes `--url`, `--agent-id`, and `--api-key` flags whose help text says the defaults come from `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY` respectively (`reasons/cli.py:3479-3489`).
- The actual fallback happens in `_resolve_config`, not in the `argparse` definitions, so deployment can either inject the env vars into the process or pass explicit CLI flags.
- Tests confirm env-only resolution (`tests/test_mind_service.py:46-48`), explicit parameters overriding env (`tests/test_mind_service.py:55`), and missing-config failures (`tests/test_mind_service.py:60-68`, `tests/test_mind_service.py:182-184`, `tests/test_mind_service.py:239-241`).
- `fetch_export` performs `GET {url}/api/agents/{agent_id}/export` with a 60-second timeout (`reasons/mind_service.py:fetch_export`).
- `push_belief` performs `POST {url}/api/agents/{agent_id}/beliefs` with a JSON body and a 30-second timeout (`reasons/mind_service.py:push_belief`).

### What is not specified in the observations

The observed docs contain no mentions of these variables, so production deployment guidance, secret-management expectations, and manifest examples are not visible here. The code also does not validate URL scheme, agent UUID format, API key format, retries, or key rotation. The observations are sufficient to understand resolution and HTTP usage, but not full operational deployment policy.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect the import/export command wiring around lines 3479-3489 to see how CLI flags reach `_resolve_config`.
- [function] `reasons/mind_service.py:_resolve_config` — Confirms precedence, required-field validation, and URL normalization for all Mind service calls.
- [function] `reasons/api.py:import_api` — Trace how import uses resolved Mind config and merges the service export into the local DB.
- [file] `tests/test_mind_service.py` — Shows expected env-only, explicit-over-env, missing-config, auth-header, and endpoint-path behavior.
- [general] `mind-service-deployment-docs` — Observed docs have no env var mentions; check README or deployment manifests for production guidance.

## Beliefs

- `mind-service-config-requires-all-three-values` — `_resolve_config` raises `RuntimeError` when URL, agent ID, or API key is empty after explicit params and env fallback.
- `explicit-params-override-mind-env-vars` — `_resolve_config` uses explicit parameters before `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY`.
- `mind-api-key-maps-to-bearer-authorization` — Both `fetch_export` and `push_belief` send the resolved API key as `Authorization: Bearer <api_key>`.
- `mind-service-url-strips-trailing-slash` — `_resolve_config` normalizes the resolved service URL by removing a trailing slash before endpoint construction.
- `missing-mind-config-causes-runtime-error` — Missing Mind configuration is surfaced as a `RuntimeError` with the missing field names, not as a silent default.
