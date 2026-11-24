---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The modern mutation path (`propose_supersession` + `accept_proposal`) that intentionally avoids this string-metadata style.

## The modern mutation path (`propose_supersession` + `accept_proposal`)

### What "string-metadata style" means here

In the older path, a mutation's intent is stored as **free-text strings embedded in a node's `metadata` dict** (or in a flat "failure_mode / basis" taxonomy on a proposal blob). Two concrete examples visible in the observations:

- **`reasons/network.py:434–441`** — `propose_update`-style changes are annotated with *"Records supersession in both nodes' metadata for display."* The supersession relationship is written as display metadata (`superseded_by`, `supersedes`, …). That's fine for rendering, but the authoritative state lives in string keys/values.
- **`reasons/propose_update.py`** — the LLM-driven path emits structured-but-string-typed proposals (`failure_mode` ∈ `FAILURE_MODES`, `basis` ∈ `BASES`, plus free-form `evidence` / `comment`), which are then parsed and later applied. The mutation is *described* in strings; applying it is a separate, non-transactional step.

The modern path deliberately **does not encode mutation state as strings on a node**. Instead it (a) creates a first-class, addressable *proposal* object, and (b) commits it through an audited `accept_proposal` step that changes the graph structurally.

### The two-step flow

**Step 1 — `propose_supersession` (`reasons/api.py:5287`)**

The function takes an existing node id plus replacement text, and returns a result dict whose `proposal_id` is later fed to `accept_proposal`. The test suite pins the shape:

- `tests/test_api.py:1042` — `api.propose_supersession("a", "Updated text", ...)`
- `tests/test_api.py:1264` — `api.propose_supersession("a", "Updated", new_id="a-fixed", ...)` (you can choose the replacement node id)
- `tests/test_api.py:1430` — `api.propose_supersession("a", "A updated", tags=["review"], ...)`
- `reasons/cli.py:3229` — CLI surface `supersede`, with a `--transitive` flag (`reasons/cli.py:3235`) matching `reasons/api.py:1166` ("With transitive=True, also defeats all ancestors in the supersession…").
- `reasons/api.py:4548` — the `update` command is now routed through `propose_supersession` (`reasons/api.py:4584` is the call site).

Validation is enforced *before* a proposal is minted. The tests assert that these all `raise`:

- `"nonexistent"` old id → error (`tests/test_api.py:1065`)
- a second pending proposal for the same node → error (`tests/test_api.py:1070`)
- a `new_id` that already exists → error (`tests/test_api.py:1074`)

That last trio is the key architectural move: **"is this mutation legal?" is answered by the proposal layer, not by scanning string metadata.**

**Step 2 — `accept_proposal` (`reasons/api.py:5622`)**

`accept_proposal` is the *only* place a proposed supersession becomes real. It takes the `proposal_id` and an optional `voter`:

- `tests/test_api.py:1177` — `api.accept_proposal(r["proposal_id"], voter="reviewer", ...)`
- `tests/test_api.py:1193` — same, inside `test_accept_supersession`
- `tests/test_api.py:1232` — `api.accept_proposal("prop-nonexistent", ...)` is expected to fail
- `tests/test_api.py:1238` — re-accepting the same proposal is expected to fail (idempotency guard)
- `reasons/mcp_server.py:484` — the MCP layer wraps the API with `def accept_proposal(proposal_id: str, voter: str = "")`, and `reasons/mcp_server.py:496` calls `api.accept_proposal(...)`
- `reasons/cli.py:1994–1996` — `cmd_accept_proposal` mirrors it

Two things fall out of this:

1. **The voter is recorded on acceptance, not on proposal.** That makes the accept step auditable in a way that a `metadata["superseded_by"]` string cannot be.
2. **Acceptance is a transition, not a write.** The proposal is a durable object that moves from *proposed* to *accepted* (or fails), and the graph mutation happens in the same step.

### Why this avoids string metadata

