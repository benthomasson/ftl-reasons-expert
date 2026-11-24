---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: export_network in reasons/api.py

# `export_network` — Full Explanation

## Purpose

`export_network` is the top-level read-only snapshot API. It serializes the entire belief network (nodes with their justifications and metadata, plus nogoods and repo config) into a plain dict that is JSON-serializable, suitable for HuggingFace publishing, review workflows, dedup, contradiction detection, and LLM prompting.

It is the counterpart to `import_json`, `import_hf`, and `import_api` — those functions reconstruct a network from a dict shaped like this one.

## Contract

**Preconditions**
- If `pg_conninfo` is falsy: a readable SQLite DB exists at `db_path`, and it contains a valid `node_tags` and `node_sources` schema (the network tables themselves are loaded through `_with_network`).
- If `pg_conninfo` is truthy: a valid PG connection string and `domain_id` are provided.

**Postconditions**
- No mutation. The function uses `_with_network(db_path)` without `write=True`, so no save is triggered and the database is untouched.
- Returns a fresh dict with exactly the four keys `meta`, `nodes`, `nogoods`, `repos`.
- `meta["node_count"]` reflects the number of nodes **after** visibility filtering, not the total.
- Node IDs appearing only in `node_tags` / `node_sources` (stale orphan rows) are silently dropped because lookup is scoped to `net.nodes`.

**Invariants**
- `nodes` iteration order is deterministic (`sorted(net.nodes.items())`), so two calls against the same DB produce byte-identical dicts (JSON with `sort_keys=True` yields stable output).
- `justifications` preserves the node's internal ordering. `supporting_justification` is a **positional index** into that same list (see `show_node` and `_node_depth`, which both treat it as an index).
- Metadata keys beginning with `_` are stripped (used internally to hide sentinel fields like `_retracted`).
- `dependents` are **not** exported — callers needing reverse edges must enrich themselves (see `propose_update`).

## Parameters

| Param | Type | Behavior |
|---|---|---|
| `visible_to` | `list[str] \| None` | Access filter. `None` = no filter. Otherwise a node is visible iff every `access_tags` entry is in `visible_to` (conjunctive — see `_is_visible`). `visible_to=[]` hides *all* nodes that carry any `access_tags`. |
| `db_path` | `str` | Path to SQLite DB. Default `"reasons.db"`. Must already exist — `sqlite3.connect` is opened directly after `_with_network`. |
| `pg_conninfo` | Any truthy value | If set, dispatches to `PgApi.export_network(visible_to=...)`. `domain_id` is passed through. |
| `domain_id` | Any | Only meaningful with `pg_conninfo`. |

**Edge cases**
- Non-existent node referenced in a nogood → nogood is dropped (`n in net.nodes` guard).
- Non-visible participant in a nogood → nogood is dropped (stricter than node filter — one hidden participant kills the whole nogood).
- Justifications can still reference *invisible* antecedent node IDs — a partial view is exported. Callers must tolerate dangling antecedent references.
- `net.meta` is assumed to be dict-like (`.get`); absent keys default to `""`.

## Return Value

```python
{
  "meta":    {"schema_version": ..., "project_name": ..., "created_at": ..., "node_count": int},
  "nodes":   {node_id: {text, truth_value, supporting_justification, justifications,
                        source, source_url, source_hash, text_hash, date,
                        metadata, tags, sources,
                        created_at, updated_at, reviewed_at, verified_at, retracted_at}},
  "nogoods": [{"id", "nodes", "discovered", "resolution"}, ...],
  "repos":   {name: path, ...},
}
```

`nodes` is a **dict keyed by node id**, not a list. Callers that iterate assume this shape everywhere (`for nid, ndata in data["nodes"].items()`).

## Algorithm Walkthrough

1. **Backend dispatch** — if `pg_conninfo` is set, immediately delegate and return.
2. **Enter read context** — `_with_network(db_path)` loads (or reuses cached) `Network`.
3. **Fetch side tables via direct SQLite** — the `Network` object does not carry tags/sources, so two queries run:
   - `SELECT node_id, tag FROM node_tags ORDER BY node_id, tag` → `all_tags`
   - `SELECT node_id, source_type, source_ref, source_url, source_hash, pinned_sha, pinned_lines, label, added_at FROM node_sources ORDER BY node_id, rowid` → `all_sources` (ordered by rowid within each node to preserve insertion order).
   - Connection is closed in a `finally`.
4. **Build `nodes` dict comprehension** — for each `(nid, n)` in sorted order, if visible:
   - Copy scalar fields directly.
   - Serialize each `Justification` as a dict of `type`, `antecedents`, `outlist`, `label`, `content_hash`.
   - Strip `_`-prefixed metadata keys.
   - Attach `all_tags.get(nid, [])` and `all_sources.get(nid, [])`.
5. **Build `meta`** — `build_meta(project_name, node_count=len(nodes), created_at)` — note `node_count` is the *post-filter* count.
6. **Build `nogoods`** — include only nogoods whose every participant is both present and visible.
7. **Repos** — `dict(net.repos)` copy.

## Side Effects

- Reads two SQLite tables (`node_tags`, `node_sources`).
- Populates/refreshes `_network_cache[db_path]` via `_with_network.__exit__` (updates mtime, stores the `Network` object).
- No write, no publish, no file output.
- Returned dict **aliases** mutable sub-structures from the live `Network`:
  - `justifications[i]["antecedents"]` and `["outlist"]` are the actual `list` objects from `Node.justifications`.
  - Metadata values are shared references.
  - If a caller mutates `data["nodes"][nid]["justifications"][0]["antecedents"]`, it corrupts the cached `Network` (the next read sees the mutated data). This is not enforced by the type system.

