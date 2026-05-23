# File: reasons_lib/contradictions.py

**Date:** 2026-05-11
**Time:** 12:59

## Purpose

`contradictions.py` is the **nogood detection module** — it finds sets of beliefs that are mutually inconsistent. In Doyle's TMS, a "nogood" is a set of assumptions that leads to a contradiction; this module implements that detection by delegating the logical analysis to an LLM rather than through formal proof.

It owns two strategies for contradiction detection:
1. **Random batching** (`detect_contradictions`) — shuffle all IN beliefs and send fixed-size batches to the LLM.
2. **Semantic clustering** (`detect_contradictions_semantic`) — group beliefs by embedding similarity first, then check within clusters, so topically related beliefs are more likely to be compared.

Both strategies share the same prompt template, parser, and output format.

## Key Components

### Constants

- **`CONTRADICTION_BATCH_SIZE = 50`** — Default number of beliefs per LLM call. Balances context window usage against coverage.

- **`CONTRADICTION_PROMPT`** — The system prompt defining four contradiction types (direct negation, incompatible properties, scope conflict, quantitative mismatch) and the exact output format (`### NOGOOD <id>`). This is a structured extraction prompt — the LLM is constrained to produce parseable output or "No contradictions detected."

### Functions

**`format_beliefs_for_contradiction_check(belief_ids, nodes)`** — Serializes beliefs into markdown bullet points for the prompt. Truncates text at 200 chars. Silently skips missing node IDs (the `if not node: continue` guard).

**`parse_contradiction_response(response, valid_ids=None)`** — Regex-based parser that extracts structured nogood records from LLM output. Contract:
- Input: raw LLM text, optional whitelist of valid belief IDs.
- Output: list of dicts `{id, claims, analysis, severity}`.
- Filters out any claim referencing a belief ID not in `valid_ids`.
- Drops any nogood with fewer than 2 valid claims (a contradiction needs at least two parties).

**`_flush_results(results, output_path, header_written)`** — Incremental writer. Delegates to `api.write_contradiction_plan`. The `header_written` flag tracks whether the file header has already been emitted, toggling between write-new-file and append modes. Returns the updated flag.

**`detect_contradictions(nodes, ...)`** — The random-batch strategy. Shuffles belief IDs to avoid ordering bias, partitions into batches of `batch_size`, and runs each through the LLM. Progress is printed to stderr. Failed batches emit a warning and are skipped (best-effort).

**`detect_contradictions_semantic(nodes, ...)`** — The cluster-based strategy. Uses `cluster.list_clusters` to group beliefs by embedding similarity, then runs contradiction detection within each cluster. Clusters with fewer than 2 beliefs are skipped. Large clusters are further sub-batched at `CONTRADICTION_BATCH_SIZE`.

## Patterns

**Prompt-parse loop**: The core pattern is (1) format data into a structured prompt, (2) call LLM, (3) regex-parse the response. This is repeated across batches/clusters. The prompt is carefully designed with an exact output schema so `_NOGOOD_PATTERN` can extract results reliably.

**Incremental output**: Results are flushed to disk after each batch via `_flush_results`, so partial results survive if later batches fail or the process is interrupted. The `header_written` boolean is threaded through as a poor-man's state flag.

**Best-effort with warnings**: Each batch/cluster is wrapped in try/except. Failures log to stderr and processing continues. This is appropriate — contradiction detection is advisory, not load-bearing.

**Shuffling for coverage**: `random.shuffle(belief_ids)` ensures that across repeated runs, different beliefs end up in the same batch. Since contradictions can only be detected between beliefs *in the same batch*, shuffling gives probabilistic coverage over the full pairwise space.

## Dependencies

**Imports:**
- `llm.invoke_model` — the LLM call abstraction (model routing, timeout handling)
- `api.write_contradiction_plan` — file output formatting (lazy import inside `_flush_results`)
- `cluster.list_clusters, DEFAULT_MODEL` — embedding-based clustering (lazy import inside `detect_contradictions_semantic`)

The two lazy imports keep the module importable without heavy dependencies (sentence-transformers, api internals) until those code paths are actually exercised.

**Imported by:**
- `reasons_lib/api.py` — wires contradiction detection into the CLI/API surface
- `tests/test_contradictions.py` — unit tests for the parser and detection functions

## Flow

For `detect_contradictions`:
1. Filter `nodes` to only IN beliefs (or intersect with provided `belief_ids`).
2. Shuffle the filtered list.
3. Partition into batches of `batch_size`.
4. For each batch: format → prompt → LLM → parse → filter invalid IDs → collect results → flush to file.
5. Return accumulated results.

For `detect_contradictions_semantic`:
1. Same filtering step.
2. Build a `{id: text}` dict and pass to `list_clusters` for embedding + clustering.
3. For each cluster with 2+ beliefs: sub-batch at `CONTRADICTION_BATCH_SIZE` → same format/prompt/parse/flush loop.
4. Return accumulated results.

## Invariants

- **Only IN beliefs are checked.** Both functions enforce `truth_value == "IN"` during filtering, even if the caller passes IDs of OUT beliefs.
- **Minimum 2 claims per nogood.** `parse_contradiction_response` drops any nogood with fewer than 2 valid claims — you can't have a contradiction with one belief.
- **Claim IDs must be valid.** When `valid_ids` is provided, claims referencing unknown IDs are stripped. This prevents the LLM from hallucinating belief IDs.
- **Belief text is capped at 200 chars** in the prompt to avoid blowing context windows on large belief descriptions.

## Error Handling

Errors are **swallowed at the batch level**. Each LLM call is wrapped in a bare `except Exception`, which prints a warning to stderr and continues to the next batch. This means:
- A single LLM timeout or API error doesn't abort the entire run.
- The caller receives whatever results *did* succeed — there's no indication of how many batches failed unless you watch stderr.
- The regex parser is tolerant of malformed output — `_NOGOOD_PATTERN` simply won't match, so garbled LLM responses produce zero results rather than crashes.

No exceptions are raised to the caller from either detection function.

## Topics to Explore

- [file] `reasons_lib/cluster.py` — The embedding and clustering logic that powers the semantic detection strategy
- [function] `reasons_lib/api.py:write_contradiction_plan` — How detected nogoods are serialized to the plan file format
- [file] `tests/test_contradictions.py` — Test coverage for the parser and detection functions, including edge cases for malformed LLM output
- [function] `reasons_lib/llm.py:invoke_model` — The LLM abstraction that handles model routing and timeouts
- [general] `nogood-to-retraction-pipeline` — How detected contradictions flow from this module's output into actual belief retraction via the TMS

## Beliefs

- `contradictions-only-checks-in-beliefs` — Both `detect_contradictions` and `detect_contradictions_semantic` filter to `truth_value == "IN"` before analysis, silently discarding any OUT beliefs passed via `belief_ids`.
- `contradictions-require-two-claims-minimum` — `parse_contradiction_response` drops any NOGOOD with fewer than 2 valid claims; single-claim nogoods are silently discarded.
- `contradictions-batch-errors-never-propagate` — LLM failures in individual batches are caught and logged to stderr; neither detection function raises exceptions to the caller.
- `contradictions-shuffle-for-coverage` — `detect_contradictions` shuffles belief IDs before batching so repeated runs sample different pairwise comparisons, since only beliefs in the same batch can be detected as contradictory.
- `contradictions-semantic-skips-singleton-clusters` — `detect_contradictions_semantic` skips any cluster containing fewer than 2 beliefs, as no pairwise contradiction is possible.

