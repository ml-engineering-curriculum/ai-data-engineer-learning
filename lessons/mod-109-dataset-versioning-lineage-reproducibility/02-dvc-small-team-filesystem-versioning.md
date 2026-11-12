# Chapter 02 — DVC: Small-Team, Filesystem-Shaped Versioning

## Why this chapter exists

DVC (Data Version Control, Iterative.ai) is the most widely-deployed small-team dataset-versioning tool and the one most new practitioners meet first. It is *the* default for a researcher or small ML team with a Git repository, a modest-size corpus, and an object-store bucket. It is also the substrate most frequently misused — teams either try to scale it past its sweet spot or ignore its pipeline layer entirely and use only the storage-pointer layer. Chapter 02's goal is to pin the model exactly: what DVC is, what shape of dataset it fits, and the three primitives that matter (tracking, remotes, pipelines).

Primary sources: <https://dvc.org/doc>, <https://github.com/iterative/dvc>.

## The DVC model in one page

DVC separates *tracking* from *tracked storage*. Git tracks small text pointers — `.dvc` files and `dvc.lock` — that describe which objects a working tree depends on. The objects themselves (model weights, datasets, intermediate artifacts) live in a *cache* on disk and are mirrored to a *remote* (S3, GCS, Azure Blob, SSH, HDFS, local filesystem, etc.). Pulling a Git commit does not pull the data; you run `dvc pull` for that.

Three concepts to internalise:

1. **Tracked file / directory** — you run `dvc add path/to/dataset`. DVC moves the bytes into the local cache (content-addressed by MD5), writes `path/to/dataset.dvc` (a small YAML pointer) into the working tree, and replaces the original path with a reflink / symlink / hardlink / copy to the cache entry. You commit the `.dvc` file to Git.
2. **Remote** — a named URL (`s3://bucket/path`) configured in `.dvc/config`. `dvc push` uploads cache entries to the remote; `dvc pull` downloads them. Remotes are the *shared* storage layer — Git carries the pointer, the remote carries the bytes.
3. **Pipeline stage** — a block in `dvc.yaml` describing a command, its input `deps`, and its output `outs`. `dvc repro` executes the pipeline, skipping stages whose inputs and command have not changed, and writes `dvc.lock` with the hashes of every input and output. The lock file is the *reproducibility* contract.

A `.dvc` pointer looks like:

```yaml
# path/to/dataset.dvc
outs:
- md5: 9a1f3c9d0b2e4f5c7a8e6b0d9c1f3a5e.dir
  size: 4823945218
  nfiles: 12438
  hash: md5
  path: dataset
```

The `.dir` suffix marks a directory hash — DVC computes it as an MD5 over a canonical JSON manifest of per-file hashes. The size, nfiles, and MD5 are the content-addressed identity of this snapshot.

`dvc.lock` looks like:

```yaml
schema: '2.0'
stages:
  prepare:
    cmd: python src/prepare.py --raw raw/ --out prepared/
    deps:
    - path: raw
      hash: md5
      md5: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6.dir
    - path: src/prepare.py
      hash: md5
      md5: 8f9e7d6c5b4a3928170605948372615a
    outs:
    - path: prepared
      hash: md5
      md5: c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8.dir
```

With `dvc.lock` committed, running `dvc repro` after a clean checkout re-runs *only* the stages whose inputs have changed since this lock was written. This is the atom of reproducibility in the DVC model.

## When DVC fits

DVC fits when:

- The dataset is **O(GB) to low-O(TB)** in size. You can enumerate files, hash them, and store them on an object-store remote without unusual engineering.
- The dataset is a **filesystem tree** — labelled images, SFT JSONL files, WAV audio clips, document corpora. It is not a table you query with SQL; if it were, Chapter 05's lakehouse substrate fits better.
- A **small team** (1–5 engineers) works on it with a Git-style collaborative mental model. A larger team with concurrent writers tends to generate merge conflicts in `.dvc` files and `dvc.lock` faster than DVC's semantics can resolve.
- The **pipeline runs locally or on a single node**. DVC's pipeline layer is a Makefile-shaped DAG without a scheduler. For cross-cluster or heterogeneous-worker pipelines, Pachyderm (Chapter 04) or an Airflow / Dagster / Prefect integration with OpenLineage (Chapter 06) fits better.
- The team is **comfortable with Git**. DVC's UX leans heavily on Git muscle memory; if the team is a data-engineering team used to lakehouse tools, lakeFS (Chapter 03) is a smaller cognitive load.

