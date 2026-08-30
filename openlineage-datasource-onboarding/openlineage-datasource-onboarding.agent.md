# OpenLineage Model Analysis — Manually-Curated Datasource Onboarding

**Document type:** agent-readable analysis. Structured for machine review by another model.
**Generated:** 2026-08-29
**Primary source:** OpenLineage repository at `HEAD` = `b995ee00c`, commit date 2026-08-28, latest release tag `1.52.0`.
**Authority rule:** Claims sourced from this repo at HEAD supersede any web documentation, which lags `main`.

---

## 0. How to read this document

Every claim has a stable ID (`C-nn`), a confidence level, and a source. Design decisions have IDs (`D-nn`) and
reference the claims they rest on. Open risks have IDs (`R-nn`).

| Confidence | Meaning |
|---|---|
| `VERIFIED` | Read directly from a schema, source file, or doc in this repo at HEAD. Path cited. |
| `INFERRED` | Follows logically from a VERIFIED claim; reasoning stated. Not written down anywhere. |
| `PENDING` | Requires evidence not obtainable from this repo. Marked for external verification. |

**Reviewer instruction:** the highest-value checks are (a) `R-01` merge semantics, (b) `D-04` replica modeling,
(c) whether `C-21`–`C-24` maturity facts remain true at your read time. Everything else is mechanical.

---

## 1. Requirement decomposition

The stated use case decomposes into six independent requirements. They are separable and can ship in any order
except that `RQ-1` and `RQ-2` are prerequisites for the rest.

| ID | Requirement (as stated) | Restated as a modeling problem |
|---|---|---|
| `RQ-1` | "I don't have all of the connections, details, data profiles etc. I need to manually gather and add as I find it." | Incremental, out-of-order, partial metadata accrual against a stable entity identity. No job run exists to hang metadata on. |
| `RQ-2` | "at first we download sample reports or database exports, usually saved as a csv sometimes a json file" | A file artifact is a first-class dataset with its own identity, and is a *stand-in* for a datasource not yet reachable. Later superseded, not deleted. |
| `RQ-3` | "some of the source data providers don't want us to read the production database, so we get data from a replica but it could be a different platform" | Two distinct physical datasets on two distinct platforms represent one logical entity. Both must be addressable. |
| `RQ-4` | "I need document the original source of the data and the data's provenance (that it's from a replica)" | A directed, queryable provenance edge from replica → origin, plus an assertion of *how* (replication) and *from where*. Must survive lineage traversal. |
| `RQ-5` | "connection details (login/password/url) need to be stored and maintained separately from what we show the users as the data source (server/database/table/etc)" | Split identity from access. The published identifier must be derivable without secrets, and stable across credential rotation. |
| `RQ-6` | "integrated with our in-house ontology tool, so I'll need to classify it by ontology" | Attach external taxonomy terms at dataset and column granularity, round-trippable to an external system of record. |

---

## 2. Core model claims

### 2.1 Event types

| ID | Claim | Confidence | Source |
|---|---|---|---|
| `C-01` | The spec root is `oneOf: [RunEvent, DatasetEvent, JobEvent]`. Three coequal event types, not one. | VERIFIED | `spec/OpenLineage.json:303` |
| `C-02` | `DatasetEvent` requires `dataset` and carries `"not": {"required": ["job","run"]}` — it is structurally forbidden from referencing a job or run. | VERIFIED | `spec/OpenLineage.json:65-78` |
| `C-03` | `JobEvent` requires `job`, permits `inputs`/`outputs`, and carries `"not": {"required": ["run"]}`. | VERIFIED | `spec/OpenLineage.json:80-102` |
| `C-04` | `DatasetEvent` and `JobEvent` are officially named "static lineage" / "design lineage" and are documented as "**not** associated with a `Run`" representing "**design-time metadata**". | VERIFIED | `website/docs/spec/object-model.md` |
| `C-05` | `DatasetEvent` is documented for exactly this purpose: "attach metadata to a dataset outside the context of a job or a job run… static schema extraction, documentation generation, or governance… emitted when a dataset's metadata is updated or first defined." | VERIFIED | `website/docs/spec/object-model.md` |
| `C-06` | All three event types share `BaseEvent`, requiring only `eventTime`, `producer`, `schemaURL`. | VERIFIED | `spec/OpenLineage.json:5-27` |
| `C-07` | `RunEvent` subsequent updates are explicitly additive: "input Datasets… can be specified along with `START`, along with `COMPLETE`, or both. This accommodates situations where information is only available at certain times." | VERIFIED | `website/docs/spec/object-model.md` |

**Consequence for `RQ-1`:** `DatasetEvent` is the designed mechanism. Registration does not require inventing a
fake job, and does not require the metadata to be complete.

### 2.2 Identity

| ID | Claim | Confidence | Source |
|---|---|---|---|
| `C-08` | Dataset identity is exactly the pair (`namespace`, `name`). Both required. `facets` is optional. | VERIFIED | `spec/OpenLineage.json`, `$defs/Dataset` |
| `C-09` | "The combined namespace and name for a Dataset should be enough to uniquely identify it within a data ecosystem." | VERIFIED | `website/docs/spec/object-model.md` |
| `C-10` | The naming convention defines namespace as `{scheme}://{host}:{port}` (or an ARN, or a bare token such as `bigquery`). **No convention includes a username, password, or URI userinfo component.** | VERIFIED | `website/docs/spec/naming.md` — full table, 40+ platforms |
| `C-11` | Oracle: namespace `oracle://{host}:{port}`, name `{serviceName}.{schema}.{table}` or `{sid}.{schema}.{table}`. | VERIFIED | `website/docs/spec/naming.md` |
| `C-12` | Snowflake: namespace `snowflake://{organization name}-{account name}` (preferred) or `snowflake://{account-locator}(.{compliance})(.{cloud_region_id})(.{cloud})` (legacy), name `{database}.{schema}.{table}`. | VERIFIED | `website/docs/spec/naming.md` |
| `C-13` | The spec warns the two Snowflake namespace formats produce non-matching dataset IDs: "If you switch formats later, existing lineage nodes won't connect to new ones." | VERIFIED | `website/docs/spec/naming.md` |
| `C-14` | Local file: namespace `file`, name `{path}`. Remote file: namespace `file://{host}`. S3: namespace `s3://{bucket name}`, name `{object key}`. | VERIFIED | `website/docs/spec/naming.md` |
| `C-15` | `spec/Naming.md` is marked obsolete and redirects to the website doc. Do not use it. | VERIFIED | `spec/Naming.md` |
| `C-16` | The Python client ships executable namespace builders per platform implementing the `DatasetNaming` protocol (`get_namespace()`, `get_name()`), covering Athena, AWSGlue, AzureCosmosDB, AzureDataExplorer, and others. | VERIFIED | `client/python/src/openlineage/client/naming/dataset.py` (512 lines) |

**Consequence for `RQ-5`:** credential separation is not something to bolt on — the identity grammar is
credential-free by construction. `C-10` is the load-bearing claim.

