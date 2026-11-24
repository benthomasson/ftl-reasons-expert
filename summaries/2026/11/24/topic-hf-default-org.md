---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Corresponds to `tests/test_hf_default_org.py`; explains how a bare `repo_id` (no slash) is resolved to a full `org/name`.

# How a bare `repo_id` becomes `org/name`

## Where resolution lives

The single source of truth for bare-name expansion is `resolve_repo_id` in `reasons/hf.py` (defined at `reasons/hf.py:27`, body through `reasons/hf.py:46`). The default org constant lives one line at the top:

```python
DEFAULT_HF_ORG = "EEM-Hub"   # reasons/hf.py:11
```

`_parse_repo_id` (`reasons/hf.py:49`) is a thin pass-through that just calls `resolve_repo_id`, and it exists so callers that used to have a separate parser keep a stable name.

## The resolution algorithm

`resolve_repo_id` runs a short, ordered series of checks (`reasons/hf.py:37–46`):

1. **Normalize input** — `repo_id = repo_id.strip().rstrip("/")` (`reasons/hf.py:37`). This strips leading/trailing whitespace and any single trailing slash, which is why the tests `test_whitespace_stripped` and `test_trailing_slash_stripped` pass (both yield `EEM-Hub/ddia-expert`).
2. **URL branch** — If the string starts with `http://` or `https://` (`reasons/hf.py:38`), it is split on the literal `"huggingface.co/"` (`reasons/hf.py:39`). If the split found that host, the suffix (`org/repo`) is returned (`reasons/hf.py:41`). Otherwise it raises `ValueError("Not a HuggingFace URL: ...")` (`reasons/hf.py:42`). This is what `test_url_parsed` and `test_non_hf_url_raises` lock in.
3. **Explicit-org branch** — If the (normalized) string contains a `/`, it is returned unchanged (`reasons/hf.py:43–44`). That is the `test_explicit_org_unchanged` case and also why `test_env_var_does_not_affect_explicit_org` passes: the env var is never consulted once a slash is present.
4. **Bare-name branch** — Otherwise the default org is looked up per call:
   ```python
   default_org = os.environ.get("REASONS_HF_ORG", DEFAULT_HF_ORG)   # reasons/hf.py:45
   return f"{default_org}/{repo_id}"                                # reasons/hf.py:46
   ```
   `REASONS_HF_ORG` overrides `EEM-Hub` at *call time*, not import time, which is what `test_env_var_override` (via `monkeypatch.setenv`) exercises. Without the env var, a bare name such as `"ddia-expert"` becomes `"EEM-Hub/ddia-expert"` — the assertion in `test_bare_name_gets_default_org`.

Because the check for `/` comes *before* the default-org lookup, the function is **idempotent**: `resolve_repo_id("EEM-Hub/foo")` returns `"EEM-Hub/foo"` unchanged. This matters because the function is called at multiple layers.

## Where it gets called — and why idempotency matters

Resolution is not done once at the entry point; it is re-applied inside every HuggingFace operation:

- `download_network` resolves via `_parse_repo_id` before building the `resolve/main/network.json` URL.
- `create_repo` calls `resolve_repo_id(repo_id)` and then splits the result with `parsed_id.split("/", 1)` to derive the `organization` and `name` fields posted to `HF_BASE/api/repos/create`. If no slash were guaranteed, that split would fail — so this call is also a *safety* check: by the time `create_repo` splits, `resolve_repo_id` has guaranteed exactly one slash exists. This is what `test_creates_repo` verifies (`body["organization"] == "EEM-Hub"`, `body["name"] == "test-eem"`).
- `upload_file` resolves again before building `/api/models/{parsed_id}/upload/main/{path_in_repo}`.

At the API layer, `publish_hf` (`reasons/api.py:2362–2397`) does its own `parsed_id = resolve_repo_id(repo_id)` at `reasons/api.py:2371`, passes `parsed_id` down, and echoes it back as `"repo_id": parsed_id` at `reasons/api.py:2397`. `import_hf` (`reasons/api.py:2313–2334`) calls `download_network(repo_id, ...)` and then calls `_parse_repo_id(repo_id)` at `reasons/api.py:2334` purely to populate the result's `repo_id` field. Because every layer applies the same idempotent function, it doesn't matter whether an intermediate step already resolved the value.

The CLI mirrors this contract in its help text: both `publish` and `pull` say *"EEM name (bare name defaults to EEM-Hub/name)"* (`reasons/cli.py:3461`, `reasons/cli.py:3468`), and the pull help mentions *"bare name, user/repo, or URL"* (`reasons/cli.py:3454`).

