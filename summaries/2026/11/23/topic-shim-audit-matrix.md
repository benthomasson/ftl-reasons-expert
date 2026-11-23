---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Build a table of {shim path, source module `__all__`, private names in source, tests that patch through shim} for all ~30 shims; the 0-match results in this prompt are not trustworthy until re-run without filters.

# Building the shim ↔ source audit table

## First: the observations have the polarity backwards

The labels in the prompt are inverted relative to the code:

- `representative_shim_file` is **`reasons/api.py`** (5886 lines) — this is the *source module*. It has real logic: `DEFAULT_DB = "reasons.db"`, `_network_cache`, `_with_network`, `_pg_dispatch`, `init_db`, etc.
- `representative_source_module` is **`reasons_lib/api.py`** (3 lines) — this is the *shim*:

  ```python
  """Backwards-compatibility shim — use ``from reasons.api`` instead."""
  from reasons.api import *  # noqa: F401,F403
  ```

So the audit is: **shims live in `reasons_lib/`, sources live in `reasons/`.** Every row of the table you want is `reasons_lib/<mod>.py → reasons/<mod>.py`.

That single fact invalidates three of the four 0-match greps by construction (see next section).

## Why the 0-match results are not trustworthy

| Observation | Pattern | Why it returned 0 | Correct query |
|---|---|---|---|
| `all_shim_source_imports` | `from reasons_lib` | Backwards — shims never import from `reasons_lib`; they import **from** `reasons` | `grep -rl '^from reasons\.\|^from reasons import' reasons_lib/` |
| `all_source_module_all_exports` | `__all__` | May genuinely be absent (the first 200 lines of `reasons/api.py` have no `__all__`), **or** the grep root excluded `reasons/` | `grep -rn '^__all__' reasons/` — if 0, that is itself a finding: `from reasons.api import *` falls back to implicit "all names not starting with `_`" |
| `all_shim_module_all_exports` | `__all__` | Shims are 3 lines and define nothing; 0 is **expected**, but it means shims also lack `__all__`, so `from reasons_lib.api import *` re-exports the source's set verbatim | Confirm by inspecting `reasons_lib/*.py` line count ≈ 3 |
| `tests_patching_through_shim` | `patch.*reasons\|reasons.*patch\|setattr.*reasons\|monkeypatch.*reasons` | Requires "reasons" and the verb to appear *on the same line* with that exact ordering; misses `@patch("reasons_lib.api.Storage")`, `monkeypatch.setattr(module, "Storage", …)`, and fixture-based patches | `grep -rnE '(patch\|setattr)\(\s*["\x27][^"\x27]*reasons' tests/` and `grep -rn 'monkeypatch\.setattr' tests/` |
| `private_names_in_source_modules` | `^def _[A-Za-z0-9_]+\|^class _…\|^_[A-Za-z0-9_]+[[:space:]]*=` | The third alternation **cannot match annotated globals**. `reasons/api.py:34` is `_network_cache: dict[str, tuple[float, Network]] = {}` — the `:` annotation breaks `_name[[:space:]]*=`. Also, if grep root was `reasons_lib/`, `^def _` finds nothing because shims have no functions | `grep -nE '^def _\|^class _\|^_[a-zA-Z0-9_]+(\s*:\s*[^=]+)?\s*=' reasons/` |

**Conclusion:** the reported `total_count: 0` values are consistent with the query being pointed at the wrong directory *or* using the wrong polarity (or both). Do not use them in the table until re-run.

## How to compute each column

### 1. `shim path`
List files under `reasons_lib/` with < ~10 lines. These are almost certainly pure re-export shims following the `reasons_lib/api.py` template:

```python
"""Backwards-compatibility shim — use ``from reasons.<mod>`` instead."""
from reasons.<mod> import *  # noqa: F401,F403
```

Expected count is "~30" — the exact number should be verified, not assumed.

### 2. `source module __all__`
For every shim, resolve its target (`reasons.<mod>`) and check:

```bash
python -c "import reasons.<mod> as m; print(getattr(m, '__all__', None))"
```

Two outcomes matter:
- **`__all__` present** → shim exports exactly that set. Anything else (including imported helpers like `pubsub`, `Storage`) stays out of the shim.
- **`__all__` absent** → shim exports **every non-underscore name defined *or imported*** in `reasons/<mod>.py`. For `reasons/api.py` this includes `copy`, `json`, `logging`, `os`, `re`, `sqlite3`, `sys`, `Callable`, `datetime`, `timezone`, `combinations`, `Path`, `Justification`, `pubsub`, `build_meta`, `Network`, `PROPOSALS_SCHEMA`, `Storage` — i.e. the shim's namespace is much wider than the intended API. That is a real finding, not noise.

### 3. Private names in source
Collect names starting with `_` in `reasons/<mod>.py`. In the visible excerpt of `reasons/api.py`:

| Symbol | ~Line | Kind |
|---|---|---|
| `_network_cache` | 34 | module global (annotated — the prompt's regex missed it) |
| `_is_visible` | 37 | function |
| `_get_node_tags` | ~45 | function |
| `_filter_by_tags` | ~62 | function |
| `_resolve_namespace` | ~87 | function |
| `_with_network` | ~99 | function (contains nested `class _Ctx`) |
| `_pg_dispatch` | ~183 | function |

These are **not** re-exported by `from reasons.api import *` because star-import always excludes leading-underscore names when `__all__` is absent. The table should flag this per-shim: any shim whose *public* row lists these names has a bug; any test that imports them via the shim will `AttributeError`.

### 4. Tests that patch through the shim
This is the column people get wrong. `from reasons.api import *` **rebinds** names in `reasons_lib/api.py.__dict__` — it does *not* create an alias to `reasons.api`. Therefore:

- `patch("reasons_lib.api.Storage")` mutates `reasons_lib/api.py`'s namespace.
- `init_db` is defined in `reasons/api.py`; its `__globals__` is `reasons.api.__dict__`. It will still resolve `Storage` from `reasons.api`, **not** from the shim.

So patching *through* the shim is a no-op for any function defined in the source module. Any test that does this is a latent bug — collect them for the column by:

```bash
grep -rnE '(patch|setattr|monkeypatch\.setattr)\(\s*["'"'"']?[^,)]*reasons_lib' tests/
grep -rnE 'from reasons_lib.*import' tests/            # who imports from the shim at all
grep -rnE 'patch(\.object)?\(\s*[^,)]*,\s*["'"'"']_(get_node_tags|is_visible|with_network)' tests/
```

Tests should patch `reasons.api.Storage` / `reasons.api.pubsub` / `reasons.api.Network` (the source module) for the patch to affect the function under test. This is why the column is worth building at all.

## Table schema

```
| shim path            | source module | source __all__       | private names in source              | tests patching via shim          |
|----------------------|---------------|----------------------|--------------------------------------|----------------------------------|
| reasons_lib/api.py   | reasons.api   | (absent — verify)    | _network_cache(L34), _is_visible(L37), _get_node_tags, _filter_by_tags, _resolve_namespace, _with_network, _pg_dispatch | (0 trustworthy; re-run) |
| reasons_lib/<mod>.py | reasons.<mod> | …                    | …                                    | …                                |
```

The representative row is only half-populated: `reasons_lib/api.py` → `reasons.api` is confirmed by the 3-line shim body; the `__all__` value and the test-patch column need a re-run.

## What to re-run (exact commands)

```bash
# 1. Enumerate the ~30 shims
wc -l reasons_lib/*.py | sort -n | head -40

# 2. Extract each shim's target
grep -Hn '^from reasons' reasons_lib/*.py

# 3. Find __all__ in every source module targeted by a shim
for f in reasons_lib/*.py; do
  tgt=$(grep -m1 -oE 'from reasons[. ]([a-zA-Z0-9_]+)' "$f" | awk '{print $2}' | tr -d . )
  echo "== $f -> reasons.$tgt"
  grep -n '^__all__' "reasons/$tgt.py" || echo "   (no __all__)"
done

# 4. Private names per source module (handles annotations)
grep -nE '^def _|^class _|^[a-zA-Z_][a-zA-Z0-9_]*:[^=]*=|^_[a-zA-Z0-9_]+ *=' reasons/*.py

# 5. Test-patch column
grep -rnE '(patch|setattr|monkeypatch\.setattr)[^)]*reasons_lib' tests/
grep -rnE 'from reasons_lib' tests/
```

## Topics to Explore

- [file] `reasons_lib/api.py` — The canonical 3-line shim; every other `reasons_lib/*.py` should look like this, so it's the template for the whole table
- [function] `reasons/api.py:_with_network` — Context manager that owns `Storage`/`Network` lifecycle; a prime patching target, so any test that patches `Storage` via the shim silently won't affect it
- [general] `star-import-rebinding-semantics` — Why `from X import *` breaks `patch("shim.name")`; essential to interpret the test-patch column correctly
- [general] `__all__-absent-set` — What `reasons_lib.api` actually exposes when `reasons/api.py` has no `__all__` (all imported helpers included); run `dir(reasons_lib.api)` and diff against `dir(reasons.api)`
- [file] `reasons/__init__.py` — Defines `Justification`, `Node`, `Nogood`; these are re-imported into `reasons.api` and thus leak through the shim too, so the table's "public names" set is larger than the source API suggests

## Beliefs

- `reasons_lib/api.py`-is-a-shim — `reasons_lib/api.py` is a 3-line backwards-compatibility shim whose only body is `from reasons.api import *` (line 3).
- `reasons/api.py`-is-the-source — `reasons/api.py` is the real implementation (5886 lines) containing `init_db`, `_with_network`, and `_pg_dispatch`.
- `shim-rebinding-blocks-patch — Patching a name on `reasons_lib.api` does **not** affect `reasons.api` because `from … import *` rebinds names locally rather than aliasing the source module.
- `no-all-means-imports-leak` — With no `__all__` in `reasons/api.py`, the shim re-exports every non-underscore name, including imported modules like `pubsub`, `Storage`, and `Network`.
- `private-names-omitted-by-star-import` — Star-import always excludes leading-underscore names, so `_is_visible`, `_get_node_tags`, `_with_network`, and `_network_cache` (line 34) are absent from `reasons_lib.api`.
- `0-matches-are-query-artifacts` — All four `total_count: 0` results are inconsistent with the observed `reasons/api.py` content (which visibly contains `_network_cache` and multiple `def _…` functions), so they must be re-run before contributing to the table.
