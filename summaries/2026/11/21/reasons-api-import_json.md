---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: import_json in reasons/api.py

## `import_json` — Lossless Network Import from JSON

### Purpose

`import_json` is the counterpart to `export_network`. It reconstructs a full reason-maintenance network — nodes, justifications (with outlists), truth values, metadata, timestamps, nogoods, repos, tags, and per-node sources — from a previously-exported JSON file. The stated contract is a **lossless round-trip**: `export → import → export` should yield an equivalent network.

It exists so that the system can move networks between machines, publish/restore snapshots (via `import_hf` / `import_api`), and seed fresh databases from serialized exports.

---

### Contract

**Preconditions (relied on, not enforced by the type system):**
- `json_file` points to a valid JSON document produced by `export_network`. It is assumed to have:
  - `"meta"` → `dict` with `schema_version`, `project_name`, `created_at`
  - `"nodes"` → `dict[node_id, node_dict]` where `node_dict` has `text`, `truth_value`, `justifications` (list of dicts with at least `"type"`), `source`, `source_url`, `source_hash`, `date`, `metadata`, `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at`, `tags` (list), `sources` (list)
  - `"nogoods"` → `list` where each item has at least `"id"`
  - `"repos"` → `dict[name, path]`
- `db_path`'s SQLite schema must contain `nodes`, `node_tags`, and `node_sources` tables (i.e., a `reasons.db` initialized by `init_db`).
- Each justification dict must have a `"type"` key (see Error Handling — this is not defended against).

**Postconditions (on success):**
- All nodes from `data["nodes"]` that were not already present in the target network have been added, with their exported truth values restored.
- Antecedent/outlist dependencies that are themselves in the JSON are added before their dependents (subject to the fallback path).
- Nogoods with new IDs are appended to `net.nogoods`. Existing IDs are preserved.
- `net.repos` entries are merged in (existing entries overwritten by key).
- Three meta fields (`schema_version`, `project_name`, `created_at`) are copied onto `net.meta`.
- Tags and sources are inserted into `node_tags` and `node_sources` via a separate raw SQLite connection *after* `_with_network` has saved the network.
- Returns `{"nodes_imported": int, "nogoods_imported": int}`.

**Invariants maintained:**
- `net.nodes` is not left in a state where a node's `truth_value` differs from `data["nodes"][nid]["truth_value"]` (restored via `retract`/`assert_node`).
- Timestamps `reviewed_at`, `verified_at`, `retracted_at`, `updated_at` are set *after* `retract`/`assert_node` so they aren't overwritten by the TMS.

---

### Parameters

| Param | Type | Meaning | Edge Cases |
|---|---|---|---|
| `json_file` | `str` | Filesystem path to the JSON export | `FileNotFoundError` if missing. JSON parse errors propagate. |
| `db_path` | `str` (default `DEFAULT_DB` = `"reasons.db"`) | SQLite database path | Never used if `pg_conninfo` is truthy. |
| `pg_conninfo` | `None` or truthy connection string | When truthy, delegates to the PostgreSQL backend | **Short-circuits** the entire SQLite code path, including the tags/sources restoration. |
| `domain_id` | `None` or str | Passed only to the PG dispatcher | Ignored on SQLite path. |

---

### Return Value

`{"nodes_imported": int, "nogoods_imported": int}`

- `nodes_imported` counts nodes actually *added* in this call. Nodes already present in the target DB are skipped and **not** counted.
- `nogoods_imported` counts nogoods appended; nogoods whose `id` already exists are skipped.
- The local variable `skipped` is incremented for pre-existing nodes but **is not returned** — it's dead code.
- On any unhandled exception, no return value is produced (see Error Handling).

---

### Algorithm Walkthrough

1. **Validate file** — `Path(json_file).exists()`, raise `FileNotFoundError` if not. Parse JSON via a *local* `import json as json_mod` (shadows the module-level `json` inside local scope).

2. **PG dispatch** — If `pg_conninfo` is truthy, call `_pg_dispatch(pg_conninfo, domain_id, "import_json", data=data)` and return. Notice the **parsed dict** is passed, not the file path.

3. **Open write context** — `with _with_network(db_path, write=True) as net:` loads the network fresh, exposes it for mutation, and saves on successful exit.

4. **Copy meta** — Only `schema_version`, `project_name`, `created_at`. Other imported meta keys (e.g., `node_count`) are silently dropped.

