---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: review_premises in reasons/api.py

# `review_premises` — Premise Factual-Accuracy Review

Located in `reasons/api.py` (note: the source snippet in the prompt is truncated at the signature — the full file context is the authoritative version).

## Purpose

`review_premises` is the functional-API entry point for the **factual accuracy review** stage of the Reason Maintenance System. It takes every *premise* in the network (IN nodes with no justifications — i.e., base assertions not derived from anything else), fetches the source document each premise cites, and sends premise+source to an LLM for adjudication: does the belief text accurately reflect what the source says, and is the belief well-scoped (not over-generalized)?

The result is either reported back to the caller (dry-run / CLI) or written back as metadata on each reviewed node so subsequent passes can find un-reviewed or failing premises.

It is deliberately a thin orchestrator: filtering, source resolution, and metadata persistence live here; the LLM prompting and per-belief evaluation live in `reasons/review_premises.py:_review`.

## Contract

**Preconditions**
- `db_path` must point to a loadable SQLite network file (via `export_network` → `_with_network`).
- For any premise to actually be reviewed, it must have a non-empty `source` field and that source must resolve (through `resolve_source_path`) to an existing, readable file.
- `_review` (from `reasons.review_premises`) must be callable with `(nodes, premise_ids, source_contents, model, timeout, on_batch, parallel)`.

**Postconditions**
- If `dry_run=True`: no writes occur; the returned `results` are informational only.
- If `dry_run=False` **and** at least one result came back: each reviewed node still present in the network gets:
  - `metadata["last_premise_reviewed"] = now` (naive local ISO timestamp)
  - `metadata["premise_review_result"]` ∈ `{"pass", "overgeneralized", "<error_type>"}` where `<error_type>` comes from the LLM result (fallback `"inaccurate"`).
- Node timestamps (`reviewed_at`, `updated_at`) are **not** touched — unlike `review_beliefs`. This is a real behavioral divergence.
- No truth values are changed. This is a read/review operation, not a mutation of belief semantics.

**Invariants**
- `results[*]["id"]` values are a subset of `review_ids`.
- `reviewed == len(review_ids)`; `skipped_no_source == skipped` (see caveat below).
- Nodes with an empty `source` string never reach `_review`.
- `source_contents` is keyed by the **source string** (not node ID), so multiple premises sharing a source share a single file read — but a *failed* read is not negatively cached, so N nodes missing the same source produce N stderr "SKIP" lines and N skipped increments.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `belief_ids` | `list[str] \| None` | If given, restrict candidates to these IDs. | Unknown IDs silently dropped (no error). If `[]` (empty list), `if belief_ids:` is falsy so **no** filtering happens — you get all premises. |
| `model` | `str` | LLM model name passed to `_review` → `invoke_model`. | Default `"claude"`. Not validated here. |
| `timeout` | `int` | Per-LLM-call timeout (seconds). | Passed through verbatim. |
| `sample` | `int \| None` | Random subset size, sampled after all other filters. | If `sample >= len(candidates)`, no sampling occurs. Uses `random.sample(sorted(...))` — deterministic set given seed, but the module doesn't seed. |
| `visible_to` | `list[str] \| None` | Access-control tags. A node with `access_tags` is only reviewed if all tags ∈ `visible_to`. Nodes with no `access_tags` are always in-scope. | Note: uses inline logic identical to `_is_visible`; cannot reuse `_is_visible` because it operates on exported dicts, not `Node` objects. |
| `dry_run` | `bool` | Skip metadata writeback. | `results` still returned and `inaccurate`/`overgeneralized` counters still computed. |
| `on_batch` | `Callable \| None` | Progress callback, delegated to `_review`. | Wrapper itself never calls it. |
| `parallel` | `int` | Concurrency level for `_review`. | Default 0 — semantics defined by `_review`. Wrapper just passes it through. |
| `db_path` | `str` | SQLite path. | Default `DEFAULT_DB = "reasons.db"`. `db_dir = Path(db_path).parent if db_path else None` — an empty `db_path` becomes `None`, silently changing relative-path resolution in `resolve_source_path`. |

## Return Value

Returns a dict:

```python
{
  "results": list[dict],            # raw per-premise LLM verdicts from _review
  "reviewed": int,                  # len(review_ids)
  "inaccurate": int,                # results where accurate == False
  "overgeneralized": int,           # results where accurate == True and well_scoped == False
  "total_premises": int,            # ALL IN premises in the network (pre-filter)
  "skipped_no_source": int,         # see caveat
  "timestamp": str,                 # naive local ISO, timespec="seconds"
}
```

