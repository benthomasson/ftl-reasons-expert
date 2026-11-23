---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Audit whether `reasons_lib/{network,storage,merkle,compact,derive,mcp_server}.py` still exist; this is the delta between "`import reasons_lib` works" and "all historical `reasons_lib.X` imports work".

## Audit result

**Short version:** five of the six audited files still exist under `reasons_lib/`; `merkle.py` does not. The directory listing shows `reasons_lib` has 33 entries and `reasons` has 34 entries, with the extra file being `reasons/merkle.py`.

| Historical import path | Exists? | Evidence | Import impact |
|---|---:|---|---|
| `reasons_lib/network.py` | Yes | Present in `list_reasons_lib_directory`; corresponding `reasons/network.py` also exists | `import reasons_lib.network` can resolve to the file |
| `reasons_lib/storage.py` | Yes | Present in `list_reasons_lib_directory`; corresponding `reasons/storage.py` also exists | `import reasons_lib.storage` can resolve to the file |
| `reasons_lib/merkle.py` | **No** | Absent from `reasons_lib`; `reasons/merkle.py` exists | `import reasons_lib.merkle` / `from reasons_lib import merkle` will fail at module resolution |
| `reasons_lib/compact.py` | Yes | Present in `list_reasons_lib_directory`; corresponding `reasons/compact.py` also exists | `import reasons_lib.compact` can resolve to the file |
| `reasons_lib/derive.py` | Yes | Present in `list_reasons_lib_directory`; corresponding `reasons/derive.py` also exists | `import reasons_lib.derive` can resolve to the file |
| `reasons_lib/mcp_server.py` | Yes | Present in `list_reasons_lib_directory`; corresponding `reasons/mcp_server.py` also exists | `import reasons_lib.mcp_server` can resolve to the file |

## Why `import reasons_lib` works

`reasons_lib/__init__.py` is only a compatibility shim. Its line 2 is:

```python
from reasons import *  # noqa: F401,F403
```

That means `import reasons_lib` exercises the package `__init__.py` and then imports public names from `reasons/__init__.py`. The observed `reasons/__init__.py` defines dataclasses such as `Justification`, `Node`, and `Nogood`, so the shim can succeed even though it does not explicitly import any `reasons_lib.X` submodules.

This is the key delta:

- **`import reasons_lib` working** only proves the package `__init__.py` is importable and that `from reasons import *` succeeds.
- **`import reasons_lib.merkle` working** requires an actual `reasons_lib/merkle.py` module, or a `merkle` attribute set on the `reasons_lib` package. Neither is present in the observations.

The shim does not alias `reasons.merkle` to `reasons_lib.merkle`; it only star-imports names from `reasons/__init__.py`. Since `reasons/__init__.py` defines dataclasses and does not appear to import `reasons.merkle`, `reasons_lib.merkle` is not covered by the shim.

## In-repo usage and packaging

The repository scans found:

- zero matches for `reasons_lib\.[A-Za-z_][A-Za-z0-9_]*`
- zero matches for `from reasons_lib import` or `import reasons_lib`

So there are no observed in-repo callers currently relying on historical `reasons_lib.X` imports. The risk is therefore mainly external/downstream compatibility.

`pyproject.toml` line 31 includes `reasons_lib` in `[tool.setuptools].packages`:

```toml
packages = ["reasons", "reasons_lib"]
```

So `reasons_lib` is configured to be distributed. A missing `reasons_lib/merkle.py` can therefore affect installed downstream users even if the current repository does not import it.

## What the observations do not prove

The file listings prove existence or absence, but not importability of the five existing modules. The contents of `reasons_lib/network.py`, `storage.py`, `compact.py`, `derive.py`, and `mcp_server.py` are not shown, so a syntax error, bad import, or runtime-side dependency in one of those files would not be visible here. To fully close the audit, run import smoke tests such as:

```bash
python -c "import reasons_lib"
python -c "import reasons_lib.network"
python -c "import reasons_lib.storage"
python -c "import reasons_lib.merkle"
python -c "import reasons_lib.compact"
python -c "import reasons_lib.derive"
python -c "import reasons_lib.mcp_server"
```

Expected result from the file observations: all pass except `reasons_lib.merkle`.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The shim only does `from reasons import *`; confirm that it does not intentionally alias submodules.
- [file] `reasons_lib/merkle.py` — Missing compatibility module; compare with `reasons/merkle.py` if backward compatibility is required.
- [file] `reasons/merkle.py` — Current implementation location for merkle functionality under the new package.
- [general] `import-smoke-test` — Run the six audited imports to verify actual importability, not just file existence.
- [repo] `pyproject.toml` — Confirm packaging includes `reasons_lib` and decide whether missing submodules should be included in the wheel.

## Beliefs

- `reasons-lib-merkle-missing` — `reasons_lib/merkle.py` is absent while `reasons/merkle.py` exists.
- `reasons-lib-init-shim-uses-star-import` — `reasons_lib/__init__.py` line 2 executes `from reasons import *`, so `import reasons_lib` only exercises the shim and `reasons/__init__.py`.
- `reasons-lib-submodule-imports-require-files-or-attributes` — Historical `reasons_lib.X` imports require either a `reasons_lib/X.py` file or an explicit `X` attribute on `reasons_lib`.
- `repo-has-no-observed-reasons-lib-imports` — The observed repository grep found zero `reasons_lib.X` references and zero `from reasons_lib import` / `import reasons_lib` statements.
- `reasons-lib-packaged-by-pyproject` — `pyproject.toml` line 31 includes `"reasons_lib"` in `[tool.setuptools].packages`.