## What `tests/test_hf_default_org.py` is pinning down

The file is split into four test classes, and `TestResolveRepoId` is the direct test suite for the algorithm:

| Test | Pins down |
|---|---|
| `test_bare_name_gets_default_org` | The core contract: bare name → `EEM-Hub/name`. |
| `test_explicit_org_unchanged` | The `/` check short-circuits defaulting. |
| `test_url_parsed` | URL → `org/repo` via the `huggingface.co/` split. |
| `test_trailing_slash_stripped` | `rstrip("/")` runs *before* the slash check. |
| `test_whitespace_stripped` | `.strip()` runs first. |
| `test_env_var_override` | `REASONS_HF_ORG` is honored for bare names. |
| `test_env_var_does_not_affect_explicit_org` | Env override is scoped to the bare-name branch only. |
| `test_non_hf_url_raises` | Non-HF URLs fail loudly with `ValueError`. |
| `test_default_org_constant` | `DEFAULT_HF_ORG == "EEM-Hub"` — no accidental constant drift. |

The behavior is then re-verified end-to-end:
- `TestPublishHf.test_publishes_three_files` asserts `result["repo_id"] == "EEM-Hub/test-eem"`.
- `TestPublishHf.test_explicit_org` asserts the explicit `myuser/my-eem` survives.
- `TestPullCommand.test_pull_with_bare_name` asserts `result["repo_id"] == "EEM-Hub/ddia-expert"` after import.

Other test classes (`TestCreateRepo`, `TestUploadFile`) verify the *downstream consumers* of the resolved ID — URL construction, org/name split in the create payload, auth header, and the error mapping (401 → "Authentication failed", 409 → silent success, other → `HTTP {code}` / "Failed to upload").

## Subtleties worth knowing

- **`repo_id.split("huggingface.co/", 1)`** matches anywhere in the URL, not just the host. So `https://evil.example.com/huggingface.co/org/repo` would parse to `org/repo`. This is not currently tested — a candidate edge case for `test_non_hf_url_raises` to be hardened.
- **URL-prefix entries** like `https://huggingface.co/datasets/org/repo` would return `datasets/org/repo` (three segments), which would then be mishandled by `create_repo`'s `split("/", 1)`. Again, not in the test surface.
- **No re-validation of the resolved ID**: `resolve_repo_id` doesn't check that `org` and `name` are non-empty or valid HF identifiers. The tests only assert on string output.
- **`_parse_repo_id` is a compatibility alias**, not an independent parser — `reasons/api.py:2333` imports it explicitly, so changes to `resolve_repo_id` propagate to both `import_hf` and `publish_hf`.

## Topics to Explore

- [function] `reasons/hf.py:resolve_repo_id` — The core resolver; understand each branch before changing defaulting behavior.
- [function] `reasons/api.py:publish_hf` — Shows how the resolved ID flows through create → upload → returned result.
- [function] `reasons/api.py:import_hf` — Shows the download path and where `_parse_repo_id` is used just for reporting.
- [file] `reasons/cli.py` — CLI argument wiring for `pub`/`pull`, including the `--token`/`--private` flags and help text.
- [general] `hf-url-parsing-edge-cases` — The `huggingface.co/` split has untested inputs (non-HF hosts containing the substring, `/datasets/` prefixed URLs).
- [file] `reasons_lib/hf.py` — Backwards-compat shim; verify no stale import paths bypass the new resolver.

## Beliefs

- `bare-repo-id-requires-default-org` — A bare `repo_id` (no `/`) can only be sent to HuggingFace after `resolve_repo_id` prepends `DEFAULT_HF_ORG` (or `REASONS_HF_ORG`).
- `default-org-is-EEM-Hub` — `DEFAULT_HF_ORG` equals `"EEM-Hub"` and is asserted by `test_default_org_constant`.
- `resolve-repo-id-is-idempotent` — `resolve_repo_id` returns an already-resolved `org/name` unchanged because the `/` check precedes defaulting.
- `explicit-org-precedes-env-override` — `REASONS_HF_ORG` has no effect when the input already contains a slash, as verified by `test_env_var_does_not_affect_explicit_org`.
- `create-repo-requires-resolved-id` — `create_repo` calls `parsed_id.split("/", 1)` (`reasons/hf.py`) and therefore depends on `resolve_repo_id` having guaranteed a single slash.
- `non-hf-url-causes-value-error` — Any URL starting with `http(s)://` that does not contain `huggingface.co/` raises `ValueError("Not a HuggingFace URL: ...")`.
- `_parse_repo_id-delegates-to-resolve_repo_id` — The private helper is a pass-through; all defaulting logic lives in `resolve_repo_id`.
