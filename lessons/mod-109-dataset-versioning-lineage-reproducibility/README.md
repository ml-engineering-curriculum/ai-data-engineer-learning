# mod-109 — Dataset Versioning, Lineage, and Reproducibility

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 14 hours

## What this module is about

Every upstream module in this track produces an *artifact*. mod-102 produces per-source licence records, Datasheets, ML-BOM entries, and C2PA indexes. mod-103 produces a curated crawl slice. mod-104 produces a quality report. mod-105 produces a dedup report. mod-106 produces a contamination firewall report. mod-107 produces a labelling program and a preference-pair corpus. mod-108 produces a synthetic-generation package. All of that evidence is defensible only if you can answer four questions at any point in the next five years:

1. **Which exact bytes made it into the training run that produced model `M`, build `B`?**
2. **Which filters, dedup passes, firewalls, scrubbers, and labelling decisions ran on those bytes?**
3. **Who signed off, with which key, at which commit of the pipeline code, against which policy version?**
4. **Can I re-produce that corpus bit-for-bit — or at least reconstruct it from first-party sources plus the pipeline — a year from now?**

mod-109 is the module that makes those questions answerable. The deliverable at the end of the module is a **dataset snapshot package** for a real slice:

1. **Versioned snapshot** — on DVC, lakeFS, Pachyderm, or a lakehouse table (Delta / Iceberg + Nessie), pinned to an immutable commit / snapshot / tag.
2. **Lineage event bundle** — OpenLineage events for every pipeline step that touched the snapshot, consumable from Marquez, DataHub, or Unity Catalog.
3. **Signature + attestation set** — Sigstore / `cosign` signatures over snapshot hashes; SLSA v1.0 provenance attestations with `builder.id`, `buildType`, materials, and byproducts.
4. **Supply-chain record pair** — CycloneDX ML-BOM entry and SPDX 3.0 Dataset-profile element for the snapshot, extending the records seeded in mod-102.
5. **Findable-metadata record set** — MLCommons Croissant JSON-LD, Hugging Face Dataset Card YAML, and a FAIR-principles self-assessment.
6. **Training-ingest handoff contract** — a signed artifact that tells `training-pipeline-engineer` (next-up, L35) which cluster-side layout the snapshot is published in (WebDataset tar-shards, MosaicML StreamingDataset, HF Datasets Parquet + Arrow, or Ray Data streaming), what the shard inventory is, and what the retry / verification behaviour on their side is.

By the end you will be able to hand a peer or regulator a *pointer* — a commit SHA, a snapshot ID, a `cosign` signature URL — and have the entire chain of evidence materialise from it. That is the difference between a reproducible pipeline and a described one.

## Why versioning is an engineering discipline, not a backup chore

Four findings pin the framing. If you treat this module as "git-for-data plus a signed tarball" you will build a system that passes casual audit and fails the first real one.

- **Papernot and the data-provenance thread** — the broader academic argument that AI systems have a *supply chain*, and that reasoning about model behaviour requires reasoning about the provenance of every byte of training data. This is the frame the US Executive Order 14028 era normalised for software; the AI Executive Orders and the EU AI Act Article 53 + Annex XI have begun to normalise it for AI. Chapters 08, 09.
- **The reproducibility crisis in ML** — Pineau et al. 2021 (*Improving Reproducibility in Machine Learning Research*, ACM) and the NeurIPS / MLSys reproducibility checklists. Dataset-side irreproducibility is one of the largest named causes. "The dataset we used is on this S3 URL" is not evidence, because objects mutate and permissions change. A signed, hash-pinned snapshot is. Chapters 01, 02, 08. <!-- needs-research: confirm the Pineau 2021 citation and page numbers in the Communications of the ACM. -->
- **OpenLineage as the dominant open protocol** — the Linux Foundation / LF AI & Data project that standardises the *shape* of a lineage event (job, run, inputs, outputs, facets). Vendor-specific lineage (DataHub, Unity Catalog, Collibra, Alation) composes with OpenLineage; a pipeline that emits OpenLineage events can be consumed by any of them. The industry has consolidated around this protocol in a way that the equivalent debates about data-catalog schemas have not. Chapters 06, 07.
- **SLSA and Sigstore as the floor** — SLSA (Supply-chain Levels for Software Artifacts, now published as a SLSA v1.0 specification by the OpenSSF) and Sigstore (`cosign`, Fulcio, Rekor) are the baseline supply-chain-security substrate downstream procurement will expect. "Unsigned" is becoming a disqualifier for datasets going into regulated-use-case training. Chapters 08, 09.

Chapter 01 is the defence of this framing. Do not skip it.

## Learning objectives

