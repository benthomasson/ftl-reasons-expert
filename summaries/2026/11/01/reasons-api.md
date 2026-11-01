---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/api.py

# `reasons/api.py` — Functional API for the Reason Maintenance System

## 1. Purpose

This file is the **primary programmatic interface** to the Reason Maintenance System (RMS), a Truth Maintenance System (TMS) that models beliefs as nodes in a justification graph. Every higher-level consumer — the CLI (`reasons/cli.py`), the MCP server (`reasons/mcp_server.py`), LangGraph tools, and integration scripts — calls into this module.

Its core responsibility is to provide **standalone, stateless functions** that encapsulate the full database lifecycle (open → operate → save → close), so callers never touch `Storage` or `argparse` directly. All functions return JSON-serializable dicts, making them safe to use from any Python context or over a network boundary.

The file also owns:
- The **in-memory network cache** (`_network_cache`) and its invalidation logic
- **Access control** enforcement (`_is_visible`, `visible_to` parameter)
- **Namespace scoping** (per-agent belief isolation via `ns:active` premise nodes)
- The **proposal workflow** (propose → accept/reject/withdraw) for multi-agent coordination
- **PostgreSQL dispatch** (`_pg_dispatch`) as an alternative backend

## 2. Key Components

### Core Infrastructure

| Component | Contract |
|-----------|----------|
| `_network_cache: dict[str, tuple[float, Network]]` | Module-level cache keyed by `db_path`. Value is `(mtime, Network)`. Read-only calls reuse the cache when mtime matches; write calls always load fresh. |
| `_with_network(db_path, write)` | Context manager. On enter: loads (or caches) a `Network`. On exit (write, success): saves, publishes pubsub events, updates cache. On exit (write, exception): does NOT save or update cache. |
| `_pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs)` | Thin wrapper that instantiates `PgApi` and delegates. Every public function checks `if pg_conninfo:` at the top and short-circuits here. |
| `clear_network_cache(db_path=None)` | Invalidates one or all cached networks. Must be called when external processes modify the DB file. |
| `_is_visible(node, visible_to)` | Returns `True` if `node.metadata["access_tags"]` ⊆ `visible_to`. Nodes with no tags are always visible. |
| `_resolve_namespace(node_id, namespace)` | Prepends `namespace:` to `node_id` unless it already contains a `:` (cross-namespace reference). |

### CRUD Operations

| Function | Notes |
|----------|-------|
| `add_node(...)` | Creates a node with SL/CP justification. Supports `any_mode` (OR semantics: one justification per antecedent). Auto-creates `ns:active` premise if `namespace` is given. Validates `source_type` against `SOURCE_TYPES = {"code", "document", "self-description", "derived"}`. |
| `add_justification(...)` | Attaches an additional justification to an existing node. Same `any_mode` expansion. |
| `retract_node(...)` | Retracts and cascades. Returns `restoration_hints` — antecedents that are still IN, suggesting what to fix. |
| `assert_node(...)` | Restores an OUT node and cascades. |
| `remove_justification(...)` | Removes justification by index, recomputes. |
| `convert_to_premise(...)` | Strips all justifications, making the node a premise (IN by default). |

### Simulation (Read-Only)

| Function | Contract |
|----------|----------|
| `what_if_retract(node_id)` | Deep-copies the network, simulates retraction in memory. Returns `retracted` (went OUT) and `restored` (went IN via outlist). **Never writes to disk.** |
| `what_if_assert(node_id)` | Mirror of above for assertion. |
| `what_if_supersede(old_id, new_text)` | Simulates adding a new node + superseding. Auto-generates `new_id` as `{old_id}-v2`, `-v3`, etc. |

All three use `copy.deepcopy(net)` to protect the cached network from mutation.

### Query & Search

| Function | Notes |
|----------|-------|
| `get_status(...)` | Full network summary with counts by IN/OUT, premise/derived, challenged/superseded, and `by_namespace` breakdown. Supports `visible_to`, `namespace`, `status_filter`, `premises_only`, `limit`/`offset`. |
| `show_node(node_id)` | Full node detail including justifications, dependents, metadata, tags (from `node_tags` table), and sources (from `node_sources` table). Raises `PermissionError` on access violation. |
| `explain_node(node_id)` | Step-by-step TMS explanation of why a node is IN or OUT. |
| `trace_assumptions(node_id)` | Walks backward to all root premises. Filters by `visible_to`. |
| `list_nodes(...)` | Filterable listing: `status`, `premises_only`, `has_dependents`, `challenged`, `min_depth`/`max_depth`, `never_reviewed`, `not_reviewed_since`, `by_impact`, `tag`, `sort`. |
| `search(query, ...)` | FTS5 full-text search with progressive relaxation, OR-pipe queries (`|`), regex mode, neighbor expansion (BFS `depth` hops), tag filtering, namespace filtering, multiple output formats. |
| `lookup(query)` | Simpler all-terms substring search with progressive term relaxation. |

