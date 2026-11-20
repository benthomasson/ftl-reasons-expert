---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: classify_defeat_reason in reasons/review.py

# `classify_defeat_reason` — Explanation

## Purpose

`classify_defeat_reason` is a best-effort single-shot classifier that asks an LLM which *logical failure mode* a TMS defeater represents. It exists to normalize free-form LLM output into one of the seven canonical strings in `DEFEAT_REASON_TYPES` (`unsupported-conjunct`, `over-generalizes`, `false-causal-claim`, `internal-contradiction`, `circular-reasoning`, `missing-bridge`, `scope-mismatch`).

It is the "single-defeater" counterpart to the bulk `defeat_reason_type` field that `review_beliefs` collects in `REVIEW_PROMPT`. Both share the same vocabulary, and both are matched against `DEFEAT_REASON_TYPES`.

## Contract

**Preconditions**
- `defeater_text` and `defeated_text` are strings suitable for insertion into `{defeater_text}` / `{defeated_text}` template holes. They are *not* validated; if they contain `{}`-brace content it does not matter (format only substitutes the two named fields), but a `None` would raise `AttributeError` inside `.format` and is outside the try block (see Error Handling).
- `model` is a string identifying the model (e.g. `"claude"`); it is opaque to this function.
- `timeout` is a numeric timeout in seconds; also opaque here and simply forwarded.
- `invoke_model` must return a `str` (or something truthy-string-like). This is **not enforced by the type system** — see "Assumptions not enforced" below.

**Postconditions**
- Returns either a member of `DEFEAT_REASON_TYPES` (exact string, lowercased) or the empty string `""`.
- Returns `""` if: the LLM call raised any `Exception`, or the returned text did not contain any of the seven canonical strings.

**Invariant**
- The return value is never `None`, never a non-listed string, and never a partially-matched string. Callers can safely use `if result:` / `if result == "..."`.

## Parameters

| Name | Type | Meaning | Edge cases |
|------|------|---------|------------|
| `defeater_text` | `str` | The prose of the *defeater* (the argument that the derived belief is invalid). | Injected verbatim into the prompt via `str.format`. Must be a `str`/`str`-ish; `None` would raise outside the try (see Error Handling). |
| `defeated_text` | `str` | The prose of the *derived belief* being defeated. Used as context so the model sees both sides. | Same as above. |
| `model` | `str` | Passed straight to `invoke_model`; identifies the LLM backend/model. | `None` or misspelled model will typically raise inside `invoke_model`, which is caught and returns `""`. |
| `timeout` | `int`/`float` | Forwarded to `invoke_model` as the call deadline. | `0` or negative values depend on `invoke_model`'s semantics — nothing here validates them. |

## Return Value

- **A member of `DEFEAT_REASON_TYPES`** on success. Two paths can produce this:
  1. *Exact match*: after `strip().lower()`, the whole result equals one type.
  2. *Substring match*: some type appears as a substring of the normalized result (returns the first match **in list order**).
- **`""`** on any of: LLM exception, empty output, or output containing no canonical type.
- **Caller obligation:** `""` means "unclassified / unknown," not "no defeater." Callers must decide whether to persist the empty string, skip the record, or fall back to another classifier. It is a valid sentinel, not an error signal.

## Algorithm

1. **Build the prompt** by formatting `CLASSIFY_DEFEAT_REASON_PROMPT` with the two texts. This uses the static template that lists the seven failure modes and instructs the model to answer with exactly one.
2. **Call the LLM** inside a `try`. On success, `result` is the raw model output. On any exception, print `WARN: classification failed: <e>` to `stderr` and return `""`.
3. **Normalize** with `result.strip().lower()`. The `.lower()` matters because the prompt says "exactly one type" but models frequently emit title/uppercase variants.
4. **Exact match**: if `result in DEFEAT_REASON_TYPES`, return it immediately.
5. **Substring fallback**: iterate `DEFEAT_REASON_TYPES` *in declaration order* and return the first type `t` such that `t in result`. This handles chatty outputs like `"unsupported-conjunct: the antecedent only establishes..."` or `"I'd classify this as over-generalizes."`.
6. **Give up**: return `""` if nothing matched.

Note: step 5 is order-sensitive. If the model mentions more than one type (e.g. `"not over-generalizes but scope-mismatch"`), the winner is whichever appears earlier in `DEFEAT_REASON_TYPES` — currently `"over-generalizes"` — regardless of the model's actual conclusion.

## Side Effects

- **External I/O**: one synchronous call to `invoke_model` (network / process, cost, latency — subject to `timeout`).
- **stderr logging**: on exception it emits `  WARN: classification failed: {e}` to `sys.stderr`. This is the only observability hook — there is no structured logging and no stack trace.
- No mutation of global state, no writes to `nodes`, no persistence.

## Error Handling

