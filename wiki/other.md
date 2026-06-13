# Other

[Back to index](index.md)

This page covers miscellaneous aspects of the ftl-reasons system that span multiple subsystems — architectural decisions, operational guarantees, resource management, and cross-cutting concerns that don't belong to a single domain. Together they paint a picture of how the system's individual components compose into a coherent whole.

## Architectural Foundations

The codebase follows a strict three-layer stack: a pure data model at the bottom (`__init__.py` containing only dataclass definitions), the TMS engine in the middle (`network.py`), and persistence at the top (`storage.py`), with `api.py` providing a functional API and `cli.py` as a thin argparse wrapper (three-layer-stack-has-clean-boundaries). Despite `network.py` being imported by virtually every other module (network-is-central-dependency), clean layer boundaries contain this central coupling's blast radius so it does not compromise architectural integrity (central-dependency-is-safely-contained).

Safety is enforced along two independent dimensions: structurally, the three-layer boundaries prevent cross-layer corruption; operationally, every mutation path is atomic and isolated through context-managed load/save (architecture-enforces-structural-and-operational-safety). The API layer ensures this atomicity through a `_with_network` context manager, per-function transaction scope, write-flag gating, and dict-only returns that prevent callers from holding live network references (api-layer-ensures-atomic-isolated-mutations).

The CLI is a pure delegation layer — every handler dispatches through a flat dict lookup to API functions with no business logic, producing binary exit codes and correct stream separation between stdout and stderr (cli-is-pure-delegation-layer, cli-exit-code-contract-is-binary).

## Truth Evaluation and Semantic Minimality

Truth maintenance semantics are fully emergent from simple uniform rules rather than explicit implementation. A node's truth is a disjunction over justifications (any valid justification makes it IN), where each justification is a conjunction (all antecedents must be IN and all outlist entries must be OUT) (truth-is-disjunctive-over-conjunctive-rules). Premise behavior arises naturally from empty justification lists — nodes with no justifications default to IN, and `_compute_truth` preserves their current truth value rather than recomputing it (premise-behavior-emerges-from-absence).

The system unifies semantic minimality with operational determinism: all non-monotonic features derive from uniform outlist/disjunction primitives, and all operations terminate predictably via BFS fixpoint with conservative failure semantics (semantic-minimality-with-operational-determinism). This yields a small trusted kernel that powers all reasoning, where completeness is achieved through minimality rather than despite it (completeness-and-minimality-are-unified).

CP and SL justifications use the same validity check in `_justification_valid` — the distinction is semantic, not computational (cp-and-sl-evaluated-identically). Evaluation is uniformly context-agnostic, producing identical results regardless of when a justification was added or whether a node is an ordinary belief or a dialectical construct (evaluation-is-uniformly-context-and-origin-agnostic).

## Absence and Edge Cases

Absence has deliberate, defined semantics at two levels throughout the system (absence-has-consistent-dual-semantics). Structural absence — having no justifications — creates premise behavior via vacuous truth over empty lists. Referential absence — missing nodes — follows a conservative/permissive asymmetry: absent antecedents fail validation while absent outlist nodes pass (missing-nodes-have-asymmetric-fail-semantics). This creates a "believe unless proven otherwise" default where missing counter-evidence is permissive but missing supporting evidence is strict (sl-outlist-asymmetry).

All semantic edge cases — vacuous premises, asymmetric absence, empty antecedent lists — emerge from the same uniform evaluation rules without special-case handling (all-semantic-edge-cases-are-uniform). An SL justification with an empty antecedent list is valid via `all([])`, allowing outlist-only justifications to function as "IN unless Y" (empty-antecedents-vacuously-valid).

## Propagation and Termination

Truth propagation uses `deque`-based BFS through the dependents graph (propagation-is-bfs), with guaranteed termination: BFS prevents stack overflow, stop-on-unchanged prevents oscillation by not enqueuing dependents whose recomputed truth matches their current value, and fixpoint iteration bounds the outer loop (propagation-terminates-deterministically). The seed node is added to `visited` immediately and never has its own truth value recomputed — callers must update it before calling `_propagate` (propagate-does-not-change-trigger).

Dangling dependent references are safely contained: BFS skips missing nodes with structured warnings rather than crashing, the changed set never includes ghost IDs, and the visited set excludes dangling IDs so later-created nodes propagate normally (dangling-dependents-are-safely-contained). Warning entries follow a stable schema with `action`, `target`, `value`, and `timestamp` fields (warning-log-schema-stable).

Retracted nodes (marked with `_retracted` metadata) are skipped during BFS propagation but remain in the network for potential restoration (retracted-nodes-skipped-in-propagation). This sticky retraction survives even `recompute_all()` — only `assert_node` on the pinned node itself can clear it (sticky-retraction-survives-recompute-all).

## Dual Storage Backends

