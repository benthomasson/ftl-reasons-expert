---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/pg.py

# `reasons/pg.py` — PostgreSQL-Native Storage Backend

## 1. Purpose

This file implements a **PostgreSQL-backed storage engine** for the Tarski-style dependency (belief) network used across the `ftl-reasons` project. The core idea: instead of loading the entire in-memory `Network` object, mutating it in Python, and serializing it back (as the in-memory `storage.py` does), every operation here is a **self-contained SQL transaction** against a relational schema.

The practical consequence is twofold:

- **Concurrency** — multiple writers can operate on the same domain simultaneously because PostgreSQL handles row-level locking.
- **Multi-tenancy** — every table is partitioned by `domain_id` (a UUID), so one database instance serves many isolated belief networks.

The file is imported by `reasons/api.py` (the public API facade) and by the test suite (`tests/test_pg.py`, `tests/test_pg_dispatch.py`, `tests/conftest.py`). There is a parallel copy in `reasons_lib/pg.py` that appears to be a vendored/legacy variant.

---

## 2. Key Components

### Schema Constants

| Constant | Contents |
|---|---|
| `SCHEMA` | DDL for 5 tables: `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, `rms_network_meta` |
| `INDEXES` | B-tree + GIN indexes. Notably `idx_rms_nodes_fts` (full-text search) and GIN indexes on the JSONB `antecedents`/`outlist` arrays for fast containment queries. |

The primary key on `rms_nodes` is the composite `(id, domain_id)` — node IDs are only unique *within* a domain.

### `PgApi` class

The single public class. Its constructor accepts either a connection-string (it opens and owns the connection) or an existing `psycopg.Connection` (borrowed). It supports the context-manager protocol (`__enter__`/`__exit__`) which rolls back on exception.

#### Public method groups

**Lifecycle / Schema**
- `init_db(project_name)` — runs `SCHEMA` + `INDEXES`, performs incremental migrations (checks `information_schema.columns` and `ALTER TABLE ADD COLUMN` for missing columns), seeds `rms_network_meta` with `schema_version`, `project_name`, timestamps.

**Core mutations** (each is a single transaction)
- `add_node(node_id, text, sl, cp, unless, …)` — inserts a node + justifications, computes initial truth value, inherits access tags, returns a dict with `truth_value` and `premise_count`.
- `add_justification(node_id, sl, cp, unless, …)` — appends a justification to an existing node, re-evaluates truth, propagates if it flipped.
- `retract_node(node_id, reason)` / `assert_node(node_id)` — flip truth value, stamp `retracted_at`, propagate cascading changes.
- `remove_justification(node_id, index)` — delete one justification by positional index, re-evaluate.
- `convert_to_premise(node_id)` — delete all justifications, force IN.
- `supersede(old_id, new_id)` — add `new_id` to every outlist on `old_id`, stamp `superseded_by` / `supersedes` metadata, propagate.
- `summarize(summary_id, text, over)` — create a summary node with an SL justification over the `over` set; back-links `summarized_by` on each covered node.

**What-if simulation**
- `what_if_retract(node_id)` / `what_if_assert(node_id)` — performs the mutation + propagation inside a transaction, then **rolls back** in `finally`. Returns a structured diff (`retracted` / `restored` lists with depth and dependent-count). This is the "dry-run" path.

**Dialectical operations**
- `challenge(target_id, reason)` — creates a premise node that enters the target's outlist, effectively defeating it.
- `defend(target_id, challenge_id, reason)` — creates a counter-challenge against the challenge node, with `defense_target` / `defends` metadata.

**Read / query**
- `get_status()` — all nodes with justification counts.
- `show_node(node_id)` — full node detail + justifications + dependents.
- `search(query, …)` — PostgreSQL full-text search (`plainto_tsquery`) with `ts_rank` ordering, plus `ILIKE` fallback. Expands to 1-hop neighbors.
- `lookup(query)` — in-Python all-terms substring search with progressive relaxation (drops terms via `itertools.combinations`, capped at 50 combinations).
- `list_nodes(status, premises_only, has_dependents, namespace, label, …)` — parameterized filter with dynamic `WHERE` clause construction.
- `list_gated()` — finds OUT nodes whose outlist contains IN nodes (i.e., "blockers").
- `list_negative()` — keyword-filter + optional LLM classification of IN beliefs describing problems.
- `topics(limit)` — tokenizes node IDs, counts word frequencies, returns top-N.
- `export_network()` — serializes the full domain to a dict matching the JSON export format.
- `get_log(last)` — reads the audit trail from `rms_propagation_log`.
- `compact(budget, …)` — renders a token-budgeted markdown summary of the belief state.

**Source tracking**
- `hash_sources(force, repos)` — backfills `source_hash` by hashing files on disk.
- `check_stale(repos, …)` — compares stored hashes to current file hashes, classifies as `content_changed`, `truncated_hash`, or `source_deleted`.
- `add_repo(name, path)` / `list_repos()` — persist repo name→path mappings in `rms_network_meta`.

**Import / sync**
- `import_json(data)` — topological-order import from the JSON export format; propagates; then applies truth-value overrides and restores timestamps.
- `import_beliefs(beliefs_text, nogoods_text)` — parses markdown, topologically sorts claims, inserts, propagates, retracts STALE/OUT claims, imports nogoods.
- `import_agent(agent_name, claims, nogoods)` — namespaced import with an `active`/`inactive` kill-switch pair.
- `sync_agent(agent_name, claims, nogoods)` — remote-wins upsert: add/update/delete/retract to converge local to remote state.

**Namespaces**
- `ensure_namespace(ns)` — creates the `{ns}:active` premise node.
- `list_namespaces()` — enumerates `*:active` nodes with belief counts.

**Nogoods / backtracking**
- `add_nogood(node_ids)` — records a contradiction set; if all members are IN, runs dependency-directed backtracking (`_find_culprits_internal`) to pick a victim premise to retract.
- `find_culprits(node_ids)` — read-only culprit analysis.
- `explain_node(node_id)` — recursive justification tree explanation.
- `trace_assumptions(node_id)` — walks antecedents to leaf premises.
- `trace_access_tags(node_id)` — unions `access_tags` across the dependency chain.

#### Internal / helper methods

| Method | Role |
|---|---|
| `_propagate(cur, changed_id)` | **BFS** over dependents; batch-fetches states, re-evaluates each dependent's justifications, enqueues newly-changed nodes. Returns `(went_out, went_in)`. |
| `_compute_truth(cur, node_id)` | Evaluates all justifications: IN if *any* justification has all antecedents IN and all outlist entries OUT. Premise (no justifications) → IN. |
| `_find_dependents(cur, node_ids)` | JSONB containment query (`antecedents @> '["id"]'` OR `outlist @> '["id"]'`) to find nodes referencing the given IDs. |
| `_validate_refs(cur, justs)` | Pre-insert check that all referenced node IDs exist in the domain. |
| `_parse_justifications(sl, cp, unless, label)` | Parses comma-separated CLI strings into justification dicts. |
| `_inherit_access_tags(cur, node_id, justs)` | Unions `access_tags` from all antecedent nodes into the new node. |
| `_is_visible(metadata, visible_to)` | Access-control check: all tags in `metadata.access_tags` must be a subset of `visible_to`. |
| `_find_culprits_internal(cur, nogood_ids)` | Traces premises for each IN member, scores candidates by `_entrenchment`, returns sorted list. |
| `_entrenchment(cur, node_id)` | Scoring heuristic: +100 if premise, +50 if has source, +25 if has hash, +10×dependents, +type-based bonus. |
| `_challenge_internal(cur, …)` | Shared logic for `challenge` and `defend`. |
| `_explain_recursive(cur, …)` | DFS explanation of why a node is IN or OUT. |
| `_collect_what_if_results` / `_cascade_depth_pg` | Post-propagation diff and BFS depth for what-if. |
| `_expand_neighbors` / `_format_*` | Search-result enrichment and rendering. |
| `_log(cur, action, target, value)` | Audit-trail insert. |
| `_load_repos` / `_import_agent_nogoods` | Utility glue. |

---

## 3. Patterns & Idioms

- **Repository pattern**: `PgApi` is a repository that encapsulates all persistence. The caller (e.g., `api.py`) never touches SQL directly.
- **Transaction-per-operation**: Every public mutation method opens a `cursor` context, does its work, and calls `self.conn.commit()` at the end. The `autocommit=False` setting on the connection makes each `BEGIN…COMMIT` atomic.
- **Context manager with rollback-on-exception**: `__exit__` rolls back if an exception propagated; `close()` is called either way. This is the standard psycopg v3 idiom.
- **BFS propagation with batching**: `_propagate` drains the queue in batches, does a single `SELECT … WHERE id = ANY(…)` per batch, and a single `UPDATE` per changed node. This avoids N+1 queries.
- **JSONB as a flexible column**: `metadata`, `antecedents`, `outlist` are all JSONB. The code consistently handles the case where psycopg returns them as `str` (defensive `isinstance(x, str): json.loads(x)`) — this matters because the return type depends on the `row_factory` in use.
- **Topological sort for imports**: Both `import_json` and `import_beliefs` do a multi-pass "ready-set" loop (essentially Kahn's algorithm) to insert nodes in dependency order, with a fallback pass for cycles.
- **What-if via rollback**: `what_if_retract`/`what_if_assert` do the full mutation + propagation, collect the diff, then `self.conn.rollback()` in `finally`. The database state is unchanged; the caller gets a structured prediction.
- **Dynamic WHERE construction**: `list_nodes` builds its `WHERE` clause by appending condition strings and params lists, parameterized throughout.
- **Optional dependency guard**: `psycopg` import is wrapped in `try/except ImportError`; `_require_psycopg()` raises a clear `ImportError` at construction time if the driver is missing.

---

## 4. Dependencies

### Imports (what this file needs)

| Dependency | Usage |
|---|---|
| `psycopg` (v3) | All SQL execution. Hard requirement, gated by `_require_psycopg()`. |
| `psycopg.rows.dict_row` | Imported but the code actually uses positional row access (`row[0]`, `row[1]`), so `dict_row` is imported but **not actively used** in this file. |
| `json`, `re`, `collections.deque`, `datetime`, `itertools.combinations` | Standard library. |
| `reasons.metadata` (`SCHEMA_VERSION`, `build_meta`) | Schema version stamping, meta-block construction for export. |
| `reasons.check_stale` (`hash_file`, `resolve_source_path`) | Source-file hashing and path resolution. |
| `reasons.api` (`_STOP_WORDS`, `NEGATIVE_TERMS`, `NEGATIVE_CLASSIFY_PROMPT`, `NEGATIVE_BATCH_SIZE`, `_TOPIC_STOP_WORDS`) | Search and classification heuristics. |
| `reasons.llm` (`invoke_model`) | LLM-based negative-belief classification. |
| `reasons.import_beliefs` (`parse_beliefs`, `parse_nogoods`, `strip_frontmatter`) | Markdown belief parsing. |
| `reasons.import_agent` (`_topo_sort_claims`) | Topological sort for agent imports. |
| `pathlib.Path` | File-path operations in `hash_sources` / `check_stale`. |

### Importers (what depends on this file)

- `reasons/api.py` — the public API dispatches to `PgApi` when a PostgreSQL backend is selected.
- `reasons_lib/pg.py` — a parallel copy (likely vendored or pre-refactor).
- `tests/conftest.py` — test fixtures create `PgApi` instances.
- `tests/test_pg.py`, `tests/test_pg_dispatch.py` — dedicated test suites.

---

## 5. Flow (Control & Data)

### Typical mutation flow (e.g., `add_justification`)

```
caller
  │
  ▼
