---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_check_stale_issue25.py

# `tests/test_check_stale_issue25.py`

## Purpose

This is a **regression test file** pinned to a specific GitHub issue (#25). The bug it guards against: `check_stale()` was *silently skipping* belief nodes whose source files no longer exist on disk. After the fix, those nodes must produce an explicit `source_deleted` result so callers can react (repair, alert, etc.) rather than losing track of orphaned beliefs.

The file's sole responsibility is to lock down the **contract** of the `source_deleted` code path in `check_stale()` — its output shape, its coexistence with `content_changed` results, and its interaction with the skip rules (retacted nodes, missing metadata).

## Key Components

| Component | Role |
|---|---|
| `_hash(content)` | Local helper. SHA-256 hex digest of a string. Used to construct expected `source_hash` values without reading a file. Note: the module also imports `hash_file` from `reasons.check_stale` but the tests use this local helper instead, keeping the hash logic under test's own control. |
| `TestSourceDeletedResult` | Core contract tests. Verifies that a missing file yields exactly one result with `reason == "source_deleted"`, `new_hash is None`, `source_path is None`, and that the result dict has **exactly** the six keys `{node_id, old_hash, new_hash, source, source_path, reason}`. Also asserts that `content_changed` results carry the *same* key set. |
| `TestMixedResults` | Ensures a single `check_stale()` call can return both `content_changed` and `source_deleted` entries in the same list, and that multiple nodes sharing one deleted file each get their own entry. |
| `TestSkipBehavior` | Guards the *negative* cases: retacted (`net.retract`) nodes, nodes with no `source`, nodes with no `source_hash`, and nodes with an empty-string `source` must all produce **zero** results even if their (hypothetical) file is missing. |
| `TestEdgeCases` | Covers reviewer-flagged corners: an existing-but-empty file is *not* deleted (hash match → no result); an empty file with a wrong hash is `content_changed` (not `source_deleted`); `repos=None` still yields `source_deleted`; results are sorted by `node_id`; `content_changed` populates `source_path` with the resolved filesystem path. |

### Result dict contract (the central invariant)

Every entry in the returned list is a dict with **exactly** these six keys:

```
{
    "node_id":      str,
    "old_hash":     str,
    "new_hash":     str | None,   # None for source_deleted
    "source":       str,          # e.g. "r/gone.md"
    "source_path":  str | None,   # None for source_deleted; resolved path for content_changed
    "reason":       "source_deleted" | "content_changed"
}
```

## Patterns

- **Pytest class-as-namespace.** No inheritance, no `setup_method`. Each class is a thematic grouping; each method is self-contained. This is the dominant idiom in the repo's `tests/` directory (see `test_check_stale.py`, `test_contradictions.py`, etc.).
- **`tmp_path` fixture isolation.** Every test that touches the filesystem gets a fresh temporary directory via pytest's built-in `tmp_path`. No shared state between tests.
- **Fresh `Network` per test.** Each test constructs its own `Network()` and adds 1–3 nodes. No fixtures in `conftest.py` provide a pre-built graph for these tests — the graph is trivial enough to inline.
- **`results, *_ = check_stale(...)` unpacking.** The function returns a tuple whose first element is the list of stale results; the rest (likely a summary dict or count) is discarded with `*_`. The tests only care about the first element.
- **Exact-set key assertion.** `assert set(results[0].keys()) == required_keys` is a stronger check than `in` — it catches both missing *and* extra keys. This is intentional: the dict is a serialized contract consumed by downstream tooling (CLI output, API responses).
- **Hash-as-oracle.** The local `_hash()` computes the *expected* hash independently of `hash_file`, so a bug in `hash_file` would surface as a mismatch rather than a false pass.

## Dependencies

**This file imports:**

- `reasons.network.Network` — the belief-graph data structure. Tests call `net.add_node(id, label, source=…, source_hash=…)` and `net.retract(id)`.
- `reasons.check_stale.check_stale` — the function under test. Signature: `check_stale(net, repos=None) -> tuple[list[dict], …]`.
- `reasons.check_stale.hash_file` — imported but **not called** in the test body (the local `_hash` is used instead). Its presence in the import line suggests it was either used in an earlier draft or is kept for potential use in future assertions.
- `hashlib`, `pathlib.Path`, `pytest` — standard test infrastructure.

**What depends on this file:**

Nothing imports it. It is a leaf test module. Its value is in being *run* by `pytest tests/` (or `pytest tests/test_check_stale_issue25.py` in isolation). It sits alongside `test_check_stale.py`, which covers the broader `content_changed` behavior; this file is the issue-specific regression layer.

## Flow

Each test follows the same five-step pattern:

1. **Build the graph.** `net = Network()`, then `net.add_node(id, label, source="r/<file>.md", source_hash=<hash>)`. The `source` string uses an `org/path` format where `org` is a key into the `repos` mapping.
2. **Prepare the filesystem** (if applicable). Write a file into `tmp_path`, or intentionally *don't* create it (to simulate deletion). Mutate a file's content to simulate a change.
3. **Invoke.** `results, *_ = check_stale(net, repos={"r": tmp_path})`. Inside `check_stale`, for each non-retacted node that has both `source` and `source_hash`, it resolves `repos[org] / path`. If the file doesn't exist → `source_deleted`. If it exists but `hash_file(path) != source_hash` → `content_changed`. Otherwise → skip (fresh).
4. **Assert shape and content.** Check list length, specific `reason` values, exact key sets, `None`-ness of `new_hash`/`source_path`, and sorting order.
5. **Assert negatives.** For skip-behavior tests, assert `results == []`.

There is no state mutation *across* tests — each test is a closed universe.

## Invariants

The tests collectively enforce these rules on `check_stale()`:

1. **Exhaustive reporting.** A node with a valid `source` + `source_hash` whose file is missing *must* appear in the results. Silent omission is the bug this file exists to prevent.
2. **Key-set stability.** Both `source_deleted` and `content_changed` results carry the identical six-key set. This means downstream consumers (CLI, API, MCP server) can process results polymorphically without branching on key presence.
3. **`source_deleted` nulls.** For `source_deleted`, `new_hash` is `None` and `source_path` is `None` — there is no file to hash or point to.
4. **`content_changed` populates `source_path`.** It must be the resolved absolute path (e.g., `str(tmp_path / "src.md")`), not the relative `r/src.md`.
5. **Skip rules are orthogonal to deletion.** Retraction, missing `source`, missing `source_hash`, and empty-string `source` all suppress results *even if the file is also missing*. The skip check precedes the existence check.
6. **Multiple nodes, one file.** Two nodes pointing at the same `r/shared.md` each get their own `source_deleted` entry. The function operates per-node, not per-file.
7. **Deterministic ordering.** Results are sorted by `node_id` ascending (verified by `test_results_sorted_by_node_id`).
8. **Empty file ≠ deleted.** A zero-byte file that exists on disk is *not* `source_deleted`; it's either fresh (hash matches) or `content_changed` (hash differs).

## Error Handling

This is a test file, so "error handling" means **asserting the absence of silent failures**:

- The original bug (issue #25) was a *swallowed* case: `check_stale` would simply not include the node in its output. The tests here assert the opposite — the node *must* be present with `reason == "source_deleted"`.
- `test_no_repos_mapping_with_missing_file` explicitly passes `repos=None` and still expects a `source_deleted` result, ensuring the code path doesn't crash or skip when the repo mapping is entirely absent.
- No tests in this file use `pytest.raises` or `try/except`. The expectation is that `check_stale` never raises for these conditions; it always returns a result list.
- The `*_` in `results, *_ = check_stale(...)` means if `check_stale` ever changes its return signature to return fewer elements, these tests would raise a `ValueError` (unpacking error) — a useful early signal.

## Topics to Explore

- [file] `reasons/check_stale.py` — The implementation under test. Read `check_stale()` to see how it iterates nodes, resolves `repos`, distinguishes missing vs. changed files, and constructs the result dicts. Compare with `test_check_stale.py` for the broader contract.
- [file] `reasons/network.py` — `Network.add_node()` and `Network.retract()` define the node metadata (`source`, `source_hash`, in/out status) that `check_stale` reads. Understanding the node schema is prerequisite to reading the tests.
- [file] `tests/test_check_stale.py` — The sibling test file covering the general `content_changed` path and likely the original (pre-issue-25) behavior. Reading both files side by side shows how the regression test layers on top of the baseline.
- [function] `reasons/check_stale.py:hash_file` — Imported in this test file but unused. Understanding its relationship to the local `_hash()` helper clarifies whether the test is deliberately decoupled from the production hash implementation.
- [general] `repos-mapping-convention` — The `repos` dict maps org-prefixed strings (`"r"`) to `Path` objects. Explore how other modules (`import_beliefs`, `export_markdown`, `cli`) use the same convention to understand the multi-repo layout.

## Beliefs

- `source-deleted-result-has-exactly-six-keys` — Every result dict from `check_stale()` contains the set `{node_id, old_hash, new_hash, source, source_path, reason}` and no others, regardless of whether the reason is `source_deleted` or `content_changed`.
- `source-deleted-nulls-both-new-hash-and-source-path` — A `source_deleted` entry always has `new_hash is None` and `source_path is None`; a `content_changed` entry always has both populated.
- `retraction-suppresses-stale-reporting` — A node that has been retracted (`net.retract(id)`) is excluded from `check_stale` results even if its source file is also missing; the skip check precedes the file-existence check.
- `check-stale-results-sorted-by-node-id` — The result list returned by `check_stale()` is deterministically sorted in ascending `node_id` order, enabling stable diffing and display.
- `issue25-test-is-regression-guard-for-silent-skip` — `test_check_stale_issue25.py` exists specifically to prevent the re-introduction of the bug where `check_stale()` silently omits nodes with missing source files; removing this file removes that guarantee.
