---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# File: reasons/api.py

# `reasons/api.py` — Explanation for a Maintainer

## Purpose

`reasons/api.py` is the **single functional facade** over the Reason Maintenance System (RMS). Everything user-facing — the CLI (`reasons/cli.py`), the MCP server (`reasons/mcp_server.py`), LangGraph/tool integrations, and the ~50 test files — calls functions in this module rather than touching `Storage` or `Network` directly.

The module's stated contract (docstring, top of file): *"Each function opens the database, operates, saves, and closes. All functions return dicts suitable for JSON serialization."* This is the design intent: a **stateless function surface** over a stateful graph core, with no `argparse` / CLI concerns.

At ~4,000 lines it is not one "thing" — it's the union of ~90 public functions organized loosely by concern: node CRUD, TMS operators (retract/assert/challenge/defend/nogood), tag & source management, search, export/import, staleness & Merkle (hash) integrity, LLM-driven review/repair/propose flows, and the proposals state machine.

---

## Key Components

### Module-level infrastructure

| Symbol | Role |
|---|---|
| `DEFAULT_DB = "reasons.db"` | Default database path used by every function signature. |
| `_network_cache: dict[str, tuple[float, Network]]` | Maps `db_path → (mtime, Network)`. Populated on `_with_network.__exit__`; invalidated by `clear_network_cache()`. |
| `_with_network(db_path, write=False)` | Custom context manager. Loads, yields the `Network`, closes and (conditionally) saves on exit. |
| `_pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs)` | Late-bound router into `reasons/pg.py`'s `PgApi` for the Postgres backend. |
| `SOURCE_TYPES = {"code","document","self-description","derived"}` | Whitelist validated in `add_node`. |

### `_with_network` (the load-bearing idiom)

Its `_Ctx` class:

- **`__init__`** — on `write=False`, tries the cache; if `os.path.getmtime(db_path)` matches the cached mtime, reuses the cached `Network` object. Otherwise opens a fresh `Storage` and calls `store.load()`.
- **`__enter__`** — if `write=True` **and** `pubsub.has_subscribers(db_path)`, snapshots `{nid: truth_value}` so it can diff after the write.
- **`__exit__`** — on success and `write=True`, saves the network and publishes computed truth-value deltas via `pubsub.compute_changes` / `pubsub.publish`. Closes the storage. Then — unless this was a write that raised — updates `_network_cache[db_path] = (mtime, self.network)`. **Raising while `write=True` means neither save nor cache update**.

This is the mechanism behind read-through caching: repeated reads on a stable file get the same `Network` object back (this is why `what_if_*` calls `copy.deepcopy(net)` before mutating — see below).

### Helper predicates

- **`_is_visible(node, visible_to)`** — a node's `access_tags` must be an **empty set or a subset** of `visible_to`. Empty tags ⇒ always visible.
- **`_get_node_tags(db_path, node_ids)`** — direct SQL on `node_tags`; returns `{node_id: [tag]}`. Uses `?`-placeholders; a `None` list means "no filter needed".
- **`_filter_by_tags(node_ids, tag_filters, db_path)`** — AND across filter keys, OR within values. Special key `"tag"` is exact-match; other keys are auto-prefixed (`topic:networking`) and matched by containment.
- **`_resolve_namespace(node_id, ns)`** — only prefixes when `":" not in node_id`, allowing cross-namespace references to stay untouched.
- **`_cascade_depth(net, target_id, retracted_id)`** — BFS over `.dependents` to compute how far the cascade propagated.
- **`_node_depth(nid, net, memo)`** — memoized recursion over supporting justification antecedents; used by `list_nodes` depth filters.

### Public API grouped by concern

**Lifecycle:** `init_db`, `ensure_namespace`, `list_namespaces`, `clear_network_cache`.

**Node & justification CRUD:** `add_node`, `add_justification`, `remove_justification`, `retract_node`, `assert_node`, `propagate`, `convert_to_premise`, `supersede`, `supersede_with_text`, `mark_duplicate`, `mark_superseded`, `summarize`.

