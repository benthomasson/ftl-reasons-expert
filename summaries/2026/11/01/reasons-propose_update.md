---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/propose_update.py

# `reasons/propose_update.py` — Explanation

## 1. Purpose

This module is the **LLM-driven belief auditing engine** for the `ftl-reasons` Truth Maintenance System (TMS). Its job is to take existing beliefs (with their justification chains, sources, and dependents), send them to an LLM in batches, and collect back **structured proposals** for either updating or retracting those beliefs.

It does *not* apply changes. It produces a reviewable artifact (markdown + CLI commands) that a human can inspect and act on. The module owns:

- The prompt engineering for belief auditing
- The taxonomy of *why* a belief might be wrong (`FAILURE_MODES`) and *how* we know (`BASES`)
- Parsing LLM output into validated structured data
- Rendering proposals into human-readable markdown with ready-to-paste CLI commands

## 2. Key Components

### Constants & Taxonomies

| Name | Type | Contract |
|------|------|----------|
| `PROPOSE_UPDATE_BATCH_SIZE` | `int = 10` | Default number of beliefs per LLM call. Larger batches risk context overflow; smaller ones waste API calls. |
| `FAILURE_MODES` | `set[str]` | Closed set of 6 categories: `smuggled-premise`, `unsupported-superlative`, `false-causal-link`, `domain-conflation`, `contradicted-by-source`, `stale`. The parser **rejects** any value outside this set (coerces to `""`). |
| `BASES` | `set[str]` | Closed set of 3 origins: `source-divergence`, `detected-contradiction`, `prior-knowledge`. Defaults to `"prior-knowledge"` if the LLM returns something else. |
| `PROPOSE_UPDATE_PROMPT` | `str` (template) | The full system+user prompt. Contains `{beliefs}` placeholder. Uses `{{ }}` escaping for JSON braces inside the f-string. |

### Functions

**`format_belief_for_update(node_id, nodes) -> str`**
Builds the per-belief section of the prompt. Includes:
- Text, truth value, source, source URL, stale reason (from metadata)
- All justifications (antecedents, outlists, labels) — numbered if >1
- Dependents (resolved to their text, or marked `(not found)`)

Returns `""` if `node_id` isn't in `nodes` (defensive; shouldn't happen in normal flow).

**`parse_update_proposals(response) -> list[dict]`**
Defensive JSON extraction. Scans every `[` character position and attempts `json.JSONDecoder().raw_decode()`. This handles LLMs that wrap the array in prose. For each item it:
- Requires an `"id"` key (skips if absent)
- Coerces `action` to `"update"` if not in `{"update", "retract"}`
- Coerces `failure_mode` to `""` if not in `FAILURE_MODES`
- Coerces `basis` to `"prior-knowledge"` if not in `BASES`

Returns `[]` if no valid array is found.

**`format_proposal_markdown(proposal, nodes=None, cascade=None) -> str`**
Renders one proposal as a markdown section. If `cascade` data is provided (from the TMS engine), it lists which dependents will go IN/OUT. Ends with a ready-to-run CLI command:
- `reasons update <id> "<text>"` for updates
- `reasons retract <id> --reason "<failure_mode>: <comment>"` for retractions

**`format_proposals_file(proposals, nodes=None, cascades=None) -> str`**
Concatenates all proposals into a single markdown document with a header and `---` separators.

**`propose_updates(nodes, belief_ids=None, model="claude", timeout=300, batch_size=10, on_batch=None) -> list[dict]`**
The main orchestrator:
1. Defaults `belief_ids` to all nodes with `truth_value == "IN"` (sorted)
2. Filters to IDs actually present in `nodes`
3. Iterates in batches, printing progress to `stderr`
4. For each batch: formats → prompts → `invoke_model()` → `parse_update_proposals()` → accumulates
5. Calls `on_batch(all_results)` after each batch if a callback is provided
6. Catches per-batch exceptions, logs to `stderr`, continues to next batch

## 3. Patterns

- **Batch-and-callback**: The `on_batch` parameter lets callers (e.g., `cli.py` for live progress, `api.py` for HTTP streaming) observe partial results without blocking on the full run.
- **Closed-enum validation via coercion**: Rather than raising on invalid LLM output, the parser silently maps unknown values to safe defaults. This is a pragmatic choice — the LLM is a non-deterministic producer, and we'd rather get *some* structured data than crash.
- **`raw_decode` scanning**: Instead of requiring the LLM to return *only* JSON, the parser scans for the first valid JSON array anywhere in the response. This is a well-known pattern for LLM output parsing.
- **Presentation separated from logic**: `format_*` functions are pure string builders with no side effects. They can be unit-tested without mocking an LLM.
- **Dual-package mirroring**: The repo has both `reasons/` and `reasons_lib/` copies. The `reasons_lib` version is likely the published library copy (see `reasons_lib/propose_update.py` in the importers list), while `reasons/` is the development package.

## 4. Dependencies

**This module imports:**
- `json` — for `raw_decode` parsing
- `sys` — for `sys.stderr` progress/warning output
- `.llm.invoke_model` — the single external LLM call abstraction