### 2.3 Facet mechanics

| ID | Claim | Confidence | Source |
|---|---|---|---|
| `C-17` | `BaseFacet` requires `_producer` and `_schemaURL`, and sets `additionalProperties: true`. This is the extension point. | VERIFIED | `spec/OpenLineage.json`, `$defs/BaseFacet` |
| `C-18` | Any facet key not in the standard set is accepted and stored verbatim, provided it carries `_producer` and `_schemaURL`: "as long as they follow the BaseFacet… it will be accepted and stored as part of the OpenLineage event, and will be able to be retrieved when you query those events." | VERIFIED | `website/docs/spec/facets/custom-facets.md` |
| `C-19` | A custom facet may point `_schemaURL` at a self-hosted schema by overriding `_get_schema()`; otherwise it defaults to the `BaseFacet` URL, which is stated to be legal. | VERIFIED | `website/docs/spec/facets/custom-facets.md` |
| `C-20` | `DatasetFacet` and `JobFacet` — **and only those two** — accept `_deleted: true`. The release note is explicit: "adds a `{ _deleted: true }` object that can take the place of any job or dataset facet (but not run or input/output facets, which are valid only for a specific run)." | VERIFIED | `spec/OpenLineage.json:169,274`; `website/docs/releases/0_30_1.md:22` |
| `C-21` | Facets are partitioned by attachment point: `facets` (dataset-level, stable), `inputFacets` (per-read), `outputFacets` (per-write), plus job and run facets. | VERIFIED | `website/docs/spec/facets/dataset-facets/dataset-facets.md` |
| `C-22` | The intended split is stability-based: "metadata that is more static from Run to Run is captured in a DatasetFacet… What changes every Run is captured as an InputFacet or an OutputFacet." | VERIFIED | `website/docs/spec/object-model.md` |
| `C-23` | Because `DatasetEvent` cannot carry a run (`C-02`), and `inputFacets`/`outputFacets` are run-scoped (`C-20`, `C-22`), **input/output facets are unreachable from a `DatasetEvent`**. Anything to be recorded statically must be a dataset-level facet. | INFERRED | Composition of `C-02`, `C-20`, `C-22` |
| `C-24` | **Facet update semantics are normatively specified, and they are per-facet-name atomic REPLACEMENT — not deep merge.** Verbatim: "A facet is an atomic piece of metadata identified by its name. This means that emitting a new facet with the same name for the same entity **replaces the previous facet instance for that entity entirely**." | VERIFIED | `spec/OpenLineage.md` §Facets; restated verbatim at `website/docs/spec/facets/facets.md` |
| `C-24a` | **Event-level metadata is additive**, and this is stated separately from `C-24`: "All metadata is additive. For example, if more inputs or outputs are detected as the job is running, we might send additional events specifically for those datasets **without re-emitting previously observed inputs or outputs**." | VERIFIED | `spec/OpenLineage.md` §Lifecycle |
| `C-24b` | Static-lineage facets persist forward in time until replaced: "sending a Dataset event with an ownership facet on a dataset, replaces any previously defined ownership facet and **applies to all versions moving forward** until replaced by a new one." Run/input/output facets by contrast "only apply to the run they are attached to." | VERIFIED | `proposals/1837/static_lineage.md` §Semantics of facets |
| `C-24c` | Deletion is destructive, not a rollback: "the `{ _deleted: true }` facet removes the owner facet so that there is no owner anymore (**we don't get back to the previous one**)." The proposal gives a three-event worked example. | VERIFIED | `proposals/1837/static_lineage.md` §update lifecycle |
| `C-24d` | **Composite consequence — the central producer rule.** You may add *new* facets at any time without resending existing ones (`C-24a`), and unmentioned facets persist untouched (`C-24b`). But changing one field inside an existing facet requires re-emitting **that entire facet object**. There is no partial-update path within a facet. | INFERRED | Composition of `C-24`, `C-24a`, `C-24b` |

---

## 3. Facet inventory relevant to this use case

Versions are the `$id` at HEAD. Pin these; they are the current values, not necessarily what a released client emits.

| Facet | `$id` version | Attach point | Required fields | Serves |
|---|---|---|---|---|
| `schema` — SchemaDatasetFacet | `1-2-0` | dataset | — (`fields[].name` required) | `RQ-1`, `RQ-2` |
| `dataSource` — DatasourceDatasetFacet | `1-0-1` | dataset | — | `RQ-4`, `RQ-5` |
| `datasetType` — DatasetTypeDatasetFacet | `1-0-1` | dataset | `datasetType` | `RQ-2` |
| `storage` — StorageDatasetFacet | `1-0-1` | dataset | `storageLayer` | `RQ-2` |
| `hierarchy` — HierarchyDatasetFacet | `1-0-0` | dataset | `hierarchy` | `RQ-5` |
| `symlinks` — SymlinksDatasetFacet | `1-0-1` | dataset | `identifiers[].{namespace,name,type}` | rejected for `RQ-4`, see `D-04` |
| `lineage` — LineageDatasetFacet | `1-0-0` | dataset | — | `RQ-4` (target model) |
| `lineage` — LineageJobFacet | `1-0-0` | job | `entries` | `RQ-4` (alternate) |
| `tags` — TagsDatasetFacet | `1-0-0` | dataset | `tags[].{key,value}` | `RQ-6` |
| `documentation` — DocumentationDatasetFacet | `1-1-0` | dataset | `description` | `RQ-1` |
| `ownership` — OwnershipDatasetFacet | `1-0-1` | dataset | `owners[].name` | `RQ-1` |
| `version` — DatasetVersionDatasetFacet | `1-0-1` | dataset | `datasetVersion` | `RQ-2` |
| `lifecycleStateChange` — LifecycleStateChangeDatasetFacet | `1-0-1` | dataset | `lifecycleStateChange` | see `R-04` — do not overload |
| `catalog` — CatalogDatasetFacet | `1-1-0` | dataset | `framework`, `type`, `name` | `RQ-5`, with `R-03` caveat |
| `columnLineage` — ColumnLineageDatasetFacet | `1-2-0` | dataset | `fields` | superseded by `lineage`, see `C-33` |
| `dataQualityMetrics` — DataQualityMetricsInputDatasetFacet | `1-0-3` | **inputFacets** | `columnMetrics` | `RQ-1` profiles — see `C-23` constraint |
| `inputStatistics` — InputStatisticsInputDatasetFacet | `1-0-0` | **inputFacets** | — | `RQ-1` profiles — see `C-23` constraint |
| `jobType` — JobTypeJobFacet | `2-0-4` | job | `processingType`, `integration` | `RQ-4` |

### 3.1 Field-level detail worth pinning