**TMS operators:** `challenge`, `defend`, `find_culprits`, `add_nogood`, `defeat_justification`, `defeat_with_scope`, `migrate_retract_to_defeaters`, `classify_defeat_reason_types`.

**Queries/simulations:** `get_status`, `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`, `get_belief_set`, `get_log`, `list_nodes`, `list_gated`, `topics`, `list_clusters`, `list_negative`, `what_if_retract`, `what_if_assert`, `what_if_supersede`.

**Search:** `lookup`, `search` + internal formatters (`_format_markdown`, `_format_json`, `_format_minimal`, `_format_compact`, `_format_names_only`) and search tools (`_fts_search`, `_or_search`, `_regex_search`, `_substring_search`, `_fts_query`).

**Tags / sources / metadata:** `add_tags`, `remove_tags`, `get_tags`, `add_source`, `set_metadata`.

**Export / import:** `export_network`, `export_markdown`, `export_card`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `import_hf`, `publish_hf`, `import_api`, `export_api`.

**Provenance & staleness:** `check_stale`, `hash_sources`, `check_integrity`, `backfill_hashes`, `pin_sources`, `pin_update`, `pin_lines`.

**LLM-assisted pipelines:** `derive_prompt`, `derive_apply`, `review_beliefs`, `review_justifications`, `review_premises`, `verify_belief`, `propose_update`, `store_llm_proposals`, `repair_premises`, `repair_smuggled`, `repair` (alias `research`), `detect_contradictions`.

**Reports:** `report_gated`, `report_belief`, `compact`, `build_wiki`.

**Plan files (human-in-the-loop):**

- Contradictions: `write_contradiction_plan`, `parse_contradiction_plan`, `apply_contradiction_plan`.
- Dedup: `deduplicate`, `write_dedup_plan`, `parse_dedup_plan`, `apply_dedup_plan`, plus `_rewrite_dependents`.

**Proposals state machine:**

- Writers: `propose_retraction`, `propose_supersession`, `propose_addition`, `propose_nogood`.
- Readers: `list_proposals`, `show_proposal`.
- State changers: `accept_proposal`, `reject_proposal`, `withdraw_proposal`.

---

## Patterns

1. **Functional facade over an OOP core.** `Storage` and `Network` are stateful; every public function recreates the load/save/close cycle. This is why the file is so wide — it's intentionally a flat API surface.

2. **Read-through cache keyed on (path, mtime).** `_with_network` reuses the same `Network` instance across reads on an unchanged file. This is why `what_if_*` does `net = copy.deepcopy(net)` after loading — the mutation must not leak into the cached object. The same caution shows up in `store_llm_proposals` and `accept_proposal` which use `_with_network(db_path, write=False)` purely to read current state, then call `add_node` / `retract_node` (which open their *own* write contexts).

3. **Dual backend dispatch.** Nearly every function begins with:
   ```python
   if pg_conninfo:
       return _pg_dispatch(pg_conninfo, domain_id, "<name>", ...)
   ```
   This is a **blocking guard**, not a try/fallback. Postgres support is partial; several functions raise `NotImplementedError` outright (e.g., `add_tags`, `get_tags`, `add_source`, `mark_duplicate`, `defeat_justification`, `defeat_with_scope`, `migrate_retract_to_defeaters`, `classify_defeat_reason_types`, `pin_*`, `check_integrity`, `backfill_hashes`, `store_llm_proposals`). Some functions silently degrade (e.g., `search` refuses `depth != 1` or `tag` filters). `list_nodes` explicitly collects the unsupported kwargs and raises one aggregated `NotImplementedError` at the top.

4. **Pub/sub on successful writes.** `_with_network.__exit__` diffs truth values via `pubsub.compute_changes` and calls `pubsub.publish` **only if `pubsub.has_subscribers(db_path)`** — subscribers are not required, so writes work standalone. Read-only contexts never publish.

5. **Deferred imports to break cycles.** Feature modules (`verify`, `review`, `review_premises`, `repair`, `contradictions`, `cluster`, `ask`, `llm`, `derive`, `build_wiki`, `export_markdown`, `export_card`, `hf`, `mind_service`, `import_agent`, `import_beliefs`, `check_stale`, `merkle`, `compact`, `report_gated`, `report_belief`, `propose_update`, `pg`) are all imported **inside** function bodies. That's what keeps the top-level import list short (only `Justification`, `pubsub`, `build_meta`, `Network`, `PROPOSALS_SCHEMA`, `Storage`).

