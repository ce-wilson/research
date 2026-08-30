# DataHub as a Manually-Curated Datasource Onboarding Model — Agent-Readable Analysis

| Field | Value |
|---|---|
| Repository | `datahub-project/datahub` (local clone, `C:/coding/projects/datahub`) |
| Commit analysed | `7d5cedd5f82e1e9608b08073dce7873ec26de248` (branch `master`, 2026-08-28) |
| Release tag containing HEAD | **None.** HEAD is in no tag. Nearest released lines: `v1.7.0` (2026-08-04, `7f81ccbfe2`) and `v1.6.0.1` (2026-08-13, `036f89a860`) |
| Analysis date | 2026-08-29 |
| Baseline compared against | OpenLineage repo `b995ee00c`, release `1.52.0`, spec `2-0-2` |
| Method | Local repo primary; web docs outranked. Claims labelled `VERIFIED` / `INFERRED` / `PENDING`. 22 agents, 1,474 tool calls, **312 claims raised — 229 confirmed, 47 corrected, 1 refuted** under adversarial re-verification against cited paths. |

> **Version drift warning.** Every path, line number, enum value, default and flag below is pinned to HEAD `7d5cedd5f8`. DataHub cuts releases from **release branches**, so HEAD is *ahead of* every released tag and some findings are not yet in any release. Release status is stated per claim.

---

## §0 Method correction — the prescribed maturity check is invalid for this repo

The brief prescribed `git tag --contains <sha>` to establish release status. **That test is structurally broken here and produces false negatives.**

- `git tag --contains 7d5cedd5f8` → empty.
- `git merge-base --is-ancestor v1.7.0 HEAD` → **false**. `v1.7.0` is not an ancestor of master.
- `v1.6.0.1` (2026-08-13) is dated **nine days later** than `v1.7.0` (2026-08-04). `v1.5.0.7` (2026-05-19) predates `v1.6.0` (2026-05-21).

Cause (`VERIFIED`): releases are cut on long-lived branches (`origin/releases/v1.7.0` etc.), branched from master at the `vX.Y.0rc1` commit and **never merged back**. Fixes reach a line only by explicit backport. Version ordering is therefore **not chronological**.

**Substitute method used throughout** — and it is important that this is *asymmetric*:

- `git merge-base --is-ancestor <sha> <tag>` returning **true** is sound positive evidence ("this shipped in that tag").
- Returning **false** proves nothing, because the fix may exist on the release branch as a separately-numbered backport commit with a different SHA.
- `git cat-file -e <tag>:<path>` establishes file presence in a tag directly.
- `docs/how/updating-datahub.md` is the release-status oracle; its `## Next` section is the unreleased marker.

