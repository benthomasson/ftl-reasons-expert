---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Trace one belief through ingest → verify → review → repair → export to understand the round-trip.

# Tracing one belief through the pipeline

Let's follow `derived-1` — "REST + YAML means declarative API." — added by `TestFullRoundTrip.test_all_fields_survive` (in `tests/test_roundtrip.py`). It has `sl="obs-1,obs-2"`, `source_type="derived"`, and `accepted_pr="https://github.com/org/repo/pull/11"`, and it is exactly the kind of derived node that every stage below is designed to exercise.

## 1. Ingest — `reasons/import_beliefs.py`

The markdown arrives in a whitespace and YAML frontmatter header. The pipeline runs:

1. **`strip_frontmatter(text)`** splits `---`…`---` from the body and returns a flat dict (`schema_version`, `project_name`, `created_at`). `import_into_network` copies those three keys into `network.meta` (test: `test_frontmatter_round_trip`).
2. **`parse_repos(body)`** scans the `## Repos` section into `network.repos`.
3. **`parse_beliefs(body)`** iterates lines. The header regex `^### (\S+) \[(IN|OUT|STALE)\]\s*(.*)` captures `id`, `status`, and `type`. Every recognised field is accumulated into a dict: `source_url`, `source_hash`, `date`, `depends_on`, `unless`, `source_type`, `accepted_pr`, `stale_reason`, `superseded_by`. For `derived-1` this yields `depends_on=["obs-1","obs-2"]`, `source_type="derived"`, `accepted_pr="…/pull/11"`.
4. **`import_into_network`** does a topological pass so premises are added before derived nodes (`claim_by_id`, `remaining`/`added` loop, "Iterative topological sort"). For `derived-1` the deps are all in `claim_by_id` and already `added`, so it lands in `ordered`.
5. A single `Justification(type="SL", antecedents=["obs-1","obs-2"], outlist=[], label="imported from beliefs: DERIVED")` is constructed. Metadata is assembled: `beliefs_type="DERIVED"`, `source_type="derived"`, `accepted_pr="…/pull/11"`.
6. `network.add_node(...)` propagates: because both antecedents are `IN` premises with no justifications, `derived-1` is computed `IN`.

The important non-obvious fact: **`parse_beliefs` has no representation for "multiple independent justifications"** — everything collapses into one `depends_on` list and one `unless` list.

## 2. Storage at rest — `reasons/storage.py`

`save(network)` rewrites the whole DB in one transaction. For `derived-1`:

- A row in `nodes` with `source`, `source_url`, `source_hash`, `date`, `metadata_json` (JSON of `{beliefs_type, source_type, accepted_pr}`).
- One row in `justifications` with `antecedents_json='["obs-1","obs-2"]'`, `outlist_json='[]'`.
- A row in `nodes_fts`.
- Network metadata into `network_meta`, repos into `repos`.

`load()` reverses this: rows are rehydrated into `Node` objects with `source_url`, `metadata=json.loads(meta_json)`, and `network._rebuild_dependents()` is called at the end — the derived index is reconstructed from justifications, not from any stored `dependents` table.

Note the explicit "snapshot external tags/sources before clearing" dance at the top of `save()` — `add_tags()` and `add_source()` write directly to `node_tags`/`node_sources` and would otherwise be blown away by the `DELETE`-and-rewrite strategy.

## 3. Verify — `reasons/verify.py`

`verify_beliefs` is scoped to **premises with a source file** (it is not what review runs on). For the sibling `obs-1` ("The API uses REST endpoints.", `source="repo/api.py"`):

- `read_source("repo/api.py", db_path)` calls `resolve_source_path(...)` from `check_stale`, reads the file, and truncates at `MAX_SOURCE_CHARS = 30_000`.
- `format_belief_for_verify` builds a prompt block containing the id, claim text, `source`, `source_url`, and the source body in a fenced block.
- `verify_beliefs` joins all belief blocks with `\n\n---\n\n`, invokes the LLM once, and `parse_verify_response` extracts a JSON object mapping id → `{verdict, reason, quote}` where verdict ∈ `CONFIRMED | STALE | PARTIAL | INCONCLUSIVE`.

**`verify_beliefs` does not mutate the network.** It returns the dict; whatever writes `nodes.verified_at` and any verdict metadata lives in a caller (likely `reasons/api.py` or `reasons/cli.py`, neither shown in the observations). `derived-1` has no `source`, so `read_source` returns `None` and any call to verify for it would come back `INCONCLUSIVE`.