**This module is imported by:**
- `reasons/api.py` — likely exposes `propose_updates` as an HTTP/MCP endpoint
- `reasons/cli.py` — wires up the `reasons propose-update` CLI command
- `reasons_lib/propose_update.py` — the library-mirror copy (may re-export or extend)
- `tests/test_propose_update.py` — unit tests

The dependency surface is intentionally minimal: one LLM call function, no database, no network. This makes it easy to test by mocking `invoke_model`.

## 5. Flow

```
propose_updates(nodes, belief_ids)
│
├─ 1. Resolve belief_ids (default: all IN beliefs, sorted)
├─ 2. For each batch of 10:
│     │
│     ├─ format_belief_for_update() × 10  →  beliefs_text
│     ├─ PROPOSE_UPDATE_PROMPT.format(beliefs=beliefs_text)  →  prompt
│     ├─ invoke_model(prompt, model, timeout)  →  raw LLM string
│     ├─ parse_update_proposals(raw)  →  list[dict]  (validated)
│     ├─ all_results.extend(results)
│     └─ on_batch(all_results)  [if callback provided]
│
└─ 3. Return all_results
```

Downstream (in `cli.py` or `api.py`), the caller would then:
- Run the TMS cascade simulation to compute `cascades`
- Call `format_proposals_file(proposals, nodes, cascades)` → markdown
- Print or serve that markdown

## 6. Invariants

- **Only IN beliefs are audited by default.** OUT beliefs are already retracted; re-reviewing them is wasteful.
- **`action` ∈ {"update", "retract"}** — no other value survives parsing.
- **For `retract`: `proposed_text` must be `null`.** The prompt instructs this; the parser doesn't enforce it (it just passes through `item.get("proposed_text")`).
- **For `update`: `proposed_text` should be meaningfully different** (enforced by the prompt, not by code).
- **`failure_mode` and `basis` are closed vocabularies.** The parser enforces membership; unknown values are silently replaced with defaults.
- **Batch size is a soft cap.** The loop uses `range(0, len, batch_size)` — the last batch may be smaller.
- **Belief IDs in proposals must correspond to IDs actually sent.** The parser doesn't validate this (it trusts the LLM to echo the correct ID), but `format_proposal_markdown` handles missing nodes gracefully.

## 7. Error Handling

| Scenario | Behavior |
|----------|----------|
| LLM call raises (timeout, network, API error) | Caught per-batch in `propose_updates`. Logged to `stderr` as `WARN: batch N failed: <e>`. Remaining batches still execute. Partial results are returned. |
| LLM returns prose around JSON | `parse_update_proposals` scans all `[` positions with `raw_decode`. Works as long as there's at least one valid JSON array. |
| LLM returns no JSON at all | `parse_update_proposals` returns `[]`. No crash. |
| LLM returns items without `"id"` | Silently skipped. |
| LLM returns invalid `failure_mode` / `basis` / `action` | Coerced to safe defaults (`""`, `"prior-knowledge"`, `"update"` respectively). |
| A node ID in dependents/antecedents isn't in `nodes` | Rendered as `(not found)` in the prompt. No exception. |
| `belief_ids` contains IDs not in `nodes` | Filtered out before processing. |

The philosophy: **never crash on a single bad batch or a single malformed LLM response.** Degrade gracefully to partial or empty results.

---

## Topics to Explore

- [file] `reasons/llm.py` — The `invoke_model` abstraction that this module depends on; understanding its retry/timeout/streaming semantics is critical for debugging batch failures
- [file] `reasons/api.py` — How `propose_updates` is exposed to external callers and how the `on_batch` callback is used for streaming responses
- [file] `tests/test_propose_update.py` — The test suite reveals expected edge cases (empty responses, malformed JSON, cascade formatting) and how `invoke_model` is mocked
- [function] `reasons/propose_update.py:parse_update_proposals` — The `raw_decode` scanning loop is the most fragile part; understanding its failure modes helps when the LLM changes its output format
- [general] `tms-cascade-simulation` — The `cascade` parameter in `format_proposal_markdown` references IN/OUT propagation logic that lives elsewhere in the TMS; understanding it explains *why* proposals list affected dependents

---

## Beliefs

- `propose-update-only-audits-in-beliefs` — `propose_updates` defaults to filtering `belief_ids` to nodes where `truth_value == "IN"`; OUT beliefs are never sent to the LLM for re-audit
- `parse-coerces-invalid-taxonomy-values` — `parse_update_proposals` never raises on unknown `failure_mode`, `basis`, or `action` values; it silently substitutes `""`, `"prior-knowledge"`, and `"update"` respectively
- `batch-failure-is-non-fatal` — An exception in `invoke_model` for one batch is caught, logged to `stderr`, and the remaining batches still execute; the function returns partial results
- `propose-update-produces-human-reviewable-artifact` — The module's output contract is a markdown document containing ready-to-paste `reasons update` / `reasons retract` CLI commands, not a programmatic mutation of belief state
- `propose-update-depends-on-invoke-model-only` — The module's sole external runtime dependency (beyond stdlib) is `reasons.llm.invoke_model`; it has no database, network, or TMS-engine imports
