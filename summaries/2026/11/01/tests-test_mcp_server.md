---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_mcp_server.py

# `tests/test_mcp_server.py`

## Purpose

This file is the **integration test suite for the MCP (Model Context Protocol) server layer** of the `reasons` knowledge-base system. The MCP server (`reasons/mcp_server.py`) wraps the internal belief-network API into a set of flat, tool-like functions that an AI agent (e.g., an LLM using MCP) can invoke over a protocol connection. This test file owns the responsibility of verifying that every exposed tool:

- Returns well-formed JSON (or plain text for display-oriented tools)
- Produces correct side-effects on the underlying SQLite database
- Handles missing/invalid inputs by returning `{"error": "..."}` rather than raising
- Respects the proposal lifecycle (propose → accept / reject / withdraw)
- Correctly discovers and caches the database path

It does **not** test the MCP transport layer (stdio/HTTP framing) or the client side (`mcp_client.py`)—only the tool-function contracts.

## Key Components

### Shared Fixture: `db`

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

Sets up a **three-node belief graph** (two premises + one derived node with a support-link) in a throwaway SQLite file, injects the path into `mcp_server._db` so tools skip discovery, and cleans up after each test. Every test class below depends on this.

### Test Classes (grouped by tool)

| Class | Tool under test | Contract verified |
|-------|----------------|-------------------|
| `TestFindDb` | `_find_db()` | Env-var > cwd-walk > `api.DEFAULT_DB` precedence |
| `TestGetDb` | `_get_db()` | Lazy discovery + caching |
| `TestSearchTool` | `search()` | Returns matching node IDs; `output_format="json"` parses |
| `TestShowTool` | `show()` | Returns `{"text", "truth_value"}`; missing → `{"error"}` |
| `TestExplainTool` | `explain()` | Returns `{"steps": [...]}` for derivations |
| `TestListBeliefsTool` | `list_beliefs()` | Filters by `premises_only`, `status`; empty status = all |
| `TestAddTool` | `add()` | Creates node; missing SL → node created but `OUT` |
| `TestRetractTool` | `retract()` | Flips truth_value to `OUT`; cascades to dependents |
| `TestAssertBeliefTool` | `assert_belief()` | Re-inverts a retracted node |
| `TestWhatIfTool` | `what_if()` | Hypothetical retract/assert without persisting |
| `TestAddJustificationTool` | `add_justification()` | Appends SL to an existing node |
| `TestNogoodTool` | `nogood()` | Records a nogood over a set of premises |
| `TestTraceTool` | `trace()` | Walks support-links back to premises |
| `TestCompactTool` | `compact()` | Returns a trimmed text summary |
| `TestStatusTool` | `status()` | Returns node counts by truth value |
| `TestListGatedTool` | `list_gated()` | Lists nodes blocked by `unless` conditions |
| `TestExportMarkdownTool` | `export_markdown()` | Produces a readable Markdown dump |
| `TestTopicsTool` | `topics()` | Groups nodes by topic |
| `TestProposeNogoodTool` | `propose_nogood()` / `accept_proposal()` | Proposal creation + acceptance |
| `TestProposalTagFiltering` | `list_proposals(tag=...)` | Tag-based filtering of proposals |
| `TestProposeUpdateTool` | `propose_update()` | LLM-driven bulk proposals (mocked) |
| `TestWhatIfSupersede` | `what_if(..., action="supersede")` | Hypothetical text replacement |
| `TestProposeRetractTool` | `propose_retract()` | Proposes retraction with impact analysis |
| `TestProposeSupersedeTool` | `propose_supersede()` | Proposes text replacement |
| `TestProposeAddTool` | `propose_add()` | Proposes a new node |
| `TestListProposalsTool` | `list_proposals()` | Pagination, status/tag filtering |
| `TestShowProposalTool` | `show_proposal()` | Full proposal detail incl. snapshot + impact |
| `TestAcceptProposalTool` | `accept_proposal()` | Applies the mutation; handles drift/stale |
| `TestRejectProposalTool` | `reject_proposal()` | Marks rejected; node unchanged |
| `TestWithdrawProposalTool` | `withdraw_proposal()` | Proposer can withdraw; node unchanged |
| `TestNamesOnlyFormat` | various (`output_format="names-only"`) | Plain-text output with no brackets/JSON |

