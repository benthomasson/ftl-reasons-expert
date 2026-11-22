---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How `defeats_node`, `defeater_type`, and `defeat_reason_type` are created and consumed outside the exporter.

## How `defeats_node`, `defeater_type`, and `defeat_reason_type` are created and consumed outside the exporter

These three fields are **metadata keys on defeater nodes** — ordinary nodes that stand in a "defeats" relationship to another (the target) node. They form a small typed triple (`who is defeated`, `how`, `why`), and the rumor trail lives almost entirely in `reasons/api.py`, with narrower reads/writes in `reasons/build_wiki.py`, `reasons/cli.py`, and `reasons/review.py`.

### 1. Creation — three producers, plus one enricher

There are **four** independent write sites, and they play distinct roles.

#### (a) `defeat_justification` — `reasons/api.py` ~1326–1410

The primary path. Signature-level defaults are `defeater_type="invalid-inference"` (api.py:1326) and `defeat_reason_type=""` (api.py:1328). The defeater ID is generated deterministically as `f"{defeater_type}-{node_id}-j{justification_index}"` (api.py:1372). Then:

```python
"defeater_type": defeater_type,          # api.py:1379
"defeats_node":  node_id,                # api.py:1380
...
if defeat_reason_type:                   # api.py:1383
    meta["defeat_reason_type"] = defeat_reason_type
```

So `defeats_node` and `defeater_type` are **unconditionally written**; `defeat_reason_type` is **conditional** — omitted entirely when empty. The returned dict (api.py:1408–1409) echoes both type fields back to callers.

This path is what `tests/test_defeat_justification.py:42` asserts against (`defeater["metadata"]["defeats_node"] == "belief"`).

#### (b) A second defeat function — `reasons/api.py` ~1419–1543

