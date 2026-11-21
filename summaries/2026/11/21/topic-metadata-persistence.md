---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `pinned_sha`, `pinned_lines`, and `verified_at` survive a save/reload cycle in storage/`pg.py`; the tests only check in-memory state.

# How `pinned_sha`, `pinned_lines`, and `verified_at` survive a save/reload cycle

## First, a clarification: there is no `storage/pg.py`

The question blends two separate persistence layers that live in two different files:

| File | Backend | Has `save()` / `load()`? |
|---|---|---|
| `reasons/storage.py` | SQLite | **Yes** — `Storage.save()` and `Storage.load()` |
| `reasons/pg.py` | PostgreSQL | **No** — module docstring (line 2): *"Each operation is a SQL transaction — no full-network load/save."* |

So the two fields groups survive **reload** differently in each backend, and the "tests only check in-memory state" concern really applies to pinning (`tests/test_pin_sources.py`), not to pg.

---

## A. `verified_at`

### SQLite — `reasons/storage.py`
- It is a **real column** on `nodes`: `verified_at TEXT DEFAULT ''` inside the `SCHEMA` block.
- `Storage.save()` inserts it as one of the 15 parameters of the `INSERT INTO nodes (...)` statement (right after `reviewed_at`, before `retracted_at`), sourcing `node.verified_at`.
- `Storage.load()` reads it back in the `SELECT` (the `has_timestamps` branch: `"created_at, updated_at, reviewed_at, verified_at, retracted_at"`) and passes `verified_at=verified_at or ""` into the `Node(...)` constructor.
- Round-trip is column-to-column; `metadata_json` is not involved.

### PostgreSQL — `reasons/pg.py`
- It is a **first-class column** on `rms_nodes`: `verified_at TIMESTAMPTZ` in the `SCHEMA` at the top of `pg.py`.
- `show_node()` (`pg.py`) selects `verified_at` from `rms_nodes` and returns it via `_fmt_ts` as an ISO string.
- Because pg has no save/load, "survival" is automatic: the value lives in the row, and every caller of `PgApi` sees it on the next transaction.

**Caveat:** No writer for `rms_nodes.verified_at` appears in the shown 800 lines of `pg.py`. `_add_node_raw()` doesn't set it, and no `verify` API is visible. In the SQLite path the writer is `reasons/api.py:3892` (`net.nodes[bid].verified_at = now`), which only affects the in-memory `Network`.

---

## B. `pinned_sha` and `pinned_lines`

### SQLite — `reasons/storage.py`
- They are **not** columns on `nodes`; they live in `node_sources`:
  ```sql
  CREATE TABLE node_sources (
      ...
      pinned_sha TEXT DEFAULT '',
      pinned_lines TEXT DEFAULT '',
      ...
  );
  ```
- `Storage.save()` writes them **twice**:
  1. Because `node.metadata` is serialized wholesale with `json.dumps(node.metadata)` into `nodes.metadata_json` — and `pin_sources()` (per `tests/test_pin_sources.py`) stores pins in `node.metadata["pinned_sha"]` — the values automatically end up in `metadata_json`.
  2. Explicitly re-inserted into `node_sources` from `node.metadata.get("pinned_sha", "")` / `node.metadata.get("pinned_lines", "")` in the "Sync node_tags and node_sources" block.
- `save()` also **snapshots** any pre-existing rows in `node_sources` into `ext_sources`, deletes the table, and re-inserts rows that weren't overwritten — this preserves rows written out-of-band by `add_source()` without touching `Network`.
- **Reload path is the weakness**: `Storage.load()` reads only `nodes`, `justifications`, `nogoods`, `network_meta`, `repos`, `propagation_log`. It **never reads `node_sources` or `node_tags`.** So the only reason pins reappear in a loaded `Network` is that `save()` had also copied them into `metadata_json`.
- Secondary readers (`reasons/api.py:1834`, `:922`, `:1838`) query `node_sources` directly, so the value is still visible through `api.py` even without `load()` hydrating it — but the in-memory `Network` after a `load()` would lack a pin that was written exclusively through `add_source()`.

### PostgreSQL — `reasons/pg.py`
- The `SCHEMA` in `pg.py` has **no `pinned_sha`, no `pinned_lines`, and no `rms_node_sources` table** in the 800 lines shown.
- The only plausible home is `metadata JSONB DEFAULT '{}'` on `rms_nodes`, which `_add_node_raw()` populates via `json.dumps(meta)`.
- `reasons/api.py:2792` confirms the design intent: `"""Stores pinned_sha and verified_at in node metadata."""`; `reasons/check_stale.py:105` reads it back as `node.metadata.get("pinned_sha")`.
- **But** no `pg.py` method in the shown slice reads or updates `node.metadata["pinned_sha"]`. `show_node()` returns the whole `metadata` dict, so *if* an UPDATE placed the pin inside JSONB, it would survive — the code that performs that UPDATE is not visible in the 800-line excerpt.

