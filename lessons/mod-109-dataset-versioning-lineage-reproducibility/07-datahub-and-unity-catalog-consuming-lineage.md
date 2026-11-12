# Chapter 07 — DataHub and Unity Catalog: Consuming Lineage Before a Dataset Change

## Why this chapter exists

Chapter 06 built the emission layer. On its own, a stream of OpenLineage events in a PostgreSQL-backed Marquez instance is a *proof of concept*: it shows the graph is coherent, but it doesn't make anyone's job easier. The operational payoff of a lineage program comes at the **consumption** layer — when an engineer about to change a production dataset can, in under a minute, answer:

> **If I change `aicg.sft_chat_owned_ugc_2026_q3` tomorrow, what will break?**

That question — the **upstream-impact query** — is what this chapter operationalises. We walk through two dominant consumption surfaces: **DataHub** (open-source catalog, now maintained by DataHub Inc. / Acryl, LF AI & Data), and **Databricks Unity Catalog** (a vendor-managed catalog bundled with Databricks).

Both consume OpenLineage events (or integrate natively with their emission sources), both expose lineage as a browsable graph plus an API, and both have their place. The decision rule is simple: if the organisation is Databricks-centric, Unity Catalog is the default; otherwise DataHub is the default open-source answer.

Primary sources: <https://datahubproject.io/>, <https://docs.databricks.com/aws/en/data-governance/unity-catalog/>.

## DataHub — the open-source consumer

DataHub is a catalog platform built around a strongly-typed metadata model (the "entity-aspect" model) with built-in lineage, search, documentation, data-quality integration, and schema registry. It was open-sourced by LinkedIn in 2019 and is now stewarded by Acryl Data; it is a Linux Foundation AI & Data project.

### The DataHub model

Core entities (abbreviated to the ones that matter for AI data engineering):

- **Dataset** — a logical dataset (a Delta / Iceberg table, a Parquet prefix, a Kafka topic, a Snowflake view, a Hugging Face dataset).
- **Data Job** — a logical pipeline step (an Airflow task, a Spark job, a Dagster op).
- **Data Flow** — a pipeline (an Airflow DAG, a Dagster job).
- **ML Model** — a deployed model (an MLflow / Vertex / SageMaker model).
- **ML Model Group** — a model family.
- **ML Feature Table** — a feature-store table.
- **Chart / Dashboard** — a BI consumer.
- **Term** — a glossary term; used to tag datasets semantically.
- **Tag** — a free-form tag.

Lineage is modelled as *upstream* / *downstream* relationships between entities. A Dataset `A` has an `upstreamLineage` aspect naming the datasets that produced it and the data jobs that performed the transformation.

### DataHub ingests OpenLineage natively

DataHub exposes an **OpenLineage-compatible ingestion endpoint**. Point your pipeline's OpenLineage transport at it:

```ini
# airflow.cfg
[openlineage]
transport = {"type": "http", "url": "https://datahub.example.com:8080/openapi/openlineage/api/v1/lineage"}
```