| ID | Claim | Confidence | Source |
|---|---|---|---|
| `C-25` | `DatasourceDatasetFacet` has exactly two optional fields: `name` (string) and `uri` (string, `format: uri`). Neither is required. | VERIFIED | `spec/facets/DatasourceDatasetFacet.json` |
| `C-26` | `TagsDatasetFacet` fields: `key` (required), `value` (required), `source` (optional, documented as "INTEGRATION\|USER\|DBT CORE\|SPARK\|etc."), `field` (optional, "Identifies the field in a dataset if a tag applies to one", example `email_address`). | VERIFIED | `spec/facets/TagsDatasetFacet.json` |
| `C-27` | `TagsDatasetFacet.field` provides **column-level** tagging within a dataset-level facet. The spec's own example is `{"key":"classification","value":"PII","source":"CONFIG","field":"tax_id"}`. | VERIFIED | `website/docs/spec/facets/dataset-facets/tag-facet.md` |
| `C-28` | `SymlinksDatasetFacet` is defined as "alternative identifiers for a single dataset… allows OpenLineage to understand that these different identifiers refer to **the same entity**, creating a **unified** lineage graph." | VERIFIED | `website/docs/spec/facets/dataset-facets/symlinks.md` |
| `C-29` | `HierarchyDatasetFacet.hierarchy` is an **ordered** array highest→lowest; "the order is important". Level types are open strings, example `DATABASE`, `SCHEMA`, `TABLE`. | VERIFIED | `spec/facets/HierarchyDatasetFacet.json` |
| `C-30` | `StorageDatasetFacet` requires `storageLayer` (documented allowed values: `iceberg`, `delta`) and offers optional `fileFormat` with allowed values `parquet, orc, avro, json, csv, text, xml`. It sets `additionalProperties: true`. | VERIFIED | `spec/facets/StorageDatasetFacet.json` |
| `C-31` | `DatasetTypeDatasetFacet.datasetType` is an open string, example values `TABLE\|VIEW\|FILE\|TOPIC\|STREAM\|MODEL\|JOB_OUTPUT`, plus optional `subType` (`MATERIALIZED`, `EXTERNAL`, `TEMPORARY`). | VERIFIED | `spec/facets/DatasetTypeDatasetFacet.json` |
| `C-32` | `LifecycleStateChangeDatasetFacet.lifecycleStateChange` is a **closed enum**: `ALTER, CREATE, DROP, OVERWRITE, RENAME, TRUNCATE`. It also carries `previousIdentifier: {namespace, name}` for renames. | VERIFIED | `spec/facets/LifecycleStateChangeDatasetFacet.json` |
| `C-33` | `ColumnLineageDatasetFacet` `transformationDescription` and `transformationType` are marked `"deprecated": true`; the live path is `fields.*.inputFields[].transformations[]`. | VERIFIED | `spec/facets/ColumnLineageDatasetFacet.json` |
| `C-34` | `JobTypeJobFacet` (`2-0-4`) carries `emissionPattern.eventContentMode` with values `ACCUMULATIVE` ("Events may contain only partial information and the complete information can be collected by combining information from all the events emitted") and `COMPLETE_SNAPSHOT` ("events contain complete state… can be processed independently"). | VERIFIED | `spec/facets/JobTypeJobFacet.json` |
| `C-35` | `DataQualityMetricsInputDatasetFacet` (`1-0-3`) supports `rowCount`, `bytes`, `fileCount`, `lastUpdated`, and per-column `nullCount`, `distinctCount`, `sum`, `count`, `min`, `max`, `quantiles`. `columnMetrics` is required. | VERIFIED | `spec/facets/DataQualityMetricsInputDatasetFacet.json` |
| `C-36` | `DocumentationDatasetFacet` (`1-1-0`) has `description` (required) and `contentType` (MIME, e.g. `text/markdown`). | VERIFIED | `spec/facets/DocumentationDatasetFacet.json` |
| `C-37` | `OwnershipDatasetFacet` recommends URN-shaped owner names: "It is recommended to define this as a URN. For example `application:foo`, `user:jdoe`, `team:data`". | VERIFIED | `spec/facets/OwnershipDatasetFacet.json` |

### 3.2 The `lineage` facet — new, and the exact semantic match for `RQ-4`

| ID | Claim | Confidence | Source |
|---|---|---|---|
| `C-38` | `LineageDatasetFacet` exists at `spec/facets/1-0-0/LineageFacet.json#/$defs/LineageDatasetFacet`, attaches at `dataset.facets.lineage` on a `DatasetEvent`. | VERIFIED | `spec/facets/LineageFacet.json` |
| `C-39` | Its stated purpose names this use case verbatim: "useful for structural relationships that have no natural event job, such as a view derived from tables, **an alias**, or **lineage curated in a data catalog**." | VERIFIED | `website/docs/spec/facets/dataset-facets/lineage.md` |
| `C-40` | The target is implicit — "the target is the dataset that carries the facet, so its namespace and name are not repeated inside the facet." | VERIFIED | `website/docs/spec/facets/dataset-facets/lineage.md` |
| `C-41` | It carries `inputs[]` (entity-level) and `fields{}` (field-level), each input being `{namespace, name, type: DATASET}` or a job reference, with optional `transformations[]`. | VERIFIED | `spec/facets/LineageFacet.json` |
| `C-42` | `LineageTransformation` has `type` (required, e.g. `DIRECT`/`INDIRECT`), `subtype` (e.g. `IDENTITY, AGGREGATION, FILTER, JOIN, GROUP_BY, WINDOW, SORT, CONDITIONAL`), `description` (free string), and `masking` (boolean). | VERIFIED | `spec/facets/LineageFacet.json` |
| `C-43` | **An empty `inputs` array is semantically distinct from an omitted one:** "An empty array explicitly means that the target has no tracked upstream source." | VERIFIED | `spec/facets/LineageFacet.json`, `LineageDatasetEntry.inputs` description |
| `C-44` | `LineageJobFacet` requires `entries[]`, each naming one target and only its own sources, explicitly to avoid the Cartesian-product false edges that arise from event-level `inputs`×`outputs`. | VERIFIED | `spec/facets/LineageFacet.json`; `website/docs/spec/facets/job-facets/lineage.md` |
| `C-45` | Precedence is stated: "The Lineage Dataset Facet supersedes the Column Lineage Dataset Facet for the relationships it describes. If both are present… consumers should use the lineage facet." | VERIFIED | `website/docs/spec/facets/dataset-facets/lineage.md` |
| `C-46` | **Maturity:** introduced by commit `2cfa2594b` ("spec: add explicit lineage facet (#4804)") dated 2026-08-14. `git tag --contains 2cfa2594b` returns **empty**. Latest release tag is `1.52.0`. It is absent from the CHANGELOG, including the Unreleased section. | VERIFIED | `git log`, `git tag --contains`, `CHANGELOG.md` |
| `C-47` | Client support at HEAD: Python has generated `client/python/src/openlineage/client/generated/lineage.py`; Java has `LineageFacetTest.java`. Both unreleased per `C-46`. | VERIFIED | filesystem |

> **`C-46` is the single most decision-relevant maturity fact in this document.** The facet is semantically ideal
> and operationally unavailable. Any plan that depends on it depends on unreleased code with zero consumer support.
> Re-check `git tag --contains 2cfa2594b` before acting on it.

---

## 4. Design decisions

