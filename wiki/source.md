# source

[Back to index](index.md)

### ask-source-chunks-persist-across-tool-iterations
**Status:** IN

Source document content included in the initial ask prompt is re-included in all subsequent prompts after tool-call round-trips, ensuring the LLM retains source context across the entire multi-turn loop.


### belief-source-metadata-is-file-level
**Status:** IN

Belief source tracking uses flat file-level pointers (source path + source_hash) with no structural awareness of where within a document the claim originated — section, page, or paragraph level provenance is not supported.


### check-stale-detects-source-staleness-only
**Status:** IN

`check-stale` detects source staleness (source file changed on disk) via hash comparison but cannot detect world staleness (the belief is outdated even though the source file hasn't changed) — beliefs can silently decay without any artifact triggering re-examination.


### check-stale-no-dedup-by-source-path
**Status:** IN

Multiple nodes referencing the same missing source file each produce independent `source_deleted` results — no deduplication by file path.


### check-stale-requires-both-source-fields
**Status:** IN

A node must have both `source` (non-empty) and `source_hash` (non-empty) to be eligible for staleness checking; nodes missing either field are silently skipped.

**Supports:** [staleness-checking-is-comprehensive](other.md#staleness-checking-is-comprehensive), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)

### check-stale-source-deleted-returns-none-hashes
**Status:** IN

When a source file is missing, `check_stale` returns a result dict with `reason="source_deleted"`, `new_hash=None`, and `source_path=None` (fix for issue #25 — previously this case was silently skipped).


### integrity-is-boundary-and-source-agnostic
**Status:** OUT

System integrity is enforced agnostically along two independent dimensions — across the internal/external boundary (local mutations vs. external ingestion) and across mutation sources (human dialectics, LLM derivation, agent import) — because the same atomic-deterministic pipeline processes all combinations without branching on origin or direction.

**Depends on:** [internal-and-external-integrity-are-unified](external.md#internal-and-external-integrity-are-unified), [operational-safety-spans-all-mutation-sources](spans.md#operational-safety-spans-all-mutation-sources)
**Supports:** [safety-integrity-and-uniformity-converge](other.md#safety-integrity-and-uniformity-converge), [verified-mutation-correctness-across-boundaries](other.md#verified-mutation-correctness-across-boundaries)

### missing-source-file-is-silent
**Status:** OUT

If a node's source file no longer exists on disk, `check_stale` silently skips it; callers cannot distinguish "file deleted" from "file never tracked."

**Supports:** [all-external-inputs-produce-correct-state](external.md#all-external-inputs-produce-correct-state), [architecture-sustains-gapless-lifecycle](lifecycle.md#architecture-sustains-gapless-lifecycle), [automated-sync-achieves-full-lifecycle-coverage](lifecycle.md#automated-sync-achieves-full-lifecycle-coverage), [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [external-belief-lifecycle-is-complete](external.md#external-belief-lifecycle-is-complete), [external-beliefs-are-defended-and-lifecycle-managed](external.md#external-beliefs-are-defended-and-lifecycle-managed), [external-beliefs-are-safe-and-current](external.md#external-beliefs-are-safe-and-current), [external-integration-is-architecturally-safe](external.md#external-integration-is-architecturally-safe), [external-integration-preserves-all-invariants](external.md#external-integration-preserves-all-invariants), [external-lifecycle-is-complete-and-automatically-maintained](external.md#external-lifecycle-is-complete-and-automatically-maintained), [lifecycle-governance-achieves-gap-free-source-coverage](lifecycle.md#lifecycle-governance-achieves-gap-free-source-coverage), [lifecycle-management-is-gapless](lifecycle.md#lifecycle-management-is-gapless), [self-correction-is-resource-sustainable](self.md#self-correction-is-resource-sustainable), [staleness-checking-is-comprehensive](other.md#staleness-checking-is-comprehensive), [staleness-gate-catches-all-drift](other.md#staleness-gate-catches-all-drift), [unified-system-is-a-closed-self-maintaining-architecture](system.md#unified-system-is-a-closed-self-maintaining-architecture)

### resolve-source-defaults-to-home-git
**Status:** IN

When no `repos` mapping is provided, `resolve_source_path` falls back to `~/git/<repo-name>/<rel-path>` by convention.


### resolve-source-fallback-path
**Status:** IN

When no `repos` mapping is provided, `resolve_source_path` constructs paths as `~/git/<repo-name>/<relative-path>` by convention.


### resolve-source-path-db-dir-precedence
**Status:** IN

`resolve_source_path` follows a strict precedence chain: `db_dir` resolution takes priority over agent repo resolution, which takes priority over repo-key-split resolution; missing files return `None`.


### resolve-source-path-never-raises
**Status:** IN

`resolve_source_path` returns `None` for nonexistent files instead of raising exceptions; callers must check the `None` case.

**Supports:** [source-resolution-is-convention-based-and-fail-safe](source.md#source-resolution-is-convention-based-and-fail-safe)

### resolve-source-path-returns-none-on-missing
**Status:** IN

`resolve_source_path` returns `None` (not an exception) when the resolved file does not exist on disk or when the source string is empty.


### search-source-chunks-filters-stop-words
**Status:** IN

`_search_source_chunks` filters out single-character tokens and common stop words before constructing FTS5 queries, returning empty string when no usable terms remain.


### source-correction-achieves-resource-efficient-assurance
**Status:** OUT

Source-grounded lifecycle correction with tripartite operational assurance — externally controlled, internally self-correcting, and query-resilient — achieves these guarantees within resource-efficient bounds spanning the full operational pipeline, ensuring correction mechanisms never exhaust the system's computational budget.

**Depends on:** [operational-assurance-is-resource-efficient](other.md#operational-assurance-is-resource-efficient), [source-grounded-correction-has-tripartite-assurance](source.md#source-grounded-correction-has-tripartite-assurance)

### source-governance-loop-is-dually-grounded-and-deterministic
**Status:** IN

The closed source-integrity-governance loop is both dually grounded (resting on two independent evaluation chains — purity and uniformity) and fully deterministic (governance determinism emerges from minimality through source integrity and is preserved by exception safety), achieving both epistemic independence and operational predictability from independent foundations

**Depends on:** [governance-determinism-is-generated-and-preserved](governance.md#governance-determinism-is-generated-and-preserved), [source-integrity-loop-is-dually-grounded](source.md#source-integrity-loop-is-dually-grounded)

### source-grounded-correction-has-tripartite-assurance
**Status:** OUT

The system's lifecycle self-correction — concretely grounded in fail-safe source integrity verification with governed output — operates within a tripartite assurance framework providing external control (bounded interfaces with defensive ingestion), internal self-correction (contradiction resolution and staleness detection), and query resilience (graceful degradation across all access paths).

**Depends on:** [source-grounded-correction-produces-governed-output](source.md#source-grounded-correction-produces-governed-output), [system-achieves-tripartite-operational-assurance](system.md#system-achieves-tripartite-operational-assurance)
**Supports:** [source-correction-achieves-resource-efficient-assurance](source.md#source-correction-achieves-resource-efficient-assurance)

### source-grounded-correction-produces-governed-output
**Status:** OUT

The system's lifecycle self-correction — concretely grounded in fail-safe source integrity verification — feeds into a governed output pipeline that is authorized (access-tag gated), bounded (token-budget constrained), and CI-ready (deterministic with nonzero exit codes), ensuring that source-level drift detection translates into actionable, permission-respecting output

**Depends on:** [output-governance-is-complete-authorized-and-ci-ready](governance.md#output-governance-is-complete-authorized-and-ci-ready), [source-integrity-grounds-lifecycle-self-correction](source.md#source-integrity-grounds-lifecycle-self-correction)
**Supports:** [source-grounded-correction-has-tripartite-assurance](source.md#source-grounded-correction-has-tripartite-assurance)

### source-integrity-and-governance-form-closed-loop
**Status:** IN

Source integrity enables lifecycle governance by unifying determinism, exception safety, and lifecycle management into a single pipeline, while lifecycle governance achieves gap-free source coverage — forming a self-reinforcing closed loop where source integrity grounds the governance that in turn ensures no source verification gap exists.

**Depends on:** [lifecycle-governance-achieves-gap-free-source-coverage](lifecycle.md#lifecycle-governance-achieves-gap-free-source-coverage), [source-integrity-unifies-determinism-exception-safety-and-lifecycle](source.md#source-integrity-unifies-determinism-exception-safety-and-lifecycle)
**Supports:** [source-integrity-loop-is-dually-grounded](source.md#source-integrity-loop-is-dually-grounded)

### source-integrity-grounds-lifecycle-self-correction
**Status:** OUT

The system's lifecycle-spanning self-correction is concretely grounded in fail-safe source integrity: the end-to-end source pipeline (convention-based path resolution, collision-resistant SHA-256 hashing, comprehensive staleness detection, CI gating) provides the verification mechanism that makes maintenance-time self-correction practically achievable alongside creation-time exhaustive derivation.

**Depends on:** [self-correction-is-exhaustive-across-lifecycle](self.md#self-correction-is-exhaustive-across-lifecycle), [source-lifecycle-is-fail-safe-and-gapless](source.md#source-lifecycle-is-fail-safe-and-gapless)
**Supports:** [self-correction-is-source-grounded-and-self-documenting](self.md#self-correction-is-source-grounded-and-self-documenting), [source-grounded-correction-produces-governed-output](source.md#source-grounded-correction-produces-governed-output)

### source-integrity-is-deterministic-and-architecturally-grounded
**Status:** IN

The fail-safe source integrity pipeline — from convention-based path resolution through collision-resistant SHA-256 hashing to comprehensive staleness detection — operates within the same deterministic, architecturally-grounded lifecycle framework as all other system operations, ensuring source verification is both predictable and structurally safe.

**Depends on:** [lifecycle-is-deterministic-and-architecturally-grounded](lifecycle.md#lifecycle-is-deterministic-and-architecturally-grounded), [source-lifecycle-is-fail-safe-and-gapless](source.md#source-lifecycle-is-fail-safe-and-gapless)
**Supports:** [determinism-spans-revision-semantics-through-source-integrity](spans.md#determinism-spans-revision-semantics-through-source-integrity), [lifecycle-governance-has-deterministic-source-integrity](lifecycle.md#lifecycle-governance-has-deterministic-source-integrity), [source-integrity-is-fail-safe-deterministic-and-grounded](source.md#source-integrity-is-fail-safe-deterministic-and-grounded), [source-to-tms-integrity-is-deterministic-and-exception-safe](source.md#source-to-tms-integrity-is-deterministic-and-exception-safe)

### source-integrity-is-fail-safe-deterministic-and-grounded
**Status:** IN

The source integrity pipeline achieves triple assurance from two independent chains: fail-safe operation (convention-based path resolution returning None on missing files, collision-resistant SHA-256 hashing with additive backfill, comprehensive staleness detection) combined with architectural determinism (grounded within clean layer boundaries ensuring predictable state trajectories).

**Depends on:** [source-integrity-is-deterministic-and-architecturally-grounded](source.md#source-integrity-is-deterministic-and-architecturally-grounded), [source-pipeline-is-end-to-end-fail-safe](source.md#source-pipeline-is-end-to-end-fail-safe)

### source-integrity-loop-is-dually-grounded
**Status:** IN

The closed source-integrity-governance loop is itself dually grounded — the governance half of the loop rests on two independent semantic foundations (evaluation purity and edge-case uniformity), so the bidirectional feedback cycle between source integrity and lifecycle governance remains sound even if one grounding chain is weakened.

**Depends on:** [governance-has-dual-independent-grounding-chains](governance.md#governance-has-dual-independent-grounding-chains), [source-integrity-and-governance-form-closed-loop](source.md#source-integrity-and-governance-form-closed-loop)
**Supports:** [source-governance-loop-is-dually-grounded-and-deterministic](source.md#source-governance-loop-is-dually-grounded-and-deterministic)

### source-integrity-spans-hashing-through-detection
**Status:** IN

Source integrity forms a complete end-to-end pipeline with no gap between measurement and verification: collision-resistant SHA-256 hashing with additive backfill computes integrity markers without overwriting existing hashes, while comprehensive staleness detection with CI gating and nonzero exit codes consumes those markers to catch all source drift.

**Depends on:** [source-tracking-is-collision-resistant-and-safe](source.md#source-tracking-is-collision-resistant-and-safe), [staleness-gate-catches-all-drift](other.md#staleness-gate-catches-all-drift)
**Supports:** [source-pipeline-is-end-to-end-fail-safe](source.md#source-pipeline-is-end-to-end-fail-safe)

### source-integrity-unifies-determinism-exception-safety-and-lifecycle
**Status:** IN

The source integrity pipeline achieves triple assurance from two independent chains: the source-to-TMS path is deterministic (convention-based resolution, collision-resistant SHA-256) and exception-safe (fail-safe across both TMS and source lifecycle domains), while lifecycle governance uses that same deterministic source integrity to drive staleness decisions and belief currency management — source integrity simultaneously serves pipeline correctness, failure recovery, and lifecycle governance.

**Depends on:** [lifecycle-governance-has-deterministic-source-integrity](lifecycle.md#lifecycle-governance-has-deterministic-source-integrity), [source-to-tms-integrity-is-deterministic-and-exception-safe](source.md#source-to-tms-integrity-is-deterministic-and-exception-safe)
**Supports:** [source-integrity-and-governance-form-closed-loop](source.md#source-integrity-and-governance-form-closed-loop)

### source-lifecycle-is-fail-safe-and-gapless
**Status:** IN

The end-to-end fail-safe source integrity pipeline — from convention-based path resolution through collision-resistant SHA-256 hashing to comprehensive drift detection — feeds directly into gapless lifecycle management, ensuring every source material change on disk is detected, surfaced, and managed through the full belief lifecycle without gaps.

**Depends on:** [lifecycle-management-is-gapless](lifecycle.md#lifecycle-management-is-gapless), [source-pipeline-is-end-to-end-fail-safe](source.md#source-pipeline-is-end-to-end-fail-safe)
**Supports:** [exception-safety-spans-tms-and-source-lifecycle](spans.md#exception-safety-spans-tms-and-source-lifecycle), [lifecycle-governance-is-metadata-enabled-and-source-grounded](lifecycle.md#lifecycle-governance-is-metadata-enabled-and-source-grounded), [source-integrity-grounds-lifecycle-self-correction](source.md#source-integrity-grounds-lifecycle-self-correction), [source-integrity-is-deterministic-and-architecturally-grounded](source.md#source-integrity-is-deterministic-and-architecturally-grounded)

### source-path-format
**Status:** IN

Source references use `"reponame/relative/path"` format; `resolve_source_path` splits on the first `/` to look up the repo key in a `repos: dict[str, Path]` mapping.

**Supports:** [source-resolution-is-convention-based-and-fail-safe](source.md#source-resolution-is-convention-based-and-fail-safe)

### source-paths-use-repo-alias-prefix
**Status:** IN

Node source paths follow a `repo-alias/relative-path` convention where the first `/`-delimited segment is a key into the `repos` dict that maps aliases to filesystem roots, decoupling belief source references from absolute paths.


### source-pipeline-is-end-to-end-fail-safe
**Status:** IN

The complete source integrity pipeline from path resolution through hash computation to staleness detection is end-to-end fail-safe: convention-based resolution returns None on missing files rather than raising exceptions, collision-resistant SHA-256 hashing backfills additively without overwriting, and staleness detection catches all drift with CI-ready exit codes — no stage in the pipeline raises exceptions on adverse conditions, and each stage's output gracefully feeds the next.

**Depends on:** [source-integrity-spans-hashing-through-detection](source.md#source-integrity-spans-hashing-through-detection), [source-resolution-is-convention-based-and-fail-safe](source.md#source-resolution-is-convention-based-and-fail-safe)
**Supports:** [source-integrity-is-fail-safe-deterministic-and-grounded](source.md#source-integrity-is-fail-safe-deterministic-and-grounded), [source-lifecycle-is-fail-safe-and-gapless](source.md#source-lifecycle-is-fail-safe-and-gapless)

### source-resolution-is-convention-based-and-fail-safe
**Status:** IN

Source path resolution follows a convention-based repo-alias/relative-path format and returns None on missing files rather than raising exceptions, providing safe path resolution for the staleness checking pipeline.

**Depends on:** [resolve-source-path-never-raises](source.md#resolve-source-path-never-raises), [source-path-format](source.md#source-path-format)
**Supports:** [source-pipeline-is-end-to-end-fail-safe](source.md#source-pipeline-is-end-to-end-fail-safe)

### source-to-tms-integrity-is-deterministic-and-exception-safe
**Status:** IN

The complete integrity pipeline from source files through TMS truth maintenance is both deterministic (convention-based path resolution, collision-resistant SHA-256 hashing, uniform pure evaluation) and exception-safe (fail-safe source resolution, contradiction-triggered backtracking, challenge-to-justified recovery) — no failure in either source verification or truth maintenance can corrupt system state or produce unpredictable outcomes.

**Depends on:** [exception-safety-spans-tms-and-source-lifecycle](spans.md#exception-safety-spans-tms-and-source-lifecycle), [source-integrity-is-deterministic-and-architecturally-grounded](source.md#source-integrity-is-deterministic-and-architecturally-grounded)
**Supports:** [source-integrity-unifies-determinism-exception-safety-and-lifecycle](source.md#source-integrity-unifies-determinism-exception-safety-and-lifecycle)

### source-tracking-is-collision-resistant-and-safe
**Status:** IN

Source integrity tracking uses full 64-character SHA-256 digests with exact string comparison for collision resistance, and backfills hashes additively without overwriting existing values — ensuring high-fidelity drift detection with no accidental data loss

**Depends on:** [hash-file-full-sha256](other.md#hash-file-full-sha256), [hash-sources-is-additive-by-default](other.md#hash-sources-is-additive-by-default), [staleness-uses-full-sha256](other.md#staleness-uses-full-sha256)
**Supports:** [source-integrity-spans-hashing-through-detection](source.md#source-integrity-spans-hashing-through-detection)
