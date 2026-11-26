---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How `review*.py` and `repair*.py` stage, gate, and write back approved changes to the belief graph.

# How `review*.py` and `repair*.py` stage, gate, and write back changes

**Short answer up front:** there is no dedicated staging table, no persisted "pending proposals" collection, and no formal approval gate anywhere in these modules. The search for `approve|gate|stage|proposed|apply|commit|write|persist` returned **zero matches** across the observed code. What actually exists is a two-phase pattern:

1. **Review phase** (`review*.py`, `propose_update.py`) — *read-only analysis*. These modules emit lists of plain dicts and never touch the database.
2. **Repair phase** (`repair.py`) — the *only* module in this set that writes to the belief graph, and the write-back is guarded by a single `dry_run` flag. `repair_premises.py` produces a *plan* but does not apply it.
3. **Human gate** — `propose_update.py` renders markdown containing literal `reasons update ...` / `reasons retract ...` shell commands for a human to run. `report_gated.py` reports on `OUT` beliefs but is not part of the approval workflow — its name is a false friend.

---

## 1. Staging: what gets produced, and where it lives

Nothing persists. Every review/repair module returns a Python `list[dict]` to its caller. The only injection seam is the optional `on_batch` / `on_result` callback, which fires *after each batch/result* with the running accumulation:

- `review.py:review_beliefs(..., on_batch=None)` — `on_batch(all_results)` after each LLM batch.
- `review_justifications.py:review_justifications(..., on_batch=None, parallel=0)` — same pattern, plus thread-pool support.
- `review_premises.py:review_premises(..., on_batch=None, parallel=0)` — same pattern.
- `repair_premises.py:repair_premises(..., on_result=None)` — callback per individual premise.
- `propose_update.py:propose_updates(..., on_batch=None)` — same.

So "staging" is the caller's responsibility: the caller must capture the returned list (or the callback payload) somewhere durable if it wants a staging layer. Nothing in these files does that for you.

### What each review module stages

- **`review.py:review_beliefs`** stages per-derived-belief verdicts with the dimensions `valid`, `sufficient`, `necessary`, plus `unnecessary_antecedents`, `scope_findings`, `missing_property`, and `defeat_reason_type` (see `REVIEW_PROMPT`). It filters candidates to `truth_value == "IN"` nodes that have at least one justification (`review.py:review_beliefs` body — the `if belief_ids is None` branch).
- **`review_justifications.py:review_justifications`** stages ALL/ANY/MIXED classifications with `required_antecedents` / `independent_antecedents`. Its docstring explicitly says: *"Reports candidates for conversion but does not modify the database."*
- **`review_premises.py:review_premises`** stages `accurate`, `well_scoped`, `error_type ∈ {misread_source, overgeneralized, fabricated, unsupported}` — plus `comment`.
- **`propose_update.py:propose_updates`** stages `action ∈ {update, retract}`, `failure_mode`, `basis ∈ {source-divergence, detected-contradiction, prior-knowledge}`, `proposed_text`, `evidence`, `comment`.

Note that `review.py:classify_defeat_reason` is defined but *not called* by `review_beliefs` — it's a standalone utility that callers must invoke themselves if they want the defeat-reason type re-derived from a defeater's text.

---

## 2. Gating: the three real gates

Because there is no approval store, "gating" happens at three concrete seams.

### Gate A — `dry_run` in `repair.py:repair_beliefs`

Every graph-mutating call in `repair.py:_do_search_and_link`, `soften`, `abandon`, and `research` is wrapped in `if not dry_run:`. With `dry_run=True` the code still runs the LLM triage/soften/matching and returns the same `status` dict (e.g. `"linked"`, `"softened"`, `"abandoned"`, `"needs_research"`), but performs zero `api.*` calls. This is the *only* mechanical gate in the pipeline.

### Gate B — human approval via `propose_update.py`

`propose_update.py:format_proposal_markdown` produces a suggested command for each proposal:

```
**Command:** `reasons update {nid} "{escaped}"`
**Command:** `reasons retract {nid} --reason "..."`
```

`format_proposals_file` assembles them all into a reviewable markdown document. The acceptance step is the human copy-pasting (or scripting) those commands. This is the closest thing to an approval workflow that exists — it's *out-of-band*, not code-enforced.

The `basis` taxonomy in `PROPOSE_UPDATE_PROMPT` is the closest thing to a "trust tier": `source-divergence` (mechanically verifiable, cheap to accept), `detected-contradiction`, and `prior-knowledge` (marked "needs scrutiny — reviewer may share your blind spot"). Nothing in the code *acts* on the basis tier differently; it is a formatting cue.

### Gate C — repair output shape in `repair_premises.py`

`repair_premises.py` is deliberately a *non-mutating* module. Look at the imports: `json`, `sys`, `concurrent.futures`, `invoke_model` — **no `from . import api`**. `_repair_one` returns `{"id", "action", "corrected_text", "rationale"}` and `repair_premises` just accumulates those dicts. This is a pure "produce the repair plan" surface — the caller must apply the `rewrite`/`retract` actions through `api.*` themselves. By contrast, `repair.py` *does* import `from . import api` inside both `_do_search_and_link` and `repair_beliefs`, and performs the writes inline.

---

## 3. Write-back: the exact `api.*` calls

All writes go through `api.*` in `repair.py:repair_beliefs`:

