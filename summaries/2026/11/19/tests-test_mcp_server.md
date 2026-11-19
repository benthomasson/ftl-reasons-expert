---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_mcp_server.py

# `tests/test_mcp_server.py` — Test Suite for MCP Server Tools

## Purpose

This file is the end-to-end contract test suite for the `reasons.mcp_server` module — the MCP (Model Context Protocol) tool surface that exposes the belief-network operations to AI clients. Every public tool exported by `mcp_server` (search, show, add, retract, assert_belief, what_if, nogood, trace, compact, status, list_beliefs, list_gated, export_markdown, topics, propose_*/accept/reject/withdraw, propose_update, list_proposals, show_proposal) is exercised against a real (temporary SQLite) database built by a shared fixture. The tests act as a behavior specification: they tell you exactly what JSON shape each tool returns, what fields are required, and how errors are reported.

The suite also tests the tool-server's DB-resolution machinery (`_find_db`, `_get_db`) independently of any tool — these are the private helpers that decide which `.db` file each MCP process talks to.

Because the file is guarded by `pytest.importorskip("mcp")`, the *entire module* is skipped when the optional `mcp` package is not installed, reflecting that the server is an optional integration layer over the core reasoning engine.

---

## Key Components

### Module-level fixture: `db`

```python
@pytest.fixture
def db(tmp_path):
    db_path = str(tmp_path / "test.db")
    api.init_db(db_path=db_path)
    api.add_node("premise-a", "A is true", db_path=db_path)
    api.add_node("premise-b", "B is true", db_path=db_path)
    api.add_node("derived-c", "C follows from A and B",
                 sl="premise-a,premise-b", db_path=db_path)
    mcp_server._db = db_path
    yield db_path
    mcp_server._db = None
```

- Bootstraps a fresh DB per test containing: `premise-a`, `premise-b` (both IN), and `derived-c` (IN, supported by both premises).
- **Critical isolation trick:** sets `mcp_server._db = db_path` so every tool call in the test hits *this* DB rather than doing live discovery. Teardown resets `_db = None` so it can't leak to the next test.
- Because `api.init_db`/`add_node` are used directly (not through MCP tools), the fixture also implicitly validates that the MCP layer is a thin wrapper over `reasons.api`.

### `TestFindDb`

Tests the pure, side-effect-free `_find_db()` resolver:

- **`test_env_var_takes_precedence`** — `REASONS_DB` env var wins over filesystem discovery.
- **`test_walks_up_from_cwd`** — creates an empty `reasons.db` in `tmp_path`, chdirs into `tmp_path/sub/deep/`, and expects `_find_db()` to walk upward and return `tmp_path/reasons.db`. Uses `patch("os.getcwd", ...)` because the resolver reads CWD internally.
- **`test_falls_back_to_default`** — no env, no file found → returns `api.DEFAULT_DB`.

### `TestGetDb`

Tests the caching wrapper `_get_db()`:

- **`test_lazy_discovery`** — when `_db is None`, discovery runs once and the result is cached.
- **`test_uses_cached_value`** — when `_db` is already populated, no discovery happens (`"/some/path.db"` is returned without touching disk).

### Tool tests (grouped by tool)

Each `TestXxxTool` class follows the same recipe: call the tool, `json.loads` the result, assert on specific fields. Representative contracts:

| Tool | Contract exercised |
|---|---|
| `search` | default format returns text; `output_format="json"` yields parseable list/dict |
| `show` | returns dict with `text` and `truth_value`; missing id → `{"error": ...}` |
| `explain` | returns dict containing `steps` |
| `list_beliefs` | supports `premises_only=True`, `status="OUT"`, and `status=""` meaning *all*; count of 3 in the fixture |
| `add` | returns `{"node_id": ..., "truth_value": ...}`; a derived node with unsatisfied `sl` comes back `OUT` |
| `retract` / `assert_belief` | state flips **IN→OUT** / **OUT→IN**; result contains `changed` / `went_in` |
| `what_if` | has `action="retract"|"assert"|"supersede"`; returns impact info (`retracted`, `restored`, `old_id`/`new_id`); with `supersede` the original stays IN (projection, not mutation) |
| `nogood`, `trace` | return `nogood_id` / `premises`; missing ids → error dict |
| `compact`, `export_markdown` | return plain strings (not JSON) |
| `status` | includes `nodes`, `in_count`, `total` |
| `list_gated` | includes `gated_count` |

### Proposal tests

These are the most important for understanding the *workflow* contract:

**`TestProposeNogoodTool`** — proposal id shape, `status: "pending"`, requires ≥2 nodes (`test_propose_nogood_too_few`), validates all nodes exist, tags supported via `tags=[...]`.

**`TestProposeUpdateTool`** — patches `reasons.propose_update.invoke_model` (note: patches the *module where it is looked up*, not `reasons.mcp_server.invoke_model`). Verifies:
- With `store=True` (default): `stored_count` present, proposals persisted, `list_proposals().count == 1`.
- With `store=False`: response contains `proposals` and `llm_proposals`, but **no** `stored_count`, and nothing persisted.
- Empty LLM response `"[]"`: `llm_proposals == 0`, no error.

