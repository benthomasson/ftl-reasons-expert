---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_check_stale.py

# `tests/test_check_stale.py`

## 1. Purpose

This file is the integration test suite for the **stale-source detection** subsystem in the `reasons` package. The `check_stale` module answers a critical question in a knowledge-network (FTL) system: *"Have any of the source files that justify a belief been modified or deleted since we recorded them?"*

The tests own three responsibilities:

1. **Hashing correctness** — verifying `hash_file` produces deterministic SHA-256 digests.
2. **Path resolution** — verifying `resolve_source_path` finds the right file across multiple resolution strategies (db_dir, repos, agent repos).
3. **Staleness detection & hash backfilling** — verifying the main `check_stale` and `hash_sources` functions correctly classify nodes as fresh, stale, deleted, or missing-hash.

## 2. Key Components

### `hash_file(path) → str`
Returns the hex-encoded SHA-256 of a file's bytes. The tests confirm:
- It matches `hashlib.sha256(b"hello world").hexdigest()` exactly.
- Different content produces different hashes.

### `resolve_source_path(source, repos=None, db_dir=None, agent=None) → Path | None`
Resolves a logical source path (e.g. `"entries/topic.md"`) to an absolute `Path`. Resolution priority (as revealed by the tests):

| Priority | Strategy | Example |
|----------|----------|---------|
| 1 | `db_dir / source` | `tmp/expert/entries/topic.md` |
| 2 | `repos[agent] / source` (agent repo) | `agent_dir/entries/topic.md` |
| 3 | `repos[<prefix>] / <rest>` (split repo) | `repos["myrepo"] / "entry.md"` |
| 4 | Fallback to `repos` keys matching the first path segment | `repos["myrepo"] / "file.md"` |

Returns `None` when the file doesn't exist on disk or the source string is empty.

### `check_stale(net, repos=None, db_dir=None, upgrade_hashes=False) → (results, upgraded, ...)`
The core function. For each **active** node (not retracted) that has a non-empty `source_hash`:

- Resolves the source path.
- If the file is **missing** → reports `{"reason": "source_deleted", "new_hash": None, ...}`.
- If the file's current hash **matches** → fresh, no report.
- If the current hash **differs** → reports `{"reason": "content_changed", "old_hash": ..., "new_hash": ...}`.
- If the stored hash is a **prefix** of the current full hash (truncated legacy hash):
  - With `upgrade_hashes=True`: silently upgrades the node's hash, counts it in `upgraded`, does **not** report stale.
  - Without the flag: reports `{"reason": "truncated_hash"}`.

Returns a tuple: `(results: list[dict], upgraded: int, ...)` — the `*_` in tests swallows additional return values.

### `hash_sources(net, repos=None, db_dir=None, force=False) → list[dict]`
Backfills `source_hash` on nodes that currently have an empty string. For each eligible node it:
- Resolves the path.
- Hashes the file.
- Sets `node.source_hash`.
- Reports `{"node_id": ..., "was_empty": True/False}`.

`force=True` re-hashes even nodes that already have a hash (and sets `was_empty: False`).

### `Network` (from `reasons.network`)
The in-memory knowledge graph. Test-relevant API:
- `net.add_node(id, label, source=..., source_hash=..., metadata=...)`
- `net.retract(id)` — marks a node as "out" (inactive).
- `net.nodes[id].source_hash` — attribute access to the stored hash.
- `net.repos` — dict mapping repo names to directory paths.

## 3. Patterns

- **`tmp_path` isolation**: Every test that touches the filesystem uses pytest's `tmp_path` fixture, guaranteeing no cross-test contamination and no pollution of the real repo.
- **Inline fixture construction**: `Network()` is constructed fresh in each test (no shared `conftest` fixture), keeping tests self-contained.
- **Hash oracle pattern**: Tests compute the expected hash independently via `hashlib.sha256(...).hexdigest()` rather than trusting the module under test, giving a true independent verification.
- **Multi-return destructuring**: `results, *_ = check_stale(...)` — the `*_` idiom signals that the function's contract returns more than two values but the test only cares about the first two.
- **Precedence testing via competing files**: `test_db_dir_takes_precedence` and `test_agent_repo_takes_precedence_over_split` create the *same* relative path under two different roots with different content, then assert which one wins. This is the standard "shadowing" test pattern.
- **Stateful mutation**: `test_prefix_hash_treated_as_fresh` asserts that `net.nodes["a"].source_hash` is *mutated* to the full hash after `upgrade_hashes=True`, confirming the function has a side-effect contract.

## 4. Dependencies

**This file imports:**
- `reasons.network.Network` — the graph model.
- `reasons.check_stale.{check_stale, hash_file, hash_sources, resolve_source_path}` — the module under test.
- `hashlib`, `pathlib.Path`, `pytest` — stdlib + test framework.

