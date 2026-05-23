# HuggingFace as a Distribution Channel for External Epistemic Memory

## Context

The ftl-reasons belief network (`network.json`, `reasons.db`) is model-agnostic — any LLM that can do tool calling or use CLI commands can query and update it. This makes it analogous to a LoRA adapter, but with a critical difference: it is not tied to any specific model architecture or checkpoint. The knowledge lives in a structured, external substrate that any model can consume.

This raises the question of distribution. LoRA adapters and fine-tuned models are shared via HuggingFace. Belief networks should be shareable the same way.

## The Problem with expert-service

The existing distribution path is `expert-service` — a FastAPI server exposing the belief network via MCP tools. This works for live querying but requires the expert owner to:

- Run and maintain a server instance
- Configure MCP endpoints
- Set up database paths and authentication

This is a developer-oriented setup, not a community-friendly one. It does not scale to "here are my beliefs about Kubernetes, take them and use them."

## EEM Card Format

We designed a standardized `EEM_CARD.md` format (issue #148) modeled on HuggingFace model cards. The card is the `README.md` in the EEM repo — following the HuggingFace convention where the model card is always `README.md`.

### YAML frontmatter (machine-readable)

```yaml
schema_version: "1.0"
type: eem
project_name: eem-expert
domain:
  - external-epistemic-memory
  - truth-maintenance-systems
license: mit
base_network: null
source_repos:
  - benthomasson/ftl-reasons
beliefs_total: 49
beliefs_in: 49
beliefs_out: 0
premises: 19
derived: 30
nogoods: 0
generator: ftl-reasons/0.40.0
```

### Markdown body (human-readable)

Sections: EEM Details, Stats, Domain Coverage, How to Use, Key Beliefs, Sources, Files, Quality, Limitations, Authors, License.

This structure lets HuggingFace parse the metadata for search and filtering while providing humans with a readable overview of what the belief network contains and how to use it.

## First EEM Published

The `eem-expert` knowledge base (49 beliefs about EEM itself) was published to HuggingFace at [benthomasson/eem-expert](https://huggingface.co/benthomasson/eem-expert) on 2026-05-23. The upload was done with:

```bash
uvx hf upload benthomasson/eem-expert .
```

The `.gitignore` excludes `reasons.db` and its WAL files, so only portable files are uploaded: `network.json`, `README.md`, `CLAUDE.md`.

### Lessons from the first upload

- The `license` field must be lowercase (`mit`, not `MIT`) per HuggingFace's metadata validation
- SQLite WAL files (`reasons.db-shm`, `reasons.db-wal`) need explicit gitignore entries
- The `.claude/` session state directory should be excluded from uploads
- HuggingFace rendered the full EEM card cleanly — tables, code blocks, YAML frontmatter all work

## Import from HuggingFace (issue #149)

The next step is `reasons import-hf`, which would allow:

```bash
reasons import-hf benthomasson/eem-expert
reasons import-hf https://huggingface.co/benthomasson/eem-expert
```

This would download `network.json`, optionally initialize `reasons.db`, and import the beliefs. The command should work without `huggingface_hub` installed by falling back to raw HTTP download for public repos.

## Why This Matters

### Model-agnostic knowledge sharing

Unlike LoRA adapters (tied to one architecture) or fine-tunes (tied to one checkpoint), an EEM works with any model. Download a belief network about Kubernetes, import it, and query it with Claude, Gemini, Ollama, or any future model that supports tool calling.

### Local ownership

With HuggingFace distribution, the consumer downloads and owns their copy. They can:

- Extend it with new beliefs from their own observations
- Retract beliefs they disagree with (cascading to dependents)
- Derive new conclusions using `reasons derive`
- Merge multiple EEMs into one `reasons.db` — shared node IDs link up, contradictions surface as nogoods

### Composability

Multiple EEM repos can be imported into a single `reasons.db`. The TMS handles the merge: shared node IDs connect across networks, justification chains span both sources, and contradictions between networks are detected as nogoods. This is something you cannot do with LoRA adapters.

### Inspectability

Every belief has a justification chain. `reasons explain NODE` traces exactly why something is believed. `reasons retract NODE` removes it and propagates the consequences. This is the epistemic advantage over parametric knowledge — "how do you know that?" has a concrete, auditable answer.

## Related Issues

| Issue | Description | Status |
|-------|-------------|--------|
| #146 | Metadata section across all storage formats | Merged (PR #147) |
| #148 | Design EEM_CARD.md format for shareable belief networks | Open |
| #149 | Support importing belief networks from HuggingFace URLs | Open |
| #142 | Belief state change subscriptions/webhooks | Open |

## Architecture Summary

```
HuggingFace Hub                    Local
┌──────────────────┐              ┌──────────────────┐
│ benthomasson/    │  hf download │                  │
│   eem-expert     │ ───────────> │ network.json     │
│                  │              │                  │
│ README.md (card) │              │ reasons import   │
│ network.json     │              │ ───────────>     │
│ CLAUDE.md        │              │ reasons.db       │
└──────────────────┘              │                  │
                                  │ Any LLM agent    │
                                  │ queries via CLI  │
                                  └──────────────────┘
```
