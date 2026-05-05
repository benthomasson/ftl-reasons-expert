# File: reasons_lib/api.py

**Date:** 2026-05-05
**Time:** 15:13

## `reasons_lib/api.py` — Functional API for the Reason Maintenance System

### 1. Purpose

This is the **public interface layer** of `ftl-reasons`. It wraps the lower-level `Network` and `Storage` classes into stateless, transactional functions that any caller — CLI, LangGraph tools, scripts — can invoke without managing database lifecycle. Every function follows the same pattern: open database, operate, optionally save, close, return a JSON-serializable dict.

It owns the translation between **user-facing concepts** (comma-separated antecedent strings, namespace prefixes, access tag filtering) and the **internal model** (`Justification` objects, `Network` graph operations).

### 2. Key Components

**`_with_network(db_path, write=False)`** — The central resource-management pattern. Returns a context manager that loads the network from SQLite on enter, and conditionally saves + closes on exit. The `write` flag gates persistence — read-only callers never accidentally mutate the database. Every public function uses this.

**Node lifecycle functions:**
- `add_node()` — Creates a node with optional SL/CP justifications, outlist, namespace prefixing, `any_mode` OR-expansion, and access tags. Returns truth value and type.
- `add_justification()` — Appends a new justification to an existing node. Supports the same `any_mode` expansion.
- `retract_node()` / `assert_node()` — Mutating truth-value changes with cascade tracking. `retract_node` also returns `restoration_hints` — surviving premises that could reconstruct retracted derived beliefs.
- `what_if_retract()` / `what_if_assert()` — **Read-only simulations**. They load the network, perform the operation in memory, diff the before/after truth values, but pass `write=False` so nothing persists. Both compute `_cascade_depth` (BFS shortest-path from source to each affected node) for sorting results.

**Namespace system:**
- `ensure_namespace()` — Creates an `{ns}:active` premise node that acts as a kill switch for an entire agent's beliefs.
- `list_namespaces()` — Discovers namespaces by scanning for `:active`-suffixed nodes with `role: agent_premise` metadata.
- `_resolve_namespace()` — Prefixes bare node IDs with `namespace:` unless they already contain a `:` (cross-namespace references stay untouched).

**Multi-agent federation:**
- `import_agent()` / `sync_agent()` — Import another agent's belief file (markdown or JSON), namespacing every belief under `agent_name:` and wiring dependency on `agent_name:active`. `sync_agent` is remote-wins: it adds new, updates changed, retracts removed.

**Search:**
- `search()` — FTS5-backed search with porter stemming, stop-word filtering, and progressive relaxation (drops terms via `combinations()` until results appear, capped at 50 relaxation queries). Falls back to substring matching. Expands results by BFS through the dependency graph (`depth` parameter). Four output formats: markdown, json, minimal, compact.
- `lookup()` — Simpler all-terms substring search across a wider corpus (ID, text, source, hashes, antecedents, dependents).

**Analysis functions:**
- `list_negative()` — Two-phase negative belief detection: keyword pre-filter against `NEGATIVE_TERMS`, then LLM classification in batches of 50 to distinguish genuinely negative beliefs from those merely describing error-handling mechanisms.
- `review_beliefs()` — LLM-based review of derived beliefs for validity, sufficiency, and necessity. Supports filtering by depth, namespace, dependency, random sampling.
- `deduplicate()` — Jaccard similarity on tokenized belief IDs to find clusters. `auto` mode retracts duplicates and rewrites their dependents to point at the survivor via `_rewrite_dependents()`.
- `list_gated()` — Finds OUT nodes blocked by IN outlist nodes — the "gates" that would flip if their blocker were retracted.

**Import/Export:**
- `export_network()` — Full serialization including nodes, nogoods, and repos.
- `import_json()` — Lossless round-trip reconstruction with topological-sort import (adds nodes whose dependencies are already loaded first, with a progress-stall fallback).
- `export_markdown()` / `import_beliefs()` — Markdown format for human-readable belief registries.

**Maintenance:**
- `check_stale()` — Compares stored source hashes against current file contents to detect beliefs whose source code has changed.
- `hash_sources()` — Backfills missing source hashes.
- `propagate()` — Forces a full `recompute_all()` pass across the network.

### 3. Patterns

**Transaction-per-call**: Every function is self-contained — open, operate, save, close. No shared state between calls. This makes the API safe for concurrent callers and trivially testable (pass a temp db path).

**Comma-separated string parsing**: SL/CP antecedents and outlists arrive as comma-separated strings (CLI-friendly), get split into lists before constructing `Justification` objects.

**Before/after diffing**: `retract_node`, `assert_node`, `what_if_*` all snapshot truth values before the operation, then compare after to classify changes into `went_out` and `went_in` lists.

**Lazy internal imports**: Heavy modules (`derive`, `review`, `compact`, `check_stale`, `import_agent`, `export_markdown`) are imported inside function bodies, not at module level. This keeps import time fast and avoids circular imports.