DVC does **not** fit when:

- The dataset is a **table you update row-by-row** (append-only stream of events, slowly-changing dimension table). DVC's unit of change is a file; a table edit means re-hashing the entire file. Chapter 05 substrates handle this natively.
- The dataset is **multi-petabyte** across many petabyte-scale producers. DVC's pointer-in-Git model does not scale to thousands of concurrent writers with any grace.
- You want **branch / merge / rollback semantics directly on the object store**. DVC branches *are* Git branches, and the data on the remote is addressed by hash — but the Git repo carries only pointers, so a "branch of the dataset" is really a branch of the Git repo. lakeFS (Chapter 03) gives you branch semantics on the object store itself.

## The tracking model — a worked example

A typical labelled-corpus layout:

```
repo/
  .dvc/
    config          # remote configuration
  .dvcignore        # paths DVC should not track
  dvc.yaml          # pipeline definition
  dvc.lock          # pipeline lockfile (hashes of every stage input/output)
  .gitignore        # excludes data/ from git
  src/
    prepare.py
    dedup.py
    firewall.py
    scrub_pii.py
  data/
    raw/            # dvc-tracked directory (ingested source)
      ...
    raw.dvc         # DVC pointer committed to git
    prepared/       # pipeline stage output
    deduped/        # pipeline stage output
    firewalled/     # pipeline stage output
    scrubbed/       # pipeline stage output (final snapshot)
```

The developer experience is close to Git's:

```bash
# first-time add
dvc add data/raw
git add data/raw.dvc data/.gitignore
git commit -m "feat(data): ingest raw corpus for sft-chat-owned-ugc-2026-q3"

# publish bytes to the shared remote
dvc remote add -d origin s3://example-dvc-remote/sft-chat-owned-ugc
dvc push

# collaborator pulls
git clone <repo>
cd <repo>
dvc pull     # downloads data/raw/ from s3://example-dvc-remote/sft-chat-owned-ugc
```

**Operating notes that trip teams up:**

