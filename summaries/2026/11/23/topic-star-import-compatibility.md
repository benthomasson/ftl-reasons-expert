---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search downstream callers before adding `__all__`, since code may be relying on incidental names like `from reasons import dataclass`.

## Why search downstream callers before adding `__all__`

`__all__` is not just documentation: it changes what `from module import *` exports and declares the package’s public surface. This repo currently has no `__all__` anywhere (`find_existing_all_definitions`: `total_count: 0`), so adding one to `reasons/__init__.py` would be the first explicit public-API gate.

The important detail is that `reasons/__init__.py:3` does:

```python
from dataclasses import dataclass, field
```

Because `reasons/__init__.py` has no `__all__`, those names are currently exposed as package attributes. A caller can do `from reasons import dataclass` or `from reasons import field`, even though they are incidental implementation imports, not intended reason-maintenance API.

A repo search found no in-repo `from reasons import dataclass` matches (`find_incidental_dataclass_import`: `total_count: 0`). However, that does not mean the change is safe, because the search also shows:

- `reasons_lib/__init__.py:2` does `from reasons import *`.
- That shim currently re-exports everything from `reasons`, including incidental names like `dataclass` and `field`.
- If `reasons.__all__` is added and omits those names, then `reasons_lib.dataclass` would disappear. So `from reasons_lib import dataclass` could break even though `from reasons import dataclass` would still work.
- If adding `__all__` is paired with cleanup of incidental imports, then even explicit `from reasons import dataclass` can break.

The downstream search also shows real intended public names. `find_imports_from_reasons_package` found 67 matches, including:

- `reasons/mcp_server.py:23` — `from reasons import api`
- `tests/test_access_tags.py:5` — `from reasons import Justification`
- `tests/test_add_justification.py:5` — `from reasons import Justification, api`
- `tests/test_dangling_dependents.py:7` — `from reasons import Justification, Node`
- `tests/test_export_card.py:23` — `from reasons import Node`
- `tests/test_export_card.py:35` — `from reasons import Nogood`
- `tests/test_hf_default_org.py:228` — `from reasons import cli`

So before adding `__all__`, you should decide whether the public API includes `Justification`, `Node`, `Nogood`, `api`, and `cli`, and whether the `reasons_lib` shim must continue re-exporting them. The only in-repo star import of `reasons` is `reasons_lib/__init__.py:2`; `tests/test_llm.py:10` imports `reasons.llm` as a module and is not affected by `__all__`. There are no in-repo `from reasons_lib import ...` matches, so any `reasons_lib` consumers are likely external and not visible in this search.

The missing piece is external usage: the repo search cannot prove that no downstream package outside this tree relies on `from reasons import dataclass` or `from reasons_lib import *`.

## Topics to Explore

- [file] `reasons/__init__.py` — Current package namespace; line 3 imports `dataclass` and `field`, and any `__all__` would be added here.
- [file] `reasons_lib/__init__.py` — Backwards-compatibility shim using `from reasons import *` at line 2; directly affected by any new `__all__`.
- [file] `reasons/mcp_server.py` — Downstream caller importing `from reasons import api` at line 23.
- [general] `python-star-import-semantics` — Clarifies that `__all__` affects `from module import *`, while explicit `from module import name` still works if the attribute remains.
- [general] `external-consumer-audit` — How to search package registries or downstream repos for `from reasons import dataclass`, `from reasons import field`, or `from reasons_lib import *` outside this repository.

## Beliefs

- `reasons-package-has-no-all` — `reasons/__init__.py` currently defines no `__all__`, so star imports expose all non-underscore names including `dataclass` and `field`.
- `reasons-lib-shim-uses-star-import` — `reasons_lib/__init__.py:2` re-exports `reasons` via `from reasons import *`, making the shim sensitive to any new `__all__` in `reasons`.
- `explicit-dataclass-import-not-found-in-repo` — The checked-in repository contains zero matches for `from reasons import dataclass`.
- `public-reasons-imports-include-model-types-and-submodules` — In-repo callers import `Justification`, `Node`, `Nogood`, `api`, and `cli` from `reasons`, as seen in tests and `reasons/mcp_server.py:23`.
- `no-in-repo-reasons-lib-importers` — The repository contains zero matches for `from reasons_lib import`, so shim consumers are not visible in the checked-in code.
