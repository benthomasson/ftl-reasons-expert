---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How the human commands printed in the markdown (`reasons retract … --reason …`, `reasons update …`) actually re-run the true cascade at accept time.

## Short answer

The commands printed in the markdown are **not** the mechanism that re-runs the cascade — they are **human-readable replay instructions**. The cascade re-runs when a human (or script) later executes those commands, because the real CLI handler calls `api.retract_node` / the update equivalent, which loads the **current** network and runs the actual cascade against live state. The cascade shown in the proposals markdown is computed earlier by `what_if_retract`, which is explicitly non-mutating and therefore only an estimate.

In other words: propose-time cascade = advisory (`what_if_*`, no DB writes); accept-time cascade = real (`retract_node` / `update`, loads DB, mutates, saves). The printed command is the bridge between the two.

## End-to-end flow

1. **Propose time — emit the command.**
   `reasons/propose_update.py:257` and `:261` build the literal strings a human will run later:
   - `` **Command:** `reasons update {nid} "{escaped}"` ``
   - `` **Command:** `reasons retract {nid} …` ``

   `reasons/cli.py:917` does the same in a different flow:
   ```python
   print(f"\n  To revoke all: reasons retract {result['active_node']}")
   ```
   These are just strings written into the proposals markdown — no cascade is executed here.

2. **Propose time — compute the *advisory* cascade.**
   `reasons/api.py` around lines 4519–4531 iterates proposals and calls:
   ```python
   cascades[p["id"]] = what_if_retract(p["id"], db_path=db_path)
   ```
   That is how the proposals document gets its per-proposal `cascades` field. Crucially, `what_if_retract` (docstring at `reasons/api.py:517`) says *"and returns the cascade effects. The database is not modified."* The same pattern exists for update (`api.py:652`, "Adds a new node in memory, supersedes old_id, and returns the cascade") and for assert (`api.py:585`). So the markdown cascade is a prediction from the state at propose time.

3. **Propose time — attach cascades to the markdown.**
   `reasons/cli.py:2816–2846` retrieves `result.get("cascades", {})` and passes it into `format_proposals_file(proposals, nodes=nodes, cascades=cascades)`, so the printed commands sit next to the predicted cascade.

4. **Accept time — the human runs the printed command.**
   When `reasons retract … --reason …` is executed, `cmd_retract` in `reasons/cli.py` (after `_print_restoration_hints`, around line 139) does:
   ```python
   result = api.retract_node(args.node_id, reason=args.reason or "", **_backend_kwargs(args))
   ```
   That is the *actual* work. `api.retract_node` (`reasons/api.py:475`, docstring `"""Retract a node and cascade.`) loads the network, marks the node OUT, and runs the real cascade expansion/contraction, returning `went_out`, `went_in`, and `restoration_hints`.

5. **Accept time — the real cascade is printed and persisted.**
   `_print_cascade` (`reasons/cli.py:114–126`) renders `went_out` / `went_in` from the live result. `_print_restoration_hints` (`cli.py:128–137`) prints follow-up `reasons add-justification … --any` hints for multi-premise SLs. The `--reason` value is threaded into `api.retract_node` and persisted in the node's metadata as `retract_reason` — that is why `reasons/export_markdown.py` reads `node.metadata.get("retract_reason")` and emits `- Stale reason: …`.

6. **Why the accept-time cascade is the “true” one.**
   Because `retract_node` (and the equivalent update path) runs against the DB state at the moment the command executes. If other proposals were accepted, nodes were asserted, or justifications changed between propose time and accept time, the true cascade will legitimately differ from the `what_if_retract` snapshot. The markdown command is the only thing that *re-executes* the cascade — nothing in the propose flow persists the effects.

## What the observations do not show

The claim should be qualified, because the snippets do not include:

- An `accept` command handler / proposals-markdown parser (e.g., `cmd_accept`, `api.accept_proposal`, or a `proposals.yaml` loader). Only the *emission* side is visible.
- The full `reasons/propose_update.py` (only lines 257 and 261 are quoted).
- The `reasons update` CLI subparser or `cmd_update` implementation. Only its *emission* is observed. The `retract`/`assert` parsers appear at `cli.py:3107–3112`, but no `update` parser is shown.
- The cascade algorithm itself (the expand/contract logic that produces `went_out`/`went_in`). We see only wrappers (`retract_node`, `what_if_retract`, `_cascade_depth`).

So the flow above is well supported for **retract**; for **update** the same shape is implied by `api.py:652` ("Adds a new node in memory, supersedes old_id, and returns the cascade") and `api.py:4471` ("mode and update basis. Computes cascade impact for retract proposals"), but the exact `reasons update → api.*` call chain is not in the observations.

## Topics to Explore

- [function] `reasons/api.py:what_if_retract` — Confirms the non-mutating advisory path; compare its result to the live `retract_node` result.
- [function] `reasons/api.py:retract_node` — The real cascade entry point; trace how it loads the network, marks OUT, and computes `went_out`/`went_in`.
- [file] `reasons/propose_update.py` — Full emitter for the markdown commands; shows whether commands are meant to be pasted or parsed by an accept step.
- [general] `proposals-accept-lifecycle` — Not covered by the observations; the missing accept handler that (if it exists) reads the markdown or metadata and replays the commands.
- [file] `reasons/network.py` — Where the actual TMS expand/contract cascade is implemented; `retract_node` and `what_if_retract` differ mainly here.

## Beliefs

- `what-if-retract-is-non-mutating` — `what_if_retract` returns cascade effects without modifying the database (docstring at `reasons/api.py:517`).
- `proposal-cascades-use-what-if` — Proposals are annotated with per-proposal cascades computed via `what_if_retract` in `reasons/api.py` (lines ~4519–4531).
- `retract-command-invokes-api-retract-node` — `cmd_retract` in `reasons/cli.py` calls `api.retract_node(args.node_id, reason=args.reason or "", …)`.
- `retract-reason-persists-as-metadata` — The `--reason` value is stored on the node as `metadata["retract_reason"]`, which `export_markdown.py` surfaces as `- Stale reason: …`.
- `propose-time-and-accept-time-cascades-can-diverge` — Because the advisory cascade is a snapshot and the command-triggered cascade re-runs against current DB state.
