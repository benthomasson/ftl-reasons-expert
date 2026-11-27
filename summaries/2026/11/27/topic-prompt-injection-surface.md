---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Understand how untrusted belief text flows into `TRIAGE_PROMPT` and whether downstream repair (e.g., `soften_belief`) has the same exposure.

## How untrusted belief text reaches `TRIAGE_PROMPT`

The triage prompt path is:

1. `repair_beliefs` receives `review_results` and `nodes`. For each invalid belief it builds:
   - `belief_context = format_belief_for_review(belief_id, nodes)`
   - `comment = r.get("comment", "")`

2. It calls `triage_belief(...)`. Inside `triage_belief`, near `reasons/repair.py:250`, the code does:
   ```python
   prompt = TRIAGE_PROMPT.format(
       belief_context=belief_context,
       review_comment=review_comment,
       depth=depth,
       flagged_ancestors=flagged_ancestors,
   )
   ```

3. `TRIAGE_PROMPT` is defined at `reasons/repair.py:67`. It embeds the values directly:
   ```text
   ## Invalid belief

   {belief_context}

   ## Review finding

   {review_comment}
   ```

   There is no escaping, quoting, JSON wrapping, delimiter hardening, or instruction that these values are untrusted data only. Python `str.format` inserts them as raw text. This is not a Python format-string vulnerability, because user values are not re-evaluated, but it is a **prompt-injection exposure**: belief text or review comments can contain instructions, fake sections, or fake JSON intended to influence the LLM.

4. `invoke_model` in `reasons/llm.py` sends the constructed prompt to the CLI or API without any prompt-content sanitization.

5. `parse_triage_response` scans for the first JSON object with a `pattern` in `VALID_PATTERNS`. It restricts the pattern value, but it does not validate that the input text was trustworthy, and `rationale` is arbitrary model output.

One caveat: the exact contents of `belief_context` are not visible in the observations because `format_belief_for_review` is imported from `.review` but its definition is not included. It likely includes belief text and justification/context data, but the exact formatting needs `reasons/review.py`. Within `reasons/repair.py`, however, there is no additional sanitization after `format_belief_for_review(...)` returns.

## Does downstream repair have the same exposure?

Yes. `soften_belief` has the same raw-interpolation exposure.

`soften_belief` is defined at `reasons/repair.py:260`:

```python
def soften_belief(belief_text, antecedents_text, model="claude", timeout=300):
    prompt = SOFTEN_PROMPT.format(
        belief_text=belief_text,
        antecedents=antecedents_text,
    )
```

The call site is in `repair_beliefs` at `reasons/repair.py:444`:

```python
ant_text = _format_antecedents(node, nodes)
soften_result = soften_belief(
    node.get("text", ""), ant_text,
    model=model, timeout=timeout,
)
```

Here:

- `belief_text` is raw `node["text"]`, i.e. the belief being repaired.
- `antecedents_text` comes from `_format_antecedents(node, nodes)`, which includes each antecedent’s `text` field:
  ```python
  lines.append(f"- {ant_id}: {ant.get('text', '')}")
  ```
- `SOFTEN_PROMPT` places both raw values into `## Original belief` and `## Antecedents (the evidence)` sections.

So `soften_belief` does **not** receive a sanitized version from triage. Triage only chooses the pattern. The soften path independently interpolates raw node text and raw antecedent text into another LLM prompt.

The impact can be at least as serious as the triage exposure. `parse_soften_response` extracts `softened_text`, and on a non-dry-run the result is written back to the network via `api.supersede_with_text(...)`. Prompt injection in `SOFTEN_PROMPT` can therefore influence not just a triage decision, but the persisted replacement belief text.

The search-and-link repair path has the same pattern:

- `extract_smuggled_claim` interpolates `belief_context` and `review_comment` into `EXTRACT_PROMPT`.
- `find_matching_premises` builds `candidate_lines` from search-result `text` values and interpolates them into `MATCH_PROMPT`.

So the exposure is systemic across repair prompt construction, not unique to `TRIAGE_PROMPT`.

## Topics to Explore

- [file] `reasons/review.py` — Needed to verify exactly what `format_belief_for_review` includes: belief text, antecedent text, labels, justifications, or any pre-sanitization.
- [function] `reasons/repair.py:soften_belief` — Shows the downstream `SOFTEN_PROMPT.format(...)` call and the raw belief/antecedent interpolation path.
- [function] `reasons/repair.py:find_matching_premises` — Shows the same raw interpolation pattern for candidate premise text in the search-and-link repair path.
- [function] `reasons/llm.py:invoke_model` — Confirms the transport layer sends prompts to CLI/API without inspecting or escaping prompt content.
- [file] `tests/test_research.py` — Contains a `soften_belief` test, but visible coverage does not appear to include adversarial prompt-injection cases.

## Beliefs

- `triage-prompt-interpolates-belief-context-and-review-comment-raw` — `triage_belief` passes `belief_context` and `review_comment` directly into `TRIAGE_PROMPT.format` at `reasons/repair.py:250` without escaping or delimiting.
- `soften-belief-interpolates-belief-and-antecedent-text-raw` — `soften_belief` at `reasons/repair.py:260` interpolates raw `belief_text` and `antecedents_text` into `SOFTEN_PROMPT`.
- `repair-parsers-validate-shape-not-input-trust` — `parse_triage_response` and `parse_soften_response` extract/validate JSON fields but do not sanitize source text or returned text content.
- `soften-repair-persists-llm-generated-text` — On non-dry-run soften, `repair_beliefs` calls `api.supersede_with_text` with the `softened_text` returned by the LLM.
- `search-and-link-also-interpolates-untrusted-premise-text` — `find_matching_premises` builds candidate lines from search-result `text` values and inserts them into `MATCH_PROMPT` without escaping.
