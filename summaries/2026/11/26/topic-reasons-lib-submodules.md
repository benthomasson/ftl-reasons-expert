---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Check whether legacy subpaths such as `reasons_lib.api`, `reasons_lib.cli`, and `reasons_lib.mcp_server` exist as shims or only the top-level `__init__.py` does.

## Verdict

Yes — `reasons_lib.api`, `reasons_lib.cli`, and `reasons_lib.mcp_server` all exist as individual backward-compatibility shim modules. It is **not** only the top-level `reasons_lib/__init__.py`.

## Evidence

- The top-level package itself is a shim:
  - `reasons_lib/__init__.py` line 1 says it is a backwards-compatibility shim.
  - Line 2 does `from reasons import *`.

- Each named legacy subpath has its own shim file:
  - `reasons_lib/api.py` line 1: docstring saying use `from reasons.api`.
  - `reasons_lib/api.py` line 2: `from reasons.api import *`.
  - `reasons_lib/cli.py` line 1: docstring saying use `from reasons.cli`.
  - `reasons_lib/cli.py` line 2: `from reasons.cli import *`.
  - `reasons_lib/mcp_server.py` line 1: docstring saying use `from reasons.mcp_server`.
  - `reasons_lib/mcp_server.py` line 2: `from reasons.mcp_server import *`.

- The import inspection confirms these are pure wildcard re-export shims:
  - `reasons_lib/api.py` has no normal imports and only `from_imports` for `reasons.api` with `names: ["*"]`.
  - `reasons_lib/cli.py` has no normal imports and only `from_imports` for `reasons.cli` with `names: ["*"]`.
  - `reasons_lib/mcp_server.py` has no normal imports and only `from_imports` for `reasons.mcp_server` with `names: ["*"]`.

- The search results show this is a broader shim package, not just a one-off:
  - Other `reasons_lib/*.py` shim files appear in the matches, including `ask.py`, `build_wiki.py`, `check_stale.py`, `cluster.py`, `compact.py`, `contradictions.py`, `derive.py`, `export_card.py`, `export_markdown.py`, `hf.py`, `import_agent.py`, `import_beliefs.py`, `llm.py`, `mcp_client.py`, `metadata.py`, `mind_service.py`, `network.py`, `pg.py`, `propose_update.py`, `pubsub.py`, `repair.py`, and `repair_premises.py`.
  - `total_count` is `40`, though that includes unrelated matches such as deprecated CLI aliases in `reasons/cli.py`.

## Important Caveat

These shims use `from reasons.X import *`. That means `from reasons_lib.api import SomeName` works only for names exported by `reasons.api` via wildcard import rules — usually all public names, or exactly `__all__` if `reasons.api` defines it. The observations do not show whether `reasons.api`, `reasons.cli`, or `reasons.mcp_server` define `__all__`, so the precise exported surface of each shim is not fully determined here.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirms the top-level legacy package shim and how `import reasons_lib` forwards to `reasons`.
- [file] `reasons/api.py` — Underlying module re-exported by `reasons_lib/api.py`; inspect whether it defines `__all__`.
- [file] `reasons_lib/mcp_server.py` — Confirms the MCP server legacy path is a wildcard shim, and whether callers import symbols or run it as a module.
- [general] `legacy-import-compatibility` — Test imports such as `import reasons_lib.api`, `from reasons_lib.cli import main`, and `import reasons_lib.mcp_server`.
- [diff] `reasons-lib-shim-coverage` — Compare all `reasons_lib/*.py` shims against corresponding `reasons/*.py` modules to find missing legacy subpaths or missing exported names.

## Beliefs

- `reasons-lib-api-maps-to-reasons-api` — `reasons_lib/api.py` is a shim that wildcard-imports from `reasons.api`.
- `reasons-lib-cli-maps-to-reasons-cli` — `reasons_lib/cli.py` is a shim that wildcard-imports from `reasons.cli`.
- `reasons-lib-mcp-server-maps-to-reasons-mcp-server` — `reasons_lib/mcp_server.py` is a shim that wildcard-imports from `reasons.mcp_server`.
- `reasons-lib-shims-use-wildcard-imports` — The observed shim files consist of a compatibility docstring plus `from reasons.<module> import *`.
- `reasons-lib-contains-multiple-shims` — The legacy package includes per-module shims beyond just `reasons_lib/__init__.py`.
