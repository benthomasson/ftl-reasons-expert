# MCP Server on Cloud Run

**Date:** 2026-07-09

## Context

The EEM builder currently runs on GCE VMs. For serving belief networks to clients via MCP, Cloud Run offers a serverless alternative that scales to zero and requires no VM management. Combined with IAP, it provides Google OAuth gating for enterprise customers.

## Architecture

```
Client (Claude Code, IDE, etc.)
  |
  | MCP Streamable HTTP
  v
Identity-Aware Proxy (Google OAuth)
  |
  v
Cloud Run (FastAPI MCP server)
  |
  v
Cloud Storage (network.json per customer)
```

## Transport

MCP supports two HTTP transports:

- **SSE (Server-Sent Events):** Long-lived connections. Works on Cloud Run (up to 60-minute timeout) but awkward with scale-to-zero — cold starts interrupt the stream, clients must reconnect.
- **Streamable HTTP:** Request/response style, no persistent connection needed. Better fit for serverless. Preferred for Cloud Run deployment.

## MCP Server Implementation

A FastAPI app exposing MCP tools for querying a belief network:

- `search(query)` — semantic search across beliefs
- `show(node_id)` — full node details with dependents
- `explain(node_id)` — trace why a belief is IN or OUT
- `list_topics()` — topic listing from wiki build
- `get_topic(topic)` — beliefs in a topic with summary

The server loads `network.json` from GCS on cold start and caches it in memory. No `reasons.db` needed — the export contains everything for read-only queries.

## Per-Customer Isolation

Each enterprise customer gets:
- Their own `network.json` in a GCS bucket
- Their own Cloud Run service (or a single service with routing by authenticated domain)
- IAP scoped to their Google Workspace domain

Single-service-per-customer is simpler for isolation. Single-service-with-routing is cheaper if there are many small customers.

## Relationship to GCE Builder

The GCE VMs handle the write side: running `code-expert`, `review-beliefs`, `derive`, and building the belief network. The Cloud Run MCP server handles the read side: serving the built network to clients. They share `network.json` via GCS.

```
GCE VM (builder)
  |
  | reasons export -o network.json
  | gsutil cp network.json gs://bucket/customer/
  v
Cloud Storage
  ^
  | load on cold start
  |
Cloud Run (MCP reader)
```

Rebuild frequency is low (daily or on-demand), so eventual consistency between builder and reader is fine.

## Cost

Cloud Run scales to zero — no cost when idle. Per-request pricing. For an MCP server that handles occasional queries against a belief network, this is significantly cheaper than a persistent GCE VM.

## Open Questions

- Should the MCP server also support write operations (add beliefs, retract, defeat) or stay read-only? Write operations would need `reasons.db` and could conflict with the GCE builder.
- How to handle network.json updates — poll GCS, or trigger a reload via Pub/Sub when the builder pushes a new export?
- Should the MCP server serve the wiki HTML too, or keep that on Cloudflare/IAP separately?
