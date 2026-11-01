---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_example.py

# `tests/test_example.py`

## 1. Purpose

This file is the integration test suite for the `--example` metadata field across the `reasons` CLI and its underlying API. Its single responsibility: **verify that a user can attach an example snippet to a belief node at creation time, retrieve it via the API, and see it rendered correctly in the `show` command output.**

It covers three surface areas:
- The programmatic API (`api.add_node(..., example=...)`)
- The CLI `add` subcommand with the `--example` flag
- The CLI `show` subcommand's display formatting of the example

It does *not* test `update` with `--example`, despite the docstring mentioning it — that's a gap worth noting.

## 2. Key Components

### `run_cli(*args, db_path=None)`
A test harness that simulates a real CLI invocation without spawning a subprocess.

| Aspect | Detail |
|--------|--------|
| **Input** | Positional CLI args (e.g. `"add", "n1", "A belief"`) and optional `db_path` |
| **Mechanism** | Patches `sys.argv` to `["reasons", "--db", <path>, *args]`, captures `sys.stdout`/`sys.stderr` via `StringIO` |
| **Output** | Tuple `(stdout_str, stderr_str, exit_code)` |
| **Error handling** | Catches `SystemExit` (which `argparse`-based CLIs raise on bad args) and normalizes `None` → `0` |

### `TestAddNodeExample`
Exercises the **API layer** directly (`api.add_node`, `api.show_node`):

- `test_add_with_example` — basic round-trip: store `"x = 1 + 2"`, read it back from `node["metadata"]["example"]`.
- `test_add_without_example` — negative case: the key must be *absent* from metadata, not present as `None` or `""`.
- `test_add_example_with_access_tags` — verifies `example` and `access_tags` coexist independently in metadata.

### `TestCmdAddExample`
Exercises the **CLI layer** via `run_cli`:

- `test_cli_add_with_example` — passes `--example "reasons add n1 'A belief'"` and asserts the stored value.
- `test_cli_add_without_example` — confirms the flag is optional and absence is clean.

### `TestCmdShowExample`
Exercises **CLI output formatting**:

- `test_show_displays_example` — asserts the literal strings `"Example:"` and the snippet appear in stdout.
- `test_show_multiline_example` — the most involved test. It stores a 3-line snippet, then manually parses stdout looking for the `"Example:"` header followed by lines indented with two spaces. It asserts exactly 3 indented lines are captured.
- `test_show_no_example` — confirms `"Example:"` is entirely absent from output when no example was set.

## 3. Patterns

- **CLI simulation via `unittest.mock.patch`**: Rather than shelling out (`subprocess.run`), the tests monkey-patch `sys.argv`/`stdout`/`stderr`. This keeps tests in-process, fast, and avoids PATH or environment issues. The tradeoff: it only tests the Python entry point, not the installed script's shebang or argument-parsing edge cases that a real shell would hit.

- **Layered testing (API → CLI)**: Each feature is tested at two levels. `TestAddNodeExample` hits `api.*` directly; `TestCmdAddExample` and `TestCmdShowExample` go through `cli.main()`. This catches bugs where the CLI layer forgets to forward a flag to the API.

- **`tmp_path` for isolation**: Every test creates its own SQLite file under pytest's temporary directory, so tests are order-independent and don't pollute a shared database.

- **Negative assertions for absence**: `assert "example" not in node["metadata"]` and `assert "Example:" not in stdout` are deliberate. They protect against a regression where the field defaults to `None`/empty string and clutters output.

## 4. Dependencies

**Imports (what this file depends on):**
- `sys`, `io.StringIO`, `unittest.mock.patch` — stdlib, for CLI simulation
- `pytest` — test runner (provides `tmp_path`, class-based grouping)
- `reasons.api` — the public API surface (`add_node`, `show_node`)
- `reasons.cli` — the CLI entry point (`main`), imported *inside* `run_cli` to avoid import-time side effects

**Dependents (what depends on this file):**
- Nothing imports this file. It's a leaf test module collected by pytest via the `tests/` directory convention. The only "dependents" are the CI pipeline (`.github/workflows/`) that runs `pytest`.

## 5. Flow

