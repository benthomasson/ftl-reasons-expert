---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The collection of `*_review_result` / `last_*_reviewed` / `reviewed_at` keys scattered across `review_beliefs`, `review_justifications`, and `review_premises` — worth understanding whether there's a unified schema or per-reviewer ad-hoc keys.

# Review metadata keys: no unified schema, just three separate conventions

## Verdict

There is **no unified schema**. Each reviewer that persists anything does so under its own ad‑hoc convention, and one reviewer (`review_justifications`) persists nothing at all. The closest thing to a shared primitive is the `reviewed_at` field on the node dataclass — but even that is only used by one of the two persisting reviewers, and the read path still falls back to the legacy `last_reviewed` metadata key.

## Per-reviewer breakdown

| Reviewer | Timestamp storage | Result storage | Persists? |
|---|---|---|---|
| `review_beliefs` | `node.reviewed_at` (first-class node field) | `node.metadata["review_result"]` | Yes (`api.py:4158–4160`) |
| `review_premises` | `node.metadata["last_premise_reviewed"]` | `node.metadata["premise_review_result"]` | Yes (`api.py:4324–4330`) |
| `review_justifications` | — | — | **No** — `test_does_not_modify_db` in `tests/test_review_justifications.py` asserts the DB is unchanged |

### `review_beliefs` — node field + unsuffixed metadata
- `reasons/__init__.py:47` defines `reviewed_at: str = ""` **as a dataclass field**, not a metadata key.
- `reasons/api.py:4158` writes `net.nodes[nid].reviewed_at = now`.
- `reasons/api.py:4160` writes `net.nodes[nid].metadata["review_result"] = _classify_review_result(result_map[nid])`. That function (signature at `api.py:4049`, body not shown in observations) collapses the rich reviewer output (`valid`, `sufficient`, `necessary`, `defeat_reason_type`, `scope_findings`, …) into a single string.
- Both writes are unconditional per reviewed node.
- Deserialization reads the field back at `reasons/api.py:2187` and `:2231` via `ndata.get("reviewed_at", "")`.

### `review_premises` — prefixed metadata keys
- `reasons/api.py:4324` writes `net.nodes[nid].metadata["last_premise_reviewed"] = now`.
- `reasons/api.py:4326–4330` writes `premise_review_result` with three possible value sources:
  - `"pass"` when the premise is accurate,
  - `r.get("error_type", "inaccurate")` when inaccurate,
  - the literal `"overgeneralized"` in a separate branch (likely the well-scoped-only failure path).
- The API docstring at `api.py:4244–4245` explicitly states it “Writes last_premise_reviewed and premise_review_result metadata”.
- Test `test_stores_metadata` in `tests/test_review_premises.py` asserts both keys land on the node, and `test_dry_run_no_metadata` asserts `dry_run=True` suppresses both — so persistence is gated.

Note: the value vocabularies differ (`review_result` is a classification string produced by `_classify_review_result`; `premise_review_result` uses the premise-level `error_type` values `misread_source`, `fabricated`, `unsupported`, `overgeneralized`, or `"pass"`).

### `review_justifications` — read-only
- `reasons/review_justifications.py` contains no `metadata[...]=` or `reviewed_at=` writes. Its only side effects are the return value and the `on_batch` callback.
- `tests/test_review_justifications.py::TestApiReviewJustifications::test_does_not_modify_db` confirms that after `api.review_justifications(...)`, the node's justifications are unchanged. There is no `justification_review_result` / `last_justification_reviewed` key anywhere in the grep matches.
- So results of justification-mode review are ephemeral unless the caller serializes the returned summary itself.

## Evidence of schema drift

The read/export path reveals that `reviewed_at` **replaced** an earlier `last_reviewed` metadata key, but backward compatibility was never removed:

- `reasons/api.py:3476`: `if node.reviewed_at or node.metadata.get("last_reviewed"):`
- `reasons/api.py:3481`: `last = node.reviewed_at or node.metadata.get("last_reviewed", "")`
- `reasons/api.py:3499`: emits `"last_reviewed": node.reviewed_at or node.metadata.get("last_reviewed")`
- `reasons/api.py:3500`: emits `"review_result": node.metadata.get("review_result")`

So in the export shape the “new” field is surfaced under the **old key name** (`last_reviewed`), meaning an external consumer sees keys (`last_reviewed`, `review_result`) that don’t match the storage location (`reviewed_at` field, `review_result` metadata) nor the premise shape (`last_premise_reviewed`, `premise_review_result`).

Counting distinct keys across the codebase from the observations:

1. `reviewed_at` — node field (`__init__.py:47`), written by belief review
2. `review_result` — metadata, written by belief review, read at `api.py:3500`
3. `last_reviewed` — legacy metadata, read-only fallback in `api.py:3476–3499`
4. `last_premise_reviewed` — metadata, written by premise review
5. `premise_review_result` — metadata, written by premise review
6. *(nothing)* — justification review

That’s four persisted keys/fields for two persisting reviewers, plus one non-persisting reviewer — the definition of per-reviewer ad hoc.

## What’s missing from the observations

- The body of `_classify_review_result` (`reasons/api.py:4049`) isn’t shown, so the exact string vocabulary of `review_result` (as distinct from premise error types) isn’t visible.
- The grep reports **134 matches but only 29 are shown**, so there may be additional key variants (e.g., CLI output, other API methods, `updated_at` interplay) not covered.
- The CLI dispatch (`cmd_review_premises` etc.) is referenced by tests but not shown; it may expose additional naming.
- Whether any code path *writes* `last_reviewed` today (or only reads it) isn’t provable from the visible lines — 3476/3481/3499 are all reads or fallbacks.

## Topics to Explore

- [function] `reasons/api.py:_classify_review_result` — Body of the classifier that collapses the belief-review dict (valid/sufficient/necessary/defeat_reason_type) into the single string stored under `metadata["review_result"]`.
- [file] `reasons/api.py` — Sections around lines 3460–3510 (export/stat path mixing `reviewed_at` and legacy `last_reviewed`) and 4040–4440 (belief vs premise persistence); clarifies which key is canonical and whether `last_reviewed` is still written anywhere.
- [file] `reasons/__init__.py` — Node dataclass definition; confirm `reviewed_at` / `updated_at` are first-class fields while `metadata` is a free-form dict, which is the root cause of the split storage.
- [general] `full-review-metadata-grep` — Re-run the `review_result|last_.*_reviewed|reviewed_at` grep across all 134 matches (tests, CLI, docs) to catalog every key variant not visible here.
- [general] `cli-review-dispatch` — Where `cmd_review_premises` / `cmd_review_beliefs` / `cmd_review_justifications` serialize their return summaries; may reveal an external-facing key schema separate from persisted metadata.

## Beliefs

- `review-beliefs-writes-reviewed-at-and-review-result` — `review_beliefs` (via `api.py:4158–4160`) writes `node.reviewed_at` and `node.metadata["review_result"]` for every reviewed node.
- `review-premises-writes-prefixed-metadata-keys` — `review_premises` writes `last_premise_reviewed` and `premise_review_result` to `node.metadata` (never to `reviewed_at`), and honors `dry_run` to suppress both.
- `review-justifications-does-not-persist` — `review_justifications` returns results but writes no metadata or node fields; `test_does_not_modify_db` enforces this.
- `review-metadata-schema-is-per-reviewer` — There is no shared helper that defines review metadata keys; each persisting reviewer embeds its own string literals at the call site (`api.py:4160`, `api.py:4324–4330`).
- `reviewed-at-maps-to-last-reviewed-in-export` — The export path (`api.py:3499`) surfaces `node.reviewed_at` under the legacy key name `last_reviewed`, falling back to `metadata["last_reviewed"]`, so external keys diverge from internal storage keys.