6. **Progressive query relaxation.** Both `lookup` and `_fts_search` strip stop-words, then if no hits: systematically drop query terms via `itertools.combinations` — bounded to 50 relaxation attempts (`_MAX_RELAXATION = 50` / `_MAX_RELAXATION_QUERIES = 50`). This is why `lookup("foo bar baz qux")` can return matches for `foo bar` when nothing matches all four.

7. **Two-step (dry-run) apply plans.** Contradict and dedup workflows generate a markdown plan file with `[APPLY]`/`[SKIP]` (or `[KEEP]`/`[RETRACT]`) tags; the parser (`parse_contradiction_plan`, `parse_dedup_plan`) turns it back into a list of dicts; `apply_*` executes. This is a deliberate human-in-the-loop pattern so LLM-derived actions aren't auto-applied.

8. **Proposal accept is a re-validate-then-apply transaction.** `accept_proposal` re-opens the network read-only to detect drift (node deleted, truth value flipped, text hash changed, new_id collision) before applying; if drift is found it flips the proposal to `stale` and returns `{"applied": False, "status": "stale", ...}`. The final `UPDATE ... WHERE status='pending'` guards against concurrent accept races (returns a stale result if `rowcount == 0`).

---

## Dependencies

### Imports (top-level)

- `. import Justification` / `. import Nogood` / `. import Node` — dataclasses for the graph model.
- `. import pubsub` — change notification on writes.
- `.metadata.build_meta` — attaches schema_version/project_name/created_at in `export_network`.
- `.network.Network` — core graph object.
- `.storage.{PROPOSALS_SCHEMA, Storage}` — SQLite schema + persistence.
- `.pg.PgApi` — imported lazily in `_pg_dispatch`.

### Lazy imports (inside functions) reveal feature dependencies

- **Git/staleness:** `check_stale` (impure — may read files), `resolve_source_path`, `hash_sources`, `pin_*`, `git`.
- **Integrity:** `merkle.verify_all`, `merkle.backfill_hashes`.
- **Text/UI:** `compact`, `export_markdown`, `export_card`, `report_gated`, `report_belief`, `build_wiki`.
- **LLM/analysis:** `review.review_beliefs`, `review.review_justifications`, `review_premises`, `repair`, `repair_premises`, `contradictions`, `cluster`, `derive`, `propose_update`, `llm.invoke_model`, `ask.search_source_chunks`.
- **Networks:** `hf`, `mind_service`, `import_agent`, `import_beliefs`.

### Imported by

- `reasons/cli.py`, `reasons/mcp_server.py` — the two main front-ends.
- `reasons/ask.py`, `contradictions.py`, `derive.py`, `repair.py`, `pg.py` — feature modules that call back into this API (so cycles exist by design and are resolved via lazy imports).
- `reasons_lib/api.py` — vendored copy or re-export path.
- **~50 test files.** Effectively all end-to-end behavior tests exercise this module.

---

## Flow

The canonical call path for a mutating operation:

```
add_node(node_id, text, sl="a,b", ...)
  ├── if pg_conninfo → _pg_dispatch(...)                   # short-circuit
  ├── parse CSV args into list
  ├── build Justification objects (SL / CP / outlist-only)
  ├── _with_network(db_path, write=True):
  │     ├── __init__:  cache miss → Storage(db_path).load()
  │     ├── __enter__: snapshot truth values if subscribers exist
  │     └── body:
  │           ├── if namespace: prefix IDs, ensure ns:active, insert as antecedent
  │           ├── validate source_type ∈ SOURCE_TYPES
  │           ├── assemble metadata (access_tags, example, source_type, accepted_pr)
  │           └── net.add_node(...) → returns truth_value, premise_count
  └── __exit__:
        ├── store.save(net)                                # write=True, no exception
        ├── pubsub.compute_changes(before, after)          # only if subscribed
        ├── store.close()
        └── update _network_cache[(db_path)] = (mtime, net)
```

