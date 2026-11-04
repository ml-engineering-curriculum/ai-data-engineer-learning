# Chapter 05 — Delta Lake, Apache Iceberg, and Nessie: the Lakehouse Substrate

## Why this chapter exists

DVC, lakeFS, and Pachyderm (Chapters 02–04) treat the dataset as a *filesystem tree* — the atom of change is a file. For many AI training-data workloads the correct atom of change is a *row*: a new chat log event, a corrected label, a dedup decision, a PII-scrub update. The lakehouse substrate — Delta Lake (Linux Foundation / Databricks) and Apache Iceberg (Apache Software Foundation), with Nessie (Project Nessie, Dremio) providing Git-like branch semantics on top of Iceberg — is purpose-built for this shape.

The lakehouse substrates give you:

- **ACID transactions** on table-shaped data at TB–PB scale.
- **Snapshot isolation + time travel** — every write produces a new snapshot; readers pin to a snapshot ID for deterministic reads.
- **Schema evolution** — columns added, renamed, or dropped without rewriting history.
- **Partition evolution** — Iceberg can repartition without rewriting.
- **Branch / merge** semantics — natively via Iceberg table branches, or via Nessie as a Git-like catalog.

Primary sources: <https://delta.io/>, <https://iceberg.apache.org/>, <https://projectnessie.org/>.

## The lakehouse model in one page

A **lakehouse table** is:

- A set of **data files** on object storage (Parquet, or ORC / Avro in Iceberg). The actual bytes.
- A **transaction log / metadata layer** that lists which data files belong to which table version. The *atomic* part.
- A **catalog** that resolves table names to metadata-layer paths. In Delta, a Hive-compatible metastore or Unity Catalog; in Iceberg, a REST catalog, Nessie, Glue, Snowflake, or JDBC catalog.

Writes happen as:

1. The writer stages new / updated data files.
2. The writer appends a new log entry (Delta) or new metadata.json (Iceberg) describing the new snapshot.
3. The log append is **atomic** — two concurrent writers race on the log, one wins, the other retries (optimistic concurrency).
4. Readers see the new snapshot once the log is committed.

The practical consequence: multiple writers can append to the same table without stomping on each other, and readers always see a consistent snapshot. This is the property DVC, lakeFS, and Pachyderm do not natively provide at the row level.

## Delta Lake

**Primary source:** <https://docs.delta.io/>. The specification at <https://github.com/delta-io/delta/tree/master/PROTOCOL.md> is the authoritative reference.

### The Delta model

A Delta table is a prefix on object storage that contains:

- `_delta_log/` — the transaction log, as a sequence of `00000000000000000000.json`, `00000000000000000001.json`, … files, each describing a *commit*.
- `part-*.parquet` — the data files, in the table's partition layout.

Each log entry records *add* actions (new Parquet files), *remove* actions (invalidated files), schema changes, and arbitrary metadata.

### Reading and writing with PySpark

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("sft-chat-prepare")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# write
(df
 .write
 .format("delta")
 .mode("append")
 .partitionBy("source_date")
 .save("s3://example-datasets/sft-chat-owned-ugc-2026-q3/"))

# read current state
current = spark.read.format("delta").load("s3://example-datasets/sft-chat-owned-ugc-2026-q3/")

# read a specific snapshot (time travel)
v1 = (spark.read
      .format("delta")
      .option("versionAsOf", 1)
      .load("s3://example-datasets/sft-chat-owned-ugc-2026-q3/"))

# read at a point in time
t1 = (spark.read
      .format("delta")
      .option("timestampAsOf", "2026-10-15T00:00:00Z")
      .load("s3://example-datasets/sft-chat-owned-ugc-2026-q3/"))
```

Via `delta-rs` (a Rust implementation with Python bindings, no JVM required):

```python
from deltalake import DeltaTable, write_deltalake

write_deltalake(
    "s3://example-datasets/sft-chat-owned-ugc-2026-q3/",
    df,                           # a pandas / pyarrow Table
    mode="append",
    partition_by=["source_date"],
)

dt = DeltaTable("s3://example-datasets/sft-chat-owned-ugc-2026-q3/")
print(dt.version())               # current version number
print(dt.history())               # commit history with timestamps + user metadata
v1 = dt.load_as_version(1)        # load a specific prior version
```

### Snapshot identity under Delta

Mapping Chapter 01's five snapshot-identity fields to Delta:

| Identity field | Delta primitive |
|---|---|
| Content-addressed hash | SHA-256 Merkle root over the data files listed in the snapshot. Delta's own commit hash is a Spark-side hash and not SHA-256 by default. |
| Substrate identifier | `delta://<catalog>.<schema>.<table>@v<version>` or an explicit `(table-path, version)` pair. |
| Pipeline version | Commit `userMetadata` (free-text JSON). Pin the Git SHA of the pipeline code here. |
| Policy versions | Commit `userMetadata` keys. Delta supports arbitrary user metadata per commit. |
| Signatures + attestations | External — `cosign sign-blob` over the SHA-256 Merkle root; Rekor reference in `userMetadata`. |

