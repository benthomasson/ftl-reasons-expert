# File: reasons_lib/api.py

**Date:** 2026-04-23
**Time:** 16:39

## `reasons_lib/api.py` — Functional API for a Reason Maintenance System

### 1. Purpose

This file is the **public API layer** for the entire reason maintenance system (RMS). It sits between consumers (CLI, LangGraph tools, scripts, tests) and the core engine (`Network`, `Storage`). Its single responsibility: expose every RMS operation as a **standalone function** that handles its own database lifecycle — open, operate, save, close — so callers never manage `Storage` or `Network` objects directly.

Every function accepts a `db_path` parameter (defaulting to `"reasons.db"`) and returns a `dict` suitable for JSON serialization. This makes the API trivially adaptable to CLI output, HTTP responses, or tool-call results.

### 2. Key Components

**Infrastructure**

- **`_with_network(db_path, write=False)`** — A hand-rolled context manager factory. Returns an object whose `__enter__` loads the network from SQLite via `Storage`, and whose `__exit__` conditionally saves (only if `write=True` and no exception occurred). This is the single chokepoint for all database access in the module.

- **`_resolve_namespace(node_id, namespace)`** — Prefixes a node ID with `namespace:` unless it already contains a colon (indicating it's already namespaced, possibly cross-namespace). This is the namespace-scoping primitive.

- **`DEFAULT_DB = "reasons.db"`** — The conventional database path, used as the default across all functions.

**Core CRUD operations**

| Function | Write? | What it does |
|---|---|---|
| `init_db` | creates file | Creates a fresh SQLite database; raises `FileExistsError` unless `force=True` |
| `add_node` | yes | Adds a node with optional SL/CP justifications, namespace support, and `any_mode` (OR-semantics) |
| `add_justification` | yes | Appends a justification to an existing node |
| `retract_node` | yes | Retracts a premise; returns cascade effects and restoration hints |
| `assert_node` | yes | Restores a retracted node; returns cascade effects |
| `convert_to_premise` | yes | Strips justifications, making a derived node into a premise |
| `show_node` | no | Full detail dump for one node |
| `get_status` | no | Lists all nodes with truth values |
| `list_nodes` | no | Filtered listing with depth, namespace, status, challenged filters |

**Reasoning / analysis (read-only)**

| Function | What it does |
|---|---|
| `explain_node` | Traces why a node is IN or OUT |
| `trace_assumptions` | BFS backward to find all premise ancestors |
| `find_culprits` | Identifies which premises to retract to resolve a contradiction |
| `what_if_retract` | Simulates retraction without mutating the DB — shows cascade + restoration |
| `what_if_assert` | Simulates assertion without mutating — shows what would change |
| `get_belief_set` | Returns all node IDs currently IN |

**Dialectical operations (write)**

| Function | What it does |
|---|---|
| `challenge` | Creates a challenge node; target goes OUT |
| `defend` | Neutralizes a challenge; target restored |
| `add_nogood` | Records a contradiction set; triggers backtracking |
| `supersede` | Old node goes OUT when new node is IN (outlist-based) |

**Import / export**

| Function | Direction | Format |
|---|---|---|
| `export_network` | out | Dict (JSON-serializable) |
| `export_markdown` | out | `beliefs.md` markdown |
| `import_beliefs` | in | `beliefs.md` markdown |
| `import_agent` | in | Namespaced import from markdown or JSON |
| `sync_agent` | in | Remote-wins sync from markdown or JSON |
| `import_json` | in | Full lossless round-trip from JSON export |
| `compact` | out | Token-budgeted summary string |

**Search**

- **`lookup`** — Simple all-terms substring search across node fields. Returns formatted markdown.
- **`search`** — FTS5-backed ranked search with 1-hop neighbor expansion. Falls back to substring matching if FTS isn't available. Supports markdown, JSON, and minimal output formats.

**Deduplication**

- **`deduplicate`** — Clusters IN beliefs by Jaccard similarity on tokenized IDs. Optionally auto-retracts duplicates, keeping the most-connected node per cluster.
- **`write_dedup_plan` / `parse_dedup_plan` / `apply_dedup_plan`** — Human-in-the-loop dedup workflow: generate a reviewable plan, parse the edited result, apply it.

### 3. Patterns

**Transaction-per-function.** Every function opens the DB, does its work, and closes — no shared state, no connection pooling. This is deliberate: the module targets CLI and tool-call usage where each invocation is independent.

**Context manager for write safety.** `_with_network` ensures the network is only saved when `write=True` and no exception was raised. Read-only operations pass `write=False`, so even if they mutate the in-memory network (like `what_if_retract` does), nothing persists.

**Namespace as dependency injection.** When `namespace` is set, the function auto-creates a `{namespace}:active` premise node and wires it as an antecedent into every justification. Retracting that single premise cascades OUT every belief from that namespace — a clean kill-switch for agent trust.

**Any-mode (OR-semantics).** When `any_mode=True` and multiple antecedents are given, each antecedent gets its own SL justification. This means the node is IN if *any* antecedent is IN, rather than requiring *all* of them (the default AND-semantics of a single multi-antecedent SL justification).

**Lazy imports.** Heavy modules (`import_beliefs`, `import_agent`, `derive`, `compact`, `check_stale`, `export_markdown`) are imported inside function bodies to keep the module lightweight at import time.

**Topological import ordering.** `import_json` uses a multi-pass algorithm to add nodes in dependency order — nodes whose antecedents are already present get added first. When no progress can be made (cycles or missing deps), it adds the remaining nodes anyway and forces truth values afterward.

### 4. Dependencies

**Imports from the package:**
- `Justification` and `Nogood` from `__init__.py` — data classes for justification records and contradiction sets
- `Network` — the core truth-maintenance engine (unused directly but present as a type)
- `Storage` — SQLite persistence layer for `Network`
- Lazy: `import_beliefs`, `import_agent`, `derive`, `compact`, `check_stale`, `export_markdown`

**Imported by:**
- `reasons_lib/cli.py` — the Click CLI dispatches to these functions
- `reasons_lib/derive.py` — uses `add_node` to apply derived proposals
- Every test file in `tests/` — the API is the primary test surface

### 5. Flow

A typical write operation like `add_node`:

1. Parse string args (`sl`, `cp`, `unless`) into `Justification` objects
2. If `namespace` is set: prefix the node ID, ensure `{ns}:active` exists, inject it as an antecedent, resolve all references
3. Enter `_with_network(write=True)` — loads the full network from SQLite
4. Call `net.add_node(...)` — the `Network` engine computes truth values and propagates
5. Exit context manager — `Storage.save()` persists the updated network
6. Return a dict with the node's final state

For `what_if_retract` / `what_if_assert`, the same load happens but with `write=False`. The retraction/assertion runs in memory, effects are computed by diffing before/after truth values, and nothing is saved.

### 6. Invariants

- **Write-or-not is declared upfront.** A function either passes `write=True` or `write=False` to `_with_network`. There's no conditional saving.
- **Namespace nodes always have the `{ns}:active` premise.** `add_node` with a namespace will create it if missing, and every namespaced belief depends on it.
- **Colon in a node ID means "already namespaced."** `_resolve_namespace` never double-prefixes.
- **`what_if_*` functions are read-only.** They mutate the in-memory network but `write=False` prevents persistence.
- **`import_json` preserves exact truth values.** After adding a node, it force-retracts or force-asserts to match the exported truth value.
- **Dedup keeps the most-connected node.** The node with the most dependents (tie-broken by ID) survives.

### 7. Error Handling

- **`FileExistsError`** from `init_db` when the database already exists and `force=False`.
- **`FileNotFoundError`** from import functions when the source file doesn't exist.
- **`KeyError`** from `show_node`, `what_if_retract`, `what_if_assert` when a node ID isn't found.
- **`ValueError`** from `add_justification` when no justification type is specified, and from `derive_prompt` when the network is empty.
- All exceptions propagate to the caller. The `_with_network` context manager catches nothing — it only skips saving if an exception occurred (`exc_type is None` check).

---

## Topics to Explore

- [file] `reasons_lib/network.py` — The core truth-maintenance engine that `api.py` delegates all reasoning to — `add_node`, `retract`, `assert_node`, `explain`, `find_culprits`, and propagation logic live here
- [file] `reasons_lib/storage.py` — SQLite persistence layer: how the network is serialized/deserialized and whether FTS5 indexes are created
- [function] `reasons_lib/network.py:add_nogood` — Backtracking implementation: how contradictions are detected and which premise gets retracted
- [file] `reasons_lib/import_agent.py` — Namespace import logic for both markdown and JSON formats, including `sync_agent` remote-wins semantics
- [general] `outlist-semantics` — How outlist (the `unless` parameter) creates non-monotonic justifications where a node is IN only when its outlist nodes are OUT — the foundation for `supersede`, `challenge`, and default-logic patterns

---

## Beliefs

- `api-functions-return-dicts` — Every public API function returns a `dict` (or `str` for markdown/compact), never a `Network` or `Node` object, ensuring JSON-serializability at the boundary
- `write-false-prevents-persistence` — Functions using `_with_network(write=False)` can mutate the in-memory network (as `what_if_retract` does) but changes are never saved to disk
- `namespace-colon-convention` — A colon in a node ID signals it is already namespaced; `_resolve_namespace` will not add a second prefix, enabling cross-namespace references
- `namespace-active-premise-is-kill-switch` — Every namespaced belief depends on `{namespace}:active`; retracting that single premise cascades OUT all beliefs from that namespace
- `import-json-topological-sort` — `import_json` uses a multi-pass topological ordering to add nodes after their dependencies, falling back to forced insertion when no progress is possible (cycles)

