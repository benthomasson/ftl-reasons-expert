---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Compare how `dry_run` is threaded through `repair_premises` vs. other `api.*` mutations (`propose_update`, `compact`, etc.) to confirm a consistent convention.

## Comparison: how `dry_run` is threaded through `repair_premises` vs. other `api.*` mutations

### What the data actually shows

There are two very different layers at play here, and the "convention" lives at only one of them:

1. **`api.` wrapper layer** (`reasons/api.py`): the public, DB-touching boundary.
2. **Pure helper modules** (`reasons/repair_premises.py`, `reasons/propose_update.py`, `reasons/compact.py`): computations that take data in and return data out, with no persistence.

The consistent pattern in the code shown is that **`dry_run` belongs (if anywhere) at the `api.` persistence boundary, not inside the pure helpers**:

- `reasons/repair_premises.py:129` — `repair_premises(nodes, premise_ids, source_contents, review_results, model="claude", timeout=300, parallel=0, on_result=None)`. No `dry_run`. It runs the LLM and returns a list of `{"id", "action", "corrected_text", "rationale"}` results; it never touches the database, so simulation is already inherent.
- `reasons/propose_update.py:propose_updates(nodes, belief_ids=None, model=..., timeout=..., batch_size=..., on_batch=None)` — no `dry_run` either. It is a *proposal generator*: the caller decides whether to apply. Simulation is baked into the contract instead of a flag.
- `reasons/compact.py:compact(network, budget=500, truncate=True)` — read-only formatter returning a string; `dry_run` is meaningless.

Compare that to the DB-touching `api.` mutators in the sampled excerpt (lines 0–800 of `reasons/api.py`):

- `add_node`, `add_justification`, `retract_node`, `assert_node`, `propagate`, `ensure_namespace` all open `_with_network(db_path, write=True)` and persist unconditionally. None accept `dry_run`.
- The simulation counterparts are **separate functions**, not a `dry_run` flag: `what_if_retract`, `what_if_assert`, `what_if_supersede` (all in the same sample). They use the non-writing branch of `_with_network(db_path, write=False)` plus `copy.deepcopy(net)` before mutating in memory. This is the "preview" convention for retract/assert/supersede.

### Where `dry_run` actually shows up for `repair_premises`

The sample never includes the body of `api.repair_premises` (defined at `reasons/api.py:4347`, well outside the 800-line window that ends around `get_status`). So we infer its signature from its callers in `tests/test_repair_premises.py`:

- `TestApiRepairPremises.test_dry_run_no_changes` calls `api.repair_premises(review_file=review_file, dry_run=True, db_path=db)` and asserts the original node text is unchanged.
- `TestRepairActionMetadata.test_dry_run_no_repair_action` calls the same and asserts `show_node("obs-1")["metadata"].get("repair_action") is None`.
- `TestApiRepairPremises.test_rewrite_updates_node` / `test_retract_removes_node` call the same API *without* `dry_run` and assert rewrites supersede the old node (old node `OUT`) and retracts flip the node `OUT`.

So the inferred contract is: `api.repair_premises` runs the LLM/parse work (delegated to `reasons/repair_premises.py`), and only the *apply* step (supersede / retract / set `repair_action` metadata) is gated by `dry_run`. The `dry_run` behavior therefore resembles `what_if_*` semantics but exposed as a parameter rather than a separate function.

### Consistency verdict — partial, and worth flagging a data gap

Observed state:

| API surface | Mechanism for simulation | Notes |
|---|---|---|
| `api.repair_premises` | `dry_run=` kwarg (per tests) | Only mutator in the code we have that takes this flag |
| `api.what_if_retract` / `what_if_assert` / `what_if_supersede` | Separate functions, `write=False` + `deepcopy` | The dominant "preview" idiom in `api.py` |
| `api.add_node`, `add_justification`, `retract_node`, `assert_node`, `propagate`, `ensure_namespace` | None | Always write; no way to preview |
| `reasons/propose_update.py:propose_updates` | Inherent (returns proposals only) | Caller applies |
| `reasons/compact.py:compact` | Inherent (read-only) | N/A |

So the convention is **not uniform**. Two idioms coexist:
- **Idiom A** — `what_if_*` helper functions for retract/assert/supersede.
- **Idiom B** — a `dry_run` kwarg, seen only on `api.repair_premises`.

The underlying "compute" functions (`reasons/repair_premises.py`, `reasons/propose_update.py`, `reasons/compact.py`) never take `dry_run`; they are pure.

### Important caveat about the observations

There is a contradiction in the provided data that should be resolved before relying on this comparison:

- `find_dry_run_usage_in_reasons` reports **0 matches** for `dry_run` anywhere under `reasons/`.
- Yet `tests/test_repair_premises.py` calls `api.repair_premises(..., dry_run=True, ...)` in two tests.

Either (a) the grep scope missed `reasons/api.py` at line ~4347 (which is exactly outside the returned sample), or (b) `api.repair_premises` does not in fact accept `dry_run` and the two tests are currently red/to-be-implemented. You cannot distinguish these from the observations alone — you must read `reasons/api.py` around line 4347.

## Topics to Explore

- [function] `reasons/api.py:repair_premises` — Read the full body (starting line 4347) to verify whether `dry_run` is a real parameter and where exactly it gates the apply step (supersede/retract/metadata writes).
- [file] `reasons/cli.py` — Check how `cmd_repair_premises` and any `--dry-run` argparse flag are wired, confirming `dry_run` is a CLI-visible option for this command only.
- [general] `dry_run-grep-discrepancy` — Rerun the `dry_run` search scoped to `reasons/api.py` specifically to resolve the 0-match result against the test callers.
- [function] `reasons/api.py:what_if_supersede` — The canonical "simulate a mutation" pattern in this codebase; good reference for whether `repair_premises` should adopt `what_if_*`-style previewing instead of a `dry_run` flag.
- [file] `tests/test_repair_premises.py` — Enumerate all `dry_run=True` call sites to see which acceptance criteria (no text change, no `repair_action` metadata) the `dry_run` path must satisfy.

## Beliefs

- `repair-premises-helper-is-pure` — `reasons/repair_premises.py:repair_premises` accepts no `dry_run` parameter and performs no DB writes; it only runs LLMs and returns result dicts.
- `api-repair-premises-is-the-dry-run-gate` — Per `tests/test_repair_premises.py::test_dry_run_no_changes` and `::test_dry_run_no_repair_action`, `api.repair_premises(..., dry_run=True, ...)` must leave node text and `repair_action` metadata untouched, implying the `dry_run` gate lives in `reasons/api.py:repair_premises` (line 4347).
- `what-if-x-implements-simulation-via-separate-functions` — `api.what_if_retract`, `api.what_if_assert`, and `api.what_if_supersede` (in `reasons/api.py`) implement non-mutating preview via `_with_network(db_path, write=False)` and `copy.deepcopy`, not via a `dry_run` kwarg.
- `add-node-always-writes` — `api.add_node` in `reasons/api.py` opens `_with_network(db_path, write=True)`, has no `dry_run` parameter, and cannot be previewed.
- `propose-updates-produces-proposals-without-applying` — `reasons/propose_update.py:propose_updates` returns structured proposal dicts and never mutates the network; application is left to the caller, so `dry_run` is unnecessary by design.
- `dry-run-grep-result-contradicts-tests` — `find_dry_run_usage_in_reasons` reports 0 matches under `reasons/`, but tests call `api.repair_premises(dry_run=True, ...)`, so the search scope is either incomplete or `api.repair_premises` currently lacks the parameter the tests expect.
