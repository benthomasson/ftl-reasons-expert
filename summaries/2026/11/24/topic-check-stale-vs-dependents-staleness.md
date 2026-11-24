---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Keep source-file staleness separate from reverse-index stale/dangling dependents.

# Keeping source-file staleness separate from reverse-index stale/dangling dependents

The codebase has two unrelated ideas that both use the word "stale" (and both get exercised by tests in `tests/test_check_stale*.py` and `tests/test_dependents_integrity.py` / `tests/test_dangling_dependents.py`). Treating them as one concern is the trap this title is guarding against.

## 1. Source-file staleness — `reasons/check_stale.py:62`

This is about **an external file diverging from the hash that was captured when the node was created.** Nothing in this module touches `node.dependents` or the reverse index.

- `hash_file` (`reasons/check_stale.py:15`) is the primitive: full SHA-256 of the file content.
- `resolve_source_path` (`reasons/check_stale.py:20`) maps `repo-name/path/to/file.md` → an absolute `Path`, using agent repo first, then `db_dir`, then `repos`/`~/git`.
- `check_stale` (`reasons/check_stale.py:62`) iterates IN nodes with a `source` and `source_hash` and emits a list of dicts shaped as `{node_id, old_hash, new_hash, source, source_path, reason}`. The `reason` is one of:
  - `content_changed` — hash differs and it isn't a truncated-hash prefix.
  - `truncated_hash` — stored hash is a 16-char prefix of the current full hash and `upgrade_hashes=False` (`check_stale.py:~118-135`).
  - `source_deleted` — `resolve_source_path` returned `None` (`check_stale.py:~100-110`).
- It returns a tuple `(results, upgraded, sha_bumped)`, and when `git_aware=True` it uses `pinned_sha` + `file_changed_since` / `file_lines_changed_since` to skip work when only untouched lines moved (`check_stale.py:215`, `check_stale.py:256`).
- The remediation API is `hash_sources` (`check_stale.py:154`), `pin_sources` (`check_stale.py:299`), `pin_update` (`check_stale.py:352`) — all mutate `source_hash` / `pinned_sha` / `verified_at`; none touch `dependents`.

Contract enforced by tests:
- Skips OUT nodes and nodes missing `source` or `source_hash` (`tests/test_check_stale.py::TestCheckStale::test_skips_out_nodes`, `test_skips_nodes_without_hash`).
- `source_deleted` results return `new_hash=None`, `source_path=None`, and the same key set as `content_changed` (`tests/test_check_stale_issue25.py::TestSourceDeletedResult::test_content_changed_has_same_keys_as_source_deleted`).

## 2. Reverse-index stale/dangling dependents — `Node.dependents`

This is **internal graph-consistency**, independent of any file on disk.

- The field is declared once: `dependents: set[str] = field(default_factory=set)  # reverse index` (`reasons/__init__.py:38`). Set semantics means duplicates collapse and iteration order is unpinned.
- It's written wherever a justification is created, e.g. `net.nodes[ant_id].dependents.add(nid)` and `net.nodes[out_id].dependents.add(nid)` (`reasons/api.py:2579-2582`, `2650-2653`), and in defeater paths (`reasons/api.py:1394, 1528, 1630`).
- It's read by traversal/rendering code: `for dep_id in current.dependents:` (`reasons/api.py:745`), `for dep_id in node.dependents:` (`reasons/api.py:2963`, `3137`), and export enrichment (`reasons/api.py:4510-4514`). `_rewrite_dependents` (`reasons/api.py:4925`) rewrites both sides on ID quarantine/rename, iterating `for dep_id in list(old_node.dependents):` (`reasons/api.py:4933`).

Two distinct kinds of failure:

- **Dangling**: a `dependents` element references an ID that isn't in `network.nodes`. `_propagate` must skip these with a `warn` log entry instead of raising `KeyError` (issue #22). Covered by `tests/test_dangling_dependents.py` — e.g. `test_retract_with_single_dangling_dependent`, `test_valid_dependent_still_retracted`, `test_dangling_does_not_enter_visited`.
- **Stale / out-of-sync**: the set disagrees with the actual justifications. Two primitives handle this — `verify_dependents()` is read-only and returns a list of error strings; `_rebuild_dependents()` clears all sets and reconstructs from justifications. Both are exercised in `tests/test_dependents_integrity.py` (`TestRebuildDependents::test_rebuild_clears_stale_entries`, `TestVerifyDependents::test_detects_extra_dependent`, `test_does_not_modify_state`, `TestCorruptionAndRepair`).

## 3. Why the boundary matters

