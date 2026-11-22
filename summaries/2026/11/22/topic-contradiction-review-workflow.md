---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: End-to-end flow: detect → write plan → human edits → parse → apply → nogood backtracking, plus the CLI wiring documented in the file header.

## End-to-end flow supported by the observations

The observed code covers the **propose/plan** side well, and the **nogood backtracking semantics** are documented by tests, but the full end-to-end workflow is not all present in the supplied observations. In particular, the human-edited-plan parser, the apply-plan command, and the `Network.add_nogood` implementation are not shown.

### 1. Detect / propose

The detection/proposal entry point is `propose_updates` in `reasons/propose_update.py`.

- By default it selects only beliefs whose `truth_value == "IN"`:
  - `belief_ids = [nid for nid, node in sorted(nodes.items()) if node.get("truth_value") == "IN"]`
- It filters explicitly supplied `belief_ids` to those present in `nodes`.
- It batches work using `PROPOSE_UPDATE_BATCH_SIZE = 10`.
- For each batch it:
  - Formats each belief with `format_belief_for_update`
  - Builds the prompt from `PROPOSE_UPDATE_PROMPT`
  - Calls `invoke_model` from `.llm`
  - Parses the response with `parse_update_proposals`
  - Accumulates results and optionally calls `on_batch(all_results)`
- Batch failures are caught and warned to stderr; `propose_updates` does not raise for LLM failures.

The prompt asks the model to decide whether each belief needs `"update"`, `"retract"`, or no change. It also instructs the model to classify proposals using:

- failure modes: `smuggled-premise`, `unsupported-superlative`, `false-causal-link`, `domain-conflation`, `contradicted-by-source`, `stale`
- bases: `source-divergence`, `detected-contradiction`, `prior-knowledge`

So in the observed code, “detect” is an LLM audit of IN beliefs, not a deterministic contradiction detector. The basis `detected-contradiction` exists in the taxonomy, but the mechanical contradiction detection code is not shown.

`format_belief_for_update` supplies the LLM with:
- belief ID and text
- truth value
- source and source URL
- stale reason from metadata
- justifications: antecedents, outlist, label
- premise marker when there are no justifications
- dependents, sorted when possible

### 2. Write plan

The plan-writing functions are also in `reasons/propose_update.py`.

`format_proposal_markdown` renders one proposal as a markdown section:

- `## UPDATE: <id>` or `## RETRACT: <id>`
- current text from the node map
- proposed text for updates
- failure mode
- basis
- evidence
- cascade impact, if a cascade dict is supplied
- suggested CLI command

For retractions, the suggested command is:

```text
reasons retract <id> --reason "<failure_mode>: <comment>"
```

For updates, the suggested command is:

```text
reasons update <id> "<escaped proposed text>"
```

Cascade rendering is action-aware:
- for `RETRACT`, affected dependents are shown as “will go OUT” or “will go IN”
- for `UPDATE`, affected dependents are shown as needing re-review
- no cascade supplied renders `**Cascade impact:** not computed`
- empty cascade renders `**Cascade impact:** none`

`format_proposals_file` renders the whole document:

```text
# Proposed Updates

Review each proposal below. Accept by running the suggested command.
```

It iterates proposals, looks up optional cascade data by proposal ID, and appends each markdown section separated by `---`.

The observations show that this function returns a markdown string. They do not show the file-write call that writes it to disk.

### 3. Human edits

The markdown plan is explicitly designed for human review: “Review each proposal below. Accept by running the suggested command.”

However, the observations do not show:

- a parser for a human-edited markdown plan
- a command that reads an edited plan file
- an accept/reject workflow for individual proposal sections
- how edited `proposed_text` values are fed back into the system

So the “human edits” step is manual in the observed code. The primary documented acceptance mechanism is running the suggested CLI command, not editing and re-parsing the plan file.

### 4. Parse

The only parser shown is `parse_update_proposals`, and it parses the **LLM response**, not a human-edited plan.

It uses `json.JSONDecoder.raw_decode` at each `[` position so it can tolerate prose around the JSON array. It then validates each item:

- skips items that are not dicts or lack `"id"`
- invalid `action` defaults to `"update"`
- invalid `failure_mode` defaults to `""`
- invalid `basis` defaults to `"prior-knowledge"`
- normalizes each accepted item into a dict with:
  - `id`
  - `action`
  - `proposed_text`
  - `failure_mode`
  - `basis`
  - `evidence`
  - `comment`

Tests in `tests/test_propose_update.py` cover valid JSON, retractions, multiple proposals, empty arrays, malformed JSON, invalid taxonomy values, prose around JSON, and items missing an ID.

Again: this is LLM-output parsing. No human-edited-plan parser is present in the observations.

### 5. Apply

The plan suggests `reasons update ...` and `reasons retract ...` commands. The CLI file header says:

> CLI for the Reason Maintenance System.  
> Thin wrappers around reasons.api — each command calls an api function and formats the result dict for terminal output.

Observed CLI functions follow that pattern:

