---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Define the canonical finding type and fields for cyclic antecedent graphs.

# Defining a Canonical Finding Type for Cyclic Antecedent Graphs

## Short answer: it isn't defined yet in this codebase

The observations do **not** contain a `Finding` dataclass, a `finding_type` enum/field, or any cyclic-antecedent detection logic. The two grep passes confirm this:

- The `Finding|finding_type|canonical` search returns **50** matches, none of which are a `Finding` type. The only `canonical` references are `mark_duplicate` (`reasons/api.py:1242-1277`) and `duplicate_of` metadata handling in `reasons/build_wiki.py:63-68`.
- The `cyclic|antecedent|cycle` search over tests returns **0** matches (`search_cyclic_tests.total_count == 0`). No test file exercises cyclic antecedent graphs.

So "define the canonical finding type" is a greenfield design task in this repo, not a documentation task. What follows is what the code *does* give you to build on, plus what is missing.

## What the antecedent model actually is

The TMS justification model lives in `reasons/__init__.py`:

- `reasons/__init__.py:11` — *"SL (Support List): node is IN iff ALL antecedents (inlist) are IN"* — this is a classic well-founded SL network. Cycles are exactly what break this invariant (a node cannot be IN only if itself is IN).
- `reasons/__init__.py:20` — `antecedents: list[str] = field(default_factory=list)` — antecedents are stored as a flat list of node IDs.

Justifications are constructed in `reasons/api.py`:
- `reasons/api.py:323`, `:330`, `:440`, `:443` — antecedents parsed from comma-separated `sl`/`cp` arguments.
- `reasons/api.py:352-355` — the namespace `ns:active` is prepended as the first antecedent, so a node's antecedent list can include itself if the namespace resolves back to the node.
- `reasons/api.py:367`, `:454` — `_resolve_namespace` rewrites antecedent IDs, another path that could synthesize a self-reference.

None of these paths detect a cycle — they will happily accept `p -> q -> p`.

## The three existing "finding-like" shapes (none cyclic)

The codebase already has three structured result shapes you could align with, but **none of them model a cycle**:

| Shape | File | Fields |
|---|---|---|
| Verification verdict | `reasons/verify.py` (`parse_verify_response`, `verify_beliefs`) | `verdict` (`CONFIRMED`/`STALE`/`PARTIAL`/`INCONCLUSIVE`), `reason`, `quote` |
| Repair triage pattern | `reasons/repair.py` (`VALID_PATTERNS`) | `{"pattern": "search_and_link" \| "soften" \| "abandon" \| "research", "rationale"}` |
| Duplicate marking result | `reasons/api.py:1242-1277` (`mark_duplicate`) | `{"source_id", "canonical_id", "changed"}` |

For consistency, a cyclic-finding type would most naturally be a fourth structured dict/dataclass, not a new ad-hoc string format.

## What a canonical cyclic-antecedent finding type would need

Given only the above, the minimum set of fields to build on the existing model:

- **`type`** — must be a new tag (e.g., `"cyclic_antecedent"`); none of the existing `VALID_PATTERNS` (`reasons/repair.py:...`) cover cycles.
- **`nodes`** — the ordered list of node IDs forming the cycle. This is directly derivable from `Justification.antecedents` (`reasons/__init__.py:20`) by DFS/walk.
- **`justification_ids`** — the justifications whose `outlist`/`antecedents` participate in the cycle.
- **`path`** — the explicit edge sequence (`p1 -> p2 -> ... -> p1`) so a reviewer can validate the cycle without recomputing.
- **`reason`** — matching the `verify`/`repair` convention (`reasons/verify.py`, `reasons/repair.py`) of a human-readable explanation.
- **`severity`** — because SL cycles are *hard* violations (they break `reasons/__init__.py:11`), whereas CP cycles may be benign.

## What's missing to actually implement it

The observations do not show:

1. **A dependency/graph traversal utility.** `reasons/network.py:30` mentions *"single canonical implementation"* but no cycle-walking function is shown.
2. **Any `Finding` dataclass or Pydantic model.** Grep for `Finding|finding_type|canonical` returns only `duplicate_of`.
3. **Any cycle test fixture.** `tests/test_verify.py` and `tests/test_deduplication.py` are shown; `search_cyclic_tests` is empty.
4. **Where findings are surfaced to the user.** `reasons/cli.py:2454` writes `# Belief Review Findings`, but the renderer for those findings is not in the observations.

To answer "define the canonical finding type," you would first need to inspect `reasons/network.py` (the canonical storage/rebuild module), `reasons/storage.py:432` (rebuild of the dependent index from justifications), and search for the review-finding renderer used at `reasons/cli.py:2454`.

## Topics to Explore

- [file] `reasons/network.py` — Claimed "single canonical implementation" of the belief network; likely the right home for a cycle-detection pass and a `Finding` dataclass.
- [file] `reasons/__init__.py` — Defines `Justification` with the `antecedents: list[str]` field (line 20) that any cyclic-finding type must consume.
- [function] `reasons/api.py:mark_duplicate` — Closest existing precedent for a structured `{source_id, canonical_id, changed}` finding return; useful as a shape template.
- [file] `reasons/repair.py` — Defines the `VALID_PATTERNS` triage vocabulary; a cyclic-finding type would need to slot into or extend this.
- [general] `tms-well-foundedness` — The SL semantics at `reasons/__init__.py:11` require a well-founded network; understanding how cycles violate it informs severity levels.

## Beliefs

- `justification-has-antecedents-list` — `Justification` stores antecedents as a `list[str]` via `field(default_factory=list)` at `reasons/__init__.py:20`.
- `sl-requires-all-antecedents-in` — SL justifications require *all* antecedents to be IN for a node to be IN (`reasons/__init__.py:11`), which makes cycles a hard semantic violation.
- `no-finding-type-exists` — No `Finding` dataclass or `finding_type` field is present in the observed files; the only `canonical` references are the `duplicate_of` metadata path in `reasons/api.py:1267`.
- `verify-produces-verdict-dict` — `verify_beliefs` returns `{belief_id: {"verdict", "reason", "quote"}}` (`reasons/verify.py`, `parse_verify_response`).
- `no-cyclic-tests-exist` — The cyclic/antecedent/cycle search over tests returns zero matches, so any cyclic-finding implementation currently has no regression coverage.
