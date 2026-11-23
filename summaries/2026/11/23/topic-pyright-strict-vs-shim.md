---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Run pyright under `strict` against a consumer that imports from `reasons_lib` to see what falls back to `Any`.

## What this check is / what we can infer

This is an **investigation step**, not a completed result in the provided observations. There is no consumer file and no captured pyright output, so we cannot name the exact symbols that fall back to `Any`.

What the observations do show is the structural risk:

- `reasons_lib/__init__.py:2` re-exports everything from `reasons`:
  - `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py:2` re-exports everything from `reasons.api`:
  - `from reasons.api import *  # noqa: F401,F403`
- `reasons_lib/ask.py:2` re-exports everything from `reasons.ask`:
  - `from reasons.ask import *  # noqa: F401,F403`

These are pure wildcard shims. The `# noqa` comments silence linters such as flake8/ruff; they do **not** affect pyright.

`pyproject.toml` includes `reasons_lib` as a package around line 31:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

But there is no pyright configuration. The only grep hit for `pyright` is `.git/hooks/pre-rebase.sample:3`, which is unrelated. The observed shim files also contain no literal `Any`, so any `Any` fallback would have to come from pyright inference or from unresolved/untyped imports through `reasons`, `reasons.api`, or `reasons.ask`.

The separate grep for `__all__|import *` returned no matches, which conflicts with the visible `import *` lines in the shim files. Treat that grep result as unreliable and re-run it if needed.

## How to run the check

Create a small consumer and run pyright in `strict` mode.

Example consumer:

```python
# consumer.py
from reasons_lib import SomeName
from reasons_lib.api import SomeApiName
from reasons_lib.ask import SomeAskName

reveal_type(SomeName)
reveal_type(SomeApiName)
reveal_type(SomeAskName)
```

The exact names must come from inspecting `reasons/__init__.py`, `reasons/api.py`, and `reasons/ask.py`; those files are not included in the observations.

Example `pyrightconfig.json`:

```json
{
  "include": ["consumer.py"],
  "typeCheckingMode": "strict",
  "reportMissingTypeStubs": true,
  "reportUnknownVariableType": true,
  "reportUnknownMemberType": true,
  "reportAny": true
}
```

Then run:

```bash
pyright -p pyrightconfig.json
```

In the output, look for:

- `reveal_type(...)` showing `Any` or `Unknown`
- `reportAny`
- `reportUnknownVariableType`
- `reportUnknownMemberType`
- `reportMissingTypeStubs`

Pyright distinguishes `Any` from `Unknown`, but under `strict` both usually surface as diagnostics. The practical question is: which names imported through `reasons_lib` lose their concrete types and become `Any`/`Unknown`.

## Expected failure mode

The likely problem is the wildcard chain:

```text
consumer -> reasons_lib -> reasons / reasons.api / reasons.ask
```

If pyright cannot resolve the exported names through those `import *` statements, it will treat them as `Any`/`Unknown`. That can happen when:

- the upstream module does not declare `__all__`,
- the upstream module exports dynamically,
- the installed package lacks `py.typed`,
- or the consumer is analyzing an installed package rather than local source.

The observations do not include `reasons/__init__.py`, `reasons/api.py`, `reasons/ask.py`, any `py.typed` marker, or the pyright version. Those are needed to predict the actual fallback symbols.

## Topics to Explore

- [file] `reasons/__init__.py` — Defines what `from reasons import *` actually exports and whether pyright can resolve those names.
- [file] `reasons/api.py` — Upstream module behind `reasons_lib/api.py`; determines whether API symbols survive the wildcard re-export with real types.
- [file] `reasons/ask.py` — Upstream module behind `reasons_lib/ask.py`; determines whether ask-related symbols become `Any`/`Unknown`.
- [general] `pyright-strict-run` — Create a consumer with explicit imports and `reveal_type` calls, then capture the actual `pyright -p pyrightconfig.json` output.
- [general] `py.typed-packaging` — Check whether `reasons` and `reasons_lib` ship `py.typed`; missing markers make installed-package consumers much more likely to see `Any` under strict mode.

## Beliefs

- `reasons-lib-init-uses-wildcard-reexport` — `reasons_lib/__init__.py:2` contains `from reasons import *`, so its public API is delegated to `reasons`.
- `reasons-lib-api-uses-wildcard-reexport` — `reasons_lib/api.py:2` contains `from reasons.api import *`, so its public API is delegated to `reasons.api`.
- `reasons-lib-ask-uses-wildcard-reexport` — `reasons_lib/ask.py:2` contains `from reasons.ask import *`, so its public API is delegated to `reasons.ask`.
- `reasons-lib-shims-declare-no-__all__` — The three observed shim files contain only a docstring and a wildcard import; none declares `__all__`.
- `pyright-config-is-absent` — The only `pyright` grep match is `.git/hooks/pre-rebase.sample:3`, not a pyright configuration file.