### Import / Export

| Function | Direction |
|----------|-----------|
| `export_network(visible_to)` | → Full network as dict (nodes, nogoods, repos, meta). Lossless. |
| `import_json(json_file)` | ← Reconstructs network via topological sort. Restores exact truth values, timestamps, tags, sources. |
| `export_markdown(...)` | → `beliefs.md`-compatible string. |
| `export_card(...)` | → HuggingFace EEM README card. |
| `import_hf(repo_id)` | ← Downloads from HuggingFace, calls `import_json`. |
| `publish_hf(repo_id)` | → Uploads `network.json` + `beliefs.md` + `README.md` to HF. |
| `import_agent(agent_name, file)` | ← Imports another agent's beliefs with `agent_name:` prefixing. |
| `sync_agent(agent_name, file)` | ← Remote-wins sync: adds, updates, retracts. |
| `import_api(url)` / `export_api(url)` | ←→ Agentic-mind-service HTTP integration. |

### Review & Repair (LLM-Driven)

| Function | Purpose |
|----------|---------|
| `review_beliefs(...)` | LLM evaluates derived beliefs for validity/sufficiency/necessity. Stamps `reviewed_at`, writes `review_result`. |
| `review_justifications(...)` | LLM checks if SL justifications should be ALL vs ANY. Read-only. |
| `review_premises(...)` | LLM checks premises against source documents for accuracy. |
| `repair_premises(...)` | Rewrites or retracts inaccurate premises based on review results. |
| `repair(...)` / `repair_smuggled(...)` | Triage invalid beliefs: link to existing, soften, or abandon. |
| `propose_update(...)` | LLM proposes structured updates/retractions with cascade impact. |
| `detect_contradictions(...)` | LLM finds contradictions between IN beliefs. Optional semantic clustering. |
| `verify_belief(...)` | LLM verifies belief against source. Can auto-retract STALE beliefs. |
| `list_negative(...)` | Keyword pre-filter + LLM classification to find problem/risk beliefs. |

### Defeat & Challenge (Dialectical)

| Function | Mechanism |
|----------|-----------|
| `defeat_justification(...)` | Creates a defeater premise node, adds it to the target justification's outlist. Graph-native — retracting the defeater restores the belief. |
| `defeat_with_scope(...)` | Creates scope beliefs + a derived defeater with SL justification. More structured, allows per-antecedent challenge. |
| `challenge(target_id, reason)` | Creates a challenge node; target goes OUT. |
| `defend(target_id, challenge_id)` | Neutralizes the challenge; target restored. |
| `migrate_retract_to_defeaters(...)` | Migrates legacy `retract_reason` metadata strings into graph-native defeater nodes. |

### Deduplication & Clustering

| Function | Notes |
|----------|-------|
| `deduplicate(threshold, auto, semantic)` | Union-Find clustering by Jaccard (ID tokens) or cosine (embeddings). `auto=True` retracts all but the most-connected node, rewriting dependents. |
| `list_clusters(...)` | Returns full cluster assignments without mutating. |
| `write_dedup_plan` / `parse_dedup_plan` / `apply_dedup_plan` | Human-in-the-loop workflow. |

### Proposals (Multi-Agent Coordination)

A full propose/accept/reject/withdraw lifecycle stored in a SQLite `proposals` table:

| Function | Effect |
|----------|--------|
| `propose_retraction(target_id)` | Stores pending proposal + impact snapshot. Does NOT change truth values. |
| `propose_supersession(old_id, new_text)` | Same, for version replacement. |
| `propose_addition(node_id, text)` | Same, for new nodes. |
| `propose_nogood(node_ids)` | Same, for contradiction recording. |
| `accept_proposal(id)` | Re-validates against live network (drift detection), then applies. |
| `reject_proposal(id)` / `withdraw_proposal(id)` | Mark as rejected/withdrawn. No truth change. |
| `list_proposals(...)` / `show_proposal(id)` | Query the proposal store. |

