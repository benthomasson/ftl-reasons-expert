---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Explore how retract/update cascade impact is computed before or during accept; propose-stage formatter accepts cascade data but does not produce it.

# How Cascade Impact Is Computed Before Accept

## The two-phase shape

The codebase splits "propose" from "impact preview" and from "accept":

1. **Propose** — `reasons/propose_update.py:propose_updates` asks the LLM for structured changes (`update` vs `retract`) and returns proposal dicts.
2. **Impact preview** — `reasons/api.py:what_if_retract` (line 512) does a read‑only in‑memory simulation; an aggregator (~line 4471–4531) builds a `{proposal_id: cascade}` map.
3. **Render** — `reasons/propose_update.py:format_proposals_file` accepts that map as `cascades=` and `format_proposal_markdown` prints a "Cascade impact" section.
4. **Accept** — there is **no programmatic accept handler** in `propose_update.py` (grep for `accept` = 0 matches; no `accept` symbol anywhere). The generated markdown simply tells the reviewer to run `reasons retract …` / `reasons update …` (see `format_proposal_markdown` command emission).

So cascade is computed **before** accept, in the preview stage, and acceptance is manual.

## Producer side — where cascade is computed

- **`reasons/api.py:512 what_if_retract`** — "Simulate retracting a node without mutating the database." It loads a read‑only network, calls `net.retract(node_id)` in memory (line 548), then separates results into `retracted` (went OUT, lines 551–564) and `restored` (went IN). Each entry is annotated with `"depth": _cascade_depth(net, nid, node_id)` (line 560), and lists are sorted by `(depth, id)` (line 568). The return shape is:
  ```python
  {"node_id": ..., "retracted": [...], "restored": [...],
   "total_affected": len(retracted) + len(restored), ...}
  ```
  (see lines 521, 574–576).
- **`reasons/api.py:_cascade_depth`** (line 737) — measures hop distance from the retracted node.
- **`reasons/api.py:652`** — a parallel what‑if for updates: "Adds a new node in memory, supersedes old_id, and returns the cascade effects" (uses `_cascade_depth(..., old_id)` at 711 and 719). This is the conceptually correct simulator for `action == "update"` proposals.
- **Aggregator at `reasons/api.py:~4471–4531`** — described as "Computes cascade impact for retract proposals." It does:
  ```python
  cascades = {}                                  # line 4519
  cascades[p["id"]] = what_if_retract(p["id"], db_path=db_path)   # 4524
  ...
  "cascades": cascades,                          # 4531
  ```
  Note the docstring scope: **retract proposals only**. This aggregator calls `what_if_retract` on each proposal id and returns the map into the CLI result.

## Consumer side — where cascade is formatted

- **`reasons/cli.py:2816`** — `cascades = result.get("cascades", {})` pulls the map out of the API result.
- **`reasons/cli.py:2836`** — `content = format_proposals_file(proposals, nodes=nodes, cascades=cascades)`.
- **`reasons/cli.py:2846`** — serializes `"cascades": {k: v for k, v in cascades.items()}` (for machine output alongside markdown).
- **`reasons/propose_update.py:format_proposals_file`** — signature `(proposals, nodes=None, cascades=None)`; per proposal does `cascade = cascades.get(nid) if cascades else None`.
- **`reasons/propose_update.py:format_proposal_markdown(proposal, nodes=None, cascade=None)`** — the renderer:
  - `if cascade:` (truthy dict), reads `retracted`, `restored`, `total_affected`.
  - `action == "RETRACT"` → "N dependent(s) affected" with `— will go OUT` / `— will go IN`.
  - otherwise (i.e. `UPDATE`) → "N dependent(s) need re-review" — same lists, relabeled.
  - `total == 0` → "**Cascade impact:** none".
  - `cascade` is `None` → "**Cascade impact:** not computed".

So the formatter *does* accept cascade data and knows how to render it for both retract and update proposals.

## The gap: propose-stage accepts but never produces cascade

`reasons/propose_update.py:propose_updates` (lines ~250–332) is a closed loop:

```
invoke_model(prompt) → parse_update_proposals(response) → extend all_results
```

`parse_update_proposals` (the whitelist stage) rebuilds each dict with exactly:
`id, action, proposed_text, failure_mode, basis, evidence, comment`.

Any `cascade` key the LLM might emit would be discarded, and `propose_updates` never calls `what_if_retract` / the supersede simulator. Consequently:

- **`propose_updates` output contains no cascade field** — it cannot feed the formatter directly.
- If a caller invokes `format_proposals_file(proposals, nodes=...)` without the aggregated `cascades=` dict, every proposal prints "Cascade impact: not computed" (the test `test_no_cascade` pins this behavior).
- The only path that yields real cascade text is the CLI‑adjacent aggregator (~api.py:4471–4531), which is outside `propose_update.py`.

**What the prompt does instead:** `format_belief_for_update` includes a `Dependents (N):` block (lines in the formatter) so the LLM sees dependent IDs/text while deciding. That is *qualitative* input to the model — not the machine‑computed cascade that `format_proposal_markdown` renders.

## The update-vs-retract asymmetry

Even when cascade *is* supplied, note the semantic mismatch:

- The formatter's `UPDATE` branch ("need re-review") consumes the **same** `retracted` / `restored` / `total_affected` shape that `what_if_retract` produces (see `test_update_cascade_says_re_review`, which feeds a retract‑shaped cascade into an update proposal).
- But the semantically correct simulation for an update is the supersede path at `api.py:652` (`_cascade_depth(..., old_id)`), and the aggregator's own docstring says it only computes "for retract proposals".

Net effect: in practice **RETRACT proposals get computed impact; UPDATE proposals typically fall through to "not computed"** — even though the renderer has an UPDATE‑specific message ready. This is the exact complaint in the framing: the formatter accepts cascade data for updates, but nothing in the propose path produces it.

## What is missing from the observations

- The exact loop body at `api.py:~4510–4525` isn't shown verbatim — I can't confirm whether it filters `p["action"] == "retract"` before calling `what_if_retract`, or whether it calls the supersede simulator for updates.
- The full signature/name of the supersede what‑if (`api.py:652`) isn't visible — only its docstring and `_cascade_depth` calls.
- Whether `format_proposals_file` is ever called with `cascades=` populated for update proposals (e.g., via `cli.py:2836` in the real propose CLI flow) can't be confirmed without reading the `propose` subcommand handler at `cli.py:~2800–2850`.
- The observation `grep_cascade_in_tests` reports 0 matches, but `tests/test_propose_update.py` visibly contains `test_with_cascade`, `test_update_cascade_says_re_review`, `test_empty_cascade`, and `cascade = {...}` literals — this looks like a grep-scope/recording inconsistency worth re‑checking.

## Topics to Explore

- [file] `reasons/api.py` — Read lines ~4450–4540 to confirm whether the aggregator filters by `action` and whether it ever calls the supersede simulator for `update` proposals.
- [function] `reasons/api.py:what_if_retract` — Understand read‑only simulation and the `retracted`/`restored`/`depth` contract the formatter depends on.
- [function] `reasons/api.py:_cascade_depth` (and the supersede what‑if near line 652) — The depth metric and the update-specific simulation that would be needed to make UPDATE cascade real.
- [file] `reasons/cli.py` — Trace the `propose` subcommand handler around lines 2800–2850 to see exactly where `cascades` gets attached before `format_proposals_file`.
- [general] `accept-flow` — How the human commands printed in the markdown (`reasons retract … --reason …`, `reasons update …`) actually re-run the true cascade at accept time.

## Beliefs

- `cascade-is-computed-pre-accept` — Cascade impact is produced at proposal-generation time via `what_if_retract` and rendered into the proposals markdown; acceptance itself is manual via suggested CLI commands.
- `what-if-retract-is-read-only` — `what_if_retract` loads a read-only network, applies `net.retract` in memory, and never mutates the database.
- `propose-updates-does-not-emit-cascade` — `propose_updates` returns only the LLM fields whitelisted by `parse_update_proposals`; it never calls a what-if simulator and never emits a cascade key.
- `parse-update-proposals-strips-extra-fields` — The parser rebuilds each proposal with exactly `id, action, proposed_text, failure_mode, basis, evidence, comment`, dropping any `cascade` field the model might output.
- `formatter-accepts-cascade-via-cascades-dict` — `format_proposals_file(proposals, nodes, cascades)` looks up `cascades.get(nid)` per proposal and passes it to `format_proposal_markdown`.
- `no-accept-function-in-propose-update` — There is no `accept` symbol in `reasons/propose_update.py`; grep for "accept" returns zero matches, confirming acceptance is out-of-band.
- `update-cascade-branch-requires-retract-shaped-data` — The UPDATE rendering branch consumes the same `retracted` / `restored` / `total_affected` shape that `what_if_retract` produces, even though the semantically correct simulator for updates is the supersede what-if near `api.py:652`.
