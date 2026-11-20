---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How `defeater_type`, `defeat_reason_type`, and `defeats_node` are produced and consumed across the codebase.

## How `defeater_type`, `defeat_reason_type`, and `defeats_node` flow through the codebase

These three strings are the contract that ties a *defeater node* to the *node it defeats*. They are written when defeaters are created (or migrated), then read back by classifiers and exporters.

### 1. `defeats_node` — the pointer from defeater → defeated

This is the load-bearing field. Without it, a node is not recognized as a defeater anywhere downstream.

**Produced (written into `metadata`):**

- `reasons/api.py:1380` — inside `defeat_justification(...)`: `"defeats_node": node_id` (the node whose justification is being defeated).
- `reasons/api.py:1507` — same in the sibling function (the scope-aware variant around 1419–1543).
- `reasons/api.py:1622` — inside `migrate_retract_to_defeaters`: `"defeats_node": nid` (the originally-retracted node that becomes the defeater's target).

**Consumed:**

- `reasons/api.py:1661, 1683` — `classify_defeat_reason_types` uses `meta.get("defeats_node")` as the definition of "this node is a defeater." Nodes without it are skipped.
- `reasons/api.py:1699` — `defeated_id = meta.get("defeats_node", "")` is then fetched via `get_node(...)` so the LLM has both sides of the defeat to classify.
- `reasons/build_wiki.py:90` — `if o_meta.get("defeats_node") == node_id:` — scanning other nodes to find who defeats the current page's node.
- `reasons/export_markdown.py:86` — same "reverse lookup" pattern for the markdown exporter.
- Tests (`tests/test_defeat_justification.py:42`, `test_defeat_with_scope.py:91`, `test_migrate_defeaters.py:30`, `test_build_wiki.py:185, 210, 230`, `test_export_markdown.py:93, 110, 126`) all assert this pointer is set to the defeated node's ID.

> Note: the wiki/markdown consumers treat `defeats_node` as a *reverse* edge — they iterate the network looking for a node whose `defeats_node` equals the current node's ID.

---

### 2. `defeater_type` — the category baked into the defeater's identity

**Produced:**

- Default `"invalid-inference"` in the `defeat_justification` signatures at `reasons/api.py:1326` and `reasons/api.py:1419`.
- Embedded in the defeater's ID at `reasons/api.py:1372` and `reasons/api.py:1470`:
  ```python
  defeater_id = f"{defeater_type}-{node_id}-j{justification_index}"
  ```
  So `defeater_type` is not just metadata — it participates in the primary key. Changing it renames the defeater.
- Written to `metadata` at `reasons/api.py:1379` and `reasons/api.py:1506`:
  ```python
  "defeater_type": defeater_type,
  ```
- Hard-coded to `"migrated-retraction"` in `migrate_retract_to_defeaters` at `reasons/api.py:1621` — producing IDs like `migrated-retraction-derived-j0` (see `tests/test_migrate_defeaters.py:32`).
- CLI passthroughs at `reasons/cli.py:208, 260, 2493, 2500` default to `"invalid-inference"` when `--type` isn't given (`reasons/cli.py:3163`).

**Consumed:**

- `reasons/api.py:1688–1689` — `classify_defeat_reason_types` filters candidates by `defeater_type_filter`; anything with a non-matching `defeater_type` is recorded in `skipped` with the reason string.
- `reasons/build_wiki.py:94` — `d_type = d_meta.get("defeater_type", "defeater")` (falls back to the literal `"defeater"` for legacy nodes).
- `reasons/export_markdown.py:87` — same fallback used for rendering.
- `reasons/cli.py:217, 270` and `reasons/api.py:1408, 1542` — echoed back in the API/CLI result dict so callers can confirm what was written.

---

### 3. `defeat_reason_type` — the LLM-classified failure mode (post-hoc)

Unlike the other two, this is **not** required at defeater-creation time, and it is written conditionally.

**Produced (at creation, optional):**

- Default `""` at `reasons/api.py:1328, 1421`.
- Only stored if truthy — `reasons/api.py:1383–1384` and `1510–1511`:
  ```python
  if defeat_reason_type:
      meta["defeat_reason_type"] = defeat_reason_type
  ```
  So a defeater created without `--reason-type` has **no** `defeat_reason_type` key at all (not an empty string).
- Ignored in the defeater ID — it is pure metadata, unlike `defeater_type`.

**Produced (post-hoc classification):**

- `reasons/api.py:1652` — `classify_defeat_reason_types(...)` is the batch job that fills it in.
- Per `reasons/api.py:1662` docstring, it finds defeater nodes, sends each pair `(defeater text, defeated text)` to an LLM, and writes the classification back.
- `reasons/api.py:1685` — already-classified defeaters are skipped whenever `meta.get("defeat_reason_type")` is truthy (verified by `tests/test_classify_defeaters.py:test_skips_already_classified`).
- `reasons/api.py:1712, 1715` — after classification, appends `{"id": nid, "defeat_reason_type": reason_type}` to `classified` and calls `set_metadata(nid, "defeat_reason_type", reason_type, ...)`.
- Underlying classifier is `reasons.review.classify_defeat_reason`, which parses a free-form LLM response into one of `DEFEAT_REASON_TYPES` (7 values; `tests/test_classify_defeaters.py:TestDefeatReasonTypes`).

**Consumed:**

- `reasons/build_wiki.py:95` — `r_type = d_meta.get("defeat_reason_type", "")` (empty fallback for unclassified defeaters).
- `reasons/export_markdown.py:88` — same.
- `reasons/cli.py:218, 271` and `reasons/api.py:1409, 1543` — echoed back in result dicts.
- `reasons/review.py:47` — used as the default `""` in the classification record template.

---

### End-to-end picture

1. **Create** a defeater: `defeat_justification(...)` (`api.py:1326`/`1419`) writes `defeats_node` and `defeater_type`, builds the defeater ID from `defeater_type`, and (optionally) writes `defeat_reason_type`.
2. **Migrate** an old retraction: `migrate_retract_to_defeaters` (`api.py:~1621`) creates a defeater with `defeater_type="migrated-retraction"` and `defeats_node=<prior OUT node>`.
3. **Enrich**: `classify_defeat_reason_types` (`api.py:1652`) iterates nodes, uses `defeats_node` to find targets, filters by `defeater_type`, skips those already carrying `defeat_reason_type`, calls the LLM, and writes back via `set_metadata` (`api.py:1712–1715`).
4. **Present**: `build_wiki.py:90–95` and `export_markdown.py:86–88` do a reverse lookup on `defeats_node == node_id` and render `defeater_type` / `defeat_reason_type` on the defeated node's page.

### Gaps in the observations

- The full bodies of `defeat_justification` (around `api.py:1326–1410` and `1419–1545`) are only visible as fragments — the exact `Justification` construction and `set_metadata` call could not be seen.
- The full list `DEFEAT_REASON_TYPES` (7 entries; tests only confirm count and hyphenation) is not shown.
- Whether `defeater_type` values are validated against an enum is not visible; the docstring says "`invalid-inference`, `over-generalizes`, etc." suggesting free-form.
- `reasons/metadata.py` (which likely documents `SCHEMA_VERSION` / `build_meta`) is not shown, so it is unknown whether these three keys are schema-registered.

## Topics to Explore

- [function] `reasons/api.py:defeat_justification` — Confirm the exact ID-construction and which metadata keys are always vs. conditionally written.
- [function] `reasons/api.py:classify_defeat_reason_types` — See the full filtering/skip/error-handling loop that drives `defeat_reason_type` production.
- [file] `reasons/review.py` — Home of `DEFEAT_REASON_TYPES` (7 values) and the LLM-call wrapper `classify_defeat_reason`.
- [file] `reasons/metadata.py` — Check whether `defeater_type`, `defeat_reason_type`, and `defeats_node` are schema-registered or ad-hoc.
- [general] `defeater-vs-justification-model` — Understand how `Justification` rows vs. defeater nodes interact (two ways to defeat a belief in this system).

## Beliefs

- `defeater-type-affects-defeater-id` — `defeater_type` is embedded in the defeater node ID (`f"{defeater_type}-{node_id}-j{justification_index}"` at `reasons/api.py:1372, 1470`), so it is part of node identity, not just metadata.
- `defeats-node-enables-defeater-classification` — `classify_defeat_reason_types` requires `metadata["defeats_node"]` to identify defeater nodes and to fetch the defeated text for the LLM (`api.py:1683, 1699`).
- `defeater-type-enables-classification-filtering` — `classify_defeat_reason_types` accepts `defeater_type_filter` and skips candidates whose `defeater_type` does not match (`api.py:1688–1689`), as covered by `tests/test_classify_defeaters.py:test_filters_by_defeater_type`.
- `defeat-reason-type-written-conditionally` — `defeat_justification` only adds `defeat_reason_type` to metadata when the argument is truthy (`api.py:1383–1384, 1510–1511`), unlike `defeater_type`/`defeats_node` which are always written.
- `migrated-retraction-produces-defeater` — `migrate_retract_to_defeaters` produces defeaters with `defeater_type="migrated-retraction"` and `defeats_node=<retracted node>` (`api.py:1621–1622`), enabling reversibility (see `tests/test_migrate_defeaters.py:test_reversibility`).
