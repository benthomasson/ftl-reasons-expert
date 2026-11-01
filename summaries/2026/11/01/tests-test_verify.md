---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_verify.py

# `tests/test_verify.py` — Test Suite for the Verify Command

## 1. Purpose

This file is the test suite for the **verify** feature in the `reasons` knowledge-base system. The verify command's job is to take a stored belief (a claim with a source reference), send it along with the source document content to an LLM, and get back a structured verdict: **CONFIRMED**, **STALE**, **PARTIAL**, or **INCONCLUSIVE**.

The test file owns responsibility for proving that:
- The LLM response parser is robust against malformed or prose-wrapped JSON.
- The prompt formatter produces the right structure for the LLM to consume.
- Source files are read and truncated safely.
- The full `api.verify_belief()` flow produces correct state transitions (verified stamps, retraction, tracing through derived beliefs).
- The CLI surface (`reasons verify <id>`) wires everything together and reports correctly.

## 2. Key Components

### Helpers

| Name | Contract |
|------|----------|
| `run_cli(*args, db_path=None)` | Splices `--db <path>` into a fake `sys.argv`, patches `sys.stdout`/`stderr`, calls `main()`, and returns `(stdout, stderr, exit_code)`. Catches `SystemExit` to recover the code. |
| `_mock_result(stdout)` | Returns a duck-typed object with `.returncode=0`, `.stdout`, `.stderr=""` — mimics `subprocess.run`'s result so `reasons.llm`'s internal `subprocess.run` call can be patched to return a canned LLM response. |

### Test Classes

**`TestParseVerifyResponse`** — exercises `reasons.verify.parse_verify_response`:
- `test_valid_response`: well-formed JSON → dict keyed by premise ID.
- `test_json_in_prose`: JSON embedded in surrounding text (LLM chit-chat) still parses.
- `test_malformed` / `test_empty`: returns `{}` rather than raising.
- `test_missing_verdict_skipped`: entries lacking a `"verdict"` key are silently dropped.
- `test_case_insensitive`: `"confirmed"` is normalized to `"CONFIRMED"`.

**`TestFormatBeliefForVerify`** — exercises `reasons.verify.format_belief_for_verify`:
- Asserts the output contains a markdown heading (`### \`p1\``), the claim text, source name, URL, and the source document body.
- When source is empty/`None`, the string `"(source not available)"` appears.

**`TestReadSource`** — exercises `reasons.verify.read_source`:
- Reads a real file from `tmp_path`.
- Returns `None` for missing files or empty path.
- Truncates content >~30k chars and appends `"[... truncated ...]"`.

**`TestApiVerifyBelief`** — the main integration block:
- `test_premise_verified`: happy path; asserts `result["verified"] == ["p1"]`.
- `test_verified_at_stamped`: after a CONFIRMED verdict, the node in the DB has a non-null `verified_at` timestamp.
- `test_stale_detected`: STALE verdict lands in `result["stale"]`.
- `test_retract_stale`: with `retract=True`, the node's `truth_value` flips to `"OUT"`.
- `test_dry_run`: no LLM call; returns `dry_run=True` and an empty `results` dict.
- `test_trace_derived`: for a derived node (`sl="p-a,p-b"`), `trace=True` causes the *premises* to be verified instead; `is_derived` is `True`.
- `test_derived_without_trace`: without `trace`, the derived node itself is sent to the LLM.
- `test_missing_node_raises`: `api.verify_belief("nonexistent")` raises `KeyError`.

**`TestCliVerify`** — end-to-end CLI:
- `test_help`: `--help` exits 0 and prints usage.
- `test_dry_run`: `verify p1 --dry-run` prints "dry-run" and the ID.
- `test_basic_verify`: full CLI run with mocked LLM; asserts "CONFIRMED" and "1 confirmed" in stdout.

## 3. Patterns

- **Three-layer test pyramid within one file.** Pure-function unit tests (`parse_verify_response`, `format_belief_for_verify`, `read_source`) sit at the bottom; API integration tests (`TestApiVerifyBelief`) in the middle; CLI tests (`TestCliVerify`) at the top. Each layer mocks one level lower.

- **LLM mocking via `subprocess.run` patch.** The project invokes an external CLI (`claude`) through `reasons.llm`. Tests patch `reasons.llm.subprocess.run` to return a `_mock_result`, and also patch `reasons.llm.shutil.which` so the code-believes the binary exists. This keeps tests hermetic — no network, no actual LLM.

- **`tmp_path` fixture for isolation.** Every API/CLI test creates its own SQLite DB and source files in a pytest temp directory. No shared mutable state.

- **CLI testing via `sys.argv` patching.** `run_cli` replaces `sys.argv`, `sys.stdout`, `sys.stderr`, then calls `main()`. `SystemExit` is caught to extract the exit code. This avoids spinning up a real subprocess.

- **Duck-typed mock (`_mock_result`).** Rather than importing `subprocess.CompletedProcess`, the helper builds a minimal namespace object with the three attributes the code under test reads. Keeps the mock decoupled from the stdlib type.

## 4. Dependencies

**This file imports (direct dependencies):**
- `reasons.api` — the public API (`add_node`, `show_node`, `verify_belief`, `init_db`).
- `reasons.verify` — the three pure helpers under test (`format_belief_for_verify`, `parse_verify_response`, `read_source`).
- `reasons.cli` — `main()` entry point for CLI tests.
- `reasons.llm` (indirectly, via patch targets) — `shutil.which`, `subprocess.run`.

**What depends on this file:**
- Nothing at runtime. It is a leaf in the dependency graph, consumed only by the test runner (`pytest`).

