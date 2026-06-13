# Other

[Back to index](index.md)

This page covers the full architecture, semantics, and operational characteristics of the ftl-reasons Truth Maintenance System — a Python implementation of Doyle's 1979 TMS with extensions for multi-agent federation, dialectical reasoning, and dual-backend persistence.

## Architecture

The codebase follows a strict three-layer stack: a pure data model at the bottom (`__init__.py` containing only `Node`, `Justification`, and `Nogood` dataclasses), the TMS engine in the middle (`network.py`), and persistence at the top (`storage.py`), with `api.py` providing a functional interface and `cli.py` acting as a thin argparse wrapper (three-layer-stack-has-clean-boundaries). Despite `network.py` being imported by virtually every module, the layer boundaries contain its blast radius so that this central coupling does not create cross-cutting mutation paths (central-dependency-is-safely-contained).

Architectural safety is enforced along two independent dimensions. Structurally, clean layer separation prevents cross-layer corruption. Operationally, every mutation path is atomic and isolated, preventing partial state within any layer (architecture-enforces-structural-and-operational-safety). The API layer enforces mutation safety through context-managed load/save, per-function transaction scope, write-flag gating, and dict-only returns that prevent callers from holding live network references (api-layer-ensures-atomic-isolated-mutations).

The project requires Python 3.10+ and has zero mandatory runtime dependencies — the entire `reasons_lib` package runs on Python's standard library alone (zero-runtime-dependencies). PostgreSQL, clustering, and MCP support are gated behind optional install extras.

## Truth Maintenance Semantics

### Premises and Emergence

Premise behavior is not explicitly implemented — it emerges from three defaults: nodes with no justifications default to IN, empty antecedent lists satisfy vacuous truth via `all([])`, and the system preserves a premise's current truth value rather than deriving it (premise-behavior-emerges-from-absence). This emergent quality extends to all semantic edge cases, which are handled by the same uniform evaluation rules without special-case code (all-semantic-edge-cases-are-uniform).

### Disjunctive Truth

A node's truth value follows disjunctive-over-conjunctive semantics: the node is IN if *any* of its justifications is valid (disjunction), where each justification requires all antecedents to be IN and all outlist entries to be OUT (conjunction) (truth-is-disjunctive-over-conjunctive-rules). When `any_mode=True`, a multi-antecedent justification is expanded into per-premise justifications, reifying OR semantics explicitly — each resulting justification inherits the complete outlist specification (any-mode-preserves-full-outlist-semantics).

CP and SL justifications use the same validity check internally; the distinction is semantic rather than computational (cp-and-sl-evaluated-identically). Evaluation is pure, uniform, and context-agnostic — it produces identical results regardless of when a justification was added or whether the node is an ordinary belief or a dialectical construct (evaluation-is-uniformly-context-and-origin-agnostic).

### Absence Semantics

Absence has deliberate, defined semantics at two levels. Structural absence (no justifications) creates premise behavior. Referential absence (missing nodes) follows a conservative/permissive asymmetry: missing antecedents fail validation (conservative), while missing outlist nodes pass (permissive), creating a "believe unless proven otherwise" default (absence-has-consistent-dual-semantics, missing-nodes-have-asymmetric-fail-semantics).

### Propagation

Truth propagation uses BFS through the dependents graph, guaranteed to terminate because stop-on-unchanged prevents oscillation and `recompute_all` uses fixpoint iteration bounded by node count (propagation-terminates-deterministically). Retracted nodes are skipped during propagation but remain in the network for potential restoration (retracted-nodes-skipped-in-propagation). A retracted node's `_retracted` metadata flag pins it OUT, surviving even `recompute_all()` — only `assert_node` on the pinned node itself can clear it (retracted-pin-survives-recompute).

Dangling dependent references are safely contained: BFS skips missing nodes with structured warnings, the changed set never includes ghost IDs, and the visited set excludes dangling IDs so later-created nodes propagate normally (dangling-dependents-are-safely-contained).

## Non-Monotonic Reasoning

### The Outlist Primitive

The outlist is the sole defeat mechanism underlying all non-monotonic features (outlist-is-universal-defeat-mechanism). When a justification has multiple outlist entries, all must be OUT for the justification to be valid — any single outlist node going IN defeats the entire justification (multiple-outlist-is-conjunction). Outlist relationships survive persistence through JSON serialization with rebuilt dependent indexes (outlist-relationships-survive-persistence).

