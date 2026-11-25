---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Look for release notes, migrations, or issue context that indicate whether the four-value vocabulary is intended to grow or be normalized.

## Summary

The gathered observations **do not contain direct evidence** that a "four-value vocabulary" is intended to grow or to be normalized. None of the changelog entries, migration code, tests, or branch names reference a four-value truth/vocabulary concept. The evidence that *is* present points in adjacent directions (format normalization, defeater migration, index denormalization), so we can rule some things out but cannot answer the growth-vs-normalization question confidently.

## What the evidence shows

### 1. Changelog contains no vocabulary-growth or vocabulary-normalization entry

The full `CHANGELOG.md` (140 lines, versions 0.1.0 → 0.18.0) mentions truth values only generically:

- `CHANGELOG.md` (0.1.0 entry): "Initial TMS implementation: nodes, justifications, **truth value propagation**"
- `CHANGELOG.md` (0.18.0): access tags, RBAC, `trace-access-tags`, tag inheritance — unrelated to truth-value vocabulary
- `CHANGELOG.md` (0.16.0): `_retracted` flag behavior, internal metadata filtering — again unrelated

There is **no release note** adding, removing, renaming, or constraining a four-value set. If the vocabulary had been intentionally expanded, the project's Keep-a-Changelog format would likely have an "Added"/"Changed" bullet for it; none exists.

### 2. The only visible "migration" is about `retract_reason`, not vocabulary

- `reasons/api.py:1549` defines `migrate_retract_to_defeaters(...)`
- `reasons/api.py:1560` documents the return shape: `{"migrated": [...], "skipped": [...], "errors": [...]}`
- `reasons/api.py:1612` constructs `migrated-retraction-{nid}-j{satisfied_idx}` IDs
- `reasons/api.py:1621` writes `"defeater_type": "migrated-retraction"`

The accompanying tests (`tests/test_migrate_defeaters.py`) confirm the scope:

- Line 12 asserts `truth_value == "IN"`
- Line 18 asserts `truth_value == "OUT"`
- Line 50 asserts a node is `"OUT"`
- Line 57 asserts restoration back to `"IN"`

So this migration converts **string metadata (`retract_reason`) into graph-native defeater nodes**, and the truth-value set used in the tests remains the classical `IN`/`OUT`. It does not show any expansion or normalization of a four-value vocabulary.

### 3. "Normalization" in this codebase currently means format normalization

- `reasons/import_agent.py:88`: comment reads *"Format normalization — both formats produce the same claim structure:"*
- `reasons/import_agent.py:95` `_normalize_markdown`, `:131` `_normalize_json`, `:167` `_normalize_nogoods_markdown`, `:179` `_normalize_nogoods_json`
- `reasons/api.py:1949-1970` and `:2037-2058` wire these functions into import flows

These "normalize" markdown-vs-JSON input into a shared claim dict shape — **not** a controlled vocabulary of four values. The `search_vocabulary_and_normalization_terms` match list is consistent with this: every visible "normaliz" hit is about ingestion formats or nogoods, not truth values.

### 4. Issue/branch context is too thin and contains false positives

`search_issue_markers_and_release_context` only surfaced branch names in `.git/packed-refs`, not issue bodies. Notably:

- `.git/packed-refs:41` — `fix/issue-24-dependents-index-is-a-fragile-denormaliz` — "denormaliz" here refers to a **dependents index**, not vocabulary normalization
- `.git/packed-refs:56-74` — issues 171–210 cover propose-update, source-type, roundtrip-fidelity, cost-tracking, search-sources, wiki, cycle detection, lifecycle-timestamps, LLM providers — none about truth values or a four-value set

Also worth flagging: `reasons/api.py:3919` and `:3927` show the strings `'outdated', 'deprecated', 'fragile', ... 'legacy', 'unmaintained'`. That is a heuristic word list, so a naive "deprecated/normalize" keyword search can produce false positives here too.

### 5. Deprecation signals exist, but not for vocabulary

- `reasons/cli.py:3064` — `help="(deprecated alias for --domain-id)"` shows the project does use deprecation-with-alias patterns for CLI options
- `reasons/ask.py:177` — `"Backward-compat wrapper."`
- `reasons/cli.py:3853` — `research (backward-compatible alias for repair)`

These demonstrate the project's style for compatibility, but again none concern a four-value vocabulary.

## Conclusion on the question

Based on these observations, **there is no release note, migration, or issue context indicating the four-value vocabulary is either intended to grow or to be normalized.** The changelog shows incremental feature additions around access tags and migrations around `retract_reason`, and the tests still describe binary `IN`/`OUT` truth states. The visible "normalization" terminology is exclusively about import formats.

## What's missing

To actually answer the question, we would need:

1. The **definition site** of the four values (an enum, constant tuple, or set such as `TRUTH_VALUES`, `VALID_VALUES`, or similar) — not present in the observations.
2. The **database schema** for the relevant column (e.g., any `CHECK` constraint on `truth_value` in `reasons/storage.py`) — not present.
3. The **issue bodies** or PR descriptions behind `.git/packed-refs` branches — only branch names were captured.
4. Any **design/RFC docs** or `docs/` entries discussing vocabulary — not present.
5. A grep for the actual four identifiers across the codebase, so we can see whether they are compared with `==`, stored in sets, or validated exhaustively.

Without those, any claim about growth vs. normalization would be speculation.

## Topics to Explore

- [file] `reasons/storage.py` — Inspect table DDL for any `CHECK`/`ENUM`-style constraint that enumerates truth values, which would reveal whether the vocabulary is fixed or open.
- [file] `reasons/network.py` — Where `_propagate` and `_justification_valid` compare truth values; look for exhaustive `if/elif` vs. set-membership to infer whether the code assumes a closed four-value set.
- [function] `reasons/api.py:migrate_retract_to_defeaters` — The existing migration precedent; a future vocabulary normalization migration would likely follow the same `migrated`/`skipped`/`errors` shape.
- [general] `four-value-vocabulary-definition` — Grep for the actual four identifiers and any canonical set/constant; without locating this we can't judge growth vs. normalization.
- [general] `issue-tracker-acceptance-criteria` — Read full issue bodies (not just branch names) for issues 171–210 and any vocabulary-related ticket, since branch names alone are insufficient.

## Beliefs

- `changelog-has-no-four-value-vocabulary-entry` — The complete `CHANGELOG.md` (0.1.0–0.18.0, 140 lines) contains no entry adding, removing, or normalizing a four-value vocabulary.
- `migrate-defeaters-migrates-retract-reason` — `reasons/api.py:migrate_retract_to_defeaters` converts `retract_reason` metadata into defeater nodes; it does not alter truth-value vocabulary.
- `migrate-defeaters-tests-use-binary-truth-values` — `tests/test_migrate_defeaters.py` asserts only `"IN"` and `"OUT"` truth values, showing those two states persist through the migration.
- `import-agent-normalization-is-format-level` — `reasons/import_agent.py` `_normalize_*` functions normalize markdown-vs-JSON ingestion into a shared claim structure, not a truth-value vocabulary.
- `issue-24-denormalization-targets-dependents-index` — The branch `.git/packed-refs:41` (`fix/issue-24-dependents-index-is-a-fragile-denormaliz`) concerns index denormalization, not vocabulary normalization.
