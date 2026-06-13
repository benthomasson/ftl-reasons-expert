# derive

[Back to index](index.md)

The derive pipeline is the LLM-driven belief generation subsystem of ftl-reasons. It constructs prompts from the existing belief network, sends them to an external model, parses the resulting proposals, validates them against safety invariants, and applies valid proposals to the database. The pipeline is designed around three core properties: reproducibility, fault tolerance, and defensive validation.

## LLM Invocation Model

The derive pipeline invokes language models by shelling out to `claude` or `gemini` CLI binaries via `asyncio.create_subprocess_exec`, rather than through any Python SDK (`derive-uses-subprocess-not-sdk`). To prevent recursive Claude Code invocation, `_derive_one_round` explicitly removes the `CLAUDECODE` environment variable before spawning the model subprocess (`derive-strips-claudecode-env`).

## Budget Allocation

When multiple agents contribute beliefs to the network, `_build_beliefs_section` allocates prompt token budget proportionally to each agent's belief count, with a floor of 5 beliefs per agent (`derive-agent-budget-proportional`). The non-agent (local) group receives the same minimum guarantee of 5 belief slots regardless of agent budget pressure (`derive-budget-local-floor-is-five`). This floor prevents proportional allocation from starving minority agents or the local group.

Agent belief counting accumulates linearly — O(N) in the number of beliefs shown per agent, not quadratically (`derive-budget-count-is-linear`). This corrected behavior followed the fix for issue #23, where `count += len(belief_ids)` had been placed inside the per-belief loop, inflating the count and shrinking the non-agent budget below its intended size (`derive-agent-count-bug`, OUT — fixed).

The proportional allocation produces correct per-agent token counts (`derive-budget-allocation-is-accurate`), a property that depends on both the per-agent budgeting logic and the round-trippable prompt format (`derive-prompt-roundtrips-through-parser`).

### Selection Strategies

The pipeline supports three budget strategies, all controlled by a single `max_beliefs` parameter (`derive-budget-three-strategies`):

- **Alphabetical truncation** (default) — deterministic ordering by belief ID
- **Random sampling** (`sample=True`) — uses a fixed seed for deterministic selection
- **Semantic clustering** (`cluster=True`) — embedding-based grouping that ensures topical diversity across the prompt

When using cluster-based selection, the pipeline achieves semantically-informed budget allocation with end-to-end determinism: sorted embedding order, fixed-seed clustering, and exact budget counts feed into reproducible prompt construction (`cluster-derive-is-semantically-informed-and-deterministic`). The three strategies resolve the tension between strategic flexibility and deterministic reproducibility — diverse exploration approaches with identical results across runs (`derive-achieves-flexibility-with-reproducibility`).

## Prompt Format and Round-Tripping

The `### DERIVE` / `### GATE` format serves as a shared contract between three components: the `DERIVE_PROMPT` LLM output, the `parse_proposals()` parser, and the `write_proposals_file()` serializer (`derive-prompt-roundtrips-through-parser`). This forms a closed serialization loop — `write_proposals_file` output can be parsed back by `parse_proposals` with no data loss, enabling a write-review-accept cycle where humans edit proposals in the same format the parser reads (`derive-prompt-round-trippable`).

The parser supports two format versions: the current `### DERIVE id` format (v0.10+) is tried first, falling back to the older `### DERIVE: \`id\`` format (v0.9) only when no new-format matches are found (`derive-parse-supports-two-formats`). This backward compatibility ensures the system tolerates format evolution at its boundaries.

## Validation and Safety

The derive pipeline applies multiple defensive layers before proposals reach the database (`derive-pipeline-is-defensive`).

### Fail-Soft Validation

`validate_proposals` filters invalid proposals into a skipped list with reasons rather than raising exceptions (`derive-fail-soft-validation`). This fail-soft approach means one malformed proposal never blocks the processing of valid siblings.

### Retraction Guards

Any proposed belief ID with Jaccard similarity >= 0.5 to an existing OUT node is rejected, preventing re-derivation of previously retracted conclusions (`derive-retraction-guard-uses-jaccard`). The similarity is computed by tokenizing IDs on hyphens and colons (`derive-validate-blocks-retracted-rediscovery`). Duplicate IDs — proposals whose belief ID already exists in the network — are also rejected, preventing overwrites through the derive pipeline (`derive-validate-rejects-duplicate-ids`).

### Minimum Antecedent Rule

The requirement that derived beliefs have at least two antecedents is enforced only by the LLM prompt instructions, not validated in code by `validate_proposals` (`derive-min-antecedents-is-prompt-only`). This is a notable gap in code-level enforcement — the pipeline trusts the model to comply with this structural constraint.

## Application Stage

The apply stage achieves defense-in-depth through a two-layer design (`derive-apply-is-isolated-and-caller-validated`). First, callers must run `validate_proposals` before calling `apply_proposals`, which trusts its input unconditionally (`derive-validate-before-apply`). Second, even if invalid proposals survive validation, `apply_proposals` wraps each `api.add_node()` call in independent error handling, accumulating `(proposal, error_string)` tuples so one malformed proposal cannot corrupt the batch (`derive-apply-isolates-per-proposal-errors`).

## Exhaustive Mode and Termination

The pipeline supports an `--exhaust` flag that enables automatic application of all discovered proposals across multiple rounds. Depth-based cycle detection with pre-recursion memoization prevents infinite loops even when justification chains are cyclic — `_get_depth` sets `memo[node_id] = 0` before recursing, so cycles resolve to depth 0 (`derive-depth-cycle-guard`).

## Error Handling and Resilience

`_derive_one_round` uses return values rather than exceptions for flow control: -1 signals an error, 0 indicates saturation (no new beliefs discoverable), and a positive integer reports the number of beliefs added (`derive-returns-negative-one-on-error`). The `cmd_derive` exhaust loop uses these codes to decide whether to continue, stop, or abort.

The pipeline persists partial results via JSON reports after each round. The `--report-dir` option writes a JSON report containing a `rounds` array with `proposals_found` and `added` counts per round (`derive-report-json-has-rounds-array`). Both `cmd_derive` and `cmd_review_beliefs` write these partial reports via `_write_derive_report`, so crash recovery is possible from the last completed step (`derive-reports-survive-partial-runs`).

Together, these mechanisms provide end-to-end fault tolerance through three independent layers: proactive defense (validation and guards), reactive resilience (persisted partial results and return-code signaling), and prompt reproducibility enabling consistent re-runs after failures (`derive-pipeline-achieves-end-to-end-fault-tolerance`).

## Retracted Assessments

Several higher-level assessments of the derive pipeline have been retracted. The claim that the pipeline achieves "complete coverage" across safety, completeness, and production-readiness axes (`derive-pipeline-has-complete-coverage`, OUT) was retracted, as was the claim that all quality constraints are comprehensively code-enforced (`derive-quality-is-comprehensively-code-enforced`, OUT) — undermined by the prompt-only minimum-antecedent rule. The "fully assured" quadruple-assurance characterization (`derive-pipeline-is-reproducible-and-fully-assured`, OUT) was also retracted, along with its dependency on exhaustive-and-terminating completeness (`derive-pipeline-is-exhaustive-and-terminating`, OUT). These retractions reflect a more precise understanding of the pipeline's actual guarantees versus aspirational characterizations.
