# File: reasons_lib/api.py

**Date:** 2026-04-24
**Time:** 16:41

## Purpose

`api.py` is the **service layer** of the Reasons belief-tracking system. It sits between callers (CLI, LangGraph tools, scripts) and the core domain (`Network`, `Storage`). Its single responsibility: expose every network operation as a standalone function that handles its own database lifecycle — open, operate, save, close — so callers never touch `Storage` directly.

Every public function accepts a `db_path` string (defaulting to `"reasons.db"`), returns a plain `dict` (or `str` for markdown/compact output), and is safe to call independently. This makes each function a natural mapping to a CLI subcommand or an LLM tool invocation.

## Key Components

### `_with_network(db_path, write=False)` — The database lifecycle wrapper

A hand-rolled context manager (class-based, not `@contextmanager`) that loads the full network from SQLite on enter and optionally saves on exit. The `write` flag controls persistence: read-only operations pass `write=False` so the database is never modified even if the caller mutates the in-memory network. This is critical for `what_if_retract` and `what_if_assert`, which perform destructive operations in memory to simulate effects.

### `_resolve_namespace(node_id, namespace)` — Namespace prefixing

Prefixes a bare node ID with `namespace:` unless it already contains a colon. The colon check enables cross-namespace references: if a node ID is `other_agent:foo`, it won't get double-prefixed.

### `_is_visible(node, visible_to)` — Access control predicate

Enforces tag-based visibility: a node is visible only if every tag in its `access_tags` metadata is present in the caller's `visible_to` list. Nodes with no tags are always visible. This is a subset check (`all(t in visible_set for t in tags)`), not an intersection.

### Node CRUD

- **`add_node`** — The most complex function. Handles SL/CP justification parsing, outlist parsing, `any_mode` (OR-expansion of antecedents into separate justifications), namespace auto-wiring (creates `ns:active` premise, injects it as an antecedent), and access tag metadata.
- **`add_justification`** — Adds a justification to an *existing* node. Separate from `add_node` because a node can have multiple justifications (a node is IN if ANY justification is valid).
- **`retract_node` / `assert_node`** — Flip truth values and return cascade diffs (`went_out`, `went_in`). `retract_node` also computes `restoration_hints`: for each node that went OUT, it reports which of its antecedents are still IN, helping the caller understand what could bring it back.

### Simulation (what-if)

- **`what_if_retract` / `what_if_assert`** — Load the network with `write=False`, perform the operation in memory, diff the before/after truth values, then let `_with_network.__exit__` discard changes. Both compute `_cascade_depth` to show how far each affected node is from the trigger.

### Import/Export

- **`import_json`** — Reconstructs a full network from JSON export. Uses a topological sort loop: repeatedly scans unprocessed nodes, adding those whose dependencies are already loaded. Falls through to a force-add pass if no progress is made (handles cycles or missing deps).
- **`import_beliefs` / `import_agent` / `sync_agent`** — Delegate to specialized importers. `import_agent` and `sync_agent` handle both `.md` and `.json` formats, dispatching on file extension.
- **`export_network` / `export_markdown`** — Full serialization. `export_markdown` filters by `visible_to` by constructing a new `Network` with only visible nodes.

### Search

- **`search`** — Two-tier: tries FTS5 first (`_fts_search`), falls back to substring matching (`_substring_search`). Expands results by one hop in the dependency graph (neighbors). Supports three output formats: markdown, JSON, minimal.
- **`lookup`** — Simpler all-terms search across a concatenated block of node fields. No FTS, no neighbor expansion.

### Deduplication

- **`deduplicate`** — Clusters IN nodes by Jaccard similarity of tokenized IDs. In `auto` mode, keeps the most-connected node per cluster and retracts the rest after rewiring dependents via `_rewrite_dependents`.
- **`write_dedup_plan` / `parse_dedup_plan` / `apply_dedup_plan`** — Human-in-the-loop workflow: generate a reviewable markdown plan, parse it back, apply the reviewed decisions.

## Patterns

**Open-use-close per call.** Every function is self-contained — no shared state, no connection pooling. This trades performance for simplicity and crash safety: each call is an independent transaction.

**Simulation via read-only context.** `what_if_*` functions exploit the `write=False` flag to perform destructive operations in memory without persisting them. The network is loaded, mutated, diffed, then discarded on context exit.

**Namespace as dependency.** Namespacing isn't just a naming convention — it creates a structural dependency. Every namespaced node depends on `ns:active`, so retracting that single premise cascades OUT the entire namespace. This is the "kill switch" for agent beliefs.

**Lazy imports.** Heavy modules (`import_beliefs`, `import_agent`, `derive`, `compact`, `export_markdown`, `check_stale`) are imported inside function bodies, not at module level. This keeps `api.py` fast to import for callers that only need a subset of operations.

**Dict-in, dict-out.** All functions accept primitive types and return dicts/strings. No domain objects cross the API boundary. This makes every function trivially serializable for CLI output or JSON API responses.

