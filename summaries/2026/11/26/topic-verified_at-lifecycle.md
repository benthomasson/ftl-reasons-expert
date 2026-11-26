---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How `verified_at` is written, read, and used across `show_node`, `check_stale`, and export — the interaction between this stamp and `pinned_sha` matters for staleness

## Summary

`verified_at` has two distinct homes in this codebase, and that split matters:

1. **Top-level `Node.verified_at` / datastore column** — defined in `reasons/__init__.py:48`, passed through `reasons/network.py:84,112`, persisted in SQLite (`reasons/storage.py:32,145,237`) and Postgres (`reasons/pg.py:36,140,683,692,732,1348,1358,1376,2357`).
2. **Metadata key `node.metadata["verified_at"]`** — written by `check_stale`’s git-aware pinning paths (`reasons/check_stale.py:119,154,350,415,463`).

`show_node` reads the **top-level** field (`reasons/api.py:908`; `reasons/pg.py:683,692,732`), while `check_stale` writes the **metadata** key. No synchronization between those two locations is visible in the observations. That means a `check_stale` pin refresh can update `metadata["verified_at"]` without changing the `verified_at` value returned by `show_node`.

`pinned_sha` lives in `node.metadata`. `check_stale` uses `pinned_sha`/`pinned_lines` as the git anchor for staleness checks, and writes `metadata["verified_at"]` as a side effect when it advances that pin. The stamp itself is not used to decide staleness.

---

## Where `verified_at` is written

### 1. API verification writes the top-level `Node.verified_at`

`reasons/api.py:3800` documents that verification “Stamps verified_at on CONFIRMED beliefs.” The actual write is:

```python
net.nodes[bid].verified_at = now
```

at `reasons/api.py:3892`.

This is the path that updates the field `show_node` reads.

### 2. JSON/import paths load the top-level field

`reasons/api.py:2188` and `reasons/api.py:2232` assign:

```python
node.verified_at = ndata.get("verified_at", "")
```

So imported/serialized node data with a top-level `verified_at` populates `Node.verified_at`.

### 3. `check_stale` writes metadata `verified_at` while advancing `pinned_sha`

In `check_stale`, when `git_aware=True` and a `pinned_sha` exists, the function reads `pinned_sha` from metadata at `reasons/check_stale.py:105` and enters a git-aware branch at `:106`.

If `pinned_lines` are present and the relevant lines have not changed since the pinned commit, `check_stale`:

- reads the current commit SHA,
- updates `node.metadata["pinned_sha"]` (`:118`),
- stamps `node.metadata["verified_at"]` (`:119`),
- refreshes `node.source_hash`,
- increments `sha_bumped`,
- and skips stale reporting.

The same metadata stamp appears again at `reasons/check_stale.py:154` in the branch where the full content hash is unchanged but the current commit SHA differs from `pinned_sha`.

The pin/bump helpers in `check_stale.py` also write metadata `verified_at`:

- docstring at `reasons/check_stale.py:320`: “Stores pinned_sha and verified_at in node metadata”
- actual writes at `reasons/check_stale.py:350` and `:415`
- another stamp at `reasons/check_stale.py:463`

So within `check_stale`, `verified_at` is coupled to **pinning or re-pinning a source**, not to the main top-level verification field.

---

## Where `verified_at` is read

### `show_node` reads the top-level field

The SQLite/API `show_node` return includes:

```python
"verified_at": node.verified_at,
```

at `reasons/api.py:908`.

The Postgres implementation is consistent: `reasons/pg.py:683` selects `verified_at` alongside other timestamp columns, `reasons/pg.py:692` unpacks it, and `reasons/pg.py:732` returns it as `"verified_at": _fmt_ts(verified_at)`.

`show_node` also exposes `pinned_sha` and `pinned_lines` for node sources at `reasons/api.py:922` and `reasons/api.py:927`. Those come from the `node_sources` table, not from `Node.verified_at`.

So a caller of `show_node` sees:

- top-level `verified_at` from the node/database column,
- per-source `pinned_sha`/`pinned_lines`.

It does **not** get `check_stale`’s `metadata["verified_at"]` through the explicit top-level `verified_at` field. If metadata is returned elsewhere in the response, that is a separate nested value.

### An additional API serialization path reads `n.verified_at`

`reasons/api.py:1865` serializes:

```python
"verified_at": n.verified_at,
```

That is another reader of the top-level field. The observations also show source serialization around `reasons/api.py:1834-1838`, including `source_hash`, `pinned_sha`, and `pinned_lines`. If this is the JSON/network export path, it exports the top-level `verified_at` and per-source pin data, unlike the Markdown/card exporters.

### `check_stale` does not read `verified_at`

Across the observed `check_stale` logic, staleness decisions use:

- `node.source_hash`,
- `node.metadata.get("pinned_sha")` (`reasons/check_stale.py:105`),
- `node.metadata.get("pinned_lines")` (`:107`),
- git checks such as `file_lines_changed_since(...)` (`:115`) and `file_changed_since(...)` (`:123`),
- `get_file_commit_sha(...)` (`:117`, `:151`).

No observed staleness branch reads `verified_at`. The stamp is bookkeeping: “the pin was checked/advanced today,” not “if this date is too old, mark stale.”