```
test_xxx(tmp_path)
  │
  ├─► api.add_node("n1", "A belief", example="x = 1 + 2", db_path=db)
  │     └─► writes to SQLite (metadata JSON blob)
  │
  ├─► api.show_node("n1", db_path=db)
  │     └─► reads SQLite, returns dict with "metadata" key
  │
  └─► run_cli("show", "n1", db_path=db)
        ├─ patches sys.argv → ["reasons", "--db", "<path>", "show", "n1"]
        ├─ patches sys.stdout / sys.stderr → StringIO buffers
        ├─ calls reasons.cli.main()
        │    ├─ argparse parses argv
        │    ├─ dispatches to show handler
        │    │    └─ calls api.show_node() internally
        │    │    └─ formats output: prints "Example:" + indented snippet
        │    └─ returns (or raises SystemExit)
        ├─ captures stdout.getvalue(), stderr.getvalue()
        └─ returns (stdout_str, stderr_str, exit_code)
```

For the multiline test, the data transformation is:
1. `example="x = 1\ny = 2\nprint(x + y)"` stored as a single string in the JSON metadata.
2. `show` splits on `\n` and indents each line by 2 spaces after the `Example:` header.
3. The test re-parses stdout line-by-line, looking for the `Example:` sentinel, then collecting subsequent 2-space-indented lines until a non-indented or empty line appears.

## 6. Invariants

| Invariant | Enforced by |
|-----------|-------------|
| `example` is **optional** — omitting it must not create the key in metadata | `test_add_without_example`, `test_cli_add_without_example` |
| `example` is stored **verbatim** (no trimming, escaping, or wrapping) | `test_add_with_example` compares exact string equality |
| `show` output includes `Example:` **only when** the field is present | `test_show_no_example` asserts absence |
| Multiline examples are rendered with **2-space indentation** on every line | `test_show_multiline_example` asserts `line.startswith("  ")` for all captured lines |
| `example` and `access_tags` are **orthogonal** metadata fields | `test_add_example_with_access_tags` sets both independently |
| `run_cli` normalizes `SystemExit(None)` to exit code `0` | The `code = e.code if e.code is not None else 0` line |

## 7. Error Handling

- **CLI invocation errors**: `run_cli` wraps `main()` in a `try/except SystemExit`. If `argparse` encounters an unknown flag, it raises `SystemExit(2)`; the test captures that as `code=2` and the assertion `code == 0` would fail, surfacing the bad args in the test report.
- **No exception-based assertions**: This file does *not* test error paths (e.g., what happens if you pass `--example` with no value, or an empty string). All tests are happy-path + absence-path.
- **`StringIO` buffering means no real I/O errors** can surface — if `main()` tries to `print()` to a closed stream, the `StringIO` will simply accumulate. There's no file-descriptor-level error handling to test here.
- **Swallowed errors**: If `api.add_node` raises (e.g., invalid node_id), it propagates uncaught to the test, which is correct behavior for a unit test — pytest will report the traceback. There's no intentional swallowing.

---

## Topics to Explore

- [file] `reasons/cli.py` — Contains the `add` and `show` command handlers; understanding how `--example` is parsed and forwarded (or rendered) is essential for debugging the formatting assertions in `TestCmdShowExample`.
- [function] `reasons/api.py:add_node` — The API entry point that stores the `example` field; look at how it serializes into the metadata JSON and whether it validates content (max length, allowed characters).
- [file] `tests/conftest.py` — May define shared fixtures (e.g., a pre-seeded database or a `reasons` module patch) that affect how `tmp_path`-based tests behave.
- [general] `example-field-lifecycle` — The docstring mentions `update` but no test covers `api.update_node(..., example=...)` or `run_cli("update", ..., "--example", ...)`. Explore whether this is a known gap or a missing feature.
- [file] `reasons/storage.py` — The persistence layer; understanding how the `metadata` JSON blob is stored and retrieved explains why `show_node` returns `node["metadata"]["example"]` as a nested dict access.

## Beliefs

- `example-is-optional-metadata` — Omitting the `example` parameter in `api.add_node` results in the key being entirely absent from `node["metadata"]`, not present as `None` or `""`.
- `show-renders-example-with-2-space-indent` — The `show` CLI command indents every line of a multiline example by exactly two spaces following an `Example:` header line.
- `run-cli-simulates-in-process` — `run_cli` in this file patches `sys.argv`/`stdout`/`stderr` in the current process rather than spawning a subprocess, so it tests the Python entry point but not the installed script's shell integration.
- `test-coverage-gap-update-example` — Despite the module docstring referencing "add, update, and show," no test in this file exercises `update` with an `--example` flag, leaving that path unverified here.
- `example-and-access-tags-are-orthogonal` — The `example` and `access_tags` fields are stored and retrieved independently in the metadata dict; setting one does not constrain or affect the other.
