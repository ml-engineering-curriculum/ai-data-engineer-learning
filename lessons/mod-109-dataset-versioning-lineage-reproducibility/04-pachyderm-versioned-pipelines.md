# Chapter 04 — Pachyderm: Versioned Pipelines and Datum Parallelism

## Why this chapter exists

DVC (Chapter 02) and lakeFS (Chapter 03) are *data-first* substrates: you version data, and the pipeline is something that runs over it. Pachyderm (Pachyderm Inc., community edition open-source) is a *pipeline-first* substrate: you version data **and** pipelines as a single unit, and the engine re-runs only the per-input-item ("datum") work that depends on what changed. For AI data engineering this matters when the per-item work is *expensive* — OCR, document conversion, synthetic-data generation per seed, labelling-tool reprocessing — and re-running everything on every refresh is prohibitively slow or costly.

Primary sources: <https://docs.pachyderm.com/>, <https://github.com/pachyderm/pachyderm>.

## The Pachyderm model in one page

Core objects:

- **Repository (repo)** — a versioned file tree, similar in shape to a lakeFS repo. Backed by object storage.
- **Commit** — a content-addressed snapshot of a repo. Immutable once closed; has a parent and a diff.
- **Branch** — a mutable reference to a commit (`master` by default). Branches track the latest commit in a logical line of versions.
- **Pipeline** — a user-authored transformation declared as a YAML / JSON spec. It consumes one or more *input repos* and produces exactly one *output repo*. The pipeline *itself* is versioned — changing the spec creates a new pipeline version.
- **Datum** — the unit of parallel work. A datum is a subset of the input commit's files, defined by a *glob pattern* in the input spec. Pachyderm schedules one job per datum, and a datum re-runs only when its own input files change.
- **Job** — one execution of a pipeline on a specific input commit. Produces a commit on the output repo.

The two big ideas:

1. **Pipelines are first-class, versioned artifacts.** The pipeline spec is stored with the repo; every pipeline run references the spec version plus the input commit. A job's output is attributable not just to the data but to the exact pipeline code that produced it.
2. **Datum-level incremental semantics.** When upstream data changes, Pachyderm computes which datums have changed inputs and runs only those. Unchanged datums inherit outputs from the previous job. This is the Pachyderm-native way to make a petabyte-scale pipeline respond to a 1% upstream change in proportional time.

## When Pachyderm fits

Pachyderm fits when:

- The pipeline is **heavy per-item** — hundreds of ms to minutes per input file. OCR, LLM-based document conversion, labelling-tool rendering, large synthetic-generation batches, image / audio transcoding.
- Upstream data changes **incrementally** — new or updated files arrive on a cadence, and you want the pipeline to run only on what changed, not the full corpus.
- The pipeline has **multiple stages with fan-in / fan-out** — Pachyderm's DAG engine coordinates them natively, with per-stage datum-level parallelism.
- You can **run a Kubernetes cluster** — Pachyderm deploys to K8s and uses K8s pods as the per-datum compute unit. Without K8s, Pachyderm is not the right choice.

Pachyderm does **not** fit (or fits less well) when:

- The pipeline is **cheap per-item** and fits in a single machine's RAM. The overhead of the datum-scheduling engine outweighs the savings.
- The team wants **in-place SQL-shaped work** on a table. Lakehouse substrates (Chapter 05) win here.
- The pipeline is **small and single-stage** — Airflow / Prefect / Dagster with OpenLineage emission (Chapter 06) will cover the lineage story with less infrastructure.
- The team is **not already running Kubernetes** at production quality. Pachyderm's operational overhead compounds the K8s overhead.

## A worked example — a per-document pipeline

Consider a pipeline that ingests licensed PDFs, converts them to Markdown with an LLM-based converter, runs the mod-106 firewall per-document, scrubs PII per mod-110, and emits a cleaned Markdown corpus.

**Input repos:**

- `pdf-ingest` — the raw PDFs, one per file.
- `firewall-policy` — the firewall config version pin (a single file).
- `pii-recognisers` — the PII recogniser set (a single file).

**Output repo:**

- `markdown-cleaned` — the final Markdown corpus.

**Pipeline specs:**

```yaml
# pdf-to-markdown.pipeline.yaml
pipeline:
  name: pdf-to-markdown
transform:
  image: registry.example.com/pdf-to-markdown:v0.3.1
  cmd: ["python", "/app/convert.py"]
input:
  pfs:
    repo: pdf-ingest
    glob: "/*"   # one datum per PDF
parallelism_spec:
  constant: 32
resource_requests:
  cpu: 2
  memory: 4G
```

```yaml
# firewall.pipeline.yaml
pipeline:
  name: firewall
transform:
  image: registry.example.com/mod-106-firewall:v2026-07-15
  cmd: ["python", "/app/firewall.py"]
input:
  cross:
    - pfs:
        repo: pdf-to-markdown
        glob: "/*"   # one datum per converted doc
    - pfs:
        repo: firewall-policy
        glob: "/*"   # the single policy file
parallelism_spec:
  constant: 16
```

