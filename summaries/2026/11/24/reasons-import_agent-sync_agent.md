---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: sync_agent in reasons/import_agent.py

# `sync_agent` — `reasons/import_agent.py`

## Purpose

`sync_agent` is the public entry point for **incrementally reconciling** an external agent's markdown beliefs into the local `Network`, applying "remote wins" semantics. Unlike `import_agent` (which adds only new beliefs and skips existing ones), `sync_agent` treats the remote as authoritative:

- adds beliefs that are new in the remote
- updates text/metadata/justifications/truth on existing beliefs
- retracts (or marks `_retracted`) beliefs that exist locally under the prefix but are missing from the remote
- flips OUT → IN and IN → OUT to match the remote truth value

It is the markdown counterpart of `sync_agent_json`; both funnel into `_sync_claims`, differing only in how the input is normalized.

## Contract

**Preconditions (implicit — see "assumptions" below):**

- `network` is a live `Network` with `nodes`, `nogoods`, `repos`, `add_node`, `retract`, `assert_node`, `recompute_all`, and `_rebuild_dependents`.
- `beliefs_text` is markdown in the format understood by `reasons.import_beliefs.parse_beliefs`.
- `nogoods_text` (if non-empty) is in the format understood by `reasons.import_beliefs.parse_nogoods`.
- `agent_name` does not contain `:` and is not an existing node ID that could clash with the `{agent}:active` / `{agent}:inactive` conventions.

**Postconditions:**

- Namespaced nodes `{agent}:{claim_id}` reflect the remote state as observed in this call.
- `{agent}:active` and `{agent}:inactive` exist afterwards (created if missing).
- Claims that exist locally under `{agent}:` but not in `claims` are either retracted (if IN) or marked `_retracted=True` so they cannot be resurrected by `recompute_all`.
- Dependents registry is rebuilt and `recompute_all()` runs before the function returns.
- If `source_path` was supplied, `network.repos[agent_name]` points to the resolved parent directory of the source file.

**Invariants the code relies on:**

- `_sync_claims` runs `_fixup_dependents(network)` *after* adding any outlist references, so `add_node` can be called without those referents existing yet.
- Every imported belief's outlist has `inactive_id` prepended (from `_build_justifications`), so retracting `{agent}:active` cascades IN → OUT on all of that agent's beliefs.
- `_retracted=True` is the sentinel that makes both `_propagate()` and `recompute_all()` skip a node — the code assumes those methods respect that key.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `network` | `Network` | Mutable RMS network being reconciled | Must already exist and be usable; no defensive checks |
| `agent_name` | `str` | Namespace / prefix for all imported nodes | Empty string gives IDs like `":claim"`; containing `:` yields ambiguous prefixes |
| `beliefs_text` | `str` | Markdown document to parse | Empty/whitespace means "remote has nothing" — will trigger removal of all existing beliefs under this prefix |
| `nogoods_text` | `str \| None` | Optional markdown nogoods | Falsy → `[]`; not validated for consistency with `claims` |
| `only_in` | `bool` | Filter remote claims to `status == "IN"` | Danger: also filters antecedent/outlist references via `claim_ids = {c["id"] for c in claims}`, silently weakening IN beliefs whose dependencies are OUT in the remote; and OUT claims become "absent", so they are retracted as if removed |
| `source_path` | `str` | Path of the source document on disk | `""` → skip `repos` registration; metadata `imported_from` set to `""` regardless |

## Return Value

Returns the dict produced by `_sync_claims`:

```python
{
  "agent": str,
  "prefix": str,                    # f"{agent_name}:"
  "active_node": str,               # f"{agent_name}:active"
  "created_premise": bool,          # True iff active premise was created this call
  "beliefs_added": int,
  "beliefs_updated": int,           # text/source/justs/truth changed
  "beliefs_removed": int,           # present locally, absent remotely
  "beliefs_retracted": int,         # OUT claims with no justifications → retract_after
  "beliefs_unchanged": int,
  "beliefs_propagated": int,        # len(recompute_all())
  "nogoods_imported": int,
}
```

