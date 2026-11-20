---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Investigate why `reasons/api.py`, `reasons/storage.py`, etc. still import from `reasons_lib.network` rather than `reasons.network` directly; that may be a leftover to clean up.

## Summary: The premise doesn't match the code

The investigation assumes `reasons/api.py`, `reasons/storage.py`, etc. import from `reasons_lib.network`. **They don't.** The observations actually show the opposite:

- A repo-wide search for the literal pattern `reasons_lib\.network` returned **0 matches** (`network_import_sites.total_count: 0`).
- A search for `from reasons_lib` inside the `reasons` package also returned **0 matches** (`reasons_pkg_cross_imports.total_count: 0`).

So this is not a stale import path to clean up — there is nothing importing the shim.

## Why the report *looks* like it's importing `reasons_lib.network`

The confusion almost certainly comes from how relative imports are serialized in the AST dump:

| Report entry | What it actually is |
|---|---|
| `from_imports` with `module: "network"` in `reasons/api.py` | `from .network import Network` (relative import inside the `reasons` package) |
| `from_imports` with `module: "network"` in `reasons/storage.py` | `from .network import Network` (same thing) |
| `from_imports` with `module: ""` in `reasons/api.py` | `from . import Justification, pubsub` |
| `from_imports` with `module: ""` in `reasons/storage.py` | `from . import Node, Justification, Nogood` |

The dumper strips the leading `.`, so a reader scanning `module: "network"` can easily misread it as either `import network` or `import reasons_lib.network`. The code is really using *relative* imports within the `reasons` package.

Corroborating evidence from the source itself:

- `reasons/network.py` line 12 contains `from . import Node, Justification, Nogood` — the same `module: ""` relative-import form.
- `reasons/api.py` and `reasons/storage.py` are both inside the `reasons` package, so `from network import Network` (absolute) would only resolve if an unrelated top-level `network` module existed — there is no such module in the packaging config, so the only sane reading is `from .network`.

## Where `reasons_lib` actually lives today

`reasons_lib/network.py` is a **3-line re-export shim**:

```python
from reasons.network import *  # noqa: F401,F403
```

It contains no logic. Its only job is to satisfy the docs note in `CLAUDE.md`:

- Line 24: `reasons/` is "also aliased as reasons_lib for backwards compat"
- Line 35: `reasons_lib/` is a "backwards-compat shim — re-exports from reasons/"

And `pyproject.toml` still ships both packages:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

`CHANGELOG.md` line 136 references an older `reasons_lib.api` Python API layer — which is the historical origin of the shim. It's not evidence of current cross-imports.

## What is actually worth cleaning up (if anything)

The real question is not "why do internal modules import `reasons_lib.network`" (they don't) but "does `reasons_lib` still need to exist at all?" That's a **package-level** decision, not a per-file import cleanup:

1. If no external consumer imports `reasons_lib.*`, the shim package and the `"reasons_lib"` entry in `[tool.setuptools] packages` can probably be removed.
2. If external consumers still exist, the shim should stay — and the current setup (shim → real package) is exactly the right direction. Nothing to fix.
3. `reasons/network.py` importing `from . import Node, Justification, Nogood` is also fine and is not a `reasons_lib` import.
4. `CHANGELOG.md` line 136 is a historical note, not active guidance, and doesn't need to change.

## What's missing from the observations

To confirm the cleanup decision you'd still need:

- **External dependency evidence**: a search of downstream repos or a `pip download` audit showing whether anyone outside this repo references `reasons_lib`. The AST-only search here can't answer that.
- **`reasons_lib/__init__.py` and sibling shims**: the observations only show `reasons_lib/network.py`. There's likely an `api.py`, `storage.py`, `cli.py` counterpart — worth confirming they are all shims.
- **`__init__.py` of `reasons/` and `reasons_lib/`**: whether `reasons_lib/__init__.py` re-exports anything beyond `reasons` (which would affect the removal plan).

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the whole package is a pure re-export shim and not a partial fork of `reasons`
- [file] `pyproject.toml` — The `[tool.setuptools] packages` list drives whether `reasons_lib` ships to PyPI; changing this is the real cleanup
- [general] `reasons-lib-external-consumer-audit` — Grep GitHub/PyPI dependents for `from reasons_lib`/`import reasons_lib` before deleting the shim
- [file] `CHANGELOG.md` — Line 136 and surrounding version entries give the migration timeline from `reasons_lib` → `reasons`
- [general] `relative-import-json-serialization` — The AST dumper collapses `from .network import X` to `module: "network"`, which is the root cause of this misread

## Beliefs

- `reasons-lib-network-is-a-shim` — `reasons_lib/network.py` is a 3-line re-export (`from reasons.network import *`) with no logic of its own.
- `no-module-imports-reasons-lib-network` — A repo-wide search for `reasons_lib\.network` returns zero matches; no source file imports the shim directly.
- `reasons-api-uses-relative-network-import` — `reasons/api.py` imports `Network` via `from .network import Network`, not via `reasons_lib`.
- `reasons-storage-uses-relative-network-import` — `reasons/storage.py` imports `Network` via `from .network import Network`, not via `reasons_lib`.
- `reasons-lib-is-still-packaged` — `pyproject.toml` lists both `reasons` and `reasons_lib` under `[tool.setuptools] packages`, so the shim ships in the wheel and any removal is a packaging change, not an internal import change.
- `reasons-lib-exists-for-backwards-compat` — `CLAUDE.md` lines 24 and 35 explicitly document `reasons_lib` as a backwards-compat alias for `reasons`, so its continued presence is intentional and its removal hinges on external consumers.
