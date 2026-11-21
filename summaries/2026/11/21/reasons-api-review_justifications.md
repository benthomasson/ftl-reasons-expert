---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: review_justifications in reasons/api.py

# `review_justifications` — Explanation

## Purpose

`review_justifications` is a **read-only audit** entry point for the RMS (Reason Maintenance System). It identifies SL ("support link") justifications that have **more than one antecedent** and asks an LLM whether the intended semantics were **conjunctive (ALL — all antecedents required)** or **disjunctive (ANY — any one antecedent suffices)**.

The rationale: `add_node`/`add_justification` treat a single `Justification` with multiple antecedents as ALL-required. If the author actually meant "any one of these is enough," that belief would behave incorrectly (it stays OUT unless *all* hold). The `any_mode=True` flag on `add_node` exists precisely to emit *one SL justification per antecedent*, which is the ANY encoding. This function finds places where that might be warranted.

The function itself does **not mutate the database** — it only reports candidates and classification counts so a human (or a subsequent tool) can decide whether to re-model those justifications.

## Contract

**Preconditions**
- `db_path` points to an existing SQLite RMS database readable by `export_network`.
- `model` resolves to an LLM model the `.llm` module understands (e.g. `"claude"`).
- `min_antecedents >= 1` for the candidate filter to make semantic sense (values < 2 will surface single-antecedent or premise nodes, which is rarely what the caller wants).

**Postconditions**
- Database is unmodified (no writes, no `_with_network(..., write=True)`).
- Returns a dict with the keys documented in the return section below.
- The `results` list reflects whatever `_review` returned — the wrapper does **not** itself filter or re-order results.

**Invariants**
- `reviewed == len(results)` by construction.
- `convert_any + convert_mixed + keep_all` equals the number of results with a recognized `classification` — a `classification` that is none of `"ANY"`, `"MIXED"`, `"ALL"` (e.g. `None`, `"UNKNOWN"`) is silently dropped from all three counters.
- `total_candidates` is computed independently from `_review` and is **not asserted to equal `reviewed`** (see Assumptions).

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `belief_ids` | `list[str] \| None` | Restrict review to these IDs. | **An empty list is falsy** and is treated the same as `None` for the `candidates` filter (`if belief_ids:` skip), so `[]` does **not** mean "review nothing" — it falls back to "review everything." It is nonetheless forwarded verbatim to `_review`, so if `_review` interprets `[]` as "review none," `total_candidates` and `reviewed` will disagree. |
| `model` | `str` | LLM selector passed to `_review`; resolved downstream. `"claude"` is a symbolic alias, not a literal model ID. | Unknown names raise inside the LLM module. |
| `timeout` | `int` | Per-LLM-call timeout in seconds. | `0` or negative may abort every call. |
| `min_antecedents` | `int` | Minimum antecedent count for a justification to be a candidate. Default `2` = "multi-antecedent." | `1` matches all justifications; `0` may match premises (implementation-defined in `_has_multi_antecedent_sl`). |
| `on_batch` | `Callable \| None` | Progress callback forwarded to `_review`. Signature is whatever `_review` invokes it with. | Raising inside the callback aborts the review. |
| `parallel` | `int` | Worker count forwarded to `_review`; `0` typically means sequential / default. | Values > 1 fan out concurrent LLM calls. |
| `db_path` | `str` | Path to the SQLite database. | Nonexistent path → `Storage` will create an empty one, or `export_network` raises in the cached context — behavior depends on `_with_network`. |

## Return Value

```python
{
  "results":          list[dict],   # per-justification review, e.g. {"id": ..., "classification": "ALL"|"ANY"|"MIXED"|...}
  "reviewed":         int,          # == len(results)
  "convert_any":      int,          # count(classification == "ANY")
  "convert_mixed":    int,          # count(classification == "MIXED")
  "keep_all":         int,          # count(classification == "ALL")
  "total_candidates": int,          # count of IN multi-antecedent-SL nodes (∩ belief_ids if truthy)
}
```

- **Empty network or no qualifying candidates**: `results=[]`, all counters `0`, `total_candidates=0`. No exception.
- **Any classification outside the three recognized strings** is invisible to the three converters but still counts in `reviewed`.