Note: `{agent}:inactive` is created but **not returned** (unlike `_import_claims`, which also isn't included here — this uses `_sync_claims`). Counters are informational only; the caller should not assume `added + updated + unchanged == len(claims)`, because missing remote claims drive `removed` and OUT-without-justification claims drive `retracted` in a separate pass.

## Algorithm

1. **Normalize** — `_normalize_markdown(beliefs_text, only_in)`:
   - Calls `parse_beliefs`.
   - If `only_in`, keeps only `status == "IN"` claims.
   - Computes `claim_ids` from the *filtered* set, then prunes each claim's antecedents/outlist down to IDs that still exist.
   - For `status in ("STALE","OUT")`, sets `is_out=True` and **`raw_justifications=[]`** — i.e., markdown imports always treat OUT claims as bare premises. Only `IN` claims retain their `SL` justification with filtered antecedents/outlist.
   - `_normalize_nogoods_markdown(nogoods_text)` returns `[]` if falsy, else parsed nongoods.

2. **Delegate** to `_sync_claims`:
   1. Ensure `{agent}:active` / `{agent}:inactive` exist (`_ensure_agent_nodes`).
   2. `network.repos[agent_name] = Path(source_path).resolve().parent` if `source_path` non-empty.
   3. `remote_ids = {prefix + c["id"]}`; `local_agent_ids` = every node starting with `prefix` except the two infra nodes.
   4. Topologically sort claims by antecedent references (`_topo_sort_claims`).
   5. For each claim:
      - **Existing node path:** diff text/source/source_url/source_hash/date; merge metadata (excluding `_*` keys); always set `imported_from`. Then:
        - `is_out and not raw_justifications`: build the fallback justification (empty antecedents, `inactive` in outlist), update if changed, enqueue `retract_after`.
        - `is_out and raw_justifications`: same justification update, but set `_retracted=True` and force `truth_value="OUT"`. This is the "locked OUT" state that survives `recompute_all`.
        - `not is_out`: update justifications, clear `_retracted`, and if it was OUT enqueue `assert_after`.
      - **New node path:** `add_node` with prefixed ID, metadata, and justifications; same OUT handling (enqueue `retract_after` for bare premise OUT, or set `_retracted` + `OUT` for justified OUT).
   6. Compute `removed_ids = local_agent_ids - remote_ids`; retract each IN node, and mark remaining (OUT, not already `_retracted`) nodes with `_retracted=True`.
   7. `_fixup_dependents(network)` → `network._rebuild_dependents()` (needed because outlist references were added after dependents could be registered by `add_node`).
   8. Process `assert_after`: pop `_retracted`, and if still OUT, call `network.assert_node(node_id)`.
   9. `propagated = len(network.recompute_all())` — single propagation pass.
   10. Process `retract_after` (OUT claims with no justifications): retract if currently not OUT, else stamp `_retracted=True`.
   11. `_import_nogoods(network, prefix, nogoods)` — dedupe by ID, keep only nogoods with ≥2 valid nodes.
   12. Return the stats dict.

Key ordering details worth noticing:

- **`assert_after` runs before `recompute_all`; `retract_after` runs after.** So OUT→IN assertions are consistent within the returned `believe_propagated` count, but IN→OUT retractions of bare-premise claims happen *after* propagation. Any cascading propagation caused by `network.retract()` in step 10 is **not** counted in `beliefs_propagated`. Correctness relies on `retract()` itself doing propagation.
- **Nogoods are imported after propagation**, so they don't trigger additional propagation in this call.
- **Metadata-only changes do not set `changed=True`.** So the `beliefs_updated` counter can miss metadata mutations (though `imported_from` is always rewritten).

## Side Effects

- Mutates `network.nodes`: adds, updates text/source/source_hash/date/metadata/justifications/truth_value; can also mutate `_retracted`.
- Mutates `network.nogoods` (append).
- Mutates `network.repos[agent_name]`.
- Calls `network.recompute_all()` (typically triggers `_propagate()` over all nodes).
- Calls `network.retract()` / `network.assert_node()` post-loop, each of which may itself propagate.
- Rebuilds the dependents index via `_rebuild_dependents()`.
- Calls `Path(source_path).resolve()` — touches the filesystem to resolve symlinks/relative paths.

## Error Handling

- **No `try`/`except` anywhere.** Any parse exception (`parse_beliefs`, `parse_nogoods`) or network-domain exception (unknown IDs, malformed justifications) propagates to the caller.
- Missing input is not distinguished from an empty remote: `beliefs_text=""` yields `claims=[]`, which drives every prefixed node into `removed_ids`.
- `source_path` pointing at a nonexistent path does not raise — `Path(...).resolve()` is tolerant of nonexistent paths by default in Python 3.6+.
- `nogoods_text=""` → `[]`; `nogoods_text=None` → `[]` (handled by `_normalize_nogoods_markdown`).
- Partial-application risk: the function mutates `network` incrementally inside the loop, so an exception mid-loop leaves the network in a half-synced state.

## Usage Patterns

- **CLI:** `reasons sync-agent rhel-expert ~/git/rhel-expert/beliefs.md` — pushes a fresh markdown snapshot into the network.
- **Companion to `import_agent`:** first import, then repeatedly sync to pick up changes as the remote belief file evolves.
- **Prefer `sync_agent_json`** when justification/outlist fidelity matters for OUT nodes. Markdown `OUT` claims lose justifications and take the "bare premise → retract after recompute" path; JSON `OUT` claims preserve justifications and use the `_retracted` + explicit `OUT` path, enabling resurrection when the remote flips to IN.
- **`only_in=True`** is appropriate for consumers who only care about what the remote currently asserts as true. Callers must be aware that this quietly strips dependencies on OUT claims (via the `claim_ids` filter in `_normalize_markdown`) and treats OUT claims as "removed", causing local retraction rather than a targeted OUT.
- **`source_path`** should be set when the caller wants the network to remember where beliefs came from (`network.repos[agent_name]`); omit it for programmatic/in-memory syncs.

## Assumptions the Type System Does Not Enforce

1. `agent_name` contains no `:` and no collision with an existing node prefix.
2. `source_path` is either `""` or an actual filesystem path; `Path.resolve()` will resolve it against CWD otherwise.
3. `network` exposes private helpers (`_rebuild_dependents`) and the `repos` attribute — not part of any typed protocol.
4. `Network.recompute_all()` and `Network.retract()` propagate correctly and honor `_retracted` metadata.
5. `Justification`/`Nogood` constructors accept the keyword arguments used by `_build_justifications` and `_import_nogoods`.
6. `parse_beliefs` yields dicts with the exact keys `id, text, status, depends_on, unless, source, source_hash, date, type`; `parse_nogoods` yields `id, affects, discovered, resolution`.
7. `_normalize_markdown` filters antecedents/outlist by exact string equality against `claim_ids` — no aliasing or reference resolution beyond that.

## Dependencies

- `reasons.import_beliefs.parse_beliefs`, `reasons.import_beliefs.parse_nogoods`
- `reasons.network.Network` (methods listed above)
- `reasons.Justification`, `reasons.Nogood` (used inside `_build_justifications` / `_import_nogoods`, which `sync_agent` reaches through `_sync_claims`)
- `pathlib.Path`
- In-directory: `_normalize_markdown`, `_normalize_nogoods_markdown`, `_sync_claims`, `_ensure_agent_nodes`, `_topo_sort_claims`, `_build_justifications`, `_justifications_match`, `_update_node_justifications`, `_import_nogoods`, `_fixup_dependents`

## Topics to Explore

- [function] `reasons/import_agent.py:_sync_claims` — The real reconciliation loop; all sync behavior (add/update/remove/retract/assert) lives here, and `sync_agent` is a thin wrapper around it.
- [function] `reasons/import_agent.py:_normalize_markdown` — Explains why markdown OUT claims end up as bare premises with no justifications, driving the `retract_after` path.
- [function] `reasons/network.py:recompute_all` — Confirms whether `_retracted` metadata actually suppresses propagation, and whether `retract()` re-propagates (impacting the `beliefs_propagated` count).
- [file] `tests/test_sync_agent.py` — Concrete verifications of remote-wins semantics: assertion on OUT→IN resurrection, removal retraction, and `_retracted` clearing.
- [general] `only-in-filtering-and-dependency-stripping` — The `only_in=True` code path strips antecedents to OUT claims; worth checking whether any caller relies on that behavior or whether it silently weakens imported IN beliefs.

## Beliefs

- `sync-agent-is-a-wrapper-for-sync-claims` — `sync_agent` normalizes markdown via `_normalize_markdown` and delegates the entire reconciliation to `_sync_claims`.
- `markdown-sync-produces-justification-less-out-nodes` — `_normalize_markdown` emits `raw_justifications=[]` for `OUT`/`STALE` claims, so `sync_agent` always takes the `retract_after` bare-premise path for those, unlike `sync_agent_json`.
- `retracted-flag-prevents-resurrection` — `_sync_claims` marks OUT-with-justifications nodes with `_retracted=True`, which keeps `_propagate()`/`recompute_all()` from flipping them IN until a subsequent sync clears the flag.
- `metadata-changes-do-not-count-as-updates` — Metadata-only mutations do not set `changed=True`, so they do not increment `beliefs_updated` even though `node.metadata["imported_from"]` is rewritten on every pass.
- `only-in-filters-antecedents-not-just-claims` — With `only_in=True`, `claim_ids` is computed from the filtered set, so IN claims lose antecedents/outlist entries that reference remote-OUT claims, silently weakening their justifications.