| | Source-file staleness | Dependents staleness |
|---|---|---|
| Module | `reasons/check_stale.py` | `reasons/__init__.py:38` + `reasons/network.py` (`verify_dependents`/`_rebuild_dependents`) |
| Trigger condition | `node.source_hash` ≠ current file hash, or file missing | `dependents` set disagrees with justifications, or references a missing node |
| Data touched | `node.source_hash`, `node.metadata["pinned_sha"|"verified_at"]` | `node.dependents` only |
| Failure mode | Wrong/outdated belief body — a *correctness* problem | `KeyError` during propagation (`_propagate`) or silently wrong UI counts — an *integrity* problem |
| Remediation | `hash_sources`, `pin_sources`, `pin_update`, re-import | `_rebuild_dependents()` for corruption; skip-with-warning for dangling |
| Test file | `tests/test_check_stale.py`, `tests/test_check_stale_issue25.py` | `tests/test_dependents_integrity.py`, `tests/test_dangling_dependents.py` |

Concretely: `check_stale` iterates nodes (`for nid, node in sorted(network.nodes.items())`) but only reads `node.source`, `node.source_hash`, `node.metadata` — it never iterates `node.dependents`. Conversely, `verify_dependents` / `_rebuild_dependents` don't touch `.source` or `.source_hash`. The two concerns do not currently cross.

## 4. Where the naming collides (and what to watch for)

- There are two `check_stale` symbols: `reasons/check_stale.py:62` and `reasons/api.py:2680`. Based on the surrounding `reasons/api.py` context (dependents work at `2650-2653`), `api.py:2680` is very likely an orchestration/CLI-facing wrapper — but the observation set doesn't include its body, so any statement that it *also* checks dependents is unverified. Flag: if someone adds dependents-integrity checking, the natural place is `verify_dependents`, **not** `check_stale`.
- The 377 `dependents` matches in `reasons/api.py` show that the reverse index is pervasive; adding source-staleness awareness to any of those call sites would couple two concerns that today are cleanly orthogonal.
- The phrase "stale" in `test_rebuild_clears_stale_entries` (`tests/test_dependents_integrity.py`) means *stale entries in the dependents set* — not the file-derived sense of `test_stale_node` in `tests/test_check_stale.py`. Same word, unrelated concept.

**Observation gaps:** the body of `reasons/api.py:2680 check_stale` is missing; `_propagate`, `verify_dependents`, and `_rebuild_dependents` implementations are only inferred from tests, not shown; the cut-off at `reasons/check_stale.py:400` hides the tail of `pin_update`.

## Topics to Explore

- [file] `reasons/check_stale.py` — Full source-staleness surface: `hash_file`, `resolve_source_path`, git-aware pinning, and the three `reason` values.
- [function] `reasons/check_stale.py:check_stale` — Entry point; note which inputs it reads and confirm it never touches `node.dependents`.
- [function] `reasons/api.py:check_stale` — Wrapper at line 2680; verify whether it's purely source-level or also calls `verify_dependents()`.
- [general] `dependents-verify-vs-rebuild` — Compare the read-only `verify_dependents()` contract with the mutating `_rebuild_dependents()` to understand how corruption is detected vs. repaired.
- [general] `dangling-propagate-guard` — How `_propagate` distinguishes a dangling dependents entry (warn + skip) from a valid one, per `tests/test_dangling_dependents.py`.

## Beliefs

- `source-staleness-requires-source-hash` — `check_stale` skips any node whose `truth_value != "IN"` or whose `source`/`source_hash` is empty (`reasons/check_stale.py:62`).
- `source-staleness-produces-enumerated-reasons` — Every result from `check_stale` carries `reason` ∈ {`content_changed`, `truncated_hash`, `source_deleted`}, with `new_hash=None` only for `source_deleted`.
- `dependents-is-a-set` — `Node.dependents` is a `set[str]` (`reasons/__init__.py:38`), so duplicate justification edges collapse and iteration order is unstable.
- `verify-dependents-is-read-only` — `verify_dependents()` reports inconsistencies without modifying state (`tests/test_dependents_integrity.py::TestVerifyDependents::test_does_not_modify_state`).
- `rebuild-dependents-repairs-from-justifications` — `_rebuild_dependents()` clears and reconstructs the reverse index, and is idempotent (`tests/test_dependents_integrity.py::TestCorruptionAndRepair::test_rebuild_idempotent`).
- `dangling-guard-prevents-keyerror-in-propagate` — Propagation must skip dependent IDs absent from `network.nodes` and emit a `warn` log entry (issue #22, `tests/test_dangling_dependents.py`).
- `source-staleness-does-not-mutate-dependents` — `check_stale` writes only to `source_hash` and `metadata` (`pinned_sha`, `verified_at`); it never touches `node.dependents`.
