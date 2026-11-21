---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: import_agent in reasons/import_agent.py

## `import_agent` — Agent Belief Importer (Markdown Format)

### Purpose

`import_agent` is the Markdown-flavored entry point into the "import an external agent's beliefs" capability. It exists to **merge another agent's belief set into the local RMS (`Network`) without ID collisions and with a built-in kill switch**. It is a thin façade over `_import_claims` that:

1. Parses a Markdown belief file (via `parse_beliefs`) into a normalized claim representation.
2. Parses an optional Markdown nogood file (via `parse_nogoods`).
3. Hands the normalized payloads to the shared `_import_claims` implementation for materialization.

The function is the counterpart to `import_agent_json` (same pipeline, JSON input) and the seed-side counterpart to `sync_agent` (which reconciles against existing nodes).

---

### Contract

**Preconditions**
- `network` is a constructed `Network` with the attributes/methods used downstream: `nodes` (mapping), `nogoods` (list), `repos` (mapping), `add_node`, `retract`, `recompute_all`, `assert_node`, `_rebuild_dependents`.
- `beliefs_text` is valid Markdown that `parse_beliefs` understands.
- `nogoods_text` is either `None`/empty or valid Markdown for `parse_nogoods`.
- `agent_name` is a stable, non-empty identifier. **It must not contain `:`** — the prefix scheme (`f"{agent_name}:"`) and reverse id-stripping rely on that.
- Claim IDs inside `beliefs_text` are unique.
- Calling convention is "import once, then `sync_agent` for updates" — re-running `import_agent` on the same agent will not update existing nodes (they are *skipped*).

**Postconditions**
- The following nodes exist in `network.nodes`:
  - `{agent}:active` (premise: "Agent 'X' beliefs are trusted"), created only if missing.
  - `{agent}:inactive` (relay: `SL [] ([{agent}:active])`), created only if missing.
- Every imported belief is stored under `{agent}:{claim_id}` with `agent:inactive` prepended to its outlist (or as the sole outlist when there is no raw justification).
- Claims that were `STALE`/`OUT` in the source are pushed OUT and marked `_retracted` (or retracted post-propagation for the bare-premise case) — see algorithm.
- Nogoods from `nogoods_text` are added under `{agent}:{nogood_id}` if both endpoints exist as prefixed nodes.
- If `source_path` is non-empty, `network.repos[agent_name]` is set to the resolved parent directory of `source_path`.
- The network's truth values are recomputed at least once; the return value reports the number of change events.

