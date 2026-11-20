---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How `import *` breaks `monkeypatch`, `__all__` propagation, and private-name access; useful for auditing the other ~30 shims in `reasons_lib/`.

# How `import *` breaks `monkeypatch`, `__all__` propagation, and private-name access

## 1. What the shims look like right now

Every shim in `reasons_lib/` follows the same three-line template:

- `reasons_lib/__init__.py:3` — `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py:3` — `from reasons.api import *  # noqa: F401,F403`

The observed source module `reasons/api.py` is 5,886 lines and its first 120 lines show:
- A handful of stdlib imports (`copy`, `json`, `logging`, `os`, `re`, `sqlite3`, `sys`, `Path`, `datetime`, `timezone`, `combinations`, `Callable`).
- Private helpers `_is_visible`, `_get_node_tags`, `_filter_by_tags`, `_resolve_namespace`, `_with_network`.
- A private module-level cache `_network_cache: dict[str, tuple[float, Network]] = {}`.
- Public constants/objects such as `DEFAULT_DB = "reasons.db"`, `logger`, `Storage`, `Network`, `Justification`, `pubsub`, `build_meta`, `PROPOSALS_SCHEMA`.

The searches `find_all_exports_in_reasons`, `find_all_exports_in_reasons_lib`, `find_monkeypatch_usage`, and `find_private_name_access_on_shims` all returned **0 matches**. The first two tell us there is no `__all__` anywhere in scope. The last two tell us there is no *in-repo* evidence of these failure modes yet — but they don't tell us anything about external consumers or deleted tests.

> ⚠️ Audit signal: `find_star_imports_in_reasons_lib` reported **0 matches** for `import \*`, yet both visible shim files clearly contain `import *`. Either the search excluded `__init__.py`/applied a path filter, or the tool result is unreliable. Do not trust a "0 matches" for star imports until you re-run without file filters.

## 2. The mechanic: `from X import *` creates *new bindings*, not *aliases to a module*

`from reasons.api import *` runs once at shim-import time and executes roughly:

```python
for name in (reasons.api.__all__ if hasattr(reasons.api, "__all__")
             else (n for n in vars(reasons.api) if not n.startswith("_"))):
    globals()[name] = getattr(reasons.api, name)
```

Three immediate consequences:

- **Snapshot, not live view.** `reasons_lib.api.Storage is reasons.api.Storage` is `True` at import time, but `reasons.api.Storage = Fake` **does not** update `reasons_lib.api.Storage`. Rebinding one module's name does not propagate to the other.
- **Shallow.** The function objects in `reasons_lib.api` are the *same* objects as in `reasons.api` (same `__code__`, same `__globals__` pointing at `reasons.api.__dict__`). That's why calling them still works — but also why patching the shim cannot reach inside them.
- **Dunder names are skipped.** `__all__`, `__name__`, `__doc__`, and `__file__` are not imported by `*`. So `reasons_lib.api` has **no `__all__` attribute** even if `reasons.api` one day gains one.

## 3. `__all__` propagation is broken in both directions

Because `reasons.api` defines no `__all__`, the star import falls back to "every non-underscore top-level name". That surfaces two problems at once:

**Downstream leak.** `reasons_lib.api` ends up with attributes like `json`, `logging`, `os`, `re`, `sqlite3`, `sys`, `Path`, `datetime`. Those are implementation details of `reasons.api`, not its public API. Somebody doing `from reasons_lib.api import *` now pulls all that in — that's the `F403` warning the `# noqa` is silencing. `F403` is literally *"'from module import *' used; unable to detect undefined names"*; suppressing it means the linter cannot verify that the shim surface matches the source surface.

**Upstream contract loss.** Even if `reasons.api.__all__` is added tomorrow, `reasons_lib.api` still doesn't get its own `__all__`. Tools that read `__all__` for introspection, doc generation, `deprecation` markers, or stub generation will see `reasons.api` with a curated surface but `reasons_lib.api` with none — a divergence that will silently drift.

