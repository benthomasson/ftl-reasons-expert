---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Grep for `"CONFIRMED"`/`"STALE"`/`"PARTIAL"`/`"INCONCLUSIVE"` to see which downstream modules assume the four-value taxonomy and whether they defend against unknown values.

## What the grep shows

The four-value verification taxonomy is defined in `reasons/verify.py`:

- `reasons/verify.py:4` says verification returns `CONFIRMED/STALE/PARTIAL/INCONCLUSIVE`.
- `reasons/verify.py:23-26` defines each value in the prompt.

Only two observed downstream modules actually consume that verdict vocabulary:

1. **`reasons/api.py` — applies verification verdicts**
   - `reasons/api.py:3800-3801` documents that `CONFIRMED` beliefs get `verified_at`, and `STALE` beliefs are retracted when requested.
   - `reasons/api.py:3874` defaults a missing verdict to `INCONCLUSIVE`:
     ```python
     v = verdict.get("verdict", "INCONCLUSIVE")
     ```
   - `reasons/api.py:3876-3878` branches only on `CONFIRMED` and `STALE`.

   **Unknown-value behavior:** it defends against a missing `"verdict"` key, but not against an unrecognized string. An unknown verdict such as `"BOGUS"` matches neither branch in the observed snippet, so it is likely silently ignored / treated as a no-op. The same is true for `PARTIAL` and `INCONCLUSIVE` in this block: they are not action-triggering values here.

2. **`reasons/cli.py` — displays verification verdicts**
   - `reasons/cli.py:2236-2237` maps all four values:
     ```python
     "CONFIRMED": "+", "STALE": "-",
     "PARTIAL": "~", "INCONCLUSIVE": "?",
     ```
   - `reasons/cli.py:2245` defaults a missing verdict to `INCONCLUSIVE`.
   - `reasons/cli.py:2256-2260` branches on `CONFIRMED`, `STALE`, and `PARTIAL`.

   **Unknown-value behavior:** the observed lines only default a missing key. They do not show explicit validation of unknown strings. Whether an unknown value renders as `?` or raises depends on the omitted lines around the symbol lookup; the snippet is insufficient to confirm. There is no visible unknown-verdict guard.

## Other `STALE` matches are mostly a different concept

Most other `STALE` hits are not the four-value verification verdict. `STALE` is overloaded across the codebase:

- **Imported belief status**: `reasons/import_agent.py:105` treats `("STALE", "OUT")` as retracted/out. Unknown statuses silently fall through as “not out.”
- **Belief markdown importer**: `reasons/import_beliefs.py:75` only parses headers matching `(IN|OUT|STALE)`. Unknown statuses do not match the regex. `reasons/import_beliefs.py:270-271` retracts only `STALE`/`OUT`.
- **PostgreSQL import path**: `reasons/pg.py:2479` has the same `("STALE", "OUT")` retraction check. Unknown values are not explicitly rejected.
- **Markdown export**: `reasons/export_markdown.py:55` maps an `OUT` node with `stale_reason`/`retract_reason` metadata to exported status `STALE`. `reasons/export_markdown.py:104` then checks `status in ("STALE", "OUT")`. Unknown truth/status values would be emitted or skipped according to those narrow membership checks.
- **Source-file staleness**: `reasons/check_stale.py` is about source hashes and returns reasons such as `source_deleted`, `truncated_hash`, and `content_changed`; it does not consume verification verdicts. `reasons/cli.py:1140` prints a `STALE` marker for that separate check.
- **Proposal status**: `reasons/cli.py:2019` prints “Proposal ... marked STALE” — again unrelated to verification verdicts.

Modules such as `reasons/export_card.py`, `reasons/report_belief.py`, `reasons/report_gated.py`, and `reasons/build_wiki.py` have no uppercase taxonomy matches in the provided results; they operate on `truth_value` values like `IN`/`OUT`, not the verification verdict enum.

## Practical conclusion

- The only full downstream enumerator of all four verification values is the CLI display mapping in `reasons/cli.py:2236-2237`.
- The API verdict applier in `reasons/api.py:3874-3878` only explicitly handles `CONFIRMED` and `STALE`, with a missing-key default to `INCONCLUSIVE`.
- No observed consumer explicitly validates unknown verdict strings. Most defenses are “default missing key” or “check membership in a narrow tuple,” which means unknown values tend to be silently ignored rather than rejected.
- The codebase has a naming collision: verification `STALE` vs imported-status `STALE` vs source-file staleness. That is worth treating carefully when changing the taxonomy.

One limitation: the JSON reports `total_count: 63` matches but only lists 29. The conclusions above cover the listed matches; the omitted matches could include additional consumers.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the full verdict-application function around lines 3800–3885 to confirm whether `PARTIAL`/unknown values fall through silently or hit an unseen `else`.
- [file] `reasons/cli.py` — Check lines 2230–2265 to see whether verdict symbols are read with `.get(..., "?")` or direct indexing, which determines unknown-value behavior.
- [file] `reasons/import_beliefs.py` — Review the status parser and retraction mapping to confirm how unknown imported statuses are skipped or rejected.
- [general] `stale-semantics` — Compare verification-verdict `STALE`, imported-status `STALE`, and source-staleness `STALE` across `verify.py`, `import_*.py`, `export_markdown.py`, and `check_stale.py`.
- [file] `reasons/verify.py` — Confirm the LLM-output contract and whether any parsing/validation layer enforces the four allowed verdict strings.

## Beliefs

- `four-value-taxonomy-is-defined-in-verify-py` — `reasons/verify.py:4` and `:23-26` define `CONFIRMED`, `STALE`, `PARTIAL`, and `INCONCLUSIVE` as the verification verdict set.
- `api-verify-consumer-defaults-missing-verdict-to-inconclusive` — `reasons/api.py:3874` uses `verdict.get("verdict", "INCONCLUSIVE")`.
- `api-verify-consumer-handles-only-confirmed-and-stale` — `reasons/api.py:3876-3878` branches only on `CONFIRMED` and `STALE`; no `PARTIAL` or unknown branch appears in the observed snippet.
- `cli-verdict-display-maps-all-four-values` — `reasons/cli.py:2236-2237` maps all four verdicts to display symbols, and `:2245` defaults missing verdicts to `INCONCLUSIVE`.
- `imported-status-retraction-recognizes-only-stale-and-out` — `reasons/import_agent.py:105`, `reasons/import_beliefs.py:271`, and `reasons/pg.py:2479` retract/flag only statuses in `("STALE", "OUT")`.