**Invariants preserved**
- Imported node IDs are always prefixed, guaranteeing they cannot collide with locally-authored nodes.
- Retracting `{agent}:active` cascades every imported belief to OUT (via `{agent}:inactive` being IN, which appears in every imported belief's outlist). This is the entire reason `active` is **not** placed in antecedents — an antecedent would give every belief a fallback IN justification, defeating per-belief retraction. This is stated explicitly in the module docstring and is the central design invariant.
- Nodes are **never overwritten** by this function (only added or skipped) — that mutation authority is reserved for `sync_agent`.

---

### Parameters

| Name | Type | Meaning | Edge Cases |
|---|---|---|---|
| `network` | `Network` | Live network graph being mutated. | Duck-typed; no protocol/ABC enforces required methods. |
| `agent_name` | `str` | Namespace and identity. Becomes the prefix and is stored in every node's metadata as `agent`. | Must not contain `:`. Must be stable across import/sync cycles. Empty string technically produces ":" prefixed nodes (broken) — not validated. |
| `beliefs_text` | `str` | Markdown source. | Empty string → `parse_beliefs` yields no claims; import still creates the infra nodes and returns zero counts. |
| `nogoods_text` | `str \| None` | Optional Markdown nogoods. | `None` or falsy → `_normalize_nogoods_markdown` returns `[]` (early return on `if not nogoods_text`). |
| `only_in` | `bool` | If `True`, normalizer drops any claim whose status is not `"IN"`. | Applied *before* prefixing; skipped claims never appear in the network and never count in `claims_skipped`. |
| `source_path` | `str` | Path to the source file. Two uses: stored in `network.repos[agent_name]` as the resolved parent, and stamped into `imported_from` metadata on every imported node. | Empty string → `repos` entry not written, `imported_from` is `""`. |

---

### Return Value

A dict with these keys (all counts are per-invocation, not cumulative):

| Key | Meaning |
|---|---|
| `agent` | Echoes `agent_name`. |
| `prefix` | `f"{agent_name}:"`. |
| `active_node` | The `{agent}:active` node ID. |
| `created_premise` | `True` iff the `active` node was newly created this call. (Note: `inactive` creation is not separately reported.) |
| `claims_imported` | Count of new `{agent}:{id}` nodes added. |
| `claims_skipped` | Count of claims whose prefixed ID already existed in the network (import is not idempotent-write — it is add-or-skip). |
| `claims_retracted` | Count of claims that were retracted *after* propagation (the bare-premise/OUT-without-justification path). Not incremented for the `_retracted`-flag path. |
| `claims_propagated` | Number of node truth-value changes returned by `network.recompute_all()`. |
| `nogoods_imported` | Count of nogoods successfully added. |

The caller must handle: the return value is informational — all errors are raised, none are surfaced in the dict.

---

### Algorithm (step by step)

**Step 1 — Markdown normalization (`_normalize_markdown`)**

`parse_beliefs(beliefs_text)` returns claim dicts with keys `id, text, type, status, source, source_hash, date, depends_on, unless`.

- If `only_in=True`, drop claims whose `status != "IN"`.
- Build `claim_ids` = set of the remaining claim IDs — this is what constrains cross-claim antecedents/outlists.
- For each claim:
  - `is_out = status in ("STALE", "OUT")`.
  - If `is_out`: `raw_justs = []` (Markdown strips justifications; a retired belief becomes a bare premise that cannot be resurrected by justification reasoning).
  - Otherwise: emit a single `SL` justification with `antecedents = [d for d in depends_on if d in claim_ids]` and `outlist = [o for o in unless if o in claim_ids]`. Filtering against `claim_ids` prevents dangling references to claims that were dropped by `only_in`.
  - Preserve metadata fields; copy `type` into `metadata["beliefs_type"]` when present.

**Step 2 — Nogood normalization (`_normalize_nogoods_markdown`)**

If `nogoods_text` is falsy → `[]`. Otherwise `parse_nogoods(...)` is filtered to `{id, nodes, discovered, resolution}` dicts (renaming `affects` → `nodes`).

**Step 3 — Shared import (`_import_claims`)**

1. **Infra nodes** (`_ensure_agent_nodes`):
   - `active_id = f"{agent}:active"` added if missing (no justifications — treated as premise; this is what "trusted by default" means in RMS terms).
   - `inactive_id = f"{agent}:inactive"` added if missing with `Justification(type="SL", antecedents=[], outlist=[active_id])` — i.e. *inactive is IN when active is OUT*.
   - Returns `(active_id, inactive_id, created_premise)`.

2. **Repo registration**: if `source_path`, `network.repos[agent_name] = resolve(source_path).parent`.

3. **Topological order** (`_topo_sort_claims`): claims are ordered so that antecedents are added before dependents. Cycle-safe — after `len(remaining)` passes, any unresolved claims are appended in original order.

4. **Node materialization** per claim:
   - `node_id = f"{prefix}{claim['id']}"`. If already in `network.nodes`, `claims_skipped += 1; continue` (add-or-skip).
   - `_build_justifications` builds `Justification` objects with:
     - `antecedents = [f"{prefix}{a}" for a in raw_antecedents]`
     - `outlist = [inactive_id] + [f"{prefix}{o}" for o in raw_outlist]` — **`inactive_id` always goes first**.
     - If the claim had no raw justifications, synthesize a default `(SL [] [inactive_id])`.
   - Metadata is `claim.metadata` plus `agent`, `original_id`, `imported_from`.
   - `network.add_node(...)` is called. `source_url` is threaded through as `claim.get("source_url", "")` — Markdown never sets it, JSON might.

5. **OUT handling (two distinct paths)**:
   - **Bare OUT**: `is_out and not raw_justifications` (Markdown retirement) — push `node_id` onto `retract_after`; no metadata flag set.
   - **Justified OUT**: `is_out` with raw justifications (JSON path, in practice) — set `metadata["_retracted"] = True` and force `truth_value = "OUT"`. This blocks `recompute_all()` from resurrecting the node via its justifications.

6. **Nogoods** (`_import_nogoods`): for each nogood, prefix node IDs, keep only nodes that exist, and add the nogood only if `len(valid_nodes) >= 2` and the prefixed nogood ID is new.

7. **Rebuild dependents** (`_fixup_dependents` → `network._rebuild_dependents()`): required because outlist entries for `inactive_id` may have been added *after* the dependents that reference them, so `add_node` could not have wired the dependents at creation time.

8. **Propagate**: `propagated = len(network.recompute_all())`.

9. **Deferred retraction**: for each `retract_after` node, `network.retract(node_id)` then increment `claims_retracted`. These are the bare-OUT beliefs — after retraction they stay OUT because they have no antecedent support and `inactive` is likely still OUT (so the `SL [] [inactive]` justification would be **valid**, meaning IN — wait, that's why they need the retraction call: retraction forces OUT independent of justification validity, and there's no outlist member pointing at them to bring them back).

10. **Return** the stats dict.

---

### Side Effects

- Mutates `network.nodes` (adds new node objects).
- Mutates individual node objects — `metadata["_retracted"]`, `truth_value`, `justifications` (only via `_build_justifications` on newly created nodes here — existing nodes are never touched).
- Appends to `network.nogoods`.
- Writes `network.repos[agent_name]` when `source_path` is truthy.
- Calls `network._rebuild_dependents()` (private API).
- Calls `network.recompute_all()` — propagates truth value changes across the network, potentially flipping truth values of *unrelated* existing nodes.
- Calls `network.retract(node_id)` which mutates the passed node's truth value (and, per the RMS model, may enqueue downstream propagation on the next recompute — note the second `recompute_all` is **not** called after `retract_after`, so any downstream effects of those retractions are deferred to the caller's next recompute/propagate cycle).

