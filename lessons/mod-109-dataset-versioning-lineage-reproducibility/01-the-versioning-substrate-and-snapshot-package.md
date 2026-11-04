# Chapter 01 — The Versioning Substrate and the Snapshot Package

## Why this chapter exists

Every other module in this track produces *evidence*. mod-102 produces a licence record and a Datasheet. mod-104 produces a quality report. mod-105 produces a dedup report. mod-106 produces a firewall report. mod-107 produces an IAA report and a labelling-vendor audit log. mod-108 produces a generation manifest and a filter log. Every one of those records is only as useful as the *binding* between it and the bytes it describes.

The binding is the job of this module. Without it:

- A reviewer cannot tell whether the Datasheet describes the bytes the model actually trained on, or a prior revision.
- A monitoring alert that says "quality on slice `sft-chat-2026-q3` dropped 4 points" cannot tell you whether the *data* changed or the *metric* changed, because there is no snapshot ID on either side of the comparison.
- A regulator asking "which bytes caused the harm in the September incident?" receives a filesystem path that no longer resolves to what was there in September.
- A reproducibility attempt six months later gets a different corpus than the authors had, because `s3://datasets/train/` has been overwritten four times.

Every one of these failure modes is routinely encountered in production AI organisations. They are not reliability problems or engineering-hygiene problems — they are *evidence* problems. The pipeline is correct; the chain of custody is broken.

This chapter sets the framing and the deliverable that fixes that.

## The deliverable — the dataset snapshot package

The deliverable at the end of this module is a **dataset snapshot package** for a real slice. The package has six artifacts, each produced by a later chapter:

1. **Versioned snapshot** (Chapters 02, 03, 04, 05) — the bytes themselves, pinned to an immutable identifier: a DVC `.dvc` file + MD5, a lakeFS commit SHA, a Pachyderm commit ID, a Delta table version, an Iceberg snapshot ID. Whatever the substrate, the identifier is a *pointer* that resolves to the exact bytes.
2. **Lineage event bundle** (Chapters 06, 07) — OpenLineage events describing every job that produced or transformed the snapshot. Consumable from Marquez, DataHub, or Unity Catalog. The bundle is a graph: nodes are datasets and jobs, edges are *was-produced-by* / *was-consumed-by*.
3. **Signature + attestation set** (Chapter 08) — Sigstore `cosign` signatures over snapshot hashes; SLSA v1.0 provenance attestations with `builder.id`, `buildType`, `invocation`, `materials`, and `byproducts`. Signatures are either keyless (Fulcio-issued short-lived certs, Rekor-logged) or key-based (KMS-held long-lived keys).
4. **Supply-chain record pair** (Chapter 09) — CycloneDX ML-BOM entry extending the mod-102 source-level entry with snapshot-specific `properties`; SPDX 3.0 Dataset-profile element citing the snapshot's hash, size, availability, and the AI-profile element downstream models will `dependsOn`.
5. **Findable-metadata record set** (Chapter 10) — a Croissant JSON-LD description of the dataset's schema and distributions; a Hugging Face Dataset Card YAML header; a FAIR-principles self-assessment table.
6. **Training-ingest handoff contract** (Chapter 11) — a YAML / JSON document pointing `training-pipeline-engineer` at the cluster-side storage layout (WebDataset, StreamingDataset, HF Parquet + Arrow, Ray Data) with a shard inventory, a checksum manifest, and the retry / verification behaviour the ingest-side is expected to implement.

Every chapter adds one artifact. Chapter 01 is the defence of why the package exists.

## The four claims you must be able to defend

Pin these four. They are the citations the role is expected to carry for the rest of the module and for every peer and regulator conversation that follows.

### 1. The dataset is a supply chain, and the supply chain must be *signable*

**Framing:** SLSA v1.0 (OpenSSF, 2023). <https://slsa.dev/spec/v1.0/>. In-toto attestation framework. <https://in-toto.io/>. Sigstore (OpenSSF). <https://www.sigstore.dev/>.

