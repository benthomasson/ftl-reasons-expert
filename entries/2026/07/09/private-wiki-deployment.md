# Private EEM Wiki Deployment Options

**Date:** 2026-07-09

## Context

The eem-wiki generates static HTML from `network.json` belief exports. Currently deployed publicly to Cloudflare Pages at `wiki.llmeem.ai`. For enterprise customers, we need private wikis gated behind authentication.

## Options Considered

### 1. FastAPI + Google OAuth (Custom)

A Python FastAPI server that serves static files from `dist/` with middleware checking for a valid Google OAuth session. Unauthorized requests redirect to the Google login flow. Scales to per-customer wikis by mapping OAuth domains to separate `dist/` directories.

Pros: Full control, familiar stack, easy to extend with API endpoints.
Cons: Custom auth code to maintain.

### 2. Google Cloud IAP (Recommended)

Google's Identity-Aware Proxy handles the OAuth flow natively with zero custom auth code:

1. Build `dist/` as usual with `eem-wiki build-all`
2. Upload to a private GCS bucket
3. Cloud Run service serves the files from the bucket
4. Enable IAP in front of Cloud Run — handles Google OAuth automatically
5. Control access per-user or per-Google-Workspace-domain via IAM

Pros: No auth code, centralized access control, granular IAM policies, scales to multiple customers by deploying separate Cloud Run services.
Cons: GCP-only, IAP configuration complexity.

### 3. Other static hosts (no built-in auth)

GitHub Pages, Netlify, Vercel, S3+CloudFront, nginx — all work since the output is plain HTML. None have built-in Google OAuth gating. Would require a reverse proxy or edge function for auth.

## Per-Customer Architecture

Each enterprise customer gets:
- Their own `network.json` (from their belief network)
- Their own `dist/` build (via `eem-wiki build-all`)
- Their own Cloud Run service behind IAP
- Access gated to their Google Workspace domain

The build pipeline stays the same — only the deployment target changes.

## Key Insight

The eem-wiki is just static HTML. Any static hosting works. The authentication question is independent of the build — it's purely a deployment concern. IAP solves it without touching the wiki codebase.
