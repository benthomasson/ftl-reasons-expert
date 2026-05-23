# File: reasons_lib/pg.py

**Date:** 2026-05-11
**Time:** 12:49

## `reasons_lib/pg.py` — PostgreSQL-Native Storage Backend

### 1. Purpose

This file is the PostgreSQL backend for the `ftl-reasons` Truth Maintenance System (TMS). While the project also has a SQLite-based `storage.py`, `pg.py` re-implements the entire belief network API directly against PostgreSQL using transactional SQL. Its key value proposition is stated in the module docstring: **each operation is a SQL transaction — no full-network load/save**, enabling concurrent writers and multi-tenant deployment via a `project_id` UUID partition key.

It is the persistence layer behind the `reasons` CLI and the MCP server when configured for PostgreSQL, and it mirrors the interface of `api.py` (which wraps the in-memory `Network` + SQLite store).

### 2. Key Components

#### Schema (lines ~25–70)

Five tables, all partitioned by `(id, project_id)` composite primary keys:

| Table | Role |
|---|---|
| `rms_nodes` | Belief nodes with text, truth value (IN/OUT), source provenance, and JSONB metadata |
| `rms_justifications` | SL/CP justifications linking a node to its antecedents and outlist |
| `rms_nogoods` | Recorded contradictions (sets of mutually inconsistent nodes) |
| `rms_propagation_log` | Append-only audit log of all mutations |
| `rms_network_meta` | Key-value store for per-project config (repo paths, nogood counter) |

Indexes include GIN indexes on `antecedents` and `outlist` JSONB columns (for `@>` containment queries in `_find_dependents`) and a full-text search index on `rms_nodes.text`.

#### `PgApi` class

The single public class. Constructor takes either a connection string or an existing `psycopg` connection, plus a `project_id`. Implements context manager protocol with rollback-on-exception.

**Core mutations:**

- **`add_node`** — Creates a node with optional SL/CP justifications. Validates that referenced antecedents/outlist nodes exist (`_validate_refs`). Supports namespace prefixing and access tag inheritance. Computes initial truth value.
- **`add_justification`** — Adds a justification to an existing node and propagates truth changes.
- **`retract_node`** — Sets a node OUT by marking `_retracted` in metadata, then propagates the cascade.
- **`assert_node`** — Reverses a retraction (sets IN, removes `_retracted` flag), then propagates.
- **`supersede`** — Adds `new_id` to `old_id`'s outlist so the old belief goes OUT when the new one is IN. Records `superseded_by`/`supersedes` in metadata.

**Read operations:**

- **`get_status`** — Lists all nodes with truth values and justification counts.
- **`show_node`** — Full detail for one node including justifications, dependents, and metadata.
- **`search`** — PostgreSQL full-text search (`plainto_tsquery`) with 1-hop neighbor expansion.
- **`lookup`** — Simpler all-terms substring search over the entire belief block (ID + text + source + refs).
- **`explain_node`** — Recursive walk producing a justification trace (why is this IN or OUT?).
- **`trace_assumptions`** — Follows justification chains to find all root premises.
- **`list_gated`** — Finds OUT nodes blocked by IN outlist nodes (GATE beliefs).

**What-if simulation:**

- **`what_if_retract` / `what_if_assert`** — Performs the mutation inside a transaction, collects cascade results, then **rolls back**. Returns what would change without actually changing it.

**Dialectical operations:**

- **`challenge`** — Creates a new premise node and adds it to the target's outlist, forcing the target OUT.
- **`defend`** — Challenges a challenge (meta-level dialectical move).

**Import operations:**

- **`import_json`** — Imports from the `network.json` export format using topological sort for dependency ordering.
- **`import_beliefs`** — Imports from markdown-format belief files.
- **`import_agent` / `sync_agent`** — Multi-agent federation: imports another agent's beliefs with namespace prefixing (`agent_name:belief_id`), creates `active`/`inactive` kill-switch nodes.

**Maintenance:**

- **`hash_sources` / `check_stale`** — Source file staleness detection by comparing stored hashes against current file content.
- **`propagate`** — Full-network fixpoint recomputation of truth values.
- **`compact`** — Generates a token-budgeted summary of the belief state (for LLM context windows).

### 3. Patterns

**Transaction-per-operation:** Every public method opens a cursor, does work, and calls `self.conn.commit()`. Internal helpers like `_add_node_raw`, `_propagate`, and `_compute_truth` operate on a passed cursor without committing — the caller owns the transaction boundary.

**BFS propagation (`_propagate`):** When a node's truth value changes, dependents are found via `_find_dependents` (JSONB `@>` containment), truth is recomputed per-node, and changed nodes are enqueued for further propagation. This is the core TMS algorithm — Doyle's dependency-directed backtracking.

**Batch-fetch then evaluate:** `_propagate` and `_compute_truth` collect all referenced node IDs, batch-fetch their truth values in one query, then evaluate locally. This avoids N+1 query patterns.

**Rollback-based simulation:** `what_if_retract`/`what_if_assert` use `try/finally: self.conn.rollback()` to make mutations visible within the transaction for cascade computation, then discard everything.

**Namespace federation:** Multi-agent beliefs are prefixed (`agent:belief-id`) and gated behind an `agent:active` premise. Retracting `agent:active` cascades OUT to all that agent's beliefs via outlist mechanics. An `agent:inactive` kill-switch node uses the outlist `[agent:active]` so it flips IN when the agent is deactivated.

**Lazy imports:** Heavy dependencies (`check_stale`, `import_beliefs`, `import_agent`, `llm`) are imported inside the methods that use them, keeping the module importable without those packages installed.

