---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Status, belief type, defeater, stale, duplicate, and nogood conventions used in the exported format.

## Exported formats at a glance

There are two relevant exporters:

- `reasons/export_markdown.py:export_markdown` produces a `beliefs.md`-compatible registry. This is where the per-node conventions live.
- `reasons/export_card.py:export_card` produces a HuggingFace/EEM `README.md` card. It mostly aggregates counts and does not serialize per-node status/type/defeater bullets.

The per-node markdown header is emitted as:

```markdown
### {node.id} [{status}] {btype}
```

from `reasons/export_markdown.py:62`.

## Status

### Markdown export

`export_markdown` starts with the node’s stored truth value:

```python
status = node.truth_value
```

at `reasons/export_markdown.py:53`. The storage schema defines `nodes.truth_value` with default `'IN'` at `reasons/storage.py:18-21`.

Then there is one export-only override:

```python
if status == "OUT" and (
    node.metadata.get("stale_reason") or node.metadata.get("retract_reason")
):
    status = "STALE"
```

at `reasons/export_markdown.py:54-55`.

So the exported markdown status can be:

- `IN`
- `OUT`
- `STALE`

`STALE` is an export label only. Underlying storage still has `truth_value == "OUT"`. Tests confirm this behavior:

- `test_retracted_node` expects `### a [OUT] OBSERVATION` after `net.retract("a")`.
- `test_stale_metadata` expects `### a [STALE]` when stale metadata exists and the node is retracted.

Sorting also follows status grouping:

```python
nodes = sorted(network.nodes.values(), key=lambda n: (n.truth_value != "IN", n.id))
```

at `reasons/export_markdown.py:48-49`. That means `IN` nodes are exported first, then non-`IN` nodes, alphabetically within each group. Since `STALE` is still `truth_value == "OUT"`, stale nodes sort with the `OUT` group. `test_in_before_out` covers the `IN`/`OUT` ordering.

### EEM card export

`export_card` does not expose `STALE`. It counts:

```python
in_count = sum(1 for n in network.nodes.values() if n.truth_value == "IN")
out_count = total - in_count
```

at `reasons/export_card.py:54-55`.

Those counts become frontmatter fields:

```yaml
beliefs_in: ...
beliefs_out: ...
```

at `reasons/export_card.py:94-100`, and the stats table row:

```markdown
| Status | {in_count} IN / {out_count} OUT |
```

at `reasons/export_card.py:115-121`. Therefore, in the card, a stale node is counted as `OUT`.

## Belief type

In `export_markdown`, the third token in the heading is the belief type:

```markdown
### a [IN] OBSERVATION
### b [IN] DERIVED
```

The exporter reads:

```python
btype = node.metadata.get("beliefs_type", "")
if not btype:
    btype = "DERIVED" if node.justifications else "OBSERVATION"
```

at `reasons/export_markdown.py:57-62`.

Conventions:

- If `metadata["beliefs_type"]` exists, it is preserved verbatim.
  - `test_beliefs_type_preserved` expects `### a [IN] WARNING`.
- If absent:
  - Node with justifications → `DERIVED`
  - Node without justifications → `OBSERVATION`

Tests confirm:

- `test_premise` expects `### a [IN] OBSERVATION`
- `test_derived_node` expects `### b [IN] DERIVED`

`export_card` uses a related but simpler aggregate notion:

```python
premises = sum(1 for n in network.nodes.values() if not n.justifications)
derived = total - premises
```

at `reasons/export_card.py:56-57`, and labels them in stats as:

```markdown
| Premises (observations) | {premises} |
| Derived (justified conclusions) | {derived} |
```

at `reasons/export_card.py:115-121`. The card does not read per-node `beliefs_type`.

## Defeater

Defeaters are derived from `outlist` entries, not from a separate node field in the markdown export.

`export_markdown` iterates every justification’s `outlist`:

```python
for o in j.outlist:
    o_node = network.nodes.get(o)
    if o_node and o_node.metadata.get("defeats_node") == node.id:
        dtype = o_node.metadata.get("defeater_type", "defeater")
        rtype = o_node.metadata.get("defeat_reason_type", "")
        ...
```

at `reasons/export_markdown.py:80-90`.

Classification convention:

- An outlist node is a **defeater** only if:
  - the referenced node exists, and
  - its metadata `defeats_node` equals the current node ID.

If so, it is emitted as:

```markdown
- Defeated by: <id> (<type>[, <reason-type>])
```

via `reasons/export_markdown.py:97-100`.

Examples from tests:

- `defeater_type: "invalid-inference"` → `- Defeated by: defeater-1 (invalid-inference)`
- with `defeat_reason_type: "unsupported-conjunct"` → `- Defeated by: defeater-1 (invalid-inference, unsupported-conjunct)`
- `test_mixed_outlist_defeater_and_regular` confirms that a non-defeater outlist node is exported as `Unless`, while the defeater is exported as `Defeated by`.

If an outlist node does **not** satisfy the `defeats_node` metadata condition, it is treated as a regular “unless” blocker:

```python
elif o not in all_unless:
    all_unless.append(o)
```

at `reasons/export_markdown.py:91-92`, and emitted as:

```markdown
- Unless: blocker
```

at `reasons/export_markdown.py:95-96`.

Antecedents are separately collected across all justifications and deduplicated:

```python
if a not in all_deps:
    all_deps.append(a)
```

at `reasons/export_markdown.py:81-83`, then emitted as:

```markdown
- Depends on: a, b
```

at `reasons/export_markdown.py:93-94`. `test_multiple_antecedents` confirms this format.

## Stale and retraction

