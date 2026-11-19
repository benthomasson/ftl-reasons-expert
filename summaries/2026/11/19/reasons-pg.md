---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/pg.py

# `reasons/pg.py` — PostgreSQL-native storage backend

## 1. Purpose

This file implements a **PostgreSQL-backed storage layer** for the "dependency network" belief system (aka the *reasons* engine). It is a full alternative to an in-memory `Network` object that is loaded, mutated, and saved as a whole.

The core value proposition, stated in the module docstring, is:
- Every operation runs as its own **SQL transaction** — no full-network load/save.
- Enables **concurrent writers** and **multi-tenant** deployment via `domain_id`.
- Keeps the belief network consistent as a live database rather than a serialized blob.

Responsibilities it owns:
- Schema creation and migration (`SCHEMA`, `INDEXES`, `init_db`).
- All belief-graph mutations: add node, add/remove justification, retract, assert, convert-to-premise, supersede, summarize, challenge, defend, nogood handling.
- Truth-value propagation (`_propagate`, `_compute_truth`) — the "defeasible logic" engine.
- Read operations: status, node detail, search, lookup, list, compact, export.
- Source-file tracking (`hash_sources`, `check_stale`, `add_repo`).
- Multi-agent import/sync with namespace isolation (`import_agent`, `sync_agent`).
- Access control via `access_tags` metadata and per-node visibility.
- Everything is scoped to a `domain_id` (tenant/workspace).

---

## 2. Key Components

### Module-level constants

| Name | Role |
|---|---|
| `SCHEMA` | DDL for the five tables: `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, `rms_network_meta`. Includes `CHECK` constraints on `truth_value ∈ {IN,OUT}` and `type ∈ {SL,CP}`, and `ON DELETE CASCADE` from justifications to nodes. |
| `INDEXES` | Regular indexes on `(domain_id, truth_value)` etc., plus **GIN** indexes for full-text search (`idx_rms_nodes_fts`) and JSONB containment on `antecedents` / `outlist`. |
| `_require_psycopg()` | Raises `ImportError` with install hint if `psycopg` is not present. `psycopg` is imported inside a `try/except` at the top so the module can be imported without psycopg installed. |

### `PgApi` class

The whole file is effectively one class. Notable constructor behavior:

```python
def __init__(self, conninfo, domain_id):
    if isinstance(conninfo, str):
        self.conn = psycopg.connect(conninfo, autocommit=False)
        self._owns_conn = True
    else:
        self.conn = conninfo
        self._owns_conn = False
    self.domain_id = str(domain_id)