PgApi.add_justification(node_id, sl, cp, unless, label)
  │
  ├── parse sl/cp/unless → justification dicts          (_parse_justifications)
  ├── namespace-prefix antecedents & outlist           (if namespace given)
  │
  ▼  [transaction begins — cursor context]
  ├── SELECT truth_value FROM rms_nodes (existence + old state)
  ├── _validate_refs  (all antecedents/outlist IDs must exist)
  ├── INSERT into rms_justifications (one row per justification)
  ├── _inherit_access_tags  (union tags from antecedents)
  ├── _compute_truth        (evaluate all justifications → IN or OUT)
  │     ├── SELECT antecedents/outlist from rms_justifications
  │     ├── SELECT truth_value for all referenced nodes  (batched)
  │     └── any-justification-satisfied? → IN : OUT
  │
  ├── IF truth changed:
  │     ├── UPDATE rms_nodes SET truth_value
  │     └── _propagate(cur, node_id)
  │           ├── BFS: drain queue in batches
  │           │     ├── _find_dependents  (JSONB containment)
  │           │     ├── SELECT state + justifications for batch
  │           │     ├── SELECT truth for all referenced nodes
  │           │     ├── Re-evaluate each dependent
  │           │     └── UPDATE + enqueue newly-changed
  │           └── return (went_out, went_in)
  │
  ├── _log (audit trail insert)
  ▼  [commit]
  │
  ▼  return {node_id, old_truth_value, new_truth_value, changed[]}
