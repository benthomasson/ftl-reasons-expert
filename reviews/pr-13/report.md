# Code Review Report

**Branch:** benthomasson/ftl-reasons#13
**Models:** claude, gemini
**Gate:** [PASS] PASS

## claude [PASS]

### reasons_lib/derive.py:parse_proposals
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

LLMs frequently emit backtick-wrapped identifiers even when not instructed to. The fix correctly applies `.strip("`")` after `.strip()` on the ID, antecedent list items, and unless list items. The old format parser (line 334-335) already had identical `.strip("`")` calls, so this brings the new format parser to parity. The regex `(\S+)` correctly matches backtick-wrapped tokens since backticks are non-whitespace. The `label` field is intentionally not stripped of backticks, which is reasonable since labels are freeform text, not node IDs used for lookups.

### tests/test_derive.py:test_parse_proposals_strips_backticks
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The test exercises the exact scenario — backticks on the ID, on each antecedent, and on the unless entry. Assertions verify all three are stripped. The test input is realistic (matches what LLMs actually produce). Coverage is thorough for the new format path. One minor observation: the test doesn't verify the GATE variant with backticks, but the code change is identical for both DERIVE and GATE since they share the same parsing block.

### Self-Review
**Limitations:** Did not verify whether any downstream consumers of `parse_proposals` (e.g., `validate_proposals`, `apply_proposals`, or LangGraph integration) would break if backticks somehow survived past this point — but since the fix only strips characters that shouldn't be in IDs, this is low risk. Did not run the test suite to confirm the new test passes.

## gemini [PASS]

### reasons_lib/derive.py:parse_proposals
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The modification correctly and safely strips backticks from the `id`, `antecedents`, and `unless` fields after stripping whitespaces. The conditional expression `[u.strip().strip("`") for u in match.group(5).split(",")] if match.group(5) else []` correctly evaluates the condition before the list comprehension to avoid AttributeError on `None`.

### tests/test_derive.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The test effectively checks the backtick-stripping functionality for all three modified fields (id, antecedents, unless) and validates that the extracted lists are properly cleaned.

### Self-Review
**Limitations:** None. The diff is small, self-contained, and accompanied by a corresponding test that validates the introduced behavior.
