---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_research.py

# `tests/test_research.py`

## Purpose

This is a pytest test module for the “research” / “repair” workflow: triaging invalid beliefs, finding and linking evidence, softening overstated claims, abandoning broken claims, and escalating claims that need further research.

Despite the filename, the canonical implementation lives in `reasons.repair`; the tests explicitly verify legacy compatibility aliases:

- `reasons.repair.research_beliefs is reasons.repair.repair_beliefs`
- `reasons.api.research is reasons.api.repair`

The file owns regression coverage for:

- LLM response parsing
- justification-depth calculation
- LLM prompt construction for triage/softening
- orchestration in `repair_beliefs`
- the `api.research` wrapper
- the `research` and `repair` CLI commands
- dry-run behavior and exact metadata side effects

Most LLM and database mutation boundaries are mocked, so these are mostly orchestration and contract tests, not end-to-end LLM tests.

---

## Key Components

### Module-level helpers

- `run_cli(*args, db_path=None)`
  - Builds an argv like `["reasons", "--db", db_path, *args]`.
  - Patches `sys.argv`, `sys.stdout`, `sys.stderr`.
  - Calls `reasons.cli.main`.
  - Catches `SystemExit` and returns `(stdout, stderr, exit_code)`.
  - Normal return without `SystemExit` maps to exit code `0`.

- `_make_nodes()`
  - Returns a small graph of nodes:
    - `premise-a`, `premise-b`, `premise-c`: premises with no justifications.
    - `derived-boil`: has an `SL` justification whose antecedents are `premise-b` and `premise-c`.
  - Used by depth and repair tests.

- `_mock_result(stdout)`
  - Returns a fake subprocess result object with:
    - `returncode = 0`
    - `stdout = stdout`
    - `stderr = ""`
  - Used as the return value of patched `reasons.llm.subprocess.run`.

- `_mock_search(results)`
  - Returns a callable `lambda query, **kw: json.dumps(results)`.
  - Injected into `repair_beliefs(..., search_fn=...)`.
  - This establishes the contract that `search_fn` returns a JSON string.

---

### Parser tests

#### `TestParseTriageResponse`

Covers `reasons.repair.parse_triage_response`.

Contract:

- It parses JSON from an LLM response.
- JSON may be embedded in prose.
- Recognized patterns include:
  - `"search_and_link"`
  - `"soften"`
  - `"abandon"`
  - `"research"`
- Unknown patterns produce `{"pattern": ""}`.
- Malformed/non-JSON input produces `{"pattern": ""}`.

Tests include:

- valid search-and-link
- valid soften
- valid abandon
- valid research
- unknown pattern ignored
- JSON embedded in surrounding prose
- malformed response

#### `TestParseSoftenResponse`

Covers `reasons.repair.parse_soften_response`.

Contract:

- Parses `{"softened_text": ..., "rationale": ...}`.
- Allows empty `softened_text`.
- Malformed input returns `softened_text == ""`.
- JSON may be embedded in prose.

---

### Depth calculation tests

#### `TestComputeDepth`

Covers `reasons.repair._compute_depth(node_id, nodes)`.

Contract:

- A premise with no justifications has depth `0`.
- A derived node with one `SL` justification over antecedents has depth `1`.
- A derived node whose antecedent is itself derived has depth `2`.
- Missing nodes return depth `0`, not an error.

This function appears to recursively compute the maximum justification depth from antecedents.

---

### LLM helper tests

#### `TestTriageBelief`

Covers `triage_belief(...)`.

Contract:

- It calls the LLM through `reasons.llm` / `subprocess.run`.
- It returns parsed triage output.
- It accepts at least:
  - `belief_context`
  - `review_comment`
  - `depth`
  - `flagged_ancestors`
- The prompt includes the numeric `depth` and `flagged_ancestors` values.

Tests patch:

```python
patch("reasons.llm.shutil.which", return_value="/usr/bin/claude")
patch("reasons.llm.subprocess.run", return_value=resp)
```

The second test inspects `mock_run.call_args[1]["input"]` and asserts `"5"` and `"3"` appear in the prompt.

#### `TestSoftenBelief`

Covers `soften_belief(claim, justification_summary)`.