### `D-01` — Use `DatasetEvent` as the registration primitive; one event per curation act

**Rests on:** `C-01`, `C-02`, `C-04`, `C-05`

Every time a curator learns a new fact about a datasource, emit one `DatasetEvent` carrying **only the facets that
changed** — but each such facet in **full**. Do not wait for completeness at the dataset level; do not send a delta
at the facet level. This matches the documented purpose of the event type and requires no synthetic job.

> **The producer rule, from `C-24`/`C-24d`.** Facets are atomic. Sending `schema` with one newly-documented column
> replaces the whole schema facet and destroys the other columns. Sending `tags` with one new tag drops every other
> tag. The curation tool must materialize the complete current value of every facet it touches and send that.
> Enforce it in one serialization layer, not at each call site.

Minimum viable first event — identity plus nothing else — is legal, because `facets` is optional (`C-08`):

```json
{
  "eventTime": "2026-08-29T10:00:00Z",
  "producer": "https://your.org/datasource-registry/1.4.2",
  "schemaURL": "https://openlineage.io/spec/2-0-2/OpenLineage.json#/$defs/DatasetEvent",
  "dataset": { "namespace": "oracle://prod-ora.provider.example:1521", "name": "ORCL.SALES.ORDERS" }
}
```

**Counter-consideration:** OpenLineage is a wire format, not a database. It has no query API, no curation UI, no
edit history, and no partial-update semantics of its own. The curation tool must own the master record; OpenLineage
is the emission and interchange format projected out of it. Do not attempt to use a lineage backend as the
system of record for in-progress curation.

### `D-02` — Model the CSV/JSON sample as a real dataset, not as a placeholder for the eventual table

**Rests on:** `C-14`, `C-30`, `C-31`, `RQ-2`

The sample file is a genuine artifact with its own identity, schema, and profile. Registering it as a dataset in
its own right means the eventual discovery of the real source **adds** a node and an edge rather than mutating or
invalidating the file's record. The provenance of any analysis done on the sample stays intact.

- namespace: `file://{host}` (or `s3://{bucket}`) per `C-14`
- name: the path / object key
- `datasetType`: `{"datasetType": "FILE"}` (`C-31`)
- `storage`: `{"storageLayer": "file", "fileFormat": "csv"}` — note `storageLayer` is required and its documented
  allowed values are `iceberg`/`delta` only (`C-30`); `additionalProperties: true` means other values validate, but
  this is a documented-vocabulary stretch. Flagged as `R-05`.
- `schema`: inferred columns
- `documentation`: provenance prose — where the extract came from, who supplied it, when

### `D-03` — Carry data profiles on a profiling `RunEvent`, not on the `DatasetEvent`

**Rests on:** `C-23`, `C-35`

This is a hard structural constraint, not a preference. `DataQualityMetricsInputDatasetFacet` and
`InputStatisticsInputDatasetFacet` are **input** facets, which live in `inputFacets`, which only exist on a
`RunEvent`'s `inputs[]` array. A `DatasetEvent` cannot carry them (`C-23`).

Therefore profiling must be modeled as what it actually is: a job that ran, read the file, and computed statistics.

```json
{
  "eventType": "COMPLETE",
  "eventTime": "2026-08-29T10:05:00Z",
  "producer": "https://your.org/profiler/2.1.0",
  "schemaURL": "https://openlineage.io/spec/2-0-2/OpenLineage.json#/$defs/RunEvent",
  "run": { "runId": "01920000-0000-7000-8000-000000000001" },
  "job": {
    "namespace": "your-org/curation",
    "name": "profile.sales_orders_sample",
    "facets": {
      "jobType": {
        "_producer": "https://your.org/profiler/2.1.0",
        "_schemaURL": "https://openlineage.io/spec/facets/2-0-4/JobTypeJobFacet.json#/$defs/JobTypeJobFacet",
        "processingType": "BATCH", "integration": "CUSTOM", "jobType": "PROFILE"
      }
    }
  },
  "inputs": [{
    "namespace": "file://curation-host",
    "name": "/landing/provider_a/sales_orders_2026-08-29.csv",
    "inputFacets": {
      "dataQualityMetrics": {
        "_producer": "https://your.org/profiler/2.1.0",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-3/DataQualityMetricsInputDatasetFacet.json#/$defs/DataQualityMetricsInputDatasetFacet",
        "rowCount": 48213, "bytes": 9114322,
        "columnMetrics": { "order_id": { "nullCount": 0, "distinctCount": 48213 } }
      }
    }
  }]
}
```

**Alternative if a profiling run is undesirable:** `DataQualityMetricsDatasetFacet` (note: no `Input` in the name)
exists at `spec/facets/DataQualityMetricsDatasetFacet.json` as a dataset-level facet. Verify its current status
before relying on it — it overlaps the input-scoped variant and the duplication looks historical.

### `D-04` — Replica provenance: two datasets and a real edge. Never symlinks.

**Rests on:** `C-28`, `C-38`–`C-47`
**This is the decision most specific to the stated use case.**

The requirement is to record that data was read *from a replica* and that the replica derives *from a named
production origin on a different platform*. Four mechanisms were evaluated.

| Option | Mechanism | Verdict | Reasoning |
|---|---|---|---|
| **A** | `SymlinksDatasetFacet` — list the Oracle prod table as an alternative identifier of the Snowflake replica | **Rejected** | `C-28`: symlinks assert the identifiers "refer to the same entity" and produce a "unified" graph. Collapsing prod and replica into one node destroys precisely the fact that must be documented — that the read happened against the replica and not against production. Symlinks are for one entity with two names (an S3 path and its Glue table), not two physical copies. **Independently corroborated by `C-55`:** the static-lineage proposal lists "create dataset symlinks more easily" (scoped to *naming variants* — FQDN vs. hostname, region vs. no-region) and "dataset connections" (connecting *2 distinct dataset entities*) as **separate** use cases. Your case is the second, not the first. |
| **B** | Two datasets + `lineage` facet on the replica's `DatasetEvent`, `inputs: [prod]` | **Target model** | `C-39` names "lineage curated in a data catalog" as the intended use. `C-40` makes the target implicit, so the replica's own event declares its origin. `C-42` allows `{"type":"DIRECT","subtype":"IDENTITY","description":"CDC replication"}`. Blocked today by `C-46` — unreleased, no consumer support. |
| **C** | Two datasets + an explicit replication job carrying the edge | **Ship this now** | Works against every backend. Two sub-variants below. |
| **D** | `DatasourceDatasetFacet` alone | **Complement, not a substitute** | `C-25`: two optional string fields. Records *what the datasource is*; creates no traversable edge. Use it alongside B or C, never instead. |

**Option C sub-variants, in descending order of consumer compatibility:**