Key invariant: `accept_proposal` checks `snapshot_json` against current state. If truth_value or text_hash drifted, it marks the proposal `stale` without applying.

### Maintenance

| Function | Purpose |
|----------|---------|
| `check_stale(...)` | Detects IN nodes whose source file has changed. Optional hash upgrade, git-aware SHA skipping. |
| `check_integrity(...)` | Merkle hash verification for all nodes. |
| `backfill_hashes(...)` | Compute missing hashes. |
| `hash_sources(...)` | Backfill source content hashes. |
| `pin_sources(...)` / `pin_update(...)` / `pin_lines(...)` | Git SHA pinning for staleness checking. |
| `compact(budget)` | Token-budgeted belief summary for LLM context windows. |
| `build_wiki(...)` | Generates interlinked markdown wiki pages. |
| `topics(...)` | Word-frequency topic extraction from node IDs. |
| `list_gated(...)` | Finds OUT nodes blocked by IN outlist nodes (active gates). |
| `report_gated(...)` / `report_belief(...)` | LLM-generated problem/evidence reports. |

## 3. Patterns

### Context Manager with Caching (`_with_network`)

The single most important pattern. Every network operation goes through this:

```python
with _with_network(db_path, write=True) as net:
    net.add_node(...)
    net.retract(...)
```

- **Read path**: Checks `_network_cache[db_path]`. If `(mtime, Network)` matches current file mtime, reuses the object. Otherwise loads via `Storage(db_path).load()`.
- **Write path**: Always loads fresh (bypasses cache). On clean exit, calls `store.save(network)`, computes pubsub diff events, publishes, then updates the cache entry.
- **Exception path**: If an exception propagates out of the `with` block during a write, the network is NOT saved and the cache is NOT updated. The `Storage` connection is still closed.

### Strategy Pattern (Backend Dispatch)

Every public function begins with:
```python
if pg_conninfo:
    return _pg_dispatch(pg_conninfo, domain_id, "method_name", **kwargs)
```
This cleanly separates the SQLite (default) implementation from the PostgreSQL one. Features unsupported by PG raise `NotImplementedError` at the dispatch site.

### Simulation Before Mutation

The `what_if_*` family and the proposal system both follow: **compute the cascade effect in a sandbox, then decide whether to apply**. The simulation uses `copy.deepcopy` to avoid corrupting the cached live network.

### Progressive Relaxation (Search)

Both `lookup` and `_fts_search` implement a fallback strategy: try all terms → drop one term at a time (combinations) → stop after a budget of 50 relaxation queries. This prevents zero-result UX on multi-term queries where one term is slightly off.

### Union-Find for Clustering

`deduplicate` uses path-compression Union-Find to group similar nodes in O(n² α(n)) time, then keeps the most-connected node per cluster.

### Human-in-the-Loop Plan Files

`write_dedup_plan` / `parse_dedup_plan` / `apply_dedup_plan` and the contradiction plan equivalents create a text format that a human edits (change `[APPLY]` to `[SKIP]`), then a parser extracts the actionable subset. This avoids destructive bulk operations.

## 4. Dependencies

### Imports (what this file needs)

| Module | Used For |
|--------|----------|
| `.network` (`Network`, `Node`, `Justification`, `Nogood`) | Core data structures |
| `.storage` (`Storage`, `PROPOSALS_SCHEMA`) | SQLite persistence + proposals table DDL |
| `.pubsub` | Publish change events on write |
| `.metadata` (`build_meta`) | Network metadata construction for export |
| `.pg` (`PgApi`) | PostgreSQL backend |
| `.hf` | HuggingFace download/upload |
| `.mind_service` | Agentic-mind HTTP integration |
| `.check_stale` | Source staleness, pinning, hash computation |
| `.merkle` | Integrity verification |
| `.cluster` | Embedding-based clustering |
| `.compact` | Token-budgeted summarization |
| `.derive` | Prompt building, proposal application, depth/tokenization utils |
| `.review` / `.review_justifications` / `.review_premises` | LLM review logic |
| `.repair` / `.repair_premises` | LLM repair logic |
| `.contradictions` | Contradiction detection |
| `.import_agent` / `.import_beliefs` | External belief import |
| `.export_markdown` / `.export_card` | Export formatting |
| `.build_wiki` | Wiki generation |
| `.report_gated` / `.report_belief` | Report generation |
| `.verify` | Source verification |
| `.ask` | Source chunk search |
| `.llm` (`invoke_model`) | LLM invocation |

