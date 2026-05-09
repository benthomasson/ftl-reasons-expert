# File: reasons_lib/contradictions.py

**Date:** 2026-05-08
**Time:** 14:04

## Purpose

`contradictions.py` is the LLM-powered contradiction detector for the belief network. Its single responsibility: take a set of currently-held (IN) beliefs, send them to an LLM in batches, and parse back structured "nogood" proposals — sets of beliefs that cannot all be true simultaneously. It doesn't *record* nogoods into the TMS; it only *detects and returns* them. The caller (`api.py`) decides what to do with the results.

This is the belief network's consistency auditor. In Doyle's TMS, nogoods are sets of assumptions whose conjunction leads to a contradiction. This module automates finding them using an LLM as the inference engine rather than a formal proof system.

## Key Components

### Constants

- **`CONTRADICTION_BATCH_SIZE = 50`** — Default number of beliefs per LLM call. Balances context window usage against the LLM's ability to reason about belief interactions.

- **`CONTRADICTION_PROMPT`** — The structured prompt template. Defines four contradiction types (direct negation, incompatible properties, scope conflict, quantitative mismatch) and specifies the exact `### NOGOOD` output format. Critically, it instructs the LLM to *not* flag tensions or tradeoffs — only genuine logical incompatibilities.

### Functions

**`format_beliefs_for_contradiction_check(belief_ids, nodes) -> str`**
Serializes beliefs into a markdown-style list for the prompt. Truncates belief text at 200 characters to stay within context budgets. Skips IDs that don't exist in `nodes`. Contract: returns a newline-joined string of `- \`id\`: text` lines.

**`parse_contradiction_response(response, valid_ids=None) -> list[dict]`**
Parses the LLM's raw text output into structured nogood proposals. Uses `_NOGOOD_PATTERN` (a compiled regex) to find `### NOGOOD <id>` blocks, then line-parses each block for Claims, Analysis, and Severity fields. Two filtering gates:
1. If `valid_ids` is provided, claims referencing unknown belief IDs are stripped.
2. Any nogood with fewer than 2 remaining claims is dropped entirely.

Returns a list of dicts: `{id, claims, analysis, severity}`.

**`detect_contradictions(nodes, belief_ids=None, model="claude", timeout=300, batch_size=50) -> list[dict]`**
The main entry point. Orchestrates the full pipeline:
1. Selects which beliefs to check (all IN beliefs, or a caller-specified subset filtered to IN only).
2. Shuffles the belief list randomly.
3. Splits into batches of `batch_size`.
4. For each batch: formats, prompts the LLM, parses results.
5. Aggregates and returns all discovered nogoods.

## Patterns

**Structured LLM output parsing** — The prompt defines a rigid output format (`### NOGOOD id` + bullet fields), and the code uses regex + line parsing to extract structured data. This is a common pattern in this codebase for getting machine-readable results from LLMs without requiring JSON mode.

**Batch processing with progress** — Large belief sets are chunked into fixed-size batches with stderr progress reporting. This prevents context window overflow and gives the user visibility into long runs.

**Random shuffling** — Beliefs are shuffled before batching. This means different runs will produce different batch compositions, increasing the chance of catching contradictions between beliefs that wouldn't land in the same batch under sorted ordering. A simple but effective strategy given that contradictions can exist between any pair.

**Defensive filtering** — The parser validates claim IDs against a known-good set and enforces a minimum cardinality of 2. This guards against LLM hallucination of non-existent belief IDs.

## Dependencies

**Imports:**
- `random` — shuffling belief order across batches
- `re` — compiled regex for parsing LLM output
- `sys` — stderr output for progress/warnings
- `.llm.invoke_model` — the single LLM interface; all model calls go through this

**Imported by:**
- `reasons_lib/api.py` — the API layer that wires contradiction detection into CLI commands
- `tests/test_contradictions.py` — unit tests for parsing and formatting

The module has no dependency on `storage.py` or the SQLite layer. It operates purely on in-memory dicts (`nodes`), making it easy to test in isolation.

## Flow

```
detect_contradictions(nodes)
  │
  ├─ Filter nodes to IN beliefs only
  ├─ Shuffle the list
  ├─ Split into batches of 50
  │
  └─ For each batch:
       ├─ format_beliefs_for_contradiction_check() → text block
       ├─ Interpolate into CONTRADICTION_PROMPT
       ├─ invoke_model() → raw LLM response
       ├─ parse_contradiction_response() → list of nogood dicts
       └─ Append to accumulated results
  │
  └─ Return all nogood dicts
```

Data flows as: `dict nodes → filtered ID list → text prompt → LLM string → parsed dicts`. Each transformation is a pure function except `invoke_model`.

## Invariants

1. **Only IN beliefs are checked.** Even if the caller passes specific IDs, they're filtered to `truth_value == "IN"`. You cannot run contradiction detection on OUT beliefs.
2. **Every returned nogood has at least 2 valid claims.** The parser enforces `len(claims) >= 2` after ID validation.
3. **Claim IDs in results are a subset of the input belief IDs.** The `valid_ids` filter ensures no hallucinated IDs leak through.
4. **Batch boundaries are non-overlapping.** Each belief appears in exactly one batch per run. This means contradictions between beliefs in *different* batches will not be detected in a single run — a fundamental limitation of the batching approach.

## Error Handling

Errors are **swallowed at the batch level**. If `invoke_model` raises (timeout, API error, malformed response), the batch is skipped with a `WARN` message to stderr, and processing continues with the next batch. This means a partial failure still returns whatever contradictions were found in successful batches. The caller gets no indication of which batches failed — only the stderr log captures it.

The parser silently drops malformed nogood blocks (missing claims, invalid IDs, fewer than 2 claims). No exceptions are raised for parse failures.

## Topics to Explore

- [function] `reasons_lib/llm.py:invoke_model` — The LLM abstraction layer that all model calls route through; understanding its retry/timeout behavior matters for contradiction detection reliability
- [file] `tests/test_contradictions.py` — Unit tests that exercise parsing edge cases and format_beliefs; shows what the module's contracts are tested against
- [function] `reasons_lib/api.py:detect_contradictions` — The API wrapper that calls this module and decides how to record discovered nogoods into the TMS
- [general] `cross-batch-contradiction-coverage` — The random shuffle helps but doesn't guarantee cross-batch pairs are ever compared; understanding this limitation matters for tuning batch_size
- [file] `reasons_lib/network.py` — Provides the `nodes` dict format that this module consumes; defines the schema for `truth_value`, `text`, and node structure

## Beliefs

- `contradictions-only-checks-in-beliefs` — `detect_contradictions` filters all input to `truth_value == "IN"` before processing; OUT beliefs are never sent to the LLM
- `contradictions-swallows-batch-errors` — If an LLM call fails for a batch, the error is logged to stderr and the batch is skipped; no exception propagates to the caller
- `contradictions-shuffle-prevents-deterministic-batching` — Belief IDs are randomly shuffled before batching so that repeated runs cover different pairwise combinations across batch boundaries
- `contradictions-min-two-claims-per-nogood` — The parser enforces that every returned nogood has at least 2 valid claim IDs; single-claim or empty results are silently dropped
- `contradictions-no-storage-dependency` — This module operates on in-memory node dicts and has no import of `storage.py` or any database layer