| Pattern | Write call | Side effect |
|---|---|---|
| `search_and_link` (`_do_search_and_link`) | `api.add_justification(belief_id, sl=",".join(new_ants), unless=",".join(original_outlist), label="repair: linked ...")` | Appends a **new** justification built from `original_ants + matched_ids`, reusing the original `outlist`. Existing justifications are not rewritten. Followed by `api.set_metadata(belief_id, "repair_action", "search_and_link")`. |
| `soften` | `api.supersede_with_text(belief_id, softened, db_path=...)` returns `{"new_id": ...}` | Creates a *new* belief node with the softened text; the original is superseded (left OUT in the TMS sense). Then `api.set_metadata(new_id, "repair_action", "softened")`. |
| `abandon` | `api.retract_node(belief_id, reason="repair: abandoned — <triage rationale>", db_path=...)` | Retracts the node (cascades to dependents). Then `api.set_metadata(belief_id, "repair_action", "abandoned")`. |
| `research` | `api.set_metadata(belief_id, "repair_research", rationale)` and `api.set_metadata(belief_id, "repair_action", "research")` | Metadata-only; the belief's truth value is untouched. Status becomes `"needs_research"`. |

The write-back is *not atomic across the batch*: each invalid belief is processed in its own `try/except` inside the `for r in invalid:` loop, and any `repair_action` metadata is written immediately after the corresponding mutation. Failures are recorded in `result["error"]` rather than raised.

Also note that `repair.py` defines `research_beliefs = repair_beliefs` at the bottom — the "research" pattern is *not* a separate function; it's a triage option within the same orchestrator. A separate `repair_smuggled_beliefs` function begins after the retracted section of the file (truncated in the observations) — it appears to be a legacy search-and-link-only wrapper that now coexists with the full triage-based `repair_beliefs`.

---

## 4. What "gated" means in `report_gated.py`

`report_gated.py` is a **reporting** module, not an approval module. Its docstring says "Identifies retracted premises (fixed defects) and active IN blockers (open problems)." The `gated_data["blockers"]` structure maps active `IN` blocker IDs → the beliefs they `gated` (held OUT). The `retracted_premises` list is derived from `retract_reason` metadata. This is read-only status reporting; it does not produce, approve, or apply changes. The name collision with the approval concept is coincidental.

---

## Summary table: who reads vs. who writes

| Module | Reads graph | Writes graph | Writes metadata | Calls LLM |
|---|---|---|---|---|
| `review.py` | ✓ (input `nodes`) | ✗ | ✗ | ✓ |
| `review_justifications.py` | ✓ | ✗ | ✗ | ✓ |
| `review_premises.py` | ✓ | ✗ | ✗ | ✓ |
| `propose_update.py` | ✓ | ✗ | ✗ | ✓ |
| `repair.py` | ✓ | ✓ via `api.*` | ✓ | ✓ |
| `repair_premises.py` | ✓ | ✗ | ✗ | ✓ |
| `report_gated.py` | ✓ (input data) | ✗ | ✗ | optional |

**Bottom line:** the "stage → gate → write-back" pipeline is really *review → (optional triage) → `repair_beliefs` with `dry_run` → `api.*`*. There is no first-class staging store; callers must persist review results themselves and decide when to promote them into a `repair_beliefs` call with `dry_run=False`. `propose_update.py` is the only module that actively encourages a human-in-the-loop gate, and it does so by emitting runnable commands rather than by holding state.

---

## Topics to Explore

- [general] `cli-bridge` — The CLI layer that wires `review_beliefs`/`review_premises` output into `repair_beliefs`/`repair_premises`; this is where the missing staging layer most likely lives, and where `dry_run` defaults are decided.
- [file] `reasons/api.py` — The `add_justification`, `supersede_with_text`, `retract_node`, and `set_metadata` primitives that every repair pattern ultimately invokes; their TMS semantics (outlist propagation, retraction cascade) determine whether repairs are truly sound.
- [function] `reasons/repair.py:repair_smuggled_beliefs` — Truncated in the observations; likely the legacy search-and-link-only entry point that predates the four-pattern triage in `repair_beliefs`.
- [function] `reasons/review.py:classify_defeat_reason` — Defined but never called inside `review_beliefs`; find who calls it (or confirm it's dead code) to understand the intended defeat-reason workflow.
- [general] `nogood-engine` — The `detected-contradiction` basis in `propose_update.py` and the implicit `outlist`/nogood handling in `add_justification` both depend on a contradiction detector; understanding it is necessary to reason about whether write-backs keep the network consistent.

## Beliefs

- `review-modules-are-read-only` — `review.py`, `review_justifications.py`, and `review_premises.py` import only `llm`/stdlib and never import `api`; all their outputs are in-memory `list[dict]` results.
- `repair-premises-does-not-write` — `repair_premises.py` has no `from . import api` import and returns repair plans (`action ∈ {rewrite, retract}`) without applying them.
- `repair-beliefs-is-the-only-writer` — `repair.py:repair_beliefs` is the only function in this set that mutates the belief graph, via `api.add_justification`, `api.supersede_with_text`, `api.retract_node`, and `api.set_metadata`.
- `dry-run-prevents-all-graph-writes` — Every `api.*` mutation in `repair.py` is guarded by `if not dry_run:`; with `dry_run=True` only LLM calls and result-dict construction occur.
- `soften-requires-supersede-with-text` — The `soften` pattern calls `api.supersede_with_text`, creating a new belief node and setting `repair_action=softened` on the new ID, not on the original.
- `propose-update-emits-human-commands` — `propose_update.py:format_proposal_markdown` generates literal `reasons update ...` / `reasons retract ...` shell commands; approval is executed by the human/CLI, not by the module.
- `research-pattern-is-metadata-only` — The `research` triage pattern writes only `repair_research` and `repair_action=research` metadata and leaves the node's truth value unchanged.
- `gated-report-is-not-approval` — `report_gated.py` reports on `OUT` beliefs blocked by `IN` blockers and retracted premises; "gated" there refers to TMS truth-value blocking, not approval gating.
