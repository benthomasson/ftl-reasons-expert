---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Explore assumptions around JSON-line formatting, prompt compliance, and schema validation gaps.

# JSON-Line Formatting, Prompt Compliance, and Schema Validation Gaps

## Executive Summary

The code assumes **single-object JSON output per LLM call** (`--output-format json`, not JSONL), assumes **models will mostly comply with prompt output rules** (but parsers gracefully degrade), and **does not enforce the taxonomies or field constraints the prompts define**. Multiple silent-failure paths exist where malformed or non-compliant LLM output is coerced into "valid-looking" results rather than flagged. Note also that two of the observations (`search_json_line_parsing_patterns` and `search_prompt_compliance_and_schema_validation_mentions`) returned `total_count: 0` — but `json.loads` appears in `reasons/llm.py:_parse_cli_json` and `reasons/verify.py:parse_verify_response`, and "prompt"/"format" appear throughout — so those search results are **unreliable** and should be re-run before drawing conclusions from them.

---

## 1. JSON-Line Formatting Assumptions

### 1a. The pipeline assumes a *single JSON object*, not JSONL

- `MODEL_COMMANDS` in `reasons/llm.py` invokes claude with `--output-format json` and gemini with `-o json`. The docstring at the top of `llm.py` explicitly says *"CLI models use --output-format json to capture token counts and costs."*
- `_parse_cli_json` calls `json.loads(output)` on the **entire stdout**. There is no `splitlines()`, no per-line parse, no accumulation. If a CLI ever emits JSONL (multiple JSON objects, one per line — e.g., streaming or multi-turn format), `json.loads` raises and the function returns the raw output as-is:
  ```python
  try:
      data = json.loads(output)
  except (json.JSONDecodeError, ValueError):
      return output
  ```
  Downstream callers (`parse_update_proposals`, `parse_verify_response`) will then receive raw JSONL as if it were LLM prose. `parse_update_proposals` will probably recover a single array via `raw_decode`, but token/cost recording is lost and `parse_verify_response`'s greedy `re.search(r"\{.*\}")` may silently misparse.
- **No test coverage for JSONL.** `tests/test_llm.py` (`TestParseCliJson`) only exercises single-object inputs (`test_claude_json`, `test_gemini_json`, etc.).

### 1b. Silent fallback masks CLI schema drift

In `_parse_cli_json`:

```python
text = data.get("result", output)         # claude
text = data.get("response", output)       # gemini
```

If the CLI schema changes (or an error envelope is returned as valid JSON, e.g. `{"error": "rate limited"}`), `text` becomes the **entire JSON blob** — which is then passed to the next LLM stage as if it were a response. Combined with `_record_cost(model, input_tokens, output_tokens, cost_usd)`, the call is recorded as successful (tokens default to `0`, cost defaults to `0.0`) even though the "response" is garbage. There is no check for `data.get("is_error")`, `data.get("subtype")`, or the presence of the expected key.

### 1c. Ollama output has no JSON contract at all

`invoke_model` special-cases ollama: thinking markers are stripped with `output.split("...done thinking.\n", 1)` and the raw string is returned — the code comment even flags this: `# Fragile: ollama thinking markers may change across versions`. There is no schema validation, no structured cost extraction (ollama never calls `_record_cost`), and no test for a version where the marker prefix changes to e.g. `"...done reasoning.\n"`.

### 1d. Malformed/absent JSON is silently accepted as "raw prose"

The only signal that JSON parsing failed is the return of `output`. `invoke_model` cannot distinguish *"the model answered in prose"* from *"the CLI emitted a JSON envelope we failed to parse."* Callers that expect a JSON array or object will simply produce empty results (`parse_update_proposals` → `[]`, `parse_verify_response` → `{}`) — indistinguishable from legitimate "no changes needed" or "nothing to verify."

---

## 2. Prompt Compliance Assumptions

### 2a. Prompt explicitly demands strict format, parser explicitly tolerates violations

- `PROPOSE_UPDATE_PROMPT` says *"Return ONLY a JSON array"* but `parse_update_proposals` is documented as using `json.JSONDecoder.raw_decode` *"at each '[' position to handle prose around the JSON array."* Compliance is expected to be imperfect.
- `VERIFY_PROMPT` says *"Return ONLY a JSON object"* but `parse_verify_response` uses `re.search(r"\{.*\}", response, re.DOTALL)` — a **greedy** regex that will over-match if the response contains more than one object or trailing prose containing a `}`.

### 2b. Two prompts make promise-style rules that are never checked

`PROPOSE_UPDATE_PROMPT` states, verbatim:

