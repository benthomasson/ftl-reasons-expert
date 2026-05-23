# File: reasons_lib/api.py

**Date:** 2026-05-11
**Time:** 12:44

## `reasons_lib/api.py` — Functional API for the Reason Maintenance System

### 1. Purpose

This is the **stateless service layer** of the ftl-reasons TMS. It sits between consumers (CLI, LangGraph tools, MCP server, scripts) and the persistence/network layers, providing a uniform function-call interface where every operation follows the same lifecycle: open database, load network, operate, optionally save, close, return a JSON-serializable dict.

It owns the responsibility of being the **single entry point** for all belief-network mutations and queries. The CLI (`cli.py`) doesn't touch `Storage` or `Network` directly — it calls these functions.

### 2. Key Components

**Database lifecycle helper:**
- `_with_network(db_path, write=False)` — Context manager that loads a `Network` from `Storage`, yields it, and conditionally saves on clean exit. Every public function uses this rather than managing `Storage` directly.

**PostgreSQL dispatch:**
- `_pg_dispatch(pg_conninfo, project_id, method_name, **kwargs)` — Every public function accepts optional `pg_conninfo`/`project_id` params. When set, the function short-circuits to delegate to `PgApi`, which mirrors the same method names. This is the dual-backend pattern — SQLite locally, Postgres for shared/server deployments.

**Core CRUD:**
- `add_node()` — Creates a belief node with optional SL/CP justifications, outlist conditions, namespace prefixing, `any_mode` (OR-semantics), and access tags.
- `add_justification()` — Appends a justification to an existing node.
- `retract_node()` / `assert_node()` — Flip truth values with cascade propagation. `retract_node` also returns `restoration_hints` — surviving premises from multi-antecedent justifications, useful for telling callers what derived beliefs could be salvaged.
- `update_node()` — In-place mutation of text/source/source_url fields only.
- `remove_justification()` — Removes a justification by index.
- `convert_to_premise()` — Strips all justifications, making a derived node into a premise.

**Simulation (read-only):**
- `what_if_retract()` / `what_if_assert()` — Load network without `write=True`, perform the operation in memory, diff truth values, return cascade predictions without persisting. Uses `_cascade_depth()` (BFS) to report how far each affected node is from the trigger.

**Querying:**
- `get_status()` — All nodes with truth values, filtered by `visible_to`.
- `show_node()` — Full node detail. Raises `PermissionError` for access-tag violations.
- `explain_node()` — Delegates to `Network.explain()` for justification-chain tracing.
- `trace_assumptions()` — Walks backward to find all premise roots.
- `trace_access_tags()` — Union of access tags across the dependency chain.
- `list_nodes()` — Filtered listing with many dimensions: status, premises-only, depth range, namespace, review staleness, impact ordering.
- `list_gated()` — Finds OUT nodes blocked by IN outlist nodes (the "gate" pattern where a belief flips IN when its blocker goes OUT).

**Search:**
- `lookup()` — All-terms substring search across the full belief block (ID, text, source, deps).
- `search()` — FTS5-backed ranked search with progressive relaxation (drops terms via combinations if no results), stop-word filtering, neighbor expansion via BFS, and four output formats (markdown, json, minimal, compact). Falls back to `_substring_search` when FTS5 isn't available.

**Import/Export:**
- `export_network()` / `import_json()` — Lossless round-trip. `import_json` does topological sort to respect dependency ordering, with a fallback pass for cycles.
- `import_beliefs()` — Parses `beliefs.md` markdown format.
- `import_agent()` / `sync_agent()` — Multi-agent federation: imports another agent's beliefs with namespace prefixing and a cascading kill-switch premise (`ns:active`).
- `export_markdown()` — Renders the network as `beliefs.md`.

**Analysis (LLM-assisted):**
- `list_negative()` — Keyword pre-filter + LLM classification to find beliefs describing problems/defects. Batches candidates in groups of 50.
- `review_beliefs()` — LLM review of derived beliefs for validity, sufficiency, necessity. Writes `last_reviewed`/`review_result` metadata.
- `detect_contradictions()` — LLM-based contradiction detection with optional semantic clustering. Can auto-apply nogoods.
- `deduplicate()` — Finds duplicate beliefs via Jaccard similarity on ID tokens or embedding cosine similarity. Can auto-retract duplicates while rewriting dependents.

**Dialectical operations:**
- `challenge()` / `defend()` — Structured argumentation: challenges make a node go OUT, defenses neutralize challenges.
- `add_nogood()` — Records a contradiction between beliefs and triggers backtracking via `find_culprits`.
- `supersede()` — Marks one belief as replacing another (old goes OUT when new is IN).

**Plan workflows:**
- `write_contradiction_plan()` / `parse_contradiction_plan()` / `apply_contradiction_plan()` — Human-in-the-loop workflow for reviewing detected contradictions before applying them.
- `write_dedup_plan()` / `parse_dedup_plan()` / `apply_dedup_plan()` — Same pattern for deduplication.

### 3. Patterns

**Open-operate-save-close:** Every function that touches the database follows this lifecycle via `_with_network`. Write operations pass `write=True`; read operations don't. The context manager only saves when no exception occurred.

**Dual-backend dispatch:** Every public function checks `pg_conninfo` first. If set, it calls `_pg_dispatch` which instantiates `PgApi` and calls the same method name via `getattr`. This keeps the SQLite and Postgres paths completely separate — the API layer is the branch point.

**Namespace isolation:** Agent beliefs are prefixed with `namespace:` and depend on a `namespace:active` premise node. Retracting that premise cascades OUT every belief from that agent. `_resolve_namespace` handles the prefixing, skipping nodes that already contain `:`.

