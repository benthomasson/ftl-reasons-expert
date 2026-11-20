---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: detect_contradictions in reasons/api.py

## `detect_contradictions` — `reasons/api.py`

### Purpose

`detect_contradictions` is a **functional API wrapper** around the LLM-driven contradiction scanner in `reasons.contradictions`. It exists so that callers (CLI, LangGraph tools, scripts) can ask "do my currently-`IN` beliefs contradict each other?" without knowing about the contradiction module's batching, clustering, or file-writing logic. It optionally **auto-applies** each detected contradiction as a nogood (which is a *mutating* operation against the belief network).

It is the "detect + optionally resolve" entry point, distinct from the plan/review flow (`write_contradiction_plan` / `parse_contradiction_plan` / `apply_contradiction_plan`) that lives further down the file.

---

### Contract

**Preconditions**
- A SQLite database exists at `db_path` (or `export_network` will attempt to create/load one — see `_with_network`).
- If `auto_apply=True`, every contradiction returned by the detector must carry a `claims` key that is a list of node IDs that `add_nogood` can resolve.
- No `pg_conninfo` / `domain_id` parameters — **this function has no PostgreSQL backend**, unlike almost every other API function. Calling with a Postgres-only deployment is not supported here.

**Postconditions**
- Returns a dict with keys: `contradictions`, `checked`, `found`, `applied`, `applied_details`, `total_in`.
- If `output_path` is provided, the detector writes/updates that file. The wrapper itself does not write.
- If `auto_apply=True` and at least one contradiction is applied, the database is mutated (nodes may go `OUT` due to backtracking in `add_nogood`).

**Invariants**
- Only nodes with `truth_value == "IN"` are ever candidates. The detector receives the *full* `nodes` dict, but `check_ids` is restricted to `IN` nodes.
- `total_in` is measured **before** `belief_ids` and `sample` filtering, so `checked <= total_in`.
- `found == len(contradictions)` always holds.

---

### Parameters

| Param | Type | Notes |
|---|---|---|
| `belief_ids` | `list[str] \| None` | Optional restriction. Intersected with the set of `IN` nodes. IDs that don't exist, or that exist but are `OUT`, are **silently dropped**. ⚠️ An **empty list is falsy**, so `belief_ids=[]` means "no filter", not "check nothing". |
| `model` | `str` | LLM model name forwarded to the detector. Default `"claude"`. |
| `timeout` | `int` | Per-LLM-call timeout in seconds, forwarded. |
| `sample` | `int \| None` | If not `None` and the candidate count exceeds it, picks `sample` keys via `random.sample(sorted(keys), sample)`. No `seed` parameter — sampling is non-deterministic across calls (though candidate order is sorted for the *reported* order). ⚠️ Negative values raise `ValueError` from `random.sample` (uncaught). |
| `auto_apply` | `bool` | If `True`, each contradiction is applied via `add_nogood`. |
| `semantic` | `bool` | Selects `detect_contradictions_semantic` instead of `detect_contradictions`. Semantic mode clusters beliefs by embedding before LLM analysis. |
| `embedding_model` | `str \| None` | Only meaningful when `semantic=True`; silently ignored otherwise. |
| `output_path` | `str \| None` | Passed straight through to the detector for incremental writing. The wrapper does no validation of the directory or the path. |
| `db_path` | `str` | SQLite DB path; default `DEFAULT_DB = "reasons.db"`. |

---

### Return Value

```python
{
    "contradictions": list[dict],   # detector output, each with at least "id" when auto_apply
    "checked":        int,          # candidates sent to the detector (post filter + sample)
    "found":          int,          # == len(contradictions)
    "applied":        int,          # number successfully converted to nogoods (0 if !auto_apply)
    "applied_details": list[dict],  # per-contradiction {id, nogood_id, changed}
    "total_in":       int,          # IN-belief count before belief_ids/sample filtering
}
```

Callers must handle two failure **modes** returned in-band:

1. **`found > applied`** when `auto_apply=True` — some detections failed to apply. The wrapper *does not* surface the errors in the return value, only to `stderr` (see below).
2. **`found == 0`** with `checked > 0` — no contradictions found (also a normal, successful outcome).

---

### Algorithm

1. Import `detect_contradictions` (and, if `semantic`, `detect_contradictions_semantic`) from `.contradictions`.
2. `export_network(db_path=db_path)` — loads the full network as a dict (all nodes, all truth values, no access-tag filtering). Populates `nodes`.
3. Build `candidates = {node_id: node for ... if node["truth_value"] == "IN"}`; record `total_in`.
4. If `belief_ids` is truthy, restrict `candidates` to the intersection.
5. If `sample` is set and smaller than `len(candidates)`, randomly sample that many keys.
6. `check_ids = sorted(candidates.keys())` — sorted for deterministic dispatch order.
7. Dispatch:
   - If `semantic`: `_detect_semantic(nodes, belief_ids=check_ids, model=model, timeout=timeout, embedding_model=embedding_model, output_path=output_path)`.
   - Else: `_detect(nodes, belief_ids=check_ids, model=model, timeout=timeout, output_path=output_path)`.
   - Note: `nodes` (the *full* network) is passed, not just candidates — the detector may use non-candidate nodes as context.
8. If `auto_apply`: for each `c` in `contradictions`, call `add_nogood(c["claims"], db_path=db_path)`. On success, bump `applied` and append to `applied_details`. On *any* exception, print a warning to `stderr` and continue.
9. Return the summary dict.

---

### Side Effects

