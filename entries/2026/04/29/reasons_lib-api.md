# File: reasons_lib/api.py

**Date:** 2026-04-29
**Time:** 17:05

## Purpose

`api.py` is the **functional API layer** for the Reason Maintenance System (RMS). It sits between callers (CLI, LangGraph tools, scripts) and the core engine (`Network` + `Storage`), providing stateless, transaction-scoped functions that each open the database, operate, save, and close. Every public function returns a dict suitable for JSON serialization, making this the canonical boundary for programmatic access.

It owns three responsibilities:
1. **Lifecycle management** — callers never touch `Storage` or worry about save/close
2. **Input normalization** — comma-separated strings become lists, namespaces get prefixed, access tags get checked
3. **Output shaping** — raw network state is projected into JSON-friendly dicts with computed fields (counts, depths, cascade effects)

## Key Components

### Infrastructure

**`_with_network(db_path, write=False)`** — The central pattern. Returns a context manager that loads the network from SQLite on enter and conditionally saves on exit (only if `write=True` and no exception). Every public function uses this, which means every API call is a self-contained transaction.

**`_is_visible(node, visible_to)`** — Access control gate. A node is visible if its `access_tags` are a subset of the caller's tags. Nodes without tags are always visible. This is checked at query boundaries, not write boundaries.

**`_resolve_namespace(node_id, namespace)`** — Prefixes bare IDs with `namespace:` but leaves already-namespaced IDs (containing `:`) untouched, enabling cross-namespace references.

### CRUD Operations

**`add_node()`** — The most complex function. Handles: SL/CP justification construction from comma-separated strings, `any_mode` (OR-semantics by expanding one SL into N single-antecedent SLs), namespace auto-wiring (creates `ns:active` premise, prepends it to antecedents), and access tag assignment.

**`add_justification()`** — Adds a justification to an existing node. Mirrors `add_node`'s parsing logic for SL/CP/outlist but operates on an already-present node. Returns before/after truth values so callers can detect state changes.

**`retract_node()` / `assert_node()`** — Mutating truth-value operations. `retract_node` snapshots truth values before the retraction, then classifies changes into `went_out` and `went_in` lists. It also computes `restoration_hints` — surviving premises from multi-antecedent justifications that could be used to rebuild retracted derived beliefs.

### Simulation (What-If)

**`what_if_retract()` / `what_if_assert()`** — Read-only simulations. They load the network with `write=False`, perform the operation in memory, diff the truth values, compute cascade depths via `_cascade_depth` (BFS through the dependents graph), and return structured results. The database is never modified.

**`_cascade_depth()`** — BFS from the retracted node through `dependents` edges to find the shortest path to a target. Used to sort cascade effects by proximity.

### Query Operations

**`get_status()`** — Full listing with optional access filtering. Returns node summaries with truth values and justification counts.

**`show_node()` / `explain_node()`** — Single-node inspection. `show_node` returns full structural detail; `explain_node` delegates to `Network.explain()` for a step-by-step proof trace.

**`trace_assumptions()`** — Backward traversal to find all premises a derived node rests on.

**`search()` / `lookup()`** — Two search strategies. `search` tries FTS5 first (`_fts_search`), falls back to substring matching (`_substring_search`), then expands results with 1-hop neighbors from the dependency graph. `lookup` is simpler: all-terms substring matching across a concatenated block of node fields.

**`list_nodes()`** — Filtered listing with predicates: status, premises-only, has-dependents, challenged, namespace, and depth range. Depth is computed lazily via `_node_depth` (recursive with memoization and cycle guard).

**`list_gated()`** — Finds OUT nodes whose outlist contains IN nodes — these are "gated" beliefs waiting for their blocker to be retracted.

### Dialectical Operations

**`challenge()` / `defend()`** — Argument-style operations. `challenge` creates a challenge node that forces the target OUT. `defend` neutralizes a challenge, restoring the target.

**`supersede()`** — Marks one belief as replaced by another. The old goes OUT when the new is IN (via outlist mechanics).

**`add_nogood()`** — Records a contradiction between a set of nodes. Finds culprits (retractable premises) and uses backtracking to resolve.

### Import/Export

**`import_json()`** — Lossless reconstruction from a JSON export. Uses a topological sort loop: repeatedly scans remaining nodes, adding those whose dependencies are already present. Falls through to force-add if no progress (handles cycles or missing deps). Restores exact truth values by retracting/asserting after node creation.

**`import_beliefs()` / `import_agent()` / `sync_agent()`** — Higher-level importers. `import_agent` namespaces all beliefs under `agent_name:` with a dependency on `agent_name:active`. `sync_agent` does remote-wins merging.

**`export_network()` / `export_markdown()`** — Serialization. `export_network` produces a full JSON-compatible dict. `export_markdown` generates `beliefs.md` format. Both support access-tag filtering by constructing a filtered `Network` in memory.

### Deduplication

**`deduplicate()`** — Clusters IN beliefs by Jaccard similarity on ID tokens. Uses union-find to group similar IDs. In `auto` mode, keeps the most-connected belief per cluster, rewrites dependents via `_rewrite_dependents`, and retracts the rest.

**`write_dedup_plan()` / `parse_dedup_plan()` / `apply_dedup_plan()`** — Human-in-the-loop dedup workflow. Write a plan file, let a human edit it, parse it back, apply it.

### LLM Integration