```

You can pass a connection string (owned) or an existing `psycopg.Connection` (not closed on `close()`). `__exit__` rolls back on exception, then closes if owned. Designed for `with PgApi(...) as api:` usage.

Public entry-point categories:

- **Lifecycle / schema**: `init_db`, `close`, `__enter__`, `__exit__`.
- **Core mutations**: `_add_node_raw`, `add_node`, `add_justification`, `retract_node`, `assert_node`, `remove_justification`, `set_metadata`, `convert_to_premise`, `supersede`, `summarize`.
- **Dialectical**: `challenge`, `defend`, `_challenge_internal`, `add_nogood`, `find_culprits`, `explain_node`, `trace_assumptions`, `trace_access_tags`.
- **Reads**: `get_status`, `show_node`, `search`, `lookup`, `list_nodes`, `list_gated`, `list_negative`, `topics`, `export_network`, `compact`, `get_log`.
- **Propagation**: `propagate` (full fixpoint), `_propagate` (BFS), `_compute_truth`.
- **Source tracking**: `hash_sources`, `check_stale`, `add_repo`, `list_repos`, `_load_repos`.
- **What-if**: `what_if_retract`, `what_if_assert`, `_collect_what_if_results`, `_cascade_depth_pg`.
- **Namespaces / agents**: `ensure_namespace`, `list_namespaces`, `import_agent`, `sync_agent`, `_import_agent_nogoods`.
- **Bulk imports**: `import_json`, `import_beliefs`.

Key contracts a maintainer should know:

- `add_node(...)` returns `{"node_id", "truth_value", "type": "premise"|"derived", "premise_count"}`. Raises `ValueError` if the node already exists.
- `add_justification(...)` returns the previous and new truth value, plus the list of nodes whose truth changed during propagation.
- `retract_node` / `assert_node` return `{"changed", "went_out", "went_in"}`, including the caller node and everything cascaded.
- `show_node` raises `KeyError` on missing nodes and `PermissionError` on tag-mismatch.
- `_add_node_raw` is the "unsafe" insert used by all import paths: it does **not** validate refs, does **not** inherit access tags, and does **not** commit — all callers manage that.

### Internal helpers worth knowing

| Helper | Purpose |
|---|---|
| `_propagate(cur, changed_id)` | BFS worklist over dependents; returns `(went_out, went_in)`. Loads dependents, justifications, and referenced truth values in **batches**, then re-evaluates each dependent. |
| `_compute_truth(cur, node_id)` | Single-node evaluator: node is `IN` if **any** justification's `antecedents all IN` and `outlist all OUT`. No justifications ⇒ premise ⇒ `IN`. |
| `_find_dependents(cur, node_ids)` | Finds nodes with any of the given IDs in `antecedents` or `outlist`, using JSONB containment (`@> '["x"]'`). Note: it issues one query per ID (not batched). |
| `_validate_refs` | Raises `KeyError` listing all missing referenced node IDs. |
| `_parse_justifications(sl, cp, unless, label)` | Comma-split parsing of the CLI-friendly `sl`/`cp`/`unless` string args into justification dicts. |
| `_inherit_access_tags` | Unions `access_tags` from all antecedents into the dependent node's metadata. |
| `_is_visible(metadata, visible_to)` | Node visible if it has no tags, or **all** its tags are in `visible_to`. |
| `_find_culprits_internal` | Uses `_trace_assumptions_recursive` to collect premises supporting each conflicting node, then ranks candidates by `_entrenchment`. |
| `_entrenchment` | Heuristic score: premise +100, has source +50, has source_hash +25, 10 × dependent count, plus a per-`beliefs_type` bonus (AXIOM/WARNING 90, OBSERVATION 80, DERIVED 40, PREDICTED 30, NOTE 10, else 20). Lowest score is chosen first. |
| `_cascade_depth_pg` | BFS from `source_id` to find hop distance to `target_id` for what-if reporting. |
| `_log(cur, action, target, value)` | Appends to `rms_propagation_log`; the append-only audit trail (`get_log` reads it). |

---

## 3. Patterns

- **Transaction-per-operation**: every public mutation ends with `self.conn.commit()`. There are no long-lived transactions spanning multiple operations. The `with PgApi(...) as api:` context manager exists only to roll back on exception and close the connection.
- **Explicit connection ownership**: `_owns_conn` prevents closing an injected connection (used by tests / connection pools).
- **Raw SQL, parameterized**: no ORM. `psycopg` `%s` placeholders everywhere except a few safe identifier interpolations (schema column names in migrations, `LIMIT/OFFSET`).
- **JSONB for graph edges**: `antecedents` and `outlist` are JSONB arrays; lookups use containment (`@>`) and `jsonb_array_elements_text(...)` for unnesting. GIN indexes make these cheap.
- **BFS worklist** for propagation: `deque`-based; each iteration processes a batch of changed nodes and only re-evaluates their direct dependents, so the cost is proportional to the affected subgraph, not the whole network.
- **Deferred (circular-safe) imports**: `from reasons.api import _STOP_WORDS`, `from .llm import invoke_model`, `from .check_stale import hash_file`, `from .import_beliefs import parse_beliefs`, `from .metadata import SCHEMA_VERSION`, `from .import_agent import _topo_sort_claims` are all imported **inside functions** to avoid an import cycle with `reasons/api.py`.
- **Rollback-by-finally in what-if**: `what_if_retract` and `what_if_assert` mutate the DB in a `try:` block and unconditionally `self.conn.rollback()` in `finally:` — dry-run semantics.
- **Metadata as an extension point**: `_retracted`, `access_tags`, `agent`, `original_id`, `imported_from`, `summarized_by`, `summarizes`, `supersedes`/`superseded_by`, `challenges`, `defense_target`, `retract_reason`, `stale_reason` all live in a single JSONB column. This is how the schema stays narrow while the domain grows.
- **Idempotent migrations** in `init_db`: probe `information_schema.columns` and `ALTER TABLE ... ADD COLUMN` if missing.
- **Optimistic re-evaluation**: most mutation paths compute an "old truth," apply the change, recompute, and only run propagation if `old_tv != new_tv`.
- **Namespacing as prefix**: agent beliefs are stored as `agent_name:belief_id`, with `agent_name:active` (premise) and `agent_name:inactive` (derived via `SL` with `outlist=[active]`) acting as a global on/off switch for the whole namespace.

---

## 4. Dependencies

### Imports (from this file)

- `psycopg` / `psycopg.rows.dict_row` — optional, guarded.
- `.metadata.SCHEMA_VERSION`, `.metadata.build_meta` — used in `init_db` and `export_network`.
- `.check_stale.hash_file`, `.check_stale.resolve_source_path` — used by `hash_sources` and `check_stale`.
- `reasons.api._STOP_WORDS`, `reasons.api._TOPIC_STOP_WORDS`, `reasons.api.NEGATIVE_TERMS`, `reasons.api.NEGATIVE_CLASSIFY_PROMPT`, `reasons.api.NEGATIVE_BATCH_SIZE` — used by `lookup`, `topics`, and `list_negative`.
- `.llm.invoke_model` — used by `list_negative` for LLM-based classification.
- `.import_beliefs.parse_beliefs / parse_nogoods / strip_frontmatter` — used by `import_beliefs`.
- `.import_agent._topo_sort_claims` — used by `import_agent` and `sync_agent`.

The `.metadata` and `.check_stale` imports are top-level, but the rest are intentionally deferred inside methods to prevent a circular import with `reasons/api.py` (which depends on `pg` when `--backend pg` is selected).

### Imported by

- `reasons/api.py` — selects the backend and forwards most calls to `PgApi` when running against PostgreSQL.
- `reasons_lib/pg.py` — a **duplicate copy** of this file (vendored package layout). Any change here should be mirrored there.
- `tests/conftest.py` — fixtures to wire up a test database.
- `tests/test_pg.py` — behavior tests for this backend; `tests/test_pg_dispatch.py` exercises backend selection from the CLI/API side.

---

## 5. Flow

### The canonical mutation — `add_node`

1. Build `metadata` from optional `access_tags` and `example`.
2. If `namespace` and node has no `:`, prefix ID and call `ensure_namespace` (which creates `namespace:active` as an `IN` premise if absent).
3. Open a cursor, verify the ID doesn't already exist (`ValueError` if it does).
4. Parse `sl`/`cp`/`unless`/`label` into justification dicts via `_parse_justifications`.
5. If namespaced, prefix antecedents/outlist and add `namespace:active` to each antecedent list (or synthesize a default `SL` justification from `[active]`).
6. Validate refs (`_validate_refs`) — all referenced nodes must exist.
7. `_add_node_raw`: `INSERT` node + justifications, compute truth via `_compute_truth`, `UPDATE` truth, log `add`.
8. `_inherit_access_tags` from the antecedents' metadata.
9. Count remaining premises for the response (`premise_count`).
10. `self.conn.commit()` and return the response dict.

### Propagation — `_propagate`

Worklist algorithm:

```
queue = [changed_id]; visited = {changed_id}
while queue nonempty:
    batch = drain queue
    dependents = _find_dependents(batch) - visited
    if empty: continue
    fetch states of dependents, justifications of dependents,
          truth values of ALL referenced nodes (batched ANY(...))
    for each dependent (skip if _retracted):
        new_tv = "IN" if any justification satisfies
                 (all antecedents IN and all outlist OUT) else "OUT"
        if new_tv != old_tv:
            UPDATE rms_nodes; log propagate
            truth_cache[dep_id] = new_tv
            visited.add(dep_id); queue.append(dep_id)
