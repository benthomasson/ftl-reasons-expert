---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Compare the `reasons_lib` shim against other deprecated aliases in the CLI to determine whether the project's convention is docstring-only deprecation or runtime warnings.

## Conclusion

Based on the observations, the `reasons_lib` shims use **docstring-only deprecation**, not runtime warnings.

Both shim files contain only a backwards-compatibility docstring plus a wildcard re-export:

- `reasons_lib/__init__.py` — 3-line file: docstring says use `import reasons` instead, followed by `from reasons import *`.
- `reasons_lib/cli.py` — 3-line file: docstring says use `from reasons.cli` instead, followed by `from reasons.cli import *`.

Neither file contains `warnings.warn`, `DeprecationWarning`, or any runtime warning logic.

The comparison against the CLI also does **not** reveal a runtime-warning convention. A search of `reasons/cli.py` for `deprecat|DeprecationWarning|warnings.warn|warn(|alias` returned **0 matches across 4015 lines**. That means:

- There are no observed runtime-warning-based deprecated aliases in `reasons/cli.py`.
- There are not even other observed deprecated-alias patterns in the CLI to compare against.
- The only observed deprecation-style aliases are the `reasons_lib` shims, and they are docstring-only.

Supporting evidence also points away from runtime warnings:

| Area | Observation | Implication |
|---|---|---|
| `reasons_lib/__init__.py` | Docstring + `from reasons import *` | No runtime warning |
| `reasons_lib/cli.py` | Docstring + `from reasons.cli import *` | No runtime warning |
| `reasons/cli.py` | 0 deprecation/alias/warning-pattern matches in 4015 lines | No runtime-warning alias convention observed |
| Deprecated alias tests | 0 matches for `reasons_lib|deprecated alias|DeprecationWarning|pytest.warns|filterwarnings` | No observed tests expecting deprecation warnings |
| Warning filters | 0 matches for `filterwarnings|DeprecationWarning` | No observed warning-filter configuration |
| Docs | `CHANGELOG.md:136` mentions `reasons_lib.api`, but only as a Python API layer | Not evidence of runtime deprecation warnings |

One caution: the visible `reasons/cli.py` context includes `_warn_multi_premise(...)`, but that is a user-facing tip for multi-premise SL nodes, not a deprecation warning. It should not be treated as evidence of a runtime deprecation convention.

So within the observed scope, the convention is: **deprecated aliases are documented in module docstrings, and the shim immediately re-exports the replacement module. No runtime warnings are emitted.** However, the observations do not cover the entire repository, and there were no other deprecated aliases found in `reasons/cli.py` to provide a broader positive comparison.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the shim is exactly a docstring plus `from reasons import *`, with no warning call.
- [file] `reasons_lib/cli.py` — Verify the CLI shim follows the same docstring-only pattern.
- [file] `reasons/cli.py` — Search the full 4015-line file for any deprecation, alias, or warning behavior; observed pattern count was zero.
- [function] `reasons/cli.py:_warn_multi_premise` — Confirm this warning-like helper is only a multi-premise usage tip, not deprecation infrastructure.
- [general] `repo-wide-deprecation-policy` — Scan all modules, tests, and config for `warnings.warn`, `DeprecationWarning`, and shim patterns to determine whether docstring-only deprecation is project-wide.

## Beliefs

- `reasons-lib-init-shim-is-docstring-only` — `reasons_lib/__init__.py` contains a backwards-compatibility docstring followed by `from reasons import *` and no runtime warning call.
- `reasons-lib-cli-shim-is-docstring-only` — `reasons_lib/cli.py` contains a backwards-compatibility docstring followed by `from reasons.cli import *` and no runtime warning call.
- `reasons-cli-has-zero-deprecation-pattern-matches` — A search of `reasons/cli.py` for deprecation, `DeprecationWarning`, `warnings.warn`, `warn(`, or alias patterns found 0 matches across 4015 lines.
- `deprecated-alias-tests-absent` — The observed test search for `reasons_lib`, deprecated alias, `DeprecationWarning`, `pytest.warns`, and `filterwarnings` found 0 matches.
- `warning-filters-absent` — The observed configuration search for `filterwarnings` and `DeprecationWarning` found 0 matches.