**One more edge case in `reasons_lib/__init__.py`.** `from reasons import *` imports names from `reasons/__init__.py`, not from `reasons.api`. If `reasons/__init__.py` re-exports `api` as a submodule name, then `reasons_lib.api` becomes a *module attribute* pointing at `reasons.api`, which will be overwritten the first time anyone does `import reasons_lib.api`. That ordering-dependence is exactly the kind of thing that bites when shims are audited in bulk.

## 4. Private-name access is silently removed (and silently widened)

`from reasons import *` skips every name starting with `_`. So:

- `_is_visible`, `_get_node_tags`, `_filter_by_tags`, `_resolve_namespace`, `_with_network`, and `_network_cache` **do not exist** on `reasons_lib.api` or `reasons_lib`.
- Any legacy caller that did `from reasons_lib.api import _get_node_tags` or `reasons_lib.api._network_cache` will now get `ImportError`/`AttributeError`. The in-repo search is 0, but that only covers *this* repo.
- Conversely, private names are not the only thing *removed*; because `__all__` is absent, some arguably-private-but-non-underscore names (`logger`, `DEFAULT_DB`) are *exposed*, widening the surface the shim advertises.

If a shim was supposed to preserve a private name (some do, e.g. a helper kept for pickling or test introspection), star import will never carry it. The fix must be explicit (`from reasons.api import _get_node_tags`) or a module-object alias (below).

## 5. `monkeypatch` is silently a no-op through the shim

This is the failure with the worst failure mode: tests keep passing while proving nothing.

```python
from reasons_lib import api as shim_api

def test_get_node_hits_fake_storage(monkeypatch):
    monkeypatch.setattr(shim_api, "Storage", FakeStorage)
    shim_api.get_node("n1")   # BUG: still uses reasons.api.Storage
```

What happens:

- `shim_api.get_node` is the *same function object* as `reasons.api.get_node`.
- That function resolves `Storage`, `DEFAULT_DB`, `_with_network`, etc. via `reasons.api.__dict__`, not through the shim.
- `monkeypatch.setattr(shim_api, "Storage", FakeStorage)` only rebinds a key in `reasons_lib.api.__dict__`. The real function never looks there.
- Result: the test exercises the real `Storage`, likely on `reasons.db`. Either it fails for a real reason (DB missing) or, worse, it passes against a real DB and deceives everyone.

Three corollary gotchas to keep in mind while auditing:

1. **Patching the underlying name on the shim is doubly wrong.** `monkeypatch.setattr("reasons_lib.api.DEFAULT_DB", ":memory:")` is a no-op for `reasons.api` and *also* wouldn't affect default-argument binding, which is evaluated at function-definition time.
2. **Patching private helpers through the shim is impossible.** `monkeypatch.setattr(shim_api, "_is_visible", ...)` raises `AttributeError` because the name was never exported.
3. **Reload drift.** `importlib.reload(reasons.api)` freshens `reasons.api.__dict__` but leaves `reasons_lib.api` holding stale references. Long-running REPLs and integration tests that reload modules will diverge.

The only ways to preserve patchability are (a) patch the *real* module (`reasons.api`) in tests, (b) delete the physical shim file and install a module alias in `reasons_lib/__init__.py` (`sys.modules["reasons_lib.api"] = reasons.api`), or (c) re-export the module itself (`from reasons import api as api`) and accept that `reasons_lib.api.__name__` will read `reasons.api`. Any of those is a real architectural change; the current three-line shim gives none of them.

## 6. Concrete audit checklist for the other ~30 shims

For each `reasons_lib/<mod>.py`:

1. **Does `reasons/<mod>.py` define `__all__`?** — If no, the shim surface is "all public names", which includes stdlib imports. Compare `dir(reasons.<mod>)` minus underscore names to what the shim should advertise.
2. **Does the shim define its own `__all__`?** — All observed shims say no (`find_all_exports_in_reasons_lib` = 0). Adding `from reasons.<mod> import __all__ as __all__` (or an explicit list) is the minimum fix for introspection.
3. **Are any `_`-prefixed helpers part of the old contract?** — They cannot cross a star import. List them, then either add explicit imports or move to a module alias.
4. **Does any test or docstring use `monkeypatch.setattr("reasons_lib.<mod>.…")`?** — In-repo count is 0, but re-run without scope filters and include `tests/`, `docs/`, `conftest.py`.
5. **Does any code do `import reasons_lib; reasons_lib.<mod>.<name>`?** — Only works if `reasons_lib/__init__.py` imported the submodule. It does not (only `from reasons import *`). This is a latent `AttributeError` waiting on someone who imports the package but not the submodule.
6. **Is `noqa: F401,F403` masking drift?** — `F403` is precisely the "undefined names undetected" warning. If a source name is renamed or deleted, the shim will drop it silently at runtime. Replace the star import with an explicit import list, or add a smoke test that asserts `set(dir(reasons_lib.<mod>)) ⊇ <expected names>`.

## 7. What the observations do **not** cover

Before generalizing to all ~30 shims, gather:

- The other ~28 shim files (only `__init__.py` and `api.py` were read).
- The source names actually expected to be public from each `reasons/<mod>.py`; only 120 of 5,886 lines of `reasons/api.py` were inspected, and only for private helpers.
- Test-suite grep for `monkeypatch.setattr` and `from reasons_lib` without a path filter (current count is 0, but the star-import search's 0 result is demonstrably wrong, so the monkeypatch count is suspect too).
- `reasons/__init__.py` contents — determines whether `import reasons_lib; reasons_lib.api` resolves at all.
- Whether downstream consumers / published artifacts pickle objects by `reasons_lib.<mod>.<name>` (pickle stores `__module__` + `__qualname__`; old pickles that reference `reasons_lib.api.Foo` will resolve fine today only because the name is re-exported — a fragile guarantee).

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm no `__all__`, no submodule imports, and evaluate whether `sys.modules` aliasing should replace `from reasons import *`.
- [file] `reasons/api.py` — Scan the remaining ~5,760 lines for public names that the shim is (or should be) exposing, and check whether any non-underscore helpers are being unintentionally leaked.
- [function] `reasons/api.py:_with_network` — Central lifecycle helper; understanding what it caches (`_network_cache`) tells you why the shim can't restore monkeypatch semantics without aliasing the module object.
- [general] `shim-audit-matrix` — Build a table of {shim path, source module `__all__`, private names in source, tests that patch through shim} for all ~30 shims; the 0-match results in this prompt are not trustworthy until re-run without filters.
- [general] `star-import-replacement-strategy` — Decide between explicit imports, `__all__` mirroring, and `sys.modules` aliasing; each has different implications for type checkers, Sphinx autosummary, and pickle compatibility.

## Beliefs

- `shim-files-contain-star-imports` — Both observed shims (`reasons_lib/__init__.py:3`, `reasons_lib/api.py:3`) use `from reasons[.api] import *`, so the `find_star_imports_in_reasons_lib` 0-match result is unreliable and must be re-run.
- `reasons-api-has-no-all` — Neither `reasons/api.py` nor `reasons_lib` defines `__all__`, so star import falls back to "all non-underscore names", leaking `json`, `logging`, `os`, `sqlite3`, `Path`, etc. into the shim namespace.
- `star-import-excludes-private-helpers` — `_is_visible`, `_get_node_tags`, `_filter_by_tags`, `_resolve_namespace`, `_with_network`, and `_network_cache` are defined in `reasons/api.py` (within the first 120 lines) and are not re-exported by `from reasons.api import *`.
- `__all__-is-not-propagated-by-star` — `__all__` starts with `__`, so `from module import *` never copies it; `reasons_lib.api.__all__` will remain absent even if `reasons.api.__all__` is added.
- `monkeypatch-on-shim-is-a-no-op` — `monkeypatch.setattr(reasons_lib.api, "Storage", Fake)` cannot affect `reasons.api` functions, because those functions resolve `Storage` through their own module globals (`reasons.api.__dict__`).
- `noqa-suppresses-undefined-name-detection` — The `# noqa: F401,F403` comments explicitly suppress `F403` ("unable to detect undefined names"), meaning the shim's surface can drift silently whenever `reasons/api.py` renames or removes a public name.