- Version non-trivial datasets with **DVC** (tracking vs. tracked storage, remotes, pipelines), **lakeFS** (branch / merge / rollback over object storage), and **Pachyderm** (versioned pipelines). Know when each fits — small-team / filesystem-shaped, object-store-native / data-engineering-shaped, pipeline-heavy. Chapters 02, 03, 04.
- Version large tables with **Delta Lake** and **Apache Iceberg** (with **Nessie** / Project Nessie for Git-like semantics) when the training-data substrate is a lakehouse — not a filesystem. Chapter 05.
- Track dataset lineage with **OpenLineage** (open protocol), **Marquez** (LF reference server), **DataHub**, and **Databricks Unity Catalog** — the authoring side (emitting lineage events from pipeline code) and the consuming side (querying lineage before a production dataset change). Chapters 06, 07.
- Sign dataset snapshots with **Sigstore** (`cosign`, Fulcio, Rekor) and emit **SLSA v1.0** level-1-and-up provenance attestations; produce **CycloneDX ML-BOM** and **SPDX 3.0 Dataset / AI-profile** records per snapshot (continuing the mod-102 provenance thread into the versioning substrate). Chapters 08, 09.
- Publish **Croissant** (MLCommons) metadata, **Hugging Face Dataset Card** YAML, and **FAIR-principles-aligned** metadata so the dataset is findable, accessible, interoperable, and reusable. Chapter 10.
- Hand off cluster-side storage layouts — **WebDataset** tar-shards, **MosaicML StreamingDataset** format, **Hugging Face Datasets** Parquet + Arrow, **Ray Data** streaming — to [`training-pipeline-engineer`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) (peer / next-up, L35) with a documented hand-off contract. This module covers the authoring layer; that peer covers the cluster-ingest layer. Chapter 11.

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-the-versioning-substrate-and-snapshot-package.md) | The versioning substrate and the snapshot package — framing and the four claims | Framing (all objectives) |
| [02](02-dvc-small-team-filesystem-versioning.md) | DVC — small-team, filesystem-shaped versioning | Objective 1 (DVC) |
| [03](03-lakefs-branch-merge-rollback-on-object-storage.md) | lakeFS — branch, merge, rollback on object storage | Objective 1 (lakeFS) |
| [04](04-pachyderm-versioned-pipelines.md) | Pachyderm — versioned pipelines and datum parallelism | Objective 1 (Pachyderm) |
| [05](05-delta-lake-iceberg-nessie-lakehouse.md) | Delta Lake, Apache Iceberg, and Nessie — the lakehouse substrate | Objective 2 |
| [06](06-openlineage-and-marquez-authoring-lineage.md) | OpenLineage and Marquez — authoring the lineage graph | Objective 3 (authoring) |
| [07](07-datahub-and-unity-catalog-consuming-lineage.md) | DataHub and Unity Catalog — consuming lineage before a dataset change | Objective 3 (consuming) |
| [08](08-sigstore-and-slsa-signing-snapshots.md) | Sigstore and SLSA — signing snapshots and attesting provenance | Objective 4 (signing) |
| [09](09-ml-bom-and-spdx-ai-per-snapshot.md) | CycloneDX ML-BOM and SPDX AI — snapshot-level supply-chain records | Objective 4 (BOM / SPDX) |
| [10](10-croissant-huggingface-card-and-fair-metadata.md) | Croissant, Hugging Face Dataset Card, and FAIR-principles metadata | Objective 5 |
| [11](11-cluster-side-handoff-webdataset-streaming-parquet-ray.md) | Cluster-side handoff — WebDataset, StreamingDataset, HF Parquet + Arrow, Ray Data | Objective 6 |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-dvc-vs-lakefs-vs-pachyderm-on-one-dataset.md) | DVC vs. lakeFS vs. Pachyderm on one dataset | 2h |
| [02](exercises/exercise-02-delta-lake-or-iceberg-for-a-training-table.md) | Delta Lake or Iceberg for a training table | 2h |
| [03](exercises/exercise-03-openlineage-event-emission-from-a-pipeline.md) | OpenLineage event emission from a pipeline | 3h |
| [04](exercises/exercise-04-sigstore-and-slsa-attestations-for-dataset-snapshots.md) | Sigstore and SLSA attestations for dataset snapshots | 3h |
| [05](exercises/exercise-05-croissant-and-fair-metadata-authoring.md) | Croissant and FAIR metadata authoring | 2h |

