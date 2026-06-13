# llm

[Back to index](index.md)

The ftl-reasons system integrates large language models as subprocess workers for three core operations: interactive querying, batch belief derivation, and belief classification. The integration follows a defense-in-depth strategy where LLM output is treated as untrusted external input — bounded, validated, and isolated at multiple layers before it can affect the belief network.

## Subprocess Isolation and Input Handling

All LLM invocations run as subprocesses rather than in-process API calls. Prompts are passed via `subprocess.run(input=...)` rather than as command-line arguments, avoiding both shell injection and argument length limits (`llm-prompt-always-via-stdin`). To prevent recursive Claude Code entry, all LLM subprocess invocations strip the `CLAUDECODE` environment variable centrally in `invoke_model()`, a pattern inherited by every LLM-facing module (`llm-subprocess-isolation-prevents-recursion`).

## Multi-Model Support

The system supports multiple LLM backends including Claude, Gemini, and Ollama. Model resolution handles provider-specific identifier formats — for instance, `resolve_model_cmd("ollama:model:tag")` splits on the first colon only, preserving the `model:tag` portion as a valid Ollama identifier (`llm-resolve-ollama-preserves-inner-colons`). Post-processing is also model-aware: thinking-marker stripping (`Thinking...` / `...done thinking.`) applies only to Ollama output, leaving Claude and Gemini responses unmodified even if they happen to contain the same markers (`llm-thinking-strip-ollama-only`).

## Interactive Querying

The `ask` command provides tiered query modes. In dual mode, it makes up to three LLM calls — one for TMS synthesis, one for FTS RAG retrieval, and one to merge the results — short-circuiting to two calls if either retrieval path returns empty (`ask-dual-mode-makes-three-llm-calls`). When `no_synth=True`, the LLM is bypassed entirely and raw FTS5 search results are returned without invoking the Claude CLI (`no-synth-mode-skips-llm`, `ask-no-synth-bypasses-llm`).

Fault tolerance is built into the query path: `ask()` catches `TimeoutExpired` and `RuntimeError` from the LLM invocation and falls back to raw search results rather than propagating exceptions, guaranteeing a string return under all conditions (`ask-never-raises-on-llm-failure`).

## Belief Derivation Pipeline

LLM-driven belief derivation is bounded at every stage of the pipeline (`llm-mutations-are-bounded-end-to-end`). At the input layer, proposals undergo fail-soft filtering, Jaccard retraction guards prevent near-duplicate beliefs, and environment stripping isolates the LLM boundary. At the persistence layer, atomic load/save with write-flag gating ensures consistency. At the output layer, deterministic terminating BFS with lifecycle-aware traversal propagates truth values safely (`llm-driven-mutations-are-safely-bounded`).

An important caveat: while structural validation ensures that justification references exist and point to IN beliefs, the logical soundness of the inference itself — whether the derived conclusion actually follows from its antecedents — is validated only by the proposing LLM. No code-level check verifies logical validity (`derived-belief-soundness-is-llm-only`).

## Belief Classification

The `list_negative()` operation classifies beliefs using LLM analysis. When the LLM returns unparseable output, the function returns `count == 0` rather than raising an exception — graceful degradation over failure (`api-list-negative-graceful-on-malformed-llm`).

## Semantic Contradiction Detection

The `detect-contradictions` command with the `--semantic` flag uses a clustering-before-LLM strategy: beliefs are embedded via sentence-transformers, clustered with KMeans, and then each cluster is sent to the LLM as a batch. This ensures topically related beliefs are analyzed together rather than being scattered across arbitrary batches of 50 (`semantic-contradictions-cluster-before-llm`).

## CLI Command Architecture

Roughly 21 CLI commands interact only with SQLite rather than the PostgreSQL API. These fall into three categories: filesystem-dependent commands (like `hash-sources` and `check-stale`), LLM-powered commands (like `derive`, `review-beliefs`, `detect-contradictions`, and `ask`), and bulk import/sync operations. All require either local file access or load-entire-network-modify-save semantics incompatible with per-operation transaction models (`sqlite-only-commands-are-filesystem-llm-or-bulk`).

## Retracted Integration Claims

Several higher-level beliefs about comprehensive LLM safety properties have been retracted (OUT), including claims about production-hardened integration (`llm-integration-is-production-hardened`), defense-in-depth across layers (`llm-integration-is-defense-in-depth-across-layers`), multi-granular fault tolerance (`llm-fault-tolerance-is-multi-granular`), and a complete LLM-driven quality lifecycle (`review-completes-llm-quality-lifecycle`). These retractions typically cascaded from underlying beliefs losing support — for example, changes to the derive pipeline's coverage claims or the ask module's bounded-execution guarantees. The concrete, premise-level observations about subprocess isolation, fault handling, and model-specific behavior remain held (IN), while the broad architectural conclusions drawn from them have been withdrawn.
