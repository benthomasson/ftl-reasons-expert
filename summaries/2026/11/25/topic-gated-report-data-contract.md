---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The actual producer-side shapes for `gated_data`, `retracted_premises`, and `blocker_details`.

## Producer-side shapes

`reasons/report_gated.py` is the consumer, not the producer. The producer-side assembly is in `reasons/api.py` around lines 3708-3727:

- `gated_data = list_gated(**vis, **backend)` at `reasons/api.py:3708`
- `blocker_details = {}` at `reasons/api.py:3710`
- loop over `gated_data.get("blockers", {})` at `reasons/api.py:3711`
- `blocker_details[bid] = { ... }` at `reasons/api.py:3716`
- call `report_gated(gated_data, retracted, blocker_details, ...)` at `reasons/api.py:3721`

The observations do **not** show the full body of `list_gated`, the full body of the `blocker_details[bid] = { ... }` assignment, or where `retracted` is built. So the shapes below are the required/minimum contracts implied by the consumer and tests; extra keys may exist.

### `gated_data`

Produced by `list_gated(...)` at `reasons/api.py:3708`.

Required shape:

```python
{
    "blockers": {
        "<blocker_id>": {
            "text": str,
            "gated": [
                {"id": str, "text": str},
                ...
            ],
        },
        ...
    },
    "blocker_count": int,  # number of blocker IDs
    "gated_count": int,    # total number of gated beliefs across blockers
}
```

Evidence:

- `report_gated.py:49-50` directly indexes `gated_data["blocker_count"]` and `gated_data["gated_count"]`, so those keys must exist.
- `report_gated.py:73` does `gated_data.get("blockers", {})`, so `blockers` is treated as optional/defensively defaulted.
- `report_gated.py:77` sorts blockers using `len(kv[1]["gated"])`, so every blocker value must have a `gated` list.
- `report_gated.py:86` reads `bdata["text"]`.
- `report_gated.py:89-90` reads each gated belief’s `g["id"]` and `g["text"]`.
- `tests/test_report_gated.py:52-66` `_make_gated_data` mirrors this exact shape and computes `blocker_count` as `len(blockers)` and `gated_count` as `sum(len(b["gated"]) for b in blockers.values())`.

### `retracted_premises`

At the API call site, this is the local variable `retracted` passed as the second argument to `report_gated(...)` at `reasons/api.py:3721`. The observations do not show how `retracted` is constructed.

Required shape:

```python
[
    {
        "id": str,
        "text": str,
        "retract_reason": str,
        "dependent_count": int,
    },
    ...
]
```

It may be an empty list.

Evidence:

- `report_gated.py:58` checks `if retracted_premises:`.
- `report_gated.py:63` sorts by `r["dependent_count"]`.
- `report_gated.py:64-68` reads `rp["dependent_count"]`, `rp["text"]`, `rp["retract_reason"]`, and `rp["id"]`.
- `report_gated.py:127-130` again reads `rp["id"]`, `rp["text"]`, `rp["retract_reason"]`, and `rp["dependent_count"]` for LLM prompt formatting.
- `tests/test_report_gated.py:78-86` constructs retracted entries with exactly `id`, `text`, `retract_reason`, and `dependent_count`.

### `blocker_details`

Built in `reasons/api.py` starting at line 3710 and populated per blocker ID at line 3716. The key set corresponds to `gated_data.get("blockers", {})` because the loop at `reasons/api.py:3711` iterates over those blocker IDs.

Required shape:

```python
{
    "<blocker_id>": {
        "dependent_count": int,
        # other fields may exist, but report_gated.py only consumes dependent_count
    },
    ...
}
```

Evidence:

- `report_gated.py:80` does `detail = blocker_details.get(bid, {})`.
- `report_gated.py:81` reads `detail.get("dependent_count", 0)`.
- `report_gated.py:141-142` repeats the same lookup for LLM prompt formatting.
- Tests use `{"blocker-a": {"dependent_count": 3}}` in `TestBuildStructuredReport.test_includes_active_blockers` and `{"b1": {"dependent_count": 2}}` in `TestFormatDataForPrompt.test_includes_blockers`.
- Because `report_gated.py` uses `.get(..., {})` and `detail.get("dependent_count", 0)`, missing blocker-detail entries or missing `dependent_count` are tolerated at report time, but the producer intends to provide `dependent_count`.

### What is missing from the observations

To state the complete producer-side schemas rather than the minimum required contracts, we would need:

- The body of `list_gated(...)` to see whether `gated_data` contains extra keys or extra per-blocker/per-gated-belief fields.
- The full body of `blocker_details[bid] = { ... }` at `reasons/api.py:3716` to see fields beyond `dependent_count`.
- The code that builds `retracted` before `reasons/api.py:3721`.

## Topics to Explore

- [function] `reasons/api.py:list_gated` — Returns the actual `gated_data` schema, including any extra fields beyond those consumed by `report_gated.py`.
- [function] `reasons/api.py:report_gated` — Shows how `gated_data`, `retracted`, and `blocker_details` are assembled before being passed into `reasons.report_gated.report_gated`.
- [file] `reasons/report_gated.py` — Consumer-side reads define the minimum contract for all three structures.
- [general] `retracted-premises-producer` — Locate the query/builder for the `retracted` variable passed at `reasons/api.py:3721`.
- [general] `blocker-details-builder` — The missing body at `reasons/api.py:3716` determines whether `blocker_details` contains anything besides `dependent_count`.

## Beliefs

- `gated-data-contains-blockers-and-counts` — `gated_data` must contain `blockers`, `blocker_count`, and `gated_count`; the report code directly indexes the count fields.
- `gated-data-blockers-contains-gated-items` — Each `gated_data["blockers"][bid]` entry must contain `text` and a `gated` list whose items have `id` and `text`.
- `retracted-premises-contains-fix-metadata` — Each retracted premise entry must contain `id`, `text`, `retract_reason`, and integer `dependent_count`.
- `blocker-details-maps-to-dependent-count` — `blocker_details` maps blocker IDs to detail dicts containing at least `dependent_count`.
- `api-produces-report-gated-inputs` — `reasons/api.py` produces `gated_data`, `retracted`, and `blocker_details` and passes them to `report_gated` at `reasons/api.py:3721`.