5. **Topological-convergence loop** —
   - `remaining = data["nodes"]` (dict), `added = set(net.nodes.keys())`.
   - For up to `len(remaining) + 1` passes:
     - Build `next_remaining`.
     - For each `(nid, ndata)`: if `nid` already in `added`, `skipped += 1; continue`.
     - Compute `all_deps` from every justification's `antecedents` + `outlist`.
     - Filter to `deps_in_data` — **only dependencies that appear in the JSON** must be satisfied. Dependencies referencing nodes already in the DB (in `added`) or nodes absent entirely contribute nothing.
     - If `all(d in added for d in deps_in_data)`: build `Justification` list, `net.add_node(...)`, then restore truth value, then restore timestamps, add to `added`, increment `nodes_imported`.
     - Otherwise, push to `next_remaining`.
     - **No-progress fallback**: if `len(next_remaining) == len(remaining)`, the same add-node block is duplicated to force-add every remaining node (handles cycles / unreachable antecedents), then `break`.

6. **Truth-value restoration** — After `add_node`, `node.truth_value` is whatever the TMS computed. If it differs from `ndata["truth_value"]` (`"IN"` default):
   - `"OUT"` → `net.retract(nid)`
   - anything else → `net.assert_node(nid)`
   Then `reviewed_at`, `verified_at`, `retracted_at`, `updated_at` are re-set explicitly so `retract`/`assert` don't stomp them.

7. **Nogoods** — `existing_nogood_ids` from `net.nogoods`; skip duplicates; append fresh `Nogood` objects.

8. **Repos** — `net.repos[name] = path` for each.

9. **Context exit** — `_with_network.__exit__` calls `store.save(network)`, recomputes truth-change pubsub events, and closes the store.

