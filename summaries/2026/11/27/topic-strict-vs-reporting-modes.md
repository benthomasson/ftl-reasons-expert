---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Decide whether verification/backfill need an optional strict mode that raises instead of reporting.

## Recommendation

Yes: add an **optional, non-default strict mode**, but scope it carefully.

- **Verification:** high value. The current code can silently degrade when the LLM response is malformed or incomplete.
- **Backfill:** useful but lower urgency. Backfill is deterministic; strict mode mainly changes “report missing work” into a nonzero/exception outcome for automation.
- **Do not make strict the default.** Existing tests and CLI behavior expect reporting and exit code `0` for normal verify/backfill paths.

The cleanest implementation is to put strict behavior in `reasons/api.py`, then expose a `--strict` flag in the CLI. `reasons/cli.py` describes itself as thin wrappers around `reasons.api`, so exception policy should live below the CLI.

## Why verification needs it

Verification currently reports rather than raises in several important places:

- `reasons/verify.py` `parse_verify_response` returns `{}` when:
  - the response is empty,
  - no JSON object is found,
  - JSON parsing fails,
  - entries lack a valid `verdict`.
- `tests/test_verify.py` explicitly pins this behavior: `test_malformed` expects `{}`, and `test_missing_verdict_skipped` expects invalid entries to be skipped silently.
- `reasons/verify.py:132` `verify_beliefs` just formats beliefs and returns `parse_verify_response(response)`. It has no strict/failure path.
- `reasons/api.py:3807` imports `read_source, verify_beliefs`, but the observation does not include the body of `api.verify_belief`.
- `reasons/cli.py:2212` defines `cmd_verify`; this is where the CLI formats verification results.

So an automated caller can get an empty or partial verification result without an exception. That is risky in CI.

A good strict mode for verification would raise a dedicated exception, e.g. `VerificationError`, when verification cannot be completed:

- LLM response is unparseable,
- a checked belief is missing from the parsed result,
- an invalid verdict is returned,
- a required source file is unavailable,
- a belief expected to be verified has no usable result.

I would **not** make `STALE`, `PARTIAL`, or `INCONCLUSIVE` raise by default. Those are valid verification outcomes. If the project wants “fail unless everything is CONFIRMED,” that should be a separate CLI policy like `--fail-on-stale`, not overloaded into “strict mode.”

Also, keep `parse_verify_response` behavior as-is. It is a pure parser with direct tests. Strictness belongs at the orchestration layer (`verify_beliefs` / `api.verify_belief`), not in the parser.

## Why backfill is a weaker but still reasonable candidate

Backfill currently report-oriented:

- `reasons/api.py:2748` defines `backfill_hashes(db_path: str = DEFAULT_DB) -> dict`.
- It imports `_backfill` from `reasons.merkle` and returns `_backfill(net)` at `reasons/api.py:2752`.
- `reasons/merkle.py:130` defines `backfill_hashes(net)`, but the observation does not include its body.
- `reasons/cli.py:1183` defines `cmd_backfill_hashes`; it calls `api.backfill_hashes(db_path=args.db)`.
- `reasons/cli.py:3527` defines the `backfill-hashes` subparser.
- `reasons/cli.py:1172` shows the `check` command reporting: `Missing hashes: ... (run 'reasons backfill-hashes' to compute)`.
- Tests such as `tests/test_merkle.py:142` and `tests/test_cli.py:570` exercise backfill as a reporting operation.

Because backfill is deterministic, strict mode is less critical than for verification. But it is still useful for automation: `reasons backfill-hashes --strict` should exit nonzero / raise if hashes remain missing or if backfill reports errors.

The exact backfill strict conditions cannot be finalized from the observations because `reasons/merkle.py:130` and the full `api.backfill_hashes` body are not shown. We would need to inspect what `_backfill(net)` returns and whether it surfaces `missing_hashes`, `errors`, or per-node failures.

## Proposed shape

Add optional API parameters similar to:

- `api.verify_belief(..., strict: bool = False)`
- `api.backfill_hashes(..., strict: bool = False)`

Then add CLI flags:

- `reasons verify ... --strict`
- `reasons backfill-hashes ... --strict`

Default behavior stays exactly as today. Strict mode raises a purpose-built exception from the API; the CLI catches it, prints the error, and exits `1`.

## Existing strict patterns

There is no existing strict-mode pattern to reuse. The search for `strict` found only unrelated text, such as access-tag restriction tests and prompt wording in `reasons/repair.py` and `reasons/review.py`. So this would be a new convention.

## Topics to Explore

- [function] `reasons/merkle.py:backfill_hashes` — Need the body to know what result schema and failure modes strict backfill should check.
- [function] `reasons/api.py:backfill_hashes` — The wrapper at line 2748 is the natural place to add an optional `strict` parameter.
- [function] `reasons/cli.py:cmd_verify` — Need the full command body to see how verification results map to output and exit codes.
- [function] `reasons/verify.py:parse_verify_response` — Central silent-failure point for malformed or incomplete LLM responses.
- [general] `strict-mode-exception-contract` — Decide whether to introduce `VerificationError` / `BackfillError` or reuse `ValueError`.

## Beliefs

- `parse-verify-response-returns-empty-dict-on-malformed-json` — `parse_verify_response` returns `{}` for empty or malformed LLM responses instead of raising, and `tests/test_verify.py` asserts this behavior.
- `verify-belief-raises-keyerror-for-missing-node` — `api.verify_belief` raises `KeyError` for a nonexistent node, as covered by `test_missing_node_raises`.
- `backfill-hashes-is-wrapped-by-api-and-cli` — `reasons/api.py:2748` wraps `reasons/merkle.py:130`, and `reasons/cli.py:1183` exposes it through the `backfill-hashes` command.
- `strict-mode-does-not-exist-in-current-code` — Searching for `strict` found no verification/backfill strict flag or exception path; matches were unrelated prompt text and access-tag tests.
- `backfill-check-reports-missing-hashes` — `reasons/cli.py:1172` reports missing hashes and instructs the user to run `reasons backfill-hashes`, rather than failing automatically.