**Trap encountered and corrected (the exact analogue of the prior run's failures).** One investigator declared PATCH-write validation "unreleased, sitting in `## Next`". The verifier refuted it: the entry sits inside an **HTML comment block spanning `docs/how/updating-datahub.md` L6–L35** — a commented-out *template* that also holds stale entries — while the live `## Next` begins at L40. The feature ships in `v1.7.0` (commit `c152d64`, #18579, 2026-07-23). *A file can lie about its own release status if you read a commented-out template as live content.*

**On method rule 1 (repo outranks web) — the null result, stated explicitly.** The repository answered every angle, so **no web source was consulted and therefore no web claim required correction.** The documentation defects in §10 are all **repo-internal**: places where DataHub's own shipped docs contradict DataHub's own shipped code at the same commit. Where corrections were needed they were to *the brief itself* — four of them, each recorded in place: patch templates live in `entity-registry/` not `metadata-io/` (`C-12`); `FeatureFlags.java` is not the gating oracle because `application.yaml` overrides it (`C-157`); `type: timeseries` is not checkable in `entity-registry.yml` (`C-73`); and `RemoveUnknownAspects` does not purge unregistered aspects (`C-145`), so the deletion-risk premise was wrong and the silent-unreadability finding (`C-146`) replaces it.

---

## §1 Requirement decomposition

| ID | Requirement (owner's words) | Modelling problem |
|---|---|---|
| `RQ-1` | "when I am considering adding a datasource I don't have all of the connections, details, data profiles etc. I need to manually gather and add as I find it" | Incremental, out-of-order, partial metadata accrual against a stable entity identity — with no job execution to hang metadata on. |
| `RQ-2` | "at first we download sample reports or database exports, usually saved as a csv sometimes a json file" | A file artifact is a first-class dataset standing in for a datasource not yet reachable. Later superseded, never deleted. Needs schema + data profile. |
| `RQ-3` | "some of the source data providers don't want us to read the production database, so we get data from a replica but it could be a different platform" | Two distinct physical datasets on two different platforms representing one logical entity. Both independently addressable. |
| `RQ-4` | "I need document the original source of the data and the data's provenance (that it's from a replica)" | A directed, traversable provenance edge replica → origin, plus an assertion of *how* (replication). Must survive graph traversal, not sit in a text field. |
| `RQ-5` | "the connection details (login/password/url) need to be stored and maintained separately from what we show the users as the data source" | Split identity from access. User-visible identifier derivable without secrets, stable across credential rotation. |
| `RQ-6` | "it also needs to be integrated with our in-house ontology tool, so I'll need to classify it by ontology" | External taxonomy term URIs at dataset *and* column granularity, round-trippable to an external system of record. |
| `RQ-7` | *(added by owner during this analysis)* "document the release branch — upgrade process for a company to do updates internally, what carries over" | Which curated metadata survives a self-hosted version upgrade, which must be rebuilt, and which can be silently lost. **The hand curation is the irreplaceable asset**: unlike harvested metadata it cannot be re-derived by re-running ingestion. |

---

## §2 Core model claims

### 2.1 The governing constraint (the analogue of "facets replace by name")

**`C-01` `VERIFIED` — UPSERT replaces the ENTIRE aspect.** The write path never merges incoming with stored. `ChangeItemImpl` builds its record solely by deserialising the MCP's own payload, and that object is set verbatim as version 0.
`metadata-io/metadata-io-api/src/main/java/com/linkedin/metadata/entity/ebean/batch/ChangeItemImpl.java` (convertToRecordTemplate ~L226–240); `metadata-io/src/main/java/com/linkedin/metadata/entity/EntityServiceImpl.java:346-349` (`latestAspect.setRecordTemplate(changeMCP.getRecordTemplate())`).
*Release: core semantics, predates all tags, no flag.*

**`C-02` `VERIFIED` — the blast radius is one aspect, not the entity.** The aspect is the atomic unit of write and an MCP carries exactly one; aspects on the same entity update independently.
`docs/modeling/metadata-model.md` ("the smallest atomic unit of write in DataHub… multiple aspects associated with the same Entity can be updated independently"); `docs/advanced/mcp-mcl.md`.

**`C-03` `VERIFIED` — the constraint IS stated normatively in prose**, unlike the prior run's model where it was buried. Two places:
`docs/what/aspect.md`: *"Metadata aspects are immutable by design… every 'update' will lead to the writing the entire large aspect back to the underlying data store."*
`docs/advanced/patch.md:11-13`: *"replacing the aspect entirely… you need to do a read-modify-write to avoid overwriting existing fields."*

**`C-04` `VERIFIED` — but the file that *should* hold the normative spec is an empty stub.** `docs/advanced/partial-update.md` is, in its entirety, `# Supporting Partial Aspect Update` + `WIP`. So is `docs/advanced/derived-aspects.md`. Five such WIP stubs exist. The semantics are stated only incidentally.

**`C-05` `VERIFIED` — there are FOUR write regimes, not one.** This is the precise resolution:

1. **UPSERT** — whole-aspect replace (`C-01`).
2. **PATCH** — escapes replace via two routes. `PatchItemImpl.applyPatch` uses the **generic** route only when `genericJsonPatch != null && (arrayPrimaryKeys non-empty || forceGenericPatch)`; otherwise the **template** route, which needs a registered template.
3. **TIMESERIES** — `UPSERT` only. `MCPItem.isValidChangeType`: *"Timeseries aspects only support UPSERT"*. No PATCH, CREATE or DELETE.
4. **GraphQL curation mutations** — a *third semantics regime* no schema documents: a **server-side read-modify-write** then a whole-aspect UPSERT. `LabelUtils` reads version 0 via `EntityUtils.getAspectFromEntity`, mutates in memory, and `MutationUtils.persistAspect` emits `ChangeType.UPSERT` with **no conditional-write header**.

`entity-registry/.../aspect/batch/MCPItem.java:62-81`; `metadata-io/metadata-io-api/.../PatchItemImpl.java:115-124, 269-304`; `datahub-graphql-core/.../resolvers/mutate/MutationUtils.java:36-91`; `.../util/LabelUtils.java:55-100`.

> **The one-line rule a team must internalise:** *DataHub never merges a write into stored state unless you explicitly ask via PATCH with a key — and the UI asks on your behalf by reading first.*

**`C-06` `INFERRED` — GraphQL curation is an unguarded lost-update race.** `MutationUtils.setProposalProperties` sets exactly entityType, aspectName, aspect, changeType and systemMetadata — `proposal.setHeaders(...)` appears nowhere in the file — so `ConditionalWriteValidator` has nothing to enforce. Two concurrent UI edits to the same aspect: the second write's base snapshot predates the first. *Reasoning: verified read-of-v0 + verified unconditional whole-aspect UPSERT + verified opt-in-only conditional writes. Not observed empirically.*

**`C-07` `VERIFIED` — prior versions survive, capped at 20.** Version 0 is the sentinel "latest"; the prior value is copied to max+1. Default retention keeps 20 versions; **version 0 is never deleted**. A byte-identical UPSERT creates no new version.
`docs/advanced/aspect-versioning.md`; `docs/advanced/db-retention.md`; `datahub-upgrade/src/main/resources/boot/retention.yaml`; `li-utils/.../Constants.java:29`.
**This is a categorical advantage over a wire format: DataHub has edit history; OpenLineage has none.**

**`C-08` `VERIFIED` — out-of-order protection is opt-in.** `lastObserved` does not gate stale writes; no code path compares incoming against stored. Protection requires HTTP headers `If-Version-Match`, `If-Modified-Since`, `If-Unmodified-Since`, `If-None-Match: *`.
`entity-registry/.../aspect/validation/ConditionalWriteValidator.java:44-67`; `CreateIfNotExistsValidator.java`.

### 2.2 Design-time registration (RQ-1)

**`C-09` `VERIFIED` — registration requires no job, run or execution.** The MCP shape is `entityType` + (`entityUrn` XOR `entityKeyAspect`) + `changeType` + `aspectName` + `GenericAspect{value,contentType}` + optional `systemMetadata`/`headers`. **Nothing in the shape references a run, task or job.**
`metadata-models/src/main/pegasus/com/linkedin/mxe/MetadataChangeProposal.pdl`.

**`C-10` `VERIFIED` — an entity "exists" iff its KEY aspect is present at version 0.** Registering a dataset with only `datasetKey` is a complete, valid registration. The Python SDK's `create()` emits the key aspect as `CREATE_ENTITY` first as a concurrency guard.
`EntityServiceImpl.exists` L3533-3556; `metadata-ingestion/src/datahub/sdk/entity_client.py:163-186`.

**`C-11` `VERIFIED` — PATCH is formally permitted on EVERY non-timeseries aspect**, but usable only via the generic envelope. `MCPItem.supportsPatch` returns unconditional `true`. **Verified byte-identical in both `v1.7.0` and `v1.6.0.1` by reading the tagged blobs** (`git cat-file -p <tag>:<path>`), not by tag containment.

**`C-12` `VERIFIED` — exactly 23 aspects have a registered patch template, hardcoded** in `SnapshotEntityRegistry.populateTemplateEngine` (comment: *"TODO: This should be more dynamic ideally, 'hardcoding' for now"*), **not** in `AspectTemplateEngine` as `docs/advanced/patch.md` claims. Registered: `ownership, datasetProperties, upstreamLineage, documentation, globalTags, editableSchemaMetadata, glossaryTerms, dataFlowInfo, dataJobInfo, dataProductProperties, chartInfo, dashboardInfo, dataJobInputOutput, structuredProperties, structuredPropertyDefinition, formInfo, versionProperties, siblings, status, domains, editableDatasetProperties, editableContainerProperties, editableMlModelGroupProperties`.
`entity-registry/src/main/java/com/linkedin/metadata/models/registry/SnapshotEntityRegistry.java:100-137`.
> **Correction to the brief:** these live in `entity-registry/`, **not** `metadata-io/…/aspect/patch/template/` — that directory does not exist.

**`C-13` `VERIFIED` — `schemaMetadata` has NO patch template.** Every write to the technical schema replaces the whole `fields[]` array. `editableSchemaMetadata` **does** have one. This asymmetry is the single most important structural fact for incremental column curation.

**`C-14` `VERIFIED` — `ConfigEntityRegistry` and `PatchEntityRegistry` each return an EMPTY `AspectTemplateEngine`.** Custom-model aspects get **zero** patch templates (both marked `TODO`).

**`C-15` `VERIFIED` — attribution-source is part of the array key** for tags, terms, owners, domains, documentation and structuredProperties, with `""` as the unattributed key. This is the model's mechanism for independent writers accruing on one entity without clobbering: each source owns its own array slot.
`metadata-ingestion/src/datahub/specific/aspect_helpers/*.py`; `common/Documentation.pdl`; `common/MetadataAttribution.pdl`.

**`C-16` `VERIFIED` — the effective array key depends on WHICH CLIENT sent the patch.** Java templates use narrower keys than the Python SDK: `globalTags.tags` is `[tag]` in Java vs `[attribution␟source, tag]` in Python; `ownership.owners` is `[owner, type]` vs `[owner, type, typeUrn, attribution␟source]`.

**`C-17` `VERIFIED` — Python patch routing is per-helper, not universal.** Entity-level helpers under `aspect_helpers/` hardcode an `arrayPrimaryKeys` constant (→ generic route). `FieldPatchHelper` and the `DatasetPatchBuilder` scalar/lineage setters pass **none** (→ template route). Consequence: `FieldPatchHelper(editable=False)` targets `schemaMetadata`, which has no template — **the SDK will build a patch the server cannot service**.

**`C-18` `VERIFIED` — even inside a PATCH, `add` at a container path is a full replace.** `set_custom_properties` silently discards every property not in the supplied dict.
`docs/advanced/patch.md`: *"Add is a bit of a misnomer… if the path already exists the value will be replaced."*

**`C-19` `VERIFIED` — `add` at a SCALAR path replaces only that field.** `/description`, `/name`, `/externalUrl`, `/upstreams/<urn>/type` are individually patchable. So PATCH *does* give field-level merge for scalars, contrary to a first-pass reading.

**`C-20` `VERIFIED` — a generic GraphQL PATCH mutation exists, ships, and is undocumented.** `patchEntity(input: PatchEntityInput!)` / `patchEntities` accept an arbitrary `aspectName`, an RFC-6902 op list (ADD/REMOVE/REPLACE/MOVE/COPY/TEST), `arrayPrimaryKeys`, `forceGenericPatch`, `systemMetadata` and `headers`. Registered unconditionally in `GmsGraphQLEngine.java:1454-1458`. Added #14823 (2025-10-29); **present in BOTH `v1.7.0` and `v1.6.0.1`** (verified by tagged blob). **`grep -rn "patchEntity" docs/` → zero hits. `grep -rn "patchEntity" datahub-web-react/src/` → zero hits.**
`datahub-graphql-core/src/main/resources/patch.graphql`.

**`C-21` `VERIFIED` — `patchEntity` carries the same routing trap.** `PatchResolverUtils` always builds a `GenericJsonPatch` envelope but defaults `arrayPrimaryKeys` to an **empty map**, so a call supplying neither it nor `forceGenericPatch:true` falls back to the template path. `validateAndTransformPatchOperations` is an explicit no-op (`return operations; // No transformation needed`).

**`C-22` `VERIFIED` — OpenAPI v3 single-aspect POST defaults to CREATE, not UPSERT.** `createIfNotExists` defaults `true` → `ChangeType.CREATE`, so a **second write to the same aspect fails** unless the caller passes `createIfNotExists=false`. The batch endpoint defaults to UPSERT.
`GenericEntitiesController.java:653-659`; `v3/EntityController.java:1066-1085`.

**`C-23` `VERIFIED` — async defaults differ by endpoint.** Single-aspect `createAspect` (L653) and `patchAspect` (L732) default `async=**false**`. Batch/multi-entity endpoints default `async=true`. *(An investigator stated this backwards; corrected by the verifier.)* The Python SDK exposes four `EmitMode` levels: `SYNC_WAIT`, `SYNC_PRIMARY`, `ASYNC`, `ASYNC_WAIT`.

**`C-24` `VERIFIED` — `ChangeType` has SEVEN values** (`UPSERT, CREATE, UPDATE, DELETE, PATCH, RESTATE, CREATE_ENTITY`); `docs/advanced/mcp-mcl.md` documents six, omitting `RESTATE`. The aspect write path accepts five.

**`C-25` `INFERRED` — `UPDATE` is accepted but its documented precondition is unenforced.** The PDL says *"NOT SUPPORTED YET / update if exists. otherwise fail"*, yet `MCPItem.CHANGE_TYPES` contains it and `ChangeItemImpl` accepts it. No validator in either validation package enforces an exists-precondition. An `UPDATE` against a non-existent aspect behaves as an UPSERT. *(The original claim cited `EntityServiceImpl.isValidAspectChangeType`; that method **does not exist** — the real method is `isCreateOrUpdate` at L3044, used only for usage telemetry. The conclusion survives; the citation was fabricated and is corrected here.)*

**`C-26` `VERIFIED` — DataHub models a first-class curation workflow.** `FormInfo` carries `FormType {COMPLETION, VERIFICATION}` plus `prompts: array[FormPrompt]`; the per-entity `forms` aspect tracks `incompleteForms`, `completedForms`, `verifications`. `forms` is registered on `dataset`. **This is the closest thing to a "draft / partially-curated entity" primitive, and OpenLineage has no equivalent.** Asymmetry: `formInfo` has a patch template; the per-entity `forms` aspect does not.

**`C-27` `VERIFIED` — `systemMetadata` carries per-write provenance**: `lastObserved, runId, lastRunId, pipelineName, registryName/Version, properties, version, schemaVersion, aspectCreated (AuditStamp), aspectModified (AuditStamp)`. `aspectCreated` is stamped only on the first version (`rowNextVersion == 1`).

**`C-28` `VERIFIED` — ingestion-run rollback is NOT a pure undo.** It deletes the run's rows and, where an older version survives, copies that version's record back into version 0 **and then deletes the surviving history row** — so the pre-run value is restored but intervening history is destroyed. When the key aspect is rolled back, the entity is soft-deleted rather than hard-deleted.
`EntityServiceImpl.java:3838-3872`; `docs/how/delete-metadata.md:284-291`.
> **Do not conflate `datahub ingest rollback` (run-scoped) with rolling back a version upgrade. They are unrelated.**

### 2.3 Identity and credential separation (RQ-3, RQ-5)

**`C-29` `VERIFIED` — a dataset URN is exactly three components**: `platform` (a `dataPlatform` URN), `name` (opaque string), `origin` (`FabricType`). Enforced by `key.size() != 3`.
`li-utils/src/main/javaPegasus/com/linkedin/common/urn/DatasetUrn.java:17-22,50-56`; `metadata/key/DatasetKey.pdl`.

**`C-30` `VERIFIED` — the URN `name` is explicitly NOT the native name.** `DatasetKey.pdl:18-19`: *"This is no longer to be used for Dataset native name. Use name, qualifiedName from DatasetProperties instead."* The URN is stable opaque identity; the human-facing name is separate mutable metadata. **This is exactly the RQ-5 split, stated in the model.**

**`C-31` `VERIFIED` — `platform_instance` is NOT a fourth URN component.** It is concatenated into `name` as `f"{platform_instance}.{table_name}"`. **Adding or changing a platform instance changes the dataset's identity.**
`metadata-ingestion/scripts/avro_codegen.py:507-518`; `docs/platform-instances.md:15`.

**`C-32` `VERIFIED` — NO shipped identifier builder in either language accepts a credential.** Java `DatasetUrn(DataPlatformUrn, String, FabricType)`; Python `make_dataset_urn(platform, name, env)`, `make_dataset_urn_with_platform_instance(...)`, `create_from_ids(platform_id, table_name, env, platform_instance)`; SDK `Dataset.__init__` identity block `(platform, name, platform_instance, env)`. The newest write surface, `patch.graphql`, likewise takes only `urn`/`entityType`. **Credential exclusion is achieved purely by API shape — there is no parameter to pass them to.**

**`C-33` `VERIFIED` — but there is NO normative prohibition.** Two grep sweeps of `docs/` for credential-and-URN co-occurrence return nothing prohibitive; the reverse sweep returns zero. The exclusion is **an accident of API shape, not a stated rule** — the same status OpenLineage had.

**`C-34` `VERIFIED` — the write path is nevertheless bounded.** `UrnValidationUtil.validateUrn` (reached from `ChangeItemImpl`/`PatchItemImpl`/`DeleteItemImpl`/`MCLItemImpl` on **every** write) unconditionally throws on leading/trailing whitespace, on `␟` anywhere, and on a **URL-encoded length above `URN_NUM_BYTES_LIMIT = 512` bytes** (`schemaField` exempt). The parenthesis/comma check is warn-only unless `STRICT_URN_VALIDATION_ENABLED=true`.
`metadata-utils/src/main/java/com/linkedin/metadata/utils/UrnValidationUtil.java:28`.
*Practical effect: a short DSN-shaped name passes; a long one does not, because `@`, `:` and `/` roughly triple under URL-encoding. This is a length ceiling, not a credential guard.*

**`C-35` `VERIFIED` — the only normative character restriction is four reserved characters**: `(`, `)`, `,`, `␟`. Characters that appear in credentials and DSNs — `@ : / = ?` — are **not** restricted.
`docs/what/urn.md:41-55`; `metadata-ingestion/src/datahub/utilities/urn_encoder.py` (`RESERVED_CHARS = {",", "(", ")", "␟"}`).

**`C-36` `VERIFIED` — DataHub has NO shared framework-level credential/DSN sanitiser in EITHER language.** Python has ad-hoc per-connector ones (`glue._sanitize_jdbc_url`, explicitly *"for safe logging"*; `git_import` delegating to GitPython's `remove_password_if_present`; `kafka_connect` rebuilding a clean URL from parts) plus a key-heuristic `redact_raw_config`. A targeted Java sweep for `removePassword|stripCredential|maskPassword|scrubUrl|sanitizeUrl|sanitizeJdbc` returns **zero hits**.
> **This inverts the prior run's finding.** In OpenLineage, Java had `JdbcUrlSanitizer` and Python did not — a port-this-code task. In DataHub **Java is the weaker side**, and there is no canonical implementation to port from either.

**`C-37` `VERIFIED` — DataHub ships a real encrypted secret store, which OpenLineage entirely lacks.** Entity `dataHubSecret` (category `internal`, key = opaque `id`), sole aspect `dataHubSecretValue`. Encryption is **AES-256-GCM** with HMAC-SHA256 key derivation, random 12-byte IV, 128-bit tag, `v2:` prefix.
`metadata-models/.../secret/DataHubSecretValue.pdl`; `metadata-operation-context/.../SecretService.java:37-52,92-127`.

**`C-38` `VERIFIED` — decryption is blocked for human callers by default as of `v1.7.0`.** `SecretService.decrypt` applies a two-check caller guard: a User-Agent check (`agentClass.isHuman()`) **and** a non-spoofable identity check (actor must be a system principal), throwing `SecurityException` in the default `ENFORCE` mode. Introduced by `08a9b1bd15` (2026-06-24, #17995).
*Javadoc: "decryption should never happen on a user-facing read path."*

**`C-39` `VERIFIED` — the GraphQL API is write-mostly for secrets by type design.** `type Secret { urn, name, description }` — **no `value` field at all**. Plaintext is reachable only via the separate `getSecretValues` query, gated on `MANAGE_SECRETS`. `ConnectionMapper` returns an empty blob for human agents *before decryption is attempted*.

**`C-40` `VERIFIED` — `dataHubConnection` is STANDALONE.** Keyed by opaque `id`; carries only `dataHubConnectionDetails` + `dataPlatformInstance`; declares **no `@Relationship`** to any entity; its GraphQL upsert input accepts no dataset URN. `DataHubJsonConnection` holds only `encryptedBlob`. **Identity and access are separated by construction in the model, not by convention.**

**`C-41` `VERIFIED` — secret NAMES are plaintext and searchable.** `DataHubSecretValue` stores `name: string` `@Searchable(TEXT_PARTIAL)` and optional plaintext `description` alongside the encrypted `value`. Only the value is encrypted.

**`C-42` `VERIFIED` — recipes reference secrets by NAME via `${VAR}` substitution**, resolved at job time by a trusted worker, with documented precedence **DataHub > File > Environment**. A recipe holds a stable non-secret handle. Gotcha documented: `${DB-PASSWORD}` parses as *"variable DB, default PASSWORD"*.
`docs/secret-resolution.md`.

**`C-43` `VERIFIED` — the user-facing hierarchy is STRUCTURED DATA, not a parsed string.** `container` is a real first-class entity; the dataset's `container` aspect is a genuine graph edge named `IsPartOf`; containers nest recursively; `BrowsePathsV2` entries each carry an optional URN.
**This is a categorical advantage over OpenLineage, where the hierarchy is a string to be parsed.**

**`C-44` `VERIFIED` — but a container's own id is an opaque md5 GUID**, computed from a sorted-JSON bag of non-secret coordinates. Credential-free and deterministic, but **not hand-derivable**. `ContainerKey.guid` → `datahub_guid(bag)`.

**`C-45` `VERIFIED` — `DataPlatformInstance` is a search facet, NOT a graph edge.** It declares two `@Searchable` and **zero `@Relationship`** annotations. Platform-instance grouping cannot be traversed; only container membership can.

**`C-46` `VERIFIED` — there is NO central per-platform dataset-naming convention table.** The only machine-readable per-platform naming metadata is a single `datasetNameDelimiter` character on each of **120** bootstrapped platforms — it says *what* separates parts, never *how many* or *which*. Per-platform URN formats exist only as connector-local prose in roughly **6 of 112** connector doc directories.
> **This is a genuine regression against OpenLineage**, which documents a naming convention for 40+ platforms.

**`C-47` `VERIFIED` — `FabricType` is required.** 17 values (several redundant aliases: `PRD`/`PROD`, `TST`/`TEST`, `SBX`/`SANDBOX`). A dataset **cannot be registered without committing to an environment up front**. Python defaults it to `"PROD"`.

### 2.4 Provenance across copies (RQ-3, RQ-4)

**`C-48` `VERIFIED` — `UpstreamLineage`/`Upstream` IS the directed, traversable edge RQ-4 asks for.** `Upstream.dataset` carries `@Relationship {name: "DownstreamOf", entityTypes: ["dataset"], isLineage: true}`. `LineageRegistry` builds lineage edges **only** from annotations where `isLineage` is true (`RelationshipAnnotation` defaults it to `false`).
**Maturity: the most mature mechanism in DataHub.** `Upstream.pdl`, `UpstreamLineage.pdl` and `DatasetLineageType.pdl` were all added **2020-05-21, commit `1283dd3ff4`** — present at every tag, no feature flag.

**`C-49` `VERIFIED` — `DatasetLineageType.COPY` exists and means exactly the right thing.** Enum: `COPY` (*"Direct copy without modification"*), `TRANSFORMED`, `VIEW`. `type` is a **required** field on `Upstream`.
> **This is DataHub's decisive win over the baseline.** OpenLineage's semantically exact mechanism (`LineageDatasetFacet`) was merged but **in no release tag**. DataHub's has been released for six years.

**`C-50` `VERIFIED` — `UpstreamLineage` supports true incremental PATCH keyed by upstream dataset URN.** `UpstreamLineageTemplate` keys `upstreams` on `dataset`; `DatasetPatchBuilder.add_upstream_lineage` emits ops at path `("upstreams", <urn>)`. One replica→origin edge can be added without clobbering others — this satisfies RQ-1's out-of-order accrual too.
Constraint: a dataset **cannot carry two edges to the same upstream with different types** — a second add replaces the first.

**`C-51` `VERIFIED` — the `Upstream.properties` map IS materialised onto the physical graph edge.** `GraphIndexUtils` reads the annotation's `properties` path; `ElasticSearchGraphService` writes it into the edge document. Values **must** be Strings — a non-String throws `UnsupportedOperationException`.

**`C-52` `VERIFIED` — CRITICAL LIMITATION: only ONE edge-property key is mapped.** `GraphRelationshipMappingsBuilder.getMappingsForEdgeProperties` maps exactly `properties.source` as a keyword, and its only defined semantic is `source == "UI"` → `isManual`. Custom keys are written but have **no declared mapping and no consumer**.

**`C-53` `VERIFIED` — THE HOW DOES NOT SURVIVE GRAPH TRAVERSAL.** Checked on all four traversal surfaces:
1. `Upstream.pdl`'s `@Relationship` projects exactly six edge attributes — `createdOn, createdActor, updatedOn, updatedActor, properties, via`. **`type` is not among them.**
2. `LineageRelationship.pdl` (returned by every lineage query) has `type: string` meaning the relationship *name* (`"DownstreamOf"`), and no properties map.
3. Rest.li `GET /relationships` returns `new EntityRelationship().setEntity(...).setType(entity.getRelationshipType())` — the relationship name only.
4. OpenAPI `GenericRelationship.NodeProperties` exposes a single `List<String> source`.
**And structurally so:** the ES edge document has no field for the lineage type at all (`ElasticSearchGraphService` writes source, destination, relationshipType, audit fields, properties) — it is not a mapping oversight that could be patched in the read path.

**`C-54` `VERIFIED` — `DatasetLineageType` is a dead end in GraphQL/UI.** In `entity.graphql` the enum is annotated `"Deprecated"` and referenced by **zero fields**. The `Dataset` type exposes only `fineGrainedLineages` from the aspect, never the `Upstream[]` array. `grep DatasetLineageType datahub-web-react/src/` returns nothing.

**`C-55` `VERIFIED` — the UI cannot assert "replication".** `LineageService` — behind the `updateLineage` mutation — **hardcodes `DatasetLineageType.TRANSFORMED`** for every manually-added upstream and stamps `properties {source: UI}`. There is no UI path to write `COPY`.
`metadata-service/services/.../LineageService.java:167-177`.

**`C-56` `VERIFIED` — but `patchEntity` CAN set it.** Because `patchEntity` accepts an arbitrary aspect and an RFC-6902 op list, a client can set `/upstreams/<urn>/type` to `"COPY"` over GraphQL. *(This corrects two investigators who claimed GraphQL cannot express the lineage type. The narrow `lineage.graphql` mutation cannot; the generic one can.)*

**`C-57` `VERIFIED` — `Siblings` is the aliasing mechanism and it is UNDIRECTED and UNTYPED.** `siblings: array[Urn]` + `primary: boolean`. No direction field, no type field, no audit stamp, no properties bag. `SiblingOf` does **not** set `isLineage`, so siblings are invisible to lineage traversal.

**`C-58` `VERIFIED` — DECISIVE ANTI-PATTERN PROOF.** When two datasets are siblings, `SiblingGraphService.filterLineageResultFromSiblings` **actively deletes any lineage relationship between them** from the merged response. Step 1's comment is verbatim *"remove the source entities siblings from this entity's downstreams"*; the partition predicate keeps a relationship only if its target is **not** in the sibling group.
**An Oracle←Snowflake `COPY` edge, if the two are also siblings, VANISHES from the default lineage read path. Siblings destroys the very fact RQ-4 exists to record.**
`metadata-io/src/main/java/com/linkedin/metadata/graph/SiblingGraphService.java:161-186`.

**`C-59` `VERIFIED` — sibling merging is the DEFAULT, not an opt-in.** `entity.graphql`: *"Optional flag to not merge siblings in the response. They are merged by default."* The resolver defaults `separateSiblings` to `false`; `FeatureFlags.showSeparateSiblings` is `false`; env `SHOW_SEPARATE_SIBLINGS` defaults false.

**`C-60` `VERIFIED` — the frontend collapses siblings into ONE node.** `combineEntityWithSiblings` deep-merges the sibling's entire payload (tags, terms, owners, schema fields, customProperties, structuredProperties, health, forms, subtypes, stats) and then **overwrites the urn**: `combinedBaseEntity.urn = entity.urn`. In search, `createSiblingEntityCombiner` suppresses already-visited sibling URNs so only one card renders. **This is aliasing, not relating.**
`datahub-web-react/src/app/entity/shared/siblingUtils.ts:398-425, 458-479`.

**`C-61` `VERIFIED` — the anti-pattern is narrower than it first appears.** `SiblingGraphService` is referenced **only** by the GraphQL `EntityLineageResultResolver`. Multi-hop impact analysis (`searchAcrossLineage`/`LineageSearchService`) and the raw graph/relationships APIs contain **no** server-side sibling logic. The COPY edge survives those reads; merging there is frontend-only. Sibling-merged lineage is also **hard-capped at one hop** — `getLineage` throws `UnsupportedOperationException` for `maxHops > 1` unless `separateSiblings` is true.

**`C-62` `VERIFIED` — the prose distinguishes the two, in a connector doc.** `metadata-ingestion/docs/sources/trino/trino_pre.md:5-8`: *"Extracts metadata and **two distinct relations**… **Siblings**: Same logical dataset in two platforms… **UpstreamLineage**: This Trino dataset reads from the connector dataset."* `docs/cli-commands/dataset.md:128` defines siblings as *"semantically equivalent datasets"*. **Nowhere in the repo is Siblings described as a derivation or provenance edge.**

**`C-63` `VERIFIED` — CANONICAL PRECEDENT: DataHub's own Snowflake Shares connector emits BOTH.** The inbound (replica) side gets `Siblings(primary=False, siblings=[origin])` **and** `UpstreamLineage(upstreams=[Upstream(dataset=origin, type=COPY)])`; the outbound side gets `Siblings(primary=True)`. Trino and Unity Catalog — both *cross-platform*, structurally closer to RQ-3 — emit the same pair but with `type=VIEW`.
`metadata-ingestion/src/datahub/ingestion/source/snowflake/snowflake_shares.py:127-156`.

**`C-64` `VERIFIED` — `SiblingAssociationHook` will NOT fire for an Oracle→Snowflake pair.** Both creation paths gate on `DBT_PLATFORM_NAME = "dbt"`. Further, **a hand-written Siblings aspect is protected**: both paths abort as soon as either side already has a siblings aspect. The method name `setSiblingsAndSoftDeleteSibling` is **vestigial — no soft delete occurs at HEAD** (grep for `soft|Status|removed` across the 547-line file returns nothing).

**`C-65` `VERIFIED` — NAMING TRAP DEFUSED: `common/Origin.pdl` has nothing to do with data provenance.** It records whether an *entity* was created `NATIVE` to DataHub or came from an `EXTERNAL` identity provider. It has **no URN field and no `@Relationship`** — it cannot point at another dataset at all.

**`C-66` `VERIFIED` — `logicalParent` cannot point one physical dataset at another.** `LogicalParentPlatformValidator` (registered at `SpringStandardPluginConfiguration.java:668`, enforced across GraphQL/OpenAPI/RestLI/SDK) rejects any parent that is not a dataset on a platform with `DataPlatformInfo.logical == true`. Javadoc: *"physical assets link to logical models, never to other physical assets."* Using it for RQ-3 requires authoring a **third, synthetic logical dataset**.

**`C-67` `VERIFIED` — no way to relate two platform instances.** `DataPlatformInstanceProperties` declares only `name` and `description` plus mixins, with **zero `@Relationship`** fields. `dataHubConnection` has no dataset link. **Verified absence.**

**`C-68` `VERIFIED` — a DataJob node is the one mechanism that makes HOW graph-visible.** `DataJobInputOutput.inputDatasetEdges` (`Consumes`) and `outputDatasetEdges` (`Produces`, `isUpstream: false`) are **both `isLineage: true`**, both `common/Edge` records with created/lastModified/properties. The DataJob can carry the replication statement via `dataTransformLogic` (registered on `dataJob`, not `dataset`).

**`C-69` `VERIFIED` — column-level provenance rides in the same aspect.** `FineGrainedLineage` carries upstream/downstream `schemaField` URN arrays, `transformOperation: optional string`, `confidenceScore: float = 1.0`. **Caveat:** the `@Relationship` on `fineGrainedLineages` **omits `isLineage: true`**, unlike the one on `Upstream.dataset` — so that edge does not register as lineage. UI column-level lineage is additionally gated by `schemaFieldCLLEnabled = false`.

**`C-70` `VERIFIED` — `Upstream.matchType` is a permanently-stale point-in-time verdict.** `LineageMatchType {EXACT, NORMALIZED, UNRESOLVED}`. Its own doc: *"a reference recorded as UNRESOLVED (its target did not exist yet) keeps that value even after the target is later ingested… the verdict only refreshes when the referencing source is re-ingested."* **Directly relevant to RQ-1's out-of-order accrual.**

### 2.5 File datasets and profiles (RQ-2)

**`C-71` `VERIFIED` — DECISIVE: a data profile attaches by a PLAIN MCP with NO execution context.** `profiler.py:462-465` yields `MetadataChangeProposalWrapper(entityUrn=dataset_urn, aspect=profile)`. Grepping the whole file for `DataProcessInstance|run_event|assertion` returns **no matches**. There is no run to hang the profile on and none is required.
> **This is where DataHub beats the baseline outright.** OpenLineage scored `NATIVE-CONSTRAINED` on RQ-2 precisely because data-profile facets are *input* facets reachable only from a `RunEvent`.

**`C-72` `VERIFIED` — a profile alone materialises the dataset.** Ingesting a timeseries aspect for a non-existent URN auto-creates the entity's key aspect (`EntityServiceImpl:2303-2320`).

**`C-73` `VERIFIED` — `datasetProfile` is declared timeseries in the PDL, NOT in `entity-registry.yml`.** The registry lists aspect **names only** and carries no `type:` key.
> **Correction to the brief:** the prescribed "check `entity-registry.yml` for `type: timeseries`" is not performable. Aspect type lives in each PDL's `@Aspect` annotation, surfaced at runtime as `AspectSpec.isTimeseries()`.

**`C-74` `VERIFIED` — `DatasetProfile` has four fields, ALL optional**: `rowCount`, `columnCount`, `fieldProfiles`, `sizeInBytes`, plus four inherited from `TimeseriesAspectBase`. It has **no** field for a run, job, assertion or source.

**`C-75` `VERIFIED` — `DatasetFieldProfile` requires exactly ONE field, `fieldPath`.** Optional: `uniqueCount`, `uniqueProportion`, `nullCount`, `nullProportion`, `min`, `max`, `mean`, `median`, `stdev` (**all typed `string`, not numeric**), `quantiles`, `distinctValueFrequencies`, `histogram`, `sampleValues`.
> The brief's enumeration omitted `histogram`.

**`C-76` `VERIFIED` — append semantics are an idempotent hash-keyed ES upsert.** The doc id is a hash of `timestampMillis + eventGranularity + urn + [collectionId] + messageId + partitionSpec`. Same key overwrites; a new `timestampMillis` creates a coexisting point. `messageId` is the caller-supplied idempotency key.

**`C-77` `VERIFIED` — no automatic retention on timeseries aspects.** The default `maxVersions: 20` policy is enforced by `EbeanRetentionService` against `EbeanAspectV2` rows with `version != 0`. Timeseries aspects have no rows in that table, and the timeseries index builder contains no ILM policy. **Profiles do not age out.**

**`C-78` `VERIFIED` — `PartitionSpec.type` is `@deprecated` with the doc `Unused!`.** Only `partition: string` (required) and `timePartition` are live. Setting **both** `partition` and `timePartition`, or **neither**, throws `IllegalArgumentException` at index time — a hard ingestion failure.

**`C-79` `VERIFIED` — profile history is readable over GraphQL**, not only Rest.li: `Dataset.datasetProfiles(startTimeMillis, endTimeMillis, filter, limit): [DatasetProfile!]`. OpenAPI v3 GET returns only the **latest** point.
**Gap:** the GraphQL `DatasetFieldProfile` type has **no `histogram` field** though the PDL does — histograms are write-only through the UI path.

**`C-80` `VERIFIED` — FILE FORMAT IS NOT MODELLED.** `DatasetProperties` has no `format` field. `DatasetSubTypes` contains no CSV/JSON/Parquet/File member. The s3 source records format only implicitly, via the customProperty `schema_inferred_from` (the full path, carrying the extension), and emits **no `subTypes` aspect at all** for the dataset.
> **This is a regression against OpenLineage**, which has `datasetType: FILE` and `storage.fileFormat: csv` as first-class.

**`C-81` `VERIFIED` — `SchemaMetadata` requires SIX values, not three.** Required: `hash: string`, `platformSchema` (union), `fields: array[SchemaField]`, plus three inherited from `SchemaMetadataKey`: `schemaName` (`@validate.strlen` min 1 max 500), `platform` (`DataPlatformUrn`), `version` (long). The `platformSchema` union has **no CSV member and no plain-JSON member** — for a downloaded CSV the only correct choices are `OtherSchema(rawSchema: string)` or `Schemaless`.

**`C-82` `VERIFIED` — hand-authoring a schema is a solved, shipped path in both clients.** CLI: `SchemaMetadataClass(schemaName=<name|id|urn>, platform=<platform urn>, version=0, hash="", fields=[...], platformSchema=OtherSchemaClass(rawSchema=yaml.dump(...)))`. SDK: same with `schemaName=""`, `platformSchema=SchemalessClass()`.

**`C-83` `VERIFIED` — HAZARD in the CLI YAML path.** `datahub dataset upsert` emits `schemaMetadata` only when **every** field carries type information and **refuses the mixed case outright** (`ValueError: "Either all fields must have type information or none of them should"`). If none do, it silently emits no `schemaMetadata` at all. **For incremental column accrual, a partially-typed YAML is a hard failure and an untyped one is a silent no-op.**

**`C-84` `VERIFIED` — a YAML with no `properties:` key SILENTLY WIPES all existing customProperties.** `generate_mcp` calls `set_custom_properties(self.properties or {})` unconditionally, and that helper *"replaces all existing custom properties"*.

**`C-85` `VERIFIED` — the high-level Python SDK has ZERO profile support.** A recursive case-insensitive grep for `profile` across all 28 `datahub/sdk/` modules returns **zero matches**. `entities.get()` can never round-trip a profile: `get_entity_semityped` is documented *"Get (all) non-timeseries aspects"*, and `get_aspect` raises `TypeError` for timeseries. The Java v2 client likewise has zero occurrences.

**`C-86` `VERIFIED` — data-lake profiling is OPT-IN and OFF by default** (`DataLakeProfilerConfig.enabled = False`). A CSV ingested with stock config gets **no** `datasetProfile`.

**`C-87` `VERIFIED` — for supersession, `deprecation.replacement` is the right field but is NOT a graph edge.** `Deprecation` requires `deprecated: boolean`, `note: string`, `actor: Urn`; optional `decommissionTime`, `replacement: Urn`. **`replacement` carries no `@Relationship`** — it is a stored pointer resolved at read time, not traversable. Traversable succession must still come from `upstreamLineage`.

**`C-88` `VERIFIED` — `status.lifecycleStage` is released but dark.** Added `82aa9b6b0a` (2026-04-15, #17041), an ancestor of **both** `v1.6.0` and `v1.7.0`, **not** feature-flag gated. `Status.removed` is *not* auto-synced from it. **But** `bootstrap_mcps/lifecycle-stages.yaml` ships as literally `[]` with every example stage commented out — so no stage exists to point at without authoring a custom `lifecycleStageType`.

**`C-89` `VERIFIED` — `versionSet`/`versionProperties` is shipped but GATED OFF.** `entityVersioning` defaults `false` in both `application.yaml:1682` and `FeatureFlags.java:51`. When off, the versioning APIs, validators and side effects are not registered.

### 2.6 Classification and extensibility (RQ-6)

**`C-90` `VERIFIED` — `GlossaryTermInfo` already models an external system of record.** `termSource` is a **REQUIRED bare `string`** (`@Searchable KEYWORD`); `sourceRef` optional, indexed; `sourceUrl` optional, **not** indexed. `rawSchema` is deprecated.
**The doc-comment's "default INTERNAL" is NOT a schema default** — the PDL has no initializer; it is supplied client-side by `business_glossary.py` `DefaultConfig.source_type` and hardcoded in `CreateGlossaryTermResolver.java:117`.

**`C-91` `VERIFIED` — the term URN id is an opaque free-form string.** `GlossaryTermKey` has a single `name: string` documented *"serves as a unique id"*. `INFERRED`: a raw IRI can legally be the id, since only `( ) , ␟ %` are reserved — none appear in a typical http(s) IRI. **No shipped code does this.**

**`C-92` `VERIFIED` — an RDF/OWL/SKOS importer exists and IS in released tags.** Maps `skos:Concept` → GlossaryTerm, builds node hierarchy from IRI path segments, writes the source IRI into **both** `sourceRef` and `sourceUrl` with `termSource='EXTERNAL'`. Added `5bf50757b6` (2026-03-13, #15741); present in **`v1.7.0` and `v1.6.0.1`**.
**Maturity caveat: `@support_status(SupportStatus.ALPHA)`, its own docs say "MVP Scope".** Do not rely on it as a production ontology bridge without validation.

**`C-93` `VERIFIED` — the RDF importer does NOT round-trip the IRI into the URN.** The id is a lossy dot-join of host + path segments (`http://example.com/path/to/term` → `urn:li:glossaryTerm:example.com.path.to.term`), with a GUID fallback for non-ASCII or reserved characters. Identity is *derived from*, not equal to, the IRI.

**`C-94` `VERIFIED` — round-trip OUT is NOT implemented in this repository.** No glossary export command, sink or serializer; `entrypoints.py` registers no `glossary` command group. The docs point only at an **external** GitHub Action (`acryldata/business-glossary-sync-action`). **Verified absence.**
**Partial mitigation:** generic aspect round-trip does exist — `datahub get --urn -a <aspect>`, `datahub properties get/list --to-file` (YAML), and `datahub dataset get --to-file` / `dataset sync --from-datahub`, the last of which reads back per-column tags, terms and structured properties.

**`C-95` `VERIFIED` — THE CURATION-PROTECTION ARCHITECTURE.** `EditableSchemaMetadata`'s own doc comment is load-bearing: *"This separates changes made from ingestion pipelines and edits in the UI **to avoid accidental overwrites of user-provided data by ingestion pipelines**."* `schemaMetadata.fields[]` is ingestion-owned; `editableSchemaMetadata.editableSchemaFieldInfo[]` is the curation surface. Added `039fe597f7` (2021-03-18).
The dataset-level analogue is `EditableDatasetProperties` (`description`, `name`, indexed as `editedDescription`/`editedName`).
> **OpenLineage has no equivalent of this split. It is DataHub's single strongest asset for RQ-1.**

**`C-96` `VERIFIED` — the UI unions three per-column term/tag sources and de-duplicates by URN**, labelled `direct` (schemaField entity), `editable` (editableSchemaMetadata) and `uneditable` (schemaMetadata + inherited businessAttribute). This is the concrete merge site that makes the protection real rather than aspirational.

**`C-97` `VERIFIED` — all 12 UI column write sites target `EDITABLE_SCHEMA_METADATA_ASPECT_NAME`; plain `SCHEMA_METADATA` never appears.**

**`C-98` `VERIFIED` — `EditableSchemaMetadata` is PATCHable with a TWO-LEVEL merge**: the array merged by `fieldPath`, and within each field the nested `globalTags`/`glossaryTerms` merged by URN via the same templates used at entity level. **This is the precise mechanism that makes incremental, out-of-order, partial column classification safe.**

**`C-99` `VERIFIED` — `schemaField` is a first-class entity carrying SIXTEEN aspects**: `schemafieldInfo, structuredProperties, forms, businessAttributes, status, schemaFieldAliases, documentation, testResults, incidentsSummary, deprecation, subTypes, logicalParent, globalTags, glossaryTerms, semanticFieldAnnotation, aiContext`. URN form `urn:li:schemaField:(<parent_dataset_urn>,<encoded_field_path>)`.

**`C-100` `VERIFIED` — but schemaField entities are NOT auto-created at default configuration.** `MCP_SIDE_EFFECTS_SCHEMA_FIELD_ENABLED` defaults **false**. This **contradicts `metadata-models/docs/entities/schemaField.md`**, which says they are *"automatically created by DataHub when datasets with schemas are ingested"*.

**`C-101` `VERIFIED` — `StructuredPropertyDefinition` requires exactly three fields**: `qualifiedName`, `valueType` (`@UrnValidation` against `dataType`), `entityTypes` (`@UrnValidation` against `entityType`). Optional/defaulted: `displayName`, `typeQualifier`, `allowedValues`, `allowedPlatforms`, `description`, `searchConfiguration`, `version`, `created`, `lastModified`, `cardinality` (default `SINGLE`), `immutable` (default `false`).
Added `943bb57cbc` (2024-01-22); docs state *"introduced in version 0.13.1"*. **GA, unflagged.** `entityTypes` **includes `schemaField`** — added in the original commit.

**`C-102` `VERIFIED` — `entityTypes` is NOT enforced at assignment write time.** Exhaustive grep of the 801-line `StructuredPropertiesValidator` finds no check of the target entity's type against `definition.getEntityTypes()`; every `getEntityTypes()` call site is a mappings builder. **Consequence differs by version:** at HEAD (unreleased #19490) `structuredProperties` is mapped `dynamic: false`, so an undeclared type is stored in `_source` but not searchable; in **every released version** such values were dynamic-mapped by Elasticsearch, indexed with a guessed type — the bug #19490 exists to fix.

**`C-103` `VERIFIED` — `allowedPlatforms` IS enforced at write time**, in sharp contrast to `entityTypes`. A platform-restricted property rejects assignment to an entity on another platform **and to any entity with no data platform at all**. Directly relevant when the same property must apply to an Oracle original and a Snowflake replica.

**`C-104` `VERIFIED` — structured properties render richly at BOTH entity and column granularity**, with display settings that are themselves a modelled aspect: `StructuredPropertySettings {isHidden, showInSearchFilters, showInAssetSummary, hideInAssetSummaryWhenEmpty, showAsAssetBadge, showInColumnsTable}`.
> **This is the decisive contrast with the baseline.** OpenLineage's custom facets are *stored but invisible to consumer UIs*. DataHub's sanctioned extension mechanism renders, is searchable, and is filterable.

**`C-105` `VERIFIED` — structured property definitions round-trip through the CLI.** `datahub properties get/list --to-file` exports YAML; `datahub properties upsert -f <file>` reads it back. **This is the round-trip the glossary lacks.**

**`C-106` `VERIFIED` — THE REPLACE-VS-MERGE TRAP.** By default a connector writing structured properties **replaces the entire aspect on every run**. `StructuredPropertyWriteMode` docstring: *"`UPSERT` replaces the whole `structuredProperties` aspect each run (recipe is source of truth). `PATCH` adds each property individually so user/UI/other-pipeline edits survive."* Merge is **opt-in per source**. Released (#17450).

**`C-107` `VERIFIED` — the custom-model extension rules are NORMATIVE but few, and none mandates a name prefix.** Binding constraints: (1) the registry file **MUST** be named `entity-registry.yaml`/`.yml`; (2) the PDL filename **MUST** match the class name and the package path **MUST** match the directory path; (3) registry `id` unique across registries; (4) aspect names unique among all aspects DataHub knows; (5) key-aspect fields **MUST** be String or Enum; (6) all aspects **MUST** be Records.
> **This differs materially from the baseline**, where extensions *must* carry a project prefix and an immutable versioned `_schemaURL`. DataHub has **no prefix requirement and no schema-URL requirement**.

**`C-108` `VERIFIED` — custom-registry versioning is a build-time gradle property**, defaulting to `0.0.0-dev`, not a required YAML field. Bumping it is required to deploy side-by-side; DataHub refuses to load a model backwards-incompatible with the previous version of the same registry.

**`C-109` `VERIFIED` — custom aspects DO render without frontend code**, via `@Aspect autoRender/renderSpec` → GraphQL `RawAspect` → a generic `DynamicTab` (tabular / properties / raw JSON). Only `entityV2`'s `EntityProfile` wires this.

**`C-110` `VERIFIED` — a custom model registry can ship executable server-side logic**, not just schema: the shipped example registers `aspectPayloadValidators` (including a Spring-scanned one), `mutationHooks`, `mclSideEffects` and `mcpSideEffects`. **This is the supported way to enforce custom rules uniformly across GraphQL/OpenAPI/Rest.li without forking DataHub** — and it is the mechanism by which a "no credentials in URNs" rule could be enforced (see `D-07`).

**`C-111` `PENDING` — whether a custom plugin aspect's `@Searchable` annotations reach Elasticsearch mappings is not established.** `INFERRED` partial: `PatchEntityRegistry` builds specs through the same `EntitySpecBuilder` as core models, which constructs a `SearchableAnnotation` handler — so searchability is *likely*, but the spec was not traced into an actual ES mapping.

**`C-112` `VERIFIED` — `BusinessAttribute` is BETA, OSS-only, DISABLED BY DEFAULT, UI-creatable only** (no ingestion path), and limited to one attribute per schema field. `businessAttributeEntityEnabled` defaults `false`.

**`C-113` `VERIFIED` — term and tag associations carry per-assignment provenance**: `actor: optional Urn`, `context: optional string`, and `attribution: optional MetadataAttribution` (time/actor/source/sourceDetail), individually indexed.
**But `C-114` narrows it.**

**`C-114` `VERIFIED` — exactly ONE GraphQL mutation writes `attribution`**: `DescriptionUtils.updateDocumentDescription`. Manual UI term-adds, tag-adds and owner-adds leave attribution **empty**. For manual curation, per-edit provenance rests on **aspect versions + systemMetadata + Timeline**, not attribution.

**`C-115` `VERIFIED` — glossary terms carry their own `structuredProperties` aspect**, so an ontology's extra attributes (IRI, ontology version, mapping confidence, system-of-record id) can be attached to the *term* as typed searchable values rather than crammed into `sourceRef`/`customProperties`.

**`C-116` `VERIFIED` — term-to-term semantics are limited to four typed relationships** (`IsA`, `HasA`, `HasValue`, `IsRelatedTo`). The RDF importer promotes only `skos:broader`/`narrower` into a real graph edge. `skos:exactMatch`, `closeMatch`, `owl:sameAs`, `equivalentClass`, `rdfs:subClassOf` are **preserved as comma-joined IRI strings in `customProperties`** — searchable text, **not** traversable edges.

**`C-117` `VERIFIED` — `customProperties` is entity-level ONLY and never reaches columns.** It is `map[string,string]`, `@Searchable TEXT queryByDefault`, PATCHable by key, rendered in the same Properties tab. Untyped, no validation, no allowed values, no cardinality.

**`C-118` `VERIFIED` — a GA no-code bulk path exists**: `csv-enricher` maps resource + subresource to glossary terms/tags/description/domain/structured properties, defaults to additive semantics, and writes column-level classification into `editableSchemaMetadata`. **It cannot set structured properties at column granularity** (*"owners and structured properties can only be applied at the resource level"*). Its "PATCH" is a **client-side read-modify-write against the live graph** and it hard-requires a graph connection.

**`C-119` `VERIFIED` — a first-class column-granular structured-property write path DOES exist.** The `datahub dataset` YAML models per-column `structured_properties`, `globalTags` and `glossaryTerms`; `dataset sync --to-datahub` emits `StructuredPropertiesClass` against each `schemaField` URN, and `--from-datahub` reads them back.

### 2.7 Consumers and backends

**`C-120` `VERIFIED` — STRUCTURAL ADVANTAGE.** This angle is answerable *only because DataHub ships its own consumer in-repo*: 36 `.graphql` schema files, Java resolvers, and the React app all at HEAD. **For OpenLineage this angle returned nothing** — its consumers are third-party and unverifiable.

**`C-121` `VERIFIED` — DataHub ships four read/query APIs plus a CLI**: GraphQL (36 schema files, incl. `searchAcrossEntities`/`searchAcrossLineage`), OpenAPI v3, OpenAPI v2/v1, Rest.li. **OpenLineage has no query API at all.**

**`C-122` `VERIFIED` — an audit trail of manual edits exists on two solid mechanisms**: numbered historical aspect versions retrievable via `GET /aspects/{urn}?aspect=X&version=N` and the OpenAPI v3 `If-Version-Match` header; and `systemMetadata` (`runId`, `pipelineName`, `aspectCreated`, `aspectModified`). *(A third — `MetadataAttribution` — is narrowed by `C-114`.)*

**`C-123` `VERIFIED` — the deletion primitive is implemented end to end.** Soft delete (`Status.removed`), hard delete, and run-scoped rollback, exposed through `datahub delete` and `datahub ingest rollback`, documented in `docs/how/delete-metadata.md`. `datahub delete --aspect <name> --start-time/--end-time` also deletes **timeseries points by timestamp range**, and can operate by filter across a whole platform — far easier to fire accidentally than the Rest.li `truncateTimeseriesAspect` action (whose `dryRun` defaults **true**).

**`C-124` `VERIFIED` — CAVEAT: sibling write-retargeting.** `BatchAddTermsResolver` and `UpdateDescriptionResolver` catch a write failure and silently retry against a **sibling** URN. **Narrow**: it fires only for (a) a single-resource `batchAddTerms` whose `subResource` (column path) is non-null, and (b) dataset schema-**field** description updates. So the hazard is *"column-level term/description edits may silently land on the sibling"*, not writes generally.

### 2.8 Release, upgrade and durability (RQ-7)

**`C-125` `VERIFIED` — release branches are cut from master AT the `vX.Y.0rc1` commit.** `merge-base(master, origin/releases/v1.5.0) = 5a49d42b65 == v1.5.0rc1`; `v1.6.0 → b814d2b109 == v1.6.0rc1`; `v1.7.0 → b88750fe58 == v1.7.0rc1`.

**`C-126` `VERIFIED` — branches diverge hard and never merge back.** `releases/v1.5.0`: 69 ahead / **1863 behind**. `v1.6.0`: 87 / 1223. `v1.7.0`: 7 / 345. **A self-hoster on a release line runs code thousands of commits behind master and receives only what is explicitly backported.**

**`C-127` `VERIFIED` — backports are per-line, separately-numbered PRs, not one fix propagated.** The same CVE carries three PR numbers and three SHAs: wire-runtime CVE-2026-45799 → master `b6ba908404` (#19094), `v1.6.0` `3aed8430a6` (#19100), `v1.7.0` `48fd8191d5` (#19317).

**`C-128` `VERIFIED` — backporting is only half mechanical.** On `releases/v1.6.0`, 51 of 87 branch-only commits carry a cherry-pick marker; `git cherry` finds only **31 of 85 patch-identical**. The branch carries hand-written adaptation commits (`fix(backport): adapt high-conflict cherry-picks to v1.6 APIs`) and **a revert of a fix that could not be adapted**.

**`C-129` `VERIFIED` — the GA tag does NOT reliably re-tag the final rc.** True for `v1.7.0 == v1.7.0rc2 == 7f81ccbfe2`, `v1.3.0`, `v1.5.0.5`, `v1.5.0.7`. **False** for `v1.5.0`, `v1.6.0` and `v1.6.0.1`, where GA sits 1, 1 and 3 commits **beyond** the final rc — **code ships in GA that no release candidate covered**. `v1.6.0.1`'s three post-rc commits are all CVE bumps. An rc line can also be abandoned: `v1.4.1rc1` exists with no `v1.4.1`.

**`C-130` `VERIFIED` — patch-branch naming is inconsistent, not cleanly drifted.** `v1.1.0.1` → `releases/v1.1.0`; `v1.2.0.1`/`v1.3.0.1`/`v1.4.0.2`/`v1.4.0.3` → `hotfixes/vX.Y.0`; `v1.5.0.x`/`v1.6.0.1` → `releases/vX.Y.0`. A script mapping version → branch must handle **at least three prefixes** plus a legacy dash-separated form (`origin/hotfix/0-14-0-1`) and **cannot infer the prefix from the version number**.

**`C-131` `INFERRED` — the de-facto support window is TWO concurrent minor lines**, the previous going quiet at or just before the next GA (`releases/v1.5.0` tip 2026-05-19 vs `v1.6.0` GA 2026-05-21). **No prose support-window, EOL or backport policy exists anywhere in `docs/` or `.github/` — verified absence.** A codified floor exists elsewhere: `quickstart_versioning.py:24` sets `MINIMUM_SUPPORTED_VERSION = "v1.1.0"`, enforced nightly — but that governs the `datahub docker quickstart` installer only, and its matrix is stale (no v1.6 or v1.7).

**`C-132` `VERIFIED` — Docker images publish ONLY on a GitHub Release event or a master push.** Pushes to `releases/**` build and test but **never publish** (`ENABLE_PUBLISH` gate). The only floating published tag is `:quickstart`, which tracks master HEAD. Registry: `acryldata`.

**`C-133` `INFERRED` — fixes merged onto a release branch are not available as an image until the next tag+Release.** Currently 5 commits sit on `releases/v1.7.0` past `v1.7.0` and 2 on `releases/v1.6.0` past `v1.6.0.1` — **including security backports** (`fix(sec): backport security fixes and CVE dependency bumps`).

**`C-134` `VERIFIED` — server and CLI are NOT versioned in lockstep.** The server carries a hand-maintained default-CLI pin in `gradle/versioning/cliVersion.gradle` (`cliVersion = "1.7.0"`); the CLI's `_version.py` is a dev placeholder (`1!0.0.0.dev0`) stamped by a `release.sh` **no workflow invokes**; `docs/cli.md` states CLI releases come from a **different repository**.

**`C-135` `VERIFIED` — the only compatibility guarantee is a non-blocking stderr warning** comparing major/minor/micro. **The 4th version component is ignored entirely** and every failure path is swallowed.
`INFERRED` consequence: the ledger's own recommendation for `v1.6.0.1` (CLI `1.7.0.3`) would make the CLI print `❗Client-Server Incompatible❗` and advise a downgrade, because the check compares the minor component.

**`C-136` `VERIFIED` — minor upgrades cannot be skipped.** `v1.7.0`: *"You **must upgrade to v1.6.0 before upgrading to v1.7.0** — do not skip 1.6.0. Deploy v1.6.0 with Helm chart **1.0.3**, let system-update complete, then upgrade to v1.7.0 with Helm chart **1.1.0**."* Pinned Helm versions are **non-monotonic** (`v1.6.0.1` requires 1.1.2 while the newer `v1.7.0` requires 1.1.0).

**`C-137` `VERIFIED` — a patch release is NOT a safe drop-in.** `v1.6.0.1` is described as *"security patches, bug fixes… No new features; no schema or model changes"* — yet its Breaking Changes section **adds new privilege requirements to logical-parent links, data-product membership writes and structured-property creation**: exactly the write paths a curation team's automation uses.

**`C-138` `VERIFIED` — the ledger does not cover every released tag.** Five v1.x patch releases have **no section at all** (`1.1.0.1, 1.2.0.1, 1.3.0.1, 1.4.0.2, 1.4.0.3`); documenting patch releases began at `v1.5.0`. **There is no CHANGELOG file anywhere in the repo.**

**`C-139` `VERIFIED` — DECISIVE (a): `metadata_aspect_v2` is authoritative for versioned aspects and structurally upgrade-proof.** The aspect body is an opaque `@Lob` blob under PK `(urn, aspect, version)`, so adding, changing or removing aspect *types* across versions requires **no DDL and no data migration**. Every hand-curated asset except profiles is a versioned aspect: `schemaMetadata`, `editableSchemaMetadata`, `glossaryTerms`, `globalTags` (dataset **and** schemaField level), `domains`, `ownership`, `institutionalMemory`, `editableDatasetProperties`, `structuredProperties`, `upstreamLineage`, `siblings`.
**This is the structural reason hand curation survives upgrades.**

**`C-140` `VERIFIED` — DECISIVE (b): FOR TIMESERIES ASPECTS, ELASTICSEARCH IS AUTHORITATIVE AND THERE IS NO REBUILD PATH.** `ingestTimeseriesProposal`'s own javadoc: *"Timeseries is pass through to MCL, no MCP."* Only a **synthesised key aspect** reaches SQL; the payload is `produceMCLAsync` only. **A `mysqldump` of `metadata_aspect_v2` contains ZERO `datasetProfile` bodies.** Eleven aspects are affected: `datasetProfile, datasetUsageStatistics, assertionRunEvent, operation, chartUsageStatistics, dashboardUsageStatistics, documentUsageStatistics, queryUsageStatistics, dataProcessInstanceRunEvent, datahubIngestionCheckpoint, datahubIngestionRunSummary`. **Neither `RestoreIndices` nor `LoadIndices` can regenerate them** — both read SQL filtered to `VERSION_COLUMN = ASPECT_LATEST_VERSION`.
The only backup is an Elasticsearch snapshot. The only other copy is the Kafka topic `MetadataChangeLog_Timeseries_v1`, retained **90 days**.

**`C-141` `VERIFIED` — the two halves must be stated together.** Timeseries data **does** survive an ordinary version upgrade and an ES mapping reindex, because `TimeseriesAspectService` is in the `ElasticSearchIndexed` set that `BuildIndices` reindexes. **Correct operational statement: hand-curated profiles are carried through upgrades and reindexes, but are unrecoverable from the relational side — they die with the ES cluster, not with the upgrade.**

**`C-142` `VERIFIED` — DECISIVE (c): the graph/lineage index is FULLY rebuildable from SQL.** The one thing that could have broken this was tested: `grep -c '@Relationship'` over **all eleven** timeseries PDLs returns **0 in every file**. Every edge in `graph_service_v1` derives from a versioned aspect. **The provenance edges between the production database and its replica are fully reconstructible.**

**`C-143` `VERIFIED` — DECISIVE (d): the search index is rebuildable, but the rebuild is ASYNCHRONOUS through Kafka and a green step is not proof.** `SendMAEStep` → `restoreIndices` emits `ChangeType.RESTATE` MCLs via `alwaysProduceMCLAsync` then `producer.flush()`. **The step reports SUCCEEDED once messages are PRODUCED, not once documents are indexed.** A company that scales consumers down for the upgrade window and then runs `RestoreIndices` can see a successful job and an empty index. `LoadIndices` is the synchronous alternative, writing indices directly via `UpdateIndicesService`.

**`C-144` `VERIFIED` — `RestoreIndices` does NOT clear by default.** All three clear steps are constructed `alwaysRun=false` and `skip()` returns true unless `-a clean` is passed. The default run is `SendMAEStep` alone — purely **additive**.
> **Corrects a premise in the brief.** Consequence: nothing in the automatic upgrade path ever deletes a search/graph document for an entity that has vanished from SQL, so stale documents accumulate.

**`C-145` `VERIFIED` — `RemoveUnknownAspects` does NOT purge unregistered aspects.** `buildSteps` adds exactly one step, `RemoveClientIdAspectStep`, deleting a single hardcoded telemetry aspect. It is also a standalone job, never part of `SystemUpdate`.
> **Corrects my own premise.** A curator with custom aspects faces **no deletion risk** from this job.

**`C-146` `VERIFIED` — the real custom-aspect hazard is SILENT UNREADABILITY, not deletion.** If a custom model plugin fails to load on the new version, GMS **still starts** (`ignoreFailureWhenLoadingRegistry` defaults true), leaving those aspects absent from the registry. `restoreIndices` then hits its step-3 check, cannot find the `AspectSpec`, and does `ignored++; continue` for every such row. **The rows remain intact in SQL but silently vanish from search and graph on the next reindex. The only signal is a log line.**

**`C-147` `VERIFIED` — THE RECOVERY TOOL IS ITSELF A TOTAL-LOSS VECTOR.** `RestoreBackup` runs `ClearAspectV2TableStep` **unconditionally** — no flag, no condition, no `skip()` override — executing `_server.find(EbeanAspectV2.class).delete()` before restoring, and runs all three `Clear*Step` with `alwaysRun=**true**` (the inverse of `RestoreIndices`). Its own javadoc misleadingly calls the wipe an *"Optional step"*.

**`C-148` `VERIFIED` — the encryption key is a single point of permanent loss with a dangerous default.** `encryptionKey: "#{systemEnvironment['SECRET_SERVICE_ENCRYPTION_KEY'] ?: 'ENCRYPTION_KEY'}"` — it **silently falls back to the literal string `ENCRYPTION_KEY`**. HMAC-derived deterministically, no salt, **no key id stored beside the ciphertext, and no rotation or re-encryption utility anywhere in the repo.**
**The counter-intuitive trap: an installation running unset has been encrypting under a publicly-known key, and correctly SETTING a real key at upgrade time is what breaks decryption of everything already stored.**

**`C-149` `VERIFIED` — default access policies are re-UPSERTed on EVERY deploy.** `IngestPoliciesUpgradeStep` is a **blocking** step whose `skip()` checks only the enabled flag — **no idempotency marker** — re-UPSERTing every `"editable": false` policy. That is **11 of the 16** policies in `boot/policies.json`, including the Admin, Editor and Reader Platform and Metadata policies. Out-of-band edits are overwritten before GMS starts.

**`C-150` `VERIFIED` — bootstrap templates overwrite customised system entities on a version bump.** The idempotency key is `bootstrap-<name>-<version>`, so bumping a template version re-runs the whole template blocking, and its `changeType: UPSERT` entries replace the current aspect. `CREATE` entries are safely rejected by `CreateIfNotExistsValidator`.

**`C-151` `VERIFIED` — `helm uninstall` DROPS THE DATABASE, but this CANNOT fire during a version upgrade.** `CLEANUP_SQL_ENABLED`/`CLEANUP_ELASTICSEARCH_ENABLED`/`CLEANUP_KAFKA_ENABLED` all default **true**, and `DropDatabaseStep` issues `DROP DATABASE IF EXISTS`. But `CleanupCondition` matches only when the CLI non-option args contain the literal string `"Cleanup"`, so the entire Spring configuration is absent from a normal `SystemUpdate` run. Class javadoc: *"Intended to run as a Helm pre-delete hook."* **Recalibrated from an upgrade hazard to a helm-uninstall hazard.**

**`C-152` `VERIFIED` — a DB restore REWINDS upgrade state.** `dataHubUpgradeResult` markers (including each backfill's `lastUrn` cursor) are ordinary versioned aspects on `urn:li:dataHubUpgrade:*` **in the same table as the curated metadata**. Restoring a point-in-time SQL dump replays every backfill, re-applies every bootstrap MCP UPSERT and re-runs the blocking policy step. **A DB restore must be planned as a full re-bootstrap, not a quiet rollback.**

**`C-153` `VERIFIED` — the zero-downtime path is OFF in every released version.** `incrementalReindexEnabled` and `rollbackDualWriteEnabled` both default to `${ZDU_STAGE_20:false}`, and `ZDU_STAGE_20` is set by a Helm chart **not in this repository** (`datahub-kubernetes/` contains only a README). Budget for the **legacy path**: blocked index writes, an alias swap that deletes the old backing index, and K8s consumer scale-down. `ELASTICSEARCH_INDEX_BUILDER_MAPPINGS_REINDEX` also defaults false — **the quickstart compose files set it true and must not be carried into production.**

**`C-154` `VERIFIED` — MySQL only: an upgrade can lock the whole aspect table for hours.** `SqlSetup` detects non-binary collation on `metadata_aspect_v2.urn`/`aspect` and issues `ALTER TABLE … CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin`, rebuilding and locking the table. The `v1.6.0.1` note states **there is no env var to skip the check** and puts the duration at minutes to hours. PostgreSQL is unaffected.

**`C-155` `VERIFIED` — there is NO documented version downgrade.** The only downgrade statement in the ledger sits under the historical `## 0.14.1` heading; nothing equivalent exists for v1.5.0–v1.7.0 or `## Next`. `rollbackDualWriteEnabled` is ES-index safety during incremental reindex, **not** an application downgrade.

**`C-156` `VERIFIED` — config drift after upgrade is silent.** There is no unknown-property validator on the GMS config surface; a removed or renamed env var left set is simply never looked up. `VIEW_RESTRICTED_ENTITY_TYPES`, replaced in v1.7.0, has **zero references** in any `.java` or `.yaml` at HEAD — an operator leaving it set believes their allowlist is in force when behaviour has flipped to restricted-by-default.

**`C-157` `VERIFIED` — feature-flag reality check: `FeatureFlags.java` is NOT the gating oracle.** Its Java field initialisers are overridden by `application.yaml`. Reading the Java file alone produces wrong conclusions in at least five cases.
Effective defaults for the RQ-relevant flags: **OFF** — `entityVersioning`, `businessAttributeEntityEnabled`, `schemaFieldCLLEnabled`, `showSeparateSiblings`, `showHasSiblingsFilter`, `MCP_SIDE_EFFECTS_SCHEMA_FIELD_ENABLED`. **ON** — `schemaFieldEntityFetchEnabled`, `showManageStructuredProperties`, `showBrowseV2`, `logicalModelsEnabled`, `dataProcessInstanceEntityEnabled`, `ENABLE_SIBLING_HOOK`.
> **Correction to the brief's method rule 4d.**

**`C-158` `VERIFIED` — a flag's default can DIFFER between two live release lines.** `LOGICAL_MODELS_ENABLED` defaults **false** in `v1.6.0.1` and **true** in `v1.7.0`. Since `v1.6.0.1` is dated *later*, a team on the newest-dated tag gets the feature **off**.

---

## §3 Schema / attribute inventory

All pinned to HEAD `7d5cedd5f8`. "Rel." = earliest confirmed release containment.

### 3.1 Write envelope

| Type | Path | Required fields | Rel. |
|---|---|---|---|
| `MetadataChangeProposal` | `metadata-models/.../mxe/MetadataChangeProposal.pdl` | `entityType`, `changeType`; exactly one of `entityUrn`/`entityKeyAspect` | v0.8.7 |
| `ChangeType` (enum) | `.../events/metadata/ChangeType.pdl` | 7 values: `UPSERT, CREATE, UPDATE, DELETE, PATCH, RESTATE, CREATE_ENTITY` | core |
| `SystemMetadata` | `.../mxe/SystemMetadata.pdl` | all optional; `lastObserved, runId, lastRunId, pipelineName, registryName/Version, properties, version, schemaVersion, aspectCreated, aspectModified` | core |
| `GenericJsonPatch` | OpenAPI/GraphQL envelope | `patch`; optional `arrayPrimaryKeys`, `forceGenericPatch` | v1.4.0 |

### 3.2 Identity

| Type | Path | Required fields | Rel. |
|---|---|---|---|
| `DatasetKey` | `.../metadata/key/DatasetKey.pdl` | `platform: Urn`, `name: string`, `origin: FabricType` | core |
| `FabricType` | `li-utils/.../FabricType.pdl` | 17 values; `PROD` is the Python default | core |
| `ContainerKey` | `.../metadata/key/ContainerKey.pdl` | `guid: optional string` (md5 of sorted-JSON coordinates) | core |
| `Container` (aspect) | `.../container/Container.pdl` | `container: Urn`, `@Relationship IsPartOf` | core |
| `DataPlatformInstance` | `.../common/DataPlatformInstance.pdl` | `platform: Urn`; `instance: optional Urn`. **Zero `@Relationship`** | core |
| URN limits | `metadata-utils/.../UrnValidationUtil.java` | `URN_NUM_BYTES_LIMIT = 512` (URL-encoded); no whitespace; no `␟` | core |

### 3.3 Provenance

| Type | Path | Required fields | Rel. |
|---|---|---|---|
| `UpstreamLineage` | `.../dataset/UpstreamLineage.pdl` | `upstreams: array[Upstream]`; `fineGrainedLineages` optional | **2020-05-21** |
| `Upstream` | `.../dataset/Upstream.pdl` | `auditStamp`, `dataset: DatasetUrn`, **`type: DatasetLineageType`**; optional `created`, `properties: map[string,string]`, `query: Urn`, `matchType` | 2020-05-21 |
| `DatasetLineageType` | `.../dataset/DatasetLineageType.pdl` | `COPY` \| `TRANSFORMED` \| `VIEW` | 2020-05-21 |
| `Siblings` | `.../common/Siblings.pdl` | `siblings: array[Urn]`, **`primary: boolean` (required, no default)** | 2022-06-22 |
| `FineGrainedLineage` | `.../dataset/FineGrainedLineage.pdl` | `upstreamType`, `downstreamType`; optional `transformOperation`, `confidenceScore = 1.0` | core |
| `DataJobInputOutput` | `.../datajob/DataJobInputOutput.pdl` | `inputDatasetEdges`/`outputDatasetEdges` — both `isLineage: true` | core |
| `LineageRelationship` (read) | `.../metadata/graph/LineageRelationship.pdl` | `type: string` = relationship **name**; **no `DatasetLineageType`, no properties map** | core |

### 3.4 Files and profiles

| Type | Path | Required fields | Rel. |
|---|---|---|---|
| `SchemaMetadata` | `.../schema/SchemaMetadata.pdl` | **6 required**: `schemaName` (1–500 chars), `platform`, `version`, `hash`, `platformSchema` (union), `fields` | core |
| `platformSchema` union members | — | `EspressoSchema, OracleDDL, MySqlDDL, PrestoDDL, KafkaSchema, BinaryJsonSchema, OrcSchema, Schemaless, KeyValueSchema, OtherSchema` — **no CSV, no plain JSON** | core |
| `EditableSchemaMetadata` | `.../schema/EditableSchemaMetadata.pdl` | `editableSchemaFieldInfo: array[EditableSchemaFieldInfo]` | 2021-03-18 |
| `DatasetProfile` (timeseries) | `.../dataset/DatasetProfile.pdl` | `timestampMillis` (inherited) only; `rowCount`, `columnCount`, `fieldProfiles`, `sizeInBytes` all optional | 2021-07-30 |
| `DatasetFieldProfile` | `.../dataset/DatasetFieldProfile.pdl` | **`fieldPath` only**; 13 optional stats incl. `histogram`; min/max/mean/median/stdev are **strings** | 2021-07-30 |
| `TimeseriesAspectBase` | `.../timeseries/TimeseriesAspectBase.pdl` | `timestampMillis: long`; `partitionSpec` default `{FULL_TABLE, FULL_TABLE_SNAPSHOT}` | 2021-07-30 |
| `Deprecation` | `.../common/Deprecation.pdl` | `deprecated`, `note`, `actor`; optional `decommissionTime`, `replacement: Urn` (**no `@Relationship`**) | core |
| `Status` | `.../common/Status.pdl` | `removed: boolean = false`; optional `lifecycleStage: Urn`, `lifecycleLastUpdated` | lifecycleStage 2026-04-15, in v1.6.0 |

### 3.5 Classification and extension

| Type | Path | Required fields | Rel. |
|---|---|---|---|
| `GlossaryTermInfo` | `.../glossary/GlossaryTermInfo.pdl` | `definition`, **`termSource: string`**; optional `sourceRef`, `sourceUrl`, `parentNode`, `customProperties` | 2021-05-10 |
| `GlossaryTermKey` | `.../metadata/key/GlossaryTermKey.pdl` | `name: string` ("serves as a unique id") | core |
| `GlossaryRelatedTerms` | `.../glossary/GlossaryRelatedTerms.pdl` | four optional arrays → `IsA`, `HasA`, `HasValue`, `IsRelatedTo` | core |
| `StructuredPropertyDefinition` | `.../structured/StructuredPropertyDefinition.pdl` | **`qualifiedName`, `valueType`, `entityTypes`**; `cardinality = SINGLE`, `immutable = false` | 2024-01-22 (v0.13.1) |
| `StructuredPropertySettings` | `.../structured/StructuredPropertySettings.pdl` | 6 booleans incl. `showInColumnsTable` | 2024-12-11 |
| `schemaField` entity | `entity-registry.yml:596-618` | keyAspect `schemaFieldKey`; **16 aspects** incl. `structuredProperties`, `glossaryTerms`, `globalTags` | core |
| `CustomProperties` | `.../common/CustomProperties.pdl` | `customProperties: map[string,string] = {}`, `@Searchable TEXT queryByDefault` | core |
| Custom registry | `metadata-models-custom/` | registry file **must** be `entity-registry.yaml`/`.yml`; PDL filename **must** match class; version via gradle `projVersion`, default `0.0.0-dev` | core |

### 3.6 Credentials

| Type | Path | Required fields | Rel. |
|---|---|---|---|
| `DataHubSecretValue` | `.../secret/DataHubSecretValue.pdl` | `name: string` (**plaintext, `@Searchable`**), `value: string` (AES-256-GCM), optional `description` | 2022-01-27 |
| `DataHubConnectionKey` | `.../metadata/key/DataHubConnectionKey.pdl` | `id: string` | 2024-05-20 |
| `DataHubConnectionDetails` | `.../connection/DataHubConnectionDetails.pdl` | `type: {JSON}`; optional `name`, `json`. **Zero `@Relationship`** | 2024-05-20 |
| `DataHubJsonConnection` | `.../connection/DataHubJsonConnection.pdl` | `encryptedBlob: string` | 2024-05-20 |
| `SecretService` caller guard | `metadata-operation-context/.../SecretService.java:241-300` | mode `ENFORCE` by default | **v1.7.0** (`08a9b1bd15`, #17995) |

---

## §4 Design decisions

**`D-01` — Register with the key aspect alone; accrue everything else by PATCH.**
*Rests on:* `C-09`, `C-10`, `C-11`, `C-15`, `C-50`, `C-98`.
*Rejected:* whole-aspect UPSERT per discovery. **Why:** `C-01` — every UPSERT destroys unmentioned fields of that aspect, and `C-06` shows even the UI's read-modify-write is racy. With months of manual accrual by multiple people, UPSERT guarantees eventual silent data loss.
*Rejected:* modelling a synthetic "curation job". **Why:** unnecessary — `C-09` proves the MCP shape has no run concept. (This was OpenLineage's forced workaround for profiles; DataHub does not need it.)

**`D-02` — Use OpenAPI v3 or the Python SDK for writes; explicitly pass `arrayPrimaryKeys` (or `forceGenericPatch: true`).**
*Rests on:* `C-05`, `C-12`, `C-21`, `C-22`, `C-23`.
*Rejected:* the bare OpenAPI v3 single-aspect POST with defaults. **Why:** `C-22` — it defaults to `ChangeType.CREATE`, so the second write to any aspect **fails**. This is the single most likely first-day surprise.
*Rejected:* relying on the template route. **Why:** `C-12` lists only 23 templated aspects, and `C-17` shows the Python `FieldPatchHelper` builds `schemaMetadata` patches the server cannot service.

**`D-03` — Write ALL human curation to the `editable*` aspects, never to `schemaMetadata`.**
*Rests on:* `C-13`, `C-95`, `C-96`, `C-97`, `C-98`.
*Rejected:* curating `schemaMetadata.fields[].glossaryTerms`. **Why:** `C-13` — no patch template exists, so every write replaces the whole `fields[]` array; and the model's own doc comment says the split exists *"to avoid accidental overwrites of user-provided data by ingestion pipelines"*. When the real datasource later becomes reachable and ingestion runs, curation in `schemaMetadata` is destroyed; curation in `editableSchemaMetadata` survives.

**`D-04` — For the replica→origin edge, use `UpstreamLineage` with `type=COPY`, added by PATCH.**
*Rests on:* `C-48`, `C-49`, `C-50`, `C-63`.
*Rejected:* **`Siblings` — this is the anti-pattern.** **Why:** `C-58` — `SiblingGraphService` **actively deletes** any lineage relationship between two siblings from the merged response, `C-59` shows merging is the default, and `C-60` shows the frontend overwrites the sibling's URN to collapse them into one node. Siblings asserts *"these are the same thing"*; RQ-4 requires asserting *"this one is derived from that one"*. **Using it destroys the fact being documented.**
*Rejected:* `logicalParent`. **Why:** `C-66` — `LogicalParentPlatformValidator` forbids physical→physical entirely; it would require inventing a third synthetic logical dataset.
*Rejected:* `ERModelRelationship`. **Why:** its own field docs say *"no directionality"*, it requires column mappings, neither `@Relationship` sets `isLineage`, and it is flag-gated off.
*Rejected:* `versionSet`/`VersionProperties`. **Why:** replicas are concurrent peers, not ordered versions; the version set has a single `latest` pointer that would wrongly demote one; and `entityVersioning` is off by default (`C-89`).
*Rejected:* `Origin.pdl`. **Why:** `C-65` — a naming trap; it is about identity-provider provenance and has no URN field at all.
*Rejected:* a datasource/connection descriptor. **Why:** `C-67` — verified absence; there is no way to relate two platform instances.

**`D-05` — Record the "HOW" redundantly, in three places, because no single one survives every read path.**
*Rests on:* `C-51`, `C-52`, `C-53`, `C-54`, `C-55`, `C-68`.
1. `Upstream.type = COPY` — durable, authoritative, readable via aspect APIs.
2. `Upstream.properties {replicationMethod: "..."}` — reaches the graph edge document (`C-51`) but has no declared mapping beyond `source` (`C-52`).
3. A **structured property** on the replica dataset — typed, searchable, filterable, and it **renders in the UI** (`C-104`), which none of the above do.
*Rejected:* relying on `Upstream.type` alone. **Why:** `C-53` — it is structurally absent from every traversal result **and from the ES edge document itself**; `C-54` — deprecated and unreferenced in GraphQL; `C-55` — the UI hardcodes `TRANSFORMED`. A curator would record the fact and never see it again.
*Optional composition:* a `DataJob` node (`C-68`) is the only mechanism that makes HOW graph-visible, at the cost of modelling a process that never runs.

**`D-06` — Derive the user-visible identifier from `(platform, name, fabric)` and commit to the environment up front.**
*Rests on:* `C-29`, `C-30`, `C-31`, `C-32`, `C-47`.
*Rejected:* encoding the platform instance later. **Why:** `C-31` — `platform_instance` is concatenated into `name`, so adding it **changes the dataset's identity**. It must be decided at registration or not used.
*Rejected:* putting a DSN in the `name`. **Why:** `C-30` — the model explicitly redirects native naming to `DatasetProperties`; and `C-34` caps the URN at 512 URL-encoded bytes.

**`D-07` — Store credentials in `dataHubSecret`/`dataHubConnection`; enforce the no-credentials-in-URN rule with a custom `AspectPayloadValidator`.**
*Rests on:* `C-32`, `C-33`, `C-36`, `C-37`, `C-38`, `C-40`, `C-110`.
*Rejected:* relying on the identifier grammar to exclude credentials. **Why:** `C-33` — there is **no normative prohibition**, and `C-35` shows `@ : / = ?` are all legal in a dataset name. The exclusion is an accident of API shape.
*Rejected:* porting a sanitiser from the other language. **Why:** `C-36` — **this inverts the prior run's finding.** OpenLineage had `JdbcUrlSanitizer` in Java to port to Python; DataHub has **no canonical implementation in either language**. This is net-new code, not a port.
*Chosen enforcement:* `C-110` — a custom registry can ship an `aspectPayloadValidator` that runs uniformly across GraphQL/OpenAPI/Rest.li, which is exactly the pattern `AGENTS.md` prescribes for validation. **But see `D-11` and `R-09`: shipping custom Java has an upgrade cost.**

**`D-08` — Use glossary terms for the portable ontology projection at BOTH granularities; use structured properties for the structure.**
*Rests on:* `C-90`, `C-92`, `C-98`, `C-99`, `C-101`, `C-104`, `C-115`.
- Dataset level: `glossaryTerms` aspect.
- Column level: `editableSchemaMetadata` (patchable two-level merge, `C-98`) as the primary route; the `schemaField` entity's own `glossaryTerms`/`structuredProperties` as the second — **but note `C-100`, schemaField entities are not auto-created at default config.**
- Ontology metadata (IRI, ontology version, mapping confidence, system-of-record id): structured properties on the **term** (`C-115`), which keeps `sourceRef`/`sourceUrl` clean.
*Rejected:* `customProperties`. **Why:** `C-117` — untyped, entity-level only, never reaches columns.
*Rejected:* `BusinessAttribute`. **Why:** `C-112` — BETA, off by default, UI-only creation, one per field.
*Rejected:* a custom aspect for ontology data. **Why:** `C-14` (no patch templates for custom aspects) and `C-146` (a plugin that fails to load makes the data silently invisible). Structured properties are **data**, not compiled model code — see `D-11`.

**`D-09` — Model the CSV/JSON extract as a `file`-platform dataset; supersede it with `deprecation.replacement` + a lineage edge, never delete it.**
*Rests on:* `C-80`, `C-81`, `C-82`, `C-87`, `C-88`, `C-89`.
*Rejected:* `status.removed = true`. **Why:** its own doc says it represents *soft deletes*; RQ-2 says superseded, never deleted.
*Rejected:* `versionSet`. **Why:** `C-89` — gated off by default.
*Rejected:* `status.lifecycleStage`. **Why:** `C-88` — released and unflagged, but `bootstrap_mcps/lifecycle-stages.yaml` ships as `[]`, so a custom `lifecycleStageType` must be authored first. **Reconsider once a stage is defined; it is the semantically correct answer.**
*Accepted constraint:* `C-80` — file format is unmodelled. Record it in `subTypes` and `customProperties` by local convention, and accept that this is **weaker than the baseline**, which has `datasetType: FILE` and `storage.fileFormat` as first-class.

**`D-10` — Attach data profiles as plain timeseries MCPs, and treat every profile write as whole-aspect.**
*Rests on:* `C-71`, `C-74`, `C-75`, `C-76`, `C-77`.
*Rejected:* modelling a profiling job. **Why:** `C-71` — unnecessary; this is exactly where DataHub beats the baseline.
*Accepted constraint:* `C-05`/`C-140` — timeseries accepts **UPSERT only**, so accrual is whole-aspect-per-timestamp. A partial profile write at the same `timestampMillis` **replaces** the earlier one. Build the complete `DatasetProfile` client-side, then emit.
*Use `messageId`* as the idempotency key when retrying at the same timestamp.

**`D-11` — Prefer structured properties over custom aspects for every extension, and treat this as an UPGRADE decision, not a modelling one.**
*Rests on:* `C-14`, `C-101`, `C-104`, `C-105`, `C-107`, `C-139`, `C-146`.
A structured property definition is **data in `metadata_aspect_v2`**: it is backed up by the SQL dump, survives upgrades with no DDL (`C-139`), round-trips through the CLI (`C-105`), and renders in the UI (`C-104`).
A custom aspect is **compiled model code**: it gets no patch templates (`C-14`), and if its plugin fails to load on a new version the curated rows survive in SQL but **silently vanish from search and graph** with only a log line (`C-146`).
*Rejected:* forking DataHub to add aspects. **Why:** `C-107`/`C-108` — the sidecar plugin path exists and is supported; forking adds the full backport burden of `C-126`–`C-128` on top.

**`D-12` — Treat the Elasticsearch snapshot as a first-class backup of equal standing to the SQL dump, and re-home genuinely hand-entered profile facts.**
*Rests on:* `C-139`, `C-140`, `C-141`, `C-142`.
*Rejected:* backing up only the relational store. **Why:** `C-140` — a `mysqldump` of `metadata_aspect_v2` contains **zero** `datasetProfile` bodies, and neither `RestoreIndices` nor `LoadIndices` can regenerate them.
*Design consequence:* if a number is **hand-entered by a curator** (e.g. "the provider told us this table has ~4M rows"), record it as a **structured property or `editableDatasetProperties`** — versioned aspects, therefore backed up and rebuildable — and reserve `datasetProfile` for machine-harvested output a profiler can re-derive.

---

## §5 End-to-end flow

### 5.1 Stages

| ID | Stage | Trigger | Event / API call | Entity | Key attributes | Idempotent |
|---|---|---|---|---|---|---|
| `S-0` | Define ontology properties (once) | Programme setup | `datahub properties upsert -f props.yaml` | `structuredProperty` | `qualifiedName`, `valueType`, `entityTypes: [dataset, schemaField, glossaryTerm]`, `cardinality`, `allowedValues` | **Yes** |
| `S-1` | Import ontology | Ontology published | `rdf` source (ALPHA) **or** `business_glossary` YAML | `glossaryTerm`, `glossaryNode` | `termSource=EXTERNAL`, `sourceRef=<IRI>`, `sourceUrl=<IRI>` | **Yes** |
| `S-2` | Register the CSV stand-in | Analyst downloads an export | MCP `CREATE_ENTITY` of `datasetKey` | `dataset` (`platform=file`) | `(file, <path-derived name>, PROD)` | **Yes** (`CREATE_ENTITY` guard) |
| `S-3` | Add what is known | Analyst learns a fact | **PATCH** `datasetProperties` (`/description`, `/name`, `/customProperties/<k>`) | same | scalar paths only | **Yes** |
| `S-4` | Attach the CSV schema | Columns inspected | UPSERT `schemaMetadata` | same | 6 required fields; `platformSchema=OtherSchema(rawSchema=<header>)` | **Yes** (whole-aspect) |
| `S-5` | Profile the CSV | Profiler run or hand entry | UPSERT `datasetProfile` (timeseries) | same | `timestampMillis`, `rowCount`, `fieldProfiles[]`, optional `messageId` | **Yes** per `(timestamp, messageId, partitionSpec)` |
| `S-6` | Classify (dataset) | Ontology mapping decided | **PATCH** `glossaryTerms`, `structuredProperties` | same | keyed by term urn / `propertyUrn` | **Yes** |
| `S-7` | Classify (column) | Column mapping decided | **PATCH** `editableSchemaMetadata` | same | two-level merge: `fieldPath` → term `urn` | **Yes** |
| `S-8` | Register the replica | Replica access granted | MCP `CREATE_ENTITY` of `datasetKey` | `dataset` (`platform=snowflake`) | `(snowflake, db.schema.table, PROD)` | **Yes** |
| `S-9` | Register the origin | Provider confirms the source | MCP `CREATE_ENTITY` of `datasetKey` | `dataset` (`platform=oracle`) | `(oracle, DB.SCHEMA.TABLE, PROD)` | **Yes** |
| `S-10` | Place both in the hierarchy | Coordinates known | UPSERT `container` on each | `container` entities | `IsPartOf` edge; container GUID = md5 of coordinates | **Yes** |
| `S-11` | **Assert provenance** | Replica relationship confirmed | **PATCH** `upstreamLineage` at `("upstreams", <oracle urn>)` | replica dataset | `Upstream{dataset=<oracle>, type=COPY, auditStamp, properties{replicationMethod}}` | **Yes** (keyed on `dataset`) |
| `S-12` | Make the HOW visible | same | **PATCH** `structuredProperties` | replica dataset | `io.acme.provenance.method = "REPLICATION"` | **Yes** |
| `S-13` | Store credentials | Access provisioned | `createSecret` / `upsertConnection` (GraphQL) | `dataHubSecret`, `dataHubConnection` | opaque `id`; `encryptedBlob`; **no dataset link** | **Yes** |
| `S-14` | Supersede the CSV | Replica becomes the source | UPSERT `deprecation` on the CSV + **PATCH** `upstreamLineage` on the replica | CSV dataset | `deprecated=true`, `note`, `actor`, `replacement=<replica urn>` | **Yes** |
| `S-15` | Verify completeness | Curation review | UPSERT `forms` / `formInfo` | dataset | `FormType.VERIFICATION`, `completedForms`, `verifications` | **Yes** |

**Non-idempotent operations to avoid:** OpenAPI v3 single-aspect POST with default `createIfNotExists=true` (`C-22`); `datahub dataset upsert` with a YAML lacking `properties:` (`C-84`); any whole-aspect UPSERT of `structuredProperties` from a connector (`C-106`).

### 5.2 Resulting graph

```
                        urn:li:structuredProperty:io.acme.provenance.method
                                        ▲ (typed, searchable, UI-rendered)
                                        │
  urn:li:container:<md5>                │
        ▲ IsPartOf                      │
        │                               │
  ┌─────┴──────────────────┐   ┌────────┴──────────────────┐
  │ dataset                │   │ dataset                   │
  │ (oracle, DB.SCH.TBL,   │◀──│ (snowflake, db.sch.tbl,   │
  │  PROD)   "the origin"  │   │  PROD)   "the replica"    │
  └────────────────────────┘   └───────────────────────────┘
        ▲   DownstreamOf (isLineage:true)      ▲
        │   Upstream{type=COPY}                │ replacement (NOT an edge)
        │                                      │
        │                          ┌───────────┴───────────┐
        │                          │ dataset               │
        │                          │ (file, exports/x.csv, │
        │                          │  PROD)  "the stand-in"│
        │                          │  + datasetProfile[]   │  ← ES-only, no SQL copy
        │                          │  + deprecation        │
        │                          └───────────────────────┘
        │
        └── glossaryTerms ──▶ urn:li:glossaryTerm:<ontology-derived id>
                                  termSource=EXTERNAL, sourceRef=<IRI>

  schemaField entities ──▶ editableSchemaMetadata.editableSchemaFieldInfo[fieldPath].glossaryTerms

  dataHubConnection / dataHubSecret   ── NO EDGE TO ANY DATASET (separated by construction)
```

**Deliberately NOT in this graph:** a `Siblings` aspect between the Oracle original and the Snowflake replica. Adding one would cause `SiblingGraphService` to **delete the `COPY` edge** from the default lineage response (`C-58`) and the UI to collapse the two datasets into one node (`C-60`).

### 5.3 Requirement → mechanism traceability

| Req | Mechanism | Stages | Claims |
|---|---|---|---|
| `RQ-1` | Key-aspect registration + PATCH accrual + `editable*` aspect split + aspect versioning + Forms | `S-2`…`S-7`, `S-15` | `C-09`, `C-10`, `C-11`, `C-15`, `C-26`, `C-95`, `C-98` |
| `RQ-2` | `file`-platform dataset + `schemaMetadata(OtherSchema)` + `datasetProfile` (run-less) + `deprecation.replacement` | `S-2`, `S-4`, `S-5`, `S-14` | `C-71`, `C-81`, `C-87`; constrained by `C-80`, `C-140` |
| `RQ-3` | `(platform, name, fabric)` URN tuple + `container` graph hierarchy | `S-8`, `S-9`, `S-10` | `C-29`, `C-43`; constrained by `C-46`, `C-31` |
| `RQ-4` | `UpstreamLineage.Upstream{type=COPY}` + edge `properties` + structured property | `S-11`, `S-12` | `C-48`, `C-49`, `C-50`; constrained by `C-53`; **`Siblings` rejected by `C-58`** |
| `RQ-5` | URN excludes credentials by API shape + `dataHubSecret`/`dataHubConnection` + caller guard | `S-13` | `C-30`, `C-32`, `C-37`, `C-38`, `C-40`; gap `C-33`, `C-36` |
| `RQ-6` | `glossaryTerms` (dataset + column) + `structuredProperties` (incl. `schemaField`) + RDF importer | `S-0`, `S-1`, `S-6`, `S-7` | `C-90`, `C-92`, `C-99`, `C-101`, `C-104`; gap `C-94` |
| `RQ-7` | `metadata_aspect_v2` opaque-blob durability + ES snapshot + structured-properties-over-custom-aspects | — | `C-139`, `C-140`, `C-142`, `C-146` |

---

## §6 Risks

| ID | Risk | Sev | Mitigation |
|---|---|---|---|
| `R-01` | **Timeseries total loss with no rebuild path.** Hand-curated profiles exist **only** in Elasticsearch. A `mysqldump` contains zero profile bodies; `RestoreIndices`/`LoadIndices` cannot regenerate them (`C-140`). | **HIGH** | Treat the ES snapshot as a first-class backup taken at the **same instant** as the SQL dump; verify a real restore into a test cluster quarterly. **Structurally:** re-home hand-entered facts to structured properties / `editableDatasetProperties` (versioned aspects) and reserve `datasetProfile` for re-derivable machine output (`D-12`). |
| `R-02` | **The documented ES restore command matches ZERO DataHub indices.** `docs/how/backup-datahub.md:198` says `"indices": "datastream*,metadataindex*"`. Real names are `*index_v2`, `<entity>_<aspect>aspect_v1`, `graph_service_v1`, `system_metadata_service_v1`. `"datastream"` appears **nowhere else in the repo**. The snapshot side is correct, so the backup looks healthy for years and fails only at recovery — on the one asset class with no rebuild path. | **HIGH** | Do not copy the doc's restore body. Use patterns derived from code: `*index_v2,*aspect_v1,graph_service_v1,system_metadata_service_v1` (prefix if `ELASTICSEARCH_INDEX_PREFIX` is set). Assert a non-zero doc count on `dataset_datasetprofileaspect_v1` specifically before trusting any backup. |
| `R-03` | **Encryption-key loss or correction permanently destroys all stored secrets.** The key falls back to the literal `ENCRYPTION_KEY`; HMAC-derived, no salt, no key id, **no rotation tooling** (`C-148`). An installation running unset has been encrypting under a publicly-known key, and *correctly setting* a key at upgrade time is what breaks decryption. | **HIGH** | Determine whether `SECRET_SERVICE_ENCRYPTION_KEY` is set **before** upgrading. If not, export every secret's plaintext out-of-band first, **then** set the key, then re-enter them. Store it in the same vault tier as the DB master credential. Leave `SECRET_SERVICE_V1_ALGORITHM_ENABLED` at its default `true`. |
| `R-04` | **The recovery tool destroys the authoritative store first.** `RestoreBackup` runs `ClearAspectV2TableStep` unconditionally (`_server.find(EbeanAspectV2.class).delete()`) and all three `Clear*Step` with `alwaysRun=true` (`C-147`). A stale or unreadable backup turns a recovery attempt into total destruction. | **HIGH** | Never run `RestoreBackup` against a live instance. Restore SQL by ordinary DB means and rebuild with `RestoreIndices` (additive by default). If it must be used, rehearse end-to-end against the exact backup file first. |
| `R-05` | **Silent clobbering by whole-aspect UPSERT.** Any tool or teammate emitting an UPSERT of `glossaryTerms`, `structuredProperties`, `ownership` or `editableSchemaMetadata` destroys every value that write did not mention (`C-01`, `C-106`). | **HIGH** | Mandate PATCH for every curation write. Set `StructuredPropertyWriteMode.PATCH` on every connector. Code-review any `UPSERT` of a curation-bearing aspect. Rely on attribution-keyed arrays (`C-15`) so independent writers own separate slots. |
| `R-06` | **`Siblings` silently deletes the provenance edge.** If anyone adds a siblings link between the origin and the replica — via `datahub dataset add_sibling`, which the CLI docs present with a Hive→Snowflake example — the `COPY` edge disappears from the default lineage view (`C-58`). | **HIGH** | Forbid `Siblings` between the origin and replica by policy. Add a CI assertion that no dataset in the catalogue carries both a `siblings` aspect and an `upstreamLineage` edge to a member of that sibling set. If a merged UX is later wanted, enable `SHOW_SEPARATE_SIBLINGS=true` first and re-verify lineage. |
| `R-07` | **The provenance type is invisible everywhere a human looks.** `COPY` is absent from every traversal result and the ES edge document, deprecated in GraphQL, unreferenced in React, and the UI hardcodes `TRANSFORMED` (`C-53`–`C-55`). Curators will record it once and never see it again. | **HIGH** | Apply `D-05`: record the HOW **three times** — `Upstream.type`, `Upstream.properties`, and a **structured property**, the last of which is the only one that renders. Never let a curator edit lineage from the UI, since that write hardcodes `TRANSFORMED`. |
| `R-08` | **A custom model plugin that fails to load makes curated data invisible without deleting it.** GMS still starts (`ignoreFailureWhenLoadingRegistry` default true) and `restoreIndices` does `ignored++; continue` for every affected row (`C-146`). | **HIGH** | Set `IGNORE_FAILURE_WHEN_LOADING_ENTITY_REGISTRY_PLUGIN=false` in staging. Make `curl -s http://<gms>/config \| jq .models` a mandatory post-upgrade gate asserting `loadResult == SUCCESS`. **Strategically, prefer structured properties (`D-11`).** |
| `R-09` | **No credential guard exists in either language.** No normative prohibition (`C-33`), no shared sanitiser in Python **or** Java (`C-36`), and `@ : / = ?` are all legal in a dataset name (`C-35`). | **HIGH** | This is **net-new code, not a port** — the inverse of the prior run. Ship a custom `AspectPayloadValidator` (`C-110`) rejecting DSN-shaped `datasetKey.name` values across all APIs at once, plus a client-side check in the registration tool. Note `C-34`'s 512-byte URN cap already blocks long DSNs but is not a security control. |
| `R-10` | **A green rebuild that indexed nothing.** `RestoreIndices` reports SUCCEEDED once MCLs are *produced*, not indexed (`C-143`). Consumers scaled down for the maintenance window produce a clean run and an empty index. | MED | Never accept exit status as proof. Scale consumers **up** first, assert consumer-group lag returns to zero, and compare `SELECT COUNT(*) FROM metadata_aspect_v2 WHERE version=0` against `GET _cat/indices/*index_v2?v`. Grep the job log for the `ignored` tally. Consider `LoadIndices` (synchronous). |
| `R-11` | **Customised default access policies are silently reverted on every upgrade.** `IngestPoliciesUpgradeStep` re-UPSERTs 11 of 16 policies with no idempotency marker, before GMS starts (`C-149`). | MED | Never modify the shipped `editable: false` policies. Express custom access control as **new** policy URNs; to neutralise a default, add an overriding policy. Add a post-upgrade assertion that diffs the policy set. |
| `R-12` | **Bootstrap templates overwrite customised system entities on a version bump** (`C-150`). | MED | Do not customise system-owned URNs (`urn:li:dataPlatform:*`, `urn:li:ownershipType:__system__*`, `urn:li:corpuser:datahub`). Before each upgrade, diff `bootstrap_mcps.yaml` between tags and check each template's `changeType` — `UPSERT` clobbers, `CREATE` is safe. |
| `R-13` | **A DB restore rewinds upgrade state and re-arms every migration** (`C-152`). | MED | Plan a DB restore as a full re-bootstrap: budget backfill re-run time, expect blocking bootstrap and policy steps to re-execute, re-verify every customised system entity. Snapshot ES at the same instant so the stores cannot diverge. |
| `R-14` | **Skipping a minor version bypasses its migrations, and ZDU is off by default** (`C-136`, `C-153`). | MED | Upgrade one minor at a time, letting system-update complete before the next hop, pairing the Helm chart each release's Requirements block names. Assume ZDU is OFF and schedule a real maintenance window. Keep `ELASTICSEARCH_INDEX_BUILDER_MAPPINGS_REINDEX=false` in production. |
| `R-15` | **A patch release is not a safe drop-in.** `v1.6.0.1` claims "no schema or model changes" yet adds privilege requirements to exactly the write paths curation automation uses (`C-137`). | MED | Treat every 4th-position bump as a breaking change: read its full ledger section, re-run curation automation against staging, pre-grant new privileges. Track the release **branch** (`git log <tag>..origin/releases/v<minor>`) since five v1.x patch tags have no ledger section at all. |
| `R-16` | **MySQL only: an upgrade can lock the aspect table for hours** (`C-154`). No env var skips the check. | MED | Check `information_schema.columns` for `utf8mb4_bin` on `metadata_aspect_v2.urn`/`aspect` before upgrading; convert ahead of time (e.g. `pt-online-schema-change`) or size the window to the table. PostgreSQL unaffected. |
| `R-17` | **No documented version downgrade** (`C-155`). A failed blocking upgrade has no rollback once bootstrap UPSERTs have landed or an alias swap has deleted the old backing index. | MED | The **backup pair** (SQL dump + simultaneous ES snapshot) **is** the rollback plan; size the window to include a restore. Rehearse the exact upgrade against a restored copy of production first. |
| `R-18` | **The CLI/server compatibility check ignores the 4th version component and only warns** (`C-135`); the ledger's own `v1.6.0.1` recommendation would trigger a spurious incompatibility warning. | MED | Pin the CLI to the version the Requirements block names and upgrade CLI and server as one change. Do not rely on the warning. |
| `R-19` | **No documented support window or backport policy exists** (`C-131`). Observed practice is two concurrent minor lines, but that is an inference from branch activity, not a commitment. | MED | Plan to be on the current or immediately-previous minor at all times; budget an upgrade every ~2–2.5 months. Do not assume an older line will receive a security backport. Watch the release branch directly, not the tag list. |
| `R-20` | **Config drift after upgrade is silent** (`C-156`). A removed env var left set is never looked up and never warned about. | MED | Diff your env-var set against `docs/deploy/environment-vars.md` at the target version as an explicit upgrade step. Verify effective **behaviour**, not that configuration "took". |
| `R-21` | **Out-of-order registration leaves a permanently stale `matchType=UNRESOLVED`** on any lineage edge written before its upstream exists; the verdict never re-evaluates (`C-70`). | MED | Register both endpoint datasets (`S-8`, `S-9`) **before** asserting the edge (`S-11`). If order cannot be guaranteed, re-emit the edge after both exist. |
| `R-22` | **The CLI YAML path has two silent-loss modes**: a YAML with no `properties:` wipes all customProperties; a partially-typed schema block raises `ValueError`, an untyped one silently emits no schema (`C-83`, `C-84`). | MED | Prefer the SDK/OpenAPI PATCH path over `datahub dataset upsert` for incremental curation. If YAML is used, always include a full `properties:` block and either fully type every column or none. |
| `R-23` | **Concurrent UI curation is an unguarded lost-update race** (`C-06`), and column-level term/description edits can silently land on a sibling (`C-124`). | MED | Route bulk/automated curation through PATCH rather than the UI. Where the UI is used, avoid two people editing the same dataset's terms simultaneously. Combined with `R-06`, another reason to avoid siblings entirely. |
| `R-24` | **Ontology round-trip is one-way.** No glossary export exists in-repo (`C-94`); the importer is ALPHA (`C-92`) and derives a lossy URN from the IRI (`C-93`). | MED | Treat DataHub as a **downstream projection** of the ontology tool, not a peer. Keep the ontology tool as system of record; re-import on change. Preserve the exact IRI in `sourceRef`/`sourceUrl` **and** in a structured property so an external reconciler can match on it without parsing the URN. |
| `R-25` | **Stale search/graph documents are never purged automatically** (`C-144`). | LOW | Schedule a periodic `RestoreIndices -a clean` during a maintenance window (accepting that it wipes and replays, so consumers must be healthy), or reindex into a fresh instance. |
| `R-26` | **Aspect version history is trimmed to 20 by default** (`C-07`). The current value is never at risk; the audit trail is. | LOW | If provenance of curation is itself a requirement, do not rely on aspect versions. Raise `maxVersions` via a retention plugin YAML for `glossaryTerms`, `editableSchemaMetadata`, `ownership`, `structuredProperties`, `globalTags`, `domains`, or consume the MCL stream into an external append-only store. |
| `R-27` | **`patchEntity` is released but untrodden** — undocumented and unused by DataHub's own UI (`C-20`), with `validateAndTransformPatchOperations` an explicit no-op (`C-21`). | LOW | Prefer the Python SDK patch builders, whose routing is exercised by connectors. If `patchEntity` is used, always pass `forceGenericPatch: true` and integration-test each aspect path. |

---

## §7 Parity scorecard — DataHub vs OpenLineage

| Req | DataHub mechanism | DataHub grade | OpenLineage grade | Which is better, and why |
|---|---|---|---|---|
| `RQ-1` | MCP with no run concept; key aspect alone is a valid registration; **PATCH released on every non-timeseries aspect**; `editable*` aspects protect curation from ingestion; aspect versioning gives real edit history; Forms model a verification workflow | **`NATIVE`** | `NATIVE` | **DataHub, decisively.** Both allow run-less registration, but OpenLineage has *only* whole-facet replace, no partial-update primitive, no edit history and no notion of a draft. DataHub adds PATCH with keyed-array merge (`C-11`, `C-98`), 20 versions of history (`C-07`), attribution-keyed slots so independent curators don't collide (`C-15`), and Forms (`C-26`). The same governing constraint applies to both — but only DataHub ships an escape hatch. |
| `RQ-2` | `file`-platform dataset + `schemaMetadata(OtherSchema)`; **`datasetProfile` attaches via a plain MCP with no run at all**; `deprecation.replacement` for supersession | **`NATIVE-CONSTRAINED`** | `NATIVE-CONSTRAINED` | **Split.** DataHub wins the question that mattered: profiling needs **no synthetic job** (`C-71`), removing OpenLineage's defining constraint. But DataHub loses two others: **file format is not modelled at all** (`C-80` — OpenLineage has `datasetType: FILE` and `storage.fileFormat` as first-class), and timeseries aspects are UPSERT-only and **have no relational copy** (`C-140`). Same grade, different and arguably more operationally dangerous constraint. |
| `RQ-3` | `(platform, name, fabric)` URN tuple; **`container` is a real entity reached by an `IsPartOf` graph edge**; both datasets independently addressable at the API/URN layer | **`NATIVE`** | `NATIVE` | **Split, leaning DataHub.** DataHub's hierarchy is **structured graph data, not a string to be parsed** (`C-43`) — a genuine modelling advantage. But OpenLineage is better on the half the requirement names: it documents a naming convention for **40+ platforms**, where DataHub ships only a single `datasetNameDelimiter` character per platform and per-platform format prose in ~6 of 112 connector dirs (`C-46`). DataHub also has an identity trap OpenLineage lacks: `platform_instance` is concatenated into `name`, so adding it changes identity (`C-31`). |
| `RQ-4` | `UpstreamLineage.Upstream{type=COPY}` — **released 2020-05-21**, `isLineage: true`, PATCHable one-edge-at-a-time. `Siblings` is the anti-pattern and is confirmed as one. | **`NATIVE`** (edge) / **`NATIVE-CONSTRAINED`** (the HOW) | `UNRELEASED` (target) / `NATIVE` (workaround); `SymlinksDatasetFacet` = `ANTI-PATTERN` | **DataHub, by a wide margin — this is the largest gap in the comparison.** OpenLineage's semantically exact mechanism was merged but **in no release tag**, forcing a replication-job workaround. DataHub's exact mechanism (`type=COPY`, *"Direct copy without modification"*) has been **released for six years** (`C-49`) and is incrementally patchable (`C-50`). The anti-pattern is structurally identical in both models — DataHub's `Siblings` collapses two identifiers onto one node (`C-60`) **and actively deletes the edge between them** (`C-58`), which is if anything worse than `SymlinksDatasetFacet`. DataHub's constraint: the type does not survive graph traversal (`C-53`). |
| `RQ-5` | URN excludes credentials by API shape at every builder; **plus first-class `dataHubSecret`/`dataHubConnection` entities**, AES-256-GCM, and a released human-caller guard | **`NATIVE`** | `NATIVE` | **Split, leaning DataHub.** DataHub has something OpenLineage structurally cannot: **an actual credential store** with encryption, privilege gating, read-redaction and a `v1.7.0` caller guard (`C-37`–`C-40`) — OpenLineage's known gaps list "no credential store" outright. And `DatasetKey.pdl` explicitly states the identity/name split (`C-30`). **But OpenLineage is better on the guard rail**: it ships `JdbcUrlSanitizer` with tests, whereas DataHub has **no shared sanitiser in either language** (`C-36`). Neither has a normative prohibition. Net: DataHub solves the storage half properly and the hygiene half worse. |
| `RQ-6` | `glossaryTerms` at dataset level + `editableSchemaMetadata`/`schemaField` at column level; `termSource`/`sourceRef`/`sourceUrl` for the external system of record; **structured properties as a typed extension that renders in the UI**; RDF/SKOS importer (ALPHA) | **`NATIVE`** (attach + inbound) + **`NATIVE`** (extension) / **`ABSENT`** (export) | `NATIVE` + `EXTENSION` | **Split.** DataHub's extension mechanism is far stronger: structured properties are typed, validated, searchable, filterable and **render in the consumer UI at both entity and column granularity** (`C-104`) — where OpenLineage's custom facets are explicitly *stored but invisible to consumer UIs*. DataHub also has fewer normative naming constraints (no prefix or schema-URL requirement, `C-107`) and a shipped ontology importer (`C-92`). **But OpenLineage wins the "round-trippable" half**: DataHub has **no glossary export at all** (`C-94`), so the ontology tool must remain system of record. |
| `RQ-7` | `metadata_aspect_v2` opaque-blob storage makes versioned curation structurally upgrade-proof; graph fully rebuildable; **but timeseries has no rebuild path** | **`NATIVE-CONSTRAINED`** | **n/a — no baseline** | **Not comparable.** OpenLineage is a wire format, not a database: it has no upgrade story, no persistence, and therefore nothing to carry over. DataHub's answer is genuinely good for versioned aspects (`C-139`: no DDL, no migration, ever) and genuinely dangerous for timeseries (`C-140`) — a distinction that cannot exist in a wire format. |

**Aggregate.** DataHub is better on `RQ-1`, `RQ-4` (decisively) and the extension half of `RQ-6`; roughly level on `RQ-2` and `RQ-3`; better on the storage half and worse on the hygiene half of `RQ-5`; worse on the export half of `RQ-6`. It answers `RQ-7`, which OpenLineage cannot address at all. It also closes every one of the baseline's listed gaps except two: **DataHub has a query API, a curation UI, edit history, a draft/verification primitive and a credential store; its custom extensions are visible; and consumer support is verifiable in-repo.** The two it does not close: no normative credential prohibition, and no ontology export.

---

## §8 Open questions the repo cannot answer

1. **Does Elasticsearch error or silently succeed when a `_restore` `indices` pattern matches nothing?** This determines whether the `R-02` defect fails loudly or produces a silent empty restore. External-system behaviour; changes the severity but not the fix.
2. **What sets `ZDU_STAGE_20`?** Both zero-downtime flags default to it, but it is set by the Helm chart, which is **not in this repository** (`datahub-kubernetes/` contains only a README). Whether a given deployment gets ZDU or the legacy blocking reindex is undeterminable here.
3. **Is there any supported version-downgrade path?** None is documented for v1.5.0–v1.7.0. Whether an older GMS can read aspects rewritten by the `MigrateAspects` `schemaVersion` sweep is unanswerable.
4. **What is the actual support window and backport policy?** No prose policy exists anywhere. Two concurrent minor lines is an *observed practice*, not a commitment. A company betting a multi-month curation effort on a version line cannot get a guarantee from this repo.
5. **Has `patchEntity`'s generic path ever been exercised in anger?** Released, unflagged, undocumented, unused by DataHub's own frontend, with a no-op validator. Only running it against a live GMS would tell.
6. **Is the GraphQL curation lost-update race (`C-06`) real in practice**, or mitigated by a database-level lock in the Ebean DAO's `forUpdate` read path? The absence of conditional headers is verified; the transaction behaviour for two interleaved UPSERTs was not traced.
7. **Do a custom plugin aspect's `@Searchable` annotations reach Elasticsearch mappings?** (`C-111`.) Likely, since the plugin registry uses the same `EntitySpecBuilder`, but the spec was never traced into an actual mapping.
8. **Is a container's md5 GUID computed identically on the Java/GMS side, or only in the Python library?** Only the Python implementation was verified. A team registering containers through OpenAPI must reproduce the algorithm exactly.
9. **Can a recipe reference a `dataHubConnection` by URN** the way it references a secret by `${NAME}`? `docs/secret-resolution.md` documents only the three secret backends and never mentions the connection entity.
10. **Does `LOGICAL_MODELS_ENABLED=false` disable only UI viewing or also the write path?** The feature guide and `application.yaml` comment read differently.
11. **What does the external `acryldata/business-glossary-sync-action` actually emit**, does it preserve `sourceRef`/`sourceUrl`, and is it maintained? It is the only pointer to an ontology round-trip and is outside this repo.
12. **Does soft-deleting a superseded file dataset also remove its timeseries profile points?** `hard_delete_timeseries_aspect` reports a separate `timeseriesRows` count, implying separate operations, but the soft-delete path was not traced.
13. **Which fields does the `MigrateAspects` `schemaVersion` sweep rewrite, and is it lossless?** It mutates the authoritative store in place, gated by `SYSTEM_UPDATE_MIGRATE_ASPECTS_ENABLED` (default `${ZDU_STAGE_20:false}`).
14. **Does any DataHub Cloud (managed) feature reinstate a distinction between aliasing and derivation that OSS lacks?** Cloud release notes mention siblings repeatedly but describe a codebase not present here.
15. **What happens to already-stored array entries when a Java-SDK patch (key `[tag]`) is applied to an aspect whose entries were written by the Python SDK (key `[source, tag]`)?** (`C-16`.) The repo contains no test covering mixed-client keys.

---

## §9 Verification checklist, ordered by value

A reviewing model should spend its budget top-down. Items 1–5 change the design if wrong; 6–12 change an estimate; 13+ are hygiene.

1. **`C-58` — does `SiblingGraphService` really delete the edge between siblings?** Read `metadata-io/src/main/java/com/linkedin/metadata/graph/SiblingGraphService.java:161-186`. If this is wrong, `D-04`'s rejection of Siblings collapses and `R-06` disappears. *This is the single highest-value check in the document.*
2. **`C-140` — do timeseries aspects really bypass the relational store?** Read `EntityServiceImpl.ingestTimeseriesProposal` (javadoc at ~L2269) and confirm only a synthesised key aspect is persisted. If wrong, `R-01`, `R-02` and `D-12` all relax. Cross-check `docs/modeling/metadata-model.md`'s Timeseries section, which states it independently.
3. **`C-49` + release status — is `DatasetLineageType.COPY` real and released?** `cat metadata-models/src/main/pegasus/com/linkedin/dataset/DatasetLineageType.pdl`, then `git log --diff-filter=A` on it, then `git merge-base --is-ancestor 1283dd3ff4 v1.7.0`. This is the RQ-4 grade and the largest claimed gap against the baseline.
4. **`C-71` — is a profile really attachable with no run?** Read `metadata-ingestion/src/datahub/ingestion/source/data_lake_common/profiling/profiler.py:427-465` and confirm the MCP carries only `(entityUrn, DatasetProfileClass)`. This is the RQ-2 grade.
5. **`C-01` + `C-11` — whole-aspect replace, and PATCH released on all aspects.** Read `ChangeItemImpl.convertToRecordTemplate`, then `git cat-file -p v1.7.0:entity-registry/src/main/java/com/linkedin/metadata/aspect/batch/MCPItem.java` and confirm `supportsPatch` returns unconditional `true` **in the tagged blob**, not at HEAD. The whole of `D-01` rests on this.
6. **§0 — reproduce the maturity-method failure.** Run `git tag --contains 7d5cedd5f8` (expect empty) and `git merge-base --is-ancestor v1.7.0 HEAD` (expect false). If these do not reproduce, every release-status claim needs re-derivation.
7. **`C-95` — read `EditableSchemaMetadata.pdl`'s doc comment verbatim.** The curation-protection argument in `D-03` rests entirely on it.
8. **`C-53` — confirm `type` is not among the six projected edge attributes** in `Upstream.pdl`'s `@Relationship`, and that `LineageRelationship.pdl` has no `DatasetLineageType` field. This drives `R-07` and the `RQ-4` sub-grade.
9. **`C-148` — confirm the encryption-key fallback literal.** `grep encryptionKey metadata-service/configuration/src/main/resources/application.yaml`. Expect `?: 'ENCRYPTION_KEY'`. Drives `R-03`.
10. **`R-02` — confirm the backup doc defect.** `sed -n '196,200p' docs/how/backup-datahub.md` and `grep -rn datastream docs/`. Expect exactly one hit.
11. **`C-22` — confirm OpenAPI v3 single-aspect defaults to CREATE.** `GenericEntitiesController.java:653-659`. Drives `D-02` and the most likely first-day failure.
12. **`C-36` — reproduce the negative sanitiser search** in Java: `grep -rn "removePassword\|stripCredential\|maskPassword\|scrubUrl\|sanitizeUrl\|sanitizeJdbc" --include=*.java .` Expect zero hits. This inverts the prior run's finding and drives `R-09`'s "net-new, not a port".
13. `C-12` — count the `aspectSpecTemplateMap.put(...)` calls in `SnapshotEntityRegistry.populateTemplateEngine`. Expect 23, and confirm no `SCHEMA_METADATA_ASPECT_NAME` entry.
14. `C-102` vs `C-103` — confirm `entityTypes` is unenforced while `allowedPlatforms` **is**, in `StructuredPropertiesValidator`.
15. `C-157` — spot-check three flags in `application.yaml` against `FeatureFlags.java` and confirm the yaml wins.
16. `C-46` — `grep -c "datasetNameDelimiter" bootstrap_mcps/data-platforms.yaml` (expect 120) and confirm no per-platform *format* table exists.
17. `C-129` — confirm `v1.5.0`, `v1.6.0` and `v1.6.0.1` GA tags sit beyond their final rc.
18. `C-94` — confirm no `glossary` command group in `metadata-ingestion/src/datahub/entrypoints.py`.

---

## §10 Documentation defects found

Grouped by severity of consequence. All at HEAD `7d5cedd5f8`.

### Would cause data loss or failed recovery

| # | Location | Defect |
|---|---|---|
| 1 | `docs/how/backup-datahub.md:198` | **The restore command matches zero DataHub indices.** `"indices": "datastream*,metadataindex*"` vs real names `*index_v2`, `<entity>_<aspect>aspect_v1`, `graph_service_v1`, `system_metadata_service_v1`. `"datastream"` occurs **nowhere else in the repo**. The snapshot side is correct, so the defect is invisible until recovery — of the one asset class with no rebuild path. |
| 2 | `docs/cli-commands/dataset.md` (whole file) | Documents `datahub dataset upsert` across 314 lines **without ever stating merge-vs-replace semantics**, while the implementation is mixed: `datasetProperties` fields are PATCHed; `schemaMetadata`, `globalTags`, `glossaryTerms`, `ownership` and per-column `structuredProperties` are full UPSERTs that replace. A YAML naming one owner replaces all owners. |
| 3 | `docs/cli-commands/dataset.md` (schema block) | Does not state the **all-or-nothing type rule**: a YAML where some fields have `type` and others do not raises `ValueError`; one where none do **silently emits no `schemaMetadata` aspect at all**. |
| 4 | `datahub-upgrade/.../restorebackup/ClearAspectV2TableStep.java:12` | Javadoc calls the step *"Optional"*. It has no flag, no condition and no `skip()` override — **it always runs**, deleting the entire authoritative aspect table. |

### Would produce a wrong design decision

| # | Location | Defect |
|---|---|---|
| 5 | `docs/advanced/partial-update.md`, `derived-aspects.md`, `pdl-best-practices.md`, `entity-hierarchy.md`, `backfilling.md` | **Five WIP stubs** containing only a heading and the word `WIP`. `partial-update.md` is the natural home of the single most decision-relevant semantic in this evaluation. Commented out of `sidebars.js`, so unpublished — but present and misleading in-repo. |
| 6 | `docs/advanced/patch.md:18` | **Dead reference.** Points to a `SUPPORTED_TEMPLATES` constant at `AspectTemplateEngine.java#L23` as the authoritative list of patchable aspects. `grep -rn "SUPPORTED_TEMPLATES" --include=*.java` returns **zero hits**; L23 is an unrelated HashMap init. The real list is hardcoded in `SnapshotEntityRegistry`. |
| 7 | `docs/advanced/patch.md` | **Stale core claim:** *"Traditional PATCH support is only available for a selected set of aspects."* Since #14823 (released in v1.4.0) `supportsPatch` returns unconditional `true` via the generic path. Also describes a `GenericTemplate` class that does not exist. |
| 8 | `docs/advanced/patch.md:213-215` | **Op-vocabulary contradiction.** Claims only `ADD` and `REMOVE` are supported. `PatchOperationType` declares six (`ADD, REMOVE, REPLACE, MOVE, COPY, TEST`) and `TemplateUtil.validatePatch` admits all six; the Python SDK restricts to three. Three layers, three answers. |
| 9 | `metadata-models/docs/entities/schemaField.md:5` | States schemaField entities are *"automatically created by DataHub when datasets with schemas are ingested"*. **False at default config**: `MCP_SIDE_EFFECTS_SCHEMA_FIELD_ENABLED` defaults `false`. |
| 10 | `docs/platform-instances.md` | **Self-contradiction within 12 lines.** "Naming Platform Instances" presents `finance_redshift` as an allowed name; "Best Practices" immediately below lists `finance_redshift` under **❌ Avoid These Patterns**. |
| 11 | `metadata-models/.../glossary/GlossaryTermInfo.pdl` | The `termSource` doc-comment promises *"with default value as INTERNAL"*, but the PDL declares a required `string` **with no default**. The default is implemented independently in at least two clients. A direct API writer that omits it fails schema validation. |
| 12 | `metadata-models/.../events/metadata/ChangeType.pdl` | `UPDATE` is annotated *"NOT SUPPORTED YET"*, yet `MCPItem.CHANGE_TYPES` includes it and no validator enforces the "otherwise fail" half (`C-25`). |
| 13 | `docs/what/graph.md` | Asserts *"All the entities and relationships are stored in a graph database, Neo4j"*. The shipped default is Elasticsearch (`application.yaml:406`, `GRAPH_SERVICE_IMPL:elasticsearch`). Still linked from four sibling docs. |

### Stale, contradictory or misleading

| # | Location | Defect |
|---|---|---|
| 14 | `docs/api/tutorials/datasets.md` | The canonical "how do I make a dataset" tutorial covers **only Create and Delete**. Zero occurrences of `schemaMetadata`, `hash`, `platformSchema`, `Schemaless` or `OtherSchema` — exactly what a team standing up a CSV stand-in needs. |
| 15 | `docs/`, `datahub-web-react/src/` | **`patchEntity`/`patchEntities` — the only generic aspect-agnostic incremental-write API in GraphQL, released in both `v1.7.0` and `v1.6.0.1` — is documented nowhere and used nowhere in DataHub's own UI.** `docs/advanced/patch.md` and the OpenAPI guide both describe generic patching as OpenAPI-only. |
| 16 | `docs/advanced/patch.md` | No doc anywhere states that **GraphQL curation mutations perform a server-side read-modify-write then a whole-aspect UPSERT** (`C-05` regime 4). `patch.md` tells readers a read-modify-write is needed but never says the UI already does one, nor that it is unguarded. |
| 17 | `metadata-ingestion/.../rdf/README.md` vs `docs/sources/rdf/rdf_post.md` vs `urn_generator.py` | **Three mutually inconsistent IRI→URN mappings.** README shows `finance/credit-risk` (host dropped, slashes); `rdf_post.md` shows `example.com/finance/credit-risk` (host kept, slashes); the code produces `example.com.finance.credit-risk` (dots). The RDF `SPEC.md` gives a fourth, tuple-style form the code never produces. |
| 18 | `docs/advanced/mcp-mcl.md` | The embedded `MetadataChangeProposal` PDL listing has drifted from the real schema (`aspectName` note, `systemMetadata` description). It also documents six ChangeType values, omitting `RESTATE`. |
| 19 | `docs/api/openapi/openapi-usage-guide.md:55-330` | Every primary worked example targets `/openapi/entities/v1/`, whose controller is annotated `@Deprecated` with the comment `/* Use v2 or v3 controllers instead */`. The doc carries no deprecation notice. |
| 20 | `docs/api/datahub-apis.md:60` | Self-declared stale: *"Last Updated : Feb 16 2024"* heads a 60-row API capability matrix that is still the canonical API-choice document. |
| 21 | `docs/modeling/extending-the-metadata-model.md:230-237` | **Malformed YAML** that would not parse: a leading dash on `keyAspect` makes it a second list element and `aspects:` is mis-indented. |
| 22 | `docs/modeling/extending-the-metadata-model.md` | The `autoRender`/`renderSpec` entries claim support is limited to *"Charts, Dashboards, DataFlows, DataJobs, Datasets, Domains, and GlossaryTerms"*. Stale — the React app now queries `autoRenderAspects` far more broadly. |
| 23 | `docs/api/tutorials/lineage.md` | **`DatasetLineageType` is documented nowhere.** `grep -rn "DatasetLineageType" docs/` returns zero hits; `COPY`/`TRANSFORMED`/`VIEW` are never mentioned, while the SDK silently hardcodes values. |
| 24 | `docs/cli-commands/dataset.md` (YAML format) | Documents a `downstreams:` key but **no `upstreams:` and no `siblings:`** — the two mechanisms most relevant to registering a replica alongside its origin cannot be expressed declaratively. |
| 25 | `metadata-models/docs/entities/dataset.md` + all of `docs/` | **No hand-authored doc explains the `Siblings` aspect, its merge semantics, or when to choose Siblings vs UpstreamLineage.** The distinction survives only incidentally in `trino_pre.md` and one CLI reference line — the single most consequential modelling choice for RQ-3/RQ-4 is essentially undocumented. |
| 26 | `metadata-models/.../dataprocess/DataProcessInfo.pdl` | **Model defect:** both `inputs` and `outputs` declare `@Relationship` with the **same name `"Consumes"`**. `outputs` should be `"Produces"` with `isUpstream: false`, as `DataJobInputOutput.pdl` correctly does. As written, a DataProcess's outputs are indexed as incoming edges, inverting direction. |
| 27 | `metadata-models/.../dataset/UpstreamLineage.pdl:15-22` | The `@Relationship` on `fineGrainedLineages` omits `"isLineage": true`, unlike the identically-named annotation on `Upstream.pdl`. `LineageRegistry` therefore discards it. |
| 28 | `metadata-jobs/.../SiblingAssociationHook.java:275` | Misleading method name retained after a behaviour change: `setSiblingsAndSoftDeleteSibling` performs **no soft delete** at HEAD. An auditor checking whether siblings destroy independent addressability would be misled by the name alone. |
| 29 | `docs/advanced/db-retention.md:2` | Frontmatter claims it covers *"historical metadata aspects **and timeseries data**"*; the body never mentions timeseries, and the policies provably apply only to `EbeanAspectV2`. |
| 30 | `docs/cli.md:1459` | Claims server releases happen *"approximately every week"*. The tag record shows minor releases roughly every **2–2.5 months**. |
| 31 | `AGENTS.md:233` | Stale: describes DataHub Cloud release notes as following `v_0_3_<N>.md`. The directory now runs to `v_2_1_0.md`. |
| 32 | `metadata-service/.../data-platforms.yaml` | The ABS source emits `urn:li:dataPlatform:abs` but there is **no `abs` entry** in the canonical platform list — ABS datasets reference a platform with no `displayName`, logo or delimiter. |
| 33 | `metadata-service/.../data-platforms.yaml:933-940` | `urn:li:dataPlatform:file` is registered with `datasetNameDelimiter: "."` while every other file/object-store platform (s3, gcs, hdfs, adlsGen1/2, excel) uses `"/"` — and file dataset names are built from slash-separated paths. |
| 34 | `metadata-ingestion/examples/library/`, `docs/api/tutorials/` | **No example or tutorial anywhere shows how to emit a `DatasetProfile`.** ~40 `dataset_*` recipes exist (schema, tags, terms, lineage, structured properties, deprecation) and 29 tutorial pages; none covers profiles. The only worked example is a demo seed fixture. |
| 35 | `metadata-models/.../structured/StructuredPropertyKey.pdl` | Typo in the file's only doc comment: *"The id for a structured **proeprty**."* |
| 36 | `metadata-models/.../businessattribute/BusinessAttributeKey.pdl` | Copy-paste error: record doc comment reads *"Key for a Query"*. |
| 37 | `metadata-models-custom/README.md` | The `/config` endpoint sample output is shown in two mutually inconsistent shapes (nested under `"models"` vs at the JSON root). Only one can be right. |
| 38 | `docs/what/entity.md` | Stub reading only *"This page has been moved"*, yet still linked as a live reference from `aspect.md`, `gma.md`, `gms.md` and `graph.md`. |
| 39 | `docs/what/delta.md` | Documents a Rest.li "metadata delta" partial-update mechanism with a worked example. Neither referenced PDL exists, and the union they were added to is now empty. |
| 40 | `docs/what/urn.md` | Says parentheses are *"not allowed anywhere in the URN"*, but the restriction applies to field **values** — the doc's own valid example on the same page contains parentheses as tuple delimiters. |
| 41 | `docs/what-is-datahub/datahub-concepts.md` | A `<details>` block titled "List of Data Platforms" enumerates ~26 platforms as authoritative; the real bootstrap list has **120**. |

---

## §11 The release-branch and upgrade process (RQ-7 operational reference)

### 11.1 How DataHub actually ships

```
master ──●──●──●──●──●──●──●──●──●──●──●──●──●──●──▶  (HEAD 7d5cedd5f8, in NO tag)
          \                    \
           \ (cut at v1.6.0rc1) \ (cut at v1.7.0rc1 = b88750fe58)
            \                    \
             releases/v1.6.0      releases/v1.7.0
             87 ahead/1223 behind  7 ahead/345 behind
             ├─ v1.6.0   (2026-05-21)   ├─ v1.7.0rc1 (2026-08-03)
             ├─ v1.6.0.1 (2026-08-13) ◀─┤ v1.7.0rc2 = v1.7.0 (2026-08-04)
             │   ↑ NEWER DATE than v1.7.0
             └─ 2 unreleased commits    └─ 5 unreleased commits
                (incl. security fixes)     (incl. security fixes)
```

Facts a self-hoster must internalise:

- **Branches are cut at `rc1` and never merged back** (`C-125`, `C-126`). Being on a release line means running code up to 1,863 commits behind master.
- **Every fix is a separate backport PR per line** (`C-127`), and roughly **40% are hand-adapted rather than clean cherry-picks** — including a revert of a fix that could not be adapted (`C-128`). Backports are not mechanically equivalent to master.
- **Version order is not chronological** (`C-129`). `v1.6.0.1` postdates `v1.7.0` by nine days. `git tag --contains` under-reports.
- **`git tag --contains` cannot map a version to a branch.** At least three branch prefixes are in use (`releases/`, `hotfixes/`, legacy `hotfix/`) with no rule derivable from the version number (`C-130`).
- **Images publish only on a GitHub Release** (`C-132`). Fixes merged to a release branch are **not installable** until the next tag — right now that includes security backports on both live lines (`C-133`).
- **There is no documented support window, EOL or backport policy anywhere** (`C-131`). Observed practice: two concurrent minor lines, the older going quiet at the next GA.

### 11.2 The upgrade sequence

| Step | What runs | Blocking? | What can go wrong |
|---|---|---|---|
| 1 | **Pre-flight**: SQL dump **+ simultaneous ES snapshot**; record `SECRET_SERVICE_ENCRYPTION_KEY`; archive custom-plugin artefacts + registry version; capture the env-var set | — | Missing the ES snapshot loses all profiles (`R-01`); missing the key loses all secrets (`R-03`) |
| 2 | **MySQL collation check** (`SqlSetup`) — `ALTER TABLE … utf8mb4_bin` if needed | Blocking | Locks the whole aspect table for minutes-to-hours; **no env var skips it** (`R-16`) |
| 3 | **`SystemUpdateBlocking`** — bootstrap MCPs, default policies, index build/reindex | Blocking (GMS will not start until done) | Re-UPSERTs 11 of 16 default policies (`R-11`); bumped bootstrap templates clobber customised system entities (`R-12`) |
| 4 | **Index build** — mappings/settings reindex, alias swap | Blocking | ZDU is **off** in every released version, so expect blocked writes, an alias swap that deletes the old backing index, and consumer scale-down (`R-14`) |
| 5 | **`SystemUpdateNonBlocking`** — aspect backfills via `AbstractMCLStep`/`AbstractMCPStep`, checkpointed by `dataHubUpgradeResult` with a `lastUrn` cursor | Background | Resumable and idempotent; but the checkpoints live in the **same table as the curation** (`R-13`) |
| 6 | **Post-upgrade verification** | — | See 11.4 |

**One minor at a time.** `v1.7.0` explicitly requires passing through `v1.6.0` first, with a specific Helm chart at each hop (`C-136`). Pinned Helm versions are non-monotonic — `v1.6.0.1` requires chart 1.1.2 while the newer `v1.7.0` requires 1.1.0.

### 11.3 What carries over

| Asset | Store | Authoritative? | Rebuildable? | Verdict |
|---|---|---|---|---|
| `schemaMetadata`, `editableSchemaMetadata` | `metadata_aspect_v2` | **Yes** | n/a | **SURVIVES** |
| `glossaryTerms`, `globalTags` (dataset **and** `schemaField`) | `metadata_aspect_v2` | **Yes** | n/a | **SURVIVES** |
| `structuredProperties` — definitions **and** values | `metadata_aspect_v2` | **Yes** | n/a | **SURVIVES** |
| `upstreamLineage` (the `COPY` provenance edges), `siblings` | `metadata_aspect_v2` | **Yes** | n/a | **SURVIVES** |
| `ownership`, `domains`, `institutionalMemory`, `editableDatasetProperties`, `deprecation` | `metadata_aspect_v2` | **Yes** | n/a | **SURVIVES** |
| `dataHubSecretValue` ciphertext | `metadata_aspect_v2` | **Yes** | n/a | **SURVIVES — but useless without the key** (`R-03`) |
| Aspect version history | `metadata_aspect_v2` | Yes | n/a | **SURVIVES, capped at 20** (`R-26`) |
| Search index (`*index_v2`) | Elasticsearch | Derived | **Yes**, from SQL | **REBUILDABLE** (asynchronously — `R-10`) |
| Graph / lineage index (`graph_service_v1`) | Elasticsearch | Derived | **Yes**, from SQL | **REBUILDABLE.** Proven: zero `@Relationship` in all 11 timeseries PDLs (`C-142`) |
| **`datasetProfile` and 10 other timeseries aspects** | **Elasticsearch only** | **YES — ES is authoritative** | **NO** | **AT RISK.** No SQL copy, no rebuild path. Sole backup is an ES snapshot; sole replay is a 90-day Kafka topic (`R-01`) |
| Custom-aspect data | `metadata_aspect_v2` | Yes | Only if the plugin loads | **MANUAL-MIGRATION.** Rows survive; a failed plugin makes them invisible (`R-08`) |
| Customised default policies | `metadata_aspect_v2` | Yes | — | **AT RISK — reverted on every deploy** (`R-11`) |
| Customised system entities (platforms, ownership types, root user) | `metadata_aspect_v2` | Yes | — | **AT RISK on template version bump** (`R-12`) |
| Env-var configuration | Deployment | — | — | **MANUAL-MIGRATION — drift is silent** (`R-20`) |

### 11.4 Minimum backup set

All six, or the backup is worthless. Items 2, 3 and 6 are each **independently sufficient** to make an otherwise-complete backup useless.

1. A dump of **`metadata_aspect_v2`**.
2. An **Elasticsearch/OpenSearch snapshot taken at the same instant** — the only copy of all 11 timeseries aspect types.
3. The value of **`SECRET_SERVICE_ENCRYPTION_KEY`** (note it silently defaults to the literal `ENCRYPTION_KEY`).
4. **Custom model-plugin artefacts and their registry version** — the SQL rows are useless without a loadable registry.
5. The deployment's **full env-var set**, for post-upgrade diffing.
6. A **restore-time index pattern that actually matches** — **not** the doc's `datastream*,metadataindex*`. Use `*index_v2,*aspect_v1,graph_service_v1,system_metadata_service_v1`.

### 11.5 Post-upgrade gates

1. `curl -s http://<gms>/config | jq .models` → assert `loadResult == SUCCESS`, `failureCount == 0` (catches `R-08`).
2. Scale MCL/MAE consumers **up**, assert consumer-group lag returns to zero (catches `R-10`).
3. `SELECT COUNT(*) FROM metadata_aspect_v2 WHERE version=0` vs `GET _cat/indices/*index_v2?v` (catches `R-10`).
4. `GET _cat/indices/dataset_datasetprofileaspect_v1` → assert non-zero doc count (catches `R-01`).
5. Grep the upgrade log for the `ignored` tally and `Validation failed for N items, continuing`.
6. Read back the policy set and diff against expectation (catches `R-11`).
7. Diff the env-var set against `docs/deploy/environment-vars.md` at the target version (catches `R-20`).
8. Re-run curation automation against staging **before** production — patch releases add privilege requirements (`R-15`).
