---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The `__all__` interaction, `sys.modules` identity divergence, and monkeypatch-visibility caveats described in the Invariants section; decide whether to upgrade the shim to `sys.modules[...] = ...` aliasing.

## What the observations establish

- No section matching `Invariants` was found (`find_invariants_section.total_count == 0`), so the specific text of those invariants cannot be verified from the provided observations.
- The current shim is `reasons_lib/__init__.py`:
  ```python
  """Backwards-compatibility shim — use ``import reasons`` instead."""
  from reasons import *  # noqa: F401,F403
  ```
  This is a **star re-export**, not a module alias.
- `reasons/__init__.py` defines the public dataclasses `Justification`, `Node`, and `Nogood`, and imports `dataclass` and `field` from `dataclasses`.
- No `__all__` exists in either module (`find_all_definitions.total_count == 0`).
- No `sys.modules` usage exists (`find_sys_modules_usage.total_count == 0`).
- No monkeypatch tests were found (`find_monkeypatch_tests.total_count == 0`).

Because the actual invariants are missing, the safest interpretation is: these are the semantic differences you would expect from the current shim, not confirmed project invariants.

## `__all__` interaction

`reasons/__init__.py` has no `__all__`. That means:

```python
from reasons import *
```

imports every top-level name in `reasons/__init__.py` that does not start with `_`. Since `reasons/__init__.py` does:

```python
from dataclasses import dataclass, field
```

the shim currently also re-exports `dataclass` and `field` from `reasons_lib`, even though those are implementation imports, not the intended public API.

If `reasons/__init__.py` later adds:

```python
__all__ = ["Justification", "Node", "Nogood"]
```

then `from reasons import *` will respect that and stop leaking `dataclass` / `field`. This is true for both the current shim and a future `sys.modules` alias, because star import consults the target module’s `__all__`.

So the `__all__` interaction is:

- Current shim: `reasons_lib` gets a snapshot of names produced by `from reasons import *`.
- No `__all__`: the snapshot includes non-underscore imports like `dataclass` and `field`.
- Adding `__all__` in `reasons` fixes the public export surface.
- If you switch to `sys.modules[__name__] = reasons`, `__all__` should still live in `reasons`, because `reasons_lib` becomes the same module object.

## `sys.modules` identity divergence

Currently:

```python
import reasons
import reasons_lib

reasons is reasons_lib              # False
sys.modules["reasons"] is sys.modules["reasons_lib"]  # False
```

`reasons_lib` is a separate module object created by the import machinery. Its namespace contains copied bindings from `reasons` at import time. The class objects themselves are shared:

```python
reasons_lib.Node is reasons.Node    # True
reasons_lib.Justification is reasons.Justification  # True
```

But the module identities differ. That matters because module-level monkeypatching and attribute lookup are about the module object, not only the class object.

If you upgrade the shim to:

```python
import sys
import reasons

sys.modules[__name__] = reasons
```

then after import:

```python
import reasons
import reasons_lib

reasons is reasons_lib              # True
sys.modules["reasons"] is sys.modules["reasons_lib"]  # True
```

That gives true module aliasing at the top level.

Important caveat: if `reasons` has submodules, aliasing only the top-level package does not automatically alias `reasons.foo` and `reasons_lib.foo`. `import reasons_lib.foo` could still load a separate module object unless each submodule is also aliased. The observations do not show whether `reasons` has submodules, so this needs checking before adopting `sys.modules` aliasing.

## Monkeypatch visibility caveats

With the current shim, module-attribute replacement diverges:

```python
import reasons
import reasons_lib
from unittest.mock import patch

with patch("reasons.Node", DummyNode):
    reasons.Node is DummyNode       # True
    reasons_lib.Node is DummyNode   # False
```

The reverse is also true:

```python
with patch("reasons_lib.Node", DummyNode):
    reasons.Node is DummyNode       # False
    reasons_lib.Node is DummyNode   # True
```

So monkeypatching the source module does not automatically affect the shim module, and monkeypatching the shim does not affect the source module.

However, class-object attribute patching is shared:

```python
reasons.Node.custom_flag = True
reasons_lib.Node.custom_flag        # True
```

because `reasons_lib.Node` and `reasons.Node` point to the same class object. The divergence is specifically at the module namespace level: replacing `reasons.Node` with a different object does not update `reasons_lib.Node`.

`sys.modules` aliasing fixes the module-attribute case because both imports resolve to the same module object. It does **not** fix ordinary Python early-binding:

```python
from reasons_lib import Node

# later monkeypatch reasons.Node = DummyNode
# the local `Node` name still points at the original class
```

That is a general `from ... import ...` caveat, not something aliasing can solve.

## Decision: upgrade the shim or not?

The observations are insufficient to prove the intended compatibility contract, because the `Invariants` section is absent and there are no monkeypatch tests or known consumers.

Based on the code alone:

- If backwards compatibility only requires that `import reasons_lib` exposes `Justification`, `Node`, and `Nogood`, the current shim is probably sufficient, but it should add an explicit `__all__` in `reasons/__init__.py` to avoid leaking `dataclass` and `field`.
- If backwards compatibility means `reasons_lib` should behave as a transparent alias of `reasons`, including `sys.modules` identity and module-level monkeypatch visibility, then upgrade the shim to `sys.modules[__name__] = reasons`.

My recommendation: add `__all__` in `reasons/__init__.py` regardless. Upgrade to `sys.modules` aliasing only if you confirm that consumers or tests expect `reasons_lib` to be the same module object as `reasons`, or expect monkeypatching `reasons.Node` to be visible through `reasons_lib.Node`. Before doing that, check for `reasons` submodules and any code that depends on `reasons_lib.__name__`, `__spec__`, `importlib.reload`, or `reasons_lib`-specific metadata.

## Topics to Explore

- [file] `reasons/__init__.py` — Add or verify `__all__` to control the star-import surface and stop exposing `dataclass` / `field`.
- [file] `reasons_lib/__init__.py` — Decide whether to keep the star-import shim or replace it with `sys.modules[__name__] = reasons`.
- [general] `import-aliasing-compatibility` — Check consumers that rely on `reasons_lib.__name__`, `__spec__`, reload behavior, pickling, or submodule imports.
- [general] `monkeypatch-test-strategy` — Add tests that patch `reasons.Node` and `reasons_lib.Node` to verify which module attributes are visible through each import path.
- [general] `reasons-submodule-layout` — Confirm whether `reasons` has submodules; top-level `sys.modules` aliasing does not automatically alias `reasons.foo` to `reasons_lib.foo`.

## Beliefs

- `reasons-lib-uses-star-import` — `reasons_lib/__init__.py` re-exports via `from reasons import *` rather than aliasing the module object.
- `reasons-defines-no-all` — `reasons/__init__.py` has no `__all__`, so the star import also exposes `dataclass` and `field`.
- `reasons-lib-module-identity-diverges` — Without a `sys.modules` assignment, `reasons_lib` and `reasons` are distinct module objects even though their class attributes initially point to the same classes.
- `module-attribute-monkeypatches-are-not-shared` — Replacing `reasons.Node` does not update `reasons_lib.Node` under the current shim.
- `class-attribute-monkeypatches-are-shared` — Patching an attribute on the `Node` class object is visible through both imports because both names bind the same class object.
