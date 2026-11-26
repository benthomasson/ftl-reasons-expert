---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How `[project.scripts]` is materialized into platform-specific launcher shims at install time.

# How `[project.scripts]` becomes executable shims

The short answer: **it doesn't happen in this repo.** The repo only *declares* two console-script mappings in `pyproject.toml`. The actual platform-specific shim files are generated later by the build backend (`setuptools`) during `pip install` / `uv install`, and then by the installer itself. The observations only cover the declaration side and the Python target functions—not the generator.

Below is the full pipeline, with what the observations do and don't tell us.

## 1. The declaration side (in this repo)

`pyproject.toml` line 26–28 is the only place the mappings exist:

```toml
[project.scripts]                       # line 26
reasons     = "reasons.cli:main"        # line 27
reasons-mcp = "reasons.mcp_server:main" # line 28
```

The build backend is declared at `pyproject.toml` lines 1–4:

```toml
[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"
```

Because `setuptools.build_meta` is the backend, **PEP 621 `[project.scripts]` is setuptools' responsibility** — no `setup.py` or `setup.cfg` is present to override it, so all console-script metadata flows from these three lines.

Two key observations support this:

- `find_entry_point_config` finds exactly one match in the whole source tree: `pyproject.toml:26 [project.scripts]`.
- `find_script_target_usages` searching for `reasons\.cli:main|console_scripts|entry_points` returns **zero matches** in tracked source. The mapping string never appears in Python code, so nothing in the repo implements shim generation.

The `reasons_lib/*.py` files that the "shim" search hits (e.g. `reasons_lib/__init__.py:1`) are **Python import shims** (backwards-compat re-exports), not install-time launcher shims. They are unrelated to the entry-point pipeline and worth keeping distinct in your head.

## 2. Build-time: setuptools writes `entry_points.txt`

When `pip install .` (or a wheel build) runs, setuptools performs roughly:

1. Parses `pyproject.toml` (`setuptools/config/_apply_pyprojecttoml.py` in recent versions), which maps PEP 621 `project.scripts` onto the legacy `entry_points["console_scripts"]` slot.
2. The `dist_info` / `egg_info` command serializes these into an INI file inside the distribution metadata directory:

   ```
   ftl_reasons-0.58.0.dist-info/entry_points.txt
   ```

   with contents equivalent to:

   ```ini
   [console_scripts]
   reasons = reasons.cli:main
   reasons-mcp = reasons.mcp_server:main
   ```

   (setuptools normalizes the distribution name `ftl-reasons` → `ftl_reasons` for the dist-info directory.)

Nothing in the wheel is a shim yet—it's just a text file describing *"when installing this wheel, create a `reasons` launcher that calls `reasons.cli:main`."*

## 3. Install-time: pip/uv materialize per-platform shims

The installer scans `*.dist-info/entry_points.txt` for `[console_scripts]` and, for each entry, writes an executable into the environment's scripts directory:

- POSIX: `<venv>/bin/reasons`
- Windows: `<venv>\Scripts\reasons.exe` (plus a small companion script)

### POSIX layout

The shim is a small text executable with a hardcoded shebang pointing at the environment interpreter, and a body equivalent to:

```python
#!/path/to/venv/bin/python
# -*- coding: utf-8 -*-
import re
import sys
from reasons.cli import main
if __name__ == "__main__":
    sys.argv[0] = re.sub(r"(-script\.pyw|\.exe)?$", "", sys.argv[0])
    sys.exit(main())
```

That is exactly why `tests/test_cli.py:9` imports `main` and calls it directly inside `run_cli()` (line 17–25): the test exercises the same callable the generated shim uses, without going through the launcher. The `main()` contract—`reasons/cli.py`, later in the file—is "read `sys.argv`, dispatch via `argparse`."

### Windows layout

Windows has no shebang interpreter convention, so the installer writes **binary shim executables**. Historically `distlib.scripts.ScriptMaker` (vendored as `pip._vendor.distlib`) produces a pair:

- `reasons.exe` — a tiny launcher stub (`t32.exe` / `t64.exe` / `w32.exe` / `w64.exe` template) with the target interpreter path and `-script.py` filename embedded.
- `reasons-script.py` — a Python script containing the real `from reasons.cli import main; sys.exit(main())`.