- **The relationship is structural, not labeled.** After acceptance the old node comes back if the new node is retracted (`tests/test_api.py:1191`, `test_accept_supersession`). That reversibility is the standard TMS *outlist* mechanism — the supersession is expressed as an `outlist` justification, not as `metadata["supersedes"] = "b"`. The `network.py:434–441` docstring describing "the supersession is reversible" is describing that outlist behavior; the `supersession in both nodes' metadata` line is explicitly *for display*.
- **Validation is centralized.** Because the proposal is a real object, the illegal cases (missing node, duplicate pending, colliding `new_id`) are checked at `propose_supersession` time and again at `accept_proposal` time. With string metadata, those checks are scattered across whatever code happens to read/write the dict.
- **Persistence is delegated.** Both functions call `_pg_dispatch(pg_conninfo, domain_id, ...)` (`reasons/api.py:5308` and `reasons/api.py:5636`). So the proposal and its acceptance live in the Postgres-backed store, not in an in-memory JSON blob attached to the node.
- **It unifies the CLI, API, and MCP surfaces.** `reasons/cli.py:1847`, `reasons/mcp_server.py:368`, and `reasons/api.py:4584` all call the same `propose_supersession`; `reasons/cli.py:1996` and `reasons/mcp_server.py:496` all call the same `accept_proposal`. There's one mutation path, not a string-key convention with multiple writers.

### What the proposals still touch: display metadata

`reasons/network.py:441` still writes supersession info into both nodes' `metadata` — but the docstring frames it as *for display*. The authoritative fact (which node supersedes which, and whether the mutation is accepted) lives in the proposal object and the outlist structural edge, so the display metadata can be regenerated and cannot drift into being the source of truth.

### Gap in the observations

The observations give us the **signature line numbers** (`reasons/api.py:5287`, `5622`) and the **dispatch lines** (`5308`, `5636`), plus the test contracts, but **not the function bodies**. So we can describe the contract (proposal id in, voter in, outlist out, errors on invalid state) but not the exact SQL/schema — e.g. whether proposals live in a `proposals` table, whether `proposal_id` is the same id as the replacement node, or how `transitive=True` walks the ancestor chain. Those details would need the bodies of `propose_supersession` / `accept_proposal` and of `_pg_dispatch`.

## Topics to Explore

- [function] `reasons/api.py:propose_supersession` — Read the body at line 5287 to see how the proposal id is minted and what preconditions are checked before `_pg_dispatch` at 5308.
- [function] `reasons/api.py:accept_proposal` — Read the body at line 5622 to see the state transition, the voter recording, and how the outlist edge is created in the same transaction.
- [file] `reasons/propose_update.py` — The LLM-driven proposal path using `FAILURE_MODES`/`BASES` string taxonomies; contrasts with the modern structural path.
- [file] `reasons/network.py` — The display-only metadata shim around lines 434–441; confirm the supersession string fields are read-only projections.
- [general] `outlist-reversibility` — How the outlist justification makes a supersession reversible (new node retracted ⇒ old node comes back), which is the mechanism that lets `network.py` promise reversibility without storing a "superseded" flag.

## Beliefs

- `propose-supersession-returns-proposal-id` — `propose_supersession` returns a result dict whose `proposal_id` key is what `accept_proposal` expects (see `tests/test_api.py:1177`, `1193`, and `accept_proposal(proposal_id, ...)` in `reasons/api.py:5622`).
- `accept-proposal-rejects-unknown-proposal-id` — `accept_proposal("prop-nonexistent", ...)` is expected to fail rather than silently succeed (`tests/test_api.py:1232`).
- `propose-supersession-rejects-duplicate-pending` — A second pending proposal for the same node fails, and a `new_id` that collides with an existing node fails (`tests/test_api.py:1070`, `1074`).
- `accept-proposal-records-voter` — `accept_proposal` accepts a `voter` argument that is supplied at acceptance time (CLI `reasons/cli.py:1996`, MCP `reasons/mcp_server.py:496`, tests `tests/test_api.py:1177`, `1193`).
- `propose-supersession-dispatches-to-postgres` — Both `propose_supersession` and `accept_proposal` persist via `_pg_dispatch(pg_conninfo, domain_id, ...)` (`reasons/api.py:5308`, `5636`).
- `supersession-records-display-metadata-separately` — `reasons/network.py:441` writes supersession info into node `metadata` for display, while authoritative state is structural.