```

Returns `(went_out, went_in)` lists used by callers. Convergence is guaranteed because a node is enqueued at most once per change and truth values come from a finite set `{IN, OUT}`.

### `propagate()` (full fixpoint)

For callers who don't know what changed (e.g., after `import_json`/`import_agent`), it loops over **all** derived nodes, recomputes, and stops when a pass makes no changes. Bounded by `node_count + 1` iterations; skips `_retracted` nodes.

### `add_nogood` — contradiction resolution

1. Verify every node exists in the domain.
2. Generate content-addressed ID `nogood-<sorted-ids>` (with `-2`, `-3` suffix on collision).
3. Insert into `rms_nogoods`; before this point it's a no-op unless all members are `IN`.
4. If not all `IN`, commit and return `{"changed": [], "backtracked_to": None}` — the conflict is dormant.
5. Otherwise pick a victim: `_find_culprits_internal` returns premises sorted by entrenchment; choose `culprits[0].premise`. Fallback (no culprits): retract the nogood member with the fewest dependents.
6. Set victim's `metadata._retracted = True` and `truth_value = 'OUT'`, log `backtrack` + `retract`.
7. `_propagate(victim)`; return `{"nogood_id", "changed", "backtracked_to": victim}`.

### `sync_agent` — remote-wins import

1. Ensure `agent:active` and `agent:inactive` exist.
2. Compute `remote_ids = {prefix+c['id']}` and `local_agent_ids` = all nodes with the prefix except infrastructure.
3. For each remote claim (topologically sorted): if local, diff and `UPDATE` text/source/source_hash/date/metadata/justifications; else `_add_node_raw`.
4. Added claims with `is_out` and no justifications are queued for retraction; those with justifications get `_retracted=True`.
5. Commit, then retract `local_agent_ids - remote_ids` (removed remotely).
6. Commit, `propagate()`, then handle deferred retractions, `propagate()` again.
7. Optionally import agent-prefixed nogoods.

### `import_json` — order-sensitive bulk load

Uses `_add_node_raw` because references may point to nodes that haven't been added yet. It topologically sorts by dependencies **in the input file** (not by the DB), adds nodes in waves, and, once all nodes are added, calls `propagate()` to fix truth values, then **rewrites** truth values and timestamps from the imported data. This is the flow that makes `_add_node_raw`'s lack of validation ok — validation happens implicitly through `_compute_truth` defaulting missing refs to `OUT`, then `propagate` recomputing.

---

## 6. Invariants

1. **Tenant isolation**: every query filters by `self.domain_id`. `domain_id` appears in `WHERE`, `PRIMARY KEY`, and `FOREIGN KEY` clauses. There is no method that returns cross-domain data.
2. **Schema-level enums**: `truth_value ∈ {'IN','OUT'}` and justification `type ∈ {'SL','CP'}` are enforced by `CHECK` constraints, not just Python.
3. **Primary key**: `(id, domain_id)` on nodes and nogoods. Two domains can share node IDs. `add_node` explicitly raises `ValueError` on collision.
4. **Premises are IN**: `_compute_truth` returns `IN` for a node with no justifications; the only way to make a premise `OUT` is `retract_node` or `convert_to_premise` (which can't change an existing premise since `old_value == "IN"` already).
5. **`_retracted` metadata** is authoritative: `_propagate` and `propagate` skip nodes with it set, even if their justifications would otherwise make them `IN`. `assert_node` and `convert_to_premise` are the ways to clear it.
6. **Cannot remove the last justification**: `remove_justification` raises `ValueError` (suggesting `convert-to-premise` or `retract`).
7. **Access tag union**: if a node's antecedents carry `access_tags`, the dependent inherits the **union**. `_is_visible` returns `True` only if *every* tag on the node is present in `visible_to`. So a derived node is at least as restricted as the most restricted node in any of its justifications.
8. **Referenced nodes must exist** at the time of `add_node` / `add_justification`. `_add_node_raw` alone does not enforce this (imports rely on `propagate` to fix it up).
9. **Imports preserve exact timestamps**: after all nodes are added and truth values propagated, `import_json` rewrites `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at` from the source data.
10. **Sync is remote-wins**: `sync_agent` overwrites local fields with the remote claim values; local-only nodes are retracted.
11. **What-if never commits**: both `what_if_*` methods call `self.conn.rollback()` in a `finally`; they must not leave a transaction open or commit.
12. **`import_query` / search use parameterized queries**: `plainto_tsquery('english', %s)` and `id ILIKE %s` prevent injection even for `search()` with arbitrary user input.

---

## 7. Error Handling

Errors are raised as ordinary Python exceptions; there is no `try/except` that swallows errors in the public API, with one important exception:

- **`what_if_*` (`IndexError`, `KeyError` also flow out, but `finally` guarantees rollback)**: these are the *only* methods that use `try/finally` to guarantee clean transaction state even on error.

Error types used deliberately:

| Exception | Raised when |
|---|---|
| `ImportError` | `psycopg` not installed (`_require_psycopg`). Message includes the exact `pip install` hint. |
| `ValueError` | Node already exists (`add_node`), no justification specified (`add_justification`), only one justification remains (`remove_justification`), challenge/defense ID already exists (`_challenge_internal`, `defend`), summary already exists (`summarize`). |
| `KeyError` | Node not found (`add_justification`, `retract_node`, `assert_node`, `remove_justification`, `show_node`, `explain_node`, `trace_*`, `supersede`, `summarize` for missing `over` nodes), referenced nodes missing (`_validate_refs`), challenge/defense targets missing (`defend`), nogood members missing (`add_nogood`). |
| `IndexError` | `remove_justification` index out of range. |
| `PermissionError` | `show_node` / `trace_access_tags` when `_is_visible` fails. |

Deliberate **silence**:

- `_compute_truth` returns `"OUT"` when a referenced node is missing (via `truth_cache.get(a, "OUT")`). This is what makes `_add_node_raw` safe for imports.
- `propagate()` skips nodes with `metadata._retracted`.
- `search` returns `"No results found."` (or `{"results": [], "count": 0}` for `format="dict"`) instead of raising.
- `lookup` returns a string `"No beliefs found matching '...'"` on miss.

**Transaction-safety caveat**: most public methods call `self.conn.commit()` at the end and do *not* wrap themselves in a try/except. If an exception is raised after some SQL is issued but before the commit, the transaction remains uncommitted with pending statements. The intended safe usage is

```python
with PgApi(conninfo, domain_id) as api:
    api.add_node(...)
    api.add_justification(...)
