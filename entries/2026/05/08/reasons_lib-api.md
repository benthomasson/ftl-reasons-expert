# File: reasons_lib/api.py

**Date:** 2026-05-08
**Time:** 14:03

## `reasons_lib/api.py` — Functional API for the Reason Maintenance System

### 1. Purpose

This is the **public API surface** of the `ftl-reasons` library. It provides stateless, functional entry points that encapsulate the full lifecycle of every database operation: open → load → operate → save → close. The module docstring says it plainly: "any Python caller can use (CLI, LangGraph tools, scripts) without dealing with Storage lifecycle or argparse."

Every function returns a `dict` suitable for JSON serialization, making this the natural boundary between the TMS engine and any external consumer — the CLI (`cli.py`), LLM tool integrations, or programmatic scripts.

### 2. Key Components

**Database lifecycle helper — `_with_network`** (line ~56)

A hand-rolled context manager (not `@contextmanager`, but a class with `__enter__`/`__exit__`). It loads the network from `Storage`, yields it, and conditionally saves on clean exit when `write=True`. This is the backbone pattern — every public function uses it.

```python
with _with_network(db_path, write=True) as net:
    # mutate net
    # auto-saved on exit if no exception
```

**Node CRUD operations:**
- `add_node` — The most complex function (~80 lines). Handles SL/CP justifications, outlist parsing, namespace auto-prefixing, `any_mode` (OR-semantics by expanding a single SL into per-antecedent justifications), and access tags.
- `add_justification` — Adds a justification to an existing node without recreating it.
- `update_node` — In-place mutation of text/source/source_url.
- `retract_node` / `assert_node` — Truth value changes with cascade propagation. Retract also computes `restoration_hints` showing which derived beliefs lost only one premise.

**Simulation (read-only):**
- `what_if_retract` / `what_if_assert` — Load the network, perform the operation in memory, diff the truth values, but never save. The `_cascade_depth` helper computes BFS distance from the retracted node to each affected node.

**Namespace/multi-agent support:**
- `ensure_namespace` — Creates a `<ns>:active` premise node that all namespaced beliefs depend on.
- `list_namespaces` — Scans for `:active` nodes with `agent_premise` role.
- `_resolve_namespace` — Prefixes IDs with `namespace:` unless already namespaced (contains `:`).
- `import_agent` / `sync_agent` — Import another agent's beliefs with namespace isolation. Supports both markdown and JSON formats. `sync_agent` is remote-wins merge semantics.

**Search:**
- `search` — FTS5-backed full-text search with progressive term relaxation (drops terms one at a time via `combinations`) and neighbor expansion (BFS along justification edges). Four output formats: markdown, json, minimal, compact.
- `lookup` — Simpler all-terms substring search over the full belief block.
- `_fts_search` — Strips stop words, quotes terms for FTS5, and relaxes by dropping terms down to `len(terms) // 2`, capped at 50 queries.

**Analysis operations:**
- `review_beliefs` — LLM-assisted review of derived beliefs for validity/sufficiency/necessity. Writes `last_reviewed` and `review_result` metadata.
- `detect_contradictions` — LLM-assisted contradiction detection between IN beliefs. Optionally auto-applies via `add_nogood`.
- `list_negative` — Keyword pre-filter + LLM classification to find beliefs asserting problems/defects.
- `list_gated` — Finds OUT nodes blocked by IN outlist nodes (GATE beliefs waiting for a blocker to be retracted).
- `check_stale` / `hash_sources` — Source file staleness detection via content hashes.

**Deduplication pipeline:**
- `deduplicate` — Jaccard similarity on ID tokens to cluster similar beliefs. `_rewrite_dependents` rewires justification references before retracting duplicates.
- `write_dedup_plan` / `parse_dedup_plan` / `apply_dedup_plan` — Human-in-the-loop workflow: generate a markdown plan, let the user edit it, then apply.

**Export/Import:**
- `export_network` — Full network dump as dict. Filters by `visible_to` access tags.
- `export_markdown` — Delegates to `export_markdown.py` for beliefs.md format.
- `import_json` — Lossless round-trip from JSON. Uses topological sort to add nodes in dependency order, with a fallback pass for cycles.
- `import_beliefs` — Imports from markdown format.

**Dialectical operations:**
- `challenge` / `defend` — Argumentation support. A challenge makes the target go OUT; a defense neutralizes the challenge.
- `add_nogood` — Records a contradiction set and backtrack-resolves it.
- `find_culprits` — Finds retractable premises to resolve a contradiction.
- `supersede` — Marks one belief as replaced by another (old goes OUT when new is IN).

### 3. Patterns

**Transaction-per-call**: Every function is a self-contained database transaction. No connection pooling, no shared state. This makes the API safe for concurrent CLI invocations but means every call pays the load/save cost.

**Dict-in, dict-out**: All functions accept primitive types and return plain dicts. No custom return types, no exceptions-as-flow-control (except `KeyError`/`PermissionError` for not-found/not-visible). This makes the API directly usable as LLM tool definitions.

**Lazy imports**: Heavy modules (`derive`, `review`, `contradictions`, `compact`, `check_stale`, `import_agent`, `import_beliefs`) are imported inside function bodies, not at module level. This keeps import time low when only basic operations are needed.

