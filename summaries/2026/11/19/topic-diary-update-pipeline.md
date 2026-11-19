---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: End-to-end stage wiring (propose → gate/accept) and failure semantics not visible in this artifact.

## End-to-end stage wiring: what is actually visible

Short version: **the artifact shows a read-only “propose” stage and read-only reporting/review stages. It does not show the gate/accept executor that turns a proposal into a database mutation.** The end-to-end path here is manual: `propose_updates()` emits proposals → markdown formatter prints suggested CLI commands → a human/external CLI handler applies them. The actual accept/gate policy, cascade recomputation, and mutation transaction are not in this artifact.

### 1. Propose stage is real and batched

`reasons/propose_update.py:291` defines `propose_updates(...)`. It:

- Selects target beliefs: if `belief_ids` is omitted, it uses all `truth_value == "IN"` nodes (`propose_update.py:297-301`).
- Batches them, default `PROPOSE_UPDATE_BATCH_SIZE = 10` (`propose_update.py:13`).
- Formats each belief with text/status/source/URL/stale reason/justifications/outlist/dependents via `format_belief_for_update()` (`propose_update.py:93`).
- Calls `invoke_model()` and parses the response with `parse_update_proposals()` (`propose_update.py:322-325`).
- Returns a list of proposal dicts only. **It never mutates the graph.**

The proposal schema is `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`. The `failure_mode` taxonomy is `smuggled-premise`, `unsupported-superlative`, `false-causal-link`, `domain-conflation`, `contradicted-by-source`, `stale` (`propose_update.py:15-22`). The `basis` taxonomy is `source-divergence`, `detected-contradiction`, `prior-knowledge` (`propose_update.py:24-28`).

The prompt says `source-divergence` is “mechanically verifiable, cheap to accept” and `prior-knowledge` “needs scrutiny” (`propose_update.py:51-54`). That is the only hint of an intended gate policy. **No code in this artifact enforces that policy.**

### 2. Gate/accept stage is missing or external

`format_proposal_markdown()` renders a proposal as markdown and emits suggested commands:

- update: `reasons update {nid} "..."` (`propose_update.py:255-257`)
- retract: `reasons retract {nid} --reason "{failure_mode}: {comment}"` (`propose_update.py:258-263`)

`format_proposals_file()` wraps those sections into a review document (`propose_update.py:268-288`). This is a **handoff to a human**, not an executor.

The provided CLI slice contains `cmd_retract()` and `cmd_assert()`, but no `cmd_propose_update`, `cmd_accept`, or `cmd_gate` handler is visible. The observations also show `propose_gate_accept_usages` with `total_count: 0`, so the usage search found no actual wiring for “propose”, “gate”, “gated”, or “accept” in the searched scope. That is strong evidence that the accept path lives outside this artifact or is not implemented here.

`reasons/report_gated.py` is also not a gate/accept stage. It is a read-only report generator: `report_gated()` consumes `gated_data`, `retracted_premises`, and `blocker_details` and returns markdown (`report_gated.py:160-188`). It does not call mutation APIs.

So the visible pipeline is:

```text
IN beliefs
  → propose_updates()
  → formatted proposal markdown with suggested commands
  → human runs reasons update / reasons retract
  → external CLI/API mutates graph
```

The last arrow is not visible in this artifact.

## Failure semantics that matter

### Propose-stage failures are partially swallowed

`propose_updates()` catches `Exception` per batch, prints a warning to stderr, and continues:

```python
try:
    response = invoke_model(...)
    results = parse_update_proposals(response)
    all_results.extend(results)
    if on_batch is not None:
        on_batch(all_results)
except Exception as e:
    print(f"  WARN: batch {batch_num} failed: {e}", file=sys.stderr)
```

See `propose_update.py:322-329`.

Consequences:

- A failed LLM call does **not** raise to the caller.
- Failed batches are skipped; successful batches still return partial results.
- If every batch fails, the function returns `[]`.
- If `on_batch` raises, that exception is caught as a “batch failed” warning **after** `all_results.extend(results)` has already happened. The results are retained, but the callback failure is swallowed.

### Parse failure is indistinguishable from “no proposals”

`parse_update_proposals()` returns `[]` when:

- no JSON array is found,
- JSON decoding fails at all `[` positions,
- parsed arrays contain no dict with an `id`,
- the LLM returns `[]`.

