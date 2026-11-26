---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Trace how credentials and env vars are sourced, scoped, and whether any are exposed to untrusted PR events.

## Trace: credential/env sourcing, scoping, and PR exposure

### 1. CI workflow surface and PR events

The observed CI surface is minimal: there is one workflow, `.github/workflows/tests.yml`, and it triggers on `pull_request` at `.github/workflows/tests.yml:6`.

The workflow search for `secrets.` / `${{ env.` returned **zero matches**. That means, from the observations, there is **no direct evidence of repo secrets being referenced in the workflow**. It also means there is no observed `pull_request_target`, `issue_comment`, or `workflow_run` trigger, which are common paths where untrusted PR-related code can run in a privileged context.

Important caveat: the search pattern would not catch every secret mechanism. It would not catch GitHub syntax such as `secrets: inherit`, nor static `env:` blocks, nor secrets passed as action inputs. The full body of `.github/workflows/tests.yml` is not included, so we cannot fully verify job-level env, permissions, or action usage.

### 2. Runtime env var and credential sourcing

The code reads environment variables directly from the process environment using `os.environ.get(...)` and `os.environ[...]`. There is no observed central secret manager or central config loader.

| Area | Source and precedence | Files / lines |
|---|---|---|
| PostgreSQL connection | CLI `--pg` preferred; fallback to `REASONS_PG_CONNINFO` | `reasons/cli.py:1289`, `reasons/cli.py:1303`; also checked at `reasons/cli.py:156`, `178`, `200`, `231`, `286`, `323` |
| PostgreSQL project/domain fallback | `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID` | `reasons/cli.py:1292-1293` |
| Alternate DB env | `REASONS_DB` checked and then read directly | `reasons/mcp_server.py:32-33` |
| HuggingFace token | Explicit token > `HF_TOKEN` env > cached token | `reasons/hf.py:15-18`; `reasons/hf.py:62`; `reasons/api.py:2356`, `2368`; CLI help at `reasons/cli.py:3457`, `3464`, `3469` |
| HuggingFace org | `REASONS_HF_ORG` fallback to `DEFAULT_HF_ORG` | `reasons/hf.py:45` |
| Mind service connection | `MIND_SERVICE_URL`, `MIND_AGENT_ID`, `MIND_API_KEY` | `reasons/mind_service.py:15-17`; CLI help at `reasons/cli.py:3481`, `3489` |
| Langfuse | Checks `LANGFUSE_PUBLIC_KEY` | `reasons/llm.py:164` |
| Google Cloud | `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_REGION` with default `us-east5` | `reasons/llm.py:219-225` |
| Tests | `DATABASE_URL` and monkeypatched env vars | `tests/conftest.py:13`, `tests/conftest.py:25`; many test patches in `tests/test_llm.py`, `tests/test_import_hf.py`, `tests/test_mind_service.py` |

Credential-like values observed are primarily:

- `HF_TOKEN` — HuggingFace auth token.
- `MIND_API_KEY` — Mind service API key.
- `REASONS_PG_CONNINFO` — PostgreSQL conninfo; likely contains credentials depending on form.
- `DATABASE_URL` — test database connection string.
- `LANGFUSE_PUBLIC_KEY` — public Langfuse key, not secret in the same sense but still configuration-sensitive.

Non-secret configuration env vars include `REASONS_HF_ORG`, `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID`, `REASONS_DB`, `MIND_SERVICE_URL`, `MIND_AGENT_ID`, `GOOGLE_CLOUD_PROJECT`, and `GOOGLE_CLOUD_REGION`.

### 3. Scoping and precedence

The observed precedence pattern is generally:

1. Explicit CLI argument / function argument.
2. Environment variable.
3. Default value or cached credential.

Examples:

- `reasons/hf.py:15` documents HuggingFace token resolution as explicit token > `HF_TOKEN` env > cached file.
- `reasons/cli.py:1289` and `1303` use `getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO")`.
- `reasons/mind_service.py:17` uses `api_key = api_key or os.environ.get("MIND_API_KEY", "")`.