The system supports both SQLite and PostgreSQL backends interchangeably (dual-storage-backends-are-interchangeable). The SQLite path uses context-managed load/save with write-flag gating and full-replace persistence — `save()` deletes all rows from every table before re-inserting the entire network (storage-save-is-full-replace). It enables WAL mode for concurrent readers (storage-uses-wal-mode) and rebuilds the FTS5 full-text search index from scratch on every save (storage-fts-is-derived-index).

The PostgreSQL backend (`PgApi`) operates as a SQL-native multi-tenant implementation with no in-memory Network object ever constructed (pgapi-no-in-memory-network). All tables use composite primary keys `(id, project_id)` for multi-tenancy (pgapi-multi-tenant-composite-keys), and each public method is a single committed transaction (pgapi-one-transaction-per-method). BFS propagation is implemented in application-level Python using JSONB containment queries against GIN indexes (pgapi-bfs-propagation-in-python).

Notably, PgApi's `_find_dependents` queries both antecedent and outlist JSONB containment (pgapi-find-dependents-queries-outlist), a capability the in-memory Network historically lacked. Both backends support safe hypothetical reasoning through what-if operations — the SQLite path uses `write=False` context managers, while PostgreSQL wraps mutations in transactions with guaranteed rollback (pg-what-if-is-safely-simulated, write-false-prevents-persistence).

Several commands remain SQLite-only: `derive`, `ask`, `review-beliefs`, `deduplicate`, and `contradictions` are guarded by `_require_sqlite()` (cli-sqlite-only-commands-exist).

## Search and Query Resilience

The search system operates correctly regardless of FTS5 index availability through multiple fallback tiers: FTS5 full-text search is tried first, then substring matching if FTS5 tables don't exist or error (search-falls-back-to-substring). Progressive relaxation drops query terms via combinations when the full-term query returns no results, capped at 50 total queries to prevent combinatorial explosion (fts-relaxation-capped-at-fifty-queries). Stop words are filtered before querying, with fallback to terms longer than 1 character if every word is a stop word (fts-stop-words-filtered-before-query).

The `ask()` function supports tiered query modes with graceful degradation: full LLM synthesis with a bounded 3-iteration tool loop, no-synth mode bypassing the LLM entirely, and automatic fallback from LLM failure to raw FTS5 results (ask-has-tiered-query-modes). It always returns a string on every code path and never raises an exception to the caller (ask-always-returns-string).

MCP tool integration adds a second tier of bounding: tool calls are both error-tolerant (exceptions caught and fed back as context) and iteration-bounded at 5 rounds (ask-mcp-integration-is-safely-bounded), with independent timeouts at connection (30s) and per-call (60s) levels (mcp-bridge-is-timeout-bounded-at-all-phases).

## Access Control

Access control enforces transitive subset-based authorization: a tagged node is visible only when its `access_tags` are a subset of the caller's `visible_to` set (access-tags-subset-gate). Derived nodes inherit the sorted, deduplicated union of all ancestor tags transitively through justification chains (tag-inheritance-is-transitive-union, access-tags-union-inheritance). Untagged nodes are always visible — they are treated as unconditionally public (untagged-always-visible).

Enforcement occurs at read boundaries only: `show_node`, `explain_node`, and `trace_assumptions` raise `PermissionError` on forbidden nodes, while list and export functions silently exclude them from results (direct-access-raises-list-access-filters, access-control-enforced-at-read-not-write). Tag propagation is dynamic — adding a justification triggers cascading recomputation through all downstream dependents (tag-propagation-is-dynamic).

## Contradiction Resolution

The nogood resolution system minimizes network disruption through layered heuristics (contradiction-resolution-is-minimal-disruption). The primary path traces justification chains back to premises and selects the least-entrenched for retraction, while a fallback uses dependent count when no traceable chain exists (backtracking-retracts-least-entrenched, add-nogood-fallback-uses-dependent-count). Contradictions are unconditionally recorded regardless of resolution outcome (add-nogood-always-records).

Nogood IDs form a durable collision-free sequence: the counter only increases (surviving deletions and clears), persists across save/load cycles via the `network_meta` table, and advances past imported IDs (nogood-ids-are-durable-and-collision-free). Databases created before the monotonic counter fix load without error through backward-compatible derivation (nogood-id-backwards-compat).

Contradiction detection complements belief review — `review-beliefs` catches invalid reasoning within individual derivation steps, while `contradictions` catches incompatible facts across independently valid beliefs (review-and-contradictions-catch-orthogonal-errors). Semantic contradiction detection reuses the clustering infrastructure from deduplication, with no duplicate embedding implementation (semantic-contradiction-reuses-cluster-infrastructure).

## Metadata as Universal Extension

Node metadata serves as the universal extension mechanism, carrying all structured lifecycle state — retraction flags, stale reasons, challenges, access tags, supersession markers — as generic key-value pairs rather than typed Node fields (metadata-is-universal-extension-mechanism). This keeps the dataclass stable while features layer on behavior. Critically, lifecycle state in metadata is not passive storage but actively governs truth propagation: retracted nodes are skipped during BFS traversal (metadata-actively-governs-truth-propagation).

