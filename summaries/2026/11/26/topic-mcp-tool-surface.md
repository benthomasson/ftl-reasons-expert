---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Which operations `mcp_server.py` exposes versus what `cli.py` and `api.py` allow; this defines the agent-facing capability set.

# `mcp_server.py` vs `cli.py` vs `api.py`: the agent-facing capability set

## 1. The shape of the surface

`reasons/mcp_server.py` registers **26 tools** (confirmed by the `@mcp.tool()` grep: 26 matches, lines 57–582). It is a *deliberately narrowed* projection of `reasons/api.py`, organized into four tiers, each marked by a comment banner in the file:

| Tier | Banner line | Tools |
|---|---|---|
| 1 — Core | `# --- Tier 1: Core ---` (~line 54) | `search` (57), `show` (74), `explain` (87), `list_beliefs` (100), `add` (127), `retract` (145), `assert_belief` (159) |
| 2 — Reasoning | `# --- Tier 2: Reasoning ---` (~line 171) | `what_if` (175), `add_justification` (203), `nogood` (222), `trace` (237), `compact` (250) |
| 3 — Proposals | `# --- Tier 3: Proposals ---` (~line 263) | `propose_update` (267), `propose_retract` (317), `propose_supersede` (345), `propose_add` (379), `propose_nogood` (419), `list_proposals` (448), `show_proposal` (469), `accept_proposal` (483), `reject_proposal` (504), `withdraw_proposal` (523) |
| 4 — Data mgmt | `# --- Tier 4: Data management ---` (~line 539) | `status` (543), `list_gated` (556), `export_markdown` (576), `topics` (582) |

The MCP layer is a **thin adapter**: with one exception (`propose_update`, which chains `api.propose_update` → `api.store_llm_proposals`), every tool is a single call into `reasons.api` plus JSON serialization. It does not reimplement TMS logic.

## 2. Three structural differences from `cli.py` / `api.py`

### (a) Database is process-global, not per-call

- `api.py` functions take `db_path: str = DEFAULT_DB` (e.g. `add_node`, `retract_node`, `get_status`) **and** `pg_conninfo` / `domain_id` for the PostgreSQL backend via `_pg_dispatch`.
- `cli.py` threads `**_backend_kwargs(args)` into every call and supports `--db` / `--pg`.
- `mcp_server.py` accepts **no** `db_path`, `pg_conninfo`, or `domain_id` on any tool. Instead `_find_db()` (module scope) resolves one path: `REASONS_DB` env var → walk up from `cwd` for `reasons.db` → `api.DEFAULT_DB`. `_get_db()` caches it lazily in the module global `_db`, and `main()` forces resolution at startup.

**Consequence:** the MCP server is **SQLite-only** and **single-database per process**; multi-tenant or PG deployments cannot be reached over MCP. Tests confirm the env-var/walk-up/default precedence (`tests/test_mcp_server.py::TestFindDb`, `TestGetDb`).

### (b) No access control (`visible_to`) plumbing

`api.py` implements `_is_visible()` (tags ⊆ `visible_to`) and accepts `visible_to` on `get_status`, `show_node`, `explain_node`, `list_nodes`, `search`/`list_negative`. `cli.py` exposes it as `--visible-to` (see `cmd_show`, `cmd_explain`, `cmd_status` via `_parse_visible_to`), and `tests/test_cli.py::TestStatus::test_status_visible_to` / `TestShow::test_show_access_denied` verify denial.

**No MCP tool forwards `visible_to`.** An agent connected via MCP therefore sees every node regardless of `access_tags`, and `export_markdown` (576) hands it the entire network as a document. The proposal tools accept `tags` for *proposal* filtering, but that is not node-level access control.

### (c) Errors become tool results, not exceptions

`api.py` raises (`KeyError`, `ValueError`, `FileExistsError`, `RuntimeError`); `cli.py` catches and does `print(..., file=sys.stderr); sys.exit(1)`. MCP wraps calls in `try/except` and returns `json.dumps({"error": ...})`, so the agent gets a well-formed result instead of a protocol failure — see `show` (74–84), `retract` (145), `what_if` (175), `propose_*`, `accept_proposal` etc., and the assertions in `tests/test_mcp_server.py::TestShowTool::test_show_missing_returns_error`, `TestProposeRetractTool::test_propose_retract_not_found`.

Caveat: several tools have **no** error handling at all — `search`, `list_beliefs`, `compact`, `list_proposals`, `status`, `list_gated`, `export_markdown`, `topics` call `api.*` directly, so a `sqlite3`/IO failure there surfaces as an MCP-level exception.

## 3. Tool-by-tool mapping