The caller must handle the case where `total_candidates != reviewed`; that signals a mismatch between the wrapper's filter and `_review`'s internal filter (or an LLM-side skip).

## Algorithm

1. **Import the underlying engine**
   ```python
   from .review_justifications import review_justifications as _review, _has_multi_antecedent_sl
   ```
   `_has_multi_antecedent_sl(node, min_antecedents)` is the predicate that decides "is this an SL justification with ≥ N antecedents?"

2. **Export the entire network** via `export_network(db_path=db_path)` → `nodes` dict keyed by node ID, each entry containing `truth_value`, `justifications`, `metadata`, tags, etc. This uses `_with_network` read-only with the mtime cache, so repeated calls against an unchanged DB reuse the in-memory network.

3. **Compute the reporting candidate set**
   ```python
   candidates = {
       nid: node for nid, node in nodes.items()
       if node.get("truth_value") == "IN"
       and _has_multi_antecedent_sl(node, min_antecedents)
   }
   if belief_ids:
       candidates = {k: v for k, v in candidates.items() if k in belief_ids}
   total_candidates = len(candidates)
   ```
   This is used **only** to fill `total_candidates`.

4. **Delegate the actual review** to `_review(nodes, belief_ids=belief_ids, model=model, timeout=timeout, min_antecedents=min_antecedents, on_batch=on_batch, parallel=parallel)` — note the **unfiltered** `belief_ids` (not `candidates.keys()`) is passed. `_review` is expected to apply its own IN + `_has_multi_antecedent_sl` filter.

5. **Aggregate** classification counts with `sum(...)` over `results` and return.

## Side Effects

