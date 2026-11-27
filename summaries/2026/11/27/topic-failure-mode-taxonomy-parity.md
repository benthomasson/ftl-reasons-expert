---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: How `FAILURE_MODES` / `BASES` stay synchronized with the prompt text; a candidate for a unit test.

## Synchronization mechanism

From the observations, `FAILURE_MODES` and `BASES` are **not automatically synchronized** with the prompt text.

- `FAILURE_MODES` is defined at `reasons/propose_update.py:15`.
- `BASES` is defined at `reasons/propose_update.py:24`.
- The only observed uses are membership checks:
  - `reasons/propose_update.py:188` — `if failure_mode not in FAILURE_MODES:`
  - `reasons/propose_update.py:191` — `if basis not in BASES:`

The grep results show only two references to `FAILURE_MODES` and two references to `BASES` — the definitions and the validators above. That means the prompt is **not generated from these constants** via something like `", ".join(FAILURE_MODES)` or `FAILURE_MODES.keys()`. Instead, the prompt text is a separate static string, and the constants act as the validation allow-list.

So the synchronization is **manual/convention-based**:

1. The prompt tells the LLM which failure modes and bases are allowed.
2. The validator checks the returned proposal against `FAILURE_MODES` / `BASES`.
3. If someone adds an option to the prompt but not the constants, the LLM may emit it but validation rejects it.
4. If someone adds an option to the constants but not the prompt, validation accepts something the LLM was never told about.

The exact prompt text and prompt variable name are **not present in the observations**, so the precise formatting is unknown. The observations are sufficient to conclude there is no dynamic coupling, but not sufficient to write the exact final assertion without seeing `reasons/propose_update.py` in full.

## Candidate unit test

A good unit test should verify that the prompt’s option list and the validation constants contain the same values.

A simple version, assuming the prompt is exposed as a constant:

```python
from reasons.propose_update import FAILURE_MODES, BASES, PROPOSE_UPDATE_PROMPT

def test_prompt_lists_all_failure_modes_and_bases():
    prompt = PROPOSE_UPDATE_PROMPT

    for failure_mode in FAILURE_MODES:
        assert failure_mode in prompt, f"prompt missing failure mode: {failure_mode}"

    for basis in BASES:
        assert basis in prompt, f"prompt missing basis: {basis}"
```

That is a useful smoke test, but it is weak: a key could appear elsewhere in the prompt and falsely pass. A stronger test should parse the relevant prompt sections and compare sets:

```python
def test_prompt_options_match_validation_sets():
    prompt = PROPOSE_UPDATE_PROMPT

    prompt_failure_modes = parse_option_list(prompt, section="Failure modes")
    prompt_bases = parse_option_list(prompt, section="Bases")

    assert prompt_failure_modes == set(FAILURE_MODES)
    assert prompt_bases == set(BASES)
```

The exact `parse_option_list` helper depends on the prompt format, which is not shown. If the prompt uses human-readable descriptions instead of raw keys, the test should compare against the descriptions rather than the keys.

It would also be worth adding a validator-side test near this one:

```python
def test_unknown_failure_mode_is_rejected():
    proposal = make_proposal(failure_mode="not-a-real-mode")
    with pytest.raises(...):
        validate_proposal(proposal)
```

But the key synchronization test is the prompt-vs-constants comparison.

## Topics to Explore

- [file] `reasons/propose_update.py` — Inspect the prompt template, `FAILURE_MODES`, `BASES`, and the validation logic around lines 15–24 and 188–191.
- [general] `propose-update-prompt-format` — Determine whether the prompt lists raw keys or human-readable descriptions, since that determines how the synchronization test should parse it.
- [file] `tests/test_derive.py` — Use it as an example of prompt-building, parsing, and validation tests to mirror for `propose_update`.
- [file] `reasons/api.py` — Compare with `derive_prompt` at `reasons/api.py:2483`, which builds a prompt through `build_prompt`.
- [general] `prompt-enum-synchronization` — Consider generating the prompt option list from `FAILURE_MODES` / `BASES` to eliminate manual drift.

## Beliefs

- `failure-modes-and-bases-are-validation-sets` — `FAILURE_MODES` and `BASES` are used in membership checks to reject unknown proposal metadata at `reasons/propose_update.py:188` and `reasons/propose_update.py:191`.
- `failure-modes-is-not-used-by-prompt-construction` — Grep results show `FAILURE_MODES` only at its definition and validator usage, so the prompt is not dynamically generated from it.
- `bases-is-not-used-by-prompt-construction` — Grep results show `BASES` only at its definition and validator usage, so the prompt is not dynamically generated from it.
- `prompt-constant-sync-is-manual` — The prompt text and the validation constants must be updated together by hand because there is no observed interpolation between them.
