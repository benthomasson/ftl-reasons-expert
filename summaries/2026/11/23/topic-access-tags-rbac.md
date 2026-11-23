---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: `0.18.0` documents inheritance, forward propagation, `PermissionError` on single-node reads, and nogood filtering in exports; trace how `--visible-to` is threaded through `show`/`list`/`search`/`export` in `reasons/cli.py` and `reasons_lib/storage.py`.

# `--visible-to` in `0.18.0`: What the Evidence Supports

## Important caveat before tracing

The observation bundle contains **zero matches** in `reasons/cli.py`, `reasons_lib/storage.py`, or any test for the token `visible_to` / `--visible-to` in executable Python (the `locate_visible_to_in_cli`, `locate_cli_commands`, `locate_visible_to_and_permission_in_storage`, `locate_storage_methods`, and `locate_access_behavior_in_tests` queries all returned `total_count: 0`). Only two files were actually read:

- `CHANGELOG.md` (top 139 lines) — feature documentation.
- `tests/test_access_tags.py` (350 lines) — behavioural contract via the public `reasons.api` and `reasons.network.Network` surfaces.

So the concrete call sites in `reasons/cli.py` and `reasons_lib/storage.py` **cannot be line-traced from this bundle**. What follows is (a) the documented feature set, (b) the contract the tests enforce on the API, and (c) the inferred threading path that the CLI and storage layer must implement to satisfy those tests. Each is labelled as observed vs. inferred.

---

## 1. What `0.18.0` documents (`CHANGELOG.md`)

| Line | Claim |
|---|---|
| 7 | Release header `## [0.18.0] - 2026-04-25` |
| 10 | Adds `access_tags` metadata for data-source provenance filtering (RBAC) |
| 11 | Adds `--visible-to` flag on **all** read commands: `show`, `list`, `search`, `lookup`, `status`, `explain`, `trace`, `export`, `export-markdown`, `compact` |
| 12 | Adds `trace-access-tags` for auditing tag provenance |
| 13 | Adds `--access-tags` flag on `add` |
| 14 | **Tag inheritance**: derived nodes inherit the *union* of parent access tags |
| 15 | **Forward tag propagation**: adding justifications cascades tags to existing dependents |
| 16 | **`PermissionError` on single-node queries** (`show`, `explain`, `trace`) when access denied |
| 17 | **Nogood filtering in exports**: nogoods referencing restricted nodes are excluded |

The user's question bundles four of these: inheritance (14), forward propagation (15), `PermissionError` (16), and nogood filtering (17). All four are observably tested in `tests/test_access_tags.py`.

---

## 2. Inheritance and forward propagation (observed in `tests/test_access_tags.py`)

These run against `Network` directly, so they pin down the *semantics* that `storage.py` must persist:

- `test_derived_inherits_single_parent_tags` — derived node gets `["finance"]` from single parent.
- `test_derived_inherits_union_of_parents` — derived node with antecedents `a`(`finance`) and `b`(`hr`) gets `["finance", "hr"]`.
- `test_derived_merges_explicit_and_inherited_tags` — explicit `["ops"]` + inherited `["finance"]` → `["finance", "ops"]`.
- `test_premise_does_not_inherit` — a node with no justifications keeps no `access_tags` key.
- `test_chain_inheritance` / `test_diamond_inheritance` — transitive inheritance through chains and diamonds.
- `test_add_justification_updates_tags` — adding a justification to `c` merges the new antecedent's tags.
- `test_add_justification_propagates_tags_to_dependents` — this is the **forward propagation** case: `a` initially untagged, `b`→`a`→`c`. Adding a new justification to `a` from `secret`(`finance`) causes `a`, `b`, **and** `c` all to gain `["finance"]`. The assertion is that propagation reaches *existing* dependents, not just future ones.
- `test_tags_persist_through_storage` — round-trips through `api.add_node` → `api.show_node` with `db_path`, confirming the SQLite layer stores and reloads inherited tags.

