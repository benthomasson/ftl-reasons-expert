---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: No tests currently assert the shim’s export contract; add tests around `__all__` and wildcard imports if this API matters.

## Explanation

### What the shim does

`reasons_lib/__init__.py` is a 3-line backwards-compatibility package:

- Line 1 — module docstring pointing callers at `reasons`.
- Line 2 — `from reasons import *  # noqa: F401,F403`.

The `# noqa` comment suppresses the linter warnings about unused imports and star imports; it does **not** pin the set of names being re-exported. Whatever `from reasons import *` yields today is what `reasons_lib` yields today, and that can silently change if `reasons/__init__.py` changes.

### The current export surface (implicit)

`reasons/__init__.py` has no `__all__` anywhere — the repository-wide search for the literal `__all__` returns **0 matches**. So Python falls back to its default wildcard rule: every module-level name not starting with `_`.

By reading `reasons/__init__.py`:

- Intentional public classes: `Justification` (line ~7), `Node` (line ~28), `Nogood` (line ~53).
- Leaked-on-accident names: `dataclass` and `field` from `from dataclasses import dataclass, field` (line 3).

That means today:

```python
>>> from reasons_lib import *
>>> [n for n in dir() if not n.startswith("_")]
['Justification', 'Node', 'Nogood', 'dataclass', 'field']
```

`dataclass` and `field` are leaking into the shim's namespace — nobody chose that, it just falls out of the wildcard.

### 33 shim submodules, all similarly blind

The `find_shim_mentions` result shows **33** `reasons_lib/*.py` files, one per submodule (e.g. `reasons_lib/api.py`, `reasons_lib/cli.py`, `reasons_lib/network.py`, …). Each ends up doing `from reasons.<sub> import *` with no explicit `__all__` on either side. None of these contracts are pinned either.

### Why this is risky *now*

- `pyproject.toml` line 33 lists `packages = ["reasons", "reasons_lib"]`, so **the shim ships as a first-class package in the wheel**. It is part of the installed public surface, not a dev-only convenience.
- `find_reasons_lib_usages` returns **0 matches** and `find_reasons_lib_wildcard_imports` returns **0 matches** inside the repo — nothing internal uses the shim. Its only consumers are external (downstream users holding `from reasons_lib import Node`). If someone removes or renames a class in `reasons/__init__.py`, no test in `tests/` (see `tests/test_api.py`, which imports only `from reasons import api`) will fail; only end users will.
- Because there is no `__all__` on either side, refactoring an internal import in `reasons/__init__.py` (e.g. changing `from dataclasses import dataclass, field` to `import dataclasses`) silently *changes the wildcard export set* of `reasons_lib` — flipping `dataclass`/`field` off without anyone deciding that.

### What a test suite would look like

There are two complementary contracts to lock down:

1. **A canonical `__all__`** on `reasons/__init__.py` (e.g. `["Justification", "Node", "Nogood"]`) — this is the choice point that decides whether `dataclass`/`field` should leak. Once set, it can be asserted.
2. **Equality tests on the shim** — that `reasons_lib`'s wildcard set and `reasons`'s wildcard set are identical, and that they contain exactly the intended names.

Sketch (new `tests/test_shim_contract.py`):