| Variant | Event | Compatibility | Semantic fit |
|---|---|---|---|
| C-1 | Synthetic `RunEvent` pair (START/COMPLETE) for a `replicate.*` job, `inputs: [prod]`, `outputs: [replica]` | Highest — every backend consumes `RunEvent` | Weakest — asserts a run that you did not observe and that has no real `runId`. Honest only if the timestamps are labeled as assertions, not observations. |
| C-2 | `JobEvent` with declared `inputs`/`outputs` | `PENDING` — static-lineage ingestion is uneven across backends | Strong — `C-03`/`C-04`: this *is* the design-time declaration event. No fabricated run. |
| C-3 | `JobEvent` + `LineageJobFacet.entries` | Lowest — `C-46` | Strongest — `C-44` avoids Cartesian false edges, matters once one replication job covers many tables. |

**Convergence property:** B, C-1, C-2 and C-3 all use the *same two dataset identities*. Shipping C-1 today and
adding B later is additive — no renaming, no rewriting of history, no broken node identity. This is why the
compatibility tier can be chosen independently of the target model.

**Recommended composite for a replica dataset:**

```json
{
  "eventTime": "2026-08-29T11:00:00Z",
  "producer": "https://your.org/datasource-registry/1.4.2",
  "schemaURL": "https://openlineage.io/spec/2-0-2/OpenLineage.json#/$defs/DatasetEvent",
  "dataset": {
    "namespace": "snowflake://acme-prod",
    "name": "REPLICA_DB.SALES.ORDERS",
    "facets": {
      "datasetType": {
        "_producer": "https://your.org/datasource-registry/1.4.2",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-1/DatasetTypeDatasetFacet.json#/$defs/DatasetTypeDatasetFacet",
        "datasetType": "TABLE", "subType": "EXTERNAL"
      },
      "dataSource": {
        "_producer": "https://your.org/datasource-registry/1.4.2",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-1/DatasourceDatasetFacet.json#/$defs/DatasourceDatasetFacet",
        "name": "Provider A — Snowflake read replica",
        "uri": "snowflake://acme-prod"
      },
      "hierarchy": {
        "_producer": "https://your.org/datasource-registry/1.4.2",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/HierarchyDatasetFacet.json#/$defs/HierarchyDatasetFacet",
        "hierarchy": [
          { "type": "DATABASE", "name": "REPLICA_DB" },
          { "type": "SCHEMA",   "name": "SALES" },
          { "type": "TABLE",    "name": "ORDERS" }
        ]
      },
      "tags": {
        "_producer": "https://your.org/datasource-registry/1.4.2",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/TagsDatasetFacet.json#/$defs/TagsDatasetFacet",
        "tags": [
          { "key": "source_role",       "value": "REPLICA", "source": "CURATION" },
          { "key": "origin_namespace",  "value": "oracle://prod-ora.provider.example:1521", "source": "CURATION" },
          { "key": "origin_name",       "value": "ORCL.SALES.ORDERS", "source": "CURATION" },
          { "key": "replication_mode",  "value": "CDC", "source": "CURATION" },
          { "key": "access_constraint", "value": "PROD_READ_FORBIDDEN", "source": "CURATION" }
        ]
      },
      "lineage": {
        "_producer": "https://your.org/datasource-registry/1.4.2",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/LineageFacet.json#/$defs/LineageDatasetFacet",
        "inputs": [{
          "namespace": "oracle://prod-ora.provider.example:1521",
          "name": "ORCL.SALES.ORDERS",
          "type": "DATASET",
          "transformations": [
            { "type": "DIRECT", "subtype": "IDENTITY", "description": "Provider-managed CDC replication Oracle -> Snowflake", "masking": false }
          ]
        }]
      }
    }
  }
}
```

The `tags` block is deliberate redundancy: it renders in backends that show tags today, while `lineage` carries the
machine-traversable edge once support lands. The tag keys are a degraded projection of the same facts, which is
acceptable because they are cheap and lossy-but-readable.

### `D-05` — Credentials: the namespace *is* the vault key

**Rests on:** `C-10`, `C-48`–`C-50`

| ID | Claim | Confidence | Source |
|---|---|---|---|
| `C-48` | The Python client has a redaction framework: `RedactMixin` with a `_skip_redact` class-var allow-list. | VERIFIED | `client/python/src/openlineage/client/utils.py:142-147` |
| `C-49` | `client/python/redact_fields.yml` drives code generation of `_additional_skip_redact` per facet class. `DatasourceDatasetFacet` lists `name` and `uri`; `Dataset` lists `namespace` and `name`. | VERIFIED | `client/python/redact_fields.yml`; generated output `client/python/src/openlineage/client/generated/datasource_dataset.py` |
| `C-50` | The generated `DatasourceDatasetFacet` carries `_additional_skip_redact: ClassVar[list[str]] = ["name", "uri"]` and a `uri` validator that runs `urlparse`. | VERIFIED | `client/python/src/openlineage/client/generated/datasource_dataset.py` |
| `C-51` | The **Java** client ships credential stripping as production code: `public class JdbcDatasetUtils` dispatches over ~15 vendor `JdbcExtractor`s and routes URLs through `JdbcUrlSanitizer.dropSensitiveData()`. `io.openlineage.client.dataset.Naming` provides **28** platform naming classes whose constructors take no username, password, or token parameter. | VERIFIED | `client/java/src/main/java/io/openlineage/client/utils/jdbc/JdbcDatasetUtils.java`; `client/java/src/main/java/io/openlineage/client/dataset/Naming.java` |
| `C-52` | `JdbcUrlSanitizer.dropSensitiveData()` applies five regex passes in order: slash-delimited userinfo → `@`; colon-delimited userinfo → `$1`; `(?i)[,;&:]?(?:user\|username\|password)=[^,;&:()]+[,;&:]?` → removed; collapse duplicated delimiters; then `\?.*$` → **drops the entire query string**. The class is package-private; the public entry point is `JdbcDatasetUtils`. | VERIFIED | `client/java/src/main/java/io/openlineage/client/utils/jdbc/JdbcUrlSanitizer.java` |
| `C-53` | **Custom facet naming is normative, not stylistic.** "Custom facets **must** use a distinct prefix named after the project defining them to avoid collision with standard facets." Convention is `{prefix}{Name}{Entity}Facet` PascalCase with attachment key `{prefix}_{name}` camelCase — canonical shipped example `BigQueryStatisticsJobFacet` / key `bigQuery_statistics`. | VERIFIED | `spec/OpenLineage.md` §Facets |
| `C-54` | `_schemaURL` **must** be an immutable, canonical pointer: "The versioned URL must be an immutable pointer to the version of the facet schema. For example, it should include a tag of a git sha and **not a branch name**. This should also be a canonical URL. There should be only one URL used for a given version of a schema." | VERIFIED | `spec/OpenLineage.md` §Facets |
| `C-55` | Proposal 1837 lists as *separate* motivating use cases: "Create dataset symlinks more easily" — explicitly scoped to naming variants, "FQDN versus hostname, including region vs. not, differing query params" — and "**Dataset connections**: Ability to connect 2 dataset entities without additional information about the run that created that connection." | VERIFIED | `proposals/1837/static_lineage.md` §Use cases |

