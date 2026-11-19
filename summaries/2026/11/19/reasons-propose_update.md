---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/propose_update.py

# `reasons/propose_update.py`

## Purpose

This module owns the **LLM-driven update-proposal pipeline** for the Truth Maintenance System (TMS). Its single responsibility: given the current node graph, send beliefs (with their justification chains, sources, and dependents) to an LLM in batches, then parse and normalize the LLM's structured suggestions into a list of "proposal" dicts — each tagged with an `action`, `failure_mode`, `basis`, and `evidence`. It does **not** apply any change to storage; it produces proposals that downstream code (`api.py`, `cli.py`) renders and the human accepts.

It also provides the **markdown rendering layer** (`format_proposals_file`) that produces the human-review document, including cascade-impact predictions passed in by the caller.

## Key Components

### Constants

- **`PROPOSE_UPDATE_BATCH_SIZE = 10`** — default number of beliefs per LLM call.
- **`FAILURE_MODES`** — a `set` of 6 allowed categories: `smuggled-premise`, `unsupported-superlative`, `false-causal-link`, `domain-conflation`, `contradicted-by-source`, `stale`.
- **`BASES`** — a `set` of 3 allowed provenance values: `source-divergence`, `detected-contradiction`, `prior-knowledge`.
- **`PROPOSE_UPDATE_PROMPT`** — a `str.format` template with a single `{beliefs}` placeholder. It embeds the failure-mode/basis taxonomy verbatim and instructs the model to return only a JSON array, skip good beliefs, and set `proposed_text: null` for retract actions.

### Functions

**`format_belief_for_update(node_id: str, nodes: dict) -> str`**
Serializes one belief into a `### <id>` markdown block containing:
- `Text`, `Status` (`truth_value`)
- `Source`, `Source URL`, `Stale reason` (from `metadata.stale_reason`) if present
- `Justification N/M` for each justification with `Antecedents`, `Unless (OUT)` lists, and `Label`. Falls back to `Type: premise (no justifications)` when none exist.
- `Dependents (N)` sorted by ID.
Returns `""` for an unknown node ID — a silent no-op so callers can feed arbitrary IDs.

**`parse_update_proposals(response: str) -> list[dict]`**
Scans each `[` character in the response and attempts `json.JSONDecoder().raw_decode` at that offset. This is the key robustness trick: the LLM may wrap the array in prose or fences, and `raw_decode` extracts the first valid JSON value without needing to strip text. For the first successfully parsed list with ≥ 1 valid item, each item is validated and coerced:
- Must be a `dict` with `"id"`.
- `action` coerced to `"update"` or `"retract"`; anything else → `"update"`.
- `failure_mode` not in `FAILURE_MODES` → `""`.
- `basis` not in `BASES` → `"prior-knowledge"`.
- Missing fields defaulted to `""`/`None`.
Returns `[]` if no array parses to a non-empty list of dicts.

**`format_proposal_markdown(proposal, nodes=None, cascade=None) -> str`**
Renders one proposal as a `## UPDATE: <id>` / `## RETRACT: <id>` section with current text, proposed text (only for UPDATE), failure mode, basis, evidence, and cascade impact. Cascade rendering is action-aware: RETRACT shows `will go OUT` / `will go IN`; UPDATE shows `need re-review`. Includes a **copy-paste command**:
- `reasons update <nid> "<proposed_text escaped>"`
- `reasons retract <nid> --reason "<failure_mode>: <comment>"`

**`format_proposals_file(proposals, nodes=None, cascades=None) -> str`**
Top-level document assembler: title, review instructions, then each proposal separated by `---`. Returns a "No updates proposed." document for an empty list.

**`propose_updates(nodes, belief_ids=None, model="claude", timeout=300, batch_size=PROPOSE_UPDATE_BATCH_SIZE, on_batch=None) -> list[dict]`**
The orchestrator:
1. If `belief_ids is None`, selects all nodes where `truth_value == "IN"`, iterating over `sorted(nodes.items())` for deterministic ordering.
2. Otherwise filters the provided list to IDs actually present in `nodes`.
3. Returns `[]` early if the resulting list is empty.
4. Loops in `batch_size` chunks. For each batch: builds a prompt, calls `invoke_model`, parses, extends `all_results`, and invokes the `on_batch(all_results)` callback with the cumulative list.
5. Prints progress to `sys.stderr` (`Reviewing batch N/M (K beliefs)...`).

## Patterns

- **Batch-with-fault-isolation**: one batch failing (API error, timeout) is logged and skipped, remaining batches continue.
- **Progressive parsing with `json.JSONDecoder.raw_decode`**: avoids brittle regex/`json.loads` on prose-wrapped output.
- **Enum coercion at the boundary**: external (LLM) data is never trusted to match the taxonomy; invalid values are downgraded rather than rejected, keeping proposals usable.
- **Callback-based incremental results**: `on_batch` allows a UI to stream progress without the function needing to know the consumer.
- **Renderer separation**: proposal data (`dict`) is decoupled from markdown formatting, so API/CLI/tests can consume the same dicts.
- **Deterministic ordering**: default belief selection and dependent formatting both sort, which matters for reproducible prompt payloads (and likely for caching/testing).

## Dependencies

**Imports (internal):** only `from .llm import invoke_model`. Everything else is stdlib (`json`, `sys`).

**Imported by:**
- `reasons/api.py` — likely wraps `propose_updates` for programmatic access.
- `reasons/cli.py` — wires up the CLI command that emits the markdown file.
- `reasons_lib/propose_update.py` — mirror copy (to be kept in sync — this file is duplicated in the repo layout).
- `tests/test_propose_update.py` — unit tests for formatting/parsing.