Writing user metadata with the commit:

```python
from deltalake import DeltaTable

dt = DeltaTable("s3://example-datasets/sft-chat-owned-ugc-2026-q3/")
dt.optimize.compact()  # a transaction that produces a new version
# attach user metadata on a write, via Spark:
#   .option("userMetadata", json.dumps({"pipeline_git_sha": "...", "firewall_version": "..."}))
```

### Operational notes for Delta

- **Checkpointing.** The log can grow large; Delta writes a Parquet checkpoint every N commits (default 10) that summarises prior state. Readers consult the latest checkpoint + subsequent log entries, not the entire log.
- **Vacuum.** `VACUUM` removes data files no longer referenced by the current log. **Do not** `VACUUM` with a retention shorter than the snapshot-identity lifetime — once vacuumed, prior versions cannot be time-travelled to. For audit-ready snapshots, keep retention at ≥ 30–90 days or exclude the audited snapshots from vacuum.
- **Z-ordering.** `OPTIMIZE ... ZORDER BY (col)` rewrites data files for better partition pruning on queries. A *new version* is produced; prior versions remain available until vacuumed.
- **Shallow and deep clones.** `CLONE` lets you copy a table efficiently; a shallow clone shares data files, a deep clone copies them. Useful for branching within Delta without a Nessie-like catalog (Delta's own multi-table branching story is weaker than Iceberg's).
- **Unity Catalog.** Databricks Unity Catalog is the dominant catalog for Delta in Databricks environments and is the lineage consumer in Chapter 07.

## Apache Iceberg

**Primary source:** <https://iceberg.apache.org/docs/latest/>. Specification: <https://iceberg.apache.org/spec/>.

### The Iceberg model

Iceberg's layering is more principled than Delta's:

- **Catalog** — maps `(namespace, table)` to a current *metadata file* URL.
- **Metadata file** (`v1.metadata.json`, `v2.metadata.json`, …) — JSON describing the current snapshot, schema, partition spec, and snapshot history.
- **Manifest list** (`snap-<id>.avro`) — the list of manifests that make up a snapshot.
- **Manifest file** (`<id>.avro`) — a list of data files with per-file statistics (lower bounds, upper bounds, null counts) for partition and column pruning.
- **Data files** (`*.parquet`) — the bytes.

Every snapshot has a numeric `snapshot-id`. Snapshots are strictly additive: each write produces a new metadata.json; nothing is rewritten unless the writer explicitly compacts.

### Reading and writing with PyIceberg

```python
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import NestedField, StringType, TimestampType, IntegerType
from pyiceberg.partitioning import PartitionSpec, PartitionField
from pyiceberg.transforms import DayTransform

catalog = load_catalog("local", **{
    "uri": "http://iceberg-rest.example.com",
    "warehouse": "s3://example-iceberg-warehouse/",
})

schema = Schema(
    NestedField(1, "instruction_id", StringType(), required=True),
    NestedField(2, "instruction", StringType(), required=True),
    NestedField(3, "response", StringType(), required=True),
    NestedField(4, "seed_source_id", StringType(), required=False),
    NestedField(5, "generation_round", IntegerType(), required=True),
    NestedField(6, "generation_timestamp", TimestampType(), required=True),
)

partition_spec = PartitionSpec(
    PartitionField(source_id=6, field_id=1000, transform=DayTransform(), name="generation_day"),
)

catalog.create_table_if_not_exists(
    "aicg.sft_chat_owned_ugc_2026_q3",
    schema=schema,
    partition_spec=partition_spec,
)

table = catalog.load_table("aicg.sft_chat_owned_ugc_2026_q3")
table.append(arrow_table)                      # arrow_table is a pyarrow.Table

# time travel
snapshot_id = table.history()[0].snapshot_id
snapshot_scan = table.scan(snapshot_id=snapshot_id)

# or by timestamp
import datetime as dt
snapshot_at_t = table.scan(as_of=dt.datetime(2026, 10, 15, 0, 0, 0))
```

### Snapshot identity under Iceberg

Mapping Chapter 01's five snapshot-identity fields to Iceberg:

