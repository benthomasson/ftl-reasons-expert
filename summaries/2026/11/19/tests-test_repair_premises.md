---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_repair_premises.py

# `tests/test_repair_premises.py` — Explanation

## Purpose

This is the pytest suite for the **"repair premises"** workflow — the feature that takes review verdicts against observed/belief nodes (e.g. "this premise misread the source") and asks an LLM to **rewrite or retract** each inaccurate premise, then persists the result to the graph.

It owns four distinct layers of verification:

1. **Prompt construction** — `format_premise_for_repair` renders a single node + its review into a Markdown-ish block.
2. **Response parsing** — `parse_repair_response` turns an LLM reply (possibly wrapped in prose) into a list of action dicts, with fail-safe defaults.
3. **The per-node repair loop** — `reasons.repair_premises.repair_premises`, exercised both sequentially and with `parallel=2`, including failure and callback semantics.
4. **The DB-facing orchestrator** — `reasons.api.repair_premises`, exercised against a real temp SQLite DB: it reads a review file, filters inaccurate items, runs the loop, and applies rewrites/retractions (except under `dry_run`).
5. A **CLI smoke test** (`TestCliDispatch`) and a **cross-module parallel-batching test** for `review_premises` (`TestReviewPremisesParallel`).

The LLM boundary is always mocked at `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`, so the suite treats `reasons.llm` as the *only* seam for model I/O.

---

## Key Components

### `TestFormatPremiseForRepair`

Tests `format_premise_for_repair(node_id, nodes, source_content, review)`:

| Case | Behavior asserted |
|---|---|
| `test_basic_formatting` | Output contains `### {id}` header, node text, `error_type`, review comment body, and full source text. |
| `test_missing_node` | Node ID absent from `nodes` → returns `""` (empty string, not an exception). |
| `test_no_review_comment` | When `review` has no `comment` key, the literal label `"Review comment:"` is omitted from output, but `error_type` still appears. |

Contract implied: the header `### obs-1` matches the block format used to compose multi-premise prompts; the comment is *conditional on presence*, not on falsiness of some key.

### `TestParseRepairResponse`

Tests `parse_repair_response(text) -> list[dict]`:

| Case | Behavior asserted |
|---|---|
| `rewrite` | Preserves `action="rewrite"` and `corrected_text`. |
| `retract` | Preserves `action="retract"` and `corrected_text=None`. |
| JSON-with-prose | Extracts the JSON array even when wrapped by `"Here is my analysis:"` / `"Done."`. |
| Malformed JSON | Returns `[]` — **swallows** the parse error. |
| Defaults | Input `[{"id": "obs-1"}]` yields `action="retract"`, `corrected_text=None`, `rationale=""`. |

**The fail-safe default (`action="retract"`) is a deliberate invariant**: if the model omits `action`, the premise is *not* silently rewritten — it's retracted. Rewriting requires an explicit `corrected_text`.

### `TestRepairPremisesLoop`

Tests `reasons.repair_premises.repair_premises(nodes, ids, source_contents, review_results, parallel=?, on_result=?)`.

Helper `_mock_result(response_text)` builds a fake `subprocess.CompletedProcess` with `returncode=0`, `stdout=response_text`, `stderr=""` — used as the return of the mocked `reasons.llm.subprocess.run`.

| Case | Behavior asserted |
|---|---|
| `test_sequential_repair` | One id, one rewrite → 1 result with `action="rewrite"`. |
| `test_parallel_repair` | Two ids, `parallel=2`; mock inspects `kwargs["input"]` and returns per-id responses → 2 results, correct actions per id. |
| `test_empty_ids` | `[]` in → `[]` out. |
| `test_on_result_callback` | `on_result` is called with cumulative results per batch: `callbacks == [1]`. |
| `test_llm_failure_returns_error` | `subprocess.run` raises `Exception("LLM down")` → result list of length 1 with `action == "error"`. Nothing is re-raised. |

**Implicit contract from `test_parallel_repair`**: prompts are delivered via `subprocess.run(..., input=...)` (stdin), *not* as argv, because the mock reads `kwargs.get("input", "")`. The mock returns only *one* id per call and the test still expects *two* results — therefore, with default batching, `repair_premises` sends **one premise per LLM call** (batch size 1). The `parallel` argument parallelizes those per-premise calls.

### `TestApiRepairPremises` and `TestRepairActionMetadata`

Both classes define an identical `db_with_reviewed` fixture:

- Temp dir gets a SQLite file (`tmp_path / "test.db"`) and a source file (`entry.md` = `"The system uses PostgreSQL for storage."`).
- `api.init_db(db_path=db)` then `api.add_node` for `obs-1` (wrong: "Redis…") and `obs-2` (fabricated: "Memcached…"), both tagged `source=<source_file>`.
- A review JSON is written with the schema
  ```json
  {"results": [
    {"id": "obs-1", "accurate": false, "well_scoped": true,
     "error_type": "misread_source", "comment": "Source says PostgreSQL"},
    {"id": "obs-2", "accurate": false, "well_scoped": true,
     "error_type": "fabricated", "comment": "Source never mentions caching"}
  ]}
  ```

Tests assert the following `api.repair_premises(...)` contract:

| Case | Behavior asserted |
|---|---|
| `test_repair_from_review_file` | `dry_run=True` → result has `total_inaccurate == 2`. |
| `test_rewrite_updates_node` | Non-dry-run: `rewritten >= 1`; `obs-1` becomes `truth_value == "OUT"`; the successor node (identified by `result["results"][i]["new_id"]`) carries the corrected text. |
| `test_retract_removes_node` | Non-dry-run: `retracted >= 1`; if `obs-2` still exists, its `truth_value == "OUT"` (the test tolerates either "node kept but OUT" or "node deleted"). |
| `test_dry_run_no_changes` | `dry_run=True` → `obs-1` text is unchanged (`"Redis is used for storage"`). |
| `test_no_args_raises` | `api.repair_premises()` (both `review_file` and `belief_ids` omitted) raises `ValueError` matching `"Either review_file or belief_ids"`. |
| `test_no_inaccurate_returns_empty` | All-accurate review file → `total_inaccurate == 0`, `results == []` (short-circuits before any LLM call). |
| `test_rewrite_sets_repair_action` | `obs-1` OUT; the successor node's `metadata["repair_action"] == "rewritten"`. |
| `test_retract_sets_repair_action` | `obs-2` still reachable via `api.show_node` and has `metadata["repair_action"] == "retracted"`. |
| `test_dry_run_no_repair_action` | Dry run → `metadata.get("repair_action") is None` (no metadata written). |

### `TestCliDispatch`

`from reasons import cli` then `assert hasattr(cli, "cmd_repair_premises")` and `callable(...)`. This is a registration smoke test — it guarantees the CLI dispatcher advertises `cmd_repair_premises` (whatever mechanism `reasons.cli` uses to bind subcommands).

### `TestReviewPremisesParallel`

Imports `from reasons.review_premises import review_premises` and runs it with `parallel=2, batch_size=5` against 10 fake nodes. The mock returns *multiple* accurate entries per call by scanning `kwargs["input"]` for each `obs-{i}` substring. Asserts `len(results) == 10`.

This test is in an odd home — it tests `review_premises`, not `repair_premises`. It's presumably here because both share the `reasons.llm` subprocess/batching infrastructure, and this file is the closest convenient home. Worth moving eventually.

---

## Patterns

- **Fixture-based integration setup**: `db_with_reviewed` (duplicated in two classes) creates a full DB via the public `reasons.api` surface — `init_db`, `add_node`, `show_node`, `export_network`. Tests never touch `storage` directly.
- **Mock at the LLM seam, not the LLM caller**: every LLM-dependent test patches `reasons.llm.shutil.which` → `"/usr/bin/claude"` (so the binary-existence guard passes) *and* `reasons.llm.subprocess.run` → a fake `CompletedProcess`. `repair_premises` and `review_premises` are tested end-to-end from the caller side.
- **Prompt-keyed side effects**: multi-node mocks use `kwargs["input"]` string containment to route a distinct response per premise id. This implicitly asserts prompt-includes-id behavior.
- **`_mock_result` factory** (`type("R", (), {...})()`) — cheap subclass-free fakes rather than `unittest.mock.MagicMock`, which keeps assertions on real dict fields rather than mock attributes.
- **Result-dict accumulation**: `api.repair_premises` returns counters (`total_inaccurate`, `rewritten`, `retracted`) plus a `results` list; tests often use `>= 1` instead of exact equality (defensive against parallelism ordering).
- **Fail-safe defaults**: `parse_repair_response` defaults to `"retract"`; the loop captures exceptions into `{"action": "error"}` instead of crashing.
- **Dry-run flag**: `dry_run=True` thread-bleeds through the whole call (`test_dry_run_no_changes`, `test_dry_run_no_repair_action`) — a cross-cutting mode, not a branch inside each mutation.

---

## Dependencies