```yaml
# scrub-pii.pipeline.yaml
pipeline:
  name: scrub-pii
transform:
  image: registry.example.com/mod-110-scrub:v0.4.2
  cmd: ["python", "/app/scrub_pii.py"]
input:
  cross:
    - pfs:
        repo: firewall
        glob: "/*"
    - pfs:
        repo: pii-recognisers
        glob: "/*"
```

**What the engine does:**

- An upload to `pdf-ingest` of 10,000 new PDFs triggers `pdf-to-markdown` with 10,000 datums. With `parallelism_spec.constant: 32`, 32 K8s pods run concurrently.
- Each datum produces a Markdown file in the `pdf-to-markdown` output commit.
- `firewall` sees the new commit on `pdf-to-markdown`; the `cross` input produces one datum per `(converted_doc, policy_file)` pair. The policy file is unchanged, so only *new* converted docs produce new datums.
- `scrub-pii` runs similarly.
- A change to `firewall-policy` invalidates **every** `firewall` datum (every converted doc × the new policy file) — the pipeline re-runs entirely. This is the correct behaviour: a policy change should re-process the corpus.

The practical consequence for AI training-data work: **policy changes trigger full re-processing; data changes trigger incremental re-processing.** This matches the operating model you want — mod-102 / mod-106 / mod-110 policy versions are load-bearing, and a change should force a full re-audit.

## The glob-pattern decision — the most important design choice

Datum granularity is the single most important design decision in a Pachyderm pipeline. The glob pattern determines how input files are grouped into datums:

- `glob: "/*"` — one datum per top-level entry. If `pdf-ingest` has `docs/a.pdf`, `docs/b.pdf`, `docs/c.pdf`, there are three datums, each processing one PDF.
- `glob: "/"` — a single datum containing the entire repo. The whole pipeline runs as one job.
- `glob: "/*/*"` — one datum per entry two levels deep. If organised as `vendor_a/doc1.pdf`, `vendor_a/doc2.pdf`, `vendor_b/doc3.pdf`, there are three datums.
- `glob: "/vendor_*"` — one datum per top-level entry matching the pattern.

The tradeoff:

- **Smaller datums** → more parallelism, more scheduling overhead, finer-grained incremental re-runs. Good when per-item work is non-trivial and inputs change independently.
- **Larger datums** → less overhead, coarser incremental semantics. Good when per-item work is small and batching amortises overhead.

For AI training-data work, the rule of thumb: pick a datum size that keeps per-datum runtime in the **30 seconds to 10 minutes** range. Below 30 seconds, the K8s scheduling overhead dominates; above 10 minutes, a failure costs too much re-work and the engine's retries become expensive.

## Cross vs. union vs. join inputs

Pachyderm's input DSL has three composition operators:

- **`cross`** — Cartesian product. Every pair of (datum from A, datum from B) is a datum of the combined input. Use when you want each file in A to be processed against every file in B (the "policy file" pattern above).
- **`union`** — multiplexes two inputs; each datum carries a file from one of them. Use when a pipeline processes both A and B uniformly.
- **`join`** — matches datums on a path pattern (e.g., both inputs have `*/doc.txt` and `*/metadata.json`). Each join datum carries both. Use when you need to co-process matching file sets from two repos.

An AI data engineer's typical pattern is `cross(data_repo, policy_repo)` — each data file is processed with the current policy file, and a policy change invalidates every datum.

## Spouts, services, and external triggering

Pachyderm supports two further input shapes that are useful in production:

- **Spouts** — long-running "listener" pipelines that poll an external source (a Kafka topic, an S3 notification queue, an HTTP endpoint) and ingest new items into an output repo. Useful for continuous crawl / labelling-vendor-feed ingestion.
- **Services** — pipelines that expose a long-running HTTP service backed by Pachyderm storage. Rare in training-data work; more common in ML-model serving.

Both are documented in the Pachyderm docs; they are not the primary surface of this chapter.

## Snapshot identity under Pachyderm

Mapping Chapter 01's five snapshot-identity fields to Pachyderm:

| Identity field | Pachyderm primitive |
|---|---|
| Content-addressed hash | The commit ID is a UUID; the commit's manifest is content-addressed over its file tree. A snapshot-level SHA-256 Merkle root is derived from the manifest. |
| Substrate identifier | `pach://<repo>@<branch>=<commit-id>` or `pach://<repo>@<commit-id>`. |
| Pipeline version | The Pachyderm pipeline spec is itself versioned; the job metadata references the spec commit. External Git SHA for the transform image is pinned in the `transform.image` tag. |
| Policy versions | Policy inputs are their own input repos (`firewall-policy`, `pii-recognisers`); their commit IDs are the version pins. |
| Signatures + attestations | External — `cosign sign-blob` over the output commit's Merkle root; the resulting Rekor entry referenced in commit metadata. |

A pinned snapshot URL for downstream consumers is a Pachyderm commit ID on the terminal output repo.

## Lineage — what Pachyderm gives you natively, and what OpenLineage adds