### Notable Test Details

- **`TestProposeUpdateTool`** patches `reasons.propose_update.invoke_model` to return a canned JSON string, avoiding any real LLM call. It verifies both the `store=True` (default) and `store=False` paths.
- **`TestAcceptProposalTool.test_accept_drift_already_out`** checks a critical invariant: if the target node was already retracted out-of-band between proposal creation and acceptance, `accept_proposal` returns `{"applied": False, "status": "stale"}` rather than erroring.
- **`TestNamesOnlyFormat`** asserts that `output_format="names-only"` produces newline-separated IDs with **no** `[` characters—confirming it's a lightweight format for token-budget-constrained agents.

## Patterns

1. **Contract-first JSON testing.** Nearly every assertion deserializes the tool's return value with `json.loads(...)` and checks specific keys. This mirrors the MCP protocol contract: tools return a JSON-serializable string.

2. **Error-as-data, not exceptions.** Every "missing node" test asserts `"error" in result` after `json.loads`. The convention is that MCP tool functions **never raise**—they return an error object. Tests enforce this uniformly.

3. **Proposal lifecycle as a state machine.** The tests cover the full transition graph:
   ```
   pending ──→ accepted (applied=True)
   pending ──→ rejected
   pending ──→ withdrawn
   accepted ──→ (further accept/reject → error)
   ```
   Drift detection (`stale`) is tested as a special accept-outcome.

4. **Environment-patching for DB discovery.** `TestFindDb` and `TestGetDb` use `patch.dict(os.environ, ...)` and `patch("os.getcwd", ...)` to simulate different runtime contexts without filesystem coupling.

5. **Mocked LLM boundary.** `TestProposeUpdateTool` isolates the LLM call via `patch("reasons.propose_update.invoke_model", ...)`, keeping the test deterministic and offline.

6. **`pytest.importorskip("mcp")`** at module level ensures the entire file is skipped (not failed) if the `mcp` package isn't installed—useful in minimal CI environments.

## Dependencies

**Imports (this file → other modules):**
- `reasons.api` — the core belief-graph API (`init_db`, `add_node`, `show_node`, `retract_node`, `DEFAULT_DB`). Used both to set up fixture data and to verify post-conditions.
- `reasons.mcp_server` — the module under test. All tool functions are called directly (no HTTP/stdio transport).
- `unittest.mock.patch` — for env-var and LLM-call isolation.
- `pytest` — fixture and `importorskip` machinery.