Reading path for a query (`get_status`):

```
_with_network(db_path, write=False)
  ├── _Ctx.__init__: mtime == cache → reuse cached Network
  └── body: iterate net.nodes, apply visibility/namespace/filter, aggregate
  └── __exit__: no save, no pubsub, cache refreshed (no-op if hit)
```

Postgres routing is orthogonal: every function starts with `if pg_conninfo: return _pg_dispatch(...)`, which lazily constructs `PgApi(pg_conninfo, domain_id)` and calls `getattr(pg, method_name)(**kwargs)`.

The **`verify_belief`** flow is instructive for the read-side cost model: one `show_node`, optionally one `trace_assumptions` + N× `show_node` for the trace, then `read_source` + `verify_beliefs` (LLM), then a *separate* `_with_network(write=True)` to stamp `verified_at` on confirmed beliefs. That means a single "verify" call may open the DB twice — once for reads (cached), once for the write stamp.

---

## Invariants

**Return-type contract.** Public functions return `dict` (or `list`, `str`) — JSON-serializable. Exceptions to the "dict" rule: `export_markdown`, `export_card`, `compact`, `lookup`, `search` return `str`; `get_belief_set` returns `list[str]`.

**Cache safety.** Reads that yield `Network` must not mutate it. Mutating simulations (`what_if_retract`, `what_if_assert`, `what_if_supersede`) **must** `copy.deepcopy(net)` after loading — omitting this corrupts the cache for subsequent readers.

**Write-then-cache.** `_with_network.__exit__` only updates the cache and saves the DB when no exception was raised during a write. A raised exception inside `write=True` leaves the database untouched **and** leaves the cache stale-but-safe (the in-memory `Network` is discarded).

