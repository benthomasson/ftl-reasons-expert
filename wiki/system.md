# system

[Back to index](index.md)

The ftl-reasons system achieves its expressive power through a remarkably small set of primitives. Three foundational properties — all currently held — characterize the architecture: semantic minimality, a single reversible defeat mechanism, and consistent artifact identification. Together they describe a system where everything from monotonic truth maintenance to non-monotonic dialectics emerges from uniform foundations, and where every artifact the system produces is as addressable as anything a user creates.

## Semantic Minimality

The most fundamental architectural claim is that the entire TMS derives from a minimal set of uniform primitives with no additional machinery required (`system-semantics-are-minimal-and-complete`). On the monotonic side, truth maintenance uses emergent rules — disjunction over conjunction for combining justifications, and premise-from-absence for grounding beliefs without antecedents. On the non-monotonic side, all defeat mechanisms reduce to a single reversible outlist primitive. These two halves together form a complete semantic foundation: monotonic truth propagation and non-monotonic belief revision share the same minimal basis, rather than being bolted together from independent subsystems.

This belief depends jointly on the uniformity of truth semantics (`[truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)`) and the universality of the outlist primitive (`non-monotonic-system-is-single-reversible-primitive`), establishing that minimality holds across both halves of the reasoning engine.

## The Outlist as Universal Defeat Primitive

Every non-monotonic feature in ftl-reasons — challenges, kill-switches, supersession, and recursive dialectical structures — is built on a single primitive: the outlist entry (`non-monotonic-system-is-single-reversible-primitive`). There is no dedicated machinery for any particular defeat pattern. A challenge is an outlist entry; supersession is an outlist entry; a dialectical rebuttal is an outlist entry applied recursively. This uniformity means that understanding one defeat mechanism is understanding all of them.

Crucially, the outlist primitive is inherently reversible. Retracting the defeating belief restores the defeated one, with no special undo logic required. This reversibility propagates upward: because every defeat pattern reduces to outlist entries, every defeat pattern is automatically reversible. The system can retract any reasoning step and return to a prior consistent state through the same mechanism that created the defeat in the first place. This property underpins the broader claim that belief revision is both comprehensive and fully reliable (`[belief-revision-is-comprehensive-and-minimal](revision.md#belief-revision-is-comprehensive-and-minimal)`, `[belief-revision-is-fully-reliable](revision.md#belief-revision-is-fully-reliable)`), and that the reasoning engine as a whole is deterministic and reversible (`[reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible)`).

## Artifact Identification

System-generated artifacts maintain consistent, referenceable identification schemes that are as stable and addressable as user-created beliefs (`system-artifacts-maintain-consistent-identification`). This applies to two categories of automatically produced artifacts: challenge nodes, which receive deterministic auto-generated IDs (`[challenge-id-auto-generation](other.md#challenge-id-auto-generation)`), and contradiction records (nogoods), which are unconditionally recorded with stable identifiers (`[nogood-resolution-maintains-consistent-ids](other.md#nogood-resolution-maintains-consistent-ids)`).

The consequence is that the system's own reasoning history — its challenges, its detected contradictions, its self-corrections — is fully traceable. Every mutation the system performs produces artifacts that can be referenced, queried, and audited after the fact (`[mutations-achieve-full-traceability](other.md#mutations-achieve-full-traceability)`, `self-correction-produces-referenceable-artifacts`). System history is consistently referenceable across all artifact types (`system-history-is-consistently-referenceable`), making the boundary between user-authored and system-generated content transparent rather than opaque.

## Architectural Significance

These three properties are mutually reinforcing. Semantic minimality ensures there are few primitives to reason about. The universal outlist mechanism ensures that the non-monotonic half of those primitives is uniform and reversible. Consistent artifact identification ensures that everything the system produces from those primitives — including its own internal reasoning steps — remains addressable and auditable. Together they support the higher-level conclusion that the system's semantics are both minimal and operationally deterministic (`[semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism)`), and that its semantic foundations and revision mechanisms share the same minimal base (`[semantics-and-revision-share-minimal-foundations](revision.md#semantics-and-revision-share-minimal-foundations)`).