Labs and quizzes are scaffolded under `labs/` and `quizzes/` and are authored in a subsequent content cycle. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) and internalised ISO/IEC 25012 *traceability* and *credibility* as formal requirements, not aspirations.
- You have completed [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md). This module extends the ML-BOM / SPDX / Datasheet / Croissant records seeded there into the versioning substrate; they are not re-derived here.
- You have completed at least [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) and [`mod-105`](../mod-105-deduplication-at-scale/README.md). The quality and dedup *reports* are what you will attach to snapshots; this module does not teach you to produce them.
- You are comfortable with **git** semantics (commits, branches, merges, tags, SHAs) and with **object storage** (S3 / GCS / Azure Blob, prefix layout, versioned buckets, lifecycle rules). All five of DVC, lakeFS, Pachyderm, Delta, and Iceberg borrow from one or the other.
- Python 3.11+ with: `dvc[s3]` or `dvc[gs]`, `lakefs-client`, `pachctl` CLI and `python-pachyderm`, `pyspark` + `delta-spark` or `delta-rs` (`deltalake`), `pyiceberg`, `pynessie`, `openlineage-python` (and the Airflow / Dagster / Spark integrations you care about), `datahub` CLI, `sigstore` + `cosign` CLI, `cyclonedx-bom` and `cyclonedx-python-lib`, `spdx-tools`, `mlcroissant`, `datasets` (Hugging Face), `webdataset`, `mosaicml-streaming`, `ray[data]`, `pyarrow`.
- A **running object-store bucket** you control (S3, GCS, Azure Blob, or MinIO / Garage for local). Exercises 01, 02, and 04 assume you can read and write it and set an object-lock / versioning policy on it.
- A **running OIDC identity** for keyless signing in Exercise 04. A free GitHub / Google / Microsoft account suffices; Fulcio will issue a short-lived certificate against it.

## How to use this module

Chapter 01 is the frame. The snapshot package and the four claims are the vocabulary you will carry for the rest of the module.

Chapters 02, 03, and 04 are the three filesystem-substrate choices. Do not read them as competitors; read them as a *decision matrix* over three dimensions — team shape, storage shape, pipeline shape. Exercise 01 forces that decision explicitly.

Chapter 05 is the lakehouse substrate. If your organisation's training data lives in a Databricks workspace, a Snowflake Iceberg catalog, or a self-managed Iceberg + Nessie stack, Chapter 05 is *the* chapter — Chapters 02–04 are context. Exercise 02 pairs with it.

Chapters 06 and 07 are the lineage chapters. OpenLineage (Chapter 06) is the authoring protocol; DataHub and Unity Catalog (Chapter 07) are the two most common catalog consumers. Exercise 03 pairs with Chapter 06.

Chapters 08 and 09 are the supply-chain chapters. Chapter 08 is the signing substrate (Sigstore, SLSA); Chapter 09 is the manifest shape (ML-BOM, SPDX). Exercise 04 pairs with both. These extend the mod-102 provenance thread one layer down — mod-102 signed the *source*; this module signs the *snapshot*.

Chapter 10 is the findability layer. Croissant and HF Dataset Card YAML are the two machine-readable formats a downstream *user* of your dataset will search by. FAIR is the self-assessment rubric. Exercise 05 pairs with it.

Chapter 11 is the handoff to `training-pipeline-engineer`. This is where *your* module ends and theirs begins. Treat the handoff contract as a load-bearing artifact — it is the one both sides sign.

## Downstream handoffs

- To [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md): the snapshot package carries the PII-scrub report emitted there. The snapshot does not *replace* PII controls — it carries the evidence that PII controls ran against *this exact* set of bytes.
- To [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md): monitoring runs against *versioned* datasets. The alert payload references a snapshot ID; drift is computed *between* snapshots; regression fixtures are pinned snapshots.
- To [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md): the EU AI Act Article 53 + Annex XI training-data summary references snapshot IDs, signatures, and lineage queries from this module. The dataset card references the Croissant record.
- To [`training-pipeline-engineer`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) (peer / next-up, L35): Chapter 11's handoff contract is the artifact they consume. Cluster-side ingestion throughput, tokenisation sharding, and I/O-pattern tuning are theirs; the shape of the shards and the inventory manifest are yours.
- To [`fine-tuning-engineer`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) (peer, L30): the SFT corpus they train on is a snapshot. The DPO / ORPO pairs from mod-108 are a snapshot. The hand-off contract is the snapshot package — not raw JSONL.
- To [`rag-engineer`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) (peer, L30): the retrieval corpus they index is a snapshot. The embeddings they compute are a *derived* snapshot that `dependsOn` the source corpus.
- To [`model-evaluation-engineer`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, L30): a benchmark is a snapshot. The decontamination firewall contract from mod-106 is evaluated against a *pinned* snapshot of the benchmark.
- To [`ai-evaluation-engineer`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (next-up, L35): the regulator-facing package they deliver carries snapshot signatures and lineage exports from this module.
- To [`ai-infra-mlops`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) (peer, L25): the Airflow / Dagster / Prefect integration of OpenLineage lives in their orchestrator platform; this module authors the per-step emission code. They own the transport; you own the payload.