```

because `__exit__` calls `self.conn.rollback()` when an exception propagates out of the `with`. Direct callers that don't use `with` are responsible for rolling back. Individual **successful** method calls commit independently, so multi-call invariants (e.g., "add A and add B atomically") are *not* guaranteed — each is its own transaction.

---

## Topics to Explore

- [file] `reasons/api.py` — The in-memory implementation of the same logical API. Reading it side-by-side with `pg.py` is the fastest way to understand what each SQL query reproduces from Python.
- [function] `reasons/pg.py:_propagate` — Core defeasible-logic engine; understanding the batch/`truth_cache` strategy is essential before changing any mutation.
- [file] `tests/test_pg.py` — Pins down the SQL backend's behavior (see also `tests/test_pg_dispatch.py` for backend selection).
- [general] `dependency-directed-backtracking` — Explains `_find_culprits_internal` + `_entrenchment`; the retraction policy is a product decision, not just an implementation detail.
- [file] `reasons_lib/pg.py` — Vendored duplicate of this module; changes here must be mirrored there or the published package drifts.

## Beliefs

- `pg-requires-optional-psycopg` — `psycopg` is imported in a `try/except`; `_require_psycopg` raises `ImportError` with install instructions when it's missing.
- `domain-id-enables-multi-tenancy` — every SQL statement filters by `self.domain_id`, and `(id, domain_id)` is the nodes/nogoods primary key, so tenant data is isolated at the schema level.
- `_propagate-implements-bfs-worklist` — `_propagate` uses a `deque`-based worklist over dependents and enqueues only nodes whose truth value actually changed; returned lists partition changes into `went_out` and `went_in`.
- `_retracted-metadata-prevents-in-propagation` — `_propagate` and `propagate` skip any node whose `metadata._retracted` is true; `assert_node`/`convert_to_premise` are the ways to clear it.
- `access-tags-inherit-by-union` — `_inherit_access_tags` unions antecedent `access_tags` into the dependent node, and `_is_visible` requires *all* of a node's tags to be in `visible_to`.
- `what-if-rollbacks-are-unconditional` — `what_if_retract` and `what_if_assert` both call `self.conn.rollback()` in a `finally`, so they can never persist a state change.
- `sync-agent-is-remote-wins` — `sync_agent` overwrites local text/source/justifications with remote values and retracts any local agent belief absent from the remote set.
- `single-justification-removal-is-blocked` — `remove_justification` raises `ValueError` when a node would be left with zero justifications; `convert_to_premise` is the sanctioned alternative.