The list is an **allow-list of fields exempt from redaction** — the framework's default posture is to redact.
That `uri` is explicitly exempted encodes the assumption that a `DatasourceDatasetFacet.uri` is *already*
credential-free. The assumption is the producer's to uphold.

**Rules:**

1. The vault key is derived from the OpenLineage namespace, not the reverse. `oracle://prod-ora.provider.example:1521`
   → vault path `datasources/oracle/prod-ora.provider.example/1521`. Credential rotation never changes the namespace,
   so no lineage node is ever orphaned by a password change.
2. Never emit userinfo. `oracle://user:pass@host:1521` is not a legal namespace under `C-10` and would additionally
   fork dataset identity on every rotation.
3. Sanitize before emitting `DatasourceDatasetFacet.uri`, `CatalogDatasetFacet.metadataUri`, and
   `CatalogDatasetFacet.warehouseUri`. See `R-03`.
4. Never put connection details in `documentation.description` — it is free text with no redaction contract.
5. The user-facing display name (`server/database/schema/table`) comes from `hierarchy` + `name` (`C-29`), which is
   exactly the credential-free projection required by `RQ-5`.

### `D-06` — Ontology: dual-emit a custom facet plus a tags projection

**Rests on:** `C-17`, `C-18`, `C-19`, `C-26`, `C-27`

Neither mechanism alone is sufficient.

| | `TagsDatasetFacet` | Custom `ontology` facet |
|---|---|---|
| Structure | flat `key`/`value` strings | arbitrary nested JSON (`C-17`) |
| Column-level | yes, via `field` (`C-27`) | yes, if you model it |
| Term URI, ontology version, confidence, curator, timestamp | not representable without encoding into strings | native |
| Rendered by existing backends | yes — tags are a common first-class UI concept | no — stored verbatim, invisible (`C-18`) |
| Schema validation | by the spec | by your self-hosted schema (`C-19`) |

Emit both from one source of truth in the curation tool. The custom facet is the system of record; the tags block
is a lossy projection for human search and filtering.

> **Naming correction (`C-53`).** The facet key `"ontology"` is **non-conformant** — the spec *requires* a distinct
> project prefix. Use key `acme_ontology` (camelCase `{prefix}_{name}`) for a schema class named
> `AcmeOntologyDatasetFacet` (`{prefix}{Name}{Entity}Facet`), following the shipped example
> `bigQuery_statistics` / `BigQueryStatisticsJobFacet`. Substitute your own org prefix throughout.

```json
"acme_ontology": {
  "_producer": "https://your.org/datasource-registry/1.4.2",
  "_schemaURL": "https://your.org/schemas/openlineage/v2026.3.1/AcmeOntologyDatasetFacet.json#/$defs/AcmeOntologyDatasetFacet",
  "ontologyVersion": "2026.3",
  "classifications": [
    { "termUri": "https://your.org/ontology/term/CustomerOrder", "termLabel": "Customer Order",
      "scope": "DATASET", "confidence": 1.0, "curator": "user:jdoe", "classifiedAt": "2026-08-29T11:20:00Z" },
    { "termUri": "https://your.org/ontology/term/TaxIdentifier", "termLabel": "Tax Identifier",
      "scope": "FIELD", "field": "tax_id", "confidence": 0.92, "curator": "service:auto-classifier",
      "classifiedAt": "2026-08-29T11:20:00Z" }
  ]
},
"tags": {
  "_producer": "https://your.org/datasource-registry/1.4.2",
  "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/TagsDatasetFacet.json#/$defs/TagsDatasetFacet",
  "tags": [
    { "key": "ontology.term", "value": "CustomerOrder",  "source": "ONTOLOGY" },
    { "key": "ontology.term", "value": "TaxIdentifier",  "source": "ONTOLOGY", "field": "tax_id" }
  ]
}
```

Normative rules for the custom facet, all `VERIFIED`:

1. **Prefix is mandatory** (`C-53`) — "Custom facets *must* use a distinct prefix named after the project defining
   them." Key `{prefix}_{name}` camelCase; schema class `{prefix}{Name}{Entity}Facet` PascalCase.
2. **`_producer` and `_schemaURL` are required** (`C-17`).
3. **`_schemaURL` must be an immutable canonical pointer** (`C-54`) — "should include a tag of a git sha and *not
   a branch name*… There should be only one URL used for a given version of a schema." Version in the path, never
   a `main`/`latest` URL that mutates under you.
4. Custom facets can later be **promoted into the standard spec**, so designing one conformantly is not a dead end.
5. `spec/registry/` is the upstream registry for facets **contributed to OpenLineage** — an internal facet does not
   belong there unless you intend to propose it upstream.

### `D-07` — Model curation completeness explicitly; do not overload `lifecycleStateChange`

**Rests on:** `C-32`, `R-04`

`RQ-1` implies datasource records exist in an incomplete state for extended periods. OpenLineage has no built-in
notion of a draft or partially-curated entity. Represent it explicitly:

```json
{ "key": "registration_state", "value": "SAMPLE_ONLY", "source": "CURATION" }
```

with a controlled vocabulary owned by the curation tool, e.g.
`DECLARED → SAMPLE_ONLY → SOURCE_IDENTIFIED → CONNECTION_ESTABLISHED → CLASSIFIED → PRODUCTION`.

Once the `lineage` facet is available, `C-43` gives a second, sharper distinction that tags cannot express:
**omitted `inputs` = "upstream unknown, still investigating"; `inputs: []` = "investigated, confirmed no tracked
upstream."** That is a genuinely useful curation state and is worth using as soon as the facet ships.

---

## 5. End-to-end flow

Stage IDs are referenced by the risk table. "Idempotent" means re-emitting the identical event is harmless.

