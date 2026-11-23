---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Compare how CLI aliases (`reasons/cli.py:3064`, `reasons/cli.py:3854`) and the Python shim are both deprecated via documentation only, suggesting a project-wide pattern.

## Comparison

The common pattern is **soft, documentation-only deprecation**: the old surface remains fully functional, but its help text/docstring tells users what to use instead. There is no observed runtime warning, removal gate, or shared deprecation helper.

| Surface | Where marked | Documentation marker | Runtime behavior observed |
|---|---|---|---|
| CLI option `--project-id` | `reasons/cli.py:3064` | `help="(deprecated alias for --domain-id)"` | Still a normal argparse option with `default=None`; old invocations still parse. |
| CLI subcommand `research` | `reasons/cli.py:3854` | `help="Alias for 'repair' (deprecated)"` plus comment “backward-compatible alias for repair” | Still a fully defined subparser, duplicating `repair`’s arguments. |
| Python package `reasons_lib` | `reasons_lib/*.py:1`, e.g. `reasons_lib/api.py:1` | `"""Backwards-compatibility shim — use ``from reasons.api`` instead."""` | Still importable; `reasons_lib/__init__.py` reexports from `reasons` via `from reasons import *`. |

### CLI alias 1: `--project-id`

At `reasons/cli.py:3064`, `--project-id` is added as an argparse option:

```python
parser.add_argument("--project-id", default=None,
                    help="(deprecated alias for --domain-id)")
```

This is documentation-only: argparse still accepts `--project-id`, and the excerpt does not show any warning, error, or removal. The intended replacement is declared inline as `--domain-id`. The excerpt does not show `_backend_kwargs`, so the exact mapping from `project_id` to `domain_id` is not visible here, but the help text is the deprecation signal.

### CLI alias 2: `research`

At `reasons/cli.py:3854`, `research` is defined as a separate subparser:

```python
# research (backward-compatible alias for repair)
p = sub.add_parser("research", help="Alias for 'repair' (deprecated)")
```

It repeats the same argument surface as `repair`: `ids`, `--review-file`, `-m/--model`, `--timeout`, and `--dry-run`. Again, there is no observed runtime warning. The alias is kept working and documented as deprecated only in the parser help/comment.

### Python shim

The Python compatibility layer is broader and more systematic. The observations show 33 `shim` mentions across `reasons_lib/*.py`, each at line 1 as a module docstring such as:

```python
"""Backwards-compatibility shim — use ``from reasons.api`` instead."""
```

`reasons_lib/__init__.py` is also documented as a backwards-compatibility shim and reexports from `reasons`:

```python
from reasons import *  # noqa: F401,F403
```

`pyproject.toml` still includes `reasons_lib` in `tool.setuptools.packages`:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

So the shim is not merely source-level documentation; it remains installed and importable. The deprecation signal is the docstring telling users to import from `reasons.*` instead.

### Project-wide pattern

The observed evidence suggests a consistent project-wide style:

1. **Keep the old surface working.**  
   `--project-id` remains an accepted argparse option; `research` remains a valid subparser; `reasons_lib` remains a packaged import path.

2. **Document the replacement locally.**  
   CLI aliases are marked in `help=` strings at `reasons/cli.py:3064` and `reasons/cli.py:3854`. Python shims are marked in module docstrings across `reasons_lib/*.py`.

3. **Do not enforce deprecation at runtime.**  
   The `deprecat` search found only three matches total:
   - `reasons/cli.py:3064` — CLI help text
   - `reasons/cli.py:3854` — CLI help text
   - `reasons/api.py:3919` — an unrelated word list containing `"deprecated"`

   No `DeprecationWarning`, `FutureWarning`, or runtime warning call is observed.

4. **Do not centralize the policy in README.**  
   The observed `README.md` search for `deprecat|shim|alias` returned zero matches. In this snapshot, deprecation guidance is code-local rather than README-level.

5. **Use “shim” rather than “deprecated” for Python compatibility.**  
   The Python layer does not literally say “deprecated” in the observed matches; it says “Backwards-compatibility shim — use … instead.” This is still documentation-only deprecation, but the wording differs from the CLI help strings.

### Limitations

The observations do not show:

- the dispatch code that routes `research` to the same handler as `repair`;
- the `_backend_kwargs` logic that maps `project_id` to `domain_id`;
- the full bodies of all `reasons_lib/*.py` shim modules;
- tests or changelog entries asserting deprecation behavior.

Those would confirm whether the pattern is fully uniform across the project.

## Topics to Explore

- [function] `reasons/cli.py:main` — See where `args.project_id` is mapped into backend kwargs and where `research` dispatches to the repair handler.
- [file] `reasons_lib/__init__.py` — Confirm the shim reexport and docstring, and check whether any runtime warning is emitted.
- [file] `pyproject.toml` — Verify `reasons_lib` packaging and whether any legacy console scripts remain.
- [general] `deprecation-warning-search` — Search for `warnings.warn`, `DeprecationWarning`, and `FutureWarning` to confirm no runtime signaling exists elsewhere.
- [file] `README.md` — Check whether user-facing migration guidance exists outside inline help/docstrings.

## Beliefs

- `cli-project-id-is-documented-deprecated-alias` — `--project-id` is an argparse option whose help text calls it a deprecated alias for `--domain-id` at `reasons/cli.py:3064`.
- `cli-research-is-documented-deprecated-alias` — `research` is a subparser labeled “Alias for 'repair' (deprecated)” at `reasons/cli.py:3854`, with arguments duplicated from `repair`.
- `python-shim-reexports-reasons` — `reasons_lib/__init__.py` reexports from `reasons` via `from reasons import *` and identifies itself as a backwards-compatibility shim.
- `python-shim-is-packaged` — `pyproject.toml` includes `reasons_lib` in `tool.setuptools.packages`, so the shim is installed alongside `reasons`.
- `deprecation-is-not-runtime-signaled` — The observed `deprecat` matches are limited to two CLI help strings and an unrelated `reasons/api.py:3919` word list; no runtime deprecation warning is observed.