**Imports in this test file:**
- `json` — encode/decode mock LLM strings and fixture review files.
- `unittest.mock.patch` — LLM seam.
- `pytest` — fixtures, `pytest.raises`.
- `from reasons import api` — `init_db`, `add_node`, `show_node`, `export_network`, and `api.repair_premises` (the DB orchestrator under test).
- `from reasons.repair_premises import format_premise_for_repair, parse_repair_response, repair_premises` — the three pure-ish units under test.
- `from reasons import cli` (imported lazily in `TestCliDispatch`) — command registration.
- `from reasons.review_premises import review_premises` (imported lazily in `TestReviewPremisesParallel`) — the parallel-batching unit under test.

**Production modules the tests reach:**
- `reasons/repair_premises.py` — subject of classes 1–3.
- `reasons/api.py` — subject of `TestApiRepairPremises` / `TestRepairActionMetadata`.
- `reasons/llm.py` — the *mock seam*; both `repair_premises` and `review_premises` presumably delegate to `reasons.llm` which calls `shutil.which("claude")` and `subprocess.run(..., input=prompt)`.
- `reasons/review_premises.py` — subject of the parallel-batch test.
- `reasons/cli.py` — subject of the registration test.

**What depends on this file:** nothing in the runtime path — it's a leaf test file. However, changes to `repair_premises.repair_premises`, `api.repair_premises`, `parse_repair_response`, `format_premise_for_repair`, node metadata schema (`repair_action`), or the `input=`-stdin LLM convention will break this suite.

---

## Flow

### `reasons.repair_premises.repair_premises(...)` (module level)

1. Empty `ids` → returns `[]` (`test_empty_ids`).
2. Otherwise groups the ids into batches (default batch size implicitly 1 — see `test_parallel_repair`).
3. For each batch, builds the prompt by calling `format_premise_for_repair` per id (id is a no-op string builder when the node is missing — returns `""`).
4. Invokes the LLM via `reasons.llm` → `shutil.which("claude")` → `subprocess.run(..., input=prompt)`. Requests run `parallel`-at-a-time when `parallel > 1`.
5. Parses each `stdout` with `parse_repair_response`. Valid results are appended to the running result list.
6. **If step 4 raises**, appends a `{"action": "error", "id": ..., ...}` entry and continues (never re-raises).
7. Invokes `on_result(cumulative_results)` after each batch, if provided.
8. Returns the full result list.

### `api.repair_premises(...)` (DB orchestrator)

1. Validate input: exactly one of `review_file` / `belief_ids` must be given — else `raise ValueError("Either review_file or belief_ids …")`.
2. **Review-file path**: load JSON, keep entries with `accurate == False`, record `total_inaccurate = len(inaccurate)`.
3. If `total_inaccurate == 0`, return a summary `{total_inaccurate: 0, results: [], ...}` early (`test_no_inaccurate_returns_empty`) — no LLM call.
4. Load each node (`api.show_node` or equivalent) and its source contents (read the file at `node["source"]`) to build `nodes` and `source_contents`.
5. Call the module-level `repair_premises` loop with the filtered ids.
6. If `dry_run` is falsy, apply each action:
   - **rewrite** → mark old node `truth_value = "OUT"` (superseded); insert a successor node with the `corrected_text` and `metadata["repair_action"] = "rewritten"`; expose `result["results"][i]["new_id"]`.
   - **retract** → set `truth_value = "OUT"` and `metadata["repair_action"] = "retracted"` on the original node.
7. Under `dry_run`, steps 6's mutations are skipped entirely — `test_dry_run_no_changes` (text untouched) and `test_dry_run_no_repair_action` (metadata untouched) both pass.
8. Return counters + results list.

### `review_premises.review_premises(...)` (only for `TestReviewPremisesParallel`)

1. Chunks `premise_ids` into blocks of `batch_size` (5).
2. For each batch, builds one prompt referencing all premise ids in the batch.
3. Runs `parallel`-many batches directly on `reasons.llm` / `subprocess.run`, each returning a JSON array of per-id verdicts.
4. Aggregates → `len(results) == 10` for 10 ids at `batch_size=5, parallel=2`.

This is a *different* batching granularity from the module-level `repair_premises` loop (`review_premises` batches premises *into* one call; `repair_premises` seems to call one-per-premise). The two are tested side-by-side here, which is the confusing bit for a new maintainer.

---

## Invariants

