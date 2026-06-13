# source

[Back to index](index.md)

Source tracking in ftl-reasons connects beliefs to the files they were derived from, enabling the system to detect when source material changes on disk and flag beliefs that may have gone stale. The mechanism spans path resolution, cryptographic hashing, staleness detection, and integration with the broader lifecycle governance pipeline.

## Path Format and Resolution

Source references use a `repo-alias/relative-path` convention, where the first `/`-delimited segment is a key into a `repos` dictionary that maps aliases to filesystem roots (`source-path-format`, `source-paths-use-repo-alias-prefix`). This decouples belief source references from absolute paths, making the belief network portable across machines.

`resolve_source_path` follows a strict precedence chain: `db_dir` resolution takes priority over agent repo resolution, which takes priority over repo-key-split resolution (`resolve-source-path-db-dir-precedence`). When no `repos` mapping is provided, the function falls back to `~/git/<repo-name>/<relative-path>` by convention (`resolve-source-defaults-to-home-git`, `resolve-source-fallback-path`). Critically, the function returns `None` for nonexistent files or empty source strings rather than raising exceptions (`resolve-source-path-never-raises`, `resolve-source-path-returns-none-on-missing`), making it safe to call unconditionally in pipeline contexts (`source-resolution-is-convention-based-and-fail-safe`).

## Source Metadata

Belief source tracking uses flat file-level pointers — a source path and a source hash — with no structural awareness of where within a document the claim originated (`belief-source-metadata-is-file-level`). Section-level, page-level, or paragraph-level provenance is not supported; a belief points to an entire file, not a specific location within it.

## Integrity Tracking

Source integrity tracking uses full 64-character SHA-256 digests with exact string comparison for collision resistance (`source-tracking-is-collision-resistant-and-safe`). The hash backfill process is additive by default — it populates missing hashes without overwriting existing values, preventing accidental data loss during bulk operations.

These two stages — hashing and detection — form a complete end-to-end pipeline with no gap between measurement and verification (`source-integrity-spans-hashing-through-detection`).

## Staleness Detection

The `check_stale` function detects source staleness by comparing the stored hash against the current file contents on disk. However, it can only detect *source staleness* (the file changed) and not *world staleness* (the belief is outdated even though the source file hasn't changed) — beliefs can silently decay without any artifact triggering re-examination (`check-stale-detects-source-staleness-only`).

A node must have both a non-empty `source` and a non-empty `source_hash` to be eligible for staleness checking; nodes missing either field are silently skipped (`check-stale-requires-both-source-fields`). When a source file is missing from disk, `check_stale` returns a result with `reason="source_deleted"` and `None` for both `new_hash` and `source_path` (`check-stale-source-deleted-returns-none-hashes`). This behavior was introduced as a fix for issue #25 — previously, missing source files were silently skipped, a defect captured by the now-retracted belief `missing-source-file-is-silent` (OUT). Multiple nodes referencing the same missing file each produce independent `source_deleted` results with no deduplication by file path (`check-stale-no-dedup-by-source-path`).

## Source Chunks in LLM Interactions

Source document content included in the initial ask prompt is re-included in all subsequent prompts after tool-call round-trips, ensuring the LLM retains source context across the entire multi-turn loop (`ask-source-chunks-persist-across-tool-iterations`). When searching source chunks, `_search_source_chunks` filters out single-character tokens and common stop words before constructing FTS5 queries, returning an empty string when no usable terms remain (`search-source-chunks-filters-stop-words`).

## The Fail-Safe Pipeline

The complete source integrity pipeline is designed to be end-to-end fail-safe (`source-pipeline-is-end-to-end-fail-safe`). No stage raises exceptions on adverse conditions: resolution returns `None` on missing files, hashing backfills additively without overwriting, and staleness detection catches all drift with CI-ready exit codes. Each stage's output gracefully feeds the next.

This pipeline operates within the same deterministic, architecturally-grounded lifecycle framework as all other system operations (`source-integrity-is-deterministic-and-architecturally-grounded`), achieving triple assurance from two independent chains: fail-safe operation combined with architectural determinism (`source-integrity-is-fail-safe-deterministic-and-grounded`). The integrity guarantees extend from source files through TMS truth maintenance — convention-based resolution, collision-resistant hashing, fail-safe error handling, and contradiction-triggered backtracking together ensure no failure in either source verification or truth maintenance can corrupt system state (`source-to-tms-integrity-is-deterministic-and-exception-safe`).

## Source Integrity and Lifecycle Governance

Source integrity and lifecycle governance form a self-reinforcing closed loop (`source-integrity-and-governance-form-closed-loop`): source integrity enables lifecycle governance by unifying determinism, exception safety, and lifecycle management into a single pipeline, while lifecycle governance achieves gap-free source coverage — ensuring every source material change on disk is detected, surfaced, and managed through the full belief lifecycle without gaps (`source-lifecycle-is-fail-safe-and-gapless`).

The source integrity pipeline simultaneously serves three roles: pipeline correctness (deterministic path resolution and hashing), failure recovery (exception-safe resolution and contradiction-triggered backtracking), and lifecycle governance (driving staleness decisions and belief currency management) (`source-integrity-unifies-determinism-exception-safety-and-lifecycle`).

This closed loop is itself dually grounded — the governance half rests on two independent semantic foundations (evaluation purity and edge-case uniformity), so the feedback cycle remains sound even if one grounding chain is weakened (`source-integrity-loop-is-dually-grounded`). Combined with fully deterministic operation, the loop achieves both epistemic independence and operational predictability from independent foundations (`source-governance-loop-is-dually-grounded-and-deterministic`).

## Retracted Claims

Several higher-order beliefs about source integrity have been retracted. The claim that system integrity is enforced agnostically across both the internal/external boundary and across mutation sources (OUT, `integrity-is-boundary-and-source-agnostic`) lost support when its dependencies were retracted. Similarly, a chain of beliefs linking source-grounded correction to tripartite operational assurance and resource-efficient guarantees (`source-grounded-correction-produces-governed-output`, `source-grounded-correction-has-tripartite-assurance`, `source-correction-achieves-resource-efficient-assurance` — all OUT) were retracted along with `source-integrity-grounds-lifecycle-self-correction` (OUT), which had connected the fail-safe source pipeline to lifecycle-spanning self-correction claims. These retractions reflect refinements in the network's understanding of how source integrity relates to the system's broader operational guarantees.