The US Executive Order 14028 era normalised the idea that *software* has a supply chain and that the chain must be documented and signed. SLSA v1.0 formalised a maturity model (levels 1–4) describing how well the build process enforces provenance. Sigstore operationalised the signing step with keyless signing via OIDC-issued short-lived certificates (`cosign sign`), a public transparency log (`rekor`), and free-to-use certificate authorities (`fulcio`). In-toto provided the *envelope* format — a signed, verifiable statement about what a build consumed and produced.

AI systems are a special case of software. A dataset snapshot is a *build artifact* in the SLSA sense: it was produced by a build process (a pipeline) with inputs (source datasets, code, configuration) and outputs (the snapshot bytes plus logs). The same vocabulary applies. SLSA v1.0 level 1 means the build produced provenance; level 2 means the provenance is tamper-resistant; level 3 means the build platform is hardened; level 4 is reserved for a hermetic / two-party-reviewed build.

**Why it matters for your pipeline:** every claim you make about the dataset — "this is licensed for commercial use", "this was decontaminated against MMLU", "the PII scrub ran" — is worth exactly as much as the binding between the claim and the bytes. An unsigned snapshot with a side-by-side report is not evidence; it is correlation. Chapter 08 is the signing substrate.

### 2. Lineage is a graph query, not a diagram

**Framing:** OpenLineage specification. <https://openlineage.io/>. The LF AI & Data project's reference server Marquez. <https://marquezproject.ai/>.

Early data-lineage tooling was *descriptive*: a diagram drawn by a human and updated when someone remembered. The modern tooling is *programmatic*: pipeline steps emit lineage events as they run, a server aggregates them, and consumers query them via API or UI. OpenLineage is the industry-converged protocol for the emission format. Marquez is the reference server; DataHub, Unity Catalog, Collibra, and others consume OpenLineage events.

The key shift is that lineage is a *side effect of running the pipeline*, not a documentation task. If the pipeline's Airflow DAG or Dagster job emits OpenLineage events for every step, the graph is built automatically. If it does not, no amount of documentation covers the gap — because the documentation is written by humans and the pipeline changes faster than humans update documentation.

**Why it matters for your pipeline:** the single most common operational use of lineage is the **upstream-impact query**: "I'm about to change dataset `D`. Which downstream models, dashboards, and evaluation runs will be affected?" If you cannot answer this in under a minute from a running system, you do not have lineage; you have a diagram. Chapters 06 and 07 build the system.

### 3. The findability of a dataset is a first-order engineering concern (FAIR)

**Framing:** Wilkinson et al. 2016. *The FAIR Guiding Principles for scientific data management and stewardship.* Scientific Data 3, 160018. <https://doi.org/10.1038/sdata.2016.18>. MLCommons Croissant specification. <https://github.com/mlcommons/croissant>.

The FAIR principles — **F**indable, **A**ccessible, **I**nteroperable, **R**eusable — were authored for scientific data management. They apply to AI training data unchanged. "Findable" means the dataset has a globally unique, persistent identifier and is indexed; "Accessible" means the identifier resolves to the data (or to a documented access procedure) via a standard protocol; "Interoperable" means the data uses a formal, broadly-applicable vocabulary; "Reusable" means the metadata is rich enough that a different team can decide whether to use the dataset.

Croissant is the MLCommons dataset-metadata vocabulary that operationalises FAIR for AI training data. It is a schema.org-based JSON-LD format that describes a dataset's resources, record sets, and fields, with ML-specific extensions. Hugging Face Dataset Card YAML is the per-dataset header consumed by the Hugging Face Hub (and now by a growing number of other tools) that carries task tags, licence tags, language tags, size, and a Croissant pointer.