**`list_negative()`** — Two-stage classification. First, keyword pre-filtering against `NEGATIVE_TERMS` (a hardcoded list of ~50 problem-indicating words). Then LLM classification via `ask._invoke_claude` to filter false positives (e.g., beliefs that *describe* error handling aren't themselves negative).

**`derive_prompt()` / `derive_apply()`** — LLM-driven belief derivation. Builds a prompt from the current network state, then applies validated proposals back.

## Patterns

1. **Transaction-per-call**: Every function opens the DB, operates, saves (if mutating), closes. No shared state between calls. This is enforced by `_with_network`.

2. **Read/write split via `write` flag**: The context manager only saves on `write=True` with no exception. What-if functions exploit this to mutate in memory without persisting.

3. **Comma-separated string parsing**: The CLI passes SL/CP/outlist as comma-separated strings. The API parses them into lists. This is a boundary concern — the core `Network` works with lists.

4. **Namespace auto-wiring**: When `namespace` is provided, the API transparently creates the `ns:active` premise, prepends it to antecedents, and resolves all references. Callers don't manage namespace plumbing.

5. **Before/after diffing**: Mutating operations snapshot truth values before the operation, then diff afterward to classify changes into `went_out`/`went_in`. This is repeated in `retract_node`, `assert_node`, `what_if_retract`, and `what_if_assert`.

6. **Lazy imports**: Heavy modules (`import_beliefs`, `import_agent`, `derive`, `compact`, `check_stale`, `ask`) are imported inside function bodies, keeping the module's import footprint light.

## Dependencies

**Depends on:**
- `network.Network` — the in-memory belief graph (nodes, justifications, propagation)
- `storage.Storage` — SQLite persistence (load/save/close)
- `Justification` / `Nogood` — data classes from `__init__.py`
- `import_beliefs`, `import_agent`, `derive`, `export_markdown`, `check_stale`, `compact`, `ask` — specialized modules, all lazily imported

**Depended on by:**
- `cli.py` — the CLI entry point delegates almost everything to these API functions
- `ask.py` / `derive.py` — LLM-driven features call back into the API for network access
- 15+ test files — the API is the primary test surface

## Flow

A typical mutating call like `retract_node("some-belief", reason="wrong")`:

1. `_with_network(db_path, write=True)` creates a `Storage`, calls `store.load()` to deserialize the SQLite DB into a `Network`
2. Snapshot all truth values: `{nid: node.truth_value for ...}`
3. Call `net.retract(node_id, reason=reason)` — the network engine propagates the retraction through the dependency graph
4. Diff the snapshot against new truth values to compute `went_out` and `went_in`
5. Compute `restoration_hints` by examining surviving premises of multi-antecedent justifications
6. Context manager exits: `store.save(network)` writes back to SQLite, `store.close()` releases the connection
7. Return a dict with `changed`, `went_out`, `went_in`, `restoration_hints`

## Invariants

- **Every public function is self-contained**: opens DB, operates, closes. No function assumes the DB is already open.
- **Write-only-on-success**: `_with_network` only saves if `write=True` AND no exception was raised.
- **Namespace IDs always contain `:`**: `_resolve_namespace` adds `namespace:` only if `:` is absent, preventing double-prefixing.
- **Access control is enforced at read boundaries**: `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags` raise `PermissionError` if the node isn't visible. Write operations don't check visibility.
- **`what_if_*` never persist**: They use `write=False`, so mutations stay in memory.
- **Topological import order**: `import_json` adds nodes in dependency order, with a fallback force-add loop to handle cycles.
- **Dedup keeps the most-connected node**: `max(members, key=lambda nid: (len(dependents), nid))` — ties broken by lexicographic ID.

## Error Handling

- **`KeyError`** raised when a node ID isn't found (`show_node`, `explain_node`, `what_if_*`, `trace_*`)
- **`PermissionError`** raised when access tags don't match (`show_node`, `explain_node`, `trace_*`)
- **`FileExistsError`** from `init_db` if DB exists without `force=True`
- **`FileNotFoundError`** from import functions when source files don't exist
- **`ValueError`** from `add_justification` when no SL/CP/unless is provided, and from `derive_prompt` when the network is empty
- **FTS5 failures are silently swallowed**: `_fts_search` catches `sqlite3.OperationalError` and returns `[]`, falling through to substring search
- **The context manager suppresses nothing**: `__exit__` returns `False`, so exceptions propagate after cleanup

## Topics to Explore

- [file] `reasons_lib/network.py` — The core propagation engine behind `retract`, `assert_node`, `add_nogood`, and `explain` — understanding it reveals what the API is actually delegating to
- [file] `reasons_lib/storage.py` — How the network is serialized to/from SQLite, including the FTS5 index that `_fts_search` queries
- [function] `reasons_lib/import_agent.py:import_agent_json` — The JSON import path for multi-agent federation, which preserves full justification structure including outlists
- [file] `reasons_lib/derive.py` — LLM-driven belief derivation: how `build_prompt` constructs prompts from network state and `apply_proposals` validates and commits new beliefs
- [general] `outlist-propagation` — How outlist (non-monotonic) justifications interact with retraction cascades — this is the mechanism behind GATE beliefs and `list_gated`

## Beliefs

- `api-transaction-per-call` — Every public function in `api.py` opens the database, operates, and closes it; no function assumes an already-open connection or shares state with other calls
- `api-write-flag-controls-persistence` — The `_with_network` context manager only calls `store.save()` when `write=True` and no exception occurred, which is how `what_if_retract` and `what_if_assert` simulate mutations without persisting them
- `api-namespace-colon-convention` — Namespace prefixing is skipped when a node ID already contains `:`, preventing double-prefixing and enabling cross-namespace references in justification antecedents
- `api-access-control-read-only` — Access tag checks (`_is_visible` / `PermissionError`) are enforced on read operations (`show_node`, `explain_node`, `trace_*`) but not on write operations (`add_node`, `retract_node`)
- `api-fts-fallback-to-substring` — `search()` tries SQLite FTS5 first and silently falls back to substring matching if the FTS table doesn't exist or the query fails, so search always returns results even without the index