- **Reads** the SQLite DB via `export_network`.
- **Writes** to `output_path` (delegated to the detector).
- **Mutates** the DB when `auto_apply=True`: `add_nogood` may retract beliefs as part of dependency-directed backtracking, cascading `IN → OUT` for dependents. This is the most consequential side effect and is **not** reversible by anything in this function.
- **Calls an external LLM** (potentially many times — once per batch/cluster in the detector).
- **Prints to `stderr`** on auto-apply failures: `WARN: failed to apply {c['id']}: {e}`.

---

### Error Handling

- Exceptions from `export_network` (e.g., DB corruption) propagate.
- Exceptions from the detector (e.g., LLM API failure, invalid `output_path`) propagate.
- **`auto_apply` errors are swallowed** — a per-contradiction failure (`KeyError` on `c["claims"]`, `add_nogood` rejection, missing node ID, etc.) is caught and only warned to `stderr`. It is **not** added to `applied_details` and **not** reported in the return dict. A caller wanting to know *which* applications failed must compare `found` vs `applied` and inspect logs.
- A malformed contradiction dict (e.g., missing `"claims"`) triggers the same swallow path.
- `random.sample` raises `ValueError` for negative `sample` or `sample > len(candidates)` — the latter is guarded, the former is not.
- `random.sample` also raises `TypeError` if `sample` is non-int — not validated.

---

### Usage Patterns

Typical call from a CLI or agent:

```python
# Detect only, no mutation
res = detect_contradictions(db_path="reasons.db")
print(f"{res['found']} contradictions among {res['checked']} IN beliefs")

# Focused check
res = detect_contradictions(
    belief_ids=["auth-1", "auth-2"],
    model="claude",
    sample=50,
)

# Detect + auto-resolve in one shot (mutating!)
res = detect_contradictions(auto_apply=True, semantic=True, embedding_model="all-MiniLM-L6-v2")
if res["applied"] < res["found"]:
    log.warning("Some contradictions could not be applied: %d/%d",
                res["applied"], res["found"])

# Review-before-apply workflow (safer)
res = detect_contradictions(output_path="contradictions.md")
# then use parse_contradiction_plan + apply_contradiction_plan
```

**Caller obligations:**
- Don't pass `belief_ids=[]` expecting "check nothing" — you get the full IN set.
- Don't rely on `applied < found` unless you've wired stderr/log capture; the return dict has no `errors` key.
- When `auto_apply=True`, understand that `add_nogood` performs backtracking retractions, which can cascade.

---

### Dependencies

- `reasons.api.export_network` — loads the network.
- `reasons.api.add_nogood` — applies contradictions (mutating).
- `reasons.contradictions.detect_contradictions` / `detect_contradictions_semantic` — the actual detection logic (LLM calls, batching, optional incremental file writing).
- Transitive (semantic path): the `reasons.contradictions` module's embedding/clustering helpers.
- `random`, `sys` (stdlib).

---

### Assumptions Not Enforced by the Type System

- Contradiction dicts are assumed to contain a `"claims"` key with resolvable node IDs when `auto_apply=True`; the type is only `list[dict]`.
- `belief_ids` values are assumed to be strings that could match node IDs; no validation, and mismatches are silent.
- `output_path` is assumed to point to a writable directory; no pre-check.
- `embedding_model` is only consumed in `semantic=True` mode; passing it otherwise is a silent no-op.
- An empty `belief_ids` list is treated as "no filter" (falsy check), not "empty candidate set".
- The wrapper assumes the caller is on the SQLite backend; there's no `pg_conninfo` path.

---

## Topics to Explore

- [function] `reasons/contradictions.py:detect_contradictions` — The non-semantic batched LLM detector; shows how `output_path` incremental writes happen and what a contradiction dict actually contains (`id`, `claims`, `analysis`, `severity`).
- [function] `reasons/contradictions.py:detect_contradictions_semantic` — The embedding-clustered variant; explains how `embedding_model` is consumed and why cluster-based analysis differs.
- [function] `reasons/api.py:add_nogood` — The mutating operation behind `auto_apply`; shows why retraction cascades can occur and what `changed` / `backtracked_to` mean.
- [function] `reasons/api.py:export_network` — The loader used here; explains why the wrapper passes the *full* node dict (including `OUT` nodes) to the detector even though only `IN` nodes are checked.
- [file] `reasons/contradictions.py` — Full module; also relevant is `write_contradiction_plan` / `parse_contradiction_plan` / `apply_contradiction_plan` in `reasons/api.py` for the review-before-apply workflow that `detect_contradictions`'s `output_path` can feed into.

---

## Beliefs

- `detect-contradictions-only-checks-in-beliefs` — Only nodes whose `truth_value` is `"IN"` are ever candidates, regardless of what `belief_ids` contains.
- `detect-contradictions-silently-ignores-unknown-ids` — IDs in `belief_ids` that are missing or `OUT` are dropped without any error or count in the return dict.
- `detect-contradictions-auto-apply-swallows-errors` — Failures during `add_nogood` are printed to `stderr` and are not raised, and are not represented in the return dict (only detectable via `applied < found`).
- `checked-is-bounded-by-total-in` — `total_in` is captured before `belief_ids`/`sample` filtering, so `checked <= total_in` always holds.
- `detect-contradictions-auto-apply-mutates-db` — With `auto_apply=True`, the call records nogoods via `add_nogood`, which can retract premises and cascade `IN → OUT` dependent beliefs.
- `detect-contradictions-delegates-to-contradictions-module` — All detection, LLM batching, semantic clustering, and output-file writing live in `reasons/contradictions.py`; the wrapper only filters, samples, dispatches, and optionally applies.
- `detect-contradictions-has-no-postgres-backend` — Unlike most functions in `reasons/api.py`, this one omits `pg_conninfo`/`domain_id` and therefore supports only the SQLite backend.