**External:**
- `pytest` (test framework, `tmp_path` fixture).
- `unittest.mock.patch` (monkey-patching).

## 5. Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  TEST EXECUTION (top-down)                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Unit layer (no DB, no LLM)                                     │
│     parse_verify_response(json_str) ──► dict or {}                  │
│     format_belief_for_verify(belief, src) ──► prompt string         │
│     read_source(path, db_path) ──► str | None                       │
│                                                                     │
│  2. API layer (real SQLite DB, mocked LLM)                          │
│     api.add_node(...)                                               │
│       │                                                             │
│       ▼                                                             │
│     api.verify_belief(id, trace=?, retract=?, dry_run=?)            │
│       │                                                             │
│       ├─► resolves node (KeyError if missing)                       │
│       ├─► if dry_run: return early                                  │
│       ├─► if trace & derived: resolve premises, verify each         │
│       ├─► else: verify the node itself                              │
│       │     read_source → format_belief_for_verify →                │
│       │     LLM call (mocked) → parse_verify_response               │
│       ├─► stamp verified_at on CONFIRMED                            │
│       ├─► if retract & STALE: set truth_value = "OUT"               │
│       ▼                                                             │
│     result dict: {verified, stale, results, is_derived, ...}        │
│                                                                     │
│  3. CLI layer (real DB, mocked LLM, patched sys.argv)               │
│     run_cli("verify", "p1", "--dry-run")                            │
│       ──► reasons.cli.main() ──► api.verify_belief ──► print        │
│       ──► assert on stdout text & exit code                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 6. Invariants

| Invariant | Where enforced |
|-----------|---------------|
| Verdict strings are **always uppercase** in parsed results. | `test_case_insensitive` asserts `"confirmed"` → `"CONFIRMED"` |
| Entries **without** a `"verdict"` key are **silently dropped**, not treated as errors. | `test_missing_verdict_skipped` |
| Source content is **truncated** to < 35 000 chars with an ellipsis marker. | `test_truncation` |
| A CONFIRMED verdict **stamps** `verified_at` on the node record. | `test_verified_at_stamped` |
| `retract=True` + STALE → `truth_value` becomes `"OUT"`. | `test_retract_stale` |
| `trace=True` on a derived node verifies **premises**, not the derivation. | `test_trace_derived` asserts `is_derived is True` and checks premises. |
| A nonexistent node ID **raises `KeyError`** (does not return an error dict). | `test_missing_node_raises` |
| `dry_run=True` performs **no** LLM call and returns an empty `results` map. | `test_dry_run` |

## 7. Error Handling

- **`parse_verify_response`** is defensive: any non-JSON input or empty string yields `{}`. It never raises. The test suite confirms this with `test_malformed` and `test_empty`.

- **`read_source`** returns `None` (not an exception) for missing files or empty paths. Callers are expected to handle `None`.

- **`api.verify_belief`** raises `KeyError` for unknown node IDs. The test uses `pytest.raises(KeyError)` — this is a *hard* failure, not a soft one.

- **`run_cli`** catches `SystemExit` (which `argparse` and many CLI frameworks use to signal exit codes) and converts it into a return tuple `(stdout, stderr, code)`. If `main()` doesn't call `sys.exit`, the helper returns code `0` as a default.

- **LLM subprocess failures** are not directly tested here; the mock always returns `returncode=0`. Failure paths (non-zero return, stderr output) would be in `reasons.llm`'s own tests.

---

## Topics to Explore

- [file] `reasons/verify.py` — The implementation under test; shows how `parse_verify_response` extracts JSON from prose, how `format_belief_for_verify` builds the prompt, and how `read_source` handles file I/O and truncation.
- [function] `reasons/api.py:verify_belief` — The orchestration function that ties together node lookup, tracing, LLM invocation, verdict application, and retraction. Understanding its branching logic (`trace`, `retract`, `dry_run`) is essential before modifying this test file.
- [file] `reasons/llm.py` — The LLM invocation layer. Tests patch `subprocess.run` and `shutil.which` in this module; knowing its call contract (how it builds the prompt, parses the response, handles timeouts) explains *why* the mock shape in `_mock_result` looks the way it does.
- [file] `reasons/cli.py` — The CLI entry point. `run_cli` in this test file drives `main()`; reading `cli.py` shows how the `verify` subcommand maps flags (`--dry-run`, `--trace`, `--retract`) to `api.verify_belief` kwargs.
- [general] `derived-belief-tracing` — How the system distinguishes premise nodes from derived nodes (via the `sl` field) and why `trace=True` changes the verification target. This is a core architectural concept that spans `verify`, `derive`, and `network` modules.

---

## Beliefs

- `parse-verify-response-tolerates-prose-wrapping` — `parse_verify_response` extracts the first JSON object from a string that may contain surrounding natural-language text, returning `{}` if no valid JSON is found.
- `verify-stamps-verified-at-on-confirm` — A CONFIRMED verdict causes the node's `verified_at` timestamp to be set to a non-null value in the database; STALE and PARTIAL verdicts do not.
- `retract-sets-truth-value-to-out` — When `retract=True` and the verdict is STALE, the node's `truth_value` field is set to the string `"OUT"`, effectively removing it from the active knowledge base.
- `trace-mode-shifts-target-to-premises` — `verify_belief(id, trace=True)` on a derived node (one with a non-empty `sl` field) verifies the *listed premises* rather than the derived node itself, and sets `is_derived=True` in the result.
- `cli-tests-mock-subprocess-not-http` — All LLM-dependent tests in this file intercept at the `subprocess.run` level (the external `claude` CLI), meaning no HTTP client, API key, or network call is ever made; the mock contract is `{returncode, stdout, stderr}`.
