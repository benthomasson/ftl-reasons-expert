---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search the whole repo (including downstream projects) for live `reasons_lib` imports to know when the shim can be deleted.

## Summary

The repo still **ships** `reasons_lib` as a backwards-compatibility package (shim), and the question is: **when is it safe to delete?** The answer requires two searches: (1) live imports inside this repo, and (2) any downstream project that still depends on it. The observations strongly suggest the in-repo answer is "none", but they are **not sufficient to justify deletion** as-is.

## What the observations do establish

- `reasons_lib/__init__.py` (3 lines) is a pure shim:
  ```python
  """Backwards-compatibility shim -- use ``import reasons`` instead."""
  from reasons import *  # noqa: F401,F403
  ```
- `pyproject.toml:31` still lists both packages: `packages = ["reasons", "reasons_lib"]`, so `pip install ftl-reasons` continues to install the shim.
- `CLAUDE.md` (the "Project Structure" block) documents it as *"reasons_lib/ # backwards-compat shim — re-exports from reasons/"* — informational, no stated end-of-life.
- `CHANGELOG.md` records the naming origin: **0.3.0** renamed `rms → reasons`; **0.1.0** mentions the API layer as `reasons_lib.api`. No entry announces a deprecation window for `reasons_lib`.
- The two "live import" searches returned **0** matches:
  - `find_all_reasons_lib_references` — pattern `reasons_lib`, 0 matches
  - `find_live_python_reasons_lib_imports` — pattern `^\s*(from\s+reasons_lib|import\s+reasons_lib)`, 0 matches

## Why that's not yet enough to delete

Four issues must be closed before deleting the shim:

1. **The "all references" search is suspicious.** Pattern `reasons_lib` returned 0 matches, yet `pyproject.toml:31` literally contains `"reasons_lib"`, and the directory listing shows a whole `reasons_lib/` tree. The search scope was almost certainly limited to a glob that excluded `.toml` and maybe path names, or was truncated. Do not trust `total_count: 0` from that search.

2. **`reasons_lib/` is not a single file.** The directory listing shows ~30 parallel `.py` files under `reasons_lib/`: `api.py`, `ask.py`, `build_wiki.py`, `check_stale.py`, `cli.py`, `cluster.py`, `compact.py`, `contradictions.py`, `derive.py`, `export_card.py`, `export_markdown.py`, `hf.py`, `import_agent.py`, `import_beliefs.py`, `llm.py`, `mcp_client.py`, `mcp_server.py`, `metadata.py`, `mind_service.py`, `network.py`, `pg.py`, `propose_update.py`, `pubsub.py`, `repair.py`, `repair_premises.py`, `report_belief.py`, `report_gated.py`, `review.py`, `review_justifications.py`, `review_premises.py`, `storage.py`, `verify.py`. Deleting the shim means deleting that entire directory and removing `"reasons_lib"` from `packaging.toml`'s `packages` list. Only `__init__.py` was inspected, so we don't know what the submodule shims look like (presumably `from reasons.api import *` etc.).

3. **The regex misses dynamic imports.** `^\s*(from\s+reasons_lib|import\s+reasons_lib)` catches `from reasons_lib.api import X` and `import reasons_lib.api`, but **not**:
   - `importlib.import_module("reasons_lib.api")`
   - `__import__("reasons_lib")`
   - String-based references in entry points or config

4. **"Downstream projects" are not actually visible.** `list_downstream_project_directories` only lists the repo root (`.`) with `costs/`, `reasons/`, `reasons_lib/`, `tests/`. If "downstream" means separate repos/consumers of the PyPI package `ftl-reasons`, they must be searched separately (GitHub org search, dependency graph, or a shim that raises `ImportError` for a release and see if telemetry / issues fire). If downstream just means intra-repo subpackages, then `tests/` is the only other consumer and it has none.

## Recommended search procedure

Run all of these from the repo root before deciding:

```bash
# 1. Any import of reasons_lib (top-level or submodule) in Python
rg -n --glob '*.py' '^\s*(from|import)\s+reasons_lib(\b|\.)'

# 2. Any textual mention outside the shim itself, excluding generated artifacts
rg -n 'reasons_lib' \
  --glob '!reasons_lib/**' \
  --glob '!*.lock' \
  --glob '!uv.lock' \
  --glob '!costs/**'

# 3. Dynamic imports
rg -n 'import_module\(["'\'']reasons_lib|__import__\(["'\'']reasons_lib' --glob '*.py'

# 4. Packaging/entry points
rg -n 'reasons_lib' pyproject.toml setup.cfg setup.py 2>/dev/null

# 5. Verify the submodule shims themselves (what do they re-export?)
for f in reasons_lib/*.py; do echo "== $f"; head -5 "$f"; done
```

Then repeat #1 and #3 in each downstream repository (or via `gh search code "reasons_lib" --owner=<org>`).

## Verdict criteria

Delete `reasons_lib/` (and remove `packaging.toml:31` + update `CLAUDE.md` and `CHANGELOG.md`) only once **all** of the following hold:

- [ ] No `.py` file outside `reasons_lib/` imports `reasons_lib` (static or dynamic).
- [ ] No non-`reasons_lib` file mentions `"reasons_lib"` except `pyproject.toml` and docs you're updating.
- [ ] Every submodule shim under `reasons_lib/*.py` is a pure re-export (no logic that `reasons` doesn't already expose).
- [ ] A deprecation release has shipped (e.g., the shim raises a `DeprecationWarning`) long enough for downstream consumers on PyPI to migrate.
- [ ] Downstream repo search (or PyPI telemetry/issues) confirms zero remaining users.

Given the observations, **the in-repo cleanup is essentially ready** — the only blockers are proving the search scope actually covered `pyproject.toml` and confirming the ~30 submodule shims are pure re-exports.

## Topics to Explore

- [file] `reasons_lib/api.py` — Only `__init__.py` was inspected; confirm the submodule shims are thin re-exports (e.g., `from reasons.api import *`) rather than carrying logic.
- [file] `pyproject.toml` — Line 31 (`packages = ["reasons", "reasons_lib"]`) must be updated in the same deletion PR, and `[project.scripts]` (lines 27–28) should be checked for any `reasons_lib`-based entry point.
- [general] downstream-pypi-consumers — The observations only cover this repo; verify external users of `ftl-reasons`'s `reasons_lib` package (GitHub code search, PyPI download telemetry, or a deprecation warning release).
- [file] `CHANGELOG.md` — No documented deprecation/EOL for `reasons_lib` despite it originating in 0.1.0; a proper deletion should add a deprecation entry before removal.
- [diff] remove-reasons-lib-shim — A hypothetical PR deleting `reasons_lib/`, updating `pyproject.toml:31`, `CLAUDE.md` (Project Structure block), and `CHANGELOG.md` is the concrete change this investigation gates.

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — `reasons_lib/__init__.py` (3 lines) consists only of a docstring directing users to `import reasons` and a `from reasons import *` re-export with `# noqa: F401,F403`.
- `reasons-lib-is-still-packaged` — `pyproject.toml:31` ships both `"reasons"` and `"reasons_lib"` in `[tool.setuptools] packages`, so the shim is still installed for every `pip install ftl-reasons`.
- `reasons-lib-contains-submodule-shims` — The repo root listing shows ~30 parallel `.py` files under `reasons_lib/` (api, ask, storage, network, …), so deletion must cover the whole directory, not just `__init__.py`.
- `no-static-reasons-lib-imports-observed` — The `^\s*(from\s+reasons_lib|import\s+reasons_lib)` search returned 0 matches, indicating no live static imports inside the searched scope.
- `reasons-lib-reference-search-is-incomplete` — The `reasons_lib` literal search returned 0 matches despite `pyproject.toml:31` containing that string, so the search scope must be widened (or was truncated) before drawing conclusions.
- `reasons-lib-deletion-requires-deprecation-window` — `CHANGELOG.md` has no entry announcing `reasons_lib` deprecation or removal; deletion should be preceded by a deprecation release to avoid breaking silent downstream users.