- **Reads the database** through `export_network` → `_with_network(db_path, write=False)` (cache-hits when the file mtime hasn't changed, otherwise loads via `Storage`).
- **No database writes**: no `_with_network(..., write=True)`, no metadata stamped (contrast with `review_beliefs`, which sets `reviewed_at` / `review_result`).
- **External I/O to an LLM** via `_review` → indirect calls to `.llm.invoke_model`. This can be slow, costly, and network-dependent.
- **May print progress** to stderr or invoke `on_batch`, depending on `_review`'s implementation.
- **Does not touch the proposal table, nomerkl hashes, or pubsub.**

## Error Handling

- **No `try/except`** in the wrapper. All exceptions propagate.
- `export_network` can raise `FileNotFoundError` / `sqlite3.DatabaseError` if the DB is missing or corrupted.
- `_review` can raise on LLM transport failures, malformed prompts, JSON parse failures if the underlying implementation doesn't catch them internally, or if `on_batch` raises.
- Empty candidate sets are **not errors** — the function returns zeroed counters rather than raising.
- The wrapper does not validate `belief_ids` against the network; unknown IDs are silently ignored (both in `candidates` and, presumably, inside `_review`).

## Usage Patterns

Typical audit workflow:

```python
# 1. Audit everything — see how many justifications were modeled as ALL
#    but should have been ANY.
report = review_justifications(db_path="reasons.db")
if report["convert_any"] or report["convert_mixed"]:
    ids_to_fix = [r["id"] for r in report["results"]
                  if r["classification"] in ("ANY", "MIXED")]

    # 2. Re-model as ANY — add_node supports any_mode=True which emits one
    #    SL justification per antecedent. (Not done by this function.)
    # add_node(..., sl=",".join(antecedents), any_mode=True)
```

Or drill into a subset (e.g. the results of `review_beliefs`):

```python
review_justifications(belief_ids=flagged_ids, min_antecedents=3, parallel=4)
```

Caller obligations:
- Treat it as **recommendations only** — nothing is persisted.
- Do not rely on `total_candidates == reviewed`; check both if you need a strict count.
- Avoid passing `[]` if "review nothing" is intended — pass `None` and skip the call, or pre-filter yourself.

## Dependencies

- **Internal (same file)**: `export_network` (uses `_with_network`, `_is_visible`, `build_meta`), `DEFAULT_DB`.
- **Internal (submodule)**: `.review_justifications.review_justifications` and `._has_multi_antecedent_sl` — the actual logic.
- **Transitive**: `.llm.invoke_model` (via `_review`), `sqlite3` (via `Storage`), `pathlib`/`os` (via `_with_network` cache mtime check).
- **No PG dispatch**: unlike most API functions, this one takes **no `pg_conninfo` / `domain_id`** — it is SQLite-only. Running this against a PostgreSQL backend is not supported.

## Assumptions not enforced by the type system

1. **`_review` re-applies the same filter.** The wrapper computes `candidates` with `truth_value == "IN" and _has_multi_antecedent_sl(...)`, but passes the *original* `belief_ids` to `_review`. If `_review` uses different criteria (e.g., it doesn't filter to IN, or uses a different `min_antecedents` interpretation), `reviewed` will diverge from `total_candidates`. This is exactly the bug pattern avoided by `review_beliefs`, which builds `review_ids = sorted(candidates.keys())` and passes *that* to `_review`.
2. **`node` dict shape.** `_has_multi_antecedent_sl` expects the exact dict produced by `export_network` (keys `truth_value`, `justifications` with `type` and `antecedents`). The wrapper only checks `truth_value` itself, so shape mismatches inside `_has_multi_antecedent_sl` will raise there.
3. **`classification` string vocabulary.** The counters assume the LLM classification uses exactly `"ALL"`, `"ANY"`, `"MIXED"`. Anything else (e.g. `"all"`, `None`, provider errors returning `{}`) silently drops out of the three aggregates but still increments `reviewed`.
4. **`belief_ids=[]` means "all".** The truthiness check `if belief_ids:` skips filtering for an empty list.

## Topics to Explore

- [function] `reasons/review_justifications.py:review_justifications` — The real engine: how it prompts the LLM, what classification strings it emits, whether it internally filters by IN and multi-antecedent SL (this is the crux of the divergence risk noted above).
- [function] `reasons/review_justifications.py:_has_multi_antecedent_sl` — Defines the candidate predicate; look at how it counts `justifications[i].antecedents` and whether it respects `min_antecedents <= 0`.
- [function] `reasons/api.py:add_node` — The `any_mode=True` flag is the remediation path: it expands a single multi-antecedent SL into one SL per antecedent (the ANY encoding). Understanding this closes the loop on how audit results get applied.
- [function] `reasons/api.py:review_beliefs` — Compare its filter-then-delegate pattern (`review_ids = sorted(candidates.keys())` then `_review(nodes, belief_ids=review_ids)`) against `review_justifications`, which passes the raw `belief_ids` — highlights the potential divergence in this wrapper.
- [general] `llm-classification-round-trip` — How provider/model invocation is executed, retried, or batched in `_review`, since `parallel`, `on_batch`, and `timeout` all funnel there and drive the function's failure modes.

## Beliefs

- `review-justifications-is-read-only` — `review_justifications` never opens a write-mode `_with_network` context; `export_network` reads and the LLM results are returned without touching `reviewed_at`, `updated_at`, or any metadata column.
- `review-justifications-requires-in-and-multi-antecedent-sl` — The candidate set counted in `total_candidates` is exactly nodes with `truth_value == "IN"` whose justifications satisfy `_has_multi_antecedent_sl(node, min_antecedents)`.
- `review-justifications-does-not-guarantee-reviewed-equals-total-candidates` — `total_candidates` is computed in the wrapper from a locally filtered dict, but `reviewed` is `len(results)` from `_review`, which is passed the *unfiltered* `belief_ids`; the equality holds only if `_review` re-applies the identical filter.
- `review-justifications-treats-empty-list-as-none` — The `if belief_ids:` guard makes `belief_ids=[]` behave identically to `belief_ids=None` for candidate selection (it does not request "review nothing").
- `review-justifications-is-sqlite-only` — Unlike most peers in `api.py`, `review_justifications` accepts no `pg_conninfo`/`domain_id` and has no `_pg_dispatch` branch, so it cannot target a PostgreSQL backend.
- `review-justifications-classification-strings-drive-counters` — `convert_any`, `convert_mixed`, and `keep_all` only count results whose `classification` is exactly `"ANY"`, `"MIXED"`, or `"ALL"`; any other value still increases `reviewed` but is omitted from all three counters.
