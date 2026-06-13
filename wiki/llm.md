# llm

[Back to index](index.md)

LLM integration in the ftl-reasons system spans query synthesis, belief derivation, and contradiction detection. The architecture treats language models as untrusted subprocesses, wrapping every invocation in isolation boundaries and graceful degradation paths so that malformed or adversarial LLM output cannot corrupt the belief network.

## Subprocess Isolation

All LLM invocations run as external subprocesses rather than in-process library calls. Prompts are passed via `subprocess.run(input=...)`, never as command-line arguments — this avoids both shell injection vulnerabilities and argument length limits (llm-prompt-always-via-stdin). To prevent recursive Claude Code entry, `invoke_model()` centrally strips the `CLAUDECODE` environment variable before spawning any LLM subprocess, a safeguard inherited by all LLM-facing modules including `ask`, `derive`, and `review` (llm-subprocess-isolation-prevents-recursion).

### Multi-Model Support

The system supports multiple LLM backends with model-specific handling. When resolving Ollama model identifiers, `resolve_model_cmd("ollama:model:tag")` splits on the first colon only, preserving the `model:tag` portion intact as the Ollama model identifier (llm-resolve-ollama-preserves-inner-colons). Output post-processing is also model-aware: thinking-marker stripping (`Thinking...` / `...done thinking.`) is applied only to Ollama output, while Claude and Gemini output passes through unmodified even if it contains the same markers (llm-thinking-strip-ollama-only).

## Query Modes

The `ask()` function offers tiered query modes that vary in how many LLM calls they make. In dual mode, up to three LLM calls are issued — one for TMS synthesis, one for FTS RAG, and one to merge the results — with short-circuiting to two calls if either retrieval path returns empty (ask-dual-mode-makes-three-llm-calls). When `no_synth=True`, the function returns formatted FTS5 search results directly without contacting the LLM at all (no-synth-mode-skips-llm, ask-no-synth-bypasses-llm), providing a fast, deterministic fallback for raw retrieval.

Beyond querying, the `detect-contradictions` command uses LLMs for semantic analysis. When invoked with the `--semantic` flag, it first embeds beliefs via sentence-transformers and clusters them with KMeans, then sends each cluster to the LLM as a batch — ensuring topically related beliefs are analyzed together rather than being scattered across arbitrary groups of 50 (semantic-contradictions-cluster-before-llm).

## Fault Tolerance

LLM calls are treated as inherently unreliable, and the system degrades gracefully at every level. If the LLM subprocess times out or raises a `RuntimeError`, `ask()` catches the exception and falls back to raw FTS5 search results, guaranteeing a string return rather than propagating the failure (ask-never-raises-on-llm-failure). Similarly, when `list_negative()` receives unparseable LLM output, it returns `count == 0` rather than raising an exception (api-list-negative-graceful-on-malformed-llm). This pattern — graceful degradation over failure — is consistent across the LLM boundary.

## Safety Boundaries for LLM-Driven Mutations

Belief derivation, where the LLM proposes new beliefs from existing ones, is the most sensitive LLM integration point. The system employs defense in depth: the derive pipeline validates proposals with fail-soft filtering, Jaccard retraction guards, and environment stripping at the LLM boundary, while the API layer enforces atomic load/save with write-flag gating and dict-only returns at the persistence boundary (llm-driven-mutations-are-safely-bounded). These guarantees compose end-to-end: input validation, atomic persistence, and deterministic terminating BFS propagation ensure that LLM-driven mutations are bounded at every stage (llm-mutations-are-bounded-end-to-end).

One intentional gap remains: while structural validation confirms that justification references exist and are IN, the *logical soundness* of the inference — whether the derived conclusion actually follows from its antecedents — is validated only by the proposing LLM. No code-level check verifies logical validity (derived-belief-soundness-is-llm-only). This is a deliberate design trade-off, relying on LLM reasoning quality and downstream review rather than formal verification.

## CLI Command Categories

Approximately 21 CLI commands operate only against a local SQLite database rather than the PostgreSQL API. These fall into three categories: filesystem-dependent commands (such as `hash-sources` and `check-stale`), LLM-powered commands (including `derive`, `review-beliefs`, `detect-contradictions`, and `ask`), and bulk import/sync operations (sqlite-only-commands-are-filesystem-llm-or-bulk). The LLM-powered commands require load-entire-network-modify-save semantics that are incompatible with the per-operation transaction model used by the PostgreSQL backend.
