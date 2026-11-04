# Chapter 06 — OpenLineage and Marquez: Authoring the Lineage Graph

## Why this chapter exists

Chapters 02–05 pinned four versioning substrates. Each stores *what* exists and *when* it changed. None of them, on its own, answers the question **"what produced this snapshot?"** — because the producer is a *pipeline*, and the pipeline lives outside the substrate.

The answer is an open protocol: **OpenLineage** (Linux Foundation / LF AI & Data). OpenLineage standardises the JSON shape of a lineage *event* — a job, a run, its inputs, its outputs, and typed *facets* that carry schema, data-quality metrics, source-code references, column-level lineage, and anything else. The reference consumer is **Marquez** (LF AI & Data); the commercial / open-core consumers are DataHub, Unity Catalog, Collibra, Alation, Astronomer / Astro, and Dagster Cloud (Chapter 07 covers consumption; this chapter covers emission).

Primary sources: <https://openlineage.io/>, <https://openlineage.io/docs/spec/>, <https://marquezproject.ai/>.

This chapter is the **authoring** chapter: how a pipeline step emits OpenLineage events, what facets matter for AI training-data work, and how to wire the emission so it is automatic rather than a documentation chore.

## The OpenLineage event in one page

An OpenLineage event is a JSON object:

```json
{
  "eventType": "START",
  "eventTime": "2026-10-15T09:00:00.123Z",
  "run": {
    "runId": "f0e9d8c7-b6a5-4948-b726-150493827160",
    "facets": {}
  },
  "job": {
    "namespace": "aicg-datapipeline-sft-chat",
    "name": "dedup.minhash",
    "facets": {}
  },
  "inputs": [
    {
      "namespace": "s3://example-lakefs/datasets",
      "name": "sft-chat-owned-ugc/prepared@a1b2c3d4",
      "facets": {}
    }
  ],
  "outputs": [
    {
      "namespace": "s3://example-lakefs/datasets",
      "name": "sft-chat-owned-ugc/deduped@c3d4e5f6",
      "facets": {}
    }
  ],
  "producer": "https://github.com/example/ai-data-pipeline/tree/f0e9d8c",
  "schemaURL": "https://openlineage.io/spec/2-0-0/OpenLineage.json"
}
```

Four event types:

- `START` — the run has started; inputs are declared but outputs may not yet exist.
- `RUNNING` — a heartbeat (rarely used; `START` → `COMPLETE` is the common pattern).
- `COMPLETE` — the run succeeded; outputs are produced.
- `FAIL` / `ABORT` — the run did not succeed; outputs may or may not exist.

The three identities:

- **`job`** — a logical operation. `(namespace, name)` is the key; `aicg-datapipeline-sft-chat.dedup.minhash` is a job that may run many times.
- **`run`** — one execution of a job. The `runId` is a UUID.
- **`dataset`** — a logical input or output. `(namespace, name)` is the key; for a lakehouse table, `(s3://warehouse, aicg.sft_chat_owned_ugc_2026_q3)`; for a lakeFS object, `(s3://example-lakefs/datasets, sft-chat-owned-ugc/deduped)`.

The graph is built by tying events together: `run.parent.runId` pins the parent run (an Airflow DAG spawning a task; a Dagster job spawning an op), and inputs / outputs tie runs to datasets.

## Facets — the extension mechanism that matters

Facets are the piece that makes OpenLineage useful for AI training-data work. A facet is a typed JSON blob attached to a run, a job, or a dataset. The specification includes standard facets; the `custom-facet` mechanism lets you define your own.

Standard facets you will use:

| Facet | Attaches to | What it carries |
|---|---|---|
| `sourceCode` | job | the source code of the transform (e.g., the SQL query, a Python snippet). |
| `sourceCodeLocation` | job | repo URL + Git SHA + file path. |
| `schema` | dataset | the column schema. |
| `dataSource` | dataset | the physical data source (JDBC URL, object-store URL). |
| `dataQualityAssertions` | dataset | Great Expectations / Soda / Deequ results. |
| `columnLineage` | output dataset | per-output-column lineage back to input columns. |
| `storage` | dataset | file format, storage layer. |
| `symlinks` | dataset | alternate identifiers (e.g., Iceberg table name ↔ S3 path). |
| `ownership` | dataset / job | human owner or team. |
| `errorMessage` | run | the error that caused `FAIL`. |

The custom facets that pay off for AI training-data work:

