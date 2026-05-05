# File: tests/test_check_stale_issue25.py

**Date:** 2026-05-05
**Time:** 15:28

I'll work from the test file content provided. Here's the explanation:

---

## Purpose

`test_check_stale_issue25.py` is a regression test suite for [issue #25](https://github.com/benthomasson/ftl-reasons/issues/25). The bug: `check_stale()` silently skipped beliefs whose source files had been deleted from disk, producing no output at all — meaning deleted sources went undetected. The fix introduced a new `source_deleted` result type, and this file validates every aspect of that behavior.

It does **not** test the general `content_changed` staleness path exhaustively — `test_check_stale.py` covers that. This file focuses specifically on the deleted-source dimension and its interaction with the existing stale-check machinery.

## Key Components

### Helper

- **`_hash(content: str) -> str`** — Convenience wrapper around `hashlib.sha256` to produce the same hashes that `check_stale` / `hash_file` compute internally. Used to set up "fresh" nodes whose hash matches their file content on disk.

### Test Classes

| Class | Responsibility |
|---|---|
| `TestSourceDeletedResult` | Core contract: a missing source file produces a result dict with `reason == "source_deleted"`, `new_hash == None`, `source_path == None`, and the correct schema. |
| `TestMixedResults` | Integration: when both deleted and changed sources exist in the same network, both kinds of results appear and fresh nodes are excluded. Also covers multiple nodes pointing at the same missing file. |
| `TestSkipBehavior` | Guard rails: nodes that should **not** produce `source_deleted` — OUT (retracted) nodes, nodes with no source, nodes with no hash, nodes with an empty source string. |
| `TestEdgeCases` | Boundary conditions from code review: empty files that exist are not "deleted"; `repos=None` still detects missing files; results are sorted by `node_id`; `content_changed` populates `source_path`. |

## Patterns

**Repo-mapping convention.** `check_stale` resolves source references like `"r/gone.md"` by splitting on the first `/` to get a repo key (`r`), then looking that key up in the `repos` dict to get a base directory. Tests use `tmp_path` as the base for repo key `"r"`. This is the same pattern the production `check_stale` / `export-markdown` workflow uses for multi-repo support.

**Write-then-mutate.** Tests that need `content_changed` write a file, record its hash, then overwrite the file before calling `check_stale`. This simulates upstream code changes between belief creation and staleness checks.

**Schema contract tests.** `test_source_deleted_has_all_required_keys` and `test_content_changed_has_same_keys_as_source_deleted` enforce that every result dict has the same six keys regardless of reason. This prevents consumers from needing to branch on reason before accessing fields — a defensive API contract.

**Negative testing via absence.** Several tests assert `results == []` to confirm that certain node configurations are intentionally excluded from stale-check results.

## Dependencies

**Imports:**
- `reasons_lib.network.Network` — The in-memory belief graph. `add_node()` registers beliefs with optional `source` and `source_hash` metadata. `retract()` marks a node OUT.
- `reasons_lib.check_stale.check_stale` — The function under test. Takes a `Network` and an optional `repos` mapping, returns `(results, ...)` where results is a list of stale-check dicts.
- `reasons_lib.check_stale.hash_file` — Imported but not directly called in any test (likely available for future tests or pulled in by convention).
- `pytest` — Imported for the test framework; `tmp_path` fixture provides isolated temp directories.

**Imported by:** Nothing — this is a leaf test module.

## Flow

Each test follows the same pattern:

1. **Arrange** — Create a `Network`, add nodes with `source` and `source_hash` metadata, optionally create/modify files in `tmp_path`.
2. **Act** — Call `check_stale(net, repos={"r": tmp_path})`, destructure into `(results, _)`.
3. **Assert** — Inspect `results` for expected count, `reason` values, field contents, key sets, or sort order.

The `check_stale` function internally iterates all IN nodes with source metadata, resolves the file path via the repos mapping, and compares the on-disk hash against `source_hash`. If the file is missing, it emits `source_deleted`. If the file exists but the hash differs, it emits `content_changed`. If the hash matches, the node is fresh and excluded from results.

## Invariants

1. **Result schema is uniform.** Every result dict has exactly `{node_id, old_hash, new_hash, source, source_path, reason}` regardless of whether the reason is `source_deleted` or `content_changed`.
2. **`source_deleted` results have `new_hash=None` and `source_path=None`.** There's no file to hash or point to.
3. **OUT nodes are never reported.** Retracted beliefs are excluded even if their source is missing.
4. **Nodes without both `source` and `source_hash` are skipped.** Either field missing (or `source` being empty string) means the node is not eligible for staleness checking.
5. **Results are sorted by `node_id`.** Alphabetical ordering for deterministic output.
6. **An empty file that exists is not deleted.** A zero-byte file with the correct hash is fresh; with the wrong hash it's `content_changed`, not `source_deleted`.

## Error Handling

The tests validate that `check_stale` does **not** raise on missing files — it produces structured result dicts instead. The `repos=None` edge case (`test_no_repos_mapping_with_missing_file`) confirms that even without a repos mapping, the function gracefully degrades to `source_deleted` rather than crashing with a `KeyError` or `TypeError`.

No exceptions are expected or caught in any test — the function's contract is to always return results, never raise.

---

## Topics to Explore

- [file] `reasons_lib/check_stale.py` — The implementation under test; see how repo-key resolution, hash comparison, and the `source_deleted` branch actually work
- [file] `tests/test_check_stale.py` — The original stale-check test suite covering `content_changed` and the general staleness workflow, before issue #25
- [function] `reasons_lib/network.py:Network.add_node` — Understand the `source` and `source_hash` parameters and how they're stored in the node graph
- [general] `issue-25-pr-history` — Look at the PR that closed issue #25 to see what `check_stale` looked like before the `source_deleted` path was added
- [file] `reasons_lib/export_markdown.py` — Uses the same repo-mapping pattern to resolve source paths; shows how `check_stale` results feed into the broader workflow

## Beliefs

- `check-stale-result-schema-uniform` — Every `check_stale` result dict contains exactly six keys (`node_id`, `old_hash`, `new_hash`, `source`, `source_path`, `reason`) regardless of reason type
- `check-stale-skips-out-nodes` — `check_stale` excludes retracted (OUT) nodes from results even if their source files are missing
- `check-stale-requires-source-and-hash` — Nodes missing either `source` or `source_hash` (or with empty `source`) are excluded from staleness checking
- `check-stale-results-sorted-by-node-id` — The results list returned by `check_stale` is sorted alphabetically by `node_id`
- `source-deleted-nulls-hash-and-path` — A `source_deleted` result always has `new_hash=None` and `source_path=None`