**Access control via `visible_to`**: A consistent pattern across read operations — if `visible_to` is provided, nodes whose `access_tags` aren't a subset are filtered out or raise `PermissionError`. The `_is_visible` helper enforces this.

**Namespace isolation**: The namespace pattern creates a synthetic dependency — every namespaced belief depends on `<ns>:active`. Retracting that single premise cascades OUT the entire namespace. This is how multi-agent federation works: you trust an agent's beliefs by keeping its `:active` node IN.

### 4. Dependencies

**Imports from `reasons_lib`:**
- `Justification` and `Nogood` (data classes from `__init__.py`)
- `Network` (the in-memory belief graph)
- `Storage` (SQLite persistence layer)
- Various submodules lazily: `derive`, `review`, `contradictions`, `compact`, `check_stale`, `import_beliefs`, `import_agent`, `export_markdown`, `llm`, `ask`

**Imported by:**
- `reasons_lib/cli.py` — CLI entry point; every CLI command is a thin wrapper around an `api` function
- `reasons_lib/ask.py` and `reasons_lib/derive.py` — LLM-driven tools that build on the API
- 16+ test files covering individual features

### 5. Flow

A typical write operation (e.g., `retract_node`):

1. `_with_network(db_path, write=True)` creates a `Storage`, calls `store.load()` → deserializes SQLite rows into a `Network` of `Node` objects
2. Snapshot all truth values as `before` dict
3. Call `net.retract(node_id)` which performs the BFS cascade in the `Network` graph
4. Diff `before` vs current to compute `went_out` and `went_in` lists
5. Compute `restoration_hints` for nodes that lost only one premise
6. Context manager exit: `store.save(network)` serializes back to SQLite, `store.close()`
7. Return dict with results

For `search`, the flow is: FTS5 query → stop word filtering → progressive relaxation → BFS neighbor expansion → access filtering → format selection.

For `import_json`, the flow is: parse JSON → topological sort by dependency availability → add nodes in order → restore exact truth values → import nogoods → import repos.

### 6. Invariants

- **Namespace prefix rule**: `_resolve_namespace` only prefixes when `node_id` contains no `:`. This means cross-namespace references (e.g., `other-agent:some-belief`) are left intact.
- **Write-on-success only**: `_with_network` only saves when `write=True` AND no exception was raised. A failed operation never corrupts the database.
- **Topological import ordering**: `import_json` adds nodes whose antecedents are already in the network first, with a cycle-breaker pass when no progress can be made.
- **FTS relaxation budget**: `_fts_search` caps progressive relaxation at 50 queries to prevent combinatorial explosion on long queries.
- **Stop word filtering**: The `_STOP_WORDS` frozenset is applied before FTS queries to avoid noise. If all terms are stop words, it falls back to terms longer than 1 character.
- **Dedup keeps most-connected**: When `auto=True`, `deduplicate` retains the node with the most dependents (ties broken by ID sort), rewiring all references before retracting others.

### 7. Error Handling

- `KeyError` raised when a node ID is not found (e.g., `show_node`, `explain_node`, `update_node`)
- `PermissionError` raised when access tags don't match `visible_to`
- `FileExistsError` from `init_db` when the database already exists and `force=False`
- `FileNotFoundError` from import functions when input files don't exist
- `ValueError` from `add_justification` when no justification type is provided, and from `derive_prompt` when the network is empty
- LLM-related errors in `list_negative` and `detect_contradictions` are handled per-batch — JSON parse failures from LLM responses are silently skipped (the `for match in re.finditer` pattern)
- `detect_contradictions` with `auto_apply=True` catches all exceptions per-contradiction and prints a warning to stderr

---

## Topics to Explore

- [file] `reasons_lib/network.py` — The in-memory graph engine that `api.py` delegates all propagation, retraction, and cascade logic to
- [file] `reasons_lib/storage.py` — SQLite serialization layer; understanding how `load()`/`save()` work explains the performance characteristics of the transaction-per-call pattern
- [function] `reasons_lib/derive.py:build_prompt` — How the LLM derive pipeline constructs prompts from the belief graph; called by `derive_prompt`
- [file] `reasons_lib/import_agent.py` — Namespace federation internals; handles the JSON vs markdown import paths and sync-with-retraction logic
- [general] `outlist-gate-propagation` — The known issue (documented in CLAUDE.md) where outlist nodes aren't tracked in the dependents index, causing GATE beliefs to require manual `assert` after blocker retraction

---

## Beliefs

- `api-functions-never-hold-connections` — Every public function in `api.py` opens, operates, and closes the database within a single call; no connection or network state survives between calls
- `write-only-on-clean-exit` — `_with_network` saves to disk only when `write=True` and the context exits without exception; a raised error always means no mutation was persisted
- `namespace-colon-convention` — `_resolve_namespace` treats `:` in a node ID as proof the ID is already namespaced and skips prefixing, which means node IDs containing colons for other reasons would bypass namespace resolution
- `fts-relaxation-capped-at-50` — `_fts_search` limits progressive term-dropping to 50 total FTS queries to prevent combinatorial blowup, meaning very long queries may not try all possible subsets
- `dedup-rewrites-before-retract` — `_rewrite_dependents` updates all justification references from the duplicate to the kept node before retracting, preserving derived belief support chains