See `parse_update_proposals()` at `propose_update.py:164-204`.

This means downstream code cannot distinguish “model reviewed and found nothing” from “model returned garbage” or “parser failed.” That is a significant failure semantic.

### Proposal normalization is fail-open

`parse_update_proposals()` normalizes invalid values:

- invalid `action` → `"update"` (`propose_update.py:184-186`)
- invalid `failure_mode` → `""` (`propose_update.py:187-189`)
- invalid `basis` → `"prior-knowledge"` (`propose_update.py:190-192`)

It does **not** enforce prompt rules:

- retract must have `proposed_text = null` — not enforced.
- update `proposed_text` must be meaningfully different — not enforced.
- proposal `id` must exist in the graph — not enforced. `parse_update_proposals()` only checks `"id" in item` (`propose_update.py:181-183`), so hallucinated IDs can flow into markdown and suggested commands.

### Cascade impact is not computed by propose

`format_proposal_markdown()` accepts a `cascade` argument and can render retracted/restored dependents (`propose_update.py:229-253`). But `propose_updates()` never computes cascade data, and `format_proposals_file()` only accepts a precomputed `cascades` mapping (`propose_update.py:268`). So the markdown often says:

```text
**Cascade impact:** not computed
```

unless an external caller supplies cascade data. The propose stage therefore does not know what would happen if the proposal were accepted.

### Review-stage failures are also fail-open

`review_beliefs()` has the same per-batch exception-swallowing pattern (`review.py:261-268`). `parse_review_response()` defaults missing `valid`, `sufficient`, and `necessary` to `True` (`review.py:200-210`). `classify_defeat_reason()` catches LLM errors and returns `""` (`review.py:124-128`). This is a fail-open review path: missing/broken LLM output can look like a successful review unless the caller checks the raw results.

### Report-stage LLM failures are not swallowed

By contrast, `report_gated()` in LLM mode calls `generate_report()` → `invoke_model()` without a `try/except` (`report_gated.py:152-157`, `report_gated.py:167-172`). So an LLM failure in report generation propagates to the caller, unlike propose/review batch failures. Structured report mode (`_build_structured_report`, `report_gated.py:42-108`) is LLM-free but assumes keys like `gated_data["blocker_count"]` and `gated_data["gated_count"]` exist.

### What is not visible

The artifact does not reveal:

- whether accept is automatic for `basis=source-divergence` and manual for `prior-knowledge`,
- whether `failure_mode` affects accept gating,
- whether accept recomputes cascade impact before mutating,
- whether accept is transactional with cascade updates,
- whether duplicate proposals for the same belief are deduplicated,
- whether hallucinated/nonexistent IDs are rejected at accept time,
- whether retries/backoff exist around `invoke_model`,
- how `review-beliefs --auto-retract` invokes retraction in the CLI implementation (tests show it exists, but the handler is outside the provided slice).

## Topics to Explore

- [file] `reasons/cli.py` — Find the actual `propose-update`, `accept`, `report-gated`, and `review-beliefs` command handlers; the provided 600-line slice does not include them.
- [function] `reasons/api.py:apply_proposal` — If present, this is likely where accept/gate semantics, cascade recomputation, and DB mutation live; not visible in the artifact.
- [general] `cascade-computation` — Explore how retract/update cascade impact is computed before or during accept; propose-stage formatter accepts cascade data but does not produce it.
- [general] `gate-policy` — Determine whether `basis` (`source-divergence` vs `prior-knowledge`) or `failure_mode` controls auto-accept vs manual review; only prompt text hints at this.
- [file] `tests/test_propose_update.py` — Confirms propose-stage formatting/parsing behavior, but contains no end-to-end accept/gate test.

## Beliefs

- `propose-updates-is-read-only` — `propose_updates()` returns proposal dicts and never mutates the belief graph.
- `propose-stage-requires-external-accept` — `format_proposal_markdown()` emits suggested `reasons update`/`reasons retract` commands but does not execute them.
- `proposal-parse-failures-return-empty-list` — `parse_update_proposals()` returns `[]` for malformed LLM output, indistinguishable from an empty proposal set.
- `proposal-parser-does-not-validate-node-ids` — `parse_update_proposals()` accepts any item with an `id` key and does not check membership in the graph.
- `propose-batch-errors-are-swallowed` — `propose_updates()` catches per-batch exceptions, logs a warning, and returns partial results rather than raising.