```

### What-if flow (`what_if_retract`)

Identical to the mutation flow above, except:
1. A `before` snapshot is taken (`SELECT id, truth_value FROM rms_nodes WHERE domain_id = …`).
2. After propagation, `_collect_what_if_results` diffs `before` vs. current DB state (which is still in the open transaction) to produce `retracted` / `restored` lists with depth.
3. `finally: self.conn.rollback()` undoes everything.

### Import flow (`import_json`)

1. Import meta (schema_version, project_name, created_at).
2. **Multi-pass topological insert**: repeatedly scan `remaining` nodes; a node is insertable when all its in-data dependencies are already in `added`. If a full pass makes no progress (cycle), force-insert the rest.
3. `propagate()` — full fixpoint recompute.
4. Apply truth-value overrides (retract/assert nodes that should be OUT/IN per the export).
5. Restore timestamps (`updated_at`, `reviewed_at`, `verified_at`, `retracted_at`).
6. Import nogoods.
7. Commit.

---

## 6. Invariants

1. **Multi-tenancy isolation**: Every `WHERE` clause includes `domain_id = %s`. No query crosses domain boundaries. The composite PK `(id, domain_id)` enforces this at the schema level.

2. **Truth-value semantics**:
   - A node with **no justifications** is a **premise** and is always IN (unless explicitly retracted via metadata `_retracted`).
   - A node with justifications is IN iff **at least one** justification is fully satisfied (all antecedents IN, all outlist entries OUT).
   - Retracted nodes (`_retracted: true` in metadata) are **skipped during propagation** — they stay OUT regardless of their justifications.

3. **Justification types**: Only `'SL'` (strictly logical) and `'CP'` (closed premise) are allowed, enforced by a `CHECK` constraint in the schema.

4. **Referential integrity**: `_validate_refs` is called before any justification insert. The `FOREIGN KEY (node_id, domain_id) REFERENCES rms_nodes(id, domain_id) ON DELETE CASCADE` on `rms_justifications` ensures orphan justifications are impossible.

5. **Namespace kill-switch**: Every namespaced node's justification outlist implicitly includes `{ns}:inactive`. The `inactive` node is IN when `active` is OUT (via an SL justification with `outlist: [active_id]`), creating a mutual-exclusion gate.

6. **Nogood IDs are content-derived**: `nogood-{sorted-node-ids-slug}`. Collisions are resolved with a `-2`, `-3` suffix.

7. **Access tags are inherited**: A derived node's `access_tags` is the union of its own tags and all antecedent tags. Visibility checks require *all* tags on a node to be a subset of the viewer's `visible_to` set.

8. **`propagate()` fixpoint**: The full `propagate()` method iterates up to `node_count + 1` passes, recomputing every derived node each pass, until no node changes. This handles cascading effects that the incremental BFS might miss after bulk imports.

---

## 7. Error Handling

| Situation | Mechanism |
|---|---|
| `psycopg` not installed | `_require_psycopg()` raises `ImportError` with install instructions at `__init__` time. |
| Node not found | `KeyError(f"Node '{node_id}' not found")` — raised in `retract_node`, `assert_node`, `show_node`, `add_justification`, etc. |
| Duplicate node on add | `ValueError(f"Node '{node_id}' already exists")` in `add_node`. |
| Referenced node missing | `KeyError(f"Referenced nodes do not exist: …")` in `_validate_refs`. |
| Justification index out of range | `IndexError` in `remove_justification`. |
| Removing last justification | `ValueError` in `remove_justification` — directs to `convert_to_premise` or `retract`. |
| Access denied | `PermissionError` in `show_node` and `trace_access_tags`. |
| Unknown truth value in data | Silently handled — the `truth_value` column has a `CHECK` constraint; bad values would be a DB error. |
| LLM classification parse failure in `list_negative` | `json.JSONDecodeError` is caught and the match is skipped; the next `[...]` bracket group is tried. |
| Exception during a transaction | `__exit__` calls `self.conn.rollback()`. For what-if methods, `finally: self.conn.rollback()` is explicit. |
| File not found during `check_stale` | `resolve_source_path` returns `None`; the node is recorded as `source_deleted` in results (not an exception). |

Notable: there is **no retry logic** and **no circuit breaker**. The code assumes a healthy PostgreSQL connection. If `psycopg.OperationalError` (e.g., connection lost) is raised, it propagates to the caller.

---

## Topics to Explore

- [file] `reasons/api.py` — The dispatch layer that decides whether to use `PgApi` or the in-memory storage, and how it maps public API calls onto `PgApi` methods. Understanding the dispatch logic clarifies which methods are actually exercised in production vs. which are test-only.
- [function] `reasons/pg.py:_propagate` — The BFS propagation engine is the most performance-sensitive code path. Its batched-query strategy, the `truth_cache` dict, and the interaction with `_retracted` metadata determine correctness under concurrent writes.
- [file] `reasons/check_stale.py` — Implements `hash_file` and `resolve_source_path`, which `pg.py` delegates to for source tracking. Understanding the path-resolution strategy (agent-specific repos, relative vs. absolute paths) is essential for debugging staleness reports.
- [file] `tests/test_pg.py` — The test suite for this module. It reveals the expected contracts for every public method, edge cases (empty networks, circular dependencies, multi-tenant isolation), and how the test fixture provisions a PostgreSQL instance.
- [general] `tarski-aba-semantics` — The justification model (SL/CP types, antecedents as inlist, outlist as defeaters, premises as nodes with no justifications) implements a specific variant of Assumption-Based Argumentation. Understanding this formal foundation makes the `_compute_truth`, `_propagate`, and nogood/backtracking logic far more intuitive.

---

## Beliefs

- `pg-api-requires-psycopg-v3` — `PgApi.__init__` raises `ImportError` if `psycopg` (v3, with `[binary]` extra) is not installed; the module is fully non-functional without it.
- `domain-id-partitions-all-tables` — Every table uses a composite primary key or filter on `domain_id`; no SQL query in `PgApi` omits the `domain_id` predicate, making cross-domain data leakage structurally impossible.
- `retracted-nodes-skip-propagation` — Nodes with `metadata._retracted == True` are excluded from `_propagate` and `propagate()` recomputation; their truth value is pinned to OUT regardless of justification satisfaction.
- `what-if-operations-always-rollback` — `what_if_retract` and `what_if_assert` guarantee zero persistent side effects by calling `self.conn.rollback()` in a `finally` block, making them safe to call from any context including read-only connections.
- `import-uses-topological-sort-with-cycle-fallback` — Both `import_json` and `import_beliefs` perform a multi-pass ready-set insertion (Kahn's algorithm); if a full pass makes zero progress (indicating a dependency cycle), all remaining nodes are force-inserted in the next iteration to prevent infinite loops.