- **`aicg.licencePolicy`** (dataset) — the licence policy version pin applied, the allowlist / denylist decisions, the per-source licence tags.
- **`aicg.contaminationFirewall`** (dataset) — the firewall version pin, the per-benchmark removal counts.
- **`aicg.piiScrub`** (dataset) — the recogniser set pin, the per-entity removal counts.
- **`aicg.datasheet`** (dataset) — a pointer to the Datasheet URL (and its hash).
- **`aicg.snapshotIdentity`** (dataset) — the full snapshot-identity record from Chapter 01.

The custom-facet namespace is yours — publish the JSON schema internally so other emitters and consumers agree on shape.

## Emitting events — the four surfaces

### 1. Airflow

OpenLineage has a first-party Airflow provider. Install `apache-airflow-providers-openlineage` and configure the transport:

```ini
# airflow.cfg
[openlineage]
transport = {"type": "http", "url": "https://marquez.example.com:5000/api/v1/lineage"}
namespace = aicg-datapipeline
```

Every Airflow task emits `START` and `COMPLETE` (or `FAIL`) events automatically. The provider extracts inputs/outputs from supported operators (`SQLOperator`, `BashOperator`, `PythonOperator` with hints, `KubernetesPodOperator`, `SparkSubmitOperator`, etc.). For custom operators, you add an *extractor*:

```python
from openlineage.airflow.extractors.base import BaseExtractor, TaskMetadata
from openlineage.client.facet import SchemaDatasetFacet, SchemaField
from openlineage.client.run import Dataset

class MinHashDedupExtractor(BaseExtractor):
    @classmethod
    def get_operator_classnames(cls) -> list[str]:
        return ["MinHashDedupOperator"]

    def extract(self) -> TaskMetadata:
        op = self.operator
        inputs = [Dataset(namespace="s3://example-lakefs/datasets", name=f"{op.slice}/prepared")]
        outputs = [Dataset(namespace="s3://example-lakefs/datasets", name=f"{op.slice}/deduped")]
        return TaskMetadata(
            name=f"{op.dag_id}.{op.task_id}",
            inputs=inputs,
            outputs=outputs,
            run_facets={},
            job_facets={},
        )
```

Register the extractor via the `openlineage_extractors` entry point.

### 2. Dagster

Dagster has first-party OpenLineage emission. In `dagster.yaml`:

```yaml
run_coordinator:
  module: dagster_openlineage
  class: OpenLineageRunCoordinator
  config:
    url: https://marquez.example.com:5000
    namespace: aicg-datapipeline
```

Dagster's `@asset` and `@op` decorators emit events automatically; `MaterializeResult.metadata` is translated into dataset facets. <!-- needs-research: confirm the current Dagster + OpenLineage integration package name and configuration; both have moved. -->

### 3. Spark

OpenLineage's Spark listener (`io.openlineage:openlineage-spark`) hooks into Spark's `QueryExecutionListener`. Add the JAR and configuration:

```python
spark = (
    SparkSession.builder
    .config("spark.jars.packages", "io.openlineage:openlineage-spark_2.12:1.14.0")
    .config("spark.extraListeners", "io.openlineage.spark.agent.OpenLineageSparkListener")
    .config("spark.openlineage.transport.type", "http")
    .config("spark.openlineage.transport.url", "https://marquez.example.com:5000")
    .config("spark.openlineage.namespace", "aicg-datapipeline")
    .getOrCreate()
)
```

Every Spark SQL / DataFrame action emits events with dataset facets pointing at the Delta / Iceberg tables involved. Column-level lineage is captured automatically where the plan permits.

<!-- needs-research: cite the current openlineage-spark coordinates + Spark version matrix at read time; the matrix evolves per quarterly release. -->

### 4. Python pipelines (explicit emission)

For pipelines that are not in Airflow / Dagster / Spark — a CLI-driven `python -m aicg.pipeline` or a Pachyderm-wrapped transform — emit events explicitly from code:

```python
from openlineage.client import OpenLineageClient, set_producer
from openlineage.client.run import Dataset, Job, Run, RunEvent, RunState
from openlineage.client.facet import (
    SourceCodeLocationJobFacet,
    SchemaDatasetFacet,
    SchemaField,
)
from datetime import datetime, timezone
from uuid import uuid4

set_producer("https://github.com/example/ai-data-pipeline/tree/f0e9d8c")
client = OpenLineageClient.from_environment()  # reads OPENLINEAGE_URL etc.

run_id = str(uuid4())
job = Job(namespace="aicg-datapipeline-sft-chat", name="dedup.minhash")
inputs = [Dataset(namespace="s3://example-lakefs/datasets", name="sft-chat-owned-ugc/prepared")]
outputs = [Dataset(namespace="s3://example-lakefs/datasets", name="sft-chat-owned-ugc/deduped")]

client.emit(RunEvent(
    eventType=RunState.START,
    eventTime=datetime.now(timezone.utc).isoformat(),
    run=Run(runId=run_id),
    job=job,
    producer="https://github.com/example/ai-data-pipeline/tree/f0e9d8c",
    inputs=inputs,
    outputs=outputs,
))

# ... run the pipeline ...

output_facets = {
    "schema": SchemaDatasetFacet(fields=[
        SchemaField(name="instruction", type="string"),
        SchemaField(name="response", type="string"),
    ]),
    # the custom AICG facet
    "aicg.snapshotIdentity": _build_snapshot_identity_facet(snapshot_id),
}

client.emit(RunEvent(
    eventType=RunState.COMPLETE,
    eventTime=datetime.now(timezone.utc).isoformat(),
    run=Run(runId=run_id),
    job=job,
    producer="https://github.com/example/ai-data-pipeline/tree/f0e9d8c",
    inputs=inputs,
    outputs=[Dataset(
        namespace="s3://example-lakefs/datasets",
        name="sft-chat-owned-ugc/deduped",
        facets=output_facets,
    )],
))
```

Wrap this in a decorator or context manager so every pipeline step gets `START` + `COMPLETE` without per-step boilerplate:

```python
@emit_lineage(
    namespace="aicg-datapipeline-sft-chat",
    name="firewall",
    inputs_fn=lambda: [Dataset("s3://example-lakefs/datasets", "sft-chat-owned-ugc/deduped")],
    outputs_fn=lambda: [Dataset("s3://example-lakefs/datasets", "sft-chat-owned-ugc/firewalled")],
)
def run_firewall():
    ...
```

The design rule: **lineage emission is a side effect of running the pipeline, not a documentation task**. If it requires humans to update, it will drift.

## Marquez — the reference server

Marquez is the LF AI & Data reference implementation of an OpenLineage consumer. It runs as a web service; it ingests events on `POST /api/v1/lineage` and exposes:

- A **REST API** for querying jobs, runs, datasets, and lineage edges.
- A **GraphQL API** for structured graph queries.
- A **web UI** for browsing the graph.

Deployment is a Docker Compose for development and a Helm chart for production. The storage backend is PostgreSQL.

### Querying lineage

Find the lineage graph for a dataset (`GET /api/v1/lineage?nodeId=dataset:namespace:name&depth=3`):

```bash
curl -s https://marquez.example.com:5000/api/v1/lineage \
  --data-urlencode 'nodeId=dataset:s3://example-lakefs/datasets:sft-chat-owned-ugc/scrubbed' \
  --data-urlencode 'depth=5' \
  | jq .
```

The response is a directed graph of nodes (jobs and datasets) and edges. Consumers plot it as the upstream / downstream view.

Find the most recent run of a job (`GET /api/v1/namespaces/{ns}/jobs/{name}/runs`):

```bash
curl -s https://marquez.example.com:5000/api/v1/namespaces/aicg-datapipeline-sft-chat/jobs/dedup.minhash/runs \
  | jq '.runs[0] | {runId, state, startedAt, endedAt, facets}'
```

Marquez is deliberately simple — it is a reference server, not a feature-rich catalog. For a production AI data program it is enough to run the emission layer and prove the graph is coherent; DataHub or Unity Catalog (Chapter 07) extend it with search, documentation, and column-level lineage over large corpuses.

## Column-level lineage

OpenLineage's `columnLineage` facet captures, for each column in an output dataset, the list of input (dataset, column) pairs it was derived from, plus the transformation type (`IDENTITY`, `AGGREGATION`, `TRANSFORM`, etc.). The Spark listener produces this automatically for most SQL plans; the Airflow SQL extractor produces it for `SQLOperator` queries with parseable SQL.

For AI training-data engineering, column-level lineage is especially useful for:

- Auditing which source columns contributed to a derived preference-pair's `chosen` field.
- Identifying which filter column in the quality report invalidates which training rows.
- Proving to a reviewer that the PII-scrubbed `response` column is downstream of exactly one upstream `response` column (not a join or a leak).

Treat column-level lineage as a *bonus* that comes for free from the Spark listener; do not hand-author it unless a specific slice needs it.

## Event-emission operational concerns