- *"For 'update' actions, proposed_text must be meaningfully different from current text."* → `parse_update_proposals` accepts any `proposed_text` value.
- *"For 'retract' actions, set proposed_text to null."* → a retract with a non-null `proposed_text` is accepted (and `format_proposal_markdown` simply ignores it, but the caller has no signal that the model was incoherent).
- *"Use the failure_mode and basis taxonomies exactly as defined."* → invalid `failure_mode` is silently coerced to `""`; invalid `basis` is coerced to `"prior-knowledge"`.
  - The `basis = "prior-knowledge"` default is especially concerning: it maps **an unrecognized value** (could be a typo, could be a hallucinated category) to a category the prompt itself flags as *"needs scrutiny — reviewer may share your blind spot."*
- *"Include source URLs in evidence when the belief has a source_url."* → never verified.

### 2c. Belief-ID hallucination is not filtered

`propose_updates` builds a `batch` list and prompts only with those IDs, but it never intersects `all_results` with `batch`:

```python
results = parse_update_proposals(response)
all_results.extend(results)
```

`parse_update_proposals` only checks that each item is a dict with an `"id"` key — no check that the ID exists in `nodes`, no check that it's in the current batch. `format_proposal_markdown` then looks up `nodes.get(nid, {})` → empty `current_text`, and emits `reasons update <hallucinated-id> "..."` commands. A reviewer running the suggested command hits a runtime error.

### 2d. Verdict enum is not enforced

`VERIFY_PROMPT` enumerates `CONFIRMED | STALE | PARTIAL | INCONCLUSIVE`. `parse_verify_response`:

```python
raw_verdict = v["verdict"]
...
"verdict": raw_verdict.upper(),
```

Any string is passed through. A model returning `"MAYBE"`, `"YES"`, or `"Confirmed (mostly)"` yields a `verdict` value downstream code cannot rely on. Unlike `failure_mode`/`basis`/`action` in `parse_update_proposals`, **there is no whitelist at all** for verdict.

### 2e. Verbatim-quote rule is unverified

`VERIFY_PROMPT` says *"The quote must be copied verbatim from the source document."* — nothing checks `quote` against source content. A fabricated quote passes through. This is the single biggest *substantive* compliance gap for verification: the verifier is presented as environment-checked but is really trusting the model on this dimension.

### 2f. Prompt-fenced source code and belief text are unescaped

`format_belief_for_verify` embeds source content inside ` ``` ` fences. If the source itself contains triple backticks (common in markdown docs), the fence breaks, and any subsequent text is interpreted as prose by the model. Similarly, markdown fence boundaries inside belief text can truncate the visible belief in `format_belief_for_update`. No test covers this.

### 2g. Batch-level silent failure

`propose_updates`:

```python
except Exception as e:
    print(f"  WARN: batch {batch_num} failed: {e}", file=sys.stderr)
```

A failed batch is logged to stderr and then forgotten. Partial `all_results` are returned as if the review was complete — the caller cannot distinguish *"no proposals for batch N"* from *"batch N crashed."* No summary/errors structure is returned.

---

## 3. Schema Validation Gaps by Function

### `_parse_cli_json` (`reasons/llm.py`)

| Gap | Effect |
|---|---|
| No `result`/`response` key presence check | Error envelope silently becomes "text" |
| No `usage` shape check | Token totals default to `0` |
| No `total_cost_usd` presence check | Cost silently `$0.00` |
| No JSONL support | Multi-line output degenerates to raw prose |
| No schema-version field | Breaking CLI changes cannot be detected |

### `parse_update_proposals` (`reasons/propose_update.py`)

| Gap | Effect |
|---|---|
| `id` not validated against `nodes` or batch | Hallucinated IDs reach `format_proposal_markdown` and shell commands |
| `proposed_text` type not validated (`item.get("proposed_text")`) | Could be list/number/None for an `update` |
| `update` + `proposed_text=None` accepted | Broken markdown section |
| `retract` + non-null `proposed_text` accepted | Incoherent proposals preserved |
| Duplicate IDs not deduplicated | Same belief appears twice |
| Returns at **first** non-empty result set | If the model emits two arrays, only the first survives |
| Missing `failure_mode` → `""` (not `None`) | Displays as blank/unknown downstream |

### `parse_verify_response` (`reasons/verify.py`)

| Gap | Effect |
|---|---|
| Greedy `re.search(r"\{.*\}")` | Over-matches multiple-object responses → parse error → empty dict |
| Verdict not validated against the 4-value enum | Downstream code sees arbitrary strings |
| Belief IDs not validated against input | Downstream dict may have missing or extra keys |
| `quote` not verified verbatim | Fabricated quotes accepted |
| Missing-ID handling | `parse_verify_response` returns only keys the LLM emitted; `verify_beliefs` doesn't backfill missing beliefs, so *"unverified"* is indistinguishable from *"absent"* |

### `format_proposal_markdown` (`reasons/propose_update.py`)

- Suggested command: `reasons update {nid} \"{escaped}\"` — only `"` is escaped. Newlines, backticks, `$`, `\`, `;`, `|` in `proposed_text` (which can come from LLM output) will break the inline code block or produce unsafe shell commands.
- Same for `reasons retract {nid} --reason \"{fm}: {comment}\"` — neither `fm` nor `comment` is escaped.