**No I/O** — this function only reads its arguments and writes into `network`.

---

### Error Handling

There is **no try/except** anywhere in `import_agent` or `_import_claims`. All exceptions propagate:

- `parse_beliefs` / `parse_nogoods` — raise on malformed Markdown; caller sees the parser's exception.
- `network.add_node` — may raise on invariant violations (the duplicate-ID path is pre-checked via `if node_id in network.nodes`).
- `network.retract` / `assert_node` / `recompute_all` — depend on `Network`'s own error surface, unspecified here.
- `Path(source_path).resolve()` — raises `OSError`/`RuntimeError` for pathological paths (e.g. symlink loops).

Because errors are not swallowed and the function is not transactional, a mid-import exception can leave the network **partially mutated** (infra nodes added, earlier claims added, later ones missing). Callers wanting atomic imports would need to construct/target a fresh `Network` or snapshot around the call.

---

### Usage Patterns

Typical call sites (per the module docstring and CLI tests):

```bash
reasons import-agent aap-expert ~/git/aap-expert/beliefs.md
reasons import-agent rhel-expert ~/git/rhel-expert/beliefs.md --only-in
```

Programmatic pattern:

```python
stats = import_agent(
    network,
    agent_name="aap-expert",
    beliefs_text=Path("~/git/aap-expert/beliefs.md").read_text(),
    nogoods_text=Path("~/git/aap-expert/nogoods.md").read_text() if ... else None,
    source_path="/home/me/git/aap-expert/beliefs.md",
)
```

**Caller obligations**
- Use `sync_agent` for updates; `import_agent` is seed-only (skips existing IDs).
- Do not reuse the same `agent_name` for two distinct belief sources — the prefix is the sole namespace.
- After `import_agent`, you should generally trigger another `recompute_all` / propagate if any `retract_after` retractions were performed, because downstream bonuses from those retractions are not repaginated inside this function.
- Be aware the function's `claims_propagated` count happens **before** the deferred retractions — so it does not include post-retract cascade effects.