## 4. Review — `reasons/review.py`

`review_beliefs(nodes, belief_ids=None, ...)` defaults `belief_ids` to `IN` nodes with at least one justification — this is exactly how a derived node like `derived-1` enters the loop, and how premises like `obs-1` are excluded.

- `format_belief_for_review` renders `### derived-1`, the claim text, and every justification's antecedent texts / outlist texts. Errors like "(not found in network)" are inserted if a referenced id is missing.
- Batches of `REVIEW_BATCH_SIZE = 20` go to the LLM. `on_batch` can be used to stream partial results.
- `parse_review_response` uses `json.JSONDecoder.raw_decode` starting at each `[` — this is what tolerates prose brackets in front of the JSON array.
- Each returned object is normalised to `{id, valid, sufficient, necessary, unnecessary_antecedents, comment, scope_findings, missing_property, defeat_reason_type}`.

When `valid=false`, the prompt requires `scope_findings` entries (antecedent → establishes / does_not_establish) and `missing_property`, which are the raw material for `defeat_with_scope` downstream.

## 5. Repair — `reasons/repair.py`

`repair_beliefs(review_results, nodes, ...)` starts with `invalid = [r for r in review_results if not r.get("valid", True)]` — only flagged beliefs proceed. For `derived-1`:

1. **Triage** (`triage_belief` with `TRIAGE_PROMPT`) classifies into `search_and_link | soften | abandon | research`, using `_compute_depth` (max antecedent depth + 1) and `flagged_ancestors` as inputs. `parse_triage_response` validates against `VALID_PATTERNS`.
2. **`search_and_link`** (`_do_search_and_link`):
   - `extract_smuggled_claim` asks the LLM for one sentence — the fact the conclusion assumes but antecedents don't establish.
   - `search_fn` defaults to `api.search(query, include_out=True, db_path=db_path)`; the JSON results are filtered: skip the belief itself, skip existing antecedents, keep only `truth_value=="IN"` nodes that are premises (no justifications).
   - `find_matching_premises` runs `MATCH_PROMPT`; `parse_match_response` filters ids to the valid candidate set.
   - On a match, `api.add_justification(belief_id, sl="obs-1,obs-2,<matched>", unless="…", label="repair: linked …")` is called.

**The crucial detail:** repair **adds a new justification**, it does not edit or delete the original one. Whether the old justification continues to count as "supporting" depends on logic in `reasons/network.py` (not shown) — `supporting_justification` is persisted in the `nodes` table.

3. **`soften`** rewrites the claim text via `SOFTEN_PROMPT` so it follows strictly from antecedents. **`abandon`** and **`research`** live in the missing ~218 lines of `repair.py` (the observation only returns lines 0–400 of 618).

## 6. Export — `reasons/export_markdown.py`

`export_markdown(network, repos)` rebuilds the markdown:

- Frontmatter via `build_meta(project_name, node_count, created_at)` — `updated_at` and `generator` are always emitted.
- Nodes sorted `IN` first, then `OUT`, alphabetical within each group.
- **Status mapping**: `status = node.truth_value`; if `OUT` and metadata has `stale_reason` or `retract_reason`, it becomes `STALE`.
- **`btype`**: `metadata.get("beliefs_type")` if present, else `DERIVED` when the node has justifications, else `OBSERVATION`. So `derived-1` re-emits `### derived-1 [IN] DERIVED`.
- Fields written, in order: text, `Source`, `Source URL`, `Source type`, `Source hash`, `Date`, `Depends on`, `Unless`, `Defeated by` (only outlist entries whose defeater node has `metadata["defeats_node"] == node.id`), `Stale reason`, `Duplicate of`, `Superseded by`, `Accepted PR`.
- **Deps are a union across all justifications.** This is where the "one justification makes it, the other widens it" problem bites: `derived-1` with two justifications (`[SL: obs-1,obs-2]` and `[SL: obs-1,obs-2,matched]` from a repair) exports as a single `- Depends on: obs-1, obs-2, matched`. On `import_into_network` this collapses to one SL requiring all three — semantically **stronger** than the original.

## 7. Round-trip guarantees — `tests/test_roundtrip.py`