| # | Stage | Trigger | Event | Entity | Key facets | Idempotent |
|---|---|---|---|---|---|---|
| `S-0` | Declare intent | Analyst opens a datasource record | *(none — internal only)* | — | — | n/a |
| `S-1` | Register sample file | CSV/JSON extract received | `DatasetEvent` | `file://curation-host` / `/landing/provider_a/orders_2026-08-29.csv` | `datasetType(FILE)`, `storage(fileFormat=csv)`, `documentation`, `tags(registration_state=SAMPLE_ONLY)` | yes |
| `S-2` | Infer schema | Parser reads header + sample rows | `DatasetEvent` | same as `S-1` | `schema` | yes |
| `S-3` | Profile the sample | Profiler job runs | `RunEvent` START+COMPLETE | job `your-org/curation` / `profile.orders_sample` | `inputFacets.dataQualityMetrics`, `inputFacets.inputStatistics`, `jobType` | no — new `runId` each time |
| `S-4` | Identify true origin | Analyst learns the prod table exists | `DatasetEvent` | `oracle://prod-ora.provider.example:1521` / `ORCL.SALES.ORDERS` | `datasetType(TABLE)`, `hierarchy`, `dataSource`, `tags(access_constraint=PROD_READ_FORBIDDEN, registration_state=SOURCE_IDENTIFIED)` | yes |
| `S-5` | Identify replica | Analyst learns which replica is readable | `DatasetEvent` | `snowflake://acme-prod` / `REPLICA_DB.SALES.ORDERS` | `datasetType`, `hierarchy`, `dataSource`, `tags(source_role=REPLICA, origin_*, replication_mode)` | yes |
| `S-6` | Assert replication edge | Both `S-4` and `S-5` complete | **Tier C-1:** `RunEvent` for `replicate.orders`; **Tier B:** `lineage` facet on the `S-5` `DatasetEvent` | job or replica dataset | `jobType(BATCH/CUSTOM)`; or `lineage.inputs=[prod]` + `transformations[DIRECT/IDENTITY]` | B: yes / C-1: no |
| `S-7` | Bind sample to source | Analyst confirms the CSV came from the replica | `JobEvent` (or `RunEvent`) for `extract.orders_sample` | job | `inputs=[replica]`, `outputs=[sample file]` | JobEvent: yes |
| `S-8` | Register connection | Credentials obtained | *(no OpenLineage event)* | vault entry keyed by `S-5` namespace | — | n/a |
| `S-9` | Confirm reachability | Connection test succeeds | `DatasetEvent` | `S-5` dataset | `tags(registration_state=CONNECTION_ESTABLISHED)`, refreshed `schema` from live DDL | yes |
| `S-10` | Classify by ontology | Curator/auto-classifier assigns terms | `DatasetEvent` | `S-5` (and `S-4` for the logical entity) | custom `ontology` facet + `tags(ontology.term)` per `D-06` | yes |
| `S-11` | Enrich progressively | Any new fact learned — owner, description, column semantics | `DatasetEvent` | any | only the changed facets: `ownership`, `documentation`, `schema` | yes |
| `S-12` | Begin live ingestion | Real pipeline runs | `RunEvent` | your ETL job | full runtime lineage; `inputFacets` on read | no |
| `S-13` | Supersede the sample | Live source replaces the extract | `DatasetEvent` | sample file dataset | `tags(registration_state=SUPERSEDED)`; optionally `lifecycleStateChange` | yes |
| `S-14` | Retract a fact | A previously asserted facet proves wrong | `DatasetEvent` | any | `{"<facetName>": {"_deleted": true}}` per `C-20` | yes |
| `S-15` | Retire | Source decommissioned | `DatasetEvent` | any | `lifecycleStateChange: DROP` (`C-32`) | yes |

### 5.1 Resulting graph

```
oracle://prod-ora.provider.example:1521
  ORCL.SALES.ORDERS                       [TABLE, PROD_READ_FORBIDDEN]
        │
        │  S-6  replication edge
        │  Tier B: lineage.inputs on replica  |  Tier C-1: replicate.orders RunEvent
        ▼
snowflake://acme-prod
  REPLICA_DB.SALES.ORDERS                 [TABLE, source_role=REPLICA, ontology: CustomerOrder]
        │
        │  S-7  extract.orders_sample
        ▼
file://curation-host
  /landing/provider_a/orders_2026-08-29.csv   [FILE, csv, profiled at S-3, SUPERSEDED at S-13]
```

Three nodes, two edges. The "we read the replica, not production" fact is a *topological* property of the graph —
it survives lineage traversal and cannot be lost the way a tag on a merged node could be. That is the whole
argument against Option A in `D-04`.

### 5.2 Requirement → mechanism traceability

| Req | Primary mechanism | Supporting | Stages | Residual gap |
|---|---|---|---|---|
| `RQ-1` | `DatasetEvent`, one per curation act (`D-01`) | `_deleted` for retraction (`C-20`) | `S-1`,`S-2`,`S-9`,`S-11`,`S-14` | Merge semantics unverified — `R-01` |
| `RQ-2` | File dataset with `datasetType(FILE)` + `storage` (`D-02`) | `schema`, profiling run (`D-03`) | `S-1`–`S-3`,`S-7`,`S-13` | `storageLayer` vocabulary — `R-05` |
| `RQ-3` | Two datasets, two namespaces, native naming per platform (`C-11`,`C-12`) | `hierarchy` for display | `S-4`,`S-5` | Snowflake namespace format lock-in — `R-02` |
| `RQ-4` | `lineage` facet (target) / replication job (today) (`D-04`) | `dataSource`, `tags` projection | `S-6` | `lineage` unreleased — `C-46`; static ingestion — `R-06` |
| `RQ-5` | Credential-free namespace as vault key (`D-05`) | `RedactMixin` allow-list (`C-48`–`C-50`) | `S-8` | URI sanitization discipline — `R-03` |
| `RQ-6` | Custom `ontology` facet + `tags` projection (`D-06`) | `field` for column scope (`C-27`) | `S-10` | Custom facets invisible to backends — `C-18` |

---

## 6. Risks

| ID | Risk | Severity | Basis | Mitigation |
|---|---|---|---|---|
| `R-01` | ~~Facet merge semantics are consumer-defined~~ — **RESOLVED, and the resolution is a hard producer rule.** Semantics are normatively specified (`C-24`): event-level additive, facet-level *atomic replacement by name*. The residual risk is now the opposite of what was feared: a producer that emits a facet containing only the field it just learned will **silently destroy every other field in that facet**. Emitting `schema` with one newly-documented column wipes the other columns. | **High** (was Critical) | `C-24`, `C-24a`–`C-24d` | **Never emit a partial facet.** The producer must materialize the complete current state of any facet it touches, from the curation tool's own master record, and send the whole object. This is why `D-01`'s "one event per curation act" means *one event carrying the full current value of each changed facet* — not a delta. Enforce it in one serialization layer, not per call site. |
| `R-02` | Snowflake namespace format is effectively irreversible. Choosing the legacy account-locator form and later migrating to `orgname-accountname` breaks node identity: "existing lineage nodes won't connect to new ones." | High | `C-13` | Standardize on `snowflake://{org}-{account}` from the first event. Record the chosen convention in the curation tool as data, not as code. |
| `R-03` | JDBC-style URIs smuggle credentials in userinfo *and* query parameters. `CatalogDatasetFacet.metadataUri`'s own example is `jdbc:mysql://host:3306/iceberg_database`; real JDBC URLs routinely append `?user=…&password=…`. The `_skip_redact` allow-list (`C-50`) exempts `uri` from redaction, so nothing downstream catches it. | Medium (was High) — **Java: mitigated**, Python: open | `C-50`, `C-51`, `C-52` | **Java gets this free**: use `JdbcDatasetUtils` (`C-51`). **Python has no equivalent — port `JdbcUrlSanitizer`'s five rules** (`C-52`). Note it drops *all* query parameters rather than allow-listing them; adopt that stricter posture. Unit-test against the vendor URL forms you actually use. |
| `R-04` | Overloading `LifecycleStateChangeDatasetFacet` for curation state. Its enum is closed (`ALTER, CREATE, DROP, OVERWRITE, RENAME, TRUNCATE`) and describes the **data object's** lifecycle, not the **catalog entry's** curation maturity. | Medium | `C-32` | Use `D-07`. Reserve `lifecycleStateChange` for real DDL against the real object. |
| `R-05` | `StorageDatasetFacet.storageLayer` is required but its documented allowed values are only `iceberg` and `delta`. A plain CSV on a filesystem has no natural value. | Low | `C-30` | `additionalProperties: true` means `"file"` or `"local"` validates. Pick one, document it, apply it consistently. Cosmetic, not blocking. |
| `R-06` | Static-lineage ingestion support is uneven across backends. `DatasetEvent`/`JobEvent` have existed since ~0.29 but consumer support has historically lagged the spec. If the chosen backend drops them, `D-01` collapses. | **Critical** | `PENDING` — see §7 | Verify before committing. Fallback: model every curation act as a `RunEvent` from a synthetic `curation` job, which every backend consumes. Ugly but universal — and dataset identities are unchanged, so it is a swappable transport decision, not a model decision. |
| `R-07` | Custom facets are invisible to backend UIs. The `ontology` facet is stored and retrievable but will not render or be searchable. | Medium | `C-18` | The `tags` projection in `D-06` exists for exactly this. Accept that the ontology system of record is the ontology tool, and OpenLineage carries a reference, not the authority. |
| `R-08` | Depending on the `lineage` facet before it ships. It is in no release tag and absent from the CHANGELOG. | High | `C-46` | Build to the Tier C-1 model, keep the dataset identities that Tier B needs, and re-check `git tag --contains 2cfa2594b` before switching. |