- **`parse_repair_response` never raises on bad input**: malformed → `[]`; missing fields → defaults (`action="retract"`, `corrected_text=None`, `rationale=""`).
- **`format_premise_for_repair` returns `""` for unknown node ids** — callers must guard against empty blocks (`test_missing_node`).
- **Review comment label is conditional**: `"Review comment:"` only appears when a `comment` key is present (`test_no_review_comment`).
- **Exception on LLM subprocess is converted to a result**, not raised: `{"action": "error"}` (`test_llm_failure_returns_error`).
- **`api.repair_premises` requires `review_file` or `belief_ids`** (ValueError otherwise).
- **`accurate: False` is the only qualifying review signal** — filtered by `accurate` (not `well_scoped`, not `error_type`). `test_no_inaccurate_returns_empty` documents the short-circuit.
- **Dry-run is a hard no-mutation guarantee**: neither node text nor metadata changes when `dry_run=True`.
- **Rewrite preserves history** — old node ID remains in the graph with `truth_value="OUT"`; a fresh node ID carries the corrected text and `metadata["repair_action"]="rewritten"`.
- **Retract sets metadata `"retracted"`** and marks the node `OUT`; the node may still be resolvable via `show_node` (per `test_retract_sets_repair_action`, which reads `metadata` *after* the retract).
- **Prompt passed through stdin** — mocks read `kwargs["input"]`, so `subprocess.run` must be called with `input=<prompt>`, not as positionals.
- **Prompt embeds the node id** (`"obs-1" in prompt`) — the mock routing depends on this.

---

## Error Handling

| Failure mode | Where handled | Behavior |
|---|---|---|
| LLM returns prose / non-JSON | `parse_repair_response` | Returns `[]` — caller appends nothing. Silent loss of a result for that id (see caveat below). |
| LLM returns JSON but omits fields | `parse_repair_response` | Filled with defaults; retract-by-default is safest. |
| `subprocess.run` raises | `repair_premises` loop | Appends `{"action": "error", ...}`; does **not** propagate. |
| Node missing from `nodes` map | `format_premise_for_repair` | Returns `""`; caller is responsible for skipping empty prompts. |
| Missing CLI binary | mocked `shutil.which` | Real code likely raises if `which("claude")` returns `None` — the tests pin a path to bypass that branch. |
| No `review_file` and no `belief_ids` | `api.repair_premises` | `ValueError("Either review_file or belief_ids …")` — asserted by `pytest.raises`. |
| Review file has no inaccurate items | `api.repair_premises` | Short-circuits with `total_inaccurate=0, results=[]`; no LLM call. |
| DB mutation error (rewrite/retract) | not exercised here | **Untested** — a maintainer should check `reasons/api.py` for the propagation contract. |

**Untested edge worth knowing**: the loop-level tests never feed a malformed LLM response through `repair_premises` (only into `parse_repair_response` directly). Because the parser returns `[]`, a bad response would silently produce *fewer* result entries than inputs. If the orchestrator assumes `len(results) == len(ids)`, that assumption does not hold here.

---

## Topics to Explore

- [file] `reasons/repair_premises.py` — The implementation of `format_premise_for_repair`, `parse_repair_response`, and the `parallel` batch runner; the tests above define the contract, so this file is the ground truth for batching semantics.
- [file] `reasons/api.py` — `api.repair_premises` and the node-supersede machinery (`add_node`, `show_node`, `export_network`) that turns `rewrite`/`retract` actions into graph mutations.
- [file] `reasons/llm.py` — The `shutil.which("claude")` + `subprocess.run(input=…)` seam that both `repair_premises` and `review_premises` depend on; this is where every LLM-based test stubs the model.
- [file] `reasons/review_premises.py` — Contains the `parallel` + `batch_size` batching patterns exercised by `TestReviewPremisesParallel`; useful for contrasting with `repair_premises`'s per-premise calling convention.
- [general] `dry-run-semantics` — Compare how `dry_run` is threaded through `repair_premises` vs. other `api.*` mutations (`propose_update`, `compact`, etc.) to confirm a consistent convention.

---

## Beliefs

- `parse-repair-response-defaults-to-retract` — When an LLM result dict omits `action`, `parse_repair_response` fills in `action="retract"`, `corrected_text=None`, and `rationale=""`.
- `api-repair-premises-requires-review-file-or-belief-ids` — Calling `api.repair_premises()` with neither argument raises `ValueError` with a message matching `"Either review_file or belief_ids"`.
- `dry-run-prevents-db-mutation` — `api.repair_premises(dry_run=True, …)` leaves node `text` and `metadata["repair_action"]` unchanged and only populates counters/results.
- `rewrite-produces-superseding-node-with-rewritten-metadata` — A `rewrite` sets the original node `truth_value="OUT"` and creates a new node carrying `metadata["repair_action"]=="rewritten"`, whose id is exposed as `new_id` on the corresponding result.
- `repair-loop-converts-llm-failures-to-error-actions` — When `reasons.llm.subprocess.run` raises inside `repair_premises`, the loop returns an entry with `action == "error"` instead of propagating the exception.
- `llm-prompts-are-passed-via-subprocess-input-kwarg` — Tests intercept LLM calls by reading `kwargs["input"]` from the mocked `subprocess.run`, establishing that the prompt travels over stdin, not argv.
