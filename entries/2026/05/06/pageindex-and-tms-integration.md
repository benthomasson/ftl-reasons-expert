# PageIndex + TMS: Complementary Tools for Knowledge Management

## What Each Tool Does

### PageIndex
A vectorless, reasoning-based RAG system. Takes PDFs and markdown documents, builds hierarchical tree indexes (like a table of contents), and lets LLMs reason their way to relevant sections. Replaces vector similarity search with structural navigation. Documents are immutable once indexed — PageIndex is purely a retrieval system.

Key properties:
- Hierarchical tree structure with section titles, page ranges, summaries
- Precise source location (document ID, section path, page number)
- No concept of beliefs, truth values, or state changes
- Achieves 98.7% accuracy on FinanceBench (financial document Q&A)

### ftl-reasons (TMS)
A Truth Maintenance System that tracks beliefs, their justifications, and dependencies. When an assumption is retracted, all beliefs that depend on it automatically cascade OUT. Supports non-monotonic reasoning (beliefs that hold only when certain conditions are absent), contradiction detection, and dependency-directed backtracking.

Key properties:
- Dynamic belief state with retraction cascades
- Justification chains tracking why each belief is held
- Contradiction detection across the belief network
- No built-in document retrieval or source material navigation

## The Gap Between Them

PageIndex knows where information lives in documents but doesn't track whether it's still true or how claims relate to each other. ftl-reasons tracks what we believe and why but has weak connections back to source material — beliefs carry a `source` field (file path) and `source_hash` (for staleness), but this is a flat pointer with no structural awareness of where in the document the claim came from.

The expert-agent-builder pipeline bridges this gap today, but crudely: it summarizes entire documents, proposes beliefs from the summaries, and records the source document path. The connection between belief and source is a file-level pointer, not a section-level one.

## Integration Opportunities

### 1. Section-Level Belief Extraction

**Current pipeline:** Document → LLM summary → propose beliefs → accept → reasons.db
**With PageIndex:** Document → PageIndex tree → per-section belief extraction → accept → reasons.db

Each belief would carry structural provenance:
```
source: "doc-uuid/section-1.2.3"
source_url: "pages 45-52"
```

This is more precise than "this Google Doc" and enables:
- Tracing any belief back to its exact source section
- Re-reading only the relevant pages when verifying a belief
- Detecting when a specific section has been updated without re-processing the entire document

### 2. Staleness Verification via Source Navigation

The staleness problem for business planning TMS (see `belief-staleness-in-business-planning.md`) is partly a retrieval problem. When a belief needs verification, someone has to find the relevant source material and check whether it still supports the claim.

PageIndex could automate the retrieval step:
1. Look up the belief's source section in the PageIndex tree
2. Fetch the current content of those pages
3. Send both the belief text and the current source text to an LLM
4. Ask: "Does this source material still support this claim?"

This doesn't eliminate the need for real-world verification (some beliefs require stakeholder conversations, not document checks), but it handles the subset of beliefs that are grounded in documents — which is all of them in the expert-agent-builder pipeline.

### 3. Cross-Document Reasoning

PageIndex can index multiple documents into a single workspace. Combined with TMS, this enables:
- Extract beliefs from multiple documents across departments
- Use ftl-reasons to track dependencies and contradictions across documents
- When a contradiction is found, use PageIndex to navigate back to the source sections in both documents
- Present the conflicting source material side-by-side for resolution

This is particularly relevant for the redhat-expert use case where beliefs span 9 business clusters sourced from different document sets.

### 4. Incremental Document Updates

When a source document is updated:
1. PageIndex re-indexes the document, producing a new tree
2. Diff the old and new trees to identify which sections changed
3. Flag beliefs sourced from changed sections as potentially stale
4. Use the new section content to verify or retract affected beliefs

This is `check-stale` for documents — but instead of comparing file hashes (binary changed/unchanged), it identifies which specific sections changed and which beliefs are affected.

### 5. Evidence-Grounded Derivation

When ftl-reasons derives new beliefs from existing ones, the derivation is purely logical — "if A and B, then C." PageIndex could add an evidence step:
1. Derive a candidate belief from existing beliefs
2. Search the indexed documents for supporting or contradicting evidence
3. If evidence is found, attach the source section as provenance
4. If contradicting evidence is found, flag the derivation as suspect

This grounds derived beliefs in source material rather than pure logical inference, which is especially important for business planning where logical derivations may miss real-world constraints that are documented somewhere but not yet in the belief network.

## Architecture Sketch

```
Documents (Google Drive, PDFs, etc.)
    │
    ▼
PageIndex (hierarchical indexing)
    │
    ├──► Section-level extraction ──► expert-agent-builder ──► reasons.db (TMS)
    │                                                              │
    ├──► Staleness verification ◄──────────────────────────────────┤
    │                                                              │
    └──► Evidence retrieval ◄── derive / contradictions ◄──────────┘
```

PageIndex handles the document side (indexing, retrieval, navigation). ftl-reasons handles the knowledge side (beliefs, justifications, dependencies, contradictions). expert-agent-builder is the pipeline that connects them.

## What Would Need to Change

### In expert-agent-builder
- Use PageIndex to index documents before extraction
- Extract beliefs per-section instead of per-document
- Store section-level provenance in belief source metadata

### In ftl-reasons
- Extend `source` metadata to support structured provenance (document ID + section path + page range)
- Extend `check-stale` to check section-level changes via PageIndex API
- Add a `verify` command that retrieves source material and asks an LLM whether the belief is still supported

### In PageIndex
- Workspace sharing or API access from other tools
- Section-level diff between old and new document versions
- Integration with external knowledge systems (output beliefs, not just answers)

## Open Questions

- Is PageIndex's LLM-based indexing cost-effective for large document sets (hundreds of docs)?
- How well does section-level provenance survive document reformatting (same content, different page numbers)?
- Should the TMS store the source text snapshot at extraction time for diff comparison, or always re-fetch from PageIndex?
- Does the hierarchical structure add value for flat documents (emails, memos) vs. structured ones (reports, specs)?