---

## 7. `PENDING` — consumer support matrix

A five-angle web research pass (105 agents, adversarial 3-vote verification) was run on 2026-08-29. Four angles
returned high-confidence findings, all folded into §2–§6 above. **The consumer/backend angle returned no verified
claims and remains open.** That is reported as a gap, not filled by inference.

Resolved by that pass — no longer pending:

| Question | Answer | Now recorded as |
|---|---|---|
| Do backends merge facets across events, or snapshot? | Neither. The *spec* mandates per-facet-name atomic replacement with event-level additivity. Not a backend choice. | `C-24`, `C-24a`–`C-24d`; `R-01` rewritten |
| Is credential exclusion normative or conventional? | Convention + tested implementation, **not** a normative spec prohibition. Java ships `JdbcUrlSanitizer`; the 28 `Naming` classes take no credential parameter. | `C-51`, `C-52` |
| Custom facet naming rules? | Normative: prefix mandatory, `{prefix}_{name}` key, immutable `_schemaURL`. | `C-53`, `C-54` |
| Is the symlinks rejection defensible? | Yes — the proposal itself separates "symlinks" (naming variants) from "dataset connections" (2 distinct entities). | `C-55` |

Still open. **Verify these against your chosen backend before committing:**

| Question | Why it matters | Blocks |
|---|---|---|
| Does Marquez ingest `DatasetEvent`/`JobEvent`, and render their facets? | `D-01` and the whole static-lineage model | `R-06` |
| Same for DataHub, OpenMetadata, Atlan, Egeria, Manta | Backend selection | `R-06` |
| Does any backend implement `_deleted` tombstones correctly? | `S-14` retraction silently no-ops if not | `R-01` |
| Which backends surface `TagsDatasetFacet` in search/filter UI? | `D-06` dual-emit only pays off if tags render | `R-07` |
| Does any backend render `SymlinksDatasetFacet` as node merging? | Would confirm the `D-04` Option A rejection empirically | — |
| Any consumer support for `LineageDatasetFacet` yet? | Would promote Tier B from target to shippable | `R-08` |

The cheapest way to close all six: stand up the candidate backend, emit the `S-1`→`S-6` sequence from §5, and
read back. That is a half-day spike and it de-risks every Critical and High item in §6.

---

## 8. Verification checklist for a reviewing agent

Ordered by value. Each is independently checkable.

1. **Confirm `C-24` first** — read `spec/OpenLineage.md` §Facets and `proposals/1837/static_lineage.md` §update
   lifecycle. Everything in `D-01` and `R-01` turns on the replacement rule. The proposal carries a three-event
   worked example with an `owner` facet; if that example says what §2.3 claims, the producer rule holds.
2. Re-run `git tag --contains 2cfa2594b`. If it now returns a tag, `C-46` is stale and `D-04` Tier B is promotable.
3. Confirm `C-23` by attempting to place `dataQualityMetrics` on a `DatasetEvent` and validating against
   `spec/OpenLineage.json`. It should fail. If it validates, `D-03` is unnecessary.
4. Confirm `C-10` by scanning the full `website/docs/spec/naming.md` table for any namespace form containing
   userinfo. A single counterexample weakens `D-05`. Note `C-51`: the guarantee is convention plus tested
   implementation, **not** a normative spec prohibition — do not overstate it.
5. Verify `R-06` empirically against the target backend. It is now the only Critical open item.
6. Confirm `C-43` — that `inputs: []` and omitted `inputs` are genuinely distinguished by the schema and not merely
   by prose. This underpins the `D-07` curation-state refinement.
7. Check whether `DataQualityMetricsDatasetFacet` (dataset-scoped) is current or vestigial; it would offer an
   alternative to `D-03`.
8. Re-derive the facet `$id` versions in §3 against `spec/facets/*.json` at your HEAD. They drift.

### Changelog for this document

| Rev | Change |
|---|---|
| 2 | `R-01` resolved and inverted — facet semantics are spec-mandated atomic replacement (`C-24`), not consumer-defined. `R-03` downgraded — Java ships `JdbcUrlSanitizer` (`C-51`, `C-52`). Custom facet key corrected to require a project prefix (`C-53`, `C-54`). Symlinks rejection independently corroborated (`C-55`). Consumer-support angle reported as still open. |
| 1 | Initial analysis from repo primary sources at `b995ee00c`. |

## 9. Known documentation defects in this repo

Minor, but they will mislead an implementer reading the docs rather than the schemas.

| Location | Defect |
|---|---|
| `website/docs/spec/facets/dataset-facets/data_source.md` | Example uses `"url"`; schema `1-0-1` defines the field as `"uri"`. The example also cites `1-0-0` while the schema is `1-0-1`. |
| `website/docs/spec/facets/dataset-facets/tag-facet.md` | `_producer`/`_schemaURL` are shown as siblings of `inputs.facets` rather than inside the `tags` facet object. Malformed as written. |
| `website/docs/spec/facets/dataset-facets/symlinks.md` | `identifiers` example is a JSON array containing bare key-value pairs rather than an object. Will not parse. |
| `website/docs/spec/facets/custom-facets.md` | Imports `openlineage.client.run` and `openlineage.client.facet`, the pre-`facet_v2` API. Current code is `openlineage.client.event_v2` / `openlineage.client.generated.*`. |
| `spec/Naming.md` | Obsolete stub; still linked as "The OpenLineage Naming Spec" from the bottom of `website/docs/spec/naming.md`. |

---

*Sources are file paths in the OpenLineage repository at commit `b995ee00c` (2026-08-28), release `1.52.0`.
Facet versions are `$id` values at that commit and drift between releases.*