### The Markdown and card exporters do not read `verified_at`

`reasons/export_card.py` builds stats from node truth values, justifications, nogoods, and metadata but never references `verified_at` or `pinned_sha`.

`reasons/export_markdown.py` exports source information such as `node.source_hash`, and uses `metadata["stale_reason"]` / `metadata["retract_reason"]` to display `STALE` status, but it does not export `verified_at` or `pinned_sha`.

Thus the primary Markdown/card exports cannot tell a consumer when a source pin was last verified or what SHA was pinned.

---

## Interaction between `verified_at` and `pinned_sha` for staleness

The important behavior is inside `reasons/check_stale.py`.

When `git_aware=True` and `pinned_sha` is present:

1. If `pinned_lines` exist, `check_stale` tests whether the pinned line range changed since `pinned_sha` (`reasons/check_stale.py:107-115`).
2. If those lines are unchanged, it advances `pinned_sha` to the current file commit SHA and writes `metadata["verified_at"] = date.today().isoformat()` (`:117-119`). It also refreshes `source_hash`. This is treated as a successful non-stale check and the loop continues.
3. If `pinned_lines` do not exist, `check_stale` can skip content hashing when `file_changed_since(path, pinned_sha)` is false (`:123`). That early-skip path does **not** write `metadata["verified_at"]`.
4. If the content hash is unchanged but the current commit SHA differs from `pinned_sha`, `check_stale` updates `pinned_sha` and stamps `metadata["verified_at"]` at `:150-154`.

So `metadata["verified_at"]` means something like “this source pin was refreshed/verified on this date.” It is tied to `pinned_sha` advancement or explicit pin/bump operations.

But the actual stale decision is driven by `source_hash`, `source_deleted`, truncated hashes, and the git checks. `verified_at` does not gate the decision and is not compared to a TTL.

The practical risk is the split storage:

- `check_stale` can update `node.metadata["pinned_sha"]` and `node.metadata["verified_at"]`.
- `show_node` reads `node.verified_at`, which comes from the top-level node/database column.
- The observations do not show code that copies `metadata["verified_at"]` into `node.verified_at`.

Unless some unobserved save/import reconciliation exists, a `check_stale` pin refresh may not be reflected in `show_node` or in the Markdown/card exports.

---

## Caveats / missing observations

The observations are enough to identify the split between top-level `verified_at` and metadata `verified_at`, but not enough to prove whether reconciliation happens elsewhere.

Missing pieces:

- The full bodies of `reasons/api.py:2792-2851` to confirm exactly how the API pin/bump functions write `pinned_sha` and `verified_at`.
- The full bodies of `reasons/check_stale.py:320-463` to name the pin/bump helpers precisely and see all write paths.
- The enclosing function for `reasons/api.py:1865`, to confirm whether that is the JSON export path.
- Save/load code that might merge `node.metadata["verified_at"]` into `Node.verified_at` or vice versa.
- Whether `show_node` returns the full metadata dict in addition to the explicit top-level `verified_at` field.

---

## Topics to Explore

- [function] `reasons/check_stale.py:check_stale` — Trace the complete git-aware branch order, especially where `pinned_sha` advances and `metadata["verified_at"]` is stamped.
- [function] `reasons/api.py:show_node` — Confirm exactly how top-level `verified_at`, source `pinned_sha`, and any metadata are assembled in the returned node detail.
- [file] `reasons/api.py` — Inspect the pin/bump functions around lines 2792-2851 and the serialization path around line 1865 to see whether they write top-level or metadata `verified_at`.
- [file] `reasons/export_markdown.py` — Understand why exports use `source_hash` and `stale_reason` but omit `verified_at` and `pinned_sha`.
- [general] `metadata-vs-node-field-reconciliation` — Determine whether `node.metadata["verified_at"]` is ever synchronized into the top-level `Node.verified_at` / database column.

---

## Beliefs

- `verified-at-has-two-write-paths` — Top-level `node.verified_at` is written by API verification (`reasons/api.py:3892`), while metadata `node.metadata["verified_at"]` is written by `check_stale` pin/bump paths (`reasons/check_stale.py:119,154,350,415,463`).
- `show-node-reads-top-level-verified-at` — `show_node` returns `node.verified_at` (`reasons/api.py:908`) and Postgres selects the `verified_at` column (`reasons/pg.py:683,692,732`), not `node.metadata["verified_at"]`.
- `check-stale-uses-pinned-sha-not-verified-at-for-staleness` — `check_stale` reads `pinned_sha` and `pinned_lines` to decide whether to skip content hashing (`reasons/check_stale.py:105-123`); no observed staleness branch reads `verified_at`.
- `check-stale-stamps-verified-at-when-pin-advances` — In git-aware mode, `check_stale` updates `metadata["pinned_sha"]` and `metadata["verified_at"]` together when the current commit SHA differs from the pinned SHA (`reasons/check_stale.py:117-119`, `:150-154`).
- `markdown-card-exports-omit-verified-at` — `reasons/export_markdown.py` and `reasons/export_card.py` do not reference `verified_at` or `pinned_sha`; Markdown exports `source_hash` and stale/retract reasons instead.
