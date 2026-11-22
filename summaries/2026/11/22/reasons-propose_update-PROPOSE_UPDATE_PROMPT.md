---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: PROPOSE_UPDATE_PROMPT in reasons/propose_update.py

# `PROPOSE_UPDATE_PROMPT` — Explanation

`PROPOSE_UPDATE_PROMPT` is a module-level `str` constant acting as a **format template** for an LLM prompt. It is not executable logic by itself; its behavior comes entirely from how `propose_updates` uses it via `str.format(...)` and passes the result to `invoke_model`. The explanation below covers the template literal, its embedded contract, and how the rest of the file relies on it.

---

## 1. Purpose

The prompt instructs an LLM to act as a **belief auditor** for a Truth Maintenance System (TMS):

- Input: a batch of beliefs (formatted by `format_belief_for_update`) including their text, truth value, source, source URL, stale reason, justification chains (antecedents / unless-clauses / labels), and dependents.
- Task: decide per belief whether to **update**, **retract**, or **skip**.
- Output format: a strict JSON array of proposal objects with the fields `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`.

It exists so that the *taxonomy of failure modes and update bases* is specified once, in natural language, in the same place the deterministic code defines the allowed sets (`FAILURE_MODES`, `BASES`). This mirrors the parsing logic in `parse_update_proposals`, which will silently downgrade any unknown value.

The prompt intentionally biases the model toward **conservatism**: "when in doubt, don't propose a change", "skip accurate and well-supported beliefs". This is the mechanism that keeps the downstream review list short.

---

## 2. Contract

Because it is a template, its contract is about **how it must be formatted**:

- **Precondition:** Every brace that is meant to be a *literal* character in the final prompt must be **doubled** (`{{` / `}}`). There is exactly one *substitution* placeholder, named `{beliefs}`.
- **Precondition:** The caller must invoke `PROPOSE_UPDATE_PROMPT.format(beliefs=<str>)`. Passing anything other than a `str` (e.g., a dict) still "works" — `format` calls `str()` — but will produce malformed belief blocks.
- **Postcondition:** The returned string is a complete prompt ending with belief text under the heading `## Beliefs to review`. All literal JSON braces in the example block appear as single `{` / `}` after formatting.
- **Invariant (soft, manual):** The failure-mode list and basis list *inside the prompt text* must equal `FAILURE_MODES` and `BASES` respectively. Today they do; nothing in the type system enforces it. If they drift, the model may emit values that `parse_update_proposals` silently normalizes to `""` (failure_mode) or `"prior-knowledge"` (basis).
- **Invariant (soft, manual):** The example JSON object shape must include the same keys `parse_update_proposals` reads: `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`. The parser defaults missing keys, so this is not strictly enforced.

---

## 3. Parameters (of the `.format(...)` call)

The symbol itself is a value. Its only "parameters" are the `.format` arguments:

| Name       | Type   | Meaning                                                                       | Edge cases |
|------------|--------|-------------------------------------------------------------------------------|------------|
| `beliefs`  | `str`  | Concatenated `format_belief_for_update` outputs, joined with `"\n\n"` by the caller | Empty string yields a prompt with an empty "Beliefs to review" section. A belief string containing `{` or `}` is safe here *because it is substituted, not re-scanned* — but any `{}` outside `{beliefs}` in the template itself would break formatting. |

> Note: `beliefs` is the only keyword accepted. Calling `.format(beliefs=...)` with a *different* keyword (e.g., `format(belief=...)`) will raise `KeyError` because `belief` is not a placeholder.

---

## 4. Return Value

- Type: `str`.
- Content: A fully rendered prompt including the taxonomy, JSON schema example, rules, and the injected belief text.
- No `None`/sentinel behavior — a successful `.format` always returns a string. Failures surface as exceptions (see below), not sentinel values.

---

## 5. Algorithm (as used by the file)

1. `propose_updates` selects `belief_ids` — by default all `IN` beliefs sorted by node id.
2. It slices into batches of `PROPOSE_UPDATE_BATCH_SIZE` (10).
3. For each batch it builds `beliefs_text = "\n\n".join(format_belief_for_update(...))`.
4. It renders the prompt: `PROPOSE_UPDATE_PROMPT.format(beliefs=beliefs_text)`.
5. It calls `invoke_model(prompt, model=model, timeout=timeout)`.
6. The raw response passes through `parse_update_proposals`, which:
   - Scans for `[`, uses `JSONDecoder().raw_decode` to tolerate prose around the JSON.
   - Skips non-dict items and items lacking `"id"`.
   - Defaults `action` to `"update"` unless literally `"update"` / `"retract"`.
   - Defaults `failure_mode` to `""` unless in `FAILURE_MODES`.
   - Defaults `basis` to `"prior-knowledge"` unless in `BASES`.
7. Valid proposals are appended to `all_results`, and `on_batch` is invoked with the running list.

The prompt's *only* job in this algorithm is to make step 5 produce parseable JSON that honors step 6's constraints.

---

## 6. Side Effects

- **None at definition time.** The string is inert. Merely importing `reasons.propose_update` does not call an LLM.
- **Indirect side effects** arise where the prompt is consumed: `invoke_model` performs the network call, and `propose_updates` writes progress messages to `sys.stderr` ("Reviewing batch …", "WARN: …").
- **No state mutation** by the prompt itself. The current file does not cache or write the prompt.