**What depends on this file:**
- The CI pipeline (`.github/workflows/`) runs these tests.
- The `test_check_stale_issue25.py` file is a companion regression test, suggesting this file is the primary contract.
- No production code imports the test module; it's a leaf in the dependency graph.

## 5. Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  Each test method (e.g. test_stale_node)                        │
├─────────────────────────────────────────────────────────────────┤
│ 1. Create temp files with known content (tmp_path / "x.md")     │
│ 2. Compute expected hash via hashlib.sha256                     │
│ 3. Build Network, add_node(source=..., source_hash=...)        │
│ 4. (Optionally) mutate file content to simulate staleness       │
│ 5. Call check_stale(net, repos=..., db_dir=..., upgrade_hashes=)│
│ 6. Destructure: results, *_                                     │
│ 7. Assert on results list (length, keys, values)                │
│ 8. (For upgrade tests) assert net.nodes["a"].source_hash        │
└─────────────────────────────────────────────────────────────────┘
```

The `check_stale` internal flow (inferred from tests):

```
for node in net.active_nodes():
    if node.source_hash is empty → skip
    resolved = resolve_source_path(node.source, ...)
    if resolved is None → report "source_deleted"
    current = hash_file(resolved)
    if node.source_hash == current → fresh
    elif current.startswith(node.source_hash) → prefix case
    else → report "content_changed"
```

## 6. Invariants

| Invariant | Enforced by test |
|-----------|-----------------|
| Retacted nodes are never reported stale | `test_skips_out_nodes` |
| Nodes with empty `source_hash` are invisible to `check_stale` | `test_skips_nodes_without_hash` |
| `db_dir` resolution wins over `repos` | `test_db_dir_takes_precedence` |
| Agent-repo resolution wins over split-repo | `test_agent_repo_takes_precedence_over_split` |
| A truncated (prefix) hash is NOT the same as a genuine content change when `upgrade_hashes=True` | `test_prefix_hash_treated_as_fresh` vs `test_genuine_change_still_detected` |
| `hash_sources` never overwrites a non-empty hash unless `force=True` | `test_skips_existing_hash` / `test_force_rehashes` |
| `resolve_source_path("")` always returns `None` | `test_resolve_empty_source` |

## 7. Error Handling

The module under test is **exception-free** by design — it reports problems as structured dicts in the `results` list rather than raising:

| Condition | `reason` value | `new_hash` | `source_path` |
|-----------|---------------|------------|---------------|
| File modified | `"content_changed"` | actual new hash | resolved `Path` |
| File deleted | `"source_deleted"` | `None` | `None` |
| Truncated hash (no upgrade) | `"truncated_hash"` | actual full hash | resolved `Path` |

`resolve_source_path` signals "not found" by returning `None` (no exception). `hash_file` is only called after successful resolution, so it never encounters a missing file in the `check_stale` flow.

The tests confirm these are the **only** failure modes — there is no `try/except` in the test file, meaning the production code is expected to handle all I/O edge cases internally and surface them as result entries.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — The implementation under test; read the actual resolution-order logic and the `upgrade_hashes` code path to understand the prefix-matching heuristic.
- [file] `reasons/network.py:Network.add_node` — Understand the node schema (`source`, `source_hash`, `metadata`, `retracted`) and how `active_nodes()` filters, since the tests rely on `retract` as the sole mechanism for exclusion.
- [file] `tests/test_check_stale_issue25.py` — A regression test for a specific bug; comparing it against this file reveals edge cases the original suite missed (likely around path normalization or concurrent mutation).
- [file] `reasons/pg.py` — The PostgreSQL persistence layer; understanding how `source_hash` is stored/retrieved clarifies why truncated hashes might exist in legacy data (the `upgrade_hashes` flag exists for migration).
- [general] `hash-upgrade-migration` — The `upgrade_hashes` flag and `test_prefix_hash_treated_as_fresh` suggest a one-time data migration from truncated to full SHA-256. Search the CHANGELOG or PRs for the migration script and the reason hashes were originally truncated.

---

## Beliefs

- `check-stale-skips-retracted-nodes` — `check_stale` only evaluates nodes that have not been retracted; retracted nodes are invisible to staleness reporting regardless of their source or hash state.
- `resolve-source-path-priority` — Path resolution follows a strict precedence: `db_dir` > agent-specific repo > generic repo split, and the first successful hit wins; a `None` return is the sole signal of failure.
- `prefix-hash-is-not-stale-with-upgrade` — When `upgrade_hashes=True`, a stored hash that is a strict prefix of the current file's full hash is treated as a format mismatch (upgraded in-place), not as content drift; without the flag it is reported as `"truncated_hash"`.
- `hash-sources-is-idempotent` — Calling `hash_sources` without `force=True` on a node that already has a non-empty `source_hash` is a no-op; the function never overwrites existing hashes.
- `check-stale-returns-structured-not-raising` — `check_stale` never raises for expected failure modes (missing files, truncated hashes); all anomalies are captured in the returned `results` list as dicts with a `reason` key, making it safe to call in bulk without try/except.