10. **Raw SQLite tags/sources** — A *separate* `sqlite3.connect(db_path)` (JSON-serialized tags/sources aren't part of `Network` state) inserts rows into `node_tags` and `node_sources` with `INSERT OR IGNORE`. `conn.commit()` in the `try`; `conn.close()` in `finally`.

---

### Side Effects

- **Writes** to `db_path`: new nodes, justifications, dependents index, nogoods, repos, meta, plus `node_tags` and `node_sources` rows.
- **Invalidates** the `_network_cache` for `db_path` on success (via `_with_network`).
- **Publishes** truth-value change events to `pubsub` subscribers (if any) — this can fire during import because `retract`/`assert_node` change truth values.
- **Cascades** during truth restoration: `net.retract`/`net.assert_node` propagate through the graph. Because nodes are added in dependency order, cascades should typically touch only already-added downstream nodes, but force-added nodes in the fallback branch can cascade unpredictably before their antecedents exist.
- **No network I/O** — pure filesystem + SQLite.

---

### Error Handling

**Raised (not swallowed):**
- `FileNotFoundError` — `json_file` missing.
- `json.JSONDecodeError` — malformed JSON.
- `KeyError` — if a justification dict lacks `"type"` (`j["type"]` in the list comp), or a nogood dict lacks `"id"` (`ng_data["id"]`).
- Any exception raised by `net.add_node`, `net.retract`, `net.assert_node`, `Storage.save`.
- `sqlite3.OperationalError` — if `node_tags`/`node_sources` don't exist.
- `sqlite3.IntegrityError` — if a FK on `node_tags`/`node_sources` is violated (unlikely since nodes were just saved, but possible if a tags/sources entry references a node ID not present — `INSERT OR IGNORE` does **not** suppress FK violations).

**Silently tolerated:**
- Nodes already present → skipped, not updated.
- Nogoods with duplicate IDs → skipped.
- Missing optional keys via `.get(...)` with sensible defaults.
- "No progress" in the topo loop → force-add path.

**Notable failure mode:** If an exception fires inside `_with_network`, the context's `__exit__` returns `False`, closes the store, and *the network is not saved* — but the intermediate `_network_cache` is **also not updated** (because `exc_type` is set). Any pending write is lost. The raw-sqlite tags/sources pass never runs.

---

### Usage Patterns

- **CLI**: `reasons import-json network.json` → calls `import_json(path)`.
- **Indirect**: `import_hf` writes the downloaded JSON to a `NamedTemporaryFile`, calls `import_json`, then unlinks the temp file.
- **Indirect**: `import_api` — same temp-file dance after fetching from the mind service.
- **PG mode**: Callers pass `pg_conninfo` and `domain_id`; the parameter `db_path` is then irrelevant.

**Caller obligations:**
- Target DB must already exist and be schema-initialized (`init_db` if needed). `import_json` does **not** create the DB.
- Re-importing into a populated DB is *approximately* idempotent for tags/sources/nogoods, but existing nodes are **skipped**, not updated — no upsert semantics.
- If fidelity of extra meta keys matters, inspect `net.meta` after the call; only three keys survive.

---

### Dependencies

- `reasons.api._with_network` — network load/save lifecycle + cache invalidation.
- `reasons.api._pg_dispatch` → `reasons.pg.PgApi.import_json`.
- `reasons.Justification`, `reasons.Nogood` — constructed manually.
- `reasons.network.Network.add_node`, `.retract`, `.assert_node`, `.nodes`, `.nogoods`, `.repos`, `.meta`.
- `reasons.storage.Storage.save` (via `_with_network`).
- `reasons.pubsub.compute_changes` / `.publish` (via `_with_network`).
- Stdlib: `json`, `pathlib.Path`, `sqlite3`.

---

### Assumptions Not Enforced by Types

1. **Export schema assumed, not validated.** A JSON with a wrong-shaped `"nodes"` (e.g., a list) would fail at `.items()` — but the failure is an `AttributeError`, not a friendly validation error.
2. **`truth_value` must be exactly `"IN"` or `"OUT"`.** Any other string is treated as `assert_node` (i.e., forces IN).
3. **Antecedents/outlists referencing unknown nodes are tolerated.** The `deps_in_data` filter only requires *JSON-present* deps to be added first. This relies on `Network.add_node` treating missing antecedents as OUT rather than raising — behavior not obvious from this function.
4. **`node_tags`/`node_sources` exist as tables with those exact column names.** No schema check is performed here.
5. **`INSERT OR IGNORE` is presumed safe** — but it silently collapses duplicate identical source rows, meaning the "lossless" promise can break if two sources with identical `(source_type, source_ref, source_url, source_hash, pinned_sha, pinned_lines, label, added_at)` existed.
6. **PG mode is not exercised here** — the entire tags/sources restoration pass is SQLite-only, so exporter→importer symmetry on PG depends entirely on `PgApi.import_json`.
7. **`skipped` and duplicated add-logic** are leftovers; the function tolerates them without correctness impact.

---

## Topics to Explore

- [function] `reasons/api.py:export_network` — The counterpart that produces the JSON this function expects; verifying the schema here pins down the round-trip contract.
- [function] `reasons/api.py:_with_network` — Manages network load/save, cache, and pubsub event emission — critical to understanding when writes commit and when the raw SQLite tags/sources pass runs.
- [function] `reasons/network.py:Network.add_node` — Determines initial `truth_value` computation and how missing antecedents are handled (the fallback path depends on this behavior).
- [function] `reasons/pg.py:PgApi.import_json` — The PostgreSQL implementation dispatched to when `pg_conninfo` is supplied; must preserve the tags/sources semantics the SQLite path handles outside the `Network` model.
- [file] `reasons/storage.py` — Defines `node_tags` / `node_sources` DDL and the save logic that commits nodes prior to the raw SQLite pass.

---

## Beliefs

- `import-json-requires-export-schema` — `import_json` assumes the exact structure produced by `export_network` (dict-keyed nodes with `justifications`/`truth_value`, nogoods with `id`, meta dict); missing required keys raise `KeyError` rather than validating.
- `import-json-restores-truth-values-after-propagation` — After `net.add_node` computes a truth value, `import_json` compares it to the exported value and forces it via `retract`/`assert_node`, then re-applies `reviewed_at`/`verified_at`/`retracted_at`/`updated_at` so the TMS doesn't overwrite them.
- `import-json-topological-sort-enables-safe-cascades` — Nodes are added in dependency order so that the `retract`/`assert_node` cascades during truth restoration only affect nodes that haven't been added yet, minimizing propagation from the import itself.
- `import-json-writes-tags-and-sources-via-raw-sqlite` — Because tags and sources are not part of the `Network` save path, `import_json` opens a fresh `sqlite3.connect(db_path)` after `_with_network` exits and inserts into `node_tags` and `node_sources` using `INSERT OR IGNORE`.
- `import-json-pg-dispatch-bypasses-sqlite-block` — When `pg_conninfo` is truthy, `import_json` returns immediately after `_pg_dispatch(...)`; the raw SQLite tags/sources restoration is never reached.
- `import-json-skips-existing-nodes` — Nodes already present in the target network are counted in a local `skipped` variable but are not inserted or updated, so re-import is an approximate no-op rather than an upsert; `skipped` is not included in the return value.
- `import-json-only-restores-three-meta-keys` — Of the source `meta` dict, only `schema_version`, `project_name`, and `created_at` are copied to `net.meta`; all other keys are dropped, so round-trip fidelity of `meta` is partial.
