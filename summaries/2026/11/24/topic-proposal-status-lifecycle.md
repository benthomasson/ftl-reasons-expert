---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Explore pending/stale/accepted proposal states and how CLI, API, and MCP callers react to each.

## Proposal States: What the Observations Actually Show

### Short answer

The observations **do not show a pending/stale/accepted proposal state machine anywhere in the codebase.** Every grep for `pending|stale|accepted`, and every grep for `proposal|propose|pending|stale|accepted` in the API, CLI, MCP server, MCP client, and storage modules, returned **zero matches**:

- `proposal_state_grep` — 0 matches
- `api_proposal_handling` — 0 matches
- `cli_proposal_handling` — 0 matches
- `mcp_server_proposal_handling` — 0 matches
- `mcp_client_proposal_handling` — 0 matches
- `storage_proposal_state` — 0 matches

The only file with relevant content is `reasons/propose_update.py` (332 lines), and it does **not** implement proposal states. It produces flat, in-memory proposal dicts and renders them to markdown with suggested shell commands. Acceptance is a manual, out-of-band action.

That means the question "how CLI, API, and MCP callers react to each state" **cannot be answered from the observations** — those callers do not appear to handle proposal states at all. See the "What's missing" section below.

---

### What `reasons/propose_update.py` does contain

Two things that *look* state-related but are not proposal states:

**1. `stale` is a `failure_mode`, not a proposal state.**

```python
FAILURE_MODES = {
    "smuggled-premise",
    "unsupported-superlative",
    "false-causal-link",
    "domain-conflation",
    "contradicted-by-source",
    "stale",
}
```

This set is used in `parse_update_proposals` to validate the LLM's classification of *why a belief should change*. A proposal with `failure_mode="stale"` is a proposal *about a stale belief* — it is not a proposal that is itself "stale." Tests confirm this in `tests/test_propose_update.py` (`test_multiple_proposals`, `test_invalid_failure_mode_defaults`).

**2. `metadata.stale_reason` is per-belief metadata, not proposal state.**

In `format_belief_for_update`:

```python
meta = node.get("metadata", {}) or {}
if meta.get("stale_reason"):
    lines.append(f"Stale reason: {meta['stale_reason']}")
```

This formats a **belief node** for LLM review, surfacing that the belief's source changed. The test `TestFormatBeliefForUpdate.test_stale_node` exercises this with `metadata={"stale_reason": "content_changed"}` on `premise-stale`. Staleness here is a property of the belief's source, not of a proposal object.

So the two occurrences of "stale" in the file are orthogonal to any proposal lifecycle.

---

### The actual proposal lifecycle: generation → markdown → manual command

The real flow in `reasons/propose_update.py` is:

1. **Generate** — `propose_updates(nodes, belief_ids=None, model="claude", timeout=300, batch_size=10, on_batch=None)` batches IN-truth beliefs, calls `invoke_model`, and accumulates results. On any exception it prints a warning to stderr and continues (test `TestProposeUpdates.test_handles_llm_error`).

2. **Parse/sanitize** — `parse_update_proposals` returns a list of dicts with keys `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`. Sanitization rules (verified by tests):
   | Invalid input | Defaulted to |
   |---|---|
   | `action` not in `{"update","retract"}` | `"update"` |
   | `failure_mode` not in `FAILURE_MODES` | `""` |
   | `basis` not in `BASES` | `"prior-knowledge"` |
   | item without `id` | skipped entirely |
   | no JSON array found / malformed | returns `[]` |

3. **Render** — `format_proposal_markdown` / `format_proposals_file` emit a markdown doc. Acceptance is expressed as a **suggested shell command string**:
   - UPDATE → `**Command:** reasons update {nid} "{escaped_text}"`
   - RETRACT → `**Command:** reasons retract {nid} --reason "{failure_mode}: {comment}"`

   There is no proposal ID, no persisted record, no `status` field. The proposal dict dies when the markdown is written.

4. **Accept** — the user runs the emitted `reasons update` / `reasons retract` command. Whatever module implements those commands (not in the observations) is what actually mutates the belief store. From this module's perspective, "accepted" is indistinguishable from "the user copied a command."

---

