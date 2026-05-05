# File: tests/test_llm.py

**Date:** 2026-05-05
**Time:** 15:30

I have the test file content and sufficient context from the test code itself. Here's the explanation:

---

## Purpose

`tests/test_llm.py` tests `reasons_lib.llm`, the shared LLM invocation layer. This module abstracts away the differences between CLI-based model backends (Claude, Gemini, Ollama) so the rest of the codebase can call `invoke_model(prompt, model="claude")` without caring about command-line argument formats. The test file validates two things: **command resolution** (translating a model spec string into the right CLI argv) and **subprocess orchestration** (binary existence checks, invocation, error handling, output post-processing).

## Key Components

### `TestResolveModelCmd`

Tests `resolve_model_cmd(model_spec) -> list[str]`, a pure function that parses a model specifier string and returns the subprocess argv.

The model spec grammar it enforces:

| Spec format | Example | Resolved command |
|---|---|---|
| `"claude"` | bare name | `["claude", "-p"]` |
| `"claude:<submodel>"` | `"claude:sonnet"` | `["claude", "-p", "--model", "sonnet"]` |
| `"gemini"` | bare name | `["gemini", "--skip-trust", "-p", ""]` |
| `"gemini:<submodel>"` | `"gemini:flash"` | `["gemini", "--skip-trust", "-m", "flash", "-p", ""]` |
| `"ollama:<model:tag>"` | `"ollama:gemma3:4b"` | `["ollama", "run", "gemma3:4b"]` |
| unknown | `"gpt-4"` | raises `ValueError` |

Notable: Ollama model specs can contain colons in the model name itself (e.g., `qwen3.5:27b`), so the parser must split on the *first* colon only and keep the rest as the model identifier.

### `TestInvokeModel`

Tests `invoke_model(prompt, model=..., timeout=...) -> str`, the function that actually shells out. Every test mocks `shutil.which` and `subprocess.run` — no real binaries are called.

Key behaviors tested:

- **Binary existence check**: If `shutil.which` returns `None`, raises `FileNotFoundError` before attempting to run anything.
- **Happy path**: Passes prompt as `input`, captures `stdout` as the return value, forwards `timeout`.
- **Non-zero exit**: Raises `RuntimeError` with the binary name in the message.
- **Ollama thinking-tag stripping**: For Ollama models, output between `Thinking...` and `...done thinking.` markers is stripped. This handles models like Qwen3 that emit chain-of-thought by default. If markers are incomplete (no end marker), output is left unchanged.
- **Claude thinking passthrough**: Claude output is *not* stripped, even if it contains the same marker text — the stripping is Ollama-specific.
- **Environment sanitization**: The `CLAUDECODE` env var is removed before spawning the subprocess, preventing recursive Claude Code invocations. Other env vars (like `HOME`) are preserved.

## Patterns

- **Full mock isolation**: Every `invoke_model` test patches both `shutil.which` and `subprocess.run` at the module level (`reasons_lib.llm.shutil.which`), so tests run without any LLM binaries installed.
- **Inline mock objects**: Uses `type("Result", (), {...})()` to create lightweight mock result objects instead of `MagicMock` — keeps the mock surface minimal and explicit.
- **Class-based test grouping**: Two test classes logically separate pure-function tests (`TestResolveModelCmd`) from side-effectful integration tests (`TestInvokeModel`). No shared fixtures between them.

## Dependencies

**Imports:**
- `reasons_lib.llm.invoke_model` — the subprocess orchestration function
- `reasons_lib.llm.resolve_model_cmd` — the model-spec-to-argv resolver
- `unittest.mock.patch` — for mocking `shutil.which`, `subprocess.run`, and `os.environ`
- `pytest` — for `pytest.raises`

**Imported by:** Nothing — this is a leaf test module.

## Flow

1. `resolve_model_cmd` tests are stateless: string in, list out, assert equality or exception.
2. `invoke_model` tests follow a consistent pattern:
   - Patch `shutil.which` to simulate binary presence/absence
   - Patch `subprocess.run` to return a crafted result object
   - Call `invoke_model` and assert on: return value, exception raised, or the args/kwargs passed to `subprocess.run`

## Invariants

- **Model spec validation is strict**: Only `claude`, `gemini`, and `ollama` prefixes are accepted. Anything else raises `ValueError`.
- **Binary must exist**: `invoke_model` checks PATH before attempting execution — fail-fast rather than letting the OS raise a cryptic error.
- **Ollama thinking stripping requires matched markers**: Both `Thinking...` and `...done thinking.` must be present; partial markers leave output unchanged (no data loss).
- **Environment isolation**: `CLAUDECODE` is always stripped from the child process environment, regardless of model.

## Error Handling

Three distinct error types map to three failure modes:

| Error | Condition | Raised by |
|---|---|---|
| `ValueError` | Unknown model prefix | `resolve_model_cmd` |
| `FileNotFoundError` | Binary not in PATH | `invoke_model` (before subprocess) |
| `RuntimeError` | Non-zero exit code | `invoke_model` (after subprocess) |

All errors include descriptive messages (model name or binary name) for debuggability. There is no retry logic — callers are expected to handle retries if needed.

## Topics to Explore

- [file] `reasons_lib/llm.py` — The implementation this file tests; see how `resolve_model_cmd` parses the colon-delimited spec and how thinking-tag stripping is implemented
- [function] `reasons_lib/derive.py:derive` — Primary consumer of `invoke_model`; shows how the LLM layer is used in belief derivation
- [file] `reasons_lib/ask.py` — Another `invoke_model` consumer; the interactive Q&A path
- [general] `ollama-thinking-strip` — Qwen3 and similar models emit `<think>` blocks; worth checking if the strip logic handles XML-style tags or just the plain-text markers tested here
- [file] `reasons_lib/review.py` — Uses multi-model invocation (Claude + Gemini); shows how the model-spec system enables model diversity

## Beliefs

- `llm-resolve-ollama-preserves-inner-colons` — `resolve_model_cmd("ollama:model:tag")` splits on the first colon only, keeping `model:tag` intact as the Ollama model identifier
- `llm-invoke-strips-claudecode-env` — `invoke_model` removes the `CLAUDECODE` environment variable from child processes to prevent recursive Claude Code invocation
- `llm-thinking-strip-ollama-only` — Thinking-marker stripping (`Thinking...` / `...done thinking.`) is applied only to Ollama models, not to Claude or Gemini
- `llm-invoke-checks-path-before-run` — `invoke_model` calls `shutil.which` and raises `FileNotFoundError` before attempting `subprocess.run`, ensuring fail-fast on missing binaries
- `llm-three-error-types` — The LLM module uses exactly three error types: `ValueError` for unknown models, `FileNotFoundError` for missing binaries, `RuntimeError` for non-zero exits