| MCP tool | `api.py` function | `cli.py` counterpart | Parameters the MCP layer drops |
|---|---|---|---|
| `search` (57) | `api.search` | `search` / `lookup` | `lookup` has no MCP equivalent; `format` limited to markdown/json/minimal/names-only |
| `show` (74) | `api.show_node` | `show` (`cmd_show`) | `visible_to`; CLI prints sources/tags/timestamps/`pinned_sha` fields the MCP JSON also carries but no filter applies |
| `explain` (87) | `api.explain_node` | `explain` (`cmd_explain`) | `visible_to`; CLI's `names-only` format |
| `list_beliefs` (100) | `api.list_nodes` | `list` | `min_depth`, `max_depth`, `label`, `visible_to` (all exercised in `tests/test_api.py::TestListNodesDepth` / `test_list_by_label`) |
| `add` (127) | `api.add_node` | `add` (`cmd_add`) | **`cp`, `source`, `source_url`, `namespace`, `any_mode`, `access_tags`, `example`, `source_type`, `accepted_pr`** |
| `retract` (145) | `api.retract_node` | `retract` | — |
| `assert_belief` (159) | `api.assert_node` | `assert` | — |
| `what_if` (175) | `what_if_retract` / `what_if_assert` / `what_if_supersede` | `what-if {retract,assert,supersede}` | — |
| `add_justification` (203) | `api.add_justification` | `add-justification` | `cp`, `any_mode` |
| `nogood` (222) | `api.add_nogood` | `nogood` | — |
| `trace` (237) | `api.trace_assumptions` | `trace` | `visible_to` |
| `compact` (250) | `api.compact` | `compact` | — |
| `propose_update` (267) | `api.propose_update` + `api.store_llm_proposals` | — (not in observed CLI window) | `store` is new; MCP is the only surface with the store/no-store switch |
| `propose_retract` (317) | `api.propose_retraction` | — | — |
| `propose_supersede` (345) | `api.propose_supersession` | — | — |
| `propose_add` (379) | `api.propose_addition` | — | — |
| `propose_nogood` (419) | `api.propose_nogood` | — | — |
| `list_proposals` (448) / `show_proposal` (469) | `api.list_proposals` / `show_proposal` | — | — |
| `accept_proposal` (483) / `reject_proposal` (504) / `withdraw_proposal` (523) | `api.accept_proposal` / `reject_proposal` / `withdraw_proposal` | — | — |
| `status` (543) | `api.get_status` | `status` (`cmd_status`) | **all filters**: `namespace`, `status`, `premises`, `limit`, `offset`, `visible_to`, `--all` |
| `list_gated` (556) | `api.list_gated` | — (not in observed CLI window) | — |
| `export_markdown` (576) | `api.export_markdown` | `export-markdown` | `--output` file target (MCP always returns the string) |
| `topics` (582) | `api.topics` | — | — |

## 4. Capabilities present in `api.py`/`cli.py` but absent from MCP

These are **not callable by an agent over MCP**:

**Lifecycle / backend**
- `api.init_db` / `cli init` — the MCP server requires an existing `reasons.db`; it will not create one.
- `api.propagate` / CLI `propagate` — recompute/repair after out-of-band mutation.
- `ensure_namespace`, `list_namespaces` — namespace (agent-premise) management.
- `get_log`, `get_belief_set`, `export_network` (JSON `export`).

**Structure mutation**
- `api.remove_justification` (`cmd_remove_justification`) — the inverse of `add_justification` (203). Notably one-way only from MCP.
- `api.supersede` / `supersede_with_text` (`cmd_supersede`, incl. `--transitive`). MCP only offers `what_if(action="supersede")` and `propose_supersede`; there is no direct supersede.
- `api.convert_to_premise` (`cmd_convert_to_premise`).
- `api.add_tags` / `remove_tags` / `get_tags` (`cmd_tag`, `cmd_untag`, `cmd_tags`) and `api.add_source` (`cmd_add_source`).
- `api.set_metadata` / `cmd_get_metadata` (the latter reads `show_node`).
- `api.mark_duplicate`, `api.mark_superseded`.

**Defeater machinery** (SQLite-only, explicitly rejected under `--pg` by `cmd_mark_duplicate`, `cmd_defeat_justification`, etc.)
- `api.defeat_justification`, `api.defeat_with_scope`, `api.migrate_retract_to_defeaters`, `api.classify_defeat_reason_types`.

**Argumentation / query extras**
- `api.challenge`, `api.defend` (`cmd_challenge`, `cmd_defend`).
- `api.summarize` (`cmd_summarize`).
- `api.list_negative` (LLM-classified negatives) — **no** MCP tool and **no** CLI cmd in the observed window.
- CLI `trace-access-tags`, `check-integrity`, `backfill-hashes`.

## 5. Design observations worth flagging to the agent author