Staleness information survives the binary IN/OUT truth model through this metadata-based approach: stale beliefs are mapped to OUT on import with `stale_reason` metadata, and the compact output surfaces this for downstream consumers to distinguish intentional retractions from staleness-driven ones (staleness-is-surfaced-despite-binary-truth-model).

## Resource Efficiency and Packaging

The core `reasons_lib` package has zero mandatory runtime dependencies — the entire library runs on Python's standard library alone (zero-runtime-dependencies). PostgreSQL, clustering, and MCP support are gated behind optional install extras (ftl-reasons-zero-runtime-deps). Both the API and CLI layers defer importing heavy modules to function bodies rather than module top-level, minimizing import-time overhead (startup-performance-uses-lazy-loading).

The project requires Python 3.10+ (python-310-minimum) and only distributes the `reasons_lib` package — tests, entries, reviews, and knowledge-base artifacts are excluded from wheels (only-reasons-lib-packaged). The pip-installable name `ftl-reasons` is deliberately decoupled from the importable package name `reasons_lib` (package-name-split), and the version must be bumped manually with no dynamic versioning plugin (version-is-manual).

## Quality Gates and CI Integration

The system enforces belief quality through dual non-mutating gates targeting complementary validity dimensions (dual-quality-gates-are-complementary-and-non-mutating). Review validates logical soundness of derived beliefs, scoped to justified nodes with dry-run gated auto-retraction (review-pipeline-is-scoped-and-mutation-safe). Staleness checking validates source currency of all IN beliefs as a conservative CI gate with nonzero exit on drift (staleness-is-conservative-ci-gate). Neither gate can corrupt network state.

Staleness detection compares full 64-character SHA-256 hex digests (staleness-uses-full-sha256) — earlier truncation to 16 hex characters was removed in PR #40 (hash-file-full-sha256). Results use a uniform 6-key schema regardless of reason type, sorted by node ID for deterministic output (check-stale-output-is-deterministic-and-structured).

Several commands follow a plan-review-apply pattern: generate proposals to a file, allow human review and editing, then apply the reviewed plan with `--accept` — with `--auto` collapsing all three phases (cli-plan-review-apply-pattern). The derive pipeline's `--exhaust` flag implies `--auto`, iterating until no new beliefs can be derived (exhaust-implies-auto).

## LLM Integration Boundaries

LLM integration is bounded and fail-soft across all modules. The `_invoke_claude` function removes the `CLAUDECODE` environment variable to prevent recursive invocation (ask-strips-claudecode-env), and `invoke_model` in `llm.py` applies this same isolation to all subprocess calls (invoke-model-strips-claudecode-env). The `ask` module implements tool use by parsing JSON lines from LLM text output rather than using native function-calling APIs, because it invokes `claude -p` which doesn't support that (ask-uses-text-based-tool-protocol).

The `list_negative` pipeline applies defense-in-depth: keyword pre-filtering narrows candidates before LLM classification, hallucinated node IDs are discarded against the actual network, and malformed output falls back gracefully to zero results (list-negative-is-defensively-bounded). Its response parser handles prose preambles via regex extraction, a pragmatic accommodation of common LLM output patterns (list-negative-json-parser-tolerates-prose-preamble).

## Clustering and Deduplication

Clustering dependencies (`sentence-transformers`, `scikit-learn`) are optional behind a `[cluster]` install extra with graceful degradation when absent (cluster-deps-optional-with-graceful-skip). When the number of beliefs is at or below budget, all ML work is skipped entirely (cluster-skips-ml-when-under-budget). The auto cluster count targets approximately 5 beliefs per cluster, clamped between 2 and `min(budget // 3, 20)` (cluster-auto-k-heuristic), with remainder slots distributed to the largest clusters first (cluster-remainder-favors-largest).

Cluster cache keys include a content hash prefix, so editing a belief's text forces re-embedding rather than serving stale vectors (cluster-cache-keys-include-content-hash). Deduplication retains the cluster member with the most dependents and rewrites all justification references — both antecedent and outlist — to point at the survivor (dedup-keeps-most-connected-node, dedup-rewrites-both-antecedents-and-outlist). The dedup plan format supports user-editable KEEP/RETRACT markers for reviewable decisions (dedup-plan-is-user-editable).

## Open Issues

Several open issues track verification gaps in the system. Issue #121 calls for auditing evolution tolerance at all system boundaries (issue-121-evolution-tolerance-audit). Issue #122 targets unhandled failure modes in the review module (issue-122-review-fault-tolerance-audit). Issue #123 asks for resource footprint auditing across all lifecycle phases beyond just deployment and startup (issue-123-resource-footprint-audit). Issue #126 seeks to verify node ID reference validation at every boundary (issue-126-reference-validation-audit). These represent known gaps between claimed properties and verified coverage.
