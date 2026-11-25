---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Whether `UNKNOWN` is a distinct third state in this codebase or absent (grep for `"UNKNOWN"` returned 0 matches), and if absent, whether it should be reintroduced for "no justification yet" nodes.

# Is `UNKNOWN` a distinct third state?

**No — `UNKNOWN` is not a truth-value state in this codebase, and it should not be reintroduced as one.**

There are two separate problems with the framing in the question. First, the premise is factually wrong. Second, the small number of `UNKNOWN`/`unknown` occurrences that *do* exist are not state-machine values — they are transient request payload markers and defensive display defaults.

## The grep premise is wrong

The question says `grep "UNKNOWN"` returned 0 matches. The observations actually show **7 uppercase `UNKNOWN` matches** and **26 total matches** for the broader state vocabulary pattern:

- `reasons/api.py:3764` and `reasons/api.py:3836` — literal `"truth_value": "UNKNOWN"` inside `premises_data.append(...)` payloads
- `reasons/report_belief.py:38, 85, 123, 149, 194` — `.get("truth_value", "UNKNOWN")` defensive defaults
- `reasons/metadata.py:20` — unrelated lowercase `"unknown"` (likely version parsing)
- `reasons/propose_update.py:101, 222, 223, 259` — lowercase `"unknown"` as display fallbacks for missing proposal fields
- Test fixtures in `tests/test_build_wiki.py` — IDs like `"xyz-unknown-thing"`, `"unknown-node"` — just naming, not state

So `UNKNOWN` is *not* absent. It just isn't a persisted state.

## The truth-value model is strictly IN/OUT

Three independent pieces of evidence confirm the two-state model:

1. **The SQLite schema** (`reasons/storage.py`, `SCHEMA`): `truth_value TEXT NOT NULL DEFAULT 'IN'`. No CHECK constraint enumerating values, but the default and the fact that `Storage.save` writes `node.truth_value` directly from the `Network` object point to IN/OUT being the only meaningful domain.
2. **`CLAUDE.md` "Design Decisions"**: *"No STALE status — uses IN/OUT only; staleness triggers retraction"* and *"Retracted nodes stay in DB (Doyle's insight: enables restoration without rederivation)"*. This is an explicit, documented decision.
3. **`CLAUDE.md` "Key Concepts"**: *"Node: A belief with a truth value (IN or OUT)"* and *"Premise: A node with no justifications — IN by default"*.

Doyle's 1979 TMS — which this project explicitly cites — has exactly two states. There is no third "unknown" state in the underlying formalism.

## What the `"UNKNOWN"` occurrences actually are

### `reasons/api.py:3764` and `:3836` — transient request payload markers

```python
premises_data.append({"id": pid, "text": "", "truth_value": "UNKNOWN", "source": ""})
```

Note the empty `"text"` and `"source"`. This is not constructing a node to persist — it's building a list of *premises to be resolved* (probably for a bulk add/derive operation, where the caller must fill in text/source before the node is actually created). `"UNKNOWN"` here means "this field will be populated later," not "this node's belief state is UNKNOWN." It's a stub in a request body, not a value written to the `nodes.truth_value` column.

The observations don't include lines 3700–3760 or 3840–3880 of `api.py`, so I can't confirm what consumes `premises_data`, but the shape strongly implies it's intermediate data for a batch mutation, not state.

### `reasons/report_belief.py:38, 85, 123, 149, 194` — defensive display defaults

```python
status = node_detail.get("truth_value", "UNKNOWN")
tv = pd.get("truth_value", "UNKNOWN")
```

These are all `.get(..., default)` calls — the default only fires if `"truth_value"` is *missing from the dict entirely*. That never happens for a node loaded from SQLite (the column is `NOT NULL DEFAULT 'IN'`). It would only surface for a malformed/partial record. This is defensive coding in report rendering, not a real state the system transitions through.

### `reasons/propose_update.py` and `reasons/metadata.py` — unrelated lowercase fallbacks

These are entirely orthogonal: `"unknown"` for missing `failure_mode`, `basis`, and version metadata. They do not touch `truth_value`.

