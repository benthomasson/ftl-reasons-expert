# source

[Back to index](index.md)

Source tracking in ftl-reasons connects beliefs to the files they were extracted from, enabling the system to detect when source material changes on disk and flag beliefs that may need re-examination. The mechanism spans path resolution, cryptographic hashing, and staleness detection, forming an end-to-end pipeline that is both fail-safe and deterministic.

## Source Path Format and Resolution

Node source references use a `repo-alias/relative-path` convention, where the first `/`-delimited segment serves as a key into a `repos` dictionary mapping aliases to filesystem roots (source-path-format, source-paths-use-repo-alias-prefix). This decouples belief source references from absolute paths, making the belief network portable across machines.

The `resolve_source_path` function follows a strict precedence chain: `db_dir` resolution takes priority over agent repo resolution, which in turn takes priority over repo-key-split resolution (resolve-source-path-db-dir-precedence). When no `repos` mapping is provided, the function falls back to `~/git/<repo-name>/<relative-path>` by convention (resolve-source-defaults-to-home-git, resolve-source-fallback-path).

A critical design choice is that `resolve_source_path` never raises exceptions. If the resolved file does not exist on disk or the source string is empty, it returns `None` instead (resolve-source-path-never-raises, resolve-source-path-returns-none-on-missing). Callers must check for `None`, but the pipeline never crashes on adverse filesystem conditions. This fail-safe behavior, combined with the convention-based format, makes source resolution both safe and predictable (source-resolution-is-convention-based-and-fail-safe).

## Source Metadata

Source tracking operates at the file level: each belief stores a flat `source` path and a `source_hash`, with no structural awareness of where within the document the claim originated (belief-source-metadata-is-file-level). Section-level, page-level, or paragraph-level provenance is not supported. This means that any change to a source file — even to an unrelated section — will flag all beliefs sourced from that file as potentially stale.

## Hashing and Integrity

Source integrity tracking uses full 64-character SHA-256 digests with exact string comparison, providing collision resistance for drift detection (source-tracking-is-collision-resistant-and-safe). The `hash_sources` operation backfills hashes additively without overwriting existing values, so running it against a database that already has hashes will not cause accidental data loss.

Together, hashing and staleness detection form a complete pipeline with no gap between measurement and verification (source-integrity-spans-hashing-through-detection): SHA-256 computes the integrity markers, and staleness detection consumes them to catch all source drift.

## Staleness Detection

The `check-stale` command detects **source staleness** — whether the source file has changed on disk since the hash was recorded — via hash comparison. However, it cannot detect **world staleness**, where a belief becomes outdated even though its source file has not changed (check-stale-detects-source-staleness-only). Beliefs can silently decay without any artifact triggering re-examination.

A node must have both a non-empty `source` and a non-empty `source_hash` to be eligible for staleness checking; nodes missing either field are silently skipped (check-stale-requires-both-source-fields). When a source file has been deleted from disk, `check_stale` returns a result with `reason="source_deleted"` and `None` values for `new_hash` and `source_path` (check-stale-source-deleted-returns-none-hashes) — a fix for issue #25, which previously caused deleted-file cases to be silently skipped. If multiple nodes reference the same missing file, each produces an independent `source_deleted` result with no deduplication by file path (check-stale-no-dedup-by-source-path).

## Source Chunks in LLM Interactions

Source document content plays a role in the LLM interaction loop as well. Content included in the initial ask prompt is re-included in all subsequent prompts after tool-call round-trips, ensuring the LLM retains source context across the entire multi-turn conversation (ask-source-chunks-persist-across-tool-iterations). When searching source chunks, the system filters out single-character tokens and common stop words before constructing FTS5 queries, returning an empty string when no usable terms remain (search-source-chunks-filters-stop-words).

## End-to-End Pipeline Properties

The complete source integrity pipeline — from path resolution through hash computation to staleness detection — is end-to-end fail-safe (source-pipeline-is-end-to-end-fail-safe). No stage raises exceptions on adverse conditions, and each stage's output gracefully feeds the next. This pipeline feeds directly into gapless lifecycle management, ensuring every source material change on disk is detected, surfaced, and managed through the full belief lifecycle (source-lifecycle-is-fail-safe-and-gapless).

The pipeline also operates within the same deterministic, architecturally-grounded lifecycle framework as all other system operations (source-integrity-is-deterministic-and-architecturally-grounded). It achieves triple assurance from two independent chains: fail-safe operation combined with architectural determinism (source-integrity-is-fail-safe-deterministic-and-grounded). The source-to-TMS path is both deterministic and exception-safe, meaning no failure in either source verification or truth maintenance can corrupt system state (source-to-tms-integrity-is-deterministic-and-exception-safe).

## Governance Integration

Source integrity and lifecycle governance form a self-reinforcing closed loop: source integrity enables governance by unifying determinism, exception safety, and lifecycle management into a single pipeline, while lifecycle governance achieves gap-free source coverage in return (source-integrity-and-governance-form-closed-loop). This loop is itself dually grounded — the governance half rests on two independent semantic foundations (evaluation purity and edge-case uniformity), so the bidirectional feedback cycle remains sound even if one grounding chain is weakened (source-integrity-loop-is-dually-grounded). The combined loop is both dually grounded and fully deterministic, achieving epistemic independence and operational predictability from independent foundations (source-governance-loop-is-dually-grounded-and-deterministic).