**Why it matters for your pipeline:** if a sibling team cannot *find* your dataset — cannot search for "customer support SFT 2026-Q3" and arrive at the signed snapshot — they will build their own. The result is three un-deduplicated near-copies of the same corpus with slightly different licence postures. Chapter 10 is where findability is engineered in.

### 4. The authoring layer and the cluster-ingest layer have a named contract

**Framing:** `training-pipeline-engineer` (L35) curriculum charter. WebDataset specification (NVIDIA / Alex Alemi). <https://github.com/webdataset/webdataset>. MosaicML StreamingDataset. <https://github.com/mosaicml/streaming>. Hugging Face Datasets library. <https://huggingface.co/docs/datasets/>. Ray Data. <https://docs.ray.io/en/latest/data/data.html>.

The *authoring layer* — this role — is responsible for producing a snapshot that is correct, licensed, decontaminated, documented, versioned, signed, and findable. The *cluster-ingest layer* — `training-pipeline-engineer` at L35 — is responsible for making that snapshot *fast to train on* on a specific cluster topology (shard count, I/O pattern, sequence-length bucketing, tokeniser throughput, resume semantics, deterministic sharding under node churn).

The two layers meet at a **storage layout**: how the snapshot bytes are physically laid out on the cluster-visible storage. The four dominant layouts are WebDataset (tar-shards with POSIX-ordered entries), MosaicML StreamingDataset (sharded MDS format with random-access indexes), Hugging Face Datasets (Parquet + Arrow, memory-mappable), and Ray Data streaming (Arrow-record-batch streams with lazy transforms). Each has different tradeoffs on random access, resume semantics, and ingest throughput.

**Why it matters for your pipeline:** if you pick a layout the cluster-ingest side cannot consume efficiently, their training run slows down and they will blame your dataset. If you do not pin the layout version and shard inventory, their retries become non-deterministic and the training run's data order diverges silently. Chapter 11 is where the contract lives.

## The versioning framing in ISO terms

As in every other module in this track, the framing lands on the ISO/IEC 25012 vocabulary you pinned in [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) and [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md):

| ISO 25012 characteristic | How versioning / lineage affects it |
|---|---|
| **Traceability** | Primary. Every snapshot is a bound set of bytes with a signed identifier; lineage events describe its production. Without this module, traceability is aspirational. |
| **Credibility** | A signed snapshot with a SLSA attestation is more credible than an unsigned one, categorically. Downstream review cannot distinguish "trusted" from "trustworthy" without signatures. |
| **Accessibility** | FAIR-A: the snapshot's identifier resolves to the bytes through a standard protocol (object-store URL + hash, or lakehouse catalog + snapshot ID). |
| **Understandability** | FAIR-I and FAIR-R: Croissant + HF card + Datasheet together make the dataset understandable to a reader who did not build it. |
| **Portability** | FAIR-I: standardised metadata formats and standardised storage layouts (WebDataset, StreamingDataset, Parquet) mean the snapshot moves between clusters and vendors without re-authoring. |
| **Recoverability** | Snapshot immutability + object-store versioning + signed hashes mean a corrupt or deleted current state is recoverable to a known-good prior state. |
| **Compliance** | EU AI Act Article 53(1)(a) + Annex XI require training-data summaries by snapshot; the snapshot package is the evidence. |

**ISO/IEC 5259-4** (*AI — Data quality for analytics and machine learning — Part 4: Data quality process framework*) treats versioning and lineage as named processes within the data-quality lifecycle. The deliverable at the end of this module is the artifact that implements that scope for your slice. <!-- needs-research: confirm the ISO/IEC 5259-4 clause and whether it names dataset-versioning explicitly or only via its lifecycle processes. -->

## The substrate decision — four shapes

Different substrates fit different team shapes and different data shapes. Pick the substrate *first*; do not reason about lineage or signing in isolation from the substrate they will be anchored to.

