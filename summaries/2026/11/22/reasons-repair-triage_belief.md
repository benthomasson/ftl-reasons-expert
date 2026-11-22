---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: triage_belief in reasons/repair.py

# `triage_belief` — Explanation

## Purpose

`triage_belief` is the **classifier stage** in the repair pipeline. It is a thin, single-LLM-call wrapper that takes a belief whose justification was flagged invalid by review, and asks the model to pick one of four repair strategies (`search_and_link`, `soften`, `abandon`, `research`). It exists so that `repair_beliefs` can branch into the correct remediation routine without embedding prompt-construction or response-parsing logic in the orchestrator.

## Contract

**Preconditions (implicit, not enforced):**
- `belief_context` is a fully rendered text block (produced by `format_belief_for_review`) — the function does not format or validate it.
- `review_comment` is the (possibly empty) reviewer text explaining why the belief is invalid.
- `depth` and `flagged_ancestors` are non-negative integers; they are injected via `str.format`, so any value that stringifies is accepted. Negatives or floats are not rejected.
- `model` must be a valid identifier accepted by `invoke_model`.
- `invoke_model` must be reachable and return a string.

**Postconditions:**
- Returns a `dict` with exactly the keys `pattern` and `rationale` (see return value section). Both are nominally strings, but `rationale` may be `None` if the LLM emitted `"rationale": null` (see caveat below).
- Side-effect-free apart from the outbound LLM call.

**Invariants:**
- If the returned `pattern` is non-empty, it is guaranteed to be a member of `VALID_PATTERNS`.
- An empty `pattern` indicates *triage failed* and the caller is expected to handle it (it does — marks status `triage_failed`).

## Parameters

| Name | Meaning | Notes / Edge cases |
|---|---|---|
| `belief_context` | Full text of the invalid belief formatted for review | Raw-substituted into the prompt. No escaping — a belief containing `{` will **not** break formatting because it's a substitution *value*, not part of the template, but it *can* inject instructions to the model. |
| `review_comment` | Human review explanation | Same injection considerations. Often `""` when the reviewer left no comment. |
| `depth` | Derivation depth of the belief (0 = premise) | `str()`-ed by `.format()`. Strings are accepted; negatives are accepted. |
| `flagged_ancestors` | Count of antecedents that are also in the invalid set | Same caveat — used only as prompt context. |
| `model` | LLM identifier | Default `"claude"`. Must be understood by `invoke_model`; not validated here. |
| `timeout` | Seconds passed through to `invoke_model` | Default `300`. Not validated. |

## Return Value

A dict of the form:

```python
{"pattern": "<one of '', 'search_and_link', 'soften', 'abandon', 'research'>",
 "rationale": "<string or None>"}
```

- On success, `pattern ∈ VALID_PATTERNS` and `rationale` is the model's explanation.
- On parse failure (no well-formed JSON object in the response, or every JSON object fails the pattern check), returns `{"pattern": "", "rationale": ""}`.
- The caller **must** tolerate an empty pattern; `repair_beliefs` does so and records `triage_failed`.
- Caveat: `parse_triage_response` uses `obj.get("rationale", "")`. If the model returns `{"pattern": "soften", "rationale": null}`, the returned `rationale` is `None`, not `""`. Downstream `result["rationale"] = ...` will store `None`, which the repair result schema already allows (`None` default).

## Algorithm

1. **Render prompt** — `TRIAGE_PROMPT.format(...)` substitutes the four values into the triage template. `{{ ... }}` in the template collapse to literal `{ ... }` so the example JSON in the prompt renders correctly.
2. **Invoke model** — `invoke_model(prompt, model=model, timeout=timeout)`. No retry, no fallback.
3. **Parse** — `parse_triage_response(response)`:
   - Iterate over each `{` character in the response.
   - Try `json.JSONDecoder().raw_decode(response, i)` (allows trailing text after the object).
   - Skip failures / non-dicts.
   - Accept only if `pattern` is a member of `VALID_PATTERNS`; otherwise keep scanning.
   - Return the first match as `{"pattern", "rationale"}`.
   - Fall through to `{"pattern": "", "rationale": ""}` if nothing valid is found.

The scan-and-continue behavior means a leading chatty preamble like `"Here you go: {...}"` still parses, and a first object with an invalid pattern (e.g., a *different* JSON the model emitted earlier) will be skipped in favor of a later valid one.