- **Swallowed**: all `Exception` subclasses raised *by* `invoke_model` (network errors, timeouts, auth errors, JSON errors, etc.) are caught, logged, and converted into a silent `""` return. This is deliberate: classification is treated as optional enrichment, not a hard requirement.
- **Not swallowed — a real gap**: the line `result = result.strip().lower()` sits **outside** the `try`. If `invoke_model` returns a non-string (e.g. `None`) without raising, this raises `AttributeError` which propagates out of `classify_defeat_reason`. The function therefore *assumes* `invoke_model`'s return contract is "a `str`" but does not enforce or defend against it. Callers that can't guarantee this must wrap the call themselves.
- **Not raised deliberately**: this function never raises for expected conditions (bad model output, timeout). Only a `str.format` failure on a `None` argument or the non-string return described above can escape.

## Usage Patterns

Typical call is on-demand, per-defeater, when a downstream consumer is about to build an `unsupported-conjunct`-style structured defeater and wants to pick the right category:

```python
reason = classify_defeat_reason(defeater.text, belief.text, model="claude", timeout=60)
if not reason:
    # fall back: default to "unsupported-conjunct", skip, or mark unknown
    reason = "unsupported-conjunct"
```

Caller obligations:
- Treat `""` as unknown and decide on a policy (default, retry, or skip).
- Do not rely on latency — a single call may block up to `timeout` seconds.
- Do not expect strict determinism; the same inputs can yield different types across runs. The substring fallback is what makes this tolerable for chatty models.

Because `classify_defeat_reason` is *not called anywhere in `reasons/review.py` itself*, it is exported for callers outside the review pipeline — likely in the "build structured defeater" step (see `tests/test_classify_defeaters.py`).

## Dependencies

- `invoke_model` from `reasons.llm` — the only external dependency; its return-type contract is what the untry'd `.strip()` relies on.
- `CLASSIFY_DEFEAT_REASON_PROMPT` — module-local constant, must stay in sync with `DEFEAT_REASON_TYPES` (both list the same seven slugs and the same one-line descriptions). Nothing enforces the sync.
- `DEFEAT_REASON_TYPES` — module-local list; its *order* determines substring-match precedence.
- `sys` — for `stderr` warning output.

## Assumptions the code makes but does not enforce

1. **`invoke_model` returns `str`.** A `None` return raises `AttributeError` outside the `try`. The signature/call site doesn't enforce or document this.
2. **The prompt template and the type list stay aligned.** If someone edits `DEFEAT_REASON_TYPES` but not the prompt (or vice versa), classification quality silently drifts — no test asserts the two lists contain the same slugs.
3. **Substring matching is unambiguous.** The code assumes no canonical type is a substring of another (true today), and that if the model mentions multiple, "list order" approximates "correct answer." Neither is guaranteed.
4. **`None`/non-string text arguments won't reach `.format`.** A `None` `defeater_text`/`defeated_text` raises `AttributeError` (or produces `"None"`) depending on Python version and whether the field is `{}` — this is outside the function's error boundary.
5. **The empty-string return is a safe sentinel.** It assumes no consumer treats `""` as `None`-equivalent and that no caller downstream expects a non-empty string; `parse_review_response` also defaults `defeat_reason_type` to `""`, keeping this consistent.

## Topics to Explore

- [function] `reasons/llm.py:invoke_model` — Its return-type contract (str? None?) is exactly what the untry'd `.strip()` relies on; changing it would silently break this function.
- [function] `reasons/review.py:review_beliefs` — Produces the `defeat_reason_type` field in bulk review and shows how empty values are handled downstream.
- [file] `tests/test_classify_defeaters.py` — Expected behaviors for exact matches, chatty outputs, and failure modes; the best way to confirm the substring-fallback contract.
- [general] `defeat-reason-type-taxonomy` — Whether `DEFEAT_REASON_TYPES` and `CLASSIFY_DEFEAT_REASON_PROMPT` are kept in sync anywhere (grep for both slugs in the repo).
- [general] `llm-output-normalization` — How other call sites cope with chatty/nonconforming LLM output (compare with `parse_review_response`'s `raw_decode` strategy).

## Beliefs

- `classify-defeat-reason-returns-empty-string-on-failure` — Any exception from `invoke_model`, or any output not containing a listed type, yields `""`, never `None` and never an out-of-taxonomy string.
- `classify-defeat-reason-catches-all-exceptions` — The `try` wraps only the `invoke_model` call; a non-string return then raises `AttributeError` from `.strip()` outside the `try` and propagates to the caller.
- `classify-defeat-reason-output-is-lowercased` — Both exact-match and substring-match operate on `result.strip().lower()`, so model output in any case is accepted if it contains a canonical slug.
- `substring-fallback-is-order-sensitive` — When multiple canonical types appear in the model output, the first in `DEFEAT_REASON_TYPES` list order wins, regardless of the model's actual intent.
- `classify-defeat-reason-has-no-test-verified-prompt-type-sync` — Nothing in the module asserts that `CLASSIFY_DEFEAT_REASON_PROMPT` and `DEFEAT_REASON_TYPES` list the same seven slugs, so they can drift independently.