**`TestProposeRetractTool` / `TestProposeSupersedeTool` / `TestProposeAddTool`** — proposal ids follow the prefix pattern `prop-<target>-<action>-<...>`. Each test asserts the target node's truth_value is unchanged (deferred execution). `propose_retract` on an already-OUT node is an error; `propose_add` on an existing id is an error.

**`TestListProposalsTool`** — `proposal_status=""` means "all statuses" (mirroring `list_beliefs(status="")`). Supports `tag="..."` filtering (`TestProposalTagFiltering`).

**`TestAcceptProposalTool`** — the acceptance path is where mutation finally happens:
- retract → target becomes OUT
- supersede → **old** target becomes OUT, new node created
- add → new node becomes IN with the proposed text
- accepting an already-resolved proposal → error
- accepting a proposal whose target already drifted (e.g., manually retracted) → `applied: False`, `status: "stale"` (not an error)
- Cascade: retracting `premise-a` via proposal acceptance cascades to `derived-c` (OUT).

**`TestRejectProposalTool` / `TestWithdrawProposalTool`** — `status` becomes `"rejected"` / `"withdrawn"`; target untouched; non-existent or already-resolved proposals → error.

### `TestNamesOnlyFormat`

A cross-cutting output-format contract: several tools accept `output_format="names-only"` and must return a plain newline-separated identifier list with **no** JSON brackets. Applies to `search`, `list_beliefs`, `compact`, `status`, `list_gated`.

---

## Patterns & Idioms

1. **Class-per-tool, method-per-scenario.** Standard pytest BDD-ish layout. Each tool becomes its own `TestXxxTool` class.
2. **JSON round-trip assertions.** `json.loads(mcp_server.tool(...))` treats the tool's output as an opaque wire format. The test knows nothing about dict internals except through parsing — mimicking a real MCP client.
3. **Manual global-state save/restore in a fixture.** `mcp_server._db` is a module global; the fixture sets it, yields, and resets it. This is the required pattern for testing a module-level singleton safely under pytest.
4. **Patch the lookup location, not the definition.** `patch("reasons.propose_update.invoke_model", ...)` targets the module namespace where `mcp_server.propose_update` resolves the symbol — a common gotcha when mocking across module boundaries.
5. **Env/CWD patching for pure resolvers.** `patch.dict(os.environ, ...)` combined with `patch("os.getcwd", ...)` exercises filesystem-walking logic without touching the real filesystem.
6. **Simulate-then-apply split.** `what_if` and `propose_*` produce no durable state change; `accept_proposal` is the single mutating commit-point for proposals. `add`/`retract`/`assert_belief` are direct mutators.
7. **Sentinel empty-string = "all".** `status=""` and `proposal_status=""` are deliberate "no filter" sentinels — a convention that must stay consistent between the two list endpoints.

---

## Dependencies

**Imports:**
- Standard library: `json`, `os`, `unittest.mock.patch`
- Third-party: `pytest`, optional `mcp` (gated via `importorskip`)
- In-repo: `reasons.api`, `reasons.mcp_server`, implicitly `reasons.propose_update` (patched)

**Depends on (contracts assumed):**
- `reasons.api` — `init_db`, `add_node`, `show_node`, `retract_node`, `DEFAULT_DB`. The fixture relies on these to seed state, and assertions use `api.show_node` to verify side effects directly in the DB (bypassing the MCP layer to avoid masking bugs).
- `reasons.mcp_server` — `_find_db`, `_get_db`, `_db`, and all public tools.
- `reasons.propose_update.invoke_model` — the LLM-call boundary, mocked here so tests don't hit any provider.

**Imported by:** Nothing (leaf test module).

---

## Flow

A typical test executes:

1. `db` fixture runs `api.init_db` + three `api.add_node` calls → temp DB populated. `mcp_server._db` pinned.
2. Test invokes an MCP tool like `mcp_server.retract("premise-a", reason="testing")`.
3. Under the hood the tool calls `_get_db()` (returns the pinned path), delegates to `reasons.api`/`storage` for the actual work, and returns a **JSON string**.
4. Test does `json.loads(...)`, asserts on fields.
5. Test may call `api.show_node("premise-a", db_path=db)` to inspect the ground-truth DB state after the side effect.
6. Teardown: `mcp_server._db = None`; `tmp_path` cleaned up by pytest.

For proposal flow specifically: `propose_*()` writes a `pending` proposal record and returns an id → mutating `accept_proposal(id)` reads the proposal, checks target drift, applies the underlying API call, updates proposal status → returns `{applied, status}`. `what_if()` short-circuits before step 4 and just returns an impact projection.

---

## Invariants