## Dependencies

**Imports from the project:**
- `__init__.py` — `Justification`, `Nogood` dataclasses
- `network.py` — `Network` class (the core dependency graph + propagation engine)
- `storage.py` — `Storage` class (SQLite persistence)
- Various modules via lazy imports: `derive`, `compact`, `export_markdown`, `check_stale`, `import_beliefs`, `import_agent`

**Imported by:**
- `cli.py` — the CLI entry point maps every subcommand to an `api.py` function
- `derive.py` — uses `add_node` to apply derived proposals
- ~15 test files — `api.py` is the primary test surface for the system

## Flow

A typical call like `retract_node("X")`:

1. `_with_network(db_path, write=True)` creates a `Storage`, calls `store.load()` to deserialize the full `Network` from SQLite.
2. Snapshots all truth values (`before` dict).
3. Calls `net.retract("X")` — the `Network` does BFS propagation, flipping dependents whose justifications are no longer satisfied.
4. Diffs `before` vs current to compute `went_out` and `went_in`.
5. Computes `restoration_hints` for collateral damage.
6. On context exit, `store.save(net)` writes the mutated network back to SQLite, then `store.close()`.

For `import_json`, the flow is more complex: a multi-pass topological sort adds nodes in dependency order, then a force-add pass handles any remainder. After all nodes are in, truth values are reconciled by explicitly retracting or asserting to match the source data.

## Invariants

- **Namespace wiring:** If `namespace` is provided to `add_node`, the `ns:active` premise is guaranteed to exist and be an antecedent of the node's justification. This is enforced regardless of whether the caller provides explicit justifications.
- **Visibility is a subset check:** `_is_visible` requires ALL of a node's access tags to be in the caller's `visible_to`. A caller with `["public"]` cannot see a node tagged `["public", "internal"]`.
- **Write-safety:** `_with_network` only calls `store.save()` when `write=True` AND no exception occurred. A crash mid-operation leaves the database untouched.
- **FTS fallback:** `search` always produces results if substring matches exist, even without FTS5 tables.
- **Dedup keeps most-connected:** In auto mode, the node with the most dependents survives. Ties break by lexicographic ID.

## Error Handling

- **`KeyError`** raised when a node ID is not found (`show_node`, `explain_node`, `trace_assumptions`, `what_if_*`).
- **`PermissionError`** raised when `visible_to` is set and the target node's access tags aren't a subset (`show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`).
- **`FileExistsError`** from `init_db` if the database already exists and `force=False`.
- **`FileNotFoundError`** from import functions when the source file doesn't exist.
- **`ValueError`** from `add_justification` when no justification type is specified, and from `derive_prompt` when the network is empty.
- **Silent fallback** in `_fts_search`: catches `sqlite3.OperationalError` and `DatabaseError`, returns empty list. This is intentional — FTS5 tables may not exist in older databases.
- Errors are **not** caught or wrapped at the API boundary — they propagate to the caller (CLI or test), which is responsible for rendering them.

## Topics to Explore

- [file] `reasons_lib/network.py` — The core propagation engine: `retract`, `assert_node`, `add_node`, BFS propagation, challenge/defend, and nogood resolution all live here. Understanding `api.py` requires knowing what `Network` guarantees.
- [function] `reasons_lib/storage.py:load` — How the full network is deserialized from SQLite on every API call. The performance profile of `api.py` is dominated by this load/save cycle.
- [function] `reasons_lib/import_agent.py:import_agent_json` — The namespace import logic that `api.py` delegates to. Understanding how cross-agent belief merging works requires reading this.
- [file] `reasons_lib/derive.py` — The `_tokenize_id` and `_jaccard` functions used by `deduplicate`, plus the `build_prompt` / `apply_proposals` pipeline for LLM-driven belief derivation.
- [general] `outlist-semantics` — Non-monotonic reasoning via outlists is a subtle feature: a justification holds only when its outlist nodes are OUT. This interacts with retraction cascades in non-obvious ways (retracting a blocker can *restore* gated beliefs).

## Beliefs

- `api-functions-are-self-contained` — Every public function in `api.py` opens the database, operates, and closes it independently; no function requires prior setup or shared state.
- `write-false-prevents-persistence` — `_with_network(db_path, write=False)` never calls `store.save()`, so in-memory mutations (used by `what_if_*`) are discarded on exit.
- `namespace-creates-structural-dependency` — When `namespace` is passed to `add_node`, the resulting node always has `ns:active` as a justification antecedent, making it retractable by retracting that single premise.
- `visibility-is-subset-not-intersection` — `_is_visible` requires every tag in a node's `access_tags` to be present in `visible_to`; having *some* matching tags is insufficient.
- `import-json-uses-topological-sort` — `import_json` adds nodes in dependency order across multiple passes, falling through to a force-add pass only when no further progress can be made.