**Caveat on `skipped_no_source`**: despite the name, the counter is incremented for *three* distinct reasons:
1. premise has no `source` string,
2. `resolve_source_path` returned `None` or a nonexistent path,
3. source exists but `read_text()` raised `OSError`/`UnicodeDecodeError`.

Case 3 does **not** print to stderr, so it is invisible except via the counter. Callers using `skipped_no_source` to mean "missing provenance" will over-count.

**Caller obligations**
- If `dry_run=True`, the caller is responsible for any persistence.
- If `dry_run=False` but `results` is empty, *no writes* happen — the caller should not assume reviewed timestamps exist.
- `overgeneralized` and `inaccurate` are **mutually exclusive counters** in this function (a node with both flags counts only as inaccurate).

## Algorithm

1. **Import and resolve dependencies.** Locally imports `sys` (redundantly — the module already imports it), `resolve_source_path` from `.check_stale`, and `_review` from `.review_premises`.
2. **Export the whole network.** `export_network(db_path=db_path)` returns `{nodes, nogoods, repos, meta}`. Nodes are dicts, not `Node` objects.
3. **Normalize repo paths.** Convert every value in `repos` to `Path` if it's a string (for downstream `resolve_source_path`).
4. **Compute `db_dir`** as `Path(db_path).parent`.
5. **Build `all_premises`**: nodes with `truth_value == "IN"` and either no `justifications` key or an empty list. `total_premises = len(all_premises)` is captured *before* any filtering.
6. **Apply `belief_ids` filter** (only if truthy).
7. **Apply access filter** — inline version of `_is_visible`, using `set(visible_to)` hoisted out of the comprehension.
8. **Apply `sample`** — `random.sample(sorted(candidates.keys()), sample)` then rebuild dict. Sort-before-sample makes it stable when input is sorted, but the function itself doesn't seed.
9. **Source resolution loop** over `sorted(candidates.keys())`:
   - If `source` missing/empty → `skipped += 1`, `continue`.
   - If `source` not yet in `source_contents`: resolve path; if path missing → stderr SKIP, increment, continue; if present but unreadable → increment, continue; otherwise cache text.
   - Append `nid` to `review_ids`.
10. **Delegate** to `_review(nodes, premise_ids=review_ids, source_contents=source_contents, model, timeout, on_batch, parallel)`. This is where the LLM calls happen (batched, optionally parallel). Exceptions from `_review` propagate.
11. **Classify and persist** (only if `not dry_run and results`):
    - Build `result_map = {r["id"]: r for r in results}`.
    - Open a write context: `_with_network(db_path, write=True)`.
    - For every `nid` in `review_ids` that is still in `net.nodes` and in `result_map`:
      - `metadata["last_premise_reviewed"] = now`
      - If `accurate and well_scoped` → `"pass"`
      - Else if `not accurate` → `r.get("error_type", "inaccurate")`
      - Else (accurate but not well_scoped) → `"overgeneralized"`
    - Note: only metadata is touched — `reviewed_at` and `updated_at` are **not** updated.
12. **Aggregate counters** from `results` and return the dict.

## Side Effects