## Error Handling

- No explicit `try/except` around the network load or SQLite queries (only `conn.close()` in `finally`).
- Can raise:
  - `sqlite3.OperationalError` / `DatabaseError` if `node_tags` or `node_sources` tables are missing or the file is corrupt.
  - Any exception raised by `Storage.load()` / `Network.__init__` via `_with_network`.
  - Whatever `PgApi.export_network` raises when `pg_conninfo` is used.
- Silently tolerated:
  - Nodes with no tags/sources (empty `[]`).
  - Nogoods referencing missing or invisible nodes (dropped).
  - Nodes cached in `net.nodes` but absent from `node_tags`/`node_sources`.

## Usage Patterns

Called by many higher-level workflows. Representative examples:

- `publish_hf` → `json.dumps(export_network(visible_to=..., **backend), sort_keys=True)` for `network.json`.
- `classify_defeat_reason_types` → reads `metadata.defeats_node` from every probe.
- `review_beliefs` / `review_premises` / `propose_update` / `detect_contradictions` → get a full node map to feed LLMs.
- `repair_smuggled` / `repair` → obtain all node texts to link against.
- `derive_prompt` → build a prompt from `data["nodes"]`.
- `export_markdown` / `export_card` (pg path) → rebuild a `Network` object from this dict.

**Caller obligations**
- Treat the returned dict as a **snapshot** — if you need consistency across multiple reads, spin up a `_with_network` context yourself.
- Do not mutate returned `antecedents`/`outlist` lists; copy first.
- Add `dependents` yourself if you need reverse edges (see `propose_update`'s enrichment block).
- If you need a sizeable number of nodes, `visible_to` is a **conjunctive** filter (all tags must match), not a union.

## Dependencies

- `_with_network` (`reasons/api.py`) — read-only load + cache.
- `_is_visible` (`reasons/api.py`) — access-tag check.
- `build_meta` (`reasons/metadata.py`) — meta construction.
- `_pg_dispatch` and `reasons.pg.PgApi` — PG backend.
- `sqlite3`, `DEFAULT_DB`.
- Implicitly, `reasons.network.Network`, `reasons.Node`, `reasons.Justification`, `reasons.Nogood` (attribute access on `net.nodes`, `n.justifications`, `net.nogoods`).

## Assumptions Not Enforced by Types

- `visible_to` is expected to be a **whitelist of access tags the caller is authorized for**, not a role hierarchy — the check is flat `all(t in visible_set ...)`.
- `n.supporting_justification` is treated as an *integer index* into `n.justifications` (used elsewhere as such, e.g. `_node_depth`); if the `Node` type ever changed this to a direct reference, `export_network` would still serialize it verbatim without complaint.
- `net.meta` is dict-like with `.get`; `net.repos` is dict-like; `net.nogoods` is iterable with `.id / .nodes / .discovered / .resolution`.
- `n.metadata` values are JSON-serializable; the function does not coerce/validate. Anything non-serializable will explode later in `json.dumps`.
- SQLite DB path is a real file with valid schemas — no `IF NOT EXISTS` defensive handling of the `node_tags`/`node_sources` tables.
- Exported nodes may contain justification references to nodes not present in the output (access filtering is per-node, not transitive). Downstream consumers assuming a closed graph will break.
- `_`-prefixed metadata keys are treated as *internal* — any user metadata starting with `_` is invisibly lost on export.

## Topics to Explore

- [function] `reasons/api.py:_is_visible` — Confirm the conjunctive semantics of `access_tags` filtering and how multiple tags interact.
- [file] `reasons/metadata.py` — See what `build_meta` injects into `meta` (schema_version, timestamps, etc.) so callers know the full shape.
- [function] `reasons/api.py:import_json` — Trace the inverse round-trip: reconstructing a `Network` from an output of `export_network`, especially the topological sort and timestamp restoration.
- [function] `reasons/api.py:propose_update` — The canonical example of enriching output with `dependents` and using `what_if_retract` after export.
- [general] `pg-backend-export-parity` — Check `reasons/pg.py` to understand whether the PG path yields an identically shaped dict (fields, sorts, filtering) — important for cross-backend portability.

## Beliefs

- `export-network-is-read-only` — `export_network` never saves; it opens the network via `_with_network(db_path)` without `write=True` and performs only reads.
- `export-network-includes-tags-and-sources` — `tags` and `sources` are populated from the separate `node_tags` and `node_sources` SQLite tables, not from the in-memory `Network` object.
- `export-network-filters-metadata-underscore-keys` — Metadata keys beginning with `_` are stripped from every exported node, hiding internal sentinels like `_retracted`.
- `export-network-omits-dependents` — The export schema does not include reverse `dependents` edges; callers like `propose_update` must enrich the dict themselves.
- `export-network-conjunctive-acl` — A node is included only when *every* tag in its `access_tags` appears in `visible_to`; a single unmatched tag hides the node.
- `export-network-drops-partial-nogoods` — A nogood is exported only if every referenced node is present in `net.nodes` and passes the visibility check.
- `export-network-shared-mutable-references` — The returned dict shares `antecedents`, `outlist`, and metadata-value references with the cached in-memory `Network`, so caller mutation can corrupt cached state.