### `propose_updates` / `verify_beliefs` return contracts

- `propose_updates` returns `[]` both for *"no changes needed"* and *"every batch raised"*.
- `verify_beliefs` returns a partial dict — beliefs the LLM omitted silently vanish. There is no `errors` field, no `unverified` set.

### Test-suite blind spots

- `tests/test_llm.py` ends at line 400 of 470 (70 lines unobserved) — JSONL/streaming tests may or may not exist; from what is visible, only single-object JSON and no `result`-missing case are covered.
- `tests/test_propose_update.py` has **no** tests for: hallucinated ID, `proposed_text` non-string, retract-with-text, duplicate IDs, shell metacharacters in `proposed_text`, or filtering against the input batch.
- No test asserts the invariant `"returned_ids ⊆ requested_ids"`.
- No test exercises `parse_verify_response` at all in the observations shown.

---

## 4. Why This Matters Together

The chain **`invoke_model → _parse_cli_json → parse_update_proposals/parse_verify_response → format_proposal_markdown`** is treated as a structured pipeline, but each stage is *lenient* in a way that compounds:

1. CLI schema drift → raw JSON silently becomes "text".
2. LLM prose tolerance → partial arrays silently become "all proposals".
3. Taxonomy coercion → invalid values silently become *specific valid-looking categories* (`prior-knowledge`, `""`).
4. Formatting step → hallucinated IDs and unescaped strings flow into copy-paste shell commands.

The net effect is that a reviewer of `Proposed Updates` markdown cannot distinguish authoritative findings from degraded or hallucinated ones, and verification quotes are trusted on the model's honor — despite the prompt's verbatim-quote requirement.

---

## Topics to Explore

- [function] `reasons/llm.py:_parse_cli_json` — Trace how a missing `result`/`response` key becomes raw-JSON-as-text; consider adding an `is_error`/schema-version check and JSONL fallback.
- [function] `reasons/propose_update.py:parse_update_proposals` — Investigate ID filtering (returned vs. batch), duplicate-ID handling, and the early-return-on-first-array behaviour; a signature change to also return `dropped_ids` would surface hallucinations.
- [function] `reasons/verify.py:parse_verify_response` — Check verdict-enum enforcement and replace the greedy `\{.*\}` regex with `JSONDecoder.raw_decode` iteration (like `parse_update_proposals`) to tolerate trailing prose safely.
- [file] `tests/test_propose_update.py` — Add tests for hallucinated IDs, retract-with-text, non-string `proposed_text`, duplicate IDs, and markdown/shell metacharacters in `proposed_text`.
- [general] `search-tool-reliability` — The two "search" observations returned zero matches for patterns that clearly occur in the code (`json.loads`, `prompt`, `format`); re-run those searches to confirm whether there is additional JSONL or schema-validation code outside the observed files before drawing conclusions.

---

## Beliefs

- `cli-json-is-single-object` — `_parse_cli_json` and `MODEL_COMMANDS` assume CLI models emit one JSON object per call; there is no JSONL/multi-record handling and no test for it.
- `missing-cli-fields-fall-through-as-text` — When the `result` (Claude) or `response` (Gemini) key is absent, `_parse_cli_json` returns the entire JSON envelope as "text", and cost is recorded with zeroed tokens.
- `prompt-taxonomy-not-enforced` — `parse_update_proposals` silently coerces unknown `failure_mode` to `""`, unknown `basis` to `"prior-knowledge"`, and unknown `action` to `"update"` instead of rejecting non-compliant output.
- `verify-verdict-not-validated` — `parse_verify_response` uppercases but never whitelists the verdict string, so values outside `CONFIRMED|STALE|PARTIAL|INCONCLUSIVE` pass through.
- `hallucinated-belief-ids-not-filtered` — Neither `parse_update_proposals` nor `propose_updates` intersects returned IDs with the input batch, so hallucinated IDs reach `format_proposal_markdown` and its suggested shell commands.
- `proposed-text-not-escaped-for-shell` — `format_proposal_markdown` escapes only `"` in `proposed_text`, `comment`, and `failure_mode`, leaving backticks, newlines, and `$`/`|`/`;` unescaped in generated `reasons update|retract` commands.
- `batch-failures-are-silent` — `propose_updates` prints `WARN: batch N failed` to stderr and continues, so a returned `[]` is indistinguishable from "all batches errored".
