---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How `__all__`, underscore names, and imported dependencies affect what this shim exports.

## What the shim exports

The shim is `reasons_lib/__init__.py:1`:

```python
from reasons import *  # noqa: F401,F403
```

It defines no local names and has no `__all__`. Therefore its exports are determined by Python’s default `import *` behavior applied to `reasons`.

### `__all__` is absent

`find_export_controls` found no `__all__`, `__getattr__`, `__dir__`, or underscore-filtering logic anywhere (`total_count: 0`). So neither `reasons/__init__.py` nor `reasons_lib/__init__.py` explicitly controls wildcard exports.

Without `__all__`, `from reasons import *` binds every name in `reasons` that does not start with `_`. In this code that means:

- `Justification` — defined at `reasons/__init__.py:6`
- `Node` — defined at `reasons/__init__.py:26`
- `Nogood` — defined at `reasons/__init__.py:52`
- `dataclass` — imported at `reasons/__init__.py:2`
- `field` — imported at `reasons/__init__.py:2`

Because `reasons_lib` is just `from reasons import *`, those same names become attributes of `reasons_lib` and are therefore exported by the shim as well.

### Imported dependencies leak through the shim

The import at `reasons/__init__.py:2` is:

```python
from dataclasses import dataclass, field
```

Since `dataclass` and `field` do not start with an underscore, they are public names in `reasons`. With no `__all__`, `from reasons import *` re-exports them. As a result:

```python
from reasons_lib import dataclass, field
```

works, even though those names are implementation details of `reasons`, not the intended domain API.

If the import had been written with underscore aliases, such as `from dataclasses import dataclass as _dataclass`, those names would not be picked up by wildcard import unless explicitly included in `__all__`.

### Underscore names

There are no user-defined underscore-prefixed names in the shown `reasons/__init__.py` content. If there were, they would be excluded from `from reasons import *` by default because there is no `__all__`.

Dunder names such as `__name__` and `__doc__` are also excluded by default.

An `__all__` list would override this default rule: names listed in `__all__` are exported by `import *` even if they start with `_`. But no such list exists here.

### Direct imports are not restricted by `__all__`

Even if `__all__` existed, it would only affect wildcard imports like:

```python
from reasons_lib import *
```

It would not normally block direct attribute access such as:

```python
from reasons_lib import Justification
import reasons_lib
reasons_lib.Node
```

Those work because the shim has already bound the names in its module namespace via `from reasons import *`.

### Shim’s own export boundary

`reasons_lib/__init__.py` also has no `__all__`. So `from reasons_lib import *` exports all public names currently bound in `reasons_lib`, which are exactly the public names imported from `reasons`.

If you wanted to fix the leak, the most direct options would be:

- Add `__all__ = ["Justification", "Node", "Nogood"]` in `reasons/__init__.py`. This would also narrow what `reasons_lib` receives through `from reasons import *`.
- Add an `__all__` in `reasons_lib/__init__.py` to control the shim’s wildcard exports explicitly.
- Rename imported implementation names with underscore aliases in `reasons/__init__.py`.

### Packaging and tests

`pyproject.toml` includes both packages in `[tool.setuptools]`:

```toml
packages = ["reasons", "reasons_lib"]
```

So the shim is part of the installed package surface.

The search for export expectations in tests found no matches (`total_count: 0`). That means the observed tests do not assert anything about `__all__`, `from reasons import`, `import reasons`, or `reasons_lib`. The re-export of `dataclass` and `field` is therefore an unguarded API surface leak unless downstream usage intentionally relies on it.

## Topics to Explore

- [file] `reasons/__init__.py` — Defines the real dataclasses and public imported names that determine the shim’s export surface.
- [file] `reasons_lib/__init__.py` — The compatibility shim itself; adding `__all__` here would explicitly control its wildcard exports.
- [general] `star-import-compatibility` — Check whether downstream code imports `dataclass` or `field` from `reasons_lib`, or relies on other incidental re-exports.
- [file] `pyproject.toml` — Confirms both `reasons` and `reasons_lib` are packaged and installed.
- [general] `export-contract-tests` — No tests currently assert the shim’s export contract; add tests around `__all__` and wildcard imports if this API matters.

## Beliefs

- `reasons-lib-export-surface-requires-reasons-public-names` — `reasons_lib/__init__.py` exports whatever public names it receives from `from reasons import *`.
- `reasons-no-all-enables-default-underscore-filtering` — Because `reasons/__init__.py` has no `__all__`, wildcard imports use Python’s default rule of excluding underscore-prefixed names.
- `reasons-lib-reexports-dataclass-and-field` — `dataclass` and `field` are public imports in `reasons/__init__.py` and therefore leak through `reasons_lib`’s star import.
- `direct-imports-do-not-require-all` — Direct imports such as `from reasons_lib import Justification` work because the shim binds names in its module namespace, independent of `__all__`.
- `packaging-includes-reasons-lib-shim` — `pyproject.toml` packages both `reasons` and `reasons_lib`, making the shim part of the installed distribution.