- **Filesystem-shaped, small team** — DVC (Chapter 02). A single researcher or small team with a git repo and an object-store remote. The dataset is a set of files and small-to-mid-sized directories. `dvc.yaml` encodes the pipeline. Fits well for a labelled corpus a few teams iterate on; fits poorly for a shared multi-petabyte lakehouse.
- **Object-store-native, data-engineering shape** — lakeFS (Chapter 03). The dataset lives in S3 / GCS / Azure Blob at volumes where copying to a laptop is not feasible, and multiple teams branch, merge, and roll back collaboratively. Zero-copy branching is the headline feature.
- **Pipeline-heavy, incremental-processing shape** — Pachyderm (Chapter 04). The pipeline re-runs on *data changes* with datum-level incremental semantics. Fits well for heavy-compute per-item pipelines (document conversion, OCR, labelling tool runs) where re-processing everything on every change is prohibitive.
- **Lakehouse / analytical shape** — Delta Lake or Apache Iceberg + Nessie (Chapter 05). The dataset is a *table*, usually billions of rows, consumed via Spark / Trino / DuckDB / Snowflake / Databricks. ACID semantics, snapshot isolation, time travel, schema evolution, and partition pruning are first-class.

These are **not** mutually exclusive. A production AI organisation routinely runs more than one: a small labelled corpus on DVC, a shared pretraining pile on lakeFS, a labelling-vendor-feed on Pachyderm, and a derived training-feature table on Iceberg. Chapter 11's handoff contract abstracts over the substrate choice — the *cluster-ingest layer* should not care whether the snapshot ID is a Git SHA, a lakeFS commit, a Delta version, or an Iceberg snapshot.

## The snapshot identity — what makes a pointer "real"

A snapshot pointer is "real" — defensible, resolvable, replay-safe — if and only if it carries all five of:

1. **Content-addressed hash** — a SHA-256 (or equivalently strong) hash over the exact bytes of the snapshot, or over a Merkle tree whose leaves are per-file hashes. The hash is what makes the snapshot *immutable* even if the underlying storage mutates.
2. **Substrate identifier** — the DVC `.dvc` file + MD5, lakeFS commit SHA, Pachyderm commit ID, Delta table version, Iceberg snapshot ID. Let the reader resolve back to the substrate.
3. **Pipeline version** — the Git SHA of the pipeline code that produced the snapshot. Without this, reproducing the snapshot requires archaeology.
4. **Policy version** — the version pin of the licence policy (mod-102), the firewall version (mod-106), the PII-scrub recogniser set (mod-110), and any other policy input that shaped the snapshot's contents. Without this, reproducing the *decisions* that produced the snapshot requires running old copies of other teams' code.
5. **Signature + attestation** — a `cosign` signature over the content hash, plus a SLSA provenance attestation binding the hash to the pipeline, policies, and principal. Without this, there is no cryptographic chain of custody.

An example snapshot identity record, in the shape Chapter 11's handoff contract will reference:

```yaml
# snapshot/identity.yaml
snapshot:
  id: "sft-chat-owned-ugc-2026-q3-v1.2.0"
  # (1) content-addressed hash
  content_hash:
    algorithm: "SHA-256"
    value: "9a1f3c9d0b2e4f5c7a8e6b0d9c1f3a5e7c9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f"
    merkle_manifest: "s3://example-datasets/.../merkle_manifest.json"
  # (2) substrate identifier
  substrate:
    type: "lakefs"
    repository: "s3://example-lakefs/datasets"
    branch: "sft-chat-owned-ugc-2026-q3"
    commit_id: "a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4"
  # (3) pipeline version
  pipeline:
    repository: "github.com/example/ai-data-pipeline"
    git_sha: "f0e9d8c7b6a5948372615049382716054938271"
  # (4) policy version pins
  policies:
    licence_policy: "licence-policy-v2026-02-01"
    contamination_firewall: "firewall-v2026-07-15"
    pii_recognisers: "presidio-recognisers-v2026-06-30"
  # (5) signature + attestation
  attestations:
    cosign_signature: "rekor:12345678"
    slsa_provenance: "s3://example-datasets/.../slsa-provenance.intoto.jsonl"
    cyclonedx_mlbom: "s3://example-datasets/.../mlbom.cyclonedx.json"
    spdx_document: "s3://example-datasets/.../spdx.ai.json"
```