DataHub translates incoming OpenLineage `START` / `COMPLETE` events (POST to GMS at `/openapi/openlineage/api/v1/lineage`, per the current [DataHub OpenLineage docs](https://docs.datahub.com/docs/lineage/openlineage/)) into:

- A `Dataset` entity per input / output (created if absent).
- A `DataJob` entity per job.
- An `UpstreamLineage` / `FineGrainedLineage` aspect wiring them together (from the `columnLineage` facet).
- `SchemaMetadata` from the `schema` facet, `Operation` time-series from `outputStatistics` + `lifecycleStateChange`, `DatasetProfile` from `dataQualityMetrics`, `GlobalTags` / `Ownership` / `DatasetProperties` / `InstitutionalMemory` from their respective facets (see the [`openlineage-converter` module](https://docs.datahub.com/docs/metadata-integration/java/openlineage-converter) for the full mapping; coverage expanded substantially in [PR #19734](https://github.com/datahub-project/datahub/pull/19734)).
- Remaining custom-facet payloads translated into `datasetProperties` or `customProperties`.

In addition, DataHub has first-party *sources* that extract lineage directly from the substrate: dbt artifacts, Airflow (via plugin), Databricks, Snowflake, BigQuery, Kafka, Hive Metastore, Glue, Delta, Iceberg, and many others. These composeable sources mean DataHub can often reconstruct a lineage graph from the substrate even without OpenLineage emission — useful for retrofitting onto legacy pipelines.

### The impact-analysis query

In the DataHub UI, navigate to `aicg.sft_chat_owned_ugc_2026_q3`, click **Lineage**, select **Downstream**, and set depth 3+. You see every dataset, job, chart, dashboard, and model that transitively consumes this snapshot.

Programmatically, via the DataHub GraphQL API:

```graphql
query DownstreamImpact {
  dataset(urn: "urn:li:dataset:(urn:li:dataPlatform:iceberg,aicg.sft_chat_owned_ugc_2026_q3,PROD)") {
    downstream: lineage(input: { direction: DOWNSTREAM, start: 0, count: 100, degree: [1, 2, 3] }) {
      total
      relationships {
        entity {
          urn
          type
          ... on Dataset {
            name
            platform { name }
          }
          ... on MLModel {
            name
          }
          ... on DataJob {
            jobId
            dataFlow { flowId }
          }
        }
      }
    }
  }
}
```

The response is the set of entities downstream of this dataset within three hops. That is the answer to "if I change this, what breaks."

### What else you get with DataHub (that Marquez doesn't)

- **Search.** Elasticsearch-backed, over every entity, with facets by platform, tag, owner, domain. Teams can discover datasets by tag (`training`, `eval`, `preference-pair`, `sensitive-pii`) without browsing.
- **Column-level lineage.** For tables with SQL-shaped transforms (Spark, dbt, Snowflake stored procs), DataHub shows which output columns were derived from which input columns, with the transform name.
- **Business glossary.** Terms (`PII`, `PHI`, `Licensed partner data`) that can be attached to datasets and searched. Useful for mod-110 and mod-102 coordination.
- **Data-quality integration.** Great Expectations and Soda assertions are surfaced on the dataset page. The `dataQualityAssertions` facet from Chapter 06 lands here.
- **Documentation.** Per-entity documentation with Markdown. The Datasheet from mod-102 is a natural per-dataset doc.
- **Ownership + access-control metadata.** Per-dataset owner, consumer, access-tier. Not an access-control plane, but a *documented-intent* plane that compliance uses.
- **Timeline.** Every change to a dataset's metadata and lineage is time-stamped; you can scrub the graph at any prior time.

### Deployment notes

DataHub deploys as a set of microservices (metadata service, GMS, frontend, ingestion jobs) with Kafka, Elasticsearch, and MySQL / PostgreSQL. The reference deployment is a Helm chart. Operating at scale is nontrivial — `ai-infra-mlops` (peer, L25) typically owns the operation; this role owns the *content* of the catalog.

## Databricks Unity Catalog — the vendor-native consumer

Unity Catalog (UC) is Databricks' catalog layer. It is a single pane across tables (Delta / Iceberg), volumes (unstructured files), ML models (MLflow), functions (SQL + Python UDFs), and — load-bearing for this chapter — **lineage**.

### The Unity Catalog model

- Three-level namespace: `catalog.schema.object` (table, volume, model, function). A model is a first-class object alongside tables, so model-to-training-dataset relationships are native.
- **System tables** expose catalog metadata as queryable Delta tables: `system.access.audit`, `system.access.column_lineage`, `system.access.table_lineage`, `system.information_schema.*`. The lineage table is the one that matters for this chapter.
- **Attribute-based access control** via tags and columns; row-level + column-level masking.
- **Native data-quality integration** via Delta Live Tables + Lakehouse Monitoring.
- **MLflow Model Registry** is a UC-native entity; a registered model carries its training-dataset references and inference-request logs.

### How lineage is captured

Unity Catalog captures lineage **automatically** for workloads executed in a Databricks workspace:

- Any SQL query or Spark DataFrame operation reads and writes tables; UC records table-level and column-level lineage from the query plan.
- A Python notebook using `spark.read.table(...)` and `df.write.saveAsTable(...)` produces lineage edges.
- A Delta Live Tables pipeline produces per-expectation lineage with the quality assertions attached.
- MLflow runs log `data_input` + `data_input_schema` metadata; a `mlflow.data.log_input(...)` call ties the run to a Delta table version.

Lineage is queryable via the UI, the REST API, and the `system.access.table_lineage` / `system.access.column_lineage` Delta tables.

### The impact-analysis query — the SQL form

Because Unity Catalog exposes lineage as system tables, the impact query is a SQL query:

```sql
-- every downstream table (1-hop) of sft_chat_owned_ugc_2026_q3
SELECT
  source_table_full_name,
  target_table_full_name,
  entity_type,
  event_time
FROM system.access.table_lineage
WHERE source_table_full_name = 'aicg.default.sft_chat_owned_ugc_2026_q3'
  AND event_date >= current_date() - INTERVAL 90 DAYS
ORDER BY event_time DESC;

-- every downstream MLflow run that read a version of this table; join to MLflow's
-- experiment / run system tables (schema `system.mlflow`) to get the trained model.
-- Note: there is no `system.ml.models` system table as of Nov 2026 — registered
-- models live in `information_schema` and the models API; the schemas documented
-- under https://docs.databricks.com/aws/en/admin/system-tables/ that touch ML are
-- `system.mlflow` (experiments, runs, metrics) and `system.serving` (endpoints).
SELECT DISTINCT
  runs.experiment_id,
  runs.run_id,
  runs.start_time
FROM system.access.table_lineage AS lineage
JOIN system.mlflow.runs_latest AS runs
  ON lineage.entity_run_id = runs.run_id
WHERE lineage.source_table_full_name = 'aicg.default.sft_chat_owned_ugc_2026_q3';
```

(Full current column list for `system.access.table_lineage` / `column_lineage` is in the [system-tables lineage reference](https://docs.databricks.com/aws/en/admin/system-tables/lineage); the lesson uses the stable subset.)

The SQL interface is a strong point — the impact analysis is a dashboard-shaped artifact a data engineer can share with the team.

### OpenLineage and Unity Catalog

Unity Catalog emits OpenLineage events too — Databricks added OpenLineage emission to the Spark lineage layer. External Marquez / DataHub consumers can therefore ingest UC lineage *and* lineage from non-Databricks sources into a unified graph.

The composition pattern:

- Databricks-internal workloads → UC-native lineage + OpenLineage emission out to DataHub.
- Non-Databricks workloads (Airflow tasks, Dagster ops, Kubernetes-hosted Python pipelines) → direct OpenLineage emission to DataHub (or UC's OpenLineage ingestion).
- DataHub is the catalog that spans both; UC is the catalog that is native to Databricks.

Teams that operate both do not have to pick; they need to decide which catalog is the *authoritative* search surface for which audience.

## The upstream-impact workflow — before a dataset change

Independent of catalog, the operational workflow before touching a production dataset:

1. **Resolve the production snapshot identity** — lookup the pinned snapshot ID (lakeFS tag, Iceberg snapshot ID, Delta version) that production currently reads.
2. **Run the downstream impact query** on that dataset — list every job, model, dashboard, and eval run that reads it, within the last 90 days (or your organisation's retention window).
3. **Classify the downstream consumers:**
   - *Active trainers / fine-tuners* — a change requires a training-run rerun. Notify `fine-tuning-engineer`.
   - *Active eval jobs* — a change requires an eval-set-contamination recheck (mod-106). Notify `model-evaluation-engineer`.
   - *Monitoring dashboards* — a change may break their schema. Notify the dashboard owners.
   - *Downstream datasets* — recursively walk one more hop; their downstreams are your downstreams.
4. **Open a PR (or Nessie / lakeFS branch)** with the change. Attach the impact list to the PR.
5. **Get sign-off** from each affected consumer's owner. The sign-off is the documented decision.
6. **Merge / tag the new snapshot** and update the pinned version in each affected consumer.

This is not a step most organisations consistently perform today — but it is *the* operation that lineage programs justify. Every other benefit (search, documentation, discovery) is downstream of being able to answer this question quickly.

## Column-level lineage — where it earns its keep

Column-level lineage is the catalog feature most commonly ignored at setup and most commonly sought at audit time. The payoffs for AI training-data work:

- **PII audit.** For every column that could carry PII, show every downstream column it reaches. A reviewer confirms that a PII column exits the scrub step and never re-enters a *name* column downstream.
- **Licence-flag audit.** For every source-licence column, show every downstream column that inherited the licence constraint. A reviewer confirms that CC-BY-NC rows never reach a commercial training set.
- **Decontamination audit.** For every benchmark-text column, show every downstream column that could carry the benchmark text. A reviewer confirms the firewall step severs the link.

Both DataHub and Unity Catalog produce column-level lineage automatically from Spark / SQL plans. For transforms outside Spark (a pure-Python filter), you can emit column-level lineage explicitly via OpenLineage's `columnLineage` facet.

## What the catalog is **not**

Mark the boundary explicitly:

- The catalog is **not** an access-control plane. UC has native ACLs; DataHub has metadata about ownership and intent, not enforcement. Access control is upstream of this module — you document intent here; the actual enforcement is the substrate's.
- The catalog is **not** a signing / attestation surface. Signatures are the `cosign` output + Rekor entry (Chapter 08); the catalog *references* the signature URL in dataset properties but does not verify it.
- The catalog is **not** a feature store. Feature stores (Feast, Tecton, Databricks Feature Store) serve features at training and inference time; the catalog indexes them.
- The catalog is **not** the single source of truth for the dataset. The *snapshot* is the source of truth; the catalog is an index. Breaking the snapshot breaks the data; breaking the catalog breaks discovery.

## Operational anti-patterns

- **The write-only catalog.** Events pour in; no one queries them; the frontend sees 10 visits per week from the team that set it up. Pair catalog deployment with a *specific* workflow the team uses it for — the impact-analysis workflow above is the canonical one.
- **The documentation-only catalog.** Datasets have great READMEs; lineage is empty because nothing emits events. This is a wiki with extra steps — the catalog's value is the *computed* graph, not the hand-authored pages.
- **The siloed catalog.** DataHub for the data-platform team, UC for Databricks users, a separate Collibra for compliance. Three partial views, no joined graph. Prefer one authoritative catalog (DataHub in a non-Databricks shop; UC in a Databricks shop) with the others as subscribers via OpenLineage ingestion.
- **The perpetual-POC catalog.** The team evaluates DataHub every quarter and never commits. Impact-analysis queries are expensive precisely because they require a complete graph — partial-coverage catalogs do not answer the question. Pick one, deploy it, cover the training-data pipelines end-to-end.

## Peer coordination — who owns what

- **You (AI Data Engineer)** are the primary producer of training-data lineage and the primary consumer of impact-analysis queries. You tag datasets with glossary terms (`PII`, `preference-pair`, `pretrain`, `eval`, `licensed-partner`), own the Datasheet documentation per dataset, and sign off on dataset changes.
- **`ai-infra-mlops` (peer, L25)** owns the catalog operation — DataHub Helm chart, UC configuration, Marquez reliability.
- **`ai-governance-analyst` (L15)** uses the catalog as read-only evidence for impact assessments. Their DPIA references dataset URNs and lineage queries.
- **`ai-risk-engineer` (peer, L25)** uses the catalog to trace the harm-model surface (which downstream models consume which sensitive-attribute columns).
- **`fine-tuning-engineer` and `model-evaluation-engineer` (peers, L30)** are impact-analysis consumers — they get the "this dataset is about to change" notification.
- **`training-pipeline-engineer` (next-up, L35)** is an impact-analysis consumer and reads the cluster-ingest handoff contract (Chapter 11) from the catalog's dataset property set.

## Summary

- **DataHub** (open-source, LF AI & Data) and **Databricks Unity Catalog** (vendor-managed) are the two dominant lineage consumers. DataHub spans heterogeneous substrates; UC is Databricks-native.
- Both consume OpenLineage events (and in UC's case, emit them too). A unified graph is practical even when both catalogs coexist.
- The single operational payoff of the lineage program is the **upstream-impact query** — "if I change this dataset, what breaks?" — answerable in under a minute, with downstream datasets, jobs, dashboards, and models enumerated.
- **Column-level lineage** earns its keep at audit time: PII, licence, and decontamination audits all become column-graph queries.
- Deploy **one authoritative catalog**, cover the training-data pipelines end-to-end, and define a documented workflow around the impact query. A catalog with no workflow is lineage-theatre.
- The catalog is an **index**, not a source of truth. Snapshots (Chapters 02–05), signatures (Chapter 08), and manifests (Chapter 09) are the substrate the catalog points at.

Chapter 08 picks up the signing layer — Sigstore `cosign`, Fulcio, Rekor, and SLSA v1.0 provenance attestations — and turns the snapshot identity into a cryptographic chain of custody.