---

## C. Why the tests don't catch this

| Test file | What it actually does | Does it exercise save/reload? |
|---|---|---|
| `tests/test_pin_sources.py` | Constructs `Network()` (e.g. `net = Network()` line ~132) and asserts `net.nodes["belief-1"].metadata["pinned_sha"] == results[0]["pinned_sha"]` | **No.** Never instantiates `Storage` or `PgApi`. Purely in-memory. |
| `tests/test_pg.py` | Asserts truth values, access tags, challenge/defense metadata, `show_node` shape | **No** assertion on `pinned_sha`, `pinned_lines`, or cross-connection `verified_at`. `TestMultiTenancy` proves *node rows* survive a second `PgApi` connection — but pin fields are not in any assertion. |

There is **no test in the observations** that does `Storage.save()` → `Storage.load()` and asserts a pin or `verified_at` survived. So the observation is accurate: persistence for pins is unverified.

---

## D. What's missing from the observations

1. The remaining ~2,936 lines of `reasons/pg.py` — need to confirm whether an `rms_node_sources` table or `metadata` UPDATE exists for pins, and whether `verified_at` has an UPDATE path.
2. `reasons/api.py` bodies around lines 2792–2900 — the `pin_sources`, `bump_pinned_sha`, `pin_lines` methods. Do they route through `PgApi` at all?
3. `Storage.load()` — confirm (as shown) that `node_sources` / `node_tags` are genuinely never read back.
4. Whether `api.py` wraps pg reads by re-deriving pins from JSONB or by querying a sources table (line 1834's `FROM node_sources` is SQLite-specific today).

---

## Topics to Explore

- [file] `reasons/pg.py` — Read the remaining 2,900+ lines to find any pin/verify mutation methods (`UPDATE rms_nodes SET metadata = ...`, `verify`, `pin_source`).
- [file] `reasons/storage.py:load` — Confirm that `load()` does not read `node_sources`/`node_tags`, and trace how a pin written only via `add_source()` would round-trip.
- [function] `reasons/api.py:pin_sources` — See whether the pin API updates `node.metadata` for pg, or only for a `Network`.
- [diff] `reasons/pg.py:SCHEMA vs reasons/storage.py:SCHEMA` — Compare column/table parity for `verified_at`, `pinned_sha`, `pinned_lines`, and `node_sources`.
- [general] `pin-persistence-tests` — Design a round-trip test that runs `pin_sources()` → `Storage.save()` → `Storage.load()` and asserts `node.metadata["pinned_sha"]`, plus an equivalent two-`PgApi`-connection test for `verified_at`.

---

## Beliefs

- `pg-api-is-transactional-without-save-load` — `PgApi` (reasons/pg.py) exposes no `save()`/`load()`; every mutation commits to PostgreSQL directly, so field survival is a property of the schema, not of a serialization step.
- `verified-at-is-a-first-class-column-in-both-backends` — `verified_at` is a dedicated column on `nodes` (SQLite SCHEMA) and on `rms_nodes` (pg SCHEMA), so it round-trips independently of `metadata`/JSONB.
- `pinned-fields-store-in-node-sources-on-sqlite` — In reasons/storage.py, `pinned_sha` and `pinned_lines` are columns of the `node_sources` table, written by `Storage.save()` from `node.metadata`; they are not columns of `nodes`.
- `storage-load-skips-node-sources` — `Storage.load()` (reasons/storage.py) rebuilds only `nodes`, `justifications`, `nogoods`, `network_meta`, `repos`, and `propagation_log` — it never reads `node_sources` or `node_tags`, so pins reappear only via the `metadata_json` copy written by `save()`.
- `pin-tests-are-in-memory-only` — `tests/test_pin_sources.py` builds a bare `Network()` and asserts `node.metadata`, so no test in the observations verifies `pinned_sha`/`pinned_lines` across a real storage round-trip.
- `pg-schema-lacks-pinned-columns` — The pg SCHEMA shown in `reasons/pg.py` contains neither `pinned_sha`/`pinned_lines` columns nor an `rms_node_sources` table, so any pg-side persistence of pins must go through the `metadata JSONB` column.
