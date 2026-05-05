# File: reasons_lib/llm.py

**Date:** 2026-05-05
**Time:** 15:29

# `reasons_lib/llm.py` — Shared LLM Invocation Layer

## Purpose

This file is the **single abstraction layer** between the reasons system and external LLM providers. Rather than calling LLMs via SDKs or HTTP APIs, it shells out to CLI tools (`claude`, `gemini`, `ollama`) as subprocesses, piping prompts to stdin and reading responses from stdout. Every module in the project that needs LLM output — `ask.py`, `review.py`, `api.py`, `cli.py` — goes through this file.

It owns two responsibilities: **resolving a model name to a CLI command** and **executing that command safely**.

## Key Components

### `MODEL_COMMANDS` (constant)

A dispatch table mapping short model names to their CLI argument lists. Only two entries: `"claude"` and `"gemini"`. Note the trailing empty string `""` in the gemini command — this is the placeholder for the prompt argument position that gemini expects after `-p`.

### `resolve_model_cmd(model: str) -> list[str]`

Pure function that translates a model specifier into a subprocess-ready command list. Handles four cases:

1. **Exact match** in `MODEL_COMMANDS` (e.g., `"claude"` → `["claude", "-p"]`)
2. **`claude:<submodel>`** — appends `--model <submodel>` (e.g., `claude:sonnet`)
3. **`gemini:<submodel>`** — uses `-m <submodel>` flag
4. **`ollama:<model>`** — uses `ollama run <model>`

Raises `ValueError` with available options on unknown input. This is the only validation gate — if your model string passes here, execution proceeds.

### `invoke_model(prompt: str, model: str = "claude", timeout: int = 300) -> str`

The workhorse. Takes a prompt string, resolves the model command, checks the binary exists in `$PATH`, runs the subprocess, and returns the raw stdout. Default timeout is 5 minutes.

## Patterns

**CLI-as-API**: Instead of importing provider SDKs, this treats CLI tools as the integration boundary. This keeps the Python dependency tree minimal and lets model providers handle auth, configuration, and API versioning in their own CLIs.

**Environment scrubbing**: The `CLAUDECODE` environment variable is explicitly stripped before invoking subprocesses (`env = {k: v for k, v in os.environ.items() if k != "CLAUDECODE"}`). This prevents recursive Claude Code invocations when `claude -p` is called from within a Claude Code session.

**Colon-delimited model syntax**: The `provider:model` convention (`claude:sonnet`, `ollama:gemma3:4b`) gives users a uniform way to specify any model without the caller needing to know provider-specific flag conventions.

## Dependencies

**Imports**: Only stdlib — `os`, `shutil`, `subprocess`. No third-party dependencies, which is the whole point of the subprocess approach.

**Imported by**: This is a leaf dependency used across the project. `ask.py` (interactive Q&A), `review.py` (code review), `api.py` (programmatic API), and `cli.py` (CLI commands) all call `invoke_model`. It's a high-fan-in, zero-fan-out module.

## Flow

1. Caller passes `(prompt, model)` to `invoke_model`
2. `resolve_model_cmd` maps model string → command list
3. `shutil.which` checks the binary exists; raises `FileNotFoundError` if not
4. `subprocess.run` executes with `input=prompt`, capturing stdout/stderr
5. Non-zero exit → `RuntimeError` with stderr
6. For ollama models only: strips thinking markers (`Thinking...\n` / `...done thinking.\n`) from output
7. Returns cleaned stdout string

## Invariants

- **Every LLM call is synchronous and blocking** — there's no async support, no streaming. The caller waits up to `timeout` seconds.
- **The model binary must exist in PATH** — checked before execution, not after failure.
- **The `CLAUDECODE` env var is never passed to child processes** — prevents infinite recursion.
- **Prompt is always piped via stdin** (`input=prompt`), never passed as a CLI argument — avoids shell injection and argument length limits.
- **Model resolution is fail-fast** — unknown model strings raise before any subprocess is spawned.

## Error Handling

Three distinct error types, each with a clear semantic:

| Condition | Exception | When |
|---|---|---|
| Binary not in PATH | `FileNotFoundError` | Before subprocess runs |
| Non-zero exit code | `RuntimeError` (includes stderr) | After subprocess completes |
| Subprocess hangs | `subprocess.TimeoutExpired` | After `timeout` seconds |

Errors are **never swallowed** — all three propagate to the caller. The comment on the ollama thinking-marker stripping explicitly flags it as fragile, acknowledging a coupling to ollama's output format that may break across versions.

---

## Topics to Explore

- [file] `reasons_lib/ask.py` — Primary consumer: see how prompts are constructed and responses parsed after `invoke_model` returns
- [file] `tests/test_llm.py` — Shows how the subprocess calls are mocked and what edge cases are covered (timeout, missing binary, stderr)
- [function] `reasons_lib/review.py:invoke_model` — Multi-model code review uses this to run the same prompt through claude and gemini for comparison
- [general] `ollama-thinking-markers` — The fragile output stripping at the end of `invoke_model` — worth tracking whether ollama has stabilized this format
- [file] `reasons_lib/cli.py` — Where the `model` argument is parsed from user input before reaching `resolve_model_cmd`

## Beliefs

- `llm-invoke-is-subprocess-only` — All LLM calls go through `subprocess.run`; there are no SDK imports or HTTP calls to any provider API
- `claudecode-env-stripped-from-child` — The `CLAUDECODE` environment variable is explicitly removed before spawning any LLM subprocess to prevent recursive Claude Code invocation
- `llm-prompt-always-via-stdin` — Prompts are passed via `subprocess.run(input=...)`, never as command-line arguments
- `llm-module-stdlib-only` — `llm.py` has zero third-party dependencies; it imports only `os`, `shutil`, and `subprocess`
- `ollama-thinking-strip-is-fragile` — The ollama output post-processing relies on exact string markers (`Thinking...\n` / `...done thinking.\n`) that may change across ollama versions

