---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_verify.py

# `tests/test_verify.py` — Verify Command Test Suite

## Purpose

This is the pytest suite for the `verify` feature (implemented in `reasons/verify.py` and exposed via `reasons/api.py` and `reasons/cli.py`). "Verify" asks an LLM (invoked through the `claude` CLI) to check each belief's claim against the raw content of its `source` document, and classifies each belief as `CONFIRMED`, `PARTIAL`, `STALE`, or `INCONCLUSIVE`. This file locks down that behavior at three layers:

1. **Pure helpers** in `reasons.verify`: `parse_verify_response`, `format_belief_for_verify`, `read_source`.
2. **The orchestration API**: `reasons.api.verify_belief` (routing, timestamps, retraction, dry-run, trace).
3. **The CLI surface**: `reasons verify ...` via `reasons.cli.main`.

The file does **not** test the LLM prompt content itself — the subprocess to `claude` is always stubbed — so it focuses on data transformation, result routing, and DB side effects.

---

## Key Components

### Helpers (module scope)

- **`run_cli(*args, db_path=None)`** — argv builder + invocation harness.
  - Prepends `["reasons"]` and, if `db_path` is given, `["--db", db_path]`.
  - Patches `sys.argv`, `sys.stdout`, `sys.stderr` onto `StringIO` buffers.
  - Calls `main()`, catches `SystemExit` and returns `(stdout, stderr, exit_code)`.
  - Contract: always returns a 3-tuple; treats a non-SystemExit normal return as `code == 0`.

- **`_mock_result(stdout)`** — factory producing a stand-in for `subprocess.CompletedProcess` with `returncode=0`, given `stdout`, empty `stderr`. Implemented with `type("R", (), {...})()` — a throwaway namespace object, not `unittest.mock.MagicMock`.

### `TestParseVerifyResponse`

Tests the LLM-output parser. Expected LLM shape is a JSON object keyed by node id, each value `{verdict, reason, quote}`.

| Test | Contract asserted |
|---|---|
| `test_valid_response` | Well-formed JSON is returned with all three fields preserved. |
| `test_json_in_prose` | JSON embedded in surrounding prose is extracted. |
| `test_malformed` | Non-JSON string → `{}` (no exception). |
| `test_empty` | Empty string → `{}`. |
| `test_missing_verdict_skipped` | Entries missing a `"verdict"` key are **dropped**; siblings survive. |
| `test_case_insensitive` | `"confirmed"` is normalized to `"CONFIRMED"`. |

### `TestFormatBeliefForVerify`

Tests `format_belief_for_verify(belief_dict, source_content)` — builds the prompt fragment for one belief.

- `test_basic_format`: output contains `### \`p1\`` header, `**Claim:** A claim`, the `source` path, `source_url`, and the raw source content.
- `test_no_source`: with empty `source`/`source_url` and `None` content, output contains the literal `(source not available)`.

### `TestReadSource`

Tests `read_source(source, db_path=...)`:

- Reads a relative filename resolved against `db_path`'s directory (`tmp_path`).
- Returns `None` for missing files and for `source == ""`.
- Truncates: writes a 40,000-char file, asserts returned length `< 35_000` and that the marker `"[... truncated ...]"` is present. Note the test asserts a *bound*, not the exact truncation constant — the internal limit is ≤ ~32k.

### `TestApiVerifyBelief`

`api.verify_belief(node_id, *, trace, retract, dry_run, db_path)` behavior. Each test builds a real SQLite DB in `tmp_path` via `api.add_node`/`api.init_db` and mocks the LLM with `_mock_result(llm_resp)`.

- `test_premise_verified`: `CONFIRMED` → `result["verified"] == ["p1"]`, `result["stale"] == []`, `result["results"]["p1"]["verdict"] == "CONFIRMED"`.
- `test_verified_at_stamped`: after verify, `api.show_node("p1")["verified_at"] is not None`. So verification writes a timestamp.
- `test_stale_detected`: `STALE` → `result["stale"] == ["p1"]`, `verified == []`.
- `test_retract_stale`: `retract=True` + `STALE` → node's `truth_value` becomes `"OUT"`.
- `test_dry_run`: `dry_run=True` returns `dry_run=True`, a non-empty `beliefs_checked` list (1 item), and `results == {}` — **no LLM call, LLM is not even mocked here**.
- `test_trace_derived`: a derived node `d1` with `sl="p-a,p-b"`, `trace=True` → checks **both premises** (`len == 2`), `set(verified) == {"p-a","p-b"}`, and `result["is_derived"] is True`.
- `test_derived_without_trace`: same shape with `trace=False` → only `d1` is checked (`len == 1`, `id == "d1"`), and its own verdict is applied.
- `test_missing_node_raises`: `verify_belief("nonexistent")` raises `KeyError`.

### `TestCliVerify`

- `test_help`: `reasons verify --help` exits 0 and prints "verify".
- `test_dry_run`: CLI prints the belief id `p1` and the word "dry-run".
- `test_basic_verify`: with a mocked confirming LLM, exit 0, stdout contains `"CONFIRMED"` and the summary phrase `"1 confirmed"`.

---

## Patterns

