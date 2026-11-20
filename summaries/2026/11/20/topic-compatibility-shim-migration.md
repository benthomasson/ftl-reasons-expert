---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Inventory remaining `reasons_lib.storage` consumers and plan migration to `reasons.storage`.

## Inventory of `reasons_lib.storage` consumers

### Confirmed from observations

| Location | Status | Evidence |
|---|---|---|
| `reasons_lib/storage.py` | Compatibility shim, not a real consumer | Line 2: `from reasons.storage import *` |
| `reasons_lib/__init__.py` | Legacy package shim | Line 2: `from reasons import *` |
| `reasons/storage.py` | Migration target / real implementation | Defines `Storage`, schema constants, `save`, `load`, etc. |
| `grep_reasons_lib_storage` | No matches | `reasons_lib\.storage` total count `0` |
| `grep_reasons_lib_import_storage` | No matches | `from reasons_lib import storage` / `import reasons_lib.storage` total count `0` |
| `grep_reasons_storage_usage` | Reported no matches, but inconsistent | `reasons_lib/storage.py:2` contains `reasons.storage`, so this grep result conflicts with the file content |

**Conclusion so far:** within the observed grep results, there are **no remaining consumers** of `reasons_lib.storage` outside the shim itself. The only concrete old-path artifact is `reasons_lib/storage.py:2`, which re-exports `reasons.storage`.

### Important caveats

The inventory is **not yet complete**:

1. The grep pattern does not include the most common import form:
   ```python
   from reasons_lib.storage import Storage
   ```
   The observed pattern `from reasons_lib import storage|import reasons_lib\.storage` would miss that.

2. `grep_reasons_storage_usage` reports zero `reasons.storage` matches, but `reasons_lib/storage.py:2` literally contains `from reasons.storage import *`. That means the grep was scoped differently, excluded that file, or the summary is inconsistent. Do not rely on that zero result without rerunning the search.

3. No observations cover dynamic imports, documentation, tests, scripts, notebooks, or external/published-package consumers:
   ```python
   importlib.import_module("reasons_lib.storage")
   __import__("reasons_lib.storage")
   ```

## Migration plan to `reasons.storage`

### 1. Run a complete import audit

Search for all plausible old-path forms, including `from` imports and dynamic imports:

```bash
rg -n --hidden --glob '!.git' \
  'reasons_lib\.storage|from\s+reasons_lib\.storage\s+import|from\s+reasons_lib\s+import\s+storage|import\s+reasons_lib\.storage'
```

Also search docs/tests/scripts and dynamic import strings:

```bash
rg -n 'importlib\.import_module\(["'\'']reasons_lib\.storage|__import__\(["'\'']reasons_lib\.storage' .
rg -n 'reasons_lib\.storage' docs tests scripts notebooks 2>/dev/null
```

### 2. Classify matches

- **Shim files**
  - `reasons_lib/storage.py:2` — expected legacy shim.
  - `reasons_lib/__init__.py:2` — package-level legacy shim.

- **Real consumers**
  - Any application/library/test code importing `reasons_lib.storage`.
  - Any dynamic import strings.
  - Any docs still recommending the old path.

### 3. Rewrite consumers to the canonical path

Typical rewrites:

```python
# Before
from reasons_lib.storage import Storage

# After
from reasons.storage import Storage
```

```python
# Before
import reasons_lib.storage as storage

# After
import reasons.storage as storage
```

```python
# Before
from reasons_lib import storage

# After
import reasons.storage as storage
```

Prefer explicit imports over relying on `reasons_lib/storage.py:2`'s star import:

```python
from reasons.storage import Storage
```

instead of:

```python
from reasons_lib.storage import *
```

The shim uses `from reasons.storage import *` with `# noqa: F401,F403`, so consumers may accidentally depend on non-public names exported by `reasons.storage`.

### 4. Validate the target package

`reasons/storage.py` imports:

- `from . import Node, Justification, Nogood` — line 12
- `from .metadata import SCHEMA_VERSION, build_meta` — line 13
- `from .network import Network` — line 14

Confirm that after migration:

```bash
python -c "from reasons.storage import Storage; print(Storage)"
```

and that the full test suite passes. Also check for import cycles, since `reasons.storage` depends on sibling modules inside the `reasons` package.

### 5. Deprecate, then remove, the shim

Because `reasons_lib.storage` is already a backwards-compatibility shim, the safe sequence is:

1. Migrate all in-repo consumers.
2. Keep `reasons_lib/storage.py` for external users if the package is published.
3. Optionally add a `DeprecationWarning` in `reasons_lib/storage.py` to surface remaining runtime consumers.
4. Remove `reasons_lib/storage.py` only in a breaking release, after confirming external migration.
5. Update packaging metadata and docs to point only to `reasons.storage`.

If this is an internal-only repo, step 4 can happen as soon as the full audit and tests are clean.

## Topics to Explore

- [file] `reasons/storage.py` — Actual persistence implementation; inspect `Storage`, schema constants, and migration logic before switching imports.
- [file] `reasons_lib/storage.py` — Compatibility shim that should eventually be removed; its line 2 is the only confirmed old-path re-export.
- [function] `reasons/storage.py:Storage.save` — Main write API consumers are likely to depend on; verify migration does not rely on shim-specific behavior.
- [general] `import-usage-audit` — Rerun a broader search including `from reasons_lib.storage import ...` and dynamic imports.
- [general] `deprecation-policy` — Decide whether `reasons_lib.storage` must remain for external consumers and whether to emit warnings before removal.

## Beliefs

- `reasons-lib-storage-is-shim` — `reasons_lib/storage.py:2` re-exports `reasons.storage` via a star import.
- `reasons-storage-contains-storage-class` — The real persistence implementation is `reasons/storage.py:Storage`, with `save` and `load` methods.
- `reasons-lib-init-is-shim` — `reasons_lib/__init__.py:2` re-exports the canonical `reasons` package.
- `provided-grep-cannot-rule-out-from-imports` — The observed grep patterns omit `from reasons_lib.storage import ...`, so zero matches do not prove there are no consumers.
- `reasons-storage-depends-on-metadata-and-network` — `reasons/storage.py` imports `SCHEMA_VERSION`, `build_meta`, and `Network` from sibling `reasons` modules.
