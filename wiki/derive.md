# derive

[Back to index](index.md)

The derive pipeline is the LLM-driven inference engine of the belief network, responsible for generating new derived beliefs from existing ones. It constructs prompts containing a budget-constrained subset of current beliefs, sends them to an LLM, parses the proposed derivations, validates them against multiple safety invariants, and applies the survivors to the network. The pipeline is designed around three principles: strategic flexibility in belief selection, deterministic reproducibility across runs, and defense-in-depth at every stage.

## Budget Allocation

Before the LLM can reason over beliefs, the pipeline must decide which beliefs to include in the prompt — a constrained optimization problem when the network is large.

The `_build_beliefs_section` function allocates prompt token budget proportionally to each agent's belief count, with a guaranteed floor of five beliefs per agent group (`derive-agent-budget-proportional`, `derive-budget-floor-five`). The local (non-agent) belief group receives the same five-slot minimum regardless of agent budget pressure (`derive-budget-local-floor-is-five`). This floor ensures proportional allocation never starves minority agents or local beliefs (`derive-budget-is-efficient-and-floor-bounded`).

The accumulation logic runs in O(N) time — a correction from an earlier quadratic bug (issue #23) where `count += len(belief_ids)` was incorrectly placed inside the per-belief loop (`derive-budget-count-is-linear`).

## Selection Strategies

The pipeline supports three strategies for choosing which beliefs fill the budget, all controlled by a single `max_beliefs` parameter (`derive-budget-three-strategies`):

- **Alphabetical truncation** (default) — beliefs sorted by ID, truncated at the budget limit
- **Random sampling** (`sample=True`) — random subset with a fixed seed for determinism
- **Semantic clustering** (`cluster=True`) — embedding-based grouping ensures topical diversity across the prompt

When using cluster-based selection, the pipeline achieves semantically-informed budget allocation with end-to-end determinism: sorted embedding order, fixed-seed clustering, and exact budget counts feed into reproducible prompt construction (`cluster-derive-is-semantically-informed-and-deterministic`). This resolves the tension between strategic flexibility and deterministic reproducibility — callers can trade off reproducibility, diversity, and semantic coherence without performance penalty (`derive-achieves-flexibility-with-reproducibility`).

## Prompt Construction and Reproducibility

Prompt construction is fully reproducible: deterministic sampling with fixed seeds selects consistent belief subsets, and accurate proportional budget allocation ensures each agent receives the same token share across runs (`derive-prompt-is-deterministic-and-reproducible`). The budget allocation's accuracy depends on the prompt format round-tripping cleanly through the parser — the `### DERIVE` / `### GATE` format serves as a shared contract between the LLM output, `parse_proposals()` input, and `write_proposals_file()` output, forming a closed serialization loop (`derive-prompt-roundtrips-through-parser`).

This round-trip property also enables a write-review-accept workflow: `write_proposals_file` output can be parsed back by `parse_proposals` with no data loss, so humans can edit proposals in the same format the parser reads (`derive-prompt-round-trippable`).

## Parsing

`parse_proposals` supports two format versions for backward compatibility (`derive-parse-supports-two-formats`). It tries the newer format first (v0.10+: `### DERIVE id`), falling back to the older format (v0.9: `### DERIVE: \`id\``) only when the new parser returns zero matches. This tolerance for format evolution is part of the system's broader approach to boundary compatibility (`system-tolerates-evolution-at-all-boundaries`).

## Validation

Before proposals reach the database, `validate_proposals` enforces several invariants:

- **Duplicate rejection** — any proposal whose belief ID already exists in the network is rejected, preventing overwrites (`derive-validate-rejects-duplicate-ids`).
- **Retraction guard** — any proposal whose ID has ≥50% Jaccard token overlap (tokenized on hyphens/colons) with an existing OUT belief is rejected, preventing re-derivation of previously retracted conclusions (`derive-retraction-guard-uses-jaccard`).
- **Fail-soft filtering** — invalid proposals are collected into a skipped list with reasons rather than raising exceptions (`derive-fail-soft-validation`).

Notably, the minimum-two-antecedents rule for derived beliefs is enforced only by the LLM prompt instructions, not validated in code (`derive-min-antecedents-is-prompt-only`). This is a deliberate design point tracked as part of the quality enforcement lifecycle.

## Application and Error Isolation

The apply stage implements defense-in-depth through two layers (`derive-apply-is-isolated-and-caller-validated`):

1. **Trust boundary** — callers must run `validate_proposals` before calling `apply_proposals`, which trusts its input unconditionally (`derive-validate-before-apply`).
2. **Per-proposal isolation** — `apply_proposals` wraps each `api.add_node()` call in try/except and accumulates `(proposal, error_string)` tuples, so one malformed proposal cannot abort or corrupt the batch (`derive-apply-isolates-per-proposal-errors`).

This batch fault isolation pattern is applied universally across all LLM-driven operations in the system (`batch-fault-isolation-is-universal-across-llm-operations`).

## Process Isolation

The derive command invokes LLMs by shelling out to `claude` or `gemini` CLI binaries via `asyncio.create_subprocess_exec`, not through any Python SDK (`derive-uses-subprocess-not-sdk`). Before spawning the model subprocess, `_derive_one_round` explicitly removes the `CLAUDECODE` environment variable to prevent recursive Claude Code invocation (`derive-strips-claudecode-env`).

## Resilience and Reporting

The pipeline uses return values rather than exceptions for flow control: `_derive_one_round` returns -1 on error, 0 on saturation, and a positive count on success, which the exhaust loop in `cmd_derive` uses to decide whether to continue (`derive-returns-negative-one-on-error`).

Partial results are persisted via JSON reports after each round through `_write_derive_report`, so crash recovery is possible from the last completed step (`derive-reports-survive-partial-runs`). The `--report-dir` flag writes a JSON report containing a `rounds` array where each entry records `proposals_found` and `added` counts (`derive-report-json-has-rounds-array`).

## End-to-End Fault Tolerance

These layers compose into end-to-end fault tolerance through three independent mechanisms (`derive-pipeline-achieves-end-to-end-fault-tolerance`):

- **Proactive defense** — fail-soft validation, Jaccard retraction guards, and environment isolation prevent bad states from forming.
- **Reactive resilience** — partial results persisted via JSON reports after each round, with error states signaled through return codes rather than exceptions.
- **Prompt reproducibility** — deterministic sampling with fixed seeds enables consistent re-runs after failures.

The pipeline also achieves both reproducibility and defense-in-depth at the application stage, ensuring runs are repeatable and any surviving bad proposal cannot corrupt sibling proposals (`derive-pipeline-is-reproducible-and-defense-in-depth`).

## Cycle Safety

The `_get_depth` function, used during derive to assess belief depth, guards against infinite recursion on cyclic justification chains by setting `memo[node_id] = 0` before recursing — cycles resolve to depth 0 (`derive-depth-cycle-guard`).