Contract:

- It calls the LLM and parses a response containing `softened_text`.
- The test passes a claim and a summary like `"- ant: evidence"`.

---

### Orchestration tests

#### `TestResearchBeliefs`

Despite the class name, these tests call `repair_beliefs`.

Important implied contract:

```python
repair_beliefs(
    review_results,
    nodes,
    db_path=...,
    search_fn=...,
    dry_run=False,
)
```

It returns a list of result dicts. Each result may contain:

- `id`
- `pattern`
- `status`
- `softened_text`
- `matched_premises`
- `error`

Tests cover these statuses:

| Pattern | Status | Mutation behavior |
|---|---|---|
| `search_and_link` | `"linked"` | add justification, set metadata |
| `soften` | `"softened"` | supersede with softened text, set metadata on new node |
| `abandon` | `"abandoned"` | retract node, set metadata |
| `research` | `"needs_research"` | set research metadata only |
| empty triage pattern | `"triage_failed"` | no mutation |
| soften returned empty text | `"soften_failed"` | no mutation |
| search extraction returned whitespace | `"extraction_failed"` | no mutation |
| missing node id | `"error"` with `"not found"` | no mutation |
| review entry with `valid=True` | skipped entirely | no result entry |

Exact metadata expectations are significant:

- `search_and_link`:
  - `set_metadata("derived-boil", "repair_action", "search_and_link", db_path="test.db")`

- `soften`:
  - `supersede_with_text("derived-boil", softened_text, db_path="test.db")`
  - `set_metadata(new_id, "repair_action", "softened", db_path="test.db")`

- `abandon`:
  - `retract_node(...)` with a `reason` containing `"abandoned"`
  - `set_metadata(node_id, "repair_action", "abandoned", db_path="test.db")`

- `research`:
  - `set_metadata(node_id, "repair_research", review_comment, db_path="test.db")`
  - `set_metadata(node_id, "repair_action", "research", db_path="test.db")`

- Dry run:
  - `research` with `dry_run=True` returns `"needs_research"` but calls no `set_metadata`.
  - Soften with `dry_run=True` returns `"softened"` but does not call `retract_node`, `add_justification`, or `set_metadata`.

The multiple-belief test verifies sequential processing and that the mocked LLM side effects are consumed in order:

```python
side_effect=[triage1, soften1, triage2]
```

---

### API wrapper tests

#### `TestApiResearch`

Covers `reasons.api.research`.

Contract:

- Can be called with `belief_ids=[...]`.
- Can be called with `review_file=...`.
- Accepts `model`, `dry_run`, and `db_path`.
- Returns a summary dict containing at least:
  - `total_invalid`
  - `linked`
  - `softened`
  - `abandoned`
  - `failed`
  - `errors`
  - `results`

Specific tests:

- `test_from_belief_ids`
  - Creates real nodes in a temporary SQLite DB.
  - LLM review response marks `derived-a` invalid.
  - Triage response says `"abandon"`.
  - With `dry_run=True`, summary reports `total_invalid == 1`, `abandoned == 1`, and one result.

- `test_counts_add_up`
  - Uses a `review_file` shaped like:
    ```json
    {"results": [{"id": "...", "valid": false, ...}]}
    ```
  - Triage says `search_and_link`.
  - Claim extraction returns whitespace, so result status is `extraction_failed`.
  - Asserts:
    ```python
    total_invalid == linked + softened + abandoned + failed + errors
    ```
  - Asserts `failed == 1`.

- `test_no_invalid`
  - Review says the belief is valid.
  - `total_invalid == 0`.
  - `results == []`.
  - No triage LLM call is made.

---

### CLI tests

#### `TestCmdResearch`

Covers the `research` CLI command.

- `research --help` exits `0` and includes `"research"` in stdout.
- `research` with no review-file or belief input exits `1`.
- `research --review-file ...` with soften triage output exits `0` and stdout contains `"SOFTENED"`.
- `research --review-file ... --dry-run`:
  - exits `0`
  - stdout contains `"ABANDONED"`
  - stdout contains `"dry run"` case-insensitively
  - verifies the node still has `truth_value == "IN"` after the dry run

#### `TestCmdRepair`