1. **Two parallel paths for every mutation.** Agents can both *commit* (`add` 127, `retract` 145, `assert_belief` 159, `add_justification` 203, `nogood` 222) and *propose* (`propose_add`, `propose_retract`, `propose_supersede`, `propose_nogood`). The tool surface does not enforce a propose-then-review workflow; the direct-write tools are first-class.
2. **`accept_proposal` (483) is exposed to the agent**, so an agent can accept its own proposal. The "human gate" lives in the proposal *convention*, not in the MCP permission boundary.
3. **`nogood` (222) mutates immediately** — `api.add_nogood` triggers dependency-directed backtracking and retracts a premise. `propose_nogood` (419) is the non-mutating sibling. Both present.
4. **Silent default in `what_if`.** The `else` branch (line ~196) falls back to `what_if_retract`, so `action="typo"` silently simulates a retraction rather than erroring.
5. **`0` means "no limit".** `search` and `list_beliefs` translate `limit or None`, so clients cannot request "zero results".
6. **`add` cannot express CP.** `api.add_node` supports `cp=` (counterfactual/CP justifications) and `any_mode` OR-expansion; MCP's `add` (127) and `add_justification` (203) expose only `sl` / `unless` / `label`. Agents are restricted to strict-antecedent and outlist reasoning.
7. **Drift semantics are real.** `accept_proposal` re-validates against live state and marks the proposal `stale` instead of applying (`tests/test_mcp_server.py::TestAcceptProposalTool::test_accept_drift_already_out`, `test_accept_cascade`).

## 6. What the observations do **not** cover

- `cli.py` is 4015 lines but only lines 1–800 were returned, and the **subparser table** (presumably near the bottom) is not visible. The CLI command list above is partly inferred from `cmd_*` functions in lines 1–800 and from `tests/test_cli.py`; the authoritative list of `add_parser(...)` names (and which `propose-*` commands exist) is unverified.
- `api.py` returned lines 1–800 of 5886: `search`, `list_nodes`, `show_node`, `explain_node`, `compact`, `trace_assumptions`, `add_nogood`, and the entire proposals/acceptance block **were not read** — their signatures are inferred from call sites in `mcp_server.py` and from tests.
- `tests/test_cli.py` and `tests/test_mcp_server.py` were truncated at 600 of 1617 and 641 lines respectively.
- No MCP tool exposes `pg_conninfo`, but `api._pg_dispatch` exists — whether any MCP tool *should* support it is a design question, not answered by the code.

## Topics to Explore

- [file] `reasons/cli.py` — Read the `argparse` subparser definitions (past line 800) to get the authoritative CLI command list and confirm which `propose-*` verbs exist outside MCP.
- [function] `reasons/api.py:search` — Verify the `format` values MCP's `search` (line 57) forwards and whether `visible_to` is accepted (it is not passed by MCP).
- [function] `reasons/api.py:list_nodes` — Confirm the full filter set (`min_depth`, `max_depth`, `label`) that `mcp_server.list_beliefs` (line 100) silently drops.
- [function] `reasons/api.py:accept_proposal` — Understand drift re-validation and why an agent holding `accept_proposal` (line 483) can bypass human review.
- [general] `mcp-access-control-gap` — The absence of `visible_to` on every MCP tool means `access_tags` and `export_markdown` interact as a full-network egress path for agents.

## Beliefs

- `mcp-server-exposes-26-tools` — `reasons/mcp_server.py` registers exactly 26 `@mcp.tool()` functions (lines 57–582) split across four tier banners: core, reasoning, proposals, data management.
- `mcp-tools-are-thin-api-wrappers` — Every MCP tool delegates to exactly one `reasons.api` function, except `propose_update` (line 267), which also calls `api.store_llm_proposals`.
- `mcp-requires-reasons-db-discovery` — No MCP tool accepts `db_path`; `_get_db()`/`_find_db()` resolve a single path from `REASONS_DB`, a cwd walk-up for `reasons.db`, or `api.DEFAULT_DB`, cached in the module global `_db`.
- `mcp-omits-visible-to` — None of the 26 MCP tools forward `visible_to`, so the `access_tags` filtering available via `cli.py --visible-to` and `api._is_visible` cannot be applied to agent requests.
- `mcp-omits-postgres-backend` — MCP never passes `pg_conninfo` or `domain_id` to `api.*`, so the server is SQLite-only even though `api._pg_dispatch` and CLI `--pg` support PostgreSQL.
- `mcp-add-drops-cp-and-namespace` — `mcp_server.add` (line 127) exposes only `node_id/text/sl/unless/label`, so agents cannot create CP justifications, namespaced nodes, `any_mode` OR-justifications, or `access_tags` that `api.add_node` supports.
- `mcp-returns-error-dicts-not-exceptions` — Wrapped MCP tools return `{"error": ...}` JSON on `KeyError`/`ValueError` (e.g. `show` line 74, `retract` line 145), whereas `api.py` raises and `cli.py` converts to `stderr` + `sys.exit(1)`.
