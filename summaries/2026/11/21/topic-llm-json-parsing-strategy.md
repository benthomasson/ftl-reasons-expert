---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Compare how other modules in `reasons/` extract and validate JSON from model responses.

## How `reasons/` modules extract & validate JSON from model responses

Short version: **there is no shared JSON-extraction helper.** Each module that needs to read model output implements its own strategy, and the strategies differ substantially — one parses the CLI *envelope*, one does character-scanning `raw_decode`, and one doesn't use JSON at all.

### 1. `reasons/llm.py` — `_parse_cli_json` (around line 118–148)

This is often mistaken for "model JSON parsing," but it is not. It parses the **CLI's own structured envelope** — the `--output-format json` wrapper emitted by `claude` and `gemini` CLIs — to pull out the `result`/`response` text plus token/cost usage.

- Uses plain `json.loads(output)` with a `try/except (JSONDecodeError, ValueError)` that **falls back to returning the raw string** on failure.
- Guards only on `isinstance(data, dict)`; no schema validation.
- Dispatches on model prefix (`gemini*` reads `response` + `stats.models.*.tokens`; else reads `result`, `usage`, `total_cost_usd`).
- This runs *before* the model's semantic output is touched — the extracted `result`/`response` string is what downstream modules must parse.

So: this function is defensive against non-JSON CLI output, but it says nothing about whether the *model's* response is well-formed JSON.

### 2. `reasons/propose_update.py` — `parse_update_proposals` (around line 149–190)

This is the only module in the observed set that actually extracts a JSON value produced *by the model*.

- Creates `decoder = json.JSONDecoder()` and iterates character-by-character, trying `decoder.raw_decode(response, i)` at **every `[` position**. This tolerates prose/markdown before and after the array.
- Skips any decoded value that isn't a `list`.
- For each item: **requires `isinstance(item, dict)` and the presence of `"id"`** — otherwise the item is silently dropped.
- Validates and normalizes taxonomy fields:
  - `action` must be `"update"` or `"retract"`, else coerced to `"update"`.
  - `failure_mode` must be in `FAILURE_MODES` (smuggled-premise, unsupported-superlative, false-causal-link, domain-conflation, contradicted-by-source, stale), else `""`.
  - `basis` must be in `BASES` (source-divergence, detected-contradiction, prior-knowledge), else `"prior-knowledge"`.
- Returns the **first** candidate list that yields at least one valid item; otherwise returns `[]`.

Note the asymmetry: `failure_mode` is *demoted to empty* on invalid value, whereas `basis` is *defaulted to a valid value*. This matters because the prompt says the taxonomies are strict and reviewers rely on them.

### 3. `reasons/derive.py` — `parse_proposals` (around line 460–500+)

This module doesn't parse JSON from model responses **at all**. The derive prompt (`DERIVE_PROMPT`) asks for a markdown-ish line format (`### DERIVE <id>`, `- Antecedents:`, `- Mode:`, `- Label:`), and the parser uses `re.compile` with `finditer` instead of `json`.

- Two regex passes: a "new" format (`### DERIVE <id>`) and a legacy "old" format (`### DERIVE: \`<id>\`` / `### GATE (outlist): \`<id>\``).
- If the new regex finds any matches, it returns immediately; the old regex only runs when the new one finds nothing.
- Validation is structural (regex shape), not semantic — no whitelist of allowed IDs or antecedents is enforced here. `unless` entries are split/cleaned but not validated against the node dict.
- `mode` defaults to `"all"` if the optional `Mode:` group is absent.

This is the clearest example of a different strategy: **format-constrained prompt + regex**, not "ask for JSON."

### Comparison at a glance

| Module | Parses | Mechanism | Failure mode | Validation |
|---|---|---|---|---|
| `llm.py` | CLI *envelope* JSON | `json.loads` on whole stdout | returns raw output | `isinstance(dict)` only |
| `propose_update.py` | model's JSON array | `JSONDecoder.raw_decode` at every `[` | returns `[]` | dict + `"id"` required; taxonomy coerced/defaulted |
| `derive.py` | model's markdown | regex `finditer`, two formats | returns `[]` | regex structural only |
| Shared helper | — | — | — | none exists (grep for `extract_json`/`parse_json` returns 0) |

### Gaps in the observations

- **The `json.loads` grep reportedly returns 0 matches across `reasons/`, yet `llm.py:_parse_cli_json` clearly calls `json.loads`.** Either the grep pattern scope excluded `llm.py`, or the grep ran before the file was in its current state. Worth resolving before citing "no module uses `json.loads`."
- Only three `reasons/` modules are shown. Other modules (e.g. `extract.py`, `review.py`, `contradictions.py`, `cluster.py` if it invokes a model) were not read; any additional `json.loads` / `raw_decode` call sites in those files are unknown.
- No evidence of retries, schema libraries (pydantic/jsonschema), or code-fence stripping (`\`\`\`json ... \`\`\``). The `grep_model_response_parsing_patterns` result is empty, but the pattern scope is unclear.
- No evidence about how `propose_update.parse_update_proposals` handles a *valid* JSON array that contains zero valid items — it falls through to the next `[` rather than returning, so trailing brackets in prose could still be scanned.

## Topics to Explore

- [file] `reasons/extract.py` — Not included in observations; likely the other major model-response consumer and would complete the JSON-parsing comparison
- [function] `reasons/propose_update.py:parse_update_proposals` — Verify the first-matching-array semantics and the action/failure_mode/basis coercion rules actually guard against reviewers accepting bad values
- [function] `reasons/derive.py:parse_proposals` — Confirm whether cross-format (new vs. old) detection can misclassify a single response mixing both formats
- [general] `json-loads-grep-discrepancy` — Reconcile the empty `json.loads` grep result with `llm.py:_parse_cli_json`; the observation set may be stale or scoped
- [general] `model-response-validation-contract` — Whether there is any test coverage asserting that malformed model output is rejected/coerced consistently across modules

## Beliefs

- `no-shared-json-extractor-exists` — Grep for `extract_json`/`parse_json` across `reasons/` returns zero definitions; each consumer implements its own parser.
- `parse-update-proposals-requires-id` — `propose_update.py:parse_update_proposals` drops any decoded array item that is not a dict containing `"id"`.
- `parse-update-proposals-validates-taxonomy` — `parse_update_proposals` coerces `action` to `update` and defaults `basis` to `prior-knowledge`, but sets `failure_mode` to `""` on invalid input.
- `parse-update-proposals-uses-raw-decode` — `parse_update_proposals` scans for `[` and calls `JSONDecoder.raw_decode`, tolerating prose around the JSON array; it never code-fence-strips.
- `derive-parse-proposals-is-regex-not-json` — `derive.py:parse_proposals` extracts proposals via `re.finditer` on a markdown format and does not invoke `json`.
- `llm-parse-cli-json-parses-envelope-not-response` — `llm.py:_parse_cli_json` unwraps the CLI `--output-format json` envelope (`result`/`response` + usage) and falls back to raw stdout on `JSONDecodeError`; it does not validate the model's semantic JSON.
