---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The audit should use regex or AST for `from reasons import *`, not the literal word `"forwarding"`.

The audit target is a Python syntax pattern, not a prose label. `from reasons import *` does not contain the word `"forwarding"`, so an audit that searches for the literal word `"forwarding"` is checking a naming assumption rather than the actual import statement. In this codebase, that heuristic would be actively misleading:

- `forwarding_literal_occurrences.total_count` is `0` — there is no literal `"forwarding"` string in the observed matches.
- `wildcard_import_checks` finds `33` wildcard imports, including the exact line `reasons_lib/__init__.py:2`:
  ```python
  from reasons import *  # noqa: F401,F403
  ```
- `reasons_wildcard_import_occurrences.total_count` is also `0`, which appears to be a false negative from pattern escaping, literal-search semantics, or scan scope. The line is clearly present in `wildcard_import_checks`.
- `audit_references.total_count` is `0`, so the actual audit implementation is not visible in the observations. The recommendation is still clear: if such an audit exists elsewhere, it should match import syntax rather than the word `"forwarding"`.

The broader scan also shows many wildcard re-exports in the `reasons_lib/` shim, e.g. `reasons_lib/api.py:2`, `reasons_lib/ask.py:2`, and 31 others:
```python
from reasons.api import *  # noqa: F401,F403
from reasons.ask import *  # noqa: F401,F403
```
`CLAUDE.md` describes `reasons_lib/` as a “backwards-compat shim — re-exports from `reasons/`”, so these wildcard imports are likely intentional forwarding/re-export points. The audit should deliberately detect them.

A regex scan can work for a lightweight text check:
```python
re.compile(r"^\s*from\s+reasons\s+import\s+\*", re.MULTILINE)
```

But an AST audit is more robust because it ignores comments, `# noqa`, and strings:
```python
import ast

tree = ast.parse(source)
for node in ast.walk(tree):
    if isinstance(node, ast.ImportFrom) and node.module == "reasons":
        if any(alias.name == "*" for alias in node.names):
            report(node.lineno, "from reasons import *")
```

If the intent is to catch any wildcard re-export from the `reasons` package, including submodules, check `node.module == "reasons" or node.module.startswith("reasons.")`. The observed broad regex `import\s+\*` catches all `33` wildcard imports, but it would also catch unrelated modules unless the `from reasons ...` part is included.

The existing codebase already uses regex in places such as `reasons/api.py:3262`, `reasons/api.py:4878`, and `reasons/contradictions.py:63`, but the observed scan found no `ast.parse` usage. So regex is consistent with current style, while AST would be the more accurate choice for a syntax-sensitive audit.

What’s missing from the observations: the actual audit implementation. Since `audit_references.total_count` is `0` and the content of `.claude/skills/reasons/SKILL.md` is not included, I cannot point to the exact faulty audit function or line. But the data is sufficient to say that a literal `"forwarding"` search would miss the real wildcard import at `reasons_lib/__init__.py:2`.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Contains the exact `from reasons import *` import at line 2; verify why the shim re-exports the public API.
- [file] `reasons/api.py` — Shows existing regex usage around lines 3262, 4878, and 5125; compare regex-based parsing with an AST-based audit.
- [general] `audit-implementation-location` — Find where the audit is implemented; the observed scan found zero `audit` references.
- [general] `wildcard-import-policy` — Decide whether the audit should flag only `from reasons import *` or all `from reasons.* import *` re-exports.
- [file] `.claude/skills/reasons/SKILL.md` — The only skill file present in `.claude/skills/reasons/`; its content was not observed and may contain audit instructions.

## Beliefs

- `reasons-lib-init-has-wildcard-import` — `reasons_lib/__init__.py` line 2 contains `from reasons import *  # noqa: F401,F403`.
- `wildcard-import-scan-finds-33-sites` — The `import\s+\*` scan found 33 wildcard imports across `reasons_lib/*.py`.
- `literal-forwarding-search-is-empty` — Searching for the literal word `forwarding` returned zero matches in the observed codebase.
- `no-ast-parse-usage-observed` — The observed regex/AST scan found `re.compile`, `re.match`, and `re.search` usages but no `ast.parse` usages.
- `audit-references-are-not-observed` — Searching for `audit` returned zero matches, so the audit implementation is outside the provided observations.