Pachyderm's `pachctl inspect job <job-id>` command prints the job's input commits, output commit, pipeline version, transform image, and runtime stats. Every output commit carries a *provenance graph* — the set of input commits (and recursively, their input commits) that produced it. This is Pachyderm-native lineage; it is strong, and it is already content-addressed.

**It is not, however, OpenLineage.** Pachyderm's lineage lives inside the Pachyderm server; a downstream consumer (Marquez, DataHub, Unity Catalog) sees nothing unless you emit OpenLineage events. Pachyderm does not ship a first-party OpenLineage adapter (it is not listed in the [OpenLineage integrations index](https://openlineage.io/docs/integrations/)); the integration is a self-authored sidecar pattern. The typical shape:

- A Pachyderm-side hook or sidecar translates each job's `COMPLETE` into an OpenLineage event with:
  - `job.name` = the pipeline name (`pdf-to-markdown`, `firewall`, `scrub-pii`).
  - `job.namespace` = the Pachyderm cluster identifier.
  - `run.runId` = the Pachyderm job ID.
  - `inputs[]` = dataset facets for each input commit.
  - `outputs[]` = dataset facets for the output commit.
  - `run.facets.sourceCode.gitSha` = the transform image's source commit.
- The hook runs on every pipeline completion and sends to Marquez / DataHub via their OpenLineage HTTP endpoints.

## Operating notes

- **Transform image versioning.** The `transform.image` is a container image tag. Pin it to a *digest* (`@sha256:...`), not a floating tag. A floating tag mutates silently and breaks reproducibility — even if the Pachyderm pipeline spec is unchanged, a tag that resolves to different bytes next week produces different outputs.
- **Per-datum resource requests.** `resource_requests` and `resource_limits` are per datum. For an LLM-based converter, a reasonable request is 4–8 vCPU + 8–16 GB RAM; for a heavier batch, GPU scheduling is supported via `resource_requests.gpu`.
- **Timeouts and retries.** Datum timeouts and job-level retries matter: a 10,000-datum job with a bad datum will otherwise stall indefinitely. Set `datum_timeout` and `job_timeout` explicitly; set `datum_tries` to a small number (3 is a sensible default).
- **Garbage collection.** Pachyderm's commit graph is content-addressed on the underlying bucket. `pachctl gc` reclaims orphaned object-store bytes; schedule it on a cadence after confirming retention requirements.
- **Multi-tenant clusters.** Pachyderm's RBAC is namespace-level. Keep sensitive-data repos in their own namespaces with distinct service accounts.

## Where Pachyderm sits in a real stack

Teams that adopt Pachyderm rarely abandon DVC or lakeFS; the three compose:

- **DVC / lakeFS / Delta / Iceberg** holds the *source of truth* for the dataset.
- **Pachyderm** runs the *pipeline* that transforms source datasets into derived datasets, with datum-level incremental semantics.
- **Pachyderm's output repos** can be mirrored or exported to the main source-of-truth substrate (e.g., committed back to a lakeFS branch) for consumption by non-Pachyderm tools.

A typical pattern: a lakeFS repo holds a petabyte-scale crawl; a Pachyderm pipeline reads a specific commit as its input, runs heavy per-document work, and emits the result; the result is mirrored back to a lakeFS branch; a merge to `main` on lakeFS gates the result into the production snapshot.

## Peer uses in the track — where this substrate lands

- A document-conversion pipeline (PDF → Markdown) over a licensed partner corpus is a canonical Pachyderm fit.
- A per-seed synthetic-generation pipeline from mod-108 is a canonical Pachyderm fit when each seed takes minutes of generator time and the seed pool changes incrementally.
- A per-item labelling-vendor-output post-processing pipeline from mod-107 is a canonical Pachyderm fit.
- A one-pass whole-corpus dedup (mod-105) is **not** a Pachyderm fit — the operation is inherently global. Keep mod-105 on Spark or a native dedup tool; Pachyderm wraps *per-item* work, not global operations.

## Summary

- Pachyderm versions **data and pipelines together** and schedules **per-datum** work. A change to inputs or policy produces proportional re-processing — not whole-corpus re-runs.
- Core primitives: **repo**, **commit**, **branch**, **pipeline**, **datum**, **job**. The glob pattern is the single most important design choice (target 30 s – 10 min per datum).
- Input composition operators: **`cross`**, **`union`**, **`join`**. The dominant AI pattern is `cross(data_repo, policy_repo)` — policy changes invalidate every datum; data changes invalidate just the touched datums.
- Pachyderm fits **heavy-per-item, incrementally-updated, K8s-deployed** pipelines. It does not displace DVC or lakeFS; it composes with them.
- Snapshot identity maps to a Pachyderm commit ID on the terminal output repo. Transform images must be **digest-pinned**, not tag-pinned, for reproducibility.
- Pachyderm's native lineage is strong but internal; OpenLineage emission (Chapter 06) is required for external consumers.

Chapter 05 picks up the lakehouse substrate — Delta Lake and Apache Iceberg (with Nessie) — which is the right choice when the training data is naturally a *table* rather than a filesystem tree, and SQL-shaped access patterns dominate.