- **Filesystem reads**: one `read_text()` per unique source path that successfully resolves. Failed reads are retried once per referencing node (no negative cache).
- **Stderr output**: `print(f"  SKIP {nid}: source not found: {source}", file=sys.stderr)` for missing paths only.
- **Database write** (conditional): updates `metadata["last_premise_reviewed"]` and `metadata["premise_review_result"]` on each reviewed node, then `Storage.save()` and `pubsub.publish()` (via `_with_network`'s exit logic) if subscribers exist.
- **LLM network calls**: via `_review`, which uses `.llm.invoke_model` internally.
- **Cache invalidation**: writing updates `_network_cache` mtime entry for `db_path`.

## Error Handling

**Propagated exceptions**
- `export_network` failures (corrupt DB, missing file) — not caught.
- `resolve_source_path` failures — not caught.
- `_review` failures (LLM timeouts, malformed prompts, transport errors) — not caught.
- `_with_network(write=True)` failures on save — not caught.

**Swallowed / handled**
- `OSError` and `UnicodeDecodeError` on `path.read_text()` — silently counted as skipped, no log, no stderr message.
- Missing source path — stderr message + skip counter.
- Empty source string — silent skip counter.

**Not raised ever**
- This function does not raise on empty `review_ids`. If no candidates pass the source filter, `_review` is called with an empty list and, assuming `_review` handles it, `results` will be `[]`, triggering the `if not dry_run and results` guard so no metadata is written.

## Usage Patterns

- **CLI (`reasons review-premises`)**: called with `--sample`, `--model`, `--dry-run`, and access tags. `dry_run=True` for spot-checks; writeback for batch audits.
- **`repair_premises` (same file)**: invokes `review_premises(belief_ids=..., dry_run=True, model=..., timeout=..., db_path=db_path)` to obtain a `results` list, filters to `accurate=False`, and drives supersession/retraction. This is the canonical re-use pattern: `dry_run=True` makes the function a pure "review query."
- **Scheduled audits**: call with `sample=N` to spot-check a random sample in CI and fail if `inaccurate > 0`.
- **Post-import verification**: call after `import_beliefs` to validate imported premises against their claimed sources.

**Notable caller obligations**
- `dry_run=True` means the caller must not assume `last_premise_reviewed` will exist afterwards.
- A node in `review_ids` may not be in `net.nodes` when the write context reopens — if the DB was concurrently mutated, that node is silently dropped from writeback but still counted in `reviewed`.

## Dependencies

- `.export_network` (same file) — full network dict export.
- `.check_stale.resolve_source_path` — resolves `"repo:path"` source strings to filesystem `Path`.
- `.review_premises.review_premises as _review` — LLM orchestration (prompt building, batching, parallel dispatch).
- `.storage.Storage` / `_with_network` — the load/save context manager.
- `datetime`, `pathlib.Path`, `random`, `sys` — stdlib.

## Assumptions Not Enforced by the Type System

1. **`v["justifications"]` is indexable after the truthiness check.** If a node dict has `justifications={}` (a dict), `len({}) == 0` so the node is treated as a premise. The type system sees `dict` here, not `list | None`.
2. **`v["metadata"]["access_tags"]` is always a list of strings** when present. A malicious DB with `"access_tags": "public"` would iterate characters.
3. **`repos` values are either `str` or `Path`.** A `Path` passes through; a non-Path non-str would reach `resolve_source_path` and fail at `.exists()`.
4. **`_review` returns per-ID dicts containing `id`.** If `_review` returned entries missing `"id"`, the `result_map` comprehension raises `KeyError`.
5. **`datetime.now()` produces a useful timestamp**. The code uses naive local time, unlike `review_beliefs`' `datetime.now(timezone.utc)`. This means `last_premise_reviewed` and `review_result`-adjacent timestamps from different commands can be in different time bases.
6. **Source file encoding is UTF-8.** `read_text()` defaults to locale encoding; a UTF-16 file would be silently skipped as `UnicodeDecodeError`, inflating `skipped_no_source`.

## Topics to Explore

- [file] `reasons/review_premises.py` — The actual LLM worker `_review` invoked by this function; how prompts are constructed and `parallel`/`on_batch` are honored.
- [function] `reasons/check_stale.py:resolve_source_path` — How `"repo:path"` source strings become filesystem paths, including `agent` scoping and `db_dir` relative resolution.
- [function] `reasons/api.py:repair_premises` — The canonical downstream consumer; shows how `dry_run=True` review results feed supersede/retract repair.
- [function] `reasons/api.py:review_beliefs` — Sibling reviewer for derived beliefs; contrasts with this function on which timestamps and metadata keys are written.
- [general] `review-metadata-schema` — The collection of `*_review_result` / `last_*_reviewed` / `reviewed_at` keys scattered across `review_beliefs`, `review_justifications`, and `review_premises` — worth understanding whether there's a unified schema or per-reviewer ad-hoc keys.

## Beliefs

- `review-premises-filters-to-in-premises-without-justifications` — Candidates are exactly IN nodes whose `justifications` field is absent or empty, before `belief_ids`/`visible_to`/`sample` narrowing.
- `review-premises-skipped-no-source-counts-all-source-failures` — The `skipped_no_source` counter is incremented for missing source, unresolvable path, and unreadable file, not only for "no source" premises.
- `review-premises-writes-only-metadata-not-reviewed-at` — Writeback updates only `metadata["last_premise_reviewed"]` and `metadata["premise_review_result"]`; unlike `review_beliefs`, it does not touch `reviewed_at` or `updated_at` node timestamps.
- `review-premises-caches-source-reads-by-source-string` — `source_contents` is keyed by the source string so multiple premises sharing the same `source` share a single `read_text()`, but a failed read is not negatively cached.
- `review-premises-reuses-dry-run-mode-for-repair-premises` — `repair_premises` calls `review_premises(..., dry_run=True)` to obtain verdicts without writing back metadata, making dry-run the canonical "query" mode for downstream repair.