### Importers (what depends on this file)

- `reasons/cli.py` — CLI command handlers call these functions directly
- `reasons/mcp_server.py` — MCP tool definitions wrap these as callable tools
- `reasons/ask.py`, `reasons/contradictions.py`, `reasons/derive.py`, `reasons/repair.py`, `reasons/pg.py` — internal cross-calls
- `reasons_lib/api.py` — appears to be a re-export or compatibility shim
- ~50 test files in `tests/`

## 5. Flow

### Typical Write Operation (e.g., `add_node`)

```
1. Check pg_conninfo → dispatch to PG if set
2. Parse justification strings (sl/cp/unless) into Justification objects
3. Enter _with_network(db_path, write=True)
   a. Bypass cache → Storage(db_path).load() → fresh Network
   b. If pubsub has subscribers: snapshot all truth values (_before)
4. Namespace resolution: prefix node_id, ensure ns:active exists
5. Resolve namespace in all antecedent/outlist references
6. Validate source_type
7. Build metadata dict (access_tags, example, source_type, accepted_pr)
8. Call net.add_node(...) → TMS propagates truth values
9. Build result dict
10. Exit context:
    a. store.save(network) → write to SQLite
    b. Compute pubsub diff (before vs after truth values)
    c. pubsub.publish(events, db_path)
    d. Update _network_cache[db_path] = (new_mtime, network)
    e. store.close()
11. Return result dict
```

### Search Flow (`search`)

```
1. Load network (read path — may hit cache)
2. Determine search strategy: regex | OR-pipe | FTS5 | substring fallback
3. Execute search → matched_ids
4. Filter: exclude OUT (unless include_out), namespace, visible_to, tags
5. Sort: relevance (FTS rank) | created_at | updated_at
6. Paginate: offset, limit
7. BFS neighbor expansion (depth hops along antecedents + dependents)
8. Format output: markdown | json | minimal | compact | names-only
```

### Proposal Accept Flow (`accept_proposal`)

```
1. Load proposal row (must be status='pending')
2. Re-validate against live network:
   - retract: node exists, is IN, truth_value unchanged, text_hash unchanged
   - supersede: old exists, is IN, new_id not taken
   - add: target_id not yet taken
   - nogood: all nodes exist
3. If drifted → mark stale, return
4. Execute the actual mutation (retract_node / supersede_with_text / add_node / add_nogood)
5. Update proposal row: status='accepted', result_json, resolved_by
6. If rowcount=0 → concurrent race, return stale
7. Return applied result
```

## 6. Invariants

1. **JSON-serializable return**: Every public function returns a `dict` (or `str` for text exports) that can be directly passed to `json.dumps`. No custom objects, no `None`-only returns.

2. **Access control is a filter, not a wall**: `_is_visible` is applied at query time. The underlying data is still in the DB. `show_node` raises `PermissionError` rather than returning a redacted view — callers must handle this.

3. **Namespace isolation via premise dependency**: All beliefs in namespace `X` depend on premise `X:active`. Retracting `X:active` cascades OUT every belief in that namespace. This is a **structural** invariant, not a metadata check.

4. **What-if is truly read-only**: `what_if_retract`, `what_if_assert`, `what_if_supersede` use `copy.deepcopy` and never call `store.save`. The DB file's mtime is unchanged after these calls.

5. **Proposals never mutate truth values**: The entire proposal lifecycle (`propose_*`, `accept_proposal`, `reject_proposal`, `withdraw_proposal`) only touches the `proposals` SQLite table and the network (on accept). A pending proposal has zero effect on the belief network.

6. **Cache coherence**: The `_network_cache` is keyed by `db_path` string. If the file is replaced (not just modified), the mtime check catches it. But if a process modifies the DB and the mtime happens to be the same (sub-second resolution), the cache could be stale. `clear_network_cache()` is the manual escape hatch.

7. **Source type validation**: `add_node` rejects `source_type` values not in `SOURCE_TYPES = {"code", "document", "self-description", "derived"}` with a `ValueError`.

