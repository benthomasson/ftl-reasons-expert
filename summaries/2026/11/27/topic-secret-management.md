---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Identify the production platform’s secret manager and how it injects environment variables.

# Identifying the Production Secret Manager and Env-Var Injection

## Short answer

**The observations do not contain enough information to identify a production platform's secret manager or how it injects environment variables.** Every search for deployment or secret-manager artifacts came back empty, and the only CI workflow present (`tests.yml`) contains no `secrets.*`, `env:`, or deploy steps.

What the observations *do* show is the **application-side contract**: the code reads configuration exclusively from the process environment via `os.environ` / `os.environ.get(...)`. Whoever runs it in production — and whichever secret backend populates that environment — is invisible in the files we were shown.

## Evidence

### 1. No secret-manager references anywhere

The pattern search across the repo for `(?i)(secret.?manager|secrets.?manager|vault|doppler|sops|sealed.?secret|external.?secret|secretKeyRef|envFrom|secretRef)` returned **0 matches**. If the platform used Doppler, Vault, SOPS, AWS Secrets Manager, GCP Secret Manager, Kubernetes `secretKeyRef`/`envFrom`, or the like, it would normally appear in:

- a `fly.toml`, `render.yaml`, `railway.json`, `Procfile`, Helm chart, Terraform file, or K8s manifest, **or**
- a production GitHub Actions workflow using `${{ secrets.X }}`.

None of those exist in the observations.

### 2. No platform/deployment references

The second search (`fly.toml|flyctl|render|railway|heroku|kubernetes|helm|terraform|cloud run|gcp|aws|azure|deploy|production`) also returned **0 matches**.

The `.github/workflows` directory contains exactly **one** file, `tests.yml`, and the third search for `secrets.|secret|env:|environment|deploy|production` inside workflows also returned **0 matches**. So the only CI pipeline visible is a test runner — it does not define production environment injection.

### 3. The application only consumes environment variables

The `search_env_var_access_in_python` results (47 matches) show the app is env-var driven, not secret-manager aware. Representative examples:

| File:line | Variable | Purpose |
|---|---|---|
| `reasons/cli.py:156,178,200,231,286,323,1289,1303` | `REASONS_PG_CONNINFO` | PostgreSQL connection string override for CLI persistence |
| `reasons/cli.py:1292` | `REASONS_DOMAIN_ID` | Domain selector for the SQLite/Postgres backend |
| `reasons/cli.py:1293` | `REASONS_PROJECT_ID` | Project selector |
| `reasons/hf.py:18` | `HF_TOKEN` | Hugging Face auth |
| `reasons/hf.py:45` | `REASONS_HF_ORG` | Default HF org / namespace |
| `reasons/llm.py:164` | `LANGFUSE_PUBLIC_KEY` | Langfuse tracing toggle |
| `reasons/llm.py:219` | `GOOGLE_CLOUD_PROJECT` | Vertex AI project |
| `reasons/llm.py:225` | `GOOGLE_CLOUD_REGION` | Vertex AI region (defaults to `us-east5`) |
| `reasons/llm.py:300` | `CLAUDECODE` | Runtime guard passed to subprocess env (all OS vars forwarded) |
| `reasons/mcp_server.py:32-33` | `REASONS_DB` | SQLite path for the MCP server |
| `reasons/mind_service.py:15-17` | `MIND_SERVICE_URL`, `MIND_AGENT_ID`, `MIND_API_KEY` | External "Mind" service credentials |
| `tests/conftest.py:13,25` | `DATABASE_URL` | Test-time DB connection (optional Postgres run) |

All of these are read through `os.environ.get(...)` (with `REASONS_DB` even read via direct `os.environ["REASONS_DB"]` at `mcp_server.py:33`). **No code path in the observations queries a secret manager SDK, and no code path performs injection of these vars into a running process** — injection is presumed to be done by the runtime environment.

### 4. Nothing in docs or project metadata

- `pyproject.toml` declares no runtime dependencies (`dependencies = []`) and no secret-management package in extras.
- `README.md` installs via `pip install ftl-reasons` / `uvx ftl-reasons` and describes CLI usage; it never mentions a production deployment.
- `CLAUDE.md` describes design decisions (SQLite for ACID, no STALE status, etc.) but nothing about hosting.

## What's missing

To actually answer "what is the production platform and how does it inject env vars?", we would need to see (any of):

- A deployment descriptor: `fly.toml`, `render.yaml`, `railway.json`, `Procfile`, `Dockerfile`, Kubernetes manifests/Helm chart, or Terraform.
- The **rest** of `.github/workflows/` beyond `tests.yml` — ideally a `deploy.yml` or `release.yml` using `${{ secrets.* }}`.
- Documentation in `README.md`/`CLAUDE.md` about hosting (currently absent).
- Any `docker-compose*.yml` or `.env.example` that names the runtime vars.

Given the current snapshot, the safest statement is: **the code assumes a configured process environment, and the mechanism that populates it (platform + secret manager) is outside of the files reviewed.**

---

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Confirm the only visible CI job does not reveal secrets injection patterns (e.g., no `env:` block).
- [file] `reasons/cli.py` — Central place where `REASONS_PG_CONNINFO`, `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID` are resolved; find all branches that depend on them.
- [function] `reasons/cli.py:main` — Entry point that reads env-derived defaults, revealing which config is mandatory vs optional in any environment.
- [file] `reasons/llm.py` — Shows GCP Vertex and Langfuse env contracts (`GOOGLE_CLOUD_PROJECT`, `LANGFUSE_PUBLIC_KEY`); useful to know which secrets must be supplied by the platform.
- [general] deployment-config-discovery — Search the repo (outside the reviewed snapshot) for `Dockerfile`, `.env*`, `*.tf`, `fly.toml`, or additional workflows to locate the actual secret manager.

## Beliefs

- `env-vars-are-read-not-injected` — All application config is read from `os.environ`; no code in the observations performs injection of environment variables itself.
- `no-secret-manager-sdk-is-imported` — `pyproject.toml` declares no runtime `dependencies` and no secret-manager client in any optional extra.
- `tests-yml-contains-no-secrets` — The sole visible workflow (`.github/workflows/tests.yml`) matched zero `secrets.`, `env:`, or `deploy` patterns.
- `pg-conninfo-overridable-via-env` — Every CLI subcommand that touches persistence falls back to `os.environ.get("REASONS_PG_CONNINFO")` when `--pg` is absent (e.g., `reasons/cli.py:156, 1289, 1303`).
- `mind-api-key-required-by-mind-service-client` — `reasons/mind_service.py:17` reads `MIND_API_KEY` directly from the environment; any production deployment must supply it externally.