---

### Dependencies

**External / stdlib**
- `pathlib.Path` — used for `Path(source_path).resolve().parent`.

**Package-internal**
- `.` — `Justification`, `Nogood` (RMS primitive types).
- `.import_beliefs` — `parse_beliefs` and `parse_nogoods` (Markdown parsers).
- `.network` — `Network` type (the target graph).

**Within the module**
- `_normalize_markdown`, `_normalize_nogoods_markdown` — format conversion.
- `_import_claims` — shared orchestration.
- `_ensure_agent_nodes`, `_topo_sort_claims`, `_build_justifications`, `_import_nogoods`, `_fixup_dependents` — helpers used transitively.

---

### Assumptions the type system does **not** enforce

1. `network` is a `Network` — Python will happily accept any object with matching attributes until the first missing method.
2. `agent_name` contains no `:` — nothing validates this; a colon would make the namespacing ambiguous and would corrupt reverse-id logic in `sync_agent`.
3. `beliefs_text` is valid Markdown — the type is just `str`; malformed input surfaces as a parser exception, not a validation error.
4. Claim IDs in the source are unique — duplicates in the input would be silently skipped after the first add (changing the meaning from "duplicate" to "ignore").
5. Cross-claim references in `depends_on` / `unless` that are not in `claim_ids` are treated as external/ignored, not as errors.
6. The `"IN"`/`"STALE"`/`"OUT"` status strings and the `"SL"` justification type are stringly-typed literals matching the parser contract — no enum enforces them.
7. `Network.repos` exists and behaves like a `dict[str, str]` — no protocol declares it.
8. `network.recompute_all()` returns an iterable of change records — the code only calls `len(...)` on it.
9. `network.retract(node_id)` is idempotent-ish in the sense that calling it on an already-OUT node with justifications works; the bare-OUT path relies on that.
10. `parse_beliefs` and `parse_nogoods` return dicts with the exact key names used here (`id, text, status, source, source_hash, date, depends_on, unless, type`, and `id, affects, discovered, resolution` respectively).

---

## Topics to Explore

- [file] `reasons/import_beliefs.py` — Where `parse_beliefs` and `parse_nogoods` live; defines the Markdown contract this function consumes.
- [function] `reasons/network.py:add_node` — How nodes are created and whether justifications must reference pre-existing nodes (drives the `_fixup_dependents` call).
- [function] `reasons/network.py:recompute_all` — The TMS propagation loop; explains what `claims_propagated` counts and why `_retracted` guards resurrection.
- [function] `reasons/import_agent.py:sync_agent` — The remote-wins update path; contrasts with this add-or-skip importer.
- [file] `tests/test_import_agent.py` — Executable spec for prefixing, kill switch, `only_in`, and bare-OUT behavior.

---

## Beliefs

- `import-agent-normalizes-markdown-beliefs` — `import_agent` converts Markdown via `_normalize_markdown` into `{id, text, is_out, source, source_hash, date, metadata, raw_justifications}` dicts before import.
- `imported-nodes-namespaced-with-agent-prefix` — Every imported node ID is stored as `{agent_name}:{claim_id}`, and existing prefixed IDs cause the claim to be counted in `claims_skipped` rather than replaced.
- `inactive-relay-node-implements-kill-switch` — `{agent}:inactive` is injected into every imported belief's outlist (or as the sole default outlist entry) so retracting `{agent}:active` cascades all imported beliefs OUT.
- `only-in-flag-filters-by-source-status` — When `only_in=True`, claims whose parsed `status` is not `"IN"` are removed before node creation and never appear in `claims_imported` / `claims_skipped`.
- `import-agent-preserves-justifications-only-in-json-path` — Markdown STALE/OUT claims get empty `raw_justifications` and are retracted post-propagation, while JSON OUT claims retain justifications and are pinned OUT via `_retracted` metadata.
