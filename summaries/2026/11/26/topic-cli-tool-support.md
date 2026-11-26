---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Investigate whether `claude` and `gemini` CLIs expose `--tools`/`--mcp` flags; the observations show none, which determines whether CLI backends can ever be native or must remain emulated.

## What We Can Determine

Short answer: **The codebase does not use `--tools` or `--mcp` flags with the `claude` or `gemini` CLIs.** The CLI backends are wired as pure stdin-prompt → stdout-response subprocesses, and MCP tooling is implemented separately in Python. That locks in an **emulated** integration model rather than a native one — *subject to one caveat about the search observations (see below)*.

## Evidence from the Code

### 1. Fixed command shapes with no tool/MCP flags

`reasons/llm.py` defines `MODEL_COMMANDS` (near line 39) as:

```python
MODEL_COMMANDS = {
    "claude": ["claude", "-p", "--output-format", "json"],
    "gemini": ["gemini", "--skip-trust", "-o", "json", "-p", ""],
}
```

And `resolve_model_cmd()` (starting around line 152) builds sub-model variants:

```python
if model.startswith("claude:"):
    return ["claude", "-p", "--model", submodel, "--output-format", "json"]
if model.startswith("gemini:"):
    return ["gemini", "--skip-trust", "-m", submodel, "-o", "json", "-p", ""]
```

Neither branch permits injection of `--tools`, `--mcp`, or any equivalent. The tests in `tests/test_llm.py` (`TestResolveModelCmd`) cement these exact argument lists, so any future addition would be an intentional, test-visible change.

### 2. The invocation model is text-in / text-out

`invoke_model()` (around line 265 in `reasons/llm.py`) does:

```python
result = subprocess.run(
    cmd,
    input=prompt,
    capture_output=True,
    text=True,
    timeout=timeout,
    env=env,
)
```

The prompt is piped to **stdin**; the response is read from **stdout**; JSON output is unwrapped by `_parse_cli_json()` (around line 96). The function's docstring says: *"For CLI models, uses `--output-format json` to capture token/cost data."* There is no channel for tool definitions or MCP server URLs to reach the CLI — the interface is one prompt in, one response out.

### 3. MCP is implemented in Python, not delegated to the CLI

`reasons/mcp_client.py` (132 lines) contains `McpBridge` — a synchronous bridge over the Python MCP SDK (`mcp.ClientSession`, `StdioServerParameters`) that:

- Spawns an event loop in a background thread (`connect`, `_run_loop`).
- Lists tools via `session.list_tools()` and exposes them through `list_tools()`.
- Dispatches tool calls via `call_tool()` → `session.call_tool(...)` on the Python side.

This is the **emulation layer**. MCP servers are connected and tool calls are dispatched by the Python process, *not* by shelling out to `claude --mcp ...` or `gemini --tools ...`. The two modules are independent: `reasons/llm.py` never imports `reasons/mcp_client.py`.

## Why the "0 matches" Search Results Are Misleading

The prompt says "the observations show none," referring to `locate_claude_gemini_backends` and `find_tools_mcp_native_emulation_logic` returning `total_count: 0`. **These search results contradict the file reads**:

- `reasons/llm.py` contains "claude" and "gemini" dozens of times.
- `reasons/mcp_client.py` contains "mcp" throughout.

So the searches themselves failed (likely a scoping/glob/regex issue), not that the tokens are absent. The **file reads are the reliable evidence**; the search summaries are not. The conclusion "no `--tools`/`--mcp` flags are used" holds because of what the command lists actually contain, not because of the empty search results.

## What the Observations *Do Not* Tell Us

We have no evidence about what the `claude` and `gemini` binary CLIs themselves accept. There is no captured `--help` output, man page, or changelog in the observations. So:

- **Safe claim:** *This codebase does not invoke those flags, and the architecture treats the CLI as a text-completions endpoint.*
- **Unsafe claim:** *The CLIs themselves lack those flags.* To make that claim, you'd need to inspect `claude --help` / `gemini --help` or vendor docs.

## Conclusion

Within this repository, **CLI backends are emulated by design**: `reasons/llm.py` drives them as pipes of text, and `reasons/mcp_client.py` layers tool-calling on top in Python. For CLI backends to become "native," two changes would be required:

1. `MODEL_COMMANDS` / `resolve_model_cmd` would need to accept and forward tool/MCP configuration flags (and the corresponding tests in `tests/test_llm.py` updated).
2. `invoke_model` would need a channel to surface tool-call requests/results from CLI JSON back into the calling loop, rather than returning only `data["result"]` / `data["response"]`.

Until then, the emulation path (`McpBridge`) is the only supported way to give these backends tools.

## Topics to Explore

- [file] `reasons/ask.py` — Likely the caller that stitches `McpBridge.list_tools()` and `invoke_model()` together into a tool-using loop; confirms whether the emulation is prompt-based or JSON-based.
- [function] `reasons/llm.py:resolve_model_cmd` — The single chokepoint for adding CLI flags; any native `--tools`/`--mcp` support would be introduced here.
- [function] `reasons/llm.py:_parse_cli_json` — Only extracts `result`/`response`; would need extension to surface tool calls if a CLI grew native tool-output blocks.
- [file] `tests/test_llm.py` — The exact expected argv lists in `TestResolveModelCmd` act as a contract test for "no tool flags."
- [general] `cli-vendor-capabilities` — Run `claude --help` and `gemini --help` (or inspect release notes) to confirm whether the underlying binaries expose tool/MCP flags at all; the repo observations alone cannot answer this.

## Beliefs

- `cli-command-lists-omit-tool-flags` — Neither `MODEL_COMMANDS` nor any branch of `resolve_model_cmd` includes `--tools` or `--mcp`.
- `invoke-model-uses-stdin-stdout-text-piping` — `invoke_model` calls `subprocess.run` with `input=prompt` and parses `stdout`, giving no channel for tool definitions to reach the CLI.
- `mcp-client-implements-mcp-in-python` — `reasons/mcp_client.py:McpBridge` uses the Python `mcp` SDK and never shells out to `claude --mcp` / `gemini --tools`.
- `llm-module-does-not-import-mcp-client` — `reasons/llm.py` and `reasons/mcp_client.py` are decoupled; the CLI path has no MCP awareness.
- `parse-cli-json-returns-only-text` — `_parse_cli_json` extracts `result` (Claude) or `response` (Gemini) strings and discards any hypothetical tool-call structure.
- `tests-pin-cli-argv-contract` — `tests/test_llm.py:TestResolveModelCmd` asserts the exact CLI argv, so any native-flag addition requires test changes.