Every chapter populates one or more fields. By the end of the module this record is the single document a reviewer or a peer resolves — every other artifact is reachable from it.

## What the snapshot package is **not**

Mark the boundary explicitly:

- It is **not** a backup. Backups are byte-for-byte copies with an RPO / RTO; snapshots are signed, policy-pinned evidence. A backup system that does not carry signatures, policy versions, or lineage is not a substitute.
- It is **not** a feature store. Feature stores (Feast, Tecton, Databricks Feature Store) serve features at training and inference time with point-in-time correctness. A snapshot is a static training corpus. Feature stores *produce* snapshots on demand; they are not the artifact.
- It is **not** a data catalog. A catalog (DataHub, Amundsen, OpenMetadata, Collibra, Alation, Unity Catalog) indexes datasets and exposes search / discovery / lineage *consumers*. The catalog consumes OpenLineage events and Croissant records; it is not the artifact.
- It is **not** the cluster-side ingest implementation. Chapter 11 pins the handoff contract; `training-pipeline-engineer` (L35) owns the ingest pipeline that turns the contract into efficient training-time reads.
- It is **not** training-run reproducibility. Reproducing a training run requires more than the dataset snapshot — it also requires the training-code version, the random seeds, the cluster topology, the checkpointing behaviour, and the eval-set snapshot. The snapshot package is a *necessary* component of training-run reproducibility; it is not sufficient. `fine-tuning-engineer` and `training-pipeline-engineer` own the rest.

## The regulatory frame — why this module is load-bearing

The EU AI Act Article 53(1)(a) and Annex XI require providers of general-purpose AI models to publish a *sufficiently detailed summary* of training data. Article 53(1)(c) requires a *copyright policy*. Both reference training-data *summaries* at a level of detail that implies — and the EU AI Office GPAI Code of Practice makes explicit — that the provider can bind the summary to the exact training-data set they used. That binding is the snapshot package.

The FDA GMLP / PCCP framework for medical-device AI requires change-management evidence, which in practice means per-release snapshot IDs and lineage back to source records. SR 11-7 (Fed banking supervision) and the Colorado AI Act require similar audit trails. In none of these regimes does an unsigned filesystem path pass review.

<!-- needs-research: confirm the controlling version of the EU AI Act GPAI Code of Practice and the exact signatory list at read time; the text and signatories evolve on roughly a six-month cycle. -->

## Summary

- Dataset versioning is a **governed engineering operation**, not a backup chore. The four framing claims: supply chains must be signable (SLSA / Sigstore), lineage is a graph query (OpenLineage), findability is engineering (FAIR / Croissant), authoring and cluster-ingest have a named contract (Chapter 11).
- The module deliverable is a **snapshot package** with six artifacts: versioned snapshot, lineage event bundle, signature + attestation set, ML-BOM + SPDX pair, Croissant + HF card + FAIR record set, and training-ingest handoff contract.
- The four substrate shapes — filesystem (DVC), object-store (lakeFS), pipeline-heavy (Pachyderm), lakehouse (Delta / Iceberg + Nessie) — are chosen per slice, not per organisation. A real pipeline usually runs more than one.
- A snapshot identity carries five fields: content hash, substrate identifier, pipeline version, policy versions, signatures + attestations. All five are load-bearing.
- Boundaries: `training-pipeline-engineer` owns cluster-ingest; `fine-tuning-engineer` owns training-run curation; feature stores and catalogs are consumers of the package, not the package itself.

Chapter 02 is the practical starting point — DVC as the small-team filesystem substrate, with its tracking model, remotes, and `dvc.yaml` pipelines pinned concretely.