**JSONB for variable-length relations:** Antecedents and outlist are stored as JSONB arrays rather than junction tables. This simplifies the schema but means dependent lookups require `@> '["id"]'::jsonb` containment checks (covered by GIN indexes).

### 4. Dependencies

**Imports:**
- `psycopg` (v3) — PostgreSQL driver, soft-imported with fallback to `None`
- `json`, `re`, `collections.deque`, `datetime` — stdlib
- `.check_stale` — `hash_file`, `resolve_source_path` (lazy, for staleness checking)
- `.api` — `NEGATIVE_TERMS`, `NEGATIVE_CLASSIFY_PROMPT`, `NEGATIVE_BATCH_SIZE` (lazy, for `list_negative`)
- `.llm` — `invoke_model` (lazy, for LLM-based classification)
- `.import_beliefs` — `parse_beliefs`, `parse_nogoods` (lazy)
- `.import_agent` — `_topo_sort_claims` (lazy)

**Imported by:**
- `reasons_lib/api.py` — dispatches to `PgApi` when a PostgreSQL connection is configured
- `tests/test_pg.py` — direct unit tests
- `tests/conftest.py` — test fixtures

### 5. Flow

A typical mutation flow (e.g., `retract_node`):

1. Open cursor, fetch current node state (truth value + metadata)
2. Update metadata (`_retracted = True`) and set `truth_value = 'OUT'`
3. Log the action to `rms_propagation_log`
4. Call `_propagate(cur, node_id)`:
   - BFS from the changed node through `_find_dependents`
   - For each dependent: recompute truth via `_compute_truth`, update if changed, enqueue dependents of changed nodes
   - Returns `(went_out, went_in)` lists
5. Commit the transaction
6. Return a result dict with all changed node IDs

For `import_json`/`import_agent`, the flow adds topological sorting: nodes are ordered so that a node's antecedents are inserted before the node itself. If a cycle or missing dep makes this impossible, a fallback pass inserts remaining nodes in arbitrary order.

### 6. Invariants

- **Composite key isolation:** Every query filters by `project_id`, ensuring complete tenant isolation within a shared database.
- **Premises are always IN** (unless explicitly retracted): `_compute_truth` returns `"IN"` when a node has no justifications.
- **Retraction is sticky:** The `_retracted` metadata flag persists across `propagate()` calls — `propagate` skips `_retracted` nodes, preventing truth recomputation from overriding an explicit retraction.
- **Referential integrity on add:** `_validate_refs` ensures all antecedent and outlist IDs exist before a node is created. Missing references raise `KeyError`.
- **No duplicate nodes:** `add_node` checks for existence and raises `ValueError` if the ID already exists.
- **Single-justification guard:** `remove_justification` refuses to remove the last justification (forces use of `convert_to_premise` or `retract` instead).
- **Access tag inheritance:** When a derived node is created, it inherits the union of access tags from its antecedents (`_inherit_access_tags`).

### 7. Error Handling

- **`ImportError`** raised by `_require_psycopg()` if `psycopg` is not installed — clear install instructions in the message.
- **`KeyError`** for missing nodes — consistent across `retract_node`, `assert_node`, `show_node`, `add_justification`, etc.
- **`ValueError`** for duplicate nodes, missing justification args, or invalid state transitions.
- **`IndexError`** for out-of-range justification index in `remove_justification`.
- **`PermissionError`** in `show_node` when `visible_to` tags don't match the node's `access_tags`.
- **Transaction safety:** The `__exit__` method calls `rollback()` on exception. `what_if_*` methods use `try/finally: rollback()` to guarantee no side effects.
- **No silent swallowing:** Errors propagate to the caller; the only "swallowed" case is `check_stale` skipping nodes whose source path can't be resolved (`path is None`).

---

## Topics to Explore

- [file] `reasons_lib/storage.py` — The SQLite storage backend; compare its API surface to understand what `PgApi` must mirror and where they diverge
- [file] `reasons_lib/api.py` — The dispatch layer that chooses between SQLite/in-memory `Network` and `PgApi` based on configuration; defines the constants `PgApi` imports lazily
- [function] `reasons_lib/network.py:Network` — The in-memory TMS implementation with Doyle's algorithm; `PgApi._propagate` is the SQL translation of this logic
- [file] `tests/test_pg.py` — Test coverage for the PostgreSQL backend; reveals which edge cases (cascades, what-if, agent import) are exercised
- [general] `outlist-dependent-tracking` — The known issue where outlist nodes are not tracked in the dependents index, causing GATE beliefs to require manual re-evaluation after outlist changes (documented in CLAUDE.md)

## Beliefs

- `pg-api-project-isolation` — Every `PgApi` query filters by `project_id`, so multiple tenants can share the same PostgreSQL database without cross-contamination
- `pg-retracted-flag-survives-propagation` — The `propagate()` method skips nodes with `metadata._retracted = True`, so an explicit retraction is never overridden by truth recomputation
- `pg-what-if-rolls-back` — `what_if_retract` and `what_if_assert` always execute `self.conn.rollback()` in a `finally` block, guaranteeing zero side effects regardless of exceptions
- `pg-find-dependents-uses-jsonb-containment` — `_find_dependents` queries `antecedents @> '["node_id"]'::jsonb` per node ID rather than using a junction table, relying on GIN indexes for performance
- `pg-namespace-kill-switch` — Agent federation creates an `agent:active` premise and an `agent:inactive` outlist-gated node; retracting `active` cascades OUT to all prefixed beliefs through their outlist dependency on `inactive`

