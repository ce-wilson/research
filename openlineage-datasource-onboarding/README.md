# Datasource Onboarding on OpenLineage

Analysis of the OpenLineage data model as the basis for a manually-curated datasource
onboarding system: incremental metadata accrual, CSV/JSON sample extracts, cross-platform
replica provenance, credential separation, and ontology classification.

## Contents

| File | Audience |
|---|---|
| [`openlineage-datasource-onboarding.agent.md`](openlineage-datasource-onboarding.agent.md) | Machine review. 55 claims with stable IDs, confidence levels, and per-claim sources; design decisions referencing the claims they rest on; a reviewer verification checklist. |
| [`datasource-onboarding-openlineage.html`](datasource-onboarding-openlineage.html) | Human reading. Source/target mapping tables, the replica-provenance option comparison, a 16-stage end-to-end flow, traceability, and risks. |

## Sourcing

Primary source is the OpenLineage repository at commit `b995ee00c` (2026-08-28), release
`1.52.0`, spec `2-0-2` — principally `spec/OpenLineage.json`, `spec/OpenLineage.md`,
`spec/facets/*.json`, `proposals/1837/static_lineage.md`, and the Java and Python client
sources.

Corroborated by a five-angle web research pass (2026-08-29). Where web findings met the
repository at HEAD, the repository won. Claims are labelled `VERIFIED`, `INFERRED`, or
`PENDING`; nothing unverifiable is asserted as fact.

## Headline conclusions

1. **`DatasetEvent` is the registration primitive.** Static/design-time lineage — run-less
   events — is the designed mechanism for incremental curation. No synthetic job required.

2. **Facets are atomic and replace by name.** Event-level metadata is additive, but emitting
   a facet replaces the previous instance *entirely*. Sending a partial facet silently
   destroys its other fields. This is the governing producer constraint.

3. **Model a replica as two datasets and a real edge — never as symlinks.** The symlinks
   facet asserts two identifiers for *one* entity and collapses them into a single node,
   destroying the fact that the read happened against the replica rather than production.

4. **Credential separation is already in the identity grammar.** No namespace convention
   across 40+ platforms includes userinfo, so the namespace is a stable, credential-free
   vault key that survives rotation. This is convention plus tested implementation, not a
   normative prohibition.

## Open

Consumer/backend support is unresolved: which backends ingest static `DatasetEvent` /
`JobEvent`, whether any implement `_deleted` tombstones, and whether `TagsDatasetFacet`
renders in search. Closing it is a half-day spike against the candidate backend.