```python
import importlib

import pytest

EXPECTED_TOP_LEVEL = {"Justification", "Node", "Nogood"}


def _wildcard_names(module_name: str) -> set[str]:
    ns: dict = {}
    exec(f"from {module_name} import *", ns)  # noqa: S102
    return {k for k in ns if not k.startswith("_")}


def test_reasons_defines_all():
    reasons = importlib.import_module("reasons")
    assert hasattr(reasons, "__all__"), "reasons/__init__.py must define __all__"
    assert set(reasons.__all__) == EXPECTED_TOP_LEVEL


def test_reasons_lib_wildcard_matches_reasons():
    assert _wildcard_names("reasons_lib") == _wildcard_names("reasons")


def test_reasons_lib_exports_expected_top_level():
    assert _wildcard_names("reasons_lib") == EXPECTED_TOP_LEVEL


SHIM_SUBMODULES = [
    "api", "ask", "build_wiki", "check_stale", "cli", "cluster",
    "compact", "contradictions", "derive", "export_card",
    "export_markdown", "hf", "import_agent", "import_beliefs", "llm",
    "mcp_client", "mcp_server", "metadata", "mind_service", "network",
    "pg", "propose_update", "pubsub", "repair", "repair_premises",
    "report_belief", "report_gated", "review", "review_justifications",
    # …one entry per shim file listed in find_shim_mentions
]


@pytest.mark.parametrize("name", SHIM_SUBMODULES)
def test_submodule_shim_reexports_same_names(name):
    real = importlib.import_module(f"reasons.{name}")
    shim = importlib.import_module(f"reasons_lib.{name}")
    real_public = {n for n in dir(real) if not n.startswith("_")}
    shim_public = {n for n in dir(shim) if not n.startswith("_")}
    assert shim_public == real_public
```

Key benefits of pinning this:

- Any rename / removal in `reasons/**` that would break `from reasons_lib import X` fails CI immediately.
- Any accidental export (e.g. `dataclass`/`field`) is either explicitly allowed or explicitly blocked by the `__all__` you choose.
- The `# noqa: F401,F403` in `reasons_lib/__init__.py` becomes a documented decision, not a code smell — the tests are the documentation of *why* the wildcard is safe.

### What's missing from the observations

- The exact contents of `reasons_lib/api.py`, `reasons_lib/cli.py`, etc. weren't read — so we don't know whether any submodule shim *already* defines `__all__` locally, or whether they all follow the same one-line pattern.
- `tests/test_api.py` only shows the first 200 lines; there may be other test files under `tests/` that indirectly exercise `reasons_lib` via CLI entry points (`reasons = "reasons.cli:main"` in `pyproject.toml`), which wouldn't appear in the `reasons_lib` text search.
- There's no aggregate list of shim file line counts — but from `find_shim_mentions` we can infer all 33 are single-line shims with a docstring + wildcard import.

## Topics to Explore

- [file] `reasons_lib/api.py` — Confirm whether submodule shims also use bare `from reasons.api import *` or define `__all__`; the top-level test only covers one file.
- [function] `reasons/__init__.py:Justification` / `Node` / `Nogood` — These are the three names that should appear in any explicit `__all__`; verify whether any other public symbol was intended to be re-exported.
- [file] `pyproject.toml` — Line 33 (`packages = ["reasons", "reasons_lib"]`) ships the shim; check whether published wheels already rely on this contract for real users.
- [general] `backwards-compat-policy` — Decide whether `dataclass`/`field` leaking through `from reasons_lib import *` is acceptable; that decision drives the expected `__all__`.
- [diff] `reasons-init-imports` — Track how the `from dataclasses import dataclass, field` line in `reasons/__init__.py` changed over time; any change there silently mutates the shim's wildcard export set.

## Beliefs

- `reasons-lib-is-a-wildcard-shim` — `reasons_lib/__init__.py` line 2 is exactly `from reasons import *  # noqa: F401,F403`; it has no explicit `__all__`.
- `reasons-has-no-all` — No file in the repository defines `__all__` (search returns 0 matches), so wildcard exports are inferred by Python's default rule.
- `reasons-exports-dataclass-and-field` — `reasons/__init__.py` imports `dataclass` and `field` at module level, so both leak through `from reasons_lib import *` today.
- `pyproject-ships-the-shim` — `pyproject.toml` line 33 sets `packages = ["reasons", "reasons_lib"]`, publishing `reasons_lib` as a first-class package in the wheel.
- `no-test-imports-the-shim` — `find_reasons_lib_usages` and `find_reasons_lib_wildcard_imports` both return 0 matches, and `tests/test_api.py` imports only `from reasons import api`.
- `thirty-three-shim-modules` — `find_shim_mentions` lists 33 backwards-compat shim files under `reasons_lib/`, one per `reasons/*` submodule.