---

## 7. Error Handling

Errors are *not* handled inside the template; they occur at the points of use:

- `str.format` can raise **`KeyError`** or **`IndexError`** if a placeholder other than `{beliefs}` exists in the template (e.g., someone adds `{source}` or forgets to double a brace in a future edit). This class of bug would fire on *every* batch.
- `str.format` can also raise **`ValueError`** for malformed replacement fields (`{bad syntax}`).
- `invoke_model` may raise (network, timeout, parsing at the provider level). `propose_updates` wraps the whole render→invoke→parse sequence in `try/except Exception` and downgrades to a warning, so a failing batch is **skipped**, not fatal.
- `parse_update_proposals` **never raises** for syntactically-valid-but-semantically-wrong output; it returns `[]` on parse failure and per-item normalization on invalid enum values. Callers cannot distinguish "model said no changes" from "response was unparseable" via this function alone.

The important consequence: the prompt is the *only* mechanism enforcing the JSON contract. If the model ignores it, `parse_update_proposals` degrades gracefully but silently.

---

## 8. Usage Patterns

- Single call site: `propose_updates`.
  ```python
  prompt = PROPOSE_UPDATE_PROMPT.format(beliefs=beliefs_text)
  response = invoke_model(prompt, model=model, timeout=timeout)
  results = parse_update_proposals(response)
  ```
- **Caller obligations:**
  - Never modify the template to add a new single-braced placeholder without updating the single `.format(beliefs=...)` call site.
  - Keep the enumerated failure modes and bases synchronized with `FAILURE_MODES` and `BASES`.
  - Any new required field in the output JSON must be reflected in *both* the template's example object *and* `parse_update_proposals`'s normalization.
- Typical end-to-end flow: `propose_updates` → proposals list → `format_proposal_markdown` / `format_proposals_file` render human review documents with suggested `reasons update` / `reasons retract` commands.

---

## 9. Dependencies

- **Internal:** the symbolic meaning of each field relies on `format_belief_for_update` (input shape) and `parse_update_proposals` (output shape). No imports are needed for the constant itself.
- **External:** none — it is a pure string literal.
- **Semantic coupling:** `FAILURE_MODES`, `BASES`, and the two constants `PROPOSE_UPDATE_BATCH_SIZE` and `PROPOSE_UPDATE_PROMPT` are read-only values consumed only by `propose_updates`.

---

## Assumptions not enforced by the type system

1. **Brace-escaping discipline:** All literal `{`/`}` are doubled. There is no test asserting this at import time; a future edit could break every call to `.format`.
2. **Single placeholder name:** Only `{beliefs}` exists in the template. No constants equal `PROPOSE_UPDATE_PROMPT` declare this; it is convention.
3. **Taxonomy parity:** `FAILURE_MODES` / `BASES` must match the prompt's bullet lists. Nothing prevents drift.
4. **`id` validity:** The prompt implies `id` refers to a real belief id. `parse_update_proposals` only checks `"id" in item`; it does not verify the id exists in `nodes`. Invalid ids can surface downstream (e.g., `format_proposal_markdown` renders "Current text: " empty).
5. **`proposed_text` semantics:** The prompt requires `null` for `retract`, but the parser does **not** enforce this — it just copies `item.get("proposed_text")`. A malformed `retract` with a string `proposed_text` will render confusingly.
6. **Evidence URLs:** The prompt requests URLs in `evidence` when `source_url` exists, but `parse_update_proposals` and the renderers never validate that evidence contains a URL.

---

## Topics to Explore

- [function] `reasons/propose_update.py:parse_update_proposals` — How tolerance for prose and enum normalization interacts with the prompt's contract; useful for understanding the failure modes of a "loose" parser.
- [function] `reasons/propose_update.py:format_belief_for_update` — The concrete shape of `{beliefs}`; explains what the LLM actually sees inside the prompt.
- [file] `reasons/llm.py` — Where `invoke_model` sends the prompt, which determines timeouts, retries, and response fidelity.
- [general] `failure-mode-taxonomy-parity` — How `FAILURE_MODES` / `BASES` stay synchronized with the prompt text; a candidate for a unit test.
- [file] `tests/test_propose_update.py` — Existing coverage of the prompt's expectations (JSON parsing, defaults, cascade rendering) and the gap around brace escaping.

---

## Beliefs

- `PROPOSE_UPDATE_PROMPT-has-placeholder-beliefs` — The template contains exactly one substitution placeholder, `{beliefs}`, and all other literal braces are escaped as `{{` / `}}`.
- `propose-updates-requires-propose-update-prompt` — `propose_updates` cannot build a batch request without calling `PROPOSE_UPDATE_PROMPT.format(beliefs=...)`.
- `propose-update-prompt-maps-to-failure-modes-and-bases` — The enumerated failure modes and bases in the prompt text correspond to the `FAILURE_MODES` and `BASES` sets in the same module.
- `parse-update-proposals-downgrades-unknown-enums` — Values outside `FAILURE_MODES`/`BASES` are silently replaced with `""`/`"prior-knowledge"` rather than raising, so prompt/parser drift degrades quietly.
- `propose-updates-swallows-batch-failures` — Exceptions from rendering, invoking the model, or parsing are caught per-batch and logged to `sys.stderr`, so one bad batch does not abort the run.