All outlist-based defeat operations are inherently reversible because outlist semantics flip truth values without deleting nodes (all-defeat-mechanisms-are-reversible). When a defeating node is retracted, recovery propagates automatically through BFS to all affected nodes without manual re-assertion (defeat-reversal-propagates-automatically).

### Challenge and Defend

The challenge mechanism creates a new premise node and adds it to the target's outlist across every justification (challenge-uses-outlist-mechanism, challenge-modifies-all-justifications). When a premise is challenged, it undergoes an irreversible identity transformation — premise identity emerges from absence of justifications, but challenge adds one, converting the premise to a justified node (challenge-destroys-premise-identity). The truth-value defeat is fully reversible, but this structural transformation is permanent (dialectical-defeat-is-reversible-but-identity-is-permanent).

Defense is implemented by calling `challenge()` on the challenge node itself, enabling arbitrarily deep dialectical chains using the same outlist mechanism recursively with no special-case code (defend-is-recursive-challenge). Challenge IDs follow the pattern `challenge-{target}`, with numeric suffixes for subsequent challenges; explicit IDs that collide raise `ValueError` (challenge-id-auto-generation).

### Supersession and Kill-Switch

Supersession adds the new node's ID to the old node's outlist rather than deleting it; retracting the new belief automatically restores the old one through normal propagation (supersession-is-reversible). Superseded nodes are excluded from gated belief lists even if they retain active blockers (api-superseded-nodes-excluded-from-gated).

Each imported agent gets an `agent:active`/`agent:inactive` relay pair. Retracting `agent:active` cascades all agent beliefs to OUT via the inactive relay flipping IN; re-asserting reverses the cascade (kill-switch-cascade-is-reversible). The active premise is deliberately excluded from imported beliefs' antecedents — if included, it would provide a second always-valid justification path that defeats per-belief retraction semantics (active-not-in-antecedents).

## Contradiction Resolution

The nogood resolution system minimizes network disruption through layered heuristics: the primary path traces justification chains back to premises and selects the least-entrenched for retraction, a fallback uses dependent count when no traceable chain exists, and all contradictions are unconditionally recorded regardless of resolution outcome (contradiction-resolution-is-minimal-disruption). The system also provides surgical restoration hints for cascade victims that have surviving premises in multi-antecedent justifications (restoration-hints-are-surgical).

Nogood IDs form a durable collision-free sequence: the counter only increases (surviving deletions and clears), persists across save/load cycles via the `network_meta` table, and advances past imported IDs to prevent collisions (nogood-ids-are-durable-and-collision-free). Databases created before the monotonic counter fix load without error through backward-compatible derivation (nogood-id-backwards-compat).

## Storage and Persistence

### SQLite Backend

Persistence follows a snapshot model: `save()` deletes all rows from every table before re-inserting the entire network (storage-save-is-full-replace), while `load()` constructs nodes directly into the network rather than calling `add_node`, bypassing truth maintenance propagation during deserialization (storage-load-bypasses-propagation). The loaded network trusts stored truth values without re-running propagation (storage-trusts-stored-truth-values).

SQLite connections enable WAL mode for concurrent readers without blocking writes, and the FTS5 full-text search index is rebuilt from scratch during every save as a derived index (storage-optimizes-concurrent-access-and-search). Antecedent IDs, outlist IDs, and metadata are stored as JSON text columns rather than junction tables (storage-uses-json-columns-for-lists).

### PostgreSQL Backend

`PgApi` operates as a SQL-native multi-tenant implementation: all operations execute directly against PostgreSQL with no in-memory Network object, composite primary keys provide project-level isolation, and each public method is a single committed transaction (pgapi-is-sql-native-multi-tenant). BFS propagation runs in application-level Python using JSONB containment queries against GIN indexes to find dependents (pgapi-bfs-propagation-in-python).