## Should `UNKNOWN` be reintroduced for "no justification yet" nodes?

**No.** In this model, "no justification yet" is already a first-class concept, and it does *not* need a third truth value. It's handled as follows:

- A node with zero justifications is a **premise**, and premises are **IN by default** (see `CLAUDE.md` Key Concepts and `tests/test_add_justification.py::test_add_justification_to_premise`, which shows adding a justification to a premise "gives it a backup route" — the premise is already IN before the justification exists).
- Whether a node is "asserted but unproven" versus "derived" is encoded by **presence/absence of justifications** (`node.justifications`), not by `truth_value`. The `test_add_justification.py` and `test_remove_justification.py` suites both exercise this: a premise with no justifications is IN; removing a justification can push a derived node OUT; adding one can restore it.
- The design explicitly rejects additional states (`CLAUDE.md`: no STALE; staunchness is expressed via retraction).

Introducing a third `UNKNOWN` state would:

1. Conflict with the Doyle TMS two-state propagation in `reasons/network.py` (not shown in observations, but referenced throughout tests).
2. Break the `SCHEMA` default and require a migration in `Storage._init_schema`.
3. Undermine "restoration without rederivation" — the whole point of keeping retracted nodes around is that their state is a pure function of justifications, recomputed by BFS propagation.
4. Force every consumer of `truth_value` (report, propose_update, compact, cli, export) to reason about a third case it currently doesn't.

If the goal is to *distinguish* "asserted, no support yet" from "supported," the right place to put that is metadata or a derived query over `node.justifications` (e.g., a `node.justifications == []` check filtered by `depth`), **not** a new truth value.

## What the observations don't show

To fully close this out, I'd want:

- `reasons/network.py` — to confirm no `"UNKNOWN"` branch exists in `propagate` / `_recompute` / `assert` / `retract`.
- `reasons/api.py` lines ~3700–3880 — to confirm what consumes `premises_data` and that `"UNKNOWN"` there is a stub.
- Any CHECK constraint on `nodes.truth_value` in a later migration (the shown schema has none).

## Topics to Explore

- [file] `reasons/network.py` — Confirms the propagation logic treats `truth_value` as a strict IN/OUT binary; look for `_recompute` / BFS cascade.
- [file] `reasons/api.py` — Read lines ~3700–3880 to see what `premises_data` with `"truth_value": "UNKNOWN"` feeds into (batch add? import? derive?).
- [file] `reasons/report_belief.py` — Verifies the `"UNKNOWN"` defaults are display-only fallbacks in belief reporting.
- [general] `premise-vs-derived-semantics` — How the codebase distinguishes premises (no justifications, IN by default) from derived nodes (justifications, IN iff at least one is valid).
- [general] `IN-OUT-state-machine` — The Doyle TMS two-state model and why "no support yet" is expressed as a premise rather than a third state.

## Beliefs

- `truth-value-is-in-out-only` — The persisted `nodes.truth_value` is constrained to IN/OUT by schema default (`DEFAULT 'IN'`) and by the explicit "uses IN/OUT only" design decision in `CLAUDE.md`.
- `premise-has-no-justification-and-is-in-by-default` — A node added without `sl`/`unless` is a premise, has zero justifications, and starts IN per `CLAUDE.md` and `tests/test_add_justification.py::test_add_justification_to_premise`.
- `unknown-appears-only-as-transient-payload-marker` — The literal `"UNKNOWN"` in `reasons/api.py:3764` and `:3836` is a stub in a `premises_data` request payload with empty `text`/`source`, not a persisted `truth_value`.
- `report-belief-uses-unknown-as-defensive-default` — `reasons/report_belief.py:38,85,123,149,194` use `"UNKNOWN"` only as the second argument to `dict.get`, so it surfaces only for malformed/missing `truth_value` fields.
- `no-stale-status-by-design` — `CLAUDE.md` records the explicit design choice: no STALE status; truth values are IN/OUT only and staleness is expressed by retraction.
