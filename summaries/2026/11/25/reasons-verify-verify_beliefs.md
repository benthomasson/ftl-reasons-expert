---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: verify_beliefs in reasons/verify.py

# `verify_beliefs`

## Purpose

`verify_beliefs` orchestrates LLM-based verification of a batch of beliefs against already-loaded source text. It does **not** resolve or read source files itself. Instead, it:

1. Formats each `(belief, source_content)` pair into a prompt section.
2. Concatenates all sections into one large verification prompt using `VERIFY_PROMPT`.
3. Makes a single call to the LLM via `invoke_model`.
4. Parses the LLM's JSON response into a mapping of `belief_id -> {verdict, reason, quote}`.

It exists as the high-level “verify these beliefs” entry point, separating source loading (`read_source`) and response parsing (`parse_verify_response`) from the actual LLM invocation.

---

## Contract

### Preconditions

- `beliefs_with_sources` is an iterable of 2-tuples: `(belief_dict, source_content_or_None)`.
- Each `belief_dict` must contain at least:
  - `"id"` — used in the prompt heading and expected as the result key.
  - `"text"` — the claim being verified.
- Each `belief_dict` may contain:
  - `"source"` — source file path/name, included in prompt if present.
  - `"source_url"` — source URL, included in prompt if present.
- `source_content_or_None` should be a `str` or `None`. `None` or empty string is rendered as `"(source not available)"`.
- `model` should be a model identifier accepted by `invoke_model`.
- `timeout` should be a value accepted by `invoke_model`, nominally seconds.
- `invoke_model` must be importable from `.llm` at call time.

### Postconditions

- Returns a dictionary mapping belief IDs to verdict dictionaries.
- The result is **sparse**: it is not guaranteed to contain every input belief ID. Missing IDs must be handled by the caller.
- Each included verdict dictionary has:
  - `"verdict"`: an uppercased string, e.g. `"CONFIRMED"`, `"STALE"`, etc.
  - `"reason"`: string, defaulting to `""` if absent in the LLM response.
  - `"quote"`: whatever the LLM supplied, typically a string or `None`.
- The function does not enforce that the verdict is one of `CONFIRMED`, `STALE`, `PARTIAL`, or `INCONCLUSIVE`; any string verdict is accepted and uppercased.
- The function does not verify that quotes are verbatim from the source.
- Inputs are not mutated.

### Important Assumptions Not Enforced by Types

- Every belief dict has `"id"` and `"text"`; otherwise `format_belief_for_verify` raises `KeyError`.
- `source_content` is a string or `None`. A truthy non-string value can cause `TypeError` during string joining.
- Belief IDs are unique. Duplicate IDs will be ambiguous in the prompt and can collapse into one result key.
- The total prompt fits the model's context window. `read_source` truncates individual sources to `MAX_SOURCE_CHARS`, but `verify_beliefs` itself has no total-size limit.
- The LLM returns a JSON object shaped as expected. The parser handles some malformed responses but not all possible non-string or unexpected JSON shapes.

---

## Parameters

### `beliefs_with_sources`

Expected: list or iterable of `(belief_dict, source_content_or_None)`.

Behavior and edge cases:

- Empty input still produces a prompt and still calls `invoke_model` with an empty beliefs section. The function does not short-circuit.
- If the iterable is a generator, it is consumed once during prompt construction.
- Missing `"id"` or `"text"` raises `KeyError`.
- `source_content` of `None` or `""` becomes `"(source not available)"` in the prompt.
- A truthy non-string `source_content` is not safely handled by `"\n".join(...)`.
- Source content is inserted inside a Markdown code fence without escaping. If the source contains triple backticks, it can disturb the prompt formatting.

### `model`

Default: `"claude"`.

Passed directly to `invoke_model`. `verify_beliefs` performs no validation or normalization of the model name.

### `timeout`

Default: `120`.

Passed directly to `invoke_model`, nominally in seconds. The function itself does not implement timeout behavior.

---

## Return Value

Returns the result of `parse_verify_response(response)`.

In normal operation, this is:

```python
{
    "belief-id": {
        "verdict": "CONFIRMED",
        "reason": "The source explicitly states this figure",
        "quote": "exact quote from the source document",
    },
    ...
}
```

Important details:

- If `invoke_model` returns a falsy value such as `None` or `""`, `parse_verify_response` returns `{}`.
- If the response contains no `{...}` JSON-looking substring, returns `{}`.
- If `json.loads` raises `json.JSONDecodeError` or `TypeError`, returns `{}`.
- Entries are skipped unless:
  - the value is a dict,
  - it contains `"verdict"`,
  - and `"verdict"` is a string.
- Verdicts are uppercased but not stripped or validated.
- Extra keys in the LLM result are ignored.
- The caller must handle missing belief IDs and possibly unexpected verdict strings.

Potential parser exceptions not converted to `{}`:

- A truthy non-string `response` can cause `re.search` to raise `TypeError`.
- If the matched JSON decodes to a non-dict, the later `data.items()` can raise `AttributeError`.

---

## Algorithm

Step by step:

1. Lazily import `invoke_model` from `.llm`.

   ```python
   from .llm import invoke_model
   ```

   This import happens inside the function, likely to avoid import cycles or optional dependency loading.

2. For each `(belief, source_content)` pair, call `format_belief_for_verify(b, src)`.

   Each formatted block includes:
   - a heading with the belief ID,
   - the claim text,
   - optional source file and URL,
   - the source document content inside a code fence,
   - or `"(source not available)"` if source content is missing.

3. Join all formatted blocks with:

   ```python
   "\n\n---\n\n"
   ```

4. Insert the combined text into `VERIFY_PROMPT`:

   ```python
   prompt = VERIFY_PROMPT.format(beliefs=beliefs_text)
   ```

5. Call the LLM:

   ```python
   response = invoke_model(prompt, model=model, timeout=timeout)
   ```

6. Parse and return:

   ```python
   return parse_verify_response(response)
   ```

---

## Side Effects

- Performs an external LLM call through `invoke_model`, which can incur network I/O, latency, cost, logging, and provider-specific side effects.
- Lazily imports `.llm`, which can load that module at call time.
- Does not read files directly.
- Does not mutate the input list, tuples, or belief dictionaries.
- Does not persist results or update the database.

---

## Error Handling

`verify_beliefs` itself does little error handling.

Errors that can propagate to the caller:

- `KeyError` if a belief dict lacks `"id"` or `"text"`.
- `TypeError` if prompt construction encounters unexpected types, such as a truthy non-string source content.
- Any exception raised by `invoke_model`, including network failures, provider errors, and timeouts.
- Possible parser exceptions for unexpected response shapes, such as non-string responses or non-object JSON.

Errors handled internally:

- `parse_verify_response` returns `{}` for:
  - falsy LLM responses,
  - responses with no `{...}` substring,
  - invalid JSON from `json.loads`,
  - `TypeError` from `json.loads`.
- Invalid individual entries in an otherwise valid JSON object are skipped silently.

The caller should generally treat `verify_beliefs` as fallible and handle both exceptions and sparse/empty result dictionaries.

---

## Usage Patterns

Typical usage in this codebase would look like:

1. Load each belief and its source text, often using `read_source`.
2. Pass the pairs into `verify_beliefs`.
3. Handle missing belief IDs and malformed/unknown verdicts.
4. Optionally batch large belief sets externally because `verify_beliefs` sends everything in one prompt.

Example caller obligations:

- Ensure belief IDs are unique.
- Ensure source content is already read and reasonably truncated.
- Skip calling the function for empty input if an unnecessary LLM call is undesirable.
- Treat the returned map as partial.
- Handle exceptions from the LLM layer.

---

## Dependencies

- `reasons.llm.invoke_model` — performs the actual LLM call.
- `reasons.verify.format_belief_for_verify` — formats each belief/source pair.
- `reasons.verify.VERIFY_PROMPT` — prompt template.
- `reasons.verify.parse_verify_response` — parses and partially validates the LLM response.
- `reasons.check_stale.resolve_source_path` — used indirectly by `read_source`, not directly by `verify_beliefs`.
- Standard library `json` and `re` are used by the parser, not directly by `verify_beliefs`.

---

## Topics to Explore

- [function] `reasons/verify.py:read_source` — Resolves and truncates source files before their content is passed into `verify_beliefs`.
- [function] `reasons/llm.py:invoke_model` — Handles the actual model call, timeout behavior, provider errors, and response shape.
- [function] `reasons/verify.py:parse_verify_response` — Determines how malformed JSON, missing verdicts, and unknown verdict strings are handled.
- [file] `tests/test_verify.py` — Shows expected behavior, monkeypatch points, and edge cases covered for verification and parsing.
- [general] `prompt-size-batching` — Callers must batch beliefs externally because `verify_beliefs` builds one unbounded combined prompt.

---

## Beliefs

- `verify-beliefs-performs-single-llm-call` — `verify_beliefs` combines all beliefs into one prompt and calls `invoke_model` once per invocation.
- `verify-beliefs-requires-id-and-text` — Each belief dict must contain `"id"` and `"text"` because `format_belief_for_verify` indexes those keys directly.
- `verify-beliefs-returns-sparse-verdict-map` — The returned dict only includes belief IDs that the LLM produced as valid JSON entries with a string `"verdict"`; missing IDs are absent.
- `verify-beliefs-does-not-read-source-files` — Source resolution and file reading are delegated to `read_source`; `verify_beliefs` only receives source content.
- `verify-beliefs-does-not-catch-invoke-errors` — Exceptions from `invoke_model` propagate to the caller rather than being converted into an empty result.