8. **Defeaters are graph-native**: `defeat_justification` and `defeat_with_scope` do NOT set metadata flags. They create actual nodes in the outlist. Retracting the defeater node automatically restores the belief — no cleanup code needed.

## 7. Error Handling

| Error Type | When | Example |
|------------|------|---------|
| `KeyError` | Node not found in network | `show_node("nonexistent")` → `KeyError: Node 'nonexistent' not found` |
| `PermissionError` | Access tags not satisfied | `show_node(id, visible_to=["public"])` when node requires `["internal"]` |
| `ValueError` | Invalid argument | `add_node` with bad `source_type`; `propose_nogood` with <2 IDs; `mark_duplicate` with same source/canonical |
| `FileExistsError` | `init_db` without `force=True` on existing DB | |
| `FileNotFoundError` | Import file doesn't exist | `import_beliefs("/missing/file.md")` |
| `NotImplementedError` | PG backend doesn't support a feature | `add_node(any_mode=True, pg_conninfo=...)` |
| `IndexError` | Justification index out of range | `defeat_justification(id, 5)` when node has 2 justifications |
| `re.error` → `ValueError` | Invalid regex in search | `search("(unterminated", regex=True)` |
| `sqlite3.OperationalError` | FTS5 table missing | Caught in `_fts_search`, falls back to `_substring_search` |
| `json.JSONDecodeError` | LLM returns malformed JSON | Caught in `list_negative` batch parsing, skipped silently |
| `Exception` (broad) | Per-node errors in bulk operations | `export_api` catches per-node push failures, increments `errors` counter |

**Notable swallowing**:
- `list_negative` wraps `json.loads` in a `try/except json.JSONDecodeError: continue` — a batch that fails to parse is silently skipped.
- `export_api` catches `Exception` per-node and logs a warning, continuing with the next node.
- `review_premises` catches `OSError, UnicodeDecodeError` on source file reads, skipping that premise.
- The `import_json` topological sort has a "no progress" escape: if a full pass adds zero new nodes (circular deps), it force-adds all remaining nodes.

**What's NOT caught**: The `_with_network` context manager does not catch exceptions from the body. They propagate to the caller. The `__exit__` method only suppresses the exception if `write=False` (it returns `False`). For `write=True` with an exception, it skips save/pubsub but still closes the connection.

---

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The context manager is the single gateway to all network mutations. Understanding its cache interaction, pubsub publishing, and exception semantics is critical before modifying any write path.
- [file] `reasons/network.py` — Contains `Network`, `Node`, `Justification`, and the TMS propagation engine (`retract`, `assert_node`, `recompute_all`, `explain`, `add_nogood`). The API file is a thin shell around these.
- [file] `reasons/pg.py` — The PostgreSQL backend implementation. Comparing its method signatures against the SQLite paths in `api.py` reveals which features are unsupported and how the dispatch contract works.
- [file] `reasons/pubsub.py` — The event publishing system. Understanding `compute_changes` and `has_subscribers` explains how multi-process coordination works when multiple agents share a single `reasons.db`.
- [general] `proposal-workflow` — The propose → accept/reject/withdraw lifecycle spans `api.py`, the `proposals` SQLite table, and the CLI. Tracing a full cycle from `propose_retraction` through `accept_proposal`'s drift detection reveals the multi-agent coordination model.

## Beliefs

- `what-if-simulation-is-read-only` — `what_if_retract`, `what_if_assert`, and `what_if_supersede` use `copy.deepcopy` and never invoke `Storage.save`; the DB file's mtime is unchanged after any what-if call.
- `namespace-isolation-is-structural` — Belief isolation across agents is enforced by a prerequisite dependency on `{namespace}:active` premise node in the justification graph, not by row-level access control or metadata filtering.
- `proposal-accept-revalidates-against-live-state` — `accept_proposal` compares the stored `snapshot_json` (truth_value, text_hash) against the current network and marks the proposal `stale` if either has drifted, preventing stale mutations in multi-agent scenarios.
- `defeaters-are-graph-native-nodes` — `defeat_justification` creates a real premise node in the justification outlist rather than setting a metadata flag; retracting the defeater node automatically restores the original belief through standard TMS propagation.
- `api-functions-return-json-serializable-dicts` — Every public function in `api.py` returns a `dict` or `str` suitable for `json.dumps`; no function returns a `Network`, `Node`, or other internal object directly to the caller.