**Implicit contract:** the `nodes` dict schema — `{id: {"text", "truth_value", "source", "source_url", "metadata": {"stale_reason"}, "justifications": [{"antecedents", "outlist", "label"}], "dependents"}}`. This schema is defined by `storage.py` / `api.py`, not here, so a schema change there silently degrades prompt quality here.

## Flow

```
propose_updates(nodes, belief_ids=None, ...)
  ├─ select belief_ids (IN-only) or filter provided list
  ├─ for each batch chunk of size batch_size:
  │    ├─ format_belief_for_update for each id  →  "### id\nText: ...\n..."
  │    ├─ PROPOSE_UPDATE_PROMPT.format(beliefs=...)  →  full prompt
  │    ├─ invoke_model(prompt, model, timeout)        →  raw response
  │    ├─ parse_update_proposals(response)            →  list of normalized dicts
  │    ├─ all_results.extend(...)
  │    ├─ on_batch(all_results)                       (if provided)
  │    └─ (on exception) print WARN, continue
  └─ return all_results
```

Rendering path (typically in CLI):
```
propose_updates(...) → list[dict]
   → format_proposals_file(proposals, nodes, cascades)
        → format_proposal_markdown(proposal, nodes, cascade) for each
```

## Invariants

- **Default candidacy is the IN set.** Only beliefs with `truth_value == "IN"` are proposed for update when `belief_ids` is `None` (no explicit set passed).
- **Unknown IDs are dropped.** If `belief_ids` is explicit, any ID not in `nodes` is filtered out before prompting — the prompt never references a missing node in that path.
- **Returned dicts always have the same 7 keys:** `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`. Consumers can rely on `.get()` success for these.
- **`failure_mode` is either one of the 6 taxonomy values or `""`.** Same for `basis` (`FAILURE_MODES`/`BASES` membership) and `action` (`"update"`/`"retract"`).
- **Only the first fit-for-purpose array wins.** `parse_update_proposals` returns on the first parsed JSON list that yields ≥ 1 valid item; later candidate arrays in the same response are ignored.
- **Empty input ⇒ `[]`, no LLM call.** No prompt is issued and no progress line printed.
- **`on_batch` receives the cumulative list, not the batch slice.** Callback consumers that append should beware double-counting if they concatenate.

## Error Handling

- **Per-batch failures are swallowed.** Any exception from `invoke_model` or downstream (including `parse_update_proposals` and even the `on_batch` callback, since it's inside the `try`) is caught and printed as `WARN: batch N failed: <e>` to stderr. Execution continues to the next batch. This means a broken `on_batch` callback will not abort the run — a subtle debugging gotcha.
- **JSON parse failures are non-fatal and silent within `parse_update_proposals`.** `json.JSONDecodeError` per `[` position is caught and the loop continues; only if no valid array is found does it return `[]`.
- **No validation that proposed `id`s exist in `nodes`.** The parser accepts any `"id"` string the LLM returns, including hallucinated IDs. `format_proposal_markdown` then displays `**Current text:** ` (empty) because `nodes.get(nid, {})` returns `{}`. Callers who care should filter proposals’ IDs against `nodes` before rendering.
- **No propagation of the last error.** If every batch fails, `propose_updates` still returns `[]` — the caller cannot distinguish "everything was fine" from "every batch errored" from that return value alone. Inspect stderr.

### Maintenance notes
- The command strings built in `format_proposal_markdown` only escape `"` in `proposed_text`. Backticks, `$`, `\`, and single quotes are passed through — safe only if the reviewer visually inspects before pasting. If this is ever piped into a shell automatically, it becomes an injection surface.
- `reasons/propose_update.py` and `reasons_lib/propose_update.py` are duplicated in the tree; changes here must be mirrored or the packages diverge.

## Topics to Explore

- [function] `reasons/propose_update.py:propose_updates` — The batch orchestration loop, including the `on_batch` callback inside the `try` block and its exception-swallowing behavior.
- [function] `reasons/propose_update.py:parse_update_proposals` — The `raw_decode` scanning strategy and enum-coercion rules; the contract that partial/redacted LLM output still yields usable proposals.
- [function] `reasons/llm.py:invoke_model` — The abstraction that actually talks to the model; how model names, timeouts, and retries are handled upstream of this module.
- [file] `reasons/cli.py` — Where `propose_updates` and `format_proposals_file` are wired together into the user-facing `reasons propose-updates` command and how cascade dicts are computed and passed in.
- [file] `tests/test_propose_update.py` — Concrete expectations for parsed proposals, default IN-selection, and markdown formatting.

## Beliefs

- `propose-updates-defaults-to-in-beliefs` — When `belief_ids` is `None`, `propose_updates` selects only nodes with `truth_value == "IN"`, iterating them in sorted order for determinism.
- `propose-updates-filters-unknown-ids` — When an explicit `belief_ids` list is passed, IDs not present in `nodes` are dropped before any LLM call.
- `batch-failures-do-not-abort-propose-updates` — Exceptions from `invoke_model`, parsing, or the `on_batch` callback are caught per batch and logged as `WARN` to stderr; remaining batches still execute.
- `parse-update-proposals-returns-first-non-empty-array` — `parse_update_proposals` returns immediately on the first `[` position that `raw_decode`s into a list with at least one valid `{id: ...}` dict, ignoring any later arrays.
- `parse-update-proposals-coerces-enum-fields` — Out-of-taxonomy `action`/`failure_mode`/`basis` values are downgraded to `"update"`/`""`/`"prior-knowledge"` respectively rather than rejected.