When `reasons` is invoked on Windows, `reasons.exe` reads its embedded metadata, locates the correct Python, and runs `reasons-script.py`. Modern pip (≥ 20.x) no longer emits `.exe.manifest` files; it uses a single `.exe` + `-script.py` pair produced by `pip/_internal/operations/install/wheel.py` (which subclasses distlib's `ScriptMaker`).

### What about `uv`?

The README (the install section around lines 24–34) advertises three install paths: `pip install ftl-reasons`, `uv tool install ftl-reasons`, and `uvx ftl-reasons <command>`. All three consume the **same** `entry_points.txt` produced by setuptools, but `uv`'s launcher generator is distinct from pip's:

- `uv` uses its own Rust crate (`uv-install-wheel`) to write shims.
- On POSIX it writes a shell/Python wrapper that hardcodes the tool's venv interpreter.
- On Windows, newer `uv` emits native Rust launchers rather than distlib `.exe` stubs.

So the same declarative metadata fans out into pip-POSIX, pip-Windows, uv-POSIX, and uv-Windows shims depending on which installer and version you use.

## 4. Runtime: from shim to behavior

Whichever shim is produced, the call chain is:

```
shim (bin/reasons | Scripts\reasons.exe)
  └─> import reasons.cli; reasons.cli.main()
        └─> argparse parse; dispatch to cmd_* wrappers
              └─> reasons.api.<function>(...)   # reasons/cli.py:12 imports api
```

`reasons/cli.py` is explicitly described in its module docstring as "Thin wrappers around reasons.api" (lines 2–3), so the shim lands on argument parsing, and business logic lives behind `api`.

## 5. What is missing from the observations

To fully trace this without inference, you would need:

- The built wheel or `*.dist-info/entry_points.txt` artifact (to confirm the serialized form).
- The `reasons/mcp_server.py` source, since its `main` is referenced at `pyproject.toml:28` but never shown.
- `reasons/cli.py:main` definition (it sits beyond the first 120 lines returned).
- The installed `setuptools`, `pip`, and `uv` **versions**, because they determine whether you get distlib `.exe` stubs vs native `uv` launchers, and whether pip uses `-script.py` vs the newer single-file `.exe` representation.
- Any `setup.py`/`setup.cfg`/`MANIFEST.in`—none were observed, so `pyproject.toml` is currently the sole source of entry-point truth.

## Topics to Explore

- [file] `pyproject.toml` — Confirms that `[project.scripts]` at lines 26–28 plus `setuptools.build_meta` at lines 1–4 are the only entry-point declarations in the source tree.
- [function] `reasons/cli.py:main` — The function every generated `reasons` shim ultimately calls; verify the `sys.argv` contract and argument parsing.
- [general] `entry-points-txt-serialization` — The INI file setuptools writes into `*.dist-info/entry_points.txt`; inspect it in a built wheel to see the raw metadata the installer consumes.
- [general] `pip-scriptmaker-shims` — The `pip/_internal/operations/install/wheel.py` → `distlib.scripts.ScriptMaker` path that produces POSIX text shims and Windows `.exe` + `-script.py` pairs.
- [general] `uv-vs-pip-launchers` — How `uv tool install` / `uvx` generate launchers differently from pip, even though both read the same `entry_points.txt`.

## Beliefs

- `project-scripts-declares-console-entry-points` — `pyproject.toml:26-28` is the single source of the `reasons` and `reasons-mcp` console-script mappings.
- `setuptools-build-meta-generates-entry-points-txt` — With `build-backend = "setuptools.build_meta"` (`pyproject.toml:4`), the `[console_scripts]` group in `*.dist-info/entry_points.txt` is produced by setuptools, not by any code in this repo.
- `pyproject-scripts-requires-an-installer` — The declaration has no effect until a wheel is built and an installer (pip or uv) reads `entry_points.txt`; the repo contains no shim-generation code (`find_script_target_usages` returns 0 matches).
- `reasons-cli-main-is-the-runtime-target-of-the-shim` — `tests/test_cli.py:9` imports `from reasons.cli import main` and `run_cli()` (lines 17–25) calls it directly, demonstrating the same callable the generated `reasons` shim invokes.
- `reasons-lib-shims-are-unrelated-to-entry-point-launchers` — The `reasons_lib/*.py` "shim" files are Python import re-exports, not install-time launcher shims, despite matching the `shim` search pattern.