- **Transport.** HTTP POST to the consumer is the default. For production, prefer a *buffered* transport (Kafka, SQS) so a Marquez outage does not take down the pipeline. OpenLineage supports a Kafka transport natively; a file transport is a reasonable fallback for local development.
- **Namespaces.** Use one per data domain (`aicg-datapipeline-sft-chat`, `aicg-datapipeline-pretrain`). Do *not* use a single global namespace; the graph becomes unreadable.
- **Dataset naming.** Follow the OpenLineage naming conventions. For object-storage: `(namespace=s3://bucket, name=path/to/object)`; for Hive-style tables: `(namespace=<catalog>://<cluster>, name=<db>.<table>)`. The specification pins per-system conventions; adhere to them so downstream consumers can disambiguate. <!-- needs-research: cite the current OpenLineage dataset-naming spec URL. -->
- **`parent.runId`.** Every nested run should set `run.facets.parent.runId` to the enclosing run. An Airflow DAG run spawns task runs; a Spark SQL statement spawns Delta commit runs. The parent linkage is what makes the graph navigable.
- **Backfills.** Events for historical runs can be emitted after the fact with a correct `eventTime`. Marquez / DataHub consume them as a backfill. Useful when adding lineage to a pre-existing pipeline.

## The authoring contract — what every pipeline step must emit

For the mod-109 snapshot package to be complete, every pipeline step that produces or transforms a snapshot must emit:

- `START` event with inputs declared (including policy input datasets, not just data input datasets).
- `COMPLETE` event with outputs, schema facet, source-code-location facet, and the custom `aicg.snapshotIdentity` facet populated with Chapter 01's identity record.
- `FAIL` event on failure, with the `errorMessage` facet populated.

The policy-input discipline is important: a step that applies the mod-106 firewall has the firewall config file as a *lineage input*, not just a configuration parameter. Pre-dedup data + firewall config → post-firewall data is a three-node subgraph. If the firewall config changes, the lineage graph reflects it; if a reviewer asks "which firewall version produced this snapshot?" they walk from the output dataset back to the config dataset in one hop.

## Peer coordination — who owns what

- **You (AI Data Engineer)** author the per-step OpenLineage emission inside your pipeline code (Python, Spark, Dagster ops, Airflow task extractors).
- **`ai-infra-mlops` (peer, L25)** owns the Marquez / DataHub / Unity Catalog deployment itself — the Helm charts, the PostgreSQL backing store, the HA posture, the transport reliability.
- **`training-pipeline-engineer` (next-up, L35)** consumes the lineage graph to trace training-run inputs back through the snapshot package.
- **`ai-evaluation-engineer` (next-up, L35)** consumes the lineage graph for regulator-facing packages — "show me every snapshot that fed any model deployed in production in the last year."

The authoring contract (events + facets) is yours; the consuming surface (API, UI, catalog features) is Chapter 07.

## Operating smells and anti-patterns

- **"We emit events but no one queries them."** A lineage graph with no consumers is lineage-theatre. Pair Chapter 06 with Chapter 07 — stand up at least Marquez and prove at least one real impact-analysis query.
- **"We emit a single event per DAG, not per step."** Coarse-grained emission loses column-level lineage and makes impact analysis useless. Emit per op / per Spark statement / per step.
- **"The pipeline code is not the source of truth for lineage."** If you edit a Confluence page after the pipeline changes, you are not doing lineage; you are doing documentation that drifts. Lineage must be derived from the pipeline, not described alongside it.
- **"We forgot to emit for the labelling-vendor step."** A step that produces or transforms data is a lineage step, even if it is operated by a vendor or runs in a different team's system. The vendor sidecar can emit OpenLineage too; the integration is the contract.

## Summary

- **OpenLineage** is the open, Linux-Foundation-stewarded protocol for lineage events. **Marquez** is the reference server; **DataHub**, **Unity Catalog**, and others are richer consumers (Chapter 07).
- Events carry `job`, `run`, `inputs`, `outputs`, and typed **facets**. The custom `aicg.snapshotIdentity` facet carries Chapter 01's identity record and is the mod-109 extension point.
- Emission surfaces: **Airflow** (provider), **Dagster** (integration), **Spark** (listener), and **explicit Python** emission for everything else. Wrap explicit emission in a decorator so it is a side effect, not a boilerplate burden.
- **Policies are lineage inputs**, not configuration parameters. The firewall config file is a *dataset* on the graph; a policy change is a graph edge.
- Marquez is a reference consumer — simple, PostgreSQL-backed, REST + GraphQL + UI. Production-grade lineage queries (search, column-level views, impact analysis) are Chapter 07.
- Treat lineage as a **side effect of running the pipeline**, not a documentation task. If it needs humans to update, it will drift.

Chapter 07 picks up the consuming side — DataHub and Databricks Unity Catalog — and the one operational question lineage must answer: *before* a production dataset change, which models, dashboards, and eval runs are downstream?