**Visibility/access control:** Many functions accept `visible_to: list[str]`. Nodes with `access_tags` metadata are filtered out unless all tags are in the caller's `visible_to` set. `show_node` and `explain_node` raise `PermissionError` rather than silently filtering.

**Progressive FTS relaxation:** `_fts_search` first tries all terms, then progressively drops terms (via `combinations`) down to `len(terms) // 2`, with a budget cap of 50 queries to prevent combinatorial explosion.

**Deferred imports:** Heavy dependencies (`numpy`, `llm`, `derive`, `cluster`, `contradictions`) are imported inside function bodies to keep the module's import footprint light.

### 4. Dependencies

**Imports from the package:**
- `Justification`, `Nogood`, `Node` from `__init__.py` — core data types
- `Network` from `network.py` — the in-memory belief graph
- `Storage` from `storage.py` — SQLite persistence
- `PgApi` from `pg.py` — PostgreSQL backend (deferred)
- `derive`, `review`, `contradictions`, `cluster`, `compact`, `check_stale`, `export_markdown`, `import_beliefs`, `import_agent`, `llm`, `ask` — various subsystems (all deferred)

**Depended on by:**
- `cli.py` — the argparse-based CLI, primary consumer
- `ask.py` — the LLM-powered question-answering module
- `derive.py` — automated belief derivation
- `contradictions.py` — contradiction detection
- `pg.py` — the Postgres backend (mirrors this API)
- ~20 test files covering every public function

### 5. Flow

A typical mutation like `retract_node("bug-123", reason="Fixed in PR #50")`:

1. Check `pg_conninfo` — if set, dispatch to Postgres and return.
2. Enter `_with_network(db_path, write=True)` — `Storage` opens the SQLite DB, deserializes the `Network`.
3. Snapshot all truth values (`before` dict).
4. Call `net.retract(node_id, reason=reason)` — the `Network` handles cascade propagation, returns list of changed node IDs.
5. Diff `before` vs. current truth values to compute `went_out` and `went_in` lists.
6. Compute `restoration_hints` — for multi-antecedent justifications where some premises survived.
7. Exit context manager — `Storage.save()` serializes the network back to SQLite, then `Storage.close()`.
8. Return JSON-serializable dict.

For `search()`, the flow is: FTS5 query → fallback to substring → access filtering → BFS neighbor expansion (configurable depth) → format selection.

### 6. Invariants

- **Write-on-success only:** `_with_network` only calls `store.save()` when `write=True` AND no exception occurred during the `with` block.
- **Namespace format:** `_resolve_namespace` only prefixes if the ID doesn't already contain `:`. Cross-namespace references (containing `:`) pass through unchanged.
- **Access tag semantics:** A node is visible iff `all(tag in visible_to for tag in node.access_tags)`. Empty tags = always visible.
- **Import ordering:** `import_json` does topological sort — nodes whose antecedents are already loaded get added first. A fixed-point check breaks cycles by adding remaining nodes regardless.
- **FTS relaxation budget:** Capped at 50 queries (`_MAX_RELAXATION_QUERIES`) to prevent combinatorial blowup when many terms yield no results.
- **Nogood ID monotonicity:** `import_json` updates `net._next_nogood_id` to be at least `max(existing) + 1` via regex parsing of `nogood-N` IDs.

### 7. Error Handling

- **KeyError** raised by `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`, `update_node` when a node ID is not found.
- **PermissionError** raised when `visible_to` is set and a node's access tags aren't a subset.
- **FileExistsError** from `init_db` when the database already exists and `force=False`.
- **FileNotFoundError** from import functions when input files don't exist.
- **ValueError** from `add_justification` when neither `--sl`, `--cp`, nor `--unless` is provided.
- **NotImplementedError** for features unsupported with PostgreSQL (`any_mode`, `depth` in search, several `list_nodes` filters).
- LLM-assisted functions (`detect_contradictions`, `list_negative`) catch and warn on per-batch failures rather than aborting the entire operation.

---

## Topics to Explore

- [file] `reasons_lib/network.py` — The in-memory belief graph that implements retraction cascades, assertion restoration, justification evaluation, and `explain()`
- [file] `reasons_lib/storage.py` — SQLite serialization/deserialization and FTS5 index management that `_with_network` wraps
- [file] `reasons_lib/pg.py` — The PostgreSQL backend that mirrors every method in this API via `_pg_dispatch`
- [function] `reasons_lib/import_agent.py:import_agent` — How namespace prefixing and the `ns:active` kill-switch premise work during multi-agent federation
- [general] `outlist-gate-evaluation` — The known issue (CLAUDE.md) where outlist nodes aren't tracked in the dependents index, causing GATE beliefs to need manual assertion after outlist changes

---

## Beliefs

- `api-write-on-success-only` — `_with_network` only persists changes when `write=True` and the context block exits without an exception; read-only calls never modify the database.
- `api-dual-backend-dispatch` — Every public function checks `pg_conninfo` first and delegates to `PgApi` via `_pg_dispatch(getattr)` before touching SQLite, making the API layer the sole branch point between backends.
- `api-namespace-active-premise` — When a namespace is specified, `add_node` creates or reuses a `namespace:active` premise node and injects it as an antecedent, so retracting that single node cascades OUT every belief from that agent.
- `api-fts-relaxation-budget-capped` — `_fts_search` progressively drops query terms via `combinations` but caps total relaxation queries at 50 to prevent combinatorial explosion on long queries.
- `api-access-tag-visibility-is-all-match` — A node is visible only when every entry in its `access_tags` metadata appears in the caller's `visible_to` list; nodes with no tags are always visible.