**Namespace prefixing rule.** `_resolve_namespace(node_id, ns)` prefixes with `f"{ns}:"` **only if** `":" not in node_id. Cross-namespace references are therefore preserved verbatim. `add_node(..., namespace=ns)` also **auto-creates** `ns:active` if missing and inserts it as the first antecedent on the surviving justification.

**`source_type` whitelist.** `add_node` rejects any `source_type` outside `SOURCE_TYPES = {"code","document","self-description","derived"}` with a `ValueError` enumerating valid options.

**`access_tags` normalization.** When set on a node, tags are `sorted(set(...))` before being stored in metadata.

**`add_justification` premise rule.** At least one of `sl`, `cp`, `unless` must be provided; otherwise `ValueError("Must provide --sl, --cp, or --unless")`.

**Proposal IDs.** `_next_proposal_id(conn, target_id, action)` = `count(*) + 1` scoped to that `(target, action)` pair. This is monotonic per pair but **not globally unique across writers** — two concurrent proposers can generate the same ID against the same `(target, action)`. The `INSERT` will not raise (no `UNIQUE` collision is guaranteed by the count, but races are possible); the accept-side `UPDATE ... WHERE status='pending'` still protects against double-accept.

**Proposal lifecycle.** `pending → accepted | rejected | withdrawn | stale`. A new `propose_*` on the same `(target, action)` **auto-stales any older pending** proposal (`_stale_older_pending`). Only `pending` proposals can be accepted/rejected/withdrawn.

**Nogood requires ≥ 2 nodes.** `propose_nogood` raises `ValueError("A nogood requires at least 2 node IDs")`.

**Dedup identifiers.** `deduplicate` defaults to Jaccard similarity over tokenized IDs; `semantic=True` switches to cosine over sentence-transformer embeddings. `auto=True` retracts all but the belief with the most dependents (ties broken by `nid`) and rewrites propagating justifications via `_rewrite_dependents`.

---

## Error Handling

Errors are largely **propagated** (not swallowed) — this is deliberate for a library-facing layer. The dominant patterns:

| Condition | Exception |
|---|---|
| `init_db` on existing path with `force=False` | `FileExistsError` |
| `show_node` / `explain_node` / `trace_*` / `what_if_*` with unknown `node_id` | `KeyError("Node '<id>' not found")` |
| `show_node` etc. when `access_tags ⊄ visible_to` | `PermissionError` |
| `accept_proposal` / `reject_proposal` / `withdraw_proposal` on a non-pending proposal | `ValueError(f"Proposal '{id}' is '{status}', not pending")` |
| `accept_proposal` / `show_proposal` on a missing ID | `KeyError` |
| Any function where `add_node`/`add_justification` receives invalid params | `ValueError` |
| Attempting a Postgres-unsupported op | `NotImplementedError` with a message naming the flag (e.g., `any_mode`, `tag`) |
| `add_node` source_type outside whitelist | `ValueError` |
| `deduplicate`, `find_culprits`, etc. | propagate from `Network` |

**Swallowed / logged:**

- `export_api` — per-node `push_belief` failures are **logged** and counted (`errors`), not raised. A partially-failed push returns `{"nodes_exported": n, "errors": m}`.
- `_fts_search` — catches `sqlite3.OperationalError` and `sqlite3.DatabaseError` and returns `[]`, triggering the `_substring_search` fallback in `search`.
- `list_negative` — invisible JSON decode failures inside the response parsing loop are `continue`d.
- `verify_belief` — when `retract=True`, retraction failures are captured in `retract_failed` / `stamp_failed` fields, not raised.
- `detect_contradictions(auto_apply=True)` — per-contradiction `add_nogood` failures are printed to stderr and skipped.
- `build_wiki` — `show_node` failures are swallowed to skip inaccessible nodes.

**One notable "no-op on race" pattern:** `accept_proposal`'s final `UPDATE ... WHERE status = 'pending'` returns `rowcount == 0` if another process accepted between the read and the write; instead of raising, it returns `{"applied": False, "status": "stale", "reason": "Concurrent accept race..."}`.

---

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class this API wraps: implements `add_node`, `retract`, `assert_node`, `recompute_all`, `supersede`, `challenge`, `defend`, and the TMS semantics the whole system rests on.
- [function] `reasons/api.py:_with_network` — The context manager that owns every load/save/close cycle plus the mtime cache — understanding this one function explains all read/write behavior in the file.
- [file] `reasons/pg.py` — The `PgApi` class that `_pg_dispatch` targets; shows which parts of the API surface have real Postgres support and which just `raise NotImplementedError`.
- [file] `reasons/pubsub.py` — Subscriber registry and `compute_changes` / `publish` logic invoked from `_with_network.__exit__` on every write; determines whether change events are ever emitted.
- [file] `reasons/cli.py` — The primary consumer; maps every public function here to a subcommand and reveals which kwargs are actually exercised end-to-end.

## Beliefs

- `api-functions-open-save-close-db` — Every write-type public function in `api.py` uses `_with_network(db_path, write=True)`, which loads the DB, mutates the in-memory `Network`, saves on successful exit, and closes `Storage`.
- `network-cache-keyed-on-mtime` — `_network_cache` maps `db_path → (mtime, Network)`; read-only calls reuse the cached `Network` only when `os.path.getmtime(db_path)` matches the stored mtime.
- `what-if-simulations-deep-copy-network` — `what_if_retract`, `what_if_assert`, and `what_if_supersede` each call `copy.deepcopy(net)` before mutating, preserving the cached `Network` object for subsequent readers.
- `pg-conninfo-short-circuits-every-public-function` — Every public API function begins with `if pg_conninfo: return _pg_dispatch(...)`, making Postgres a routing decision rather than a try/fallback.
- `proposal-accept-revalidates-for-drift` — `accept_proposal` re-opens the network read-only and checks node existence, truth value, and `text_hash` against the proposal's snapshot before applying; drift flips the proposal to `stale` without applying.
- `namespace-prefix-skips-already-namespaced-ids` — `_resolve_namespace` only prepends `f"{namespace}:"` when the ID contains no `":"`, preserving cross-namespace references verbatim.
- `pubsub-fires-only-on-subscribed-writes` — `_with_network.__exit__` computes and publishes change events only when `write=True`, the context exited without exception, and `pubsub.has_subscribers(db_path)` is truthy.