**Progressive relaxation in search**: `_fts_search` tries all terms first, then drops terms one at a time (largest subsets first) until something matches, capped at 50 total queries to bound cost.

### 4. Dependencies

**Imports from the package:**
- `Justification`, `Nogood` — data classes from `__init__.py`
- `Network` — the in-memory belief graph
- `Storage` — SQLite persistence layer
- Lazy: `derive`, `review`, `compact`, `check_stale`, `import_beliefs`, `import_agent`, `export_markdown`, `llm`, `ask`

**Imported by (meaningful callers):**
- `reasons_lib/cli.py` — the CLI layer that maps argparse to these functions
- `reasons_lib/ask.py`, `reasons_lib/derive.py` — higher-level LLM-driven workflows that compose these primitives
- ~20 test files covering individual features

### 5. Flow

A typical write operation like `retract_node("some-belief")`:
1. `_with_network(db_path, write=True)` opens `Storage`, calls `store.load()` to deserialize the full `Network` from SQLite.
2. Snapshot all truth values: `{nid: node.truth_value for ...}`.
3. Call `net.retract(node_id)` — this is where the TMS algorithm runs (cascade, outlist evaluation, dependent re-evaluation). Returns list of changed node IDs.
4. Diff the snapshot against current truth values to classify `went_out` / `went_in`.
5. Build `restoration_hints` by scanning retracted nodes for justifications with surviving premises.
6. Context manager exit: `store.save(network)` writes everything back to SQLite, `store.close()`.
7. Return the dict.

For search, the flow branches: `_fts_search` opens a raw `sqlite3` connection to query the `nodes_fts` FTS5 table directly (bypassing `Storage`), then falls back to `_substring_search` which operates on the loaded `Network` object.

### 6. Invariants

- **Write gating**: The database is only written when `_with_network(..., write=True)` is used AND no exception occurred during the operation.
- **Namespace isolation**: A namespaced belief always depends on `{ns}:active`. Retracting that premise cascades OUT every belief in the namespace.
- **Cross-namespace references preserved**: `_resolve_namespace` skips IDs that already contain `:`, so `alice:some-belief` referenced from Bob's namespace won't become `bob:alice:some-belief`.
- **Access tag enforcement**: `_is_visible` is checked at read boundaries (`get_status`, `show_node`, `explain_node`, `trace_assumptions`, `search`, `list_nodes`, `list_negative`, etc.). Nodes with no `access_tags` are always visible.
- **Topological import ordering**: `import_json` adds nodes in dependency order, falling through to a force-add pass if the graph has cycles or missing dependencies.
- **FTS relaxation budget**: At most 50 relaxation queries are executed in `_fts_search` to prevent combinatorial explosion on long queries.

### 7. Error Handling

- `KeyError` raised for missing nodes (`show_node`, `explain_node`, `what_if_retract`, etc.).
- `PermissionError` raised when `visible_to` doesn't cover a node's `access_tags`.
- `FileExistsError` from `init_db` if the database already exists without `force=True`.
- `FileNotFoundError` from import functions if the source file doesn't exist.
- `ValueError` from `add_justification` if neither `sl`, `cp`, nor `unless` is provided; from `derive_prompt` if the network is empty.
- SQLite errors in `_fts_search` are silently caught — returns `[]` and the caller falls back to substring search. This is the one place errors are swallowed, and it's deliberate (FTS5 table may not exist).

## Topics to Explore

- [file] `reasons_lib/network.py` — The core TMS graph implementation that this API wraps — retraction cascades, outlist evaluation, `recompute_all`, `explain`, `trace_assumptions`
- [file] `reasons_lib/storage.py` — SQLite persistence layer including FTS5 index creation and the `load`/`save` serialization contract
- [function] `reasons_lib/derive.py:build_prompt` — How the LLM derivation prompt is constructed from the current belief state
- [file] `reasons_lib/import_agent.py` — Multi-agent federation internals: how namespace prefixing and justification rewriting work for both markdown and JSON formats
- [general] `outlist-gated-belief-evaluation` — The known limitation where outlist nodes aren't tracked in the dependents index, causing GATE beliefs to not auto-re-evaluate (documented in CLAUDE.md)

## Beliefs

- `api-functions-are-transactional` — Every public function in `api.py` opens the database, operates, saves (if write=True), and closes within a single call — no state persists between invocations.
- `what-if-functions-never-persist` — `what_if_retract` and `what_if_assert` always pass `write=False` to `_with_network`, so they can never modify the database even if an exception occurs.
- `namespace-active-premise-is-kill-switch` — Retracting `{ns}:active` cascades OUT every belief in that namespace because all namespaced beliefs include it as a justification antecedent.
- `fts-search-caps-relaxation-at-50-queries` — `_fts_search` limits progressive term-dropping to 50 total FTS queries via `_MAX_RELAXATION_QUERIES` to prevent combinatorial blowup.
- `list-negative-uses-two-phase-classification` — Keyword pre-filtering against `NEGATIVE_TERMS` reduces candidates before sending batches of 50 to an LLM for genuine-negative classification.

