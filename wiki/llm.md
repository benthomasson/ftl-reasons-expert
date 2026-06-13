# llm

[Back to index](index.md)

### all-llm-interactions-are-bounded-and-fail-soft
**Status:** OUT

All LLM-facing operations apply consistent defensive patterns across both interactive (ask) and batch (derive) paths: bounded execution (iteration caps, timeout handling), fail-soft error recovery (fallback to raw results or skipped proposals), and environment isolation (stripping recursive invocation variables).

**Depends on:** [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded), [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive)
**Supports:** [all-llm-operations-are-defensively-bounded](llm.md#all-llm-operations-are-defensively-bounded), [llm-integration-is-bounded-fail-soft-and-process-isolated](llm.md#llm-integration-is-bounded-fail-soft-and-process-isolated)

### all-llm-operations-achieve-coverage-and-fault-tolerance
**Status:** OUT

All LLM-driven knowledge operations achieve both complete coverage and fault tolerance: the derive pipeline provides safe, complete, production-ready derivation with exhaustive exploration and accurate budget allocation, while interactive queries via ask are fault-tolerant, execution-bounded, and gracefully degrading — no LLM-facing operation can crash, hang, produce unbounded output, or corrupt the network.

**Depends on:** [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded), [derive-pipeline-has-complete-coverage](derive.md#derive-pipeline-has-complete-coverage)
**Supports:** [llm-integration-is-production-hardened](llm.md#llm-integration-is-production-hardened), [reasoning-and-knowledge-expansion-are-both-exhaustive](other.md#reasoning-and-knowledge-expansion-are-both-exhaustive)

### all-llm-operations-are-defensively-bounded
**Status:** OUT

All three LLM-facing operations — interactive query (ask), batch derivation (derive), and belief classification (list_negative) — apply consistent defensive patterns: bounded execution, fail-soft error handling, hallucination filtering, and graceful degradation on LLM unavailability.

**Depends on:** [all-llm-interactions-are-bounded-and-fail-soft](llm.md#all-llm-interactions-are-bounded-and-fail-soft), [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded)
**Supports:** [llm-integration-is-defense-in-depth-across-layers](llm.md#llm-integration-is-defense-in-depth-across-layers)

### api-list-negative-graceful-on-malformed-llm
**Status:** IN

When the LLM returns unparseable output, `list_negative()` returns `count == 0` rather than raising an exception — graceful degradation over failure.

**Supports:** [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded), [list-negative-parser-is-fully-resilient](other.md#list-negative-parser-is-fully-resilient)

### ask-dual-mode-makes-three-llm-calls
**Status:** IN

Dual mode makes up to 3 LLM calls (1 TMS synthesis + 1 FTS RAG + 1 merge), short-circuiting to 2 if either retrieval path returns empty.


### ask-never-raises-on-llm-failure
**Status:** IN

`ask()` catches `TimeoutExpired` and `RuntimeError` from `_invoke_claude()` and returns raw FTS5 search results instead of propagating the exception — the function guarantees a string return.

**Supports:** [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded)

### ask-no-synth-bypasses-llm
**Status:** IN

When `no_synth=True`, `ask()` returns raw `api.search()` results without invoking the Claude CLI.

**Supports:** [ask-has-tiered-query-modes](other.md#ask-has-tiered-query-modes)

### batch-fault-isolation-is-universal-across-llm-operations
**Status:** OUT

Both LLM-facing batch operations — derive proposal application (try/except per proposal with error accumulation) and belief review (silent skip on per-batch LLM failure) — isolate faults at the individual item level, preventing any single bad item from aborting the entire batch.

**Depends on:** [derive-apply-isolates-per-proposal-errors](derive.md#derive-apply-isolates-per-proposal-errors), [review-batch-failure-is-silent-skip](other.md#review-batch-failure-is-silent-skip)
**Supports:** [llm-fault-tolerance-is-multi-granular](llm.md#llm-fault-tolerance-is-multi-granular)

### derived-belief-soundness-is-llm-only
**Status:** IN

Structural validation ensures justification references exist and are IN, but the logical soundness of the inference from antecedents to derived conclusion is validated only by the proposing LLM — no code-level check verifies that the reasoning step is logically valid.

**Supports:** [derived-belief-pipeline-achieves-code-enforced-quality](other.md#derived-belief-pipeline-achieves-code-enforced-quality)

### llm-belief-operations-span-creation-and-classification
**Status:** OUT

All LLM-driven belief operations — creation via derive (with safety, completeness, and resource efficiency) and classification via list-negative (with defensive bounding and batch scaling) — share consistent defensive patterns across the complete belief quality lifecycle.

**Depends on:** [derive-pipeline-is-safe-complete-and-efficient](derive.md#derive-pipeline-is-safe-complete-and-efficient), [list-negative-is-bounded-and-batch-scalable](other.md#list-negative-is-bounded-and-batch-scalable)
**Supports:** [review-completes-llm-quality-lifecycle](llm.md#review-completes-llm-quality-lifecycle), [review-driven-quality-lifecycle-is-fully-code-enforced](lifecycle.md#review-driven-quality-lifecycle-is-fully-code-enforced)

### llm-belief-pipeline-is-fully-quality-enforced
**Status:** OUT

The system's complete LLM-driven belief pipeline — both generation (derive with safety, completeness, and coverage) and classification (list_negative with defensive bounding) — achieves fully code-enforced quality constraints at every stage, provided minimum-antecedent validation moves from prompt-only to code-enforced.

**Depends on:** [derive-pipeline-has-complete-coverage](derive.md#derive-pipeline-has-complete-coverage), [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded)

### llm-driven-mutations-are-safely-bounded
**Status:** IN

LLM-driven belief derivation is safely bounded by defense in depth: the derive pipeline validates proposals with fail-soft filtering, Jaccard retraction guards, and environment stripping at the LLM boundary, while the API layer enforces atomic load/save with write-flag gating and dict-only returns at the persistence boundary — malformed or adversarial LLM output cannot corrupt the network.

**Depends on:** [api-layer-ensures-atomic-isolated-mutations](other.md#api-layer-ensures-atomic-isolated-mutations), [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive)
**Supports:** [all-external-inputs-safely-integrated](external.md#all-external-inputs-safely-integrated), [llm-mutations-are-bounded-end-to-end](llm.md#llm-mutations-are-bounded-end-to-end)

### llm-fault-tolerance-is-multi-granular
**Status:** OUT

LLM fault tolerance operates at two independent granularities: module-level fail-soft handling ensures entire operations degrade gracefully when the LLM is unavailable, while item-level batch fault isolation ensures individual failures within derive and review batches are contained without affecting other items in the same batch.

**Depends on:** [batch-fault-isolation-is-universal-across-llm-operations](llm.md#batch-fault-isolation-is-universal-across-llm-operations), [llm-integration-fails-softly-across-modules](llm.md#llm-integration-fails-softly-across-modules)

### llm-integration-fails-softly-across-modules
**Status:** OUT

All LLM-facing modules apply consistent fail-soft error handling: the ask module always returns a string even when the LLM is unavailable, and the derive pipeline accumulates per-proposal errors rather than raising exceptions — no LLM failure path crashes the system.

**Depends on:** [ask-always-returns-string](other.md#ask-always-returns-string), [derive-fail-soft-validation](derive.md#derive-fail-soft-validation), [list-negative-is-defensively-bounded](other.md#list-negative-is-defensively-bounded), [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant)
**Supports:** [llm-fault-tolerance-is-multi-granular](llm.md#llm-fault-tolerance-is-multi-granular), [self-correction-is-resilient-to-llm-unavailability](self.md#self-correction-is-resilient-to-llm-unavailability)

### llm-integration-is-bounded-fail-soft-and-process-isolated
**Status:** OUT

LLM integration achieves three independent safety properties across all modules: execution bounds (iteration caps and timeouts), fail-soft error handling (always returns usable results on failure), and process isolation (subprocess invocation with CLAUDECODE environment stripping to prevent recursive entry)

**Depends on:** [all-llm-interactions-are-bounded-and-fail-soft](llm.md#all-llm-interactions-are-bounded-and-fail-soft), [ask-strips-claudecode-env](other.md#ask-strips-claudecode-env), [derive-strips-claudecode-env](derive.md#derive-strips-claudecode-env)
**Supports:** [llm-integration-is-production-hardened](llm.md#llm-integration-is-production-hardened)

### llm-integration-is-defense-in-depth-across-layers
**Status:** OUT

All LLM integration achieves defense-in-depth across two independent layers: application-level defensive bounding provides iteration caps, fail-soft error handling, Jaccard retraction guards, and hallucination filtering across all LLM-facing operations, while infrastructure-level process isolation executes all LLM calls through subprocess boundaries with CLAUDECODE environment scrubbing to prevent recursive invocation — ensuring safety at both the semantic and process boundaries.

**Depends on:** [all-external-execution-is-subprocess-isolated](external.md#all-external-execution-is-subprocess-isolated), [all-llm-operations-are-defensively-bounded](llm.md#all-llm-operations-are-defensively-bounded)
**Supports:** [defense-in-depth-spans-llm-and-system-boundaries](spans.md#defense-in-depth-spans-llm-and-system-boundaries)

### llm-integration-is-production-hardened
**Status:** OUT

LLM integration achieves production-grade robustness across all dimensions: infrastructure-level safety through bounded execution, fail-soft error handling, and process isolation prevents runaway or recursive invocations, while operational-level coverage and fault tolerance ensures both derive (batch) and ask (interactive) paths complete successfully or degrade gracefully under all failure modes.

**Depends on:** [all-llm-operations-achieve-coverage-and-fault-tolerance](llm.md#all-llm-operations-achieve-coverage-and-fault-tolerance), [llm-integration-is-bounded-fail-soft-and-process-isolated](llm.md#llm-integration-is-bounded-fail-soft-and-process-isolated)
**Supports:** [external-integration-is-hardened-and-boundary-controlled](external.md#external-integration-is-hardened-and-boundary-controlled), [knowledge-expansion-is-exhaustive-within-hardened-boundaries](other.md#knowledge-expansion-is-exhaustive-within-hardened-boundaries)

### llm-mutations-are-bounded-end-to-end
**Status:** IN

LLM-driven belief derivation is bounded at every stage of the pipeline: input validation (fail-soft filtering, Jaccard retraction guard, environment isolation), atomic persistence (context-managed load/save), and output propagation (deterministic terminating BFS with lifecycle-aware traversal).

**Depends on:** [llm-driven-mutations-are-safely-bounded](llm.md#llm-driven-mutations-are-safely-bounded), [propagation-is-safe-and-terminating](safe.md#propagation-is-safe-and-terminating)
**Supports:** [all-external-belief-paths-are-safely-bounded](external.md#all-external-belief-paths-are-safely-bounded), [all-external-inputs-produce-correct-state](external.md#all-external-inputs-produce-correct-state)

### llm-prompt-always-via-stdin
**Status:** IN

Prompts are passed to LLM subprocesses via `subprocess.run(input=...)`, never as command-line arguments — avoiding shell injection and argument length limits.


### llm-resolve-ollama-preserves-inner-colons
**Status:** IN

`resolve_model_cmd("ollama:model:tag")` splits on the first colon only, keeping `model:tag` intact as the Ollama model identifier.


### llm-subprocess-isolation-prevents-recursion
**Status:** IN

All LLM subprocess invocations strip the CLAUDECODE environment variable to prevent recursive Claude Code entry, enforced centrally in invoke_model() and inherited by all LLM-facing modules (ask, derive, review).

**Depends on:** [invoke-model-strips-claudecode-env](other.md#invoke-model-strips-claudecode-env)
**Supports:** [all-external-execution-is-subprocess-isolated](external.md#all-external-execution-is-subprocess-isolated)

### llm-thinking-strip-ollama-only
**Status:** IN

Thinking-marker stripping (`Thinking...` / `...done thinking.`) is applied only to Ollama model output; Claude and Gemini output passes through unmodified even if it contains the same markers.


### no-synth-mode-skips-llm
**Status:** IN

When `no_synth=True`, `ask()` returns formatted FTS5 search results without invoking `_invoke_claude()` — the LLM is never contacted.


### review-completes-llm-quality-lifecycle
**Status:** OUT

The LLM-driven belief quality lifecycle is complete across all phases: creation via derive (safe, complete, efficient), classification via list-negative (bounded, batch-scalable), and quality evaluation via review (scoped to derived beliefs, mutation-safe, fault-tolerant) — covering belief genesis, categorization, and ongoing quality assessment with no unmonitored phase.

**Depends on:** [llm-belief-operations-span-creation-and-classification](llm.md#llm-belief-operations-span-creation-and-classification), [review-pipeline-is-scoped-and-mutation-safe](safe.md#review-pipeline-is-scoped-and-mutation-safe)
**Supports:** [quality-lifecycle-is-complete-and-resource-efficient](lifecycle.md#quality-lifecycle-is-complete-and-resource-efficient)

### semantic-contradictions-cluster-before-llm
**Status:** IN

The `--semantic` flag on `detect-contradictions` embeds beliefs via sentence-transformers, clusters with KMeans via `list_clusters()`, and sends each cluster to the LLM as a batch — ensuring topically related beliefs are analyzed together instead of being scattered across random batches of 50.


### sqlite-only-commands-are-filesystem-llm-or-bulk
**Status:** IN

The ~21 SQLite-only CLI commands fall into three categories: filesystem-dependent (hash-sources, check-stale, add-repo), LLM-powered (derive, review-beliefs, detect-contradictions, ask), and bulk import/sync — all requiring either local file access or load-entire-network-modify-save semantics incompatible with PgApi's per-operation transaction model.