- **Error-as-value, not exception.** Every failed tool call returns a JSON object containing `"error"`; no tool in this file ever raises on bad input (missing node, already-resolved proposal, too-few nogood members, supersede without `text`, etc.).
- **Deferred mutation for proposals.** Until `accept_proposal` runs, `propose_retract/supersede/add` leave the DB identical. Enforced explicitly by `TestProposeRetractTool.test_propose_retract` (checks `premise-a` still IN) and `TestProposeAddTool.test_propose_add` (checks `new-node` still absent).
- **`_find_db` priority is total and deterministic:** `REASONS_DB` env → upward walk from CWD for `reasons.db` → `api.DEFAULT_DB`.
- **`_get_db` caches:** once a value is assigned to `mcp_server._db`, `_get_db` never re-discovers.
- **Proposal-id grammar:** `prop-<target_id>-<action>-<suffix>` (all three `propose_*` tools).
- **Proposal status lifecycle:** `pending` → `accepted` | `rejected` | `withdrawn` | `stale`. Only `pending` proposals may be resolved.
- **Empty-string filter sentinels:** `list_beliefs(status="")` and `list_proposals(proposal_status="")` both mean "unfiltered".
- **Cascade integrity:** accepting a retract that makes `premise-a` OUT also propagates to `derived-c` OUT.
- **`output_format="names-only"` output contains no `[` characters** (validated in `TestNamesOnlyFormat`).

---

## Error Handling

The suite encodes the server's **"errors are data"** contract:

- Every "missing X" test (`show`, `explain`, `retract`, `assert_belief`, `what_if`, `nogood`, `trace`, `propose_*_not_found`, `accept/reject/withdraw_not_found`, `show_proposal_not_found`) asserts `"error" in json.loads(result)` — never `pytest.raises`.
- **State-conflict errors** are also error dicts: `propose_retract_already_out`, `propose_supersede_already_out`, `propose_add_already_exists`, `accept_already_resolved`, `reject_already_resolved`, `withdraw_already_resolved`.
- **Staleness is not an error** — a proposal whose target drifted still *accepts* but reports `applied: False, status: "stale"` (`test_accept_drift_already_out`). This distinction matters for callers: distinguish "couldn't apply because already done" from "invalid request".
- **LLM boundary contamination is prevented.** If `invoke_model` returns empty `[]`, `propose_update` yields `{llm_proposals: 0}` without raising — the tool swallows benign empty results into a normal dict.
- **Partial/validation errors:** `propose_nogood` with fewer than 2 nodes or any unknown node id returns `{"error": ...}` before the proposal is created — so proposals can never reference missing nodes.

---

## Topics to Explore

- [file] `reasons/mcp_server.py` — The module under test; see how `@mcp.tool` decorators, JSON serialization, and error-to-dict conversion are actually implemented.
- [function] `reasons/mcp_server.py:propose_update` — The one tool that dispatches to the LLM (`reasons.propose_update.invoke_model`); worth seeing how `store=False` toggles between "return proposals" and "persist proposals".
- [file] `reasons/propose_update.py` — The LLM prompt/parse logic behind `invoke_model`; understand the JSON schema (`id`, `action`, `failure_mode`, `basis`, `evidence`) the mock is simulating.
- [file] `tests/test_ask_mcp.py` — The sibling MCP test file covering the `ask` tool, likely sharing the same fixture/discovery mechanics.
- [general] `proposal-lifecycle` — Trace how a proposal transitions `pending → accepted/rejected/withdrawn/stale`; check whether the storage layer enforces transitions or the server does.

---

## Beliefs

- `mcp-server-tools-return-json-strings` — Every MCP tool in `mcp_server` returns a JSON-parseable `str`; `compact`, `export_markdown`, and any `output_format="names-only"` call are the only string-returning exceptions (non-JSON payloads).
- `mcp-server-errors-are-value-objects` — Every failure path (missing id, bad state, invalid argument) is reported as `{"error": ...}` in the JSON payload, never by raising an exception.
- `mcp-server-uses-module-global-db-cache` — `mcp_server._db` is a module-level cache consulted by `_get_db()`; setting it in a fixture bypasses filesystem discovery.
- `_find-db-precedence-is-env-then-walk-then-default` — `_find_db()` checks `REASONS_DB`, then walks up from `os.getcwd()` looking for `reasons.db`, then falls back to `api.DEFAULT_DB`.
- `proposals-defer-mutation-until-acceptance` — `propose_retract`, `propose_supersede`, and `propose_add` create `pending` records but leave the belief DB unchanged until `accept_proposal` is invoked.
- `supersede-acceptance-retracts-old-node` — Accepting a supersede proposal sets the *old* node's `truth_value` to `OUT` while creating the new node (`test_accept_supersede`), unlike `what_if(action="supersede")` which previews without retraction.
- `proposal-statuses-enumerated` — Valid terminal proposal statuses are `accepted`, `rejected`, `withdrawn`, and `stale`; any further resolution of an already-resolved proposal returns `{"error": ...}`.