The `STALE` status is set only for `OUT` nodes with stale/retraction metadata:

```python
if status == "OUT" and (
    node.metadata.get("stale_reason") or node.metadata.get("retract_reason")
):
    status = "STALE"
```

at `reasons/export_markdown.py:54-55`.

The exported reason is selected from:

```python
retract_reason = node.metadata.get("retract_reason") or node.metadata.get("stale_reason")
```

at `reasons/export_markdown.py:102-103`.

If the node is `STALE` or `OUT` and a reason exists, the exporter emits:

```markdown
- Stale reason: <reason>
```

at `reasons/export_markdown.py:104-105`.

`superseded_by` is also exported:

```python
if node.metadata.get("superseded_by"):
    lines.append(f"- Superseded by: {node.metadata['superseded_by']}")
```

at `reasons/export_markdown.py:108-109`.

`test_stale_metadata` confirms the combined output:

```markdown
### a [STALE] ...
- Stale reason: superseded
- Superseded by: b
```

Note that storage has a `retracted_at` column at `reasons/storage.py:18-33`, but `export_markdown` does not read it. The markdown export relies on `truth_value` plus `retract_reason` / `stale_reason` metadata.

The CLI also exposes stale checking via `reasons check-stale`, documented at `README.md:124` and `README.md:185`.

## Duplicate

Duplicates are metadata-only in the exported markdown. If a node has `metadata["duplicate_of"]`, the exporter emits:

```markdown
- Duplicate of: <id>
```

from `reasons/export_markdown.py:106-107`.

This does not change the node’s status label or truth value. It is just an extra bullet under the claim. The same block also exports `accepted_pr` when present at `reasons/export_markdown.py:110-111`.

The provided observations do not include a test specifically for `duplicate_of`, so the exact import/round-trip behavior for duplicates is not visible here.

## Nogood

Nogoods are network-level records, not per-node status markers.

In `export_markdown`, nogoods are appended only when the network has any:

```python
if network.nogoods:
    lines.append("## Nogoods")
    ...
```

at `reasons/export_markdown.py:115-126`.

Each nogood is emitted as:

```markdown
### <nogood-id>
- Discovered: <discovered>      # optional
- Resolution: <resolution>      # optional
- Affects: <comma-separated node ids>
```

from `reasons/export_markdown.py:119-125`.

Storage defines nogoods as:

```sql
CREATE TABLE IF NOT EXISTS nogoods (
    id TEXT PRIMARY KEY,
    nodes_json TEXT NOT NULL DEFAULT '[]',
    discovered TEXT DEFAULT '',
    resolution TEXT DEFAULT ''
);
```

at `reasons/storage.py:46-50`.

The API returns nogood metadata such as `nogood_id`, `nodes`, `changed`, and `backtracked_to` from `reasons/api.py:1759-1781`. The README describes a nogood as a set of nodes that cannot all be `IN` simultaneously and notes that detecting one causes backtracking at `README.md:13` and `README.md:76-77`.

`test_nogoods_section` confirms the markdown section and ID format:

```markdown
## Nogoods
### nogood-a-b
- Affects: a, b
```

`export_card` only summarizes nogoods by count:

```python
nogood_count = len(network.nogoods)
```

at `reasons/export_card.py:58`, then emits `nogoods: {nogood_count}` in frontmatter at `reasons/export_card.py:94-100` and `Nogoods (contradictions)` in the stats table at `reasons/export_card.py:115-121`.

## What is not shown in these observations

- The `Node`, `Justification`, and `Nogood` class definitions are not included, so the exact in-memory invariants are not fully visible.
- The `beliefs.md` import/parser path is only partially visible through API references such as `import_into_network` and `_normalize_nogoods_markdown` in `reasons/api.py`; the actual normalization rules are not shown.
- The propagation logic that sets `stale_reason`, `retract_reason`, `defeats_node`, `defeater_type`, and `defeat_reason_type` is not shown.
- Exact per-line annotations for `reasons/export_markdown.py` and `reasons/export_card.py` are not provided in the observation; line references above are counted from the supplied file content.

## Topics to Explore

- [function] `reasons/export_markdown.py:export_markdown` — The source of per-node `[status]`, belief type, defeater, stale, duplicate, and nogood markdown conventions.
- [function] `reasons/export_card.py:export_card` — Generates the aggregate EEM card and shows which status metrics are exposed there.
- [file] `reasons/storage.py` — Defines the persisted shape of `truth_value`, justifications, metadata, and nogoods.
- [general] `defeater-metadata-propagation` — How `defeats_node`, `defeater_type`, and `defeat_reason_type` are created and consumed outside the exporter.
- [general] `beliefs-md-import-normalization` — How exported markdown and nogood sections are parsed back into a `Network`, including `duplicate_of` and stale metadata handling.

## Beliefs

- `export-markdown-status-maps-truth-value` — `export_markdown` uses `node.truth_value` as the status and only maps `OUT` to `STALE` when `stale_reason` or `retract_reason` metadata exists.
- `stale-is-export-only-label` — `STALE` is an export-only label; the underlying node remains `OUT`, and `export_card` counts it as `OUT`.
- `belief-type-defaults-from-justifications` — Without `beliefs_type` metadata, `export_markdown` emits `DERIVED` for nodes with justifications and `OBSERVATION` otherwise.
- `defeater-classification-requires-defeats-node` — An outlist node becomes `Defeated by` only when its `defeats_node` metadata matches the current node ID; otherwise it is exported under `Unless`.
- `duplicate-and-nogoods-do-not-change-node-status` — `duplicate_of` is a metadata bullet and nogoods are a separate `## Nogoods` network section; neither changes a node’s exported status label.