Because most reads use `or` semantics, an empty-string env var will fall through to the next source. `reasons/mcp_server.py:33` is different: it checks `REASONS_DB` first and then reads `os.environ["REASONS_DB"]` directly.

The broadest scope observed is `reasons/llm.py:300`:

```python
env = {k: v for k, v in os.environ.items() if k != "CLAUDECODE"}
```

That creates a copy of the entire process environment except `CLAUDECODE`. If this mapping is passed to a subprocess or external invocation, then any credential present in the parent process — `HF_TOKEN`, `MIND_API_KEY`, `REASONS_PG_CONNINFO`, `DATABASE_URL`, etc. — would be inherited unless filtered later. The observation shows the env construction but not the downstream sink, so the exact consumer needs confirmation.

### 4. Exposure to untrusted PR events

Based on the observations:

- The only PR-related trigger is `pull_request` in `.github/workflows/tests.yml:6`.
- No `pull_request_target`, `issue_comment`, or `workflow_run` trigger was observed.
- No `secrets.*` or `${{ env.* }}` usage was found in workflows.
- Therefore, there is **no observed workflow-level secret exposure to PR events**.

However, the codebase itself is not the only factor. If `.github/workflows/tests.yml` or a called action injects credentials into the job environment, PR test code can read them because tests execute in the same process environment. The broad environment copy at `reasons/llm.py:300` would then include those credentials if that code path runs.

For standard GitHub `pull_request` events from forks, repository secrets are not passed by default and the `GITHUB_TOKEN` is typically read-only. But same-repository PRs, protected environments, or custom workflow configuration can change that. We cannot determine actual exposure without the full workflow file and repository settings.

### 5. What the observations do not cover

- `find_env_var_reads` reports `total_count: 47`, but only 30 matches are shown. Seventeen env reads are missing from the excerpt.
- `find_credential_keys` reports `total_count: 32`, but only 30 matches are shown.
- The full contents of `.github/workflows/tests.yml` are missing.
- The `secrets.*` / `${{ env.* }}` search does not prove there are no secrets; it would miss `secrets: inherit`, static `env:` blocks, and action-level inputs.
- The downstream use of the env copy at `reasons/llm.py:300` is not shown.
- No credential documentation matches were returned, so credential env vars appear undocumented in the scanned matches.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Read the full workflow to verify `permissions`, job-level `env`, action inputs, and whether any secret context or inherited secrets are used.
- [file] `reasons/llm.py` — Inspect the code around line 300 to see where the filtered environment copy is passed and whether it reaches a subprocess or external service.
- [file] `reasons/hf.py` — Confirm the exact HuggingFace token precedence and cached-token fallback behavior.
- [file] `reasons/cli.py` — Review the CLI argument and environment fallback paths for PostgreSQL and Mind service credentials.
- [general] `github-actions-secret-scoping` — Check repository/environment settings and default fork PR secret behavior to determine actual CI exposure.

## Beliefs

- `pull-request-workflow-has-no-observed-secret-expressions` — `.github/workflows/tests.yml` triggers on `pull_request`, and the workflow search found no `secrets.*` or `${{ env.* }}` matches.
- `hf-token-resolution-is-explicit-then-env-then-cache` — `reasons/hf.py:15` documents and `reasons/hf.py:18` implements `HF_TOKEN` as the environment fallback after an explicit token and before the cached token.
- `pg-connection-resolution-prefers-cli-over-env` — `reasons/cli.py:1289` and `reasons/cli.py:1303` resolve `pg` from `args.pg` before `REASONS_PG_CONNINFO`.
- `llm-env-copy-inherits-all-but-claudecode` — `reasons/llm.py:300` builds an environment mapping from all `os.environ` entries except `CLAUDECODE`.
- `mind-service-key-falls-back-to-mind-api-key` — `reasons/mind_service.py:17` uses `MIND_API_KEY` when no explicit `api_key` is provided.