- `dvc add` on a *directory* hashes every file and records a directory hash. Adding 100M small files is slow and wasteful; prefer sharded Parquet or a tarball-of-shards if the files are tiny.
- DVC 3.x uses MD5 for cache content addressing, and the algorithm is not user-configurable — the `.dvc` file spec states only `md5` is currently supported ([dvc.org .dvc file spec](https://dvc.org/doc/user-guide/project-structure/dvc-files), [tracking issue #3069](https://github.com/iterative/dvc/issues/3069)). To align with Sigstore and CycloneDX SHA-256 defaults, compute a separate SHA-256 Merkle manifest over the DVC-tracked bytes at snapshot time and sign that (see "Hashes, signatures, and the mod-109 chain" below).
- `dvc.lock` is the reproducibility contract. Treat it like `package-lock.json` — commit it, review diffs, and investigate unexpected hash changes before merging.
- Reflinks on APFS / XFS / Btrfs avoid double-storing cache entries in the working tree. On ext4 / NTFS without reflink support DVC falls back to copies; set `cache.type` to `reflink,hardlink,symlink,copy` in `.dvc/config` for the best-available behaviour.

## Remotes — the shared storage layer

A remote is a named URL that DVC can `push` to and `pull` from. Configure one per slice or one per team; the choice is operational.

```ini
# .dvc/config
['remote "origin"']
    url = s3://example-dvc-remote/sft-chat-owned-ugc
    region = us-east-1
    server_side_encryption = aws:kms
    sse_kms_key_id = arn:aws:kms:us-east-1:123456789012:key/...
```

Supported remote types include S3, GCS, Azure Blob, SSH, HDFS, HTTP(S), WebDAV, OSS, and local-filesystem (useful as a shared mount). For AI-training-data workloads the S3 / GCS / Azure Blob backends are the realistic choices.

**Operating considerations:**

- **Versioned buckets.** Enable object-store versioning on the remote bucket. DVC's content-addressed layout is immutable *by convention*; versioned buckets make it immutable *by enforcement* — a corrupted or maliciously overwritten object can be rolled back.
- **Object-lock / write-once-read-many (WORM).** For regulated use cases, use S3 Object Lock or GCS Bucket Lock to prevent even the owning account from overwriting snapshot objects within a retention window. This is the primitive that turns a "trust the pipeline" posture into a "trust the storage" posture.
- **KMS encryption.** For any snapshot that contains PII (post mod-110 scrub) or licensed partner data, use a KMS key bound to the training organisation. Record the key ARN in the snapshot's ML-BOM `externalReferences`.
- **Cost.** Cache entries are content-addressed; the same bytes under different paths or different revisions share storage. A naive `git mv` of a large dataset into a subdirectory does *not* double the remote storage. This is one of DVC's best properties.
- **Garbage collection.** `dvc gc --workspace --cloud` prunes cache entries not referenced by any `.dvc` file in the current workspace. Run it on a cadence, but only after confirming the pruned snapshots are not referenced by a *historical* Git ref you still need.

## The pipeline layer — `dvc.yaml`

DVC's pipeline layer is where most teams get the most value and where most teams under-use the tool. A `dvc.yaml` encodes a DAG of *stages*; each stage declares a command, inputs (`deps`), outputs (`outs`), and optionally parameters and metrics.

```yaml
# dvc.yaml
stages:
  prepare:
    cmd: python src/prepare.py --raw data/raw --out data/prepared
    deps:
    - data/raw
    - src/prepare.py
    outs:
    - data/prepared

  dedup:
    cmd: python src/dedup.py --inp data/prepared --out data/deduped --threshold 0.85
    deps:
    - data/prepared
    - src/dedup.py
    params:
    - dedup.threshold
    outs:
    - data/deduped
    metrics:
    - metrics/dedup_report.json:
        cache: false

  firewall:
    cmd: python src/firewall.py --inp data/deduped --out data/firewalled --config policies/firewall-v2026-07-15.yaml
    deps:
    - data/deduped
    - src/firewall.py
    - policies/firewall-v2026-07-15.yaml
    outs:
    - data/firewalled
    metrics:
    - metrics/firewall_report.json:
        cache: false

  scrub_pii:
    cmd: python src/scrub_pii.py --inp data/firewalled --out data/scrubbed --recognisers policies/presidio-recognisers-v2026-06-30.yaml
    deps:
    - data/firewalled
    - src/scrub_pii.py
    - policies/presidio-recognisers-v2026-06-30.yaml
    outs:
    - data/scrubbed
    metrics:
    - metrics/pii_report.json:
        cache: false
```

Running the pipeline:

```bash
dvc repro
# or, selectively:
dvc repro scrub_pii
```

`dvc repro`:

1. Reads `dvc.yaml` and the current `dvc.lock`.
2. For each stage, computes a hash over its declared `deps` (file contents + command string + parameter values).
3. If the hash matches the one in `dvc.lock`, the stage is skipped.
4. Otherwise the stage runs, outputs are hashed, and `dvc.lock` is updated.

The three properties that make this load-bearing for AI training-data work:

- **Policy inputs are first-class `deps`.** A change to `policies/firewall-v2026-07-15.yaml` invalidates the `firewall` stage and every downstream stage. This is the DVC-native way to pin a policy version to a snapshot (snapshot identity field #4 in Chapter 01).
- **Parameters are first-class.** Numeric thresholds, feature flags, and sampling ratios live in `params.yaml` and are read by DVC as part of stage hashing. Changing a parameter invalidates the stages that use it — deterministic replay requires the same parameters.
- **Metrics are tracked.** `dvc.yaml`'s `metrics` section exposes stage-local JSON files to `dvc metrics show` and `dvc metrics diff`. This is the mechanism for reporting the dedup rate, firewall removal count, and PII-scrub rate at pipeline run time without conflating them with the snapshot bytes.

### `dvc exp` — experiment tracking (brief)

DVC 2.0+ added the `exp` command set for experiment tracking. Each `dvc exp run` creates a lightweight ephemeral commit with a parameter perturbation, lets you re-run the pipeline, and makes the result comparable. For AI training-data work this is useful when sweeping filter thresholds or dedup band counts; it is not the primary surface of this chapter and is covered end-to-end in the DVC docs.

## Hashes, signatures, and the mod-109 chain

DVC's MD5-by-default hashing is a weak point for AI supply-chain use cases. MD5 is collision-vulnerable; a sufficiently motivated attacker can produce two different corpora with the same `.dvc` hash. For *ordinary* collaboration this is a non-issue; for *regulated* training data it is below the floor downstream procurement will accept.

Options:

1. **Re-hash under SHA-256 at snapshot time.** DVC 3.x does not expose a public config option to switch the cache hash algorithm — `.dvc` files are MD5-only ([dvc/issues/3069](https://github.com/iterative/dvc/issues/3069)). Instead, compute a Merkle manifest over the snapshot files with SHA-256 and store it alongside the `.dvc` file. `cosign sign-blob` signs the SHA-256 Merkle root (Chapter 08).
2. **Stop trusting DVC hashes for supply-chain purposes.** Treat the `.dvc` file as a *pointer* only; derive snapshot identity from the SHA-256 Merkle manifest and bind the signature to that. This is the posture most regulated-use-case pipelines settle on.

Either way, the output of this chapter at snapshot time is a *content-hash record* (Chapter 01 identity field #1) that downstream Sigstore and SLSA chapters (08, 09) sign.

## Lineage — how DVC composes with OpenLineage

DVC has no built-in lineage emission; `dvc dag` renders a static dependency graph of the current `dvc.yaml`, which is not a lineage event stream. For the OpenLineage integration Chapter 06 pins, a thin wrapper around `dvc repro` emits an OpenLineage `START` event before each stage and a `COMPLETE` / `FAIL` event after, with:

- `job.name` = `dvc.repro.<stage>`
- `job.namespace` = `aicg-datapipeline-<slice>`
- `run.runId` = a stable UUID per `dvc repro` invocation
- `inputs[]` = dataset facets for each `dep`, with `datasetVersion` derived from the DVC hash
- `outputs[]` = dataset facets for each `out`, with `datasetVersion` derived from the DVC hash
- `run.facets.sourceCode.gitSha` = the commit SHA of the `dvc.yaml` repo

Chapter 06 and Exercise 03 pin the integration concretely; keep in mind at Chapter 02 reading time that DVC is a *substrate*, not a lineage system, and the pipeline-run stream is separate work.

## Peer uses in the track — where this substrate lands

- A labelled corpus from mod-107 (`sft-chat-owned-ugc-2026-q3`) is a canonical DVC fit — filesystem tree, O(GB) size, small team.
- A synthetic-generation output from mod-108 (preference pairs, distillation outputs) is a DVC fit as long as the output is one or a few Parquet files per round; if the pipeline produces millions of per-example JSONs, batch them into shards first.
- A benchmark fixture from mod-106 (a pinned decontamination check set) is a canonical DVC fit.
- A multi-TB pretraining corpus from mod-103 is **not** a DVC fit — move to lakeFS (Chapter 03) or an Iceberg table (Chapter 05).

## Summary

- DVC separates **tracking** (Git-committed `.dvc` pointers and `dvc.lock`) from **tracked storage** (cache + remote). The pointer is small text; the bytes live on an object-store remote.
- Three primitives: **tracking** (`dvc add` / `.dvc` files), **remotes** (`dvc remote add` / `dvc push` / `dvc pull`), and **pipelines** (`dvc.yaml` / `dvc repro` / `dvc.lock`).
- The **pipeline layer** is where DVC earns its keep for AI training-data work: policy files are first-class `deps`, parameters are tracked, and `dvc.lock` is the reproducibility contract.
- DVC's default MD5 hashing is **below the supply-chain floor**. Either switch to SHA-256 or derive a parallel SHA-256 Merkle manifest for signing.
- DVC fits a **filesystem-shaped, small-team, O(GB)–O(TB), locally-executed-pipeline** workload. It does not fit multi-petabyte, multi-team, row-level-mutated, lakehouse-shaped workloads.
- DVC does not emit lineage events natively; Chapter 06's OpenLineage wrapper is the composition point.

Chapter 03 picks up the object-store-native substrate — lakeFS — where branch, merge, and rollback semantics apply directly to S3 / GCS / Azure Blob bytes, and the substrate itself scales past DVC's sweet spot.