**Depended upon by:**
- The CI pipeline (`.github/workflows/`) runs this file as part of the test suite.
- No other test file imports from this file (it's a leaf test module).

## Flow

Execution per test follows a consistent shape:

```
1. Fixture `db` runs:
   a. Create temp SQLite DB
   b. Seed 3 nodes (premise-a, premise-b, derived-c)
   c. Set mcp_server._db = db_path

2. Test body:
   a. Call mcp_server.<tool>(...) with arguments
   b. json.loads() the return string (or use as-is for text tools)
   c. Assert on specific keys / values
   d. Optionally verify DB state via api.show_node(...)

3. Fixture teardown:
   a. mcp_server._db = None
   b. tmp_path is deleted by pytest
```

For the **proposal flow** specifically:

```
propose_*()  →  returns proposal_id (status="pending")
     │
     ├── accept_proposal(id)   →  applies mutation, cascades to dependents
     ├── reject_proposal(id)   →  no-op on DB, marks rejected
     └── withdraw_proposal(id) →  no-op on DB, marks withdrawn
```

`what_if()` is a **read-only simulation**: it computes the cascade but does not persist, so subsequent `show_node` calls still reflect the original state (verified in `TestWhatIfSupersede.test_what_if_supersede` where `premise-a` remains `IN` after the what-if).

## Invariants

- **JSON-or-text return contract.** Every tool returns either a JSON string (parseable by `json.loads`) or a plain-text string. It never returns `None`, a raw object, or raises.
- **Error key.** Any failure produces `{"error": "<message>"}` as the top-level object. No HTTP-like status codes.
- **Proposal idempotency guard.** Once a proposal is `accepted`, `rejected`, or `withdrawn`, any further `accept`/`reject`/`withdraw` call returns an error. Tested in `TestAcceptProposalTool.test_accept_already_resolved` and peers.
- **Drift detection.** `accept_proposal` checks whether the target node's current state matches the proposal's expectation; if not, it returns `applied=False, status="stale"` without mutating.
- **`names-only` format has no brackets.** `TestNamesOnlyFormat` asserts `"[" not in result`—this is a hard contract for agents that parse line-by-line.
- **`sl` referencing a non-existent node does not fail `add`.** The node is still created but marked `OUT` (see `TestAddTool.test_add_with_missing_sl_creates_out`).

## Error Handling

The pattern is **uniform and deliberate**:

- MCP tool functions **catch** all internal exceptions and return `{"error": str(e)}`.
- Tests **never** use `pytest.raises` against tool calls. Instead they assert `"error" in json.loads(result)`.
- The only external error path is `pytest.importorskip("mcp")` at module scope, which skips the entire file if the `mcp` package is absent.
- In `TestProposeUpdateTool`, the mocked `invoke_model` always returns a valid JSON string; the real error path (LLM returns garbage) is not tested here—that would belong in `test_propose_update.py`.
- `TestAcceptProposalTool.test_accept_drift_already_out` shows that "business-logic errors" (stale state) are returned as data (`applied=False`), not as `{"error": ...}`. This is a subtle distinction: the operation *succeeded* in processing but *refused to apply*.

## Topics to Explore

- [file] `reasons/mcp_server.py` — The module under test; reveals how each tool function maps to `api.*` calls, how `_find_db`/`_get_db` work, and how the proposal state machine is implemented
- [file] `reasons/api.py` — The core belief-graph API (`add_node`, `retract_node`, `show_node`, cascade logic) that every MCP tool delegates to
- [function] `reasons/mcp_server.py:accept_proposal` — The most complex tool: applies retract/supersede/add, checks drift, cascades to dependents, and transitions proposal status
- [file] `reasons/proposal_update.py` — The LLM-driven bulk proposal engine mocked in `TestProposeUpdateTool`; understanding `invoke_model`'s contract clarifies what `store=True/False` means
- [general] `mcp-protocol-tool-contract` — The broader MCP spec constraints (tool name, input schema, return type) that shape why every function here returns a JSON string and never raises

## Beliefs

- `mcp-tools-return-json-never-raise` — Every public tool function in `mcp_server` returns a `str` (JSON or plain text) and signals failures via `{"error": "..."}` rather than raising exceptions; tests uniformly assert on the parsed dict, never use `pytest.raises`.
- `proposal-lifecycle-is-terminal` — A proposal transitions from `pending` to exactly one of `accepted`, `rejected`, or `withdrawn`; any subsequent state-change call on that proposal ID returns an error, and no proposal can be re-opened.
- `what-if-is-pure` — `what_if()` computes cascade effects but does not persist any mutation; the underlying DB state is unchanged after the call (verified by post-call `show_node` assertions).
- `accept-proposal-detects-drift` — `accept_proposal` compares the target node's current truth_value against the proposal's expectation; if the node was already mutated out-of-band, it returns `{"applied": false, "status": "stale"}` instead of applying or erroring.
- `output-format-names-only-is-bracket-free` — When `output_format="names-only"` is passed, the return string contains newline-separated identifiers with zero `[` or `{` characters, guaranteeing compatibility with line-parsing agents.