- **Class-per-unit** pytest organization (`TestParseVerifyResponse`, `TestApiVerifyBelief`, ...) with method names encoding the behavior under test.
- **Fixture-driven isolation**: `tmp_path` supplies a fresh filesystem + DB per test; every test that touches persistence builds a real SQLite file with `api.add_node(..., db_path=db)`.
- **CLI harness idiom**: `run_cli` centralizes the `sys.argv`/`sys.stdout`/`sys.stderr` patching and `SystemExit` capture, so CLI tests read like assertions on `(stdout, stderr, code)`.
- **LLM stub pattern**: `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` are patched *in the `reasons.llm` module namespace* (not `shutil`/`subprocess` globally) — this means `reasons.verify` must route its LLM call through `reasons.llm`. `_mock_result` mimics `CompletedProcess` enough for `stdout` parsing.
- **Arrange-Act-Assert with explicit side-effect checks**: after the call under test, the suite re-reads state via `api.show_node(...)` to assert persistence occurred.
- **Bound assertions for truncation** instead of exact equality, so the internal truncation constant isn't pinned.

## Dependencies

**Imports (this file):**
- stdlib: `json`, `sys`, `io.StringIO`, `pathlib.Path`, `unittest.mock.patch`
- `pytest` (for `tmp_path` fixture and `pytest.raises`)
- `reasons.api` — `add_node`, `init_db`, `show_node`, `verify_belief`
- `reasons.verify` — `format_belief_for_verify`, `parse_verify_response`, `read_source`
- `reasons.cli` — `main`

**Imported by (dependents):** the pytest runner / CI only. No source module imports this file.

**Note:** `from pathlib import Path` is imported but not referenced in the visible code — a minor dead import.

## Flow

Typical test lifecycle:

1. `db = str(tmp_path / "test.db")`; write a `doc.md` with source text into `tmp_path`.
2. Seed the graph via `api.add_node(id, text, source=..., sl=..., db_path=db)` — for derived-node tests, seed premises first, then the derived node with `sl` pointing at them.
3. Build the desired LLM JSON response and a `_mock_result`.
4. Enter `with patch("reasons.llm.shutil.which", return_value="/usr/bin/claude"), patch("reasons.llm.subprocess.run", return_value=mock):`.
5. Call the target (`api.verify_belief(...)` or `run_cli("verify", ...)`).
6. Assert on the returned dict (`verified`/`stale`/`results`/`is_derived`/`beliefs_checked`) and/or re-read node state with `api.show_node`.

For `run_cli`, the control flow is: build argv → patch std streams → `main()` → catch `SystemExit` → return tuple. `argparse`'s `--help` raises `SystemExit(0)`, which the harness converts to `code == 0`.

## Invariants

- **Parser agnostic to noise**: `parse_verify_response` returns `{}` for malformed or empty input and never raises; entries lacking `verdict` are excluded.
- **Verdict normalization**: verdict strings are case-insensitive on input and uppercase on output.
- **Source reading is total**: `read_source` returns `None` for empty/missing sources rather than raising; files above the internal size limit are truncated with `"[... truncated ...]"`, keeping output under ~35k chars.
- **Dry-run is side-effect-free at the LLM layer**: `dry_run=True` returns `results == {}` and never invokes the LLM (the test does not stub it, proving this).
- **Trace routing**: `trace=True` on a derived node (`sl` set) verifies **premises**, not the derived node, and sets `is_derived=True`; `trace=False` verifies the node itself.
- **Retraction**: `retract=True` + `STALE` sets the node's `truth_value` to `"OUT"`.
- **Confirmation stamps time**: any successful verification updates `verified_at`.
- **Missing node is an error**: `verify_belief` on unknown id raises `KeyError` (propagated to the caller; the CLI layer, not this API, decides how to surface it).

## Error Handling

- **Swallowed (by design):** `parse_verify_response` absorbs malformed/empty JSON → `{}`. `read_source` absorbs missing files / empty source → `None`. `run_cli` absorbs `SystemExit` and converts it to an exit code in the returned tuple.
- **Propagated:** `api.verify_belief` raises `KeyError` for unknown node ids (asserted via `pytest.raises(KeyError)`).
- **Mocked away:** all LLM/transport failures (`subprocess` exit codes, CLI absence) are stubbed with a clean `returncode=0`; the suite does not exercise LLM error paths.

---

## Topics to Explore

- [file] `reasons/verify.py` — The module under test; contains `parse_verify_response`, `format_belief_for_verify`, `read_source`, and the truncation constant this suite only bounds.
- [function] `reasons/verify.py:parse_verify_response` — How JSON-in-prose extraction and missing-verdict filtering are implemented (regex? `json.JSONDecoder.raw_decode`?).
- [function] `reasons/api.py:verify_belief` — Orchestrates trace expansion, per-node verification, retraction to `OUT`, and `verified_at` stamping.
- [general] `trace-expansion` — How `sl` premises are recursively collected and de-duplicated for `trace=True`, and how `is_derived` is computed.
- [file] `tests/conftest.py` — Shared fixtures and any autouse setup that could affect DB/CLI state across these tests.

## Beliefs

- `verify-dry-run-skips-llm` — With `dry_run=True`, `api.verify_belief` returns `results == {}` without mocking or invoking the LLM.
- `verify-trace-checks-premises-not-derived` — With `trace=True`, a derived node's `beliefs_checked` contains its premises (e.g. `p-a, p-b`), not the derived node itself.
- `verify-retract-sets-out` — `retract=True` combined with a `STALE` verdict sets the node's `truth_value` to `"OUT"`.
- `parse-verify-response-drops-missing-verdict` — Entries in the LLM JSON without a `verdict` key are excluded from the parsed result.
- `verify-success-stamps-verified-at` — Any successful `api.verify_belief` call populates the node's `verified_at` field.
- `read-source-truncates-to-marker` — Sources over the internal limit are truncated and contain the marker `"[... truncated ...]"`, and the result is `< 35_000` chars for a 40_000-char input.