- `TestFullRoundTrip.test_all_fields_survive` asserts text, `source`, `source_url`, `source_hash`, `date`, `metadata["source_type"]`, `metadata["accepted_pr"]`, antecedent set, and counts on `obs-1`, `obs-2`, `derived-1` after `export_markdown → import_into_network`.
- `test_double_round_trip_stability` asserts bytewise equal markdown on export→import→export, ignoring lines starting with `updated_at`. This is the strongest structural guarantee in the repo.
- `test_stale_node_round_trip` covers `STALE` mapping through `retract()` and back.
- `test_unless_round_trip` covers `justification.outlist`.

## 8. Where the round trip leaks

- **Multiple justifications are not representable.** `parse_beliefs` produces one `depends_on`/`unless` pair, `export_markdown` emits a union across justifications, `import_into_network` builds one `Justification`. Any node with ≥2 justifications loses its alternative-support structure through markdown.
- **`- Defeated by:` is export-only.** `parse_beliefs` has branches for `- Source:`, `- Depends on:`, `- Unless:`, `- Accepted PR:`, `- stale reason:`, `- superseded by:`, `- Source type:`, and a catch-all `elif line.startswith("- "): pass`. `- Defeated by:` and `- Duplicate of:` fall into the catch-all and are silently dropped.
- **Nogood format mismatch.** `export_markdown` writes `### {ng.id}` with no colon; `parse_nogoods`'s regex is `^### (nogood-\S+?):\s*(.*)` — colon required. `test_nogoods_export_format` explicitly documents this as a pre-existing gap; the test only asserts the export shape, not round-trip.
- **`verify_beliefs` doesn't persist.** It returns a dict; whoever writes `nodes.verified_at` and any verdict metadata is not in the observation set.
- **`repair_beliefs`'s output is stale relative to the DB.** It emits result dicts; it does not re-export the network, so a subsequent `export_markdown` shows a union `Depends on:` — the extra matched alternative justification is invisible on disk.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — Contains `resolve_source_path`, which `verify.py:read_source` uses to resolve relative paths against the DB directory; you need this to understand why verify works when run from a subdirectory.
- [function] `reasons/network.py:add_node` — Determines whether a just-added derived node is `IN` or `OUT`, how `supporting_justification` is chosen when multiple justifications exist, and how propagation cascades. Both ingest and repair depend on it.
- [function] `reasons/api.py:search` — The `search_fn` default for `_do_search_and_link`; the filtering (`include_out=True`, JSON format, candidate pruning in `search_and_link`) is what makes or breaks a repair.
- [file] `reasons/repair.py` — Only lines 0–400 of 618 were observed; the `abandon` and `research` branches, and the tail of `repair_beliefs` that dispatches to them, are not in the trace.
- [file] `tests/test_roundtrip.py` — `test_double_round_trip_stability` is the ground truth for "what survives a round trip"; `test_nogoods_export_format`'s docstring documents the known nogood gap.

## Beliefs

- `export-collapses-multiple-justifications-into-one-union` — `export_markdown` accumulates antecedents and outlists across all `node.justifications` and writes a single `- Depends on:` / `- Unless:` pair, while `import_into_network` reconstructs exactly one `Justification` per claim.
- `import-drops-defeated-by-lines` — `parse_beliefs` has no branch for `- Defeated by:`, so it falls into the generic `elif line.startswith("- "): pass` and the defeat relationship is lost across export→import.
- `nogoods-export-format-does-not-match-parser-regex` — `export_markdown` writes `### {ng.id}` without a colon, but `parse_nogoods` requires `^### (nogood-\S+?):`, meaning nogood round-trip requires the format fix noted in `test_nogoods_export_format`.
- `verify-beliefs-returns-without-mutating-network` — `verify_beliefs` returns a `dict[str, {verdict, reason, quote}]` and does not set `verified_at` or any metadata on the node; persistence happens in a caller outside the observed files.
- `source-url-is-persisted-as-a-first-class-column` — `nodes.source_url` is a real column (added by migration in `storage.py:_init_schema`), unlike `accepted_pr`, which lives only inside `metadata_json`; both still round-trip through markdown via dedicated emit/parse branches.
- `search-and-link-repair-adds-rather-than-replaces-a-justification` — `_do_search_and_link` calls `api.add_justification` with the original antecedents plus `matched_ids`, leaving the original justification in place.