Structurally identical: same defaults (api.py:1419, 1421), same ID pattern (api.py:1470), same metadata construction (api.py:1506), and the same conditional write of `defeat_reason_type` (api.py:1510–1511). The conditional at **1506–1511** is the giveaway that both paths behave the same way for the reason-type field. This second entry point is what `tests/test_defeat_with_scope.py:91` exercises (the test name suggests the function is scope-aware; I can't see its signature from the evidence).

#### (c) Migration — `reasons/api.py` ~1620–1622

A migration path writes both keys with a **fixed** `defeater_type`:

```python
"defeater_type": "migrated-retraction",   # api.py:1621
"defeats_node":  nid,                     # api.py:1622
```

`tests/test_migrate_defeaters.py:30` confirms this. Notably, no `defeat_reason_type` is written here — the migration relies on the enricher (below) to classify later.

#### (d) Enricher: `classify_defeat_reason_types` — `reasons/api.py` ~1652–1715

This function **only writes `defeat_reason_type`**, and only onto nodes that already carry `defeats_node`. It uses an LLM classifier (`classify_defeat_reason`, imported by `tests/test_classify_defeaters.py:4` from `reasons.review`) and writes back with `set_metadata(nid, "defeat_reason_type", reason_type, ...)` at api.py:1715. In dry-run mode it skips the write but still returns the classification in `result["classified"]` (api.py:1712).

#### Summary of creation logic

| Field | Where written | Default |
|---|---|---|
| `defeats_node` | two defeat functions + migration | required arg (`node_id` / `nid`) |
| `defeater_type` | two defeat functions + migration | `"invalid-inference"` / `"migrated-retraction"` |
| `defeat_reason_type` | two defeat functions (conditional) + `classify_defeat_reason_types` (post-hoc) | `""` — omitted unless non-empty |

### 2. Consumption outside the exporter

#### (a) Inside `classify_defeat_reason_types` itself — api.py:1683–1715

The enricher consumes both of the other fields to decide what to work on:

- `if not meta.get("defeats_node"):` (api.py:1683) — **gate**: only nodes that defeat something are classified.
- `defeated_id = meta.get("defeats_node", "")` (api.py:1699) — retrieves the target node so the classifier can compare defeater text vs. defeated text.
- `if meta.get("defeat_reason_type"):` (api.py:1685) — **idempotency**: skip if already classified.
- `if defeater_type_filter and meta.get("defeater_type") != defeater_type_filter:` (api.py:1688) — **filter**: only classify matching defeater types; the skipped list records the reason string (api.py:1689).

The interplay here is non-obvious: `defeats_node` is a hard prerequisite, `defeater_type` is an optional filter, and `defeat_reason_type` is a "have I done this already?" marker.

#### (b) `reasons/build_wiki.py:90–95` — wiki synthesis

```python
if o_meta.get("defeats_node") == node_id:      # build_wiki.py:90
    ...
    d_type = d_meta.get("defeater_type", "defeater")     # build_wiki.py:94
    r_type = d_meta.get("defeat_reason_type", "")        # build_wiki.py:95
```

The pattern is identical to the exporter's — it iterates a node's outlist, finds outlist nodes whose `defeats_node` points back at this node, and then uses `defeater_type` (fallback `"defeater"`) plus `defeat_reason_type` (fallback `""`) to label the defeat in the wiki. `tests/test_build_wiki.py:185–231` pins this behavior, including the fallback to the literal string `"defeater"` (test line 210).

#### (c) `reasons/cli.py` — orchestration

Multiple CLI commands shuffle these fields across the API boundary:

- **`defeat` commands** (cli.py:208–218 and cli.py:260–271): forward `args.type or "invalid-inference"` as `defeater_type` and `getattr(args, "reason_type", None) or ""` as `defeat_reason_type`; then read them back off the returned dict (`result['defeater_type']`, `result.get('defeat_reason_type', '')`).
- **Re-apply flow** (cli.py:252): `reason_type = getattr(args, "reason_type", None) or data.get("defeat_reason_type", "")` — the CLI re-uses the *existing* `defeat_reason_type` from storage when the user doesn't supply one, preserving the classification across re-defeats.
- **Bulk re-defeat** (cli.py:2488–2501): reads `r.get("defeat_reason_type", "")` back from records and re-writes with `defeater_type="invalid-inference"` and the preserved `defeat_reason_type`.
- **`classify-defeaters` command** (cli.py:320–345): passes `defeater_type_filter=getattr(args, "type", None)` into the API call and prints `c['defeat_reason_type']` per classified node.
- **`--type` argument** (cli.py:3163): the user-facing knob that surfaces the `defeater_type_filter`.

#### (d) `reasons/review.py:47`

A template/default dict emits `"defeat_reason_type": ""` — this is where the empty-string default is materialized for review entries. The same module also owns `classify_defeat_reason` (the LLM caller) and `DEFEAT_REASON_TYPES` (7 valid types, per `tests/test_classify_defeaters.py:158`).

### 3. What the evidence does *not* show

A few things are inferred, not observed:

- The **exact names** of the two defeat functions in `api.py` (1326 and 1419). I've described them by line range and test file. `tests/test_defeat_justification.py` and `tests/test_defeat_with_scope.py` strongly suggest their names, but the observations don't quote a `def` line.
- The **body of `classify_defeat_reason`** in `review.py` — only the default dict at line 47 is visible; the LLM call and type validation are inferred from `tests/test_classify_defeaters.py`.
- The **contents of `DEFEAT_REASON_TYPES`** — only the existence of 7 hyphenated values is known from tests (`unsupported-conjunct`, `over-generalizes`, `scope-mismatch` seen).

---

## Topics to Explore

- [file] `reasons/review.py` — Source of `classify_defeat_reason` and the canonical `DEFEAT_REASON_TYPES` tuple; the observation only shows line 47.
- [function] `reasons/api.py:classify_defeat_reason_types` — Full filtering, deduplication, and write-back logic; the observations show call sites but not the LLM invocation itself.
- [function] `reasons/api.py:defeat_justification` — Primary creation path; verify the exact signature of the two defeat functions at api.py:1326 and 1419.
- [general] `defeater-id-naming-convention` — The `f"{defeater_type}-{node_id}-j{justification_index}"` pattern (api.py:1372, 1470) is the deterministic key that lets re-defeats find the same prefix.
- [file] `tests/test_defeat_with_scope.py` — Confirms the second defeat entry point's scope semantics and metadata shape.

## Beliefs

- `defeats-node-is-required-for-classification` — `classify_defeat_reason_types` skips any node without a `defeats_node` metadata entry (api.py:1683).
- `defeater-type-defaults-to-invalid-inference` — The default `defeater_type` is `"invalid-inference"` in both defeat API functions (api.py:1326, 1419) and their CLI wrappers (cli.py:208, 260, 2493, 2500).
- `defeat-reason-type-is-omitted-when-empty` — `defeat_reason_type` is only written to metadata when the caller supplies a non-empty string (api.py:1383–1384, 1510–1511).
- `migration-produces-migrated-retraction-defeaters` — The migration path tags every migrated defeater with `defeater_type = "migrated-retraction"` (api.py:1621) and does not set `defeat_reason_type`.
- `defeat-reason-type-requires-llm-classification` — `defeat_reason_type` is populated post-hoc by `classify_defeat_reason_types` via `set_metadata` (api.py:1715), not at defeat-creation time.
- `defeater-type-filters-classification-scope` — `defeater_type_filter` in `classify_defeat_reason_types` restricts which defeaters are sent to the LLM (api.py:1688–1689), exposed to users via `cli.py:3163` `--type`.