## Side Effects

- **One outbound LLM request** per call via `invoke_model`. This is network I/O, may be slow, may incur cost.
- No file, DB, or global state mutation.
- No logging, no stderr writes (the caller `repair_beliefs` prints a `"Triaging ..."` line, not this function).

## Error Handling

- **Swallows parser-level failures**: malformed JSON is skipped, not raised. An entirely garbage response silently returns the empty-pattern dict.
- **Does not catch anything else**: any exception raised by `invoke_model` (network error, timeout, auth failure, rate limit) propagates to the caller. In practice `repair_beliefs` wraps this in `try/except Exception` and records `result["error"]`.
- **No retries**: a transient failure means the whole belief's repair falls back to the caller's error path.

## Usage Patterns

- Called exactly once per invalid belief inside `repair_beliefs`, after computing depth (`_compute_depth`) and `flagged_ancestors` from the current batch.
- Caller obligation: check `if not pattern:` and treat empty as triage failure; also handle any raised exception.
- Should not be called directly from user code — the orchestrator owns iteration, error capture, and pattern dispatch.

Typical call site:

```python
triage = triage_belief(belief_context, comment,
                       depth=depth,
                       flagged_ancestors=flagged_ancestors,
                       model=model, timeout=timeout)
pattern = triage.get("pattern", "")
```

## Dependencies

- `reasons/repair.py`: `TRIAGE_PROMPT` (module-level template with placeholders `{belief_context}`, `{review_comment}`, `{depth}`, `{flagged_ancestors}`), `parse_triage_response`, `VALID_PATTERNS`.
- `reasons/llm.py`: `invoke_model(prompt, model=..., timeout=...)`.
- Standard library: `json` (used indirectly via `parse_triage_response`).

## Assumptions not enforced by the type system

- That `TRIAGE_PROMPT` and this function stay in lockstep on placeholder names — renaming a placeholder breaks at `str.format` time with `KeyError`.
- That `invoke_model` returns a `str` (the code immediately calls `.strip()` inside the parser via `response.strip()`… actually that's in `parse_extract_response`; `parse_triage_response` iterates characters, which works on any iterable of chars but breaks on `None`).
- That `belief_context` and `review_comment` are trustworthy text — no prompt-injection hardening is done; a malicious belief can attempt to steer the model.
- That `depth` and `flagged_ancestors` are meaningfully formatted (e.g., integers); passing objects with a bad `__str__` produces nonsensical prompt text.
- That the caller will handle exceptions from `invoke_model` and empty `pattern` returns — this function is intentionally not defensive.

## Topics to Explore

- [function] `reasons/repair.py:parse_triage_response` — The JSON-scanning parser that defines what this function actually returns; worth reading to understand the "empty pattern" fallback and the null-rationale caveat.
- [function] `reasons/llm.py:invoke_model` — The single point of external I/O; determines what exceptions can propagate out of `triage_belief`.
- [function] `reasons/repair.py:repair_beliefs` — The only caller; shows how `depth`/`flagged_ancestors` are computed, and how triage failure and exceptions are handled.
- [file] `tests/test_repair.py` — Exercises the triage→dispatch path; likely has mocks for `invoke_model` and assertions on `VALID_PATTERNS` handling.
- [general] `prompt-injection-surface` — Understand how untrusted belief text flows into `TRIAGE_PROMPT` and whether downstream repair (e.g., `soften_belief`) has the same exposure.

## Beliefs

- `triage-belief-returns-parsed-dict` — `triage_belief` always returns a dict with keys `pattern` and `rationale`, never raising on malformed model output.
- `triage-belief-pattern-is-validated` — Any non-empty `pattern` in the returned dict is guaranteed to be a member of `VALID_PATTERNS`.
- `triage-belief-propagates-invoke-model-errors` — Exceptions from `invoke_model` (network, timeout, auth) are not caught inside `triage_belief`; the caller must handle them.
- `triage-belief-empty-pattern-signals-failure` — An empty `pattern` return indicates triage failure and is the caller's signal (via `if not pattern`) to mark `triage_failed`.
- `triage-belief-no-retry` — The function issues exactly one LLM call per invocation; no retry or fallback logic exists within it.