Both backends enforce atomic isolated operations through backend-appropriate mechanisms, achieving the same transactional guarantee at different architectural levels (atomicity-is-backend-independent). What-if operations achieve safe simulation through transaction rollback in PostgreSQL and write-flag gating in SQLite ([both-backends-support-safe-hypothetical-reasoning](safe.md#both-backends-support-safe-hypothetical-reasoning)).

PostgreSQL routing uses a function-level early-return pattern — each API function checks `pg_conninfo` and short-circuits to `_pg_dispatch` (pg-dispatch-is-function-level-early-return). Several commands (`derive`, `ask`, `review-beliefs`, `deduplicate`, `contradictions`) are SQLite-only and exit with an error if `--pg` is set (cli-sqlite-only-commands-exist).

## API and CLI Design

Every public API function returns a `dict` or `str`, never a `Network` or `Node` object, ensuring JSON-serializability at the boundary (api-functions-return-dicts). Mutating operations snapshot all truth values before the operation and diff afterward to classify changes into `went_out`/`went_in` lists (api-mutating-ops-use-before-after-diffing). Every network mutation simultaneously maintains the audit log and dependents index (network-mutations-are-audited-and-index-consistent).

The CLI uses flat dict dispatch mapping subcommand strings to handler functions (cli-dispatch-is-flat-dict-lookup). Every `cmd_*` function delegates to `api.*` and only formats the returned dict for terminal output — no business logic lives in the CLI layer (cli-is-pure-formatter). Error diagnostics go to stderr, success output to stdout, and all error paths exit with code 1 (cli-exit-code-contract-is-binary).

Both layers defer importing heavy modules to function bodies rather than module top-level, keeping CLI responsiveness high (startup-performance-uses-lazy-loading).

## Access Control

Access control enforces transitive subset-based authorization: a tagged node is visible only when its `access_tags` are a subset of the caller's `visible_to` set (access-tags-subset-gate). Derived nodes inherit the sorted, deduplicated union of all ancestor tags transitively through justification chains (tag-inheritance-is-transitive-union). Adding a justification triggers access tag recomputation that cascades through all downstream dependents via BFS (tag-propagation-is-dynamic).

Enforcement occurs at read boundaries only — single-node functions raise `PermissionError` while collection endpoints silently filter (direct-access-raises-list-access-filters). Write operations are unrestricted. Nodes without `access_tags` metadata are unconditionally public (untagged-always-visible).

## Search and Query

### FTS5 Search

Search tries FTS5 full-text search first, falls back to substring matching if FTS tables don't exist or error, then expands results with 1-hop neighbors from the dependency graph (search-uses-fts5-with-substring-fallback). Progressive FTS query relaxation drops terms via combinations when the full-term query returns no results, capped at 50 queries to prevent combinatorial explosion (fts-relaxation-capped-at-fifty-queries). Stop words are filtered before querying, with a fallback to terms longer than 1 character if all terms are stop words (fts-stop-words-filtered-before-query).

### Ask Module

The ask module supports tiered query modes with graceful degradation: full LLM synthesis with a bounded 3-iteration tool loop, no-synth mode that bypasses the LLM, and automatic fallback from LLM failure to raw FTS5 search results (ask-has-tiered-query-modes). The function always returns a string — it never raises an exception to the caller (ask-always-returns-string). It implements tool use by parsing JSON lines from LLM text output rather than using native function calling, because it invokes `claude -p` pipe mode (ask-uses-text-based-tool-protocol).

The `_invoke_claude` function removes the `CLAUDECODE` environment variable to prevent recursive invocation when running inside Claude Code (ask-strips-claudecode-env).

### MCP Integration

MCP tool calls in `ask()` are both error-tolerant (exceptions caught and fed back as context) and iteration-bounded (5 tool-call rounds max) (ask-mcp-integration-is-safely-bounded). The MCP bridge runs its own asyncio event loop on a daemon thread, with timeout bounds at both connection establishment (30s) and per-tool execution (60s) (mcp-bridge-is-timeout-bounded-at-all-phases). The tool catalog is populated once during `connect()` and never refreshed (mcp-bridge-tools-snapshot-at-connect).

## Quality Gates

### Staleness Checking

Staleness checking is designed as a safe CI gate: it never mutates state, only checks IN nodes, requires both source fields, and exits nonzero to fail the pipeline (staleness-is-conservative-ci-gate). It uses full 64-character SHA-256 digests for comparison (staleness-uses-full-sha256) and returns deterministic results sorted by node ID with a uniform 6-key schema regardless of reason type ([check-stale-output-is-deterministic-and-structured](deterministic.md#check-stale-output-is-deterministic-and-structured)).

Despite the TMS using binary IN/OUT truth values with no distinct STALE state, staleness information is preserved through metadata — stale beliefs are mapped to OUT on import with `stale_reason` metadata (staleness-is-surfaced-despite-binary-truth-model).

### Belief Review

Review evaluates only derived beliefs (premises are excluded) by presenting each belief with its direct antecedents to an LLM reviewer (review-skips-premises, review-evaluates-direct-antecedents-only). The review module operates entirely on in-memory node dicts with no storage dependency (review-has-no-storage-dependency). Parse results default missing fields to passing — the LLM only needs to report failures explicitly (parse-review-defaults-to-passing). The `--dry-run` flag prevents both truth-value changes and metadata side-effects (dry-run-prevents-both-retraction-and-metadata).

Review and contradiction detection are complementary: review catches invalid reasoning within individual derivation steps, while contradictions catch incompatible facts across independently valid beliefs (review-and-contradictions-catch-orthogonal-errors).

### Deduplication and Contradiction Detection

In auto-dedup mode, the node with the most dependents survives each cluster, and all justification references across the network are rewritten to point at the kept node (dedup-keeps-most-connected-node, dedup-rewrites-both-antecedents-and-outlist). The dedup plan format uses KEEP/RETRACT markers that users can swap before applying (dedup-plan-is-user-editable).

Contradiction detection shuffles beliefs before partitioning into batches of 50 to probabilistically surface cross-belief comparisons that fixed batching would miss (contradictions-shuffles-before-batching). Semantic contradiction detection reuses the clustering infrastructure from deduplication (semantic-contradiction-reuses-cluster-infrastructure).

## Import, Export, and Reconciliation

The namespace system uses a colon-based convention with automatic infrastructure wiring: node creation auto-wires a `{ns}:active` premise as an antecedent, and colon presence prevents double-prefixing (namespace-is-colon-convention-with-auto-wiring). During agent import, nodes are added and truth values propagated before explicit retractions are applied, ensuring the dependency graph is fully wired before OUT transitions are forced (deferred-retraction-ordering).

Sync implements remote-wins reconciliation: remote text/metadata overwrites local, beliefs removed from remote are retracted locally, and beliefs remotely IN but locally OUT are re-asserted (sync-is-remote-wins). Both `_normalize_markdown` and `_normalize_json` silently drop references to IDs not present in the import set, preventing dangling edges (normalization-drops-unknown-refs).

Whether beliefs enter through stored-state bootstrap or through the deterministic reasoning engine, the system reaches correct truth states — establishing path independence of initialization (initialization-is-path-independent).

## Metadata and Lifecycle Governance

Node metadata is the universal extension mechanism carrying all structured lifecycle state — retraction flags, stale reasons, challenges, access tags, and supersession markers — in a generic `metadata` dict rather than typed Node fields (metadata-is-universal-extension-mechanism). This metadata actively governs truth propagation: retracted nodes are skipped during BFS and trigger nodes are never recomputed (metadata-actively-governs-truth-propagation).

The system achieves completeness through minimality rather than despite it — forward truth computation and backward belief revision derive from the same small set of primitives (outlist, disjunctive truth, vacuous validity), so completeness requires no feature accumulation beyond what minimality already provides (completeness-and-minimality-are-unified).

## Clustering

Clustering is behind an optional `[cluster]` install extra; when `sentence-transformers` or `scikit-learn` are missing, dependent operations raise `ImportError` with install instructions (cluster-deps-optional-with-graceful-skip). Auto cluster count targets approximately 5 beliefs per cluster, and when the number of beliefs is at or below budget, all ML work is skipped entirely (cluster-skips-ml-when-under-budget). The cluster cache keys embeddings by content hash so that editing a belief's text forces re-embedding (cluster-cache-keys-include-content-hash).

## Negative Belief Listing

`list_negative` uses a two-stage pipeline: keyword pre-filtering against a hardcoded `NEGATIVE_TERMS` list narrows candidates before LLM classification eliminates false positives (list-negative-uses-two-stage-classification). Hallucinated node IDs are discarded against the actual network, and malformed LLM output falls back gracefully to zero results (list-negative-is-defensively-bounded). The response parser uses `re.finditer` to extract JSON objects from responses that include prose preamble, handling the common LLM pattern of prefacing structured output with natural language (list-negative-json-parser-tolerates-prose-preamble).