Note: this test does **not** exercise tag *removal* (e.g., retracting a justification). So whether `storage.py` recomputes tags on retraction is not covered by these observations.

---

## 3. `--visible-to` contract per command (observed via `TestVisibleTo`)

All assertions go through `reasons.api.*` functions. Because the changelog (line 11) says `--visible-to` is on the CLI commands, the pattern `cli → api → storage` is the only path consistent with these tests. Each test names the *api* function that the CLI command must dispatch to.

### Filtering semantics — shared predicate

From `test_list_nodes_subset_match`, `test_list_nodes_multi_tag_requires_all`, and `test_list_nodes_untagged_always_visible`:

- A node is visible iff **every** tag in `node.metadata["access_tags"]` is present in the caller's `visible_to` set — i.e. `set(node_tags) ⊆ set(visible_to)`. This is confirmed by `test_list_nodes_multi_tag_requires_all`: a node tagged `["finance","hr"]` is hidden from `visible_to=["finance"]` and shown only when both are supplied.
- Untagged nodes are always visible regardless of `visible_to`.
- `visible_to=None` (default) → no filtering at all (`test_list_nodes_visible_to_none_shows_all`, `test_show_node_no_filter`).

### Per-command behaviour

| CLI command | API entry point tested | Failure mode |
|---|---|---|
| `show` | `api.show_node(id, visible_to=...)` | **Raises `PermissionError`** if denied (`test_show_node_raises_on_forbidden`); returns node otherwise (`test_show_node_allowed`) |
| `list` | `api.list_nodes(visible_to=...)` | **Silently filters**, returns `{nodes, count}` (`test_list_nodes_filters_by_access`) |
| `search` | `api.search(query, visible_to=...)` | **Silently filters**, and **also filters neighbour expansion** – a public node whose antecedent is restricted must not leak the antecedent's body (`test_search_neighbors_filtered`) |
| `export` | `api.export_network(visible_to=...)` | **Silently filters nodes**, and additionally **filters nogoods** (see §4) |
| `lookup` | `api.lookup(term, visible_to=...)` | Silently filters (`test_lookup_respects_visible_to`) |
| `status` | `api.get_status(visible_to=...)` | Silently filters, and `total` recomputed over the filtered set (`test_status_respects_visible_to`) |
| `explain` | `api.explain_node(id, visible_to=...)` | **Raises `PermissionError`**; also inherits restrictions because derived nodes carry parent tags (`test_explain_inherits_restriction_from_antecedent`) |
| `trace` | `api.trace_assumptions(id, visible_to=...)` | **Raises `PermissionError`**; premise list also filtered (`test_trace_raises_on_forbidden`, `test_trace_filters_premises`) |
| `export-markdown` | `api.export_markdown(visible_to=...)` | Silently filters (`test_export_markdown_respects_visible_to`) |
| `compact` | `api.compact(visible_to=...)` | Silently filters (`test_compact_respects_visible_to`) |
| `trace-access-tags` | `api.trace_access_tags(id, visible_to=...)` | Also raises `PermissionError` (`test_api_trace_access_tags_raises_on_forbidden`) |

The **single-node vs. multi-node** split (changelog line 16; `PermissionError` only on `show`/`explain`/`trace`) is the key error-semantics contract. Multi-node reads never raise; they drop rows.

---

## 4. Nogood filtering in exports (observed)

`test_export_network_filters_nogoods`:

```
pub  (no tags)
fin  (finance)
nogood ["pub", "fin"]
```

- `api.export_network(visible_to=["public"])` → `len(data["nogoods"]) == 0`
- `api.export_network(visible_to=["finance","public"])` → `len(data["nogoods"]) == 1`

The rule: a nogood is included in the export **only if every node it references is visible** under `visible_to`. That prevents a nogood from leaking the existence of a restricted node id even when the restricted node itself is filtered out of the nodes list.

---

## 5. Inferred threading path (not observable here)

Given the tests above, the required wiring is:

1. **`reasons/cli.py`** — each read subcommand (`show`, `list`, `search`, `export`, plus the other nine listed on changelog line 11) must declare a `--visible-to` argparse argument (most naturally a comma/space-delimited list of tags, defaulting to `None`) and pass it into the corresponding `reasons.api.*` function along with `db_path`.
2. **`reasons_lib/api.py`** — the API functions (`show_node`, `list_nodes`, `search`, `export_network`, `export_markdown`, `lookup`, `get_status`, `explain_node`, `trace_assumptions`, `compact`, `trace_access_tags`) must accept `visible_to=` and forward it.
3. **`reasons_lib/storage.py`** — the SQL/Python read layer must apply the subset predicate `set(node_tags) ⊆ set(visible_to)` when materialising nodes, and must *also* apply it to associated collections (search neighbours, trace premise lists, export nogoods) — otherwise the specific tests in §3 and §4 would fail.
4. **Error semantics** — storage should raise `PermissionError` for `get_node` (used by `show`/`explain`/`trace`) when the node is present but denied, while `list`/`search`/`export` path should filter silently.

**What's missing to confirm any of this:** the actual contents of `reasons/cli.py`, `reasons_lib/storage.py`, and `reasons_lib/api.py`. The pattern queries for those files returned zero matches, so we cannot assert *where* the subset test lives, whether it is SQL (`WHERE access_tags <@ ...`) or Python post-filtering, or which argparse type the flag uses. Until those files are pulled, step 5 is inference from the public contract, not a line trace.

---

## Topics to Explore

- [file] `reasons/cli.py` — Locate the argparse declarations for each read subcommand to confirm exactly which commands expose `--visible-to` and its parsing type (comma-separated vs. space-separated).
- [file] `reasons_lib/storage.py` — Find where the `visible_to` filter is applied; check whether subset matching is done in SQL or in Python after loading, and whether all three of `nodes`, `nogoods`, and neighbour expansions are guarded.
- [function] `reasons_lib/api.py:export_network` — Verify the nogood-filtering branch (the test only proves behaviour, not the exact predicate — is it "all referenced nodes visible" or "no referenced node restricted"?).
- [general] `tag-recompute-on-retraction` — The tests cover forward propagation on `add_justification` but not tag removal on `retract`; inspect whether `Network.retract` recomputes `access_tags` or leaves stale tags.
- [general] `partial-access-leakage` — Consider whether a caller with a *subset* of a derived node's tags can still see the derived node's text/metadata during `search`/`list`, since `multi_tag_requires_all` only guarantees exclusion, not the inverse.

---

## Beliefs

- `visible-to-subset-match` — A node passes the `visible_to` filter iff the set of its `access_tags` is a subset of the caller's `visible_to`; a node tagged `["finance","hr"]` is hidden when only `["finance"]` is supplied (`tests/test_access_tags.py::test_list_nodes_multi_tag_requires_all`).
- `untagged-nodes-always-visible` — Nodes with no `access_tags` key are returned regardless of the `visible_to` argument (`test_list_nodes_untagged_always_visible`).
- `single-node-reads-raise-permissionerror` — `show`, `explain`, and `trace` (and `trace-access-tags`) raise `PermissionError` on access denial, while `list`, `search`, `lookup`, `status`, `export`, `export-markdown`, and `compact` filter silently (`test_show_node_raises_on_forbidden` vs. `test_list_nodes_filters_by_access`).
- `export-filters-nogoods-referencing-restricted-nodes` — `export_network` omits any nogood that references a node invisible under `visible_to` (`test_export_network_filters_nogoods`).
- `tag-inheritance-is-union-of-parents` — A derived node's `access_tags` equals the union of its justifications' antecedents' tags plus any explicit tags it holds (`test_derived_inherits_union_of_parents`, `test_derived_merges_explicit_and_inherited_tags`).
- `add-justification-propagates-tags-to-existing-dependents` — Adding a justification with new tags cascades those tags to nodes already dependent on the retagged node (`test_add_justification_propagates_tags_to_dependents`).
