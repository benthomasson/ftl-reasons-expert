---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How the `reasons-mcp` console script is declared and dispatches to this `main`

## How `reasons-mcp` is declared and dispatched

### 1. The declaration (PEP 621 / setuptools entry point)

`pyproject.toml` line 28 declares the console script inside the standard `[project.scripts]` table:

```toml
[project.scripts]
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

The value uses the entry-point notation `module.path:callable`. So `reasons-mcp` maps to the attribute `main` in the module `reasons.mcp_server`. The package is built with `setuptools>=68.0` (line 2) and installs both top-level packages (line 33: `packages = ["reasons", "reasons_lib"]`), which makes `reasons.mcp_server` importable at install time.

### 2. The generated launcher

At `pip install` / wheel-install time, setuptools turns the entry point into a small executable shim (in `<venv>/bin/reasons-mcp` on Unix; a `*-script.py` wrapper on Windows). The shim is effectively:

```python
from reasons.mcp_server import main
sys.exit(main())
```

The script does **not** contain the server code — it only imports the module and calls `main()`. This is why `reasons_lib/mcp_server.py` existing as a 3-line shim (`from reasons.mcp_server import *`) does not affect the console script: the entry point targets `reasons.mcp_server`, not `reasons_lib.mcp_server`.

### 3. What happens when the module is imported

Because the shim imports `reasons.mcp_server`, the whole module body runs before `main()` is called. In `reasons/mcp_server.py` (lines 1–180 shown), that means:

1. **Dependency guard** (roughly lines 15–20): `try: from mcp.server.fastmcp import FastMCP` / `except ImportError:` re-raises with the hint `"Install with: pip install 'ftl-reasons[mcp]'"`. So the `mcp` extra declared at `pyproject.toml` line 19 is a hard runtime requirement of this script.
2. **Server singleton**: `mcp = FastMCP("reasons")` (around line 24).
3. **State**: global `_db: str | None = None`, plus helpers `_find_db()` and `_get_db()`. `_find_db` prefers `REASONS_DB` from the environment, otherwise walks upward from `os.getcwd()` looking for `reasons.db`, otherwise returns `api.DEFAULT_DB`. `_get_db` lazily resolves and memoizes the path.
4. **Tool registration**: every `@mcp.tool()`-decorated function (`search`, `show`, `explain`, `list_beliefs`, `add`, `retract`, `assert_belief`, `what_if`, …) registers itself on the `FastMCP` instance at import time. All of them call `_get_db()` so the DB path is resolved lazily, not at boot.

After the module finishes importing, the shim calls `main()`, which presumably starts the FastMCP server (typically `mcp.run()`).

### 4. Gap in the observations — the body of `main` was not seen

The observations do **not** show the definition of `main` itself:

- The `read_reasons_mcp_server_start` excerpt only covers lines 1–180 of a 600-line file, and no `main` appears in that range.
- The `locate_main_in_reasons_mcp_server` search (pattern `^def main|^async def main|def main\(`) returned **zero matches**, and the same search over `reasons_lib/mcp_server.py` also returned zero (expected, since that file just star-imports).

Two possibilities remain, and the observations cannot distinguish them:

- `main` is defined later in the 600-line file (e.g., a bottom-of-file `def main() -> None: mcp.run()`), and the grep somehow missed it — unlikely given `def main\(` is unanchored, so this deserves verification.
- The declaration `reasons.mcp_server:main` is broken at runtime (`AttributeError: module 'reasons.mcp_server' has no attribute 'main'`), or `main` is bound by some other mechanism (e.g., `main = mcp.run`).

Either way, the **declaration → import → attribute lookup → call** chain described above is exactly how the entry point resolves; what's unverified is only the concrete body bound to the name `main`.

## Topics to Explore
- [file] `reasons/mcp_server.py` — Read the tail of the 600-line file to confirm how `main` is defined (or if it's missing) and whether it calls `mcp.run()`.
- [function] `reasons/mcp_server.py:_find_db` — Auto-discovery of `reasons.db` (env var → upward walk → `api.DEFAULT_DB`) is the only real "configuration" this script exposes.
- [general] `setuptools-entry-points` — How `[project.scripts]` is materialized into platform-specific launcher shims at install time.
- [file] `reasons_lib/mcp_server.py` — Backwards-compat star-import shim; worth confirming what symbols it actually re-exports (including whether `main` survives the star import).
- [file] `pyproject.toml` — Compare `reasons` → `reasons.cli:main` and `reasons-mcp` → `reasons.mcp_server:main`, plus the `mcp` optional extra at line 19.

## Beliefs
- `pyproject-declares-reasons-mcp-entry-point` — `pyproject.toml` line 28 declares `reasons-mcp = "reasons.mcp_server:main"` under `[project.scripts]`.
- `reasons-mcp-requires-mcp-extra` — Importing `reasons.mcp_server` raises `ImportError` with the hint `pip install 'ftl-reasons[mcp]'` when the `mcp` package is absent.
- `mcp-server-instantiates-fastmcp-singleton` — `reasons/mcp_server.py` creates a module-level `FastMCP("reasons")` instance that every `@mcp.tool()` function registers against.
- `reasons-mcp-resolves-db-via-REASONS_DB-or-discovery` — `_find_db` uses `REASONS_DB` if set, otherwise walks up from `os.getcwd()` for `reasons.db`, otherwise falls back to `api.DEFAULT_DB`.
- `reasons_lib-mcp-server-re-exports-reasons-mcp-server` — `reasons_lib/mcp_server.py` is a 3-line shim that does `from reasons.mcp_server import *`; the console script does not target it.
- `reasons-mcp-main-definition-unobserved` — The `main` callable targeted by the entry point is not present in lines 1–180 and the grep for `def main(` returned zero matches; its actual definition and behavior are unverified.
