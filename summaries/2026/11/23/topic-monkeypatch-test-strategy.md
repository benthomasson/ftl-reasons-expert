---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Add tests that patch `reasons.Node` and `reasons_lib.Node` to verify which module attributes are visible through each import path.

## What the tests should prove

The canonical `Node` class is defined once at `reasons/__init__.py:27`. The `reasons_lib` package is a compatibility shim; its entire implementation is essentially:

```python
# reasons_lib/__init__.py
from reasons import *  # noqa: F401,F403
```

Because `reasons_lib` uses a wildcard import at import time, `reasons_lib.Node` is initially the same class object as `reasons.Node`, but it is stored as a separate module attribute. It is not a lazy proxy and does not dynamically forward to `reasons.Node`.

So the tests should verify two things:

1. Both import paths expose the same public model classes immediately after import.
2. Patching one module attribute does **not** change the other module attribute after both modules have been imported.

There are currently no existing tests patching `reasons.Node` or `reasons_lib.Node` according to `find_existing_patches_of_import_paths`, so this is a gap worth filling. A new focused test file such as `tests/test_import_paths.py` is cleaner than adding this to `tests/test_api.py`, which is API-behavior focused.

## Suggested tests

```python
# tests/test_import_paths.py
from unittest.mock import patch

import pytest

import reasons
import reasons_lib


@pytest.mark.parametrize("name", ["Justification", "Node", "Nogood"])
def test_reasons_lib_reexports_public_model_classes(name):
    assert hasattr(reasons_lib, name)
    assert getattr(reasons_lib, name) is getattr(reasons, name)


def test_patch_reasons_node_is_isolated_from_reasons_lib():
    sentinel = object()
    original_lib_node = reasons_lib.Node

    with patch("reasons.Node", sentinel):
        assert reasons.Node is sentinel
        assert reasons_lib.Node is original_lib_node
        assert reasons_lib.Node is not sentinel


def test_patch_reasons_lib_node_is_isolated_from_reasons():
    sentinel = object()
    original_reasons_node = reasons.Node

    with patch("reasons_lib.Node", sentinel):
        assert reasons_lib.Node is sentinel
        assert reasons.Node is original_reasons_node
        assert reasons.Node is not sentinel
```

The first test confirms that `reasons_lib` re-exports the public model classes defined in `reasons/__init__.py`. The second confirms that patching `reasons.Node` only changes the canonical `reasons` module namespace. The third confirms that patching `reasons_lib.Node` only changes the shim namespace.

A pytest `monkeypatch` equivalent would also work:

```python
def test_patch_reasons_node_with_monkeypatch(monkeypatch):
    sentinel = object()
    original_lib_node = reasons_lib.Node

    monkeypatch.setattr(reasons, "Node", sentinel)

    assert reasons.Node is sentinel
    assert reasons_lib.Node is original_lib_node
```

## Important import-order caveat

These tests assume both `reasons` and `reasons_lib` are imported before the patch is applied. If `reasons_lib` is imported **after** `reasons.Node` has been patched, then the wildcard import in `reasons_lib/__init__.py` will copy the patched value:

```python
with patch("reasons.Node", sentinel):
    importlib.reload(reasons_lib)
    assert reasons_lib.Node is sentinel
```

That is a different scenario. For the normal compatibility-shim behavior, import both modules first and then patch each module namespace independently.

## Optional observation

`reasons/__init__.py` does not define `__all__`, so `from reasons import *` also exposes the imported `dataclass` and `field` names. If you want to document current behavior, you could add:

```python
def test_star_import_also_exposes_dataclass_helpers():
    assert reasons_lib.dataclass is reasons.dataclass
    assert reasons_lib.field is reasons.field
```

I would keep this separate from the `Node` tests because it locks in an incidental consequence of not having `__all__`.

The observations are sufficient for the `Node`-focused tests. They do not provide exact line numbers for `Justification` and `Nogood`, so reference those by symbol rather than line number.

## Topics to Explore

- [file] `reasons/__init__.py` — Canonical definitions of `Justification`, `Node`, and `Nogood`, including the `Node` class at line 27.
- [file] `reasons_lib/__init__.py` — Backwards-compatibility shim using `from reasons import *`; inspect how wildcard exports work.
- [function] `reasons/__init__.py:Node` — The dataclass whose identity is being compared across import paths.
- [general] `python-import-star-and-patching` — Understand import-time binding versus dynamic module forwarding.
- [file] `tests/test_api.py` — Existing pytest style and the already-present but currently unused `patch` import at line 3.

## Beliefs

- `reasons-lib-is-star-shim` — `reasons_lib/__init__.py` re-exports names from `reasons` using `from reasons import *`, not by binding the `reasons` module itself.
- `node-defined-only-in-reasons` — `Node` is defined once at `reasons/__init__.py:27`; `reasons_lib.Node` is a copied reference to that class at import time.
- `patching-reasons-node-does-not-update-reasons-lib-node` — After both modules are imported, patching `reasons.Node` leaves `reasons_lib.Node` unchanged.
- `patching-reasons-lib-node-does-not-update-reasons-node` — Patching `reasons_lib.Node` leaves the canonical `reasons.Node` unchanged.
- `reasons-init-has-no-all` — `reasons/__init__.py` does not define `__all__`, so the wildcard import also exposes `dataclass` and `field`.