Covers the `repair` CLI alias.

- `repair --help` exits `0` and includes `"repair"`.
- `repair --review-file ...` behaves like `research --review-file ...` and outputs `"SOFTENED"`.

#### `TestBackwardCompatAlias`

Identity checks:

```python
research_beliefs is repair_beliefs
api.research is api.repair
```

These are not “equivalent behavior” checks; they assert the same function object is exported under both names.

---

## Patterns

- **Arrange/Act/Assert** in each test method.
- **Test classes group behavior** by function or subsystem.
- **Boundary mocking**:
  - LLM boundary: `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`
  - Database mutation boundary: `reasons.api.add_justification`, `set_metadata`, `retract_node`, `supersede_with_text`
  - Search boundary: injectable `search_fn`
- **Side-effect sequencing** with `unittest.mock`:
  - `subprocess.run` uses `side_effect=[triage_resp, soften_resp, ...]` to simulate multiple LLM calls in order.
- **Dependency injection**:
  - `repair_beliefs(..., search_fn=...)` allows tests to supply a fake search implementation.
- **Fake subprocess result** via `_mock_result`.
- **CLI harness** via `run_cli`, which patches `sys.argv`, captures output, and normalizes `SystemExit` to an exit code.
- **Temporary SQLite DB** via pytest’s `tmp_path` fixture for API and CLI tests.
- **Compatibility alias assertions** ensure legacy `research` names remain identity aliases of `repair` names.

---

## Dependencies

### Imports used by this test file

- `json` — encodes/decodes mock LLM/search/review payloads.
- `sys` — patched by `run_cli`.
- `StringIO` — captures stdout/stderr in CLI tests.
- `unittest.mock.patch` — replaces LLM, search, and DB mutation boundaries.
- `pytest` — imported; `tmp_path` is used as a fixture in API/CLI tests.
- `reasons.repair`:
  - `parse_triage_response`
  - `parse_soften_response`
  - `triage_belief`
  - `soften_belief`
  - `repair_beliefs`
  - `research_beliefs`
  - `_compute_depth`
- `reasons.api` — used for `add_node`, `show_node`, `research`, and patched mutation calls.
- `reasons.cli.main` — invoked by `run_cli`.

### What this file depends on

This test file depends on the public and semi-public contracts of:

- `reasons/repair.py`
- `reasons/api.py`
- `reasons/cli.py`
- `reasons/llm.py` indirectly, because tests patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`

### What depends on this file

No production module imports this test file. It is collected by pytest as part of the test suite.

---

## Flow

A typical non-dry-run repair flow tested here is:

1. `api.research` receives either:
   - explicit `belief_ids`, or
   - a `review_file` containing a `results` array.
2. If `belief_ids` are supplied, review runs first via the LLM. If a `review_file` is supplied, the existing review results are loaded.
3. `api.research` delegates to `repair_beliefs` / `api.repair`.
4. `repair_beliefs` iterates over review results.
5. Review entries with `valid=True` are skipped and produce no result.
6. For each invalid belief:
   - Look up the node by id.
   - If missing, return a result with `status="error"` and an error message containing `"not found"`.
   - Compute depth using `_compute_depth`.
   - Call `triage_belief(...)` with belief context, review comment, depth, and flagged ancestor count.
   - Parse the triage response.
   - Dispatch based on `pattern`.

Dispatch behavior:

- `search_and_link`:
  - Ask the LLM to extract a factual claim from the review context.
  - If extraction is empty/whitespace, result is `"extraction_failed"`.
  - Otherwise call injected `search_fn(query)`.
  - Ask the LLM to match search results to premise ids.
  - Link matched premises to the original belief via `add_justification`.
  - Record metadata `repair_action = "search_and_link"`.
  - Result status is `"linked"`.

- `soften`:
  - Call `soften_belief(...)`.
  - If no softened text, result is `"soften_failed"`.
  - Otherwise call `supersede_with_text(old_id, softened_text, db_path=...)`.
  - Record metadata on the new node: `repair_action = "softened"`.
  - Result status is `"softened"`.

- `abandon`:
  - Call `retract_node(..., reason=...)` with an abandonment reason.
  - Record metadata: `repair_action = "abandoned"`.
  - Result status is `"abandoned"`.

- `research`:
  - No mutation.
  - Record `repair_research = review_comment`.
  - Record `repair_action = "research"`.
  - Result status is `"needs_research"`.

- Empty triage pattern:
  - Result status is `"triage_failed"`.

When `dry_run=True`, the tests expect planned statuses to still be reported, but database mutation APIs must not be called. The CLI dry-run test additionally confirms the node remains `IN`.

At the API layer, results are aggregated into summary counters. In the tested accounting case:

```python
total_invalid == linked + softened + abandoned + failed + errors
```

At the CLI layer, `run_cli` invokes `main()`, captures output, and tests assert on user-facing strings like `"SOFTENED"`, `"ABANDONED"`, and `"dry run"`.

---

## Invariants

- `repair_beliefs` only processes invalid review entries; `valid=True` entries are skipped.
- Missing node ids produce `status="error"` with `"not found"` in the error message.
- `parse_triage_response` only accepts the four known patterns:
  - `search_and_link`
  - `soften`
  - `abandon`
  - `research`
- Unknown or malformed triage responses produce an empty pattern and later `"triage_failed"`.
- Malformed soften responses produce empty `softened_text` and later `"soften_failed"`.
- Search-and-link extraction must contain non-whitespace text; whitespace-only extraction produces `"extraction_failed"`.
- `_compute_depth` returns `0` for missing nodes and for premises with no justifications.
- `dry_run=True` must not call mutation APIs such as `retract_node`, `add_justification`, or `set_metadata`.
- Metadata keys and values are contractually asserted:
  - `repair_action = "search_and_link"`
  - `repair_action = "softened"`
  - `repair_action = "abandoned"`
  - `repair_action = "research"`
  - `repair_research = <review comment>`
- `research_beliefs` must be the same object as `repair_beliefs`.
- `api.research` must be the same object as `api.repair`.
- Both `research` and `repair` CLI commands must exist and produce equivalent successful output for the tested soften flow.

---

## Error Handling

- Parser functions do not raise on malformed LLM output; they return empty fields.
- `repair_beliefs` converts failure cases into result statuses rather than raising:
  - `"triage_failed"`
  - `"soften_failed"`
  - `"extraction_failed"`
  - `"error"` for missing nodes
- `run_cli` catches `SystemExit` from argparse or CLI error paths and converts it to a numeric exit code.
- `run_cli` does not catch arbitrary exceptions; unexpected exceptions still fail tests.
- Missing CLI arguments are expected to exit `1` via `SystemExit`.
- API summary counters hide individual failure details in aggregate fields, but the `results` list preserves per-belief status information.

---

## Topics to Explore

- [file] `reasons/repair.py` — The implementation under test: triage dispatch, parser behavior, prompt construction, dry-run handling, and metadata writes.
- [function] `reasons/repair.py:repair_beliefs` — Central orchestration loop for invalid beliefs and the search/soften/abandon/research branches.
- [function] `reasons/api.py:research` — API wrapper that normalizes `belief_ids` and `review_file` inputs and aggregates result counters.
- [file] `reasons/llm.py` — The LLM subprocess boundary patched by these tests; explains how `shutil.which` and `subprocess.run` are actually used.
- [file] `reasons/cli.py` — Defines `research` and `repair` command wiring and user-facing output such as `"SOFTENED"` and `"ABANDONED"`.

---

## Beliefs

- `research-beliefs-is-repair-beliefs-alias` — `reasons.repair.research_beliefs` is the same function object as `reasons.repair.repair_beliefs`.
- `api-research-is-api-repair-alias` — `reasons.api.research` is the same function object as `reasons.api.repair`.
- `repair-beliefs-skips-valid-results` — `repair_beliefs` produces zero result entries for review items with `valid=True`.
- `repair-beliefs-reports-missing-nodes-as-errors` — Missing node IDs produce `status="error"` with `"not found"` in the error message rather than raising an exception.
- `dry-run-prevents-database-mutations` — When `dry_run=True`, the tested repair path avoids `retract_node`, `add_justification`, and `set_metadata` calls while still reporting planned statuses such as `softened` or `abandoned`.