- `cmd_init` calls `api.init_db`
- `cmd_add` calls `api.add_node`
- `cmd_add_justification` calls `api.add_justification`
- `cmd_remove_justification` calls `api.remove_justification`
- `cmd_retract` calls `api.retract_node`
- several other commands call `api.mark_duplicate`, `api.mark_superseded`, `api.defeat_justification`, etc.

For retraction specifically, `cmd_retract`:

- calls `api.retract_node(args.node_id, reason=args.reason or "", **_backend_kwargs(args))`
- catches `KeyError`
- prints “already OUT” if nothing changed
- otherwise prints the retraction and cascade output via `_print_cascade`
- prints restoration hints via `_print_restoration_hints`

`_print_cascade` splits nodes into `went_out` and `went_in`. `_print_restoration_hints` suggests restoring multi-premise SL nodes with:

```text
reasons add-justification <node> --sl <surviving-premises> --any
```

However, the observed `cli.py` snippet does **not** include:

- `main()`
- argparse subparser registration
- a `cmd_update` function
- a `cmd_propose_update` function
- a command that writes or reads the proposal markdown file

`pyproject.toml` confirms the CLI entry point:

```toml
[project.scripts]
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

So the CLI wiring is documented at the entry-point level and the thin-wrapper pattern is visible, but the full command registration is missing from the observations.

### 6. Nogood backtracking

The actual backtracking implementation is not shown, but `tests/test_backtracking.py` documents the intended behavior.

The core semantics are:

- `Network.trace_assumptions(node_id)` walks backward through justifications to find premises.
- `Network.find_culprits(nogood_ids)` traces each nogood node back to premises and returns culprit records.
- Culprits are sorted with fewer dependents first.
- Culprit records include `would_resolve`, i.e. which nogood nodes would be resolved by retracting that premise.
- `Network.add_nogood([...])` detects an active contradiction only when all listed nodes are IN.
- If one nogood node is already OUT, no backtracking is needed.
- When backtracking is needed, the system retracts a premise traced through the justification graph, not an arbitrary derived node.
- `_entrenchment` influences victim selection: sourced premises, `OBSERVATION` belief types, and nodes with more dependents are more entrenched.
- Backtracking logs an entry with `action == "backtrack"` and a `culprit` value.
- Cascading is expected: retracting a shared premise can turn multiple derived nodes OUT.

The API layer is also tested via `api.trace_assumptions` and `api.find_culprits`, but their implementations are not in the observations.

A codebase search for `nogood|backtrack` returned no matches in the supplied observation set, even though the tests reference these symbols. That means the implementation likely lives in files not included here, probably `reasons/network.py` and `reasons/api.py`.

## What is missing from the observations

To fully explain the requested end-to-end flow, we would need:

- `reasons/network.py`: actual `trace_assumptions`, `find_culprits`, `add_nogood`, `_entrenchment`, and backtracking/cascade implementation
- `reasons/api.py`: actual `retract_node`, `trace_assumptions`, `find_culprits`, and any update/apply API
- full `reasons/cli.py`: `main()`, argparse setup, subcommand registration, `cmd_update`, and any proposal-file command
- the file-writing/reading layer for the markdown proposal plan
- any parser for a human-edited proposal plan
- the actual `reasons.llm.invoke_model` implementation
- the code that computes cascade impact before writing the plan

## Topics to Explore

- [file] `reasons/network.py` — Contains the real `trace_assumptions`, `find_culprits`, `add_nogood`, and entrenchment/backtracking implementation referenced by `tests/test_backtracking.py`.
- [file] `reasons/cli.py` — The full file should show `main()`, argparse subparsers, and any `propose-update`, `update`, and `retract` command wiring beyond the first 350 lines.
- [file] `reasons/api.py` — The CLI is described as thin wrappers around this module; it should show the apply/retract/trace/find-culprits API contracts.
- [function] `reasons/propose_update.py:parse_update_proposals` — This is the LLM-response parser; its distinction from any human-edited-plan parser is important for the workflow.
- [general] `human-edited-plan-format` — The observed code writes markdown with suggested commands but does not show how an edited plan file is parsed and applied.

## Beliefs

- `propose-updates-defaults-to-in-beliefs` — When `belief_ids` is `None`, `propose_updates` selects only nodes whose `truth_value` is `"IN"`.
- `parse-update-proposals-validates-taxonomy` — Invalid `action` defaults to `"update"`, invalid `failure_mode` defaults to `""`, and invalid `basis` defaults to `"prior-knowledge"`.
- `proposal-markdown-suggests-cli-commands` — `format_proposal_markdown` emits `reasons update <id> "<text>"` for updates and `reasons retract <id> --reason "<mode>: <comment>"` for retractions.
- `nogood-backtracking-retracts-premises` — According to `tests/test_backtracking.py`, `Network.add_nogood` retracts a premise traced through justifications rather than an arbitrary derived node.
- `cli-entrypoint-is-reasons-cli-main` — `pyproject.toml` maps the `reasons` script to `reasons.cli:main`.