### Signals that *resemble* states but aren't

Two fields carry review-readiness hints — worth noting because they're the closest thing to a state machine:

- **`basis` as an accept-cost hint.** The prompt text (and `BASES` set) documents `source-divergence` as *"mechanically verifiable, cheap to accept"* and `prior-knowledge` as *"needs scrutiny — reviewer may share your blind spot."* This is a property of the proposal's justification, not a state.

- **Cascade impact as an effect prediction.** In `format_proposal_markdown`, `cascade` distinguishes outcomes:
  - RETRACT with `total > 0`: `"**Cascade impact:** {total} dependent(s) affected"` with items marked `will go OUT` / `will go IN`.
  - UPDATE with `total > 0`: `"**Cascade impact:** {total} dependent(s) need re-review"`.
  - No `cascade` arg: `"**Cascade impact:** not computed"`.

  Tests `test_with_cascade`, `test_update_cascade_says_re_review`, `test_no_cascade`, `test_empty_cascade` cover these branches. This is a prediction of downstream effects, **not** a stored state on the proposal.

`propose_updates`'s `on_batch` callback (test `test_on_batch_callback`) is a progress hook for batching, not a state tracker.

---

### What's missing from the observations

The following files/functions are **not** present in the observations, so the React-to-state question for CLI/API/MCP is unanswerable:

- **No CLI module** is shown. The strings `reasons update` / `reasons retract` are generated but the command implementations are unseen. If proposals are ever persisted or marked accepted/pending, it happens there or in storage.
- **No API module** is shown — no endpoint or method references proposals.
- **No MCP server or client module** is shown.
- **No storage module** is shown. The `metadata.stale_reason` field is *read* in `format_belief_for_update`, but nothing in the observations *writes* it — the writer lives in unseen code.
- **No cascade computation** is shown. `format_proposal_markdown` accepts a `cascade` argument, but who computes it and when is not visible.
- **No pending/accepted vocabulary anywhere.** If these states exist, they're named differently (e.g., the CLI may just apply immediately with no intermediate state).

A careful reader should treat "there is no state machine" as **the most likely conclusion**, not a certainty, until the CLI/storage files are inspected.

---

## Topics to Explore

- [file] `reasons/cli.py` (or equivalent CLI entry module) — Implements the `reasons update` / `reasons retract` commands emitted by `format_proposal_markdown`; this is where, if anywhere, a proposal could be marked accepted or held pending.
- [function] `reasons/propose_update.py:propose_updates` — The batching loop and `on_batch` callback shape the caller contract; the caller (CLI/MCP) decides what to do with `all_results`.
- [general] belief-store-staleness-writer — Nothing in the observations sets `metadata.stale_reason`; find the code that writes it to confirm whether staleness is a source-sync signal or a proposal signal.
- [general] cascade-computation — `format_proposal_markdown` accepts a `cascade` dict (`retracted`, `restored`, `total_affected`); locate the module that computes it to understand what "will go OUT / will go IN" is predicting.
- [file] `reasons/llm.py` — `invoke_model` is the only external dependency of `propose_updates`; its error surface determines what counts as a batch failure (swallowed into a stderr warning).

## Beliefs

- `proposal-states-absent` — No `pending` or `accepted` proposal state strings appear anywhere in the observed code; the proposal lifecycle ends at markdown rendering.
- `stale-is-a-failure-mode` — In `reasons/propose_update.py`, `"stale"` is a member of `FAILURE_MODES` classifying *why a belief* is wrong, not a state of the proposal object.
- `stale-reason-is-node-metadata` — `format_belief_for_update` reads `metadata.stale_reason` from a belief node and renders it as `Stale reason: ...`; it is belief-level metadata, not proposal state.
- `acceptance-is-manual-command` — `format_proposal_markdown` emits `reasons update <id> "..."` / `reasons retract <id> --reason "..."` strings; the module itself persists no acceptance state and does not invoke those commands.
- `parse-proposals-sanitizes-invalid-fields` — `parse_update_proposals` coerces invalid `action`→`"update"`, invalid `failure_mode`→`""`, invalid `basis`→`"prior-knowledge"`, skips items lacking `id`, and returns `[]` when no JSON array parses.