| Identity field | Iceberg primitive |
|---|---|
| Content-addressed hash | SHA-256 Merkle root over the data files listed in the snapshot's manifest list. Iceberg metadata.json is itself content-addressed on a stable path. |
| Substrate identifier | `iceberg://<catalog>.<namespace>.<table>@<snapshot-id>` or `(catalog, namespace, table, snapshot-id)`. |
| Pipeline version | Snapshot summary properties (`summary.pipeline_git_sha`). Iceberg supports per-snapshot summary KV pairs. |
| Policy versions | Snapshot summary properties. |
| Signatures + attestations | External — `cosign sign-blob` over the SHA-256 Merkle root; Rekor reference in snapshot summary. |

Writing snapshot summary properties (via PyIceberg's transaction API):

```python
with table.transaction() as tx:
    tx.append(arrow_table)
    tx.set_summary_properties({
        "pipeline_git_sha": "f0e9d8c7b6a5948372615049382716054938271",
        "firewall_version": "firewall-v2026-07-15",
        "pii_recognisers": "presidio-recognisers-v2026-06-30",
        "cosign_rekor_entry": "12345678",
    })
```

### Iceberg vs. Delta — the practical distinctions

Both are mature; both have well-documented data-model differences. For AI training-data engineering, the decisions that actually matter:

- **Catalog ecosystem.** Delta's deepest integration is with Databricks Unity Catalog; Iceberg has a broader catalog ecosystem (REST, Glue, Nessie, Snowflake, Hive, JDBC, Polaris). If your organisation is Databricks-first, Delta is the default; if you want to avoid a specific-vendor catalog, Iceberg is the default.
- **Branch / merge semantics.** Iceberg has first-class *table branches* (`main`, `feature-x`) and *tags* (`v1.0`, `pre-dedup`) in the spec. Delta's native branching is weaker; clones are the workaround. For data-engineering workflows that want Git-like semantics, Iceberg + Nessie (below) is the stronger fit.
- **Partition evolution.** Iceberg supports partition evolution — changing the partition scheme without rewriting data. Delta requires a rewrite.
- **File format.** Delta is Parquet-only. Iceberg supports Parquet, ORC, and Avro (though Parquet is dominant for AI workloads).
- **Query engine coverage.** Both are read by Spark, Trino, Dremio, Flink, and Snowflake. Delta has deeper Spark coupling; Iceberg has deeper Trino / Flink coupling. For an AI data engineer the practical difference is small.

A reasonable organisational posture: use Delta when the organisation is on Databricks and Unity Catalog; use Iceberg + Nessie otherwise. Either is a defensible lakehouse substrate.

## Nessie — Git-like semantics on top of Iceberg

**Primary source:** <https://projectnessie.org/>.

Nessie (Dremio, Apache 2.0) is a *catalog* for Iceberg (and, in recent releases, Delta) that adds Git-like semantics: **branches**, **tags**, **merge**, **revert**. The catalog tracks the history of table metadata pointers across branches; a branch of a table is O(1) and copy-on-write.

### The Nessie model

- **Branches** (`main`, `dev-q3-2026`, `experiment-firewall-v2`) — mutable named references to table-metadata trees.
- **Tags** (`v1.2.0`, `release-2026-10-15`) — immutable named references.
- **References** — the generic term for either.
- **Content** — a (table name → metadata-file URL) pair, per reference.

Reads are *reference-scoped*: a query specifies `aicg.sft_chat_owned_ugc_2026_q3@dev-q3-2026`, and Nessie resolves the current metadata file for that table on that branch.

### Working with Nessie

```python
from pynessie import init
from pynessie.model import ContentKey

nessie = init()

# list refs
for ref in nessie.list_references():
    print(ref.name, ref.hash_)

# create a dev branch from main
nessie.create_branch("dev-q3-2026", source_ref="main")

# query a table on the branch
# (in Spark SQL: `USE REFERENCE dev-q3-2026 IN nessie;`
#  in Trino: `USE nessie.aicg_prod.dev_q3_2026;`)

# merge back to main
nessie.merge("main", from_ref="dev-q3-2026")

# tag a release
nessie.create_tag("v1.2.0", source_ref="main")
```

### Snapshot identity under Nessie

For an Iceberg-on-Nessie snapshot, identity is `(catalog, branch_or_tag, table, iceberg_snapshot_id)`. A pinned production snapshot is typically referenced by *tag*:

```
nessie://aicg_prod/v1.2.0/aicg.sft_chat_owned_ugc_2026_q3
```

The tag is stable; the Iceberg snapshot ID beneath it is the resolve-to identity.

### Nessie + lakeFS — pick one

Both Nessie and lakeFS expose branch / merge / rollback semantics on a data lake. They differ in substrate scope:

- **lakeFS** is substrate-agnostic — it brands branches onto S3 / GCS / Azure Blob prefixes, and anything S3-compatible can read/write through it. It is file-shaped, not table-shaped.
- **Nessie** is lakehouse-table-native — branches apply to Iceberg / Delta *tables*, with row-level ACID semantics inherited from the table format.

For AI training-data pipelines where the atom of change is a *row*, Nessie fits better; for pipelines where the atom of change is a *file* (an ingested document, a labelled image, a WAV clip), lakeFS fits better. Teams that have both shapes often run both substrates — lakeFS over the file-shaped data, Nessie over the derived table-shaped data.

## Consuming lakehouse snapshots at training time

The cluster-side ingest layer (Chapter 11) consumes lakehouse snapshots in one of three shapes:

- **Direct read from Spark / Ray Data** — the training job reads the table at a pinned version and materialises Arrow batches on the fly. Good for mid-sized tables (sub-TB) with Spark or Ray already deployed.
- **Export to Parquet shards** — a periodic job exports the pinned snapshot to a shard layout the training job reads natively (HF Datasets + Parquet + Arrow). Good for the dominant case where the training job reads Parquet directly.
- **Export to WebDataset or StreamingDataset** — for sequence-model training where tar-shard ordering matters, export the snapshot to the ingest-side layout in Chapter 11.

In all three cases the handoff carries the *snapshot identity* (Chapter 01) — even if the training job ultimately reads tar-shards, the shards are derived from an Iceberg snapshot ID, and that ID is what the signature attests.

## Lineage integration — lakehouse + OpenLineage

Both Delta and Iceberg have OpenLineage integration via Spark:

- **Spark + OpenLineage.** The `openlineage-spark` listener emits `START` and `COMPLETE` events for every Spark SQL / DataFrame operation, with dataset facets pointing at the Delta / Iceberg tables involved. Chapter 06.
- **Databricks Unity Catalog.** Unity Catalog maintains its own lineage graph natively; it also emits OpenLineage events. Chapter 07.
- **Nessie commits.** Each Nessie commit can emit an OpenLineage event via the Nessie events API or a webhook.

The payoff is a lineage graph that follows the data through Spark jobs, Delta / Iceberg tables, and Nessie branches without per-stage custom code.

<!-- needs-research: confirm the current OpenLineage adapter surface for Nessie specifically; the integration has moved between a native event listener and a webhook pattern. -->

## Peer uses in the track — where this substrate lands

- A derived *training-row* table — one row per (prompt, response) pair, with provenance columns (source dataset ID, upstream commit, filter verdicts) — is a canonical lakehouse fit.
- A *labelling-vendor-events* table — one row per annotator decision with calibration and QA columns — is a canonical lakehouse fit.
- A *feature-store-backing* table — point-in-time correct features for mod-108's synthetic-generation seeds — is a canonical lakehouse fit.
- A raw crawl of HTML files is **not** a canonical lakehouse fit — keep it on lakeFS or object storage directly; convert to a lakehouse table only once the data is row-shaped.

## Summary

- The lakehouse substrates — **Delta Lake** (Linux Foundation) and **Apache Iceberg** (ASF) — provide ACID, time-travel, and schema-evolution semantics on table-shaped data at TB–PB scale. **Nessie** adds Git-like branch / tag / merge on top of Iceberg (and Delta).
- The practical substrate decision: Databricks + Unity Catalog → Delta; vendor-neutral or Trino / Flink-native → Iceberg + Nessie.
- Snapshot identity maps to `(catalog, table, snapshot-id)` or `(catalog, branch_or_tag, table, snapshot-id)` with Nessie. Policy pins and pipeline Git SHAs go in snapshot summary properties (Iceberg) or commit user metadata (Delta).
- **Do not vacuum / expire snapshots** inside the audit-retention window. Vacuum destroys the ability to time-travel; that is the audit evidence the module relies on.
- Lakehouse substrates are **row-shaped**; file-shaped data belongs on lakeFS or DVC. Real pipelines run both — file-shaped ingestion on lakeFS, derived row-shaped training tables on Iceberg + Nessie.
- OpenLineage integrations exist for Spark + Delta / Iceberg and for Unity Catalog. Chapters 06 and 07 pin these.

Chapter 06 picks up the lineage authoring layer — OpenLineage events emitted from pipeline code, consumed by the Marquez reference server — which composes with every substrate from Chapters 02–05.
