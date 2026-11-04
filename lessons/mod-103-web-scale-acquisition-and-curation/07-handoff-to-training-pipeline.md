# Chapter 07 — Hand-Off to `training-pipeline-engineer`

## Why this chapter exists

This curriculum stops at the **authoring** of a curated corpus. What happens to the shards on the cluster — how they are read at tens of GB/s into tokenisers, sharded across data-parallel ranks, buffered to keep GPUs saturated, shuffled across epochs — belongs to [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) (level 35, same family, one rung up).

The two roles work. Every time. The hand-off between them is the single most-common place in a training program where a bug or a documentation gap costs weeks. This chapter defines the contract.

## What you hand off

Three artifacts per slice:

1. **A set of shards** — files on object storage, in a specific shard format, with a specific shard size, in a specific layout.
2. **A slice manifest** — the machine-readable record of every operator, parameter, source, and per-shard statistic that produced those shards.
3. **A dataset card** — the human-readable companion that describes intended use, known limitations, licensing, and governance evidence.

The shards are what training actually reads. The manifest is what lets training reproduce and audit. The dataset card is what every other stakeholder (model team, evaluation team, governance council, legal) reads. All three ship together; losing any one of them breaks the hand-off.

## Shard formats you will meet

You will not pick a format in isolation — the training team picks it based on their cluster's I/O profile — but you need to speak each one. The four production formats:

### WebDataset

[`WebDataset`](https://github.com/webdataset/webdataset) is a specification and Python library for storing training data as POSIX tar files with per-sample file-grouping conventions. It is the current dominant open format for large-scale pretraining.

- **Shape**: a shard is one `.tar` (or `.tar.gz`) containing many samples. Within a shard, all files sharing a basename belong to one sample. A text sample is typically one `.txt` or one `.json` per basename.
- **Read model**: streaming, record-by-record. Reader reads the tar sequentially; no random access within a shard.
- **Shard-level random access**: a training job picks a random shard per data-parallel rank per epoch. Within a shard, sequential reads.
- **Why it works**: tar + gzip is the single best-understood primitive for streaming large files out of S3/GCS; one HTTP GET per shard; the sequential-read-within-shard pattern plays nicely with object stores.
- **Reference**: Aizman et al., "High Performance I/O for Large Scale Deep Learning", <https://arxiv.org/abs/2001.01858>.

Typical shard layout you'd hand off:

```
s3://my-bucket/slice-v1/shards/
  shard-00000.tar.gz
  shard-00001.tar.gz
  ...
  shard-04312.tar.gz
```

Inside `shard-00000.tar.gz`:

```
0000000.json
0000001.json
...
0012483.json
```

Each `.json` is one document with `{ "text": "...", "url": "...", "warc_record_id": "...", "slice_version": "..." }`. Document count per shard is chosen such that compressed shard size is ~1–4 GB (the sweet spot for object-store GETs).

### MosaicML StreamingDataset

[`MosaicML Streaming`](https://github.com/mosaicml/streaming) is Mosaic's shard format and client library, designed for cloud-native streaming with strong deterministic shuffling guarantees.

- **Shape**: a dataset is a set of `.mds` (Mosaic Data Shard) files plus an `index.json` describing each shard's offsets and sample count.
- **Read model**: samples addressable by global index; the client prefetches shards as needed and caches locally with eviction.
- **Shuffle guarantees**: deterministic shuffle with configurable `shuffle_seed` and `shuffle_block_size`; same seed on different cluster sizes yields the same sample order modulo parallelism.
- **Why it works**: sample-level random access with shard-level prefetch gives good GPU utilisation even under cluster re-sizing.
- **Reference**: <https://docs.mosaicml.com/projects/streaming/en/stable/>.

### Hugging Face Datasets (Parquet + Arrow)

The Hugging Face `datasets` library emits Parquet files and loads them via Apache Arrow. This is the baseline for small-to-medium corpora and for anything you ship on the Hugging Face Hub.

- **Shape**: columnar Parquet, one file per shard.
- **Read model**: column-oriented reads via Arrow; whole-row reads fine for `text`-only datasets.
- **Why it works**: Parquet is the lingua franca of data platforms; good for inter-op with Spark / Trino / DuckDB.
- **Not always right for training**: for pretraining-scale corpora on GPU clusters, WebDataset or Streaming usually beats Parquet on I/O throughput.

### Ray Data streaming

Ray Data is less a shard format than a streaming abstraction. You can emit Parquet or binary files and let Ray Data stream them; the training rig reads them through `ray.data.Dataset` partitions.

- **Right when**: the training rig already runs on Ray and has a `ray.data` reader in the data-loader.

## The slice manifest schema

Here is a concrete, defensible shape. Adapt field names to taste but keep every field.

```json
{
  "slice_id": "cc-en-head-v1",
  "slice_version": "1.0.0",
  "created_at": "2026-11-04T18:22:01Z",
  "owner": "ai-data-engineering-team",
  "scope": {
    "source_snapshot": "CC-MAIN-2024-30",
    "selection_predicate_sql": "SELECT ... WHERE url_host_tld = 'com' AND content_languages LIKE 'eng%' AND fetch_status = 200",
    "input_warc_file_count": 243,
    "input_record_count": 48_215_043
  },
  "acquire": {
    "ai_use_opt_out_policy_version": "aicg-policy-v1.3",
    "robots_cache_snapshot_at": "2026-11-03T00:00:00Z",
    "allowed_url_count": 36_104_210,
    "dropped_url_count": 12_110_833,
    "evidence_index": "s3://my-bucket/slice-v1/acquire_evidence/index.parquet"
  },
  "operators": [
    {
      "name": "trafilatura_extract",
      "version": "1.12.0",
      "config": {"favor_precision": true, "include_tables": true},
      "docs_in": 36_104_210,
      "docs_out": 34_921_055,
      "drop_fraction": 0.033
    },
    {
      "name": "fasttext_lang_id",
      "version": "lid.176.bin@2017-09-04",
      "config": {"target": "en", "threshold": 0.5},
      "docs_in": 34_921_055,
      "docs_out": 16_843_201,
      "drop_fraction": 0.518
    },
    {
      "name": "kenlm_perplexity_tier",
      "version": "en-wiki-20240401.arpa.bin",
      "config": {"tiering": "percentile_33_67"},
      "docs_in": 16_843_201,
      "docs_out": 16_843_201,
      "tier_fractions": {"head": 0.333, "middle": 0.334, "tail": 0.333}
    },
    {
      "name": "minhash_lsh_dedup",
      "version": "datasketch==1.6.4",
      "config": {"num_perm": 128, "threshold": 0.8, "num_bands": 9, "rows_per_band": 13},
      "docs_in": 16_843_201,
      "docs_out": 12_104_903,
      "drop_fraction": 0.281
    },
    {
      "name": "benchmark_decontaminate",
      "version": "lm-eval-decontam@0.4.0",
      "config": {"benchmarks": ["mmlu", "gsm8k", "humaneval", "arc", "truthfulqa", "..."], "ngram": 13},
      "docs_in": 12_104_903,
      "docs_out": 12_098_441,
      "drop_fraction": 0.000534
    },
    {
      "name": "pii_scrub",
      "version": "presidio@2.2.355",
      "config": {"recognisers": ["EMAIL", "PHONE_NUMBER", "US_SSN", "IP_ADDRESS"], "redaction": "surrogate"}
    }
  ],
  "output": {
    "format": "webdataset",
    "path": "s3://my-bucket/slice-v1/shards/",
    "shard_count": 4313,
    "shard_size_bytes_mean": 1_250_000_000,
    "total_doc_count": 12_098_441,
    "total_token_count_estimate": 8_423_900_000,
    "tokenizer_used_for_estimate": "gpt2-byte-level-bpe",
    "per_shard_manifest": "s3://my-bucket/slice-v1/shards/shard_index.parquet"
  },
  "handoff_to": "training-pipeline-engineer-team",
  "signatures": {
    "sigstore_bundle": "s3://my-bucket/slice-v1/slice.sigstore.bundle",
    "slsa_attestation": "s3://my-bucket/slice-v1/slice.slsa-provenance.json"
  },
  "governance": {
    "dataset_card_path": "s3://my-bucket/slice-v1/DATASET_CARD.md",
    "rmf_subcategory_map": "s3://my-bucket/slice-v1/rmf_alignment.yaml",
    "cyclonedx_mlbom": "s3://my-bucket/slice-v1/ml-bom.cyclonedx.json"
  }
}
```

This is the hand-off artifact. Everything the training team needs to read, shuffle, and audit is reachable from it.

## Tokenisation throughput — what to expect

Even though you don't *run* the tokeniser, you need to reason about what will happen when the training team hits your shards:

- **CPU tokenisation**, typical BPE / Unigram SentencePiece / tiktoken, is on the order of **5–30 MB/s per core** depending on vocabulary and implementation. For a 15 TB corpus that is on the order of 10^7 core-seconds — real time but manageable on a multi-hundred-core prep node.
- **Fast tokenisers** (Hugging Face `tokenizers` crate, written in Rust) reach 50–100 MB/s per core.
- **Pretokenisation at authoring time** (you tokenise once and hand off token IDs rather than text) is an option; the trade-off is that you lock the tokeniser choice in the artifact. Most teams hand off text and let training tokenise.

Rule of thumb: if your shards are N TB of text and the tokeniser runs at 50 MB/s per core, an 8-core prep node will take **N × 10^6 / 50 / 8 seconds** to tokenise the corpus linearly. Scale to a 128-core node and you're at **N × 10^6 / 50 / 128 ≈ N × 156 seconds per TB**. A 15 TB corpus is ~40 minutes of tokenisation time on a 128-core prep node — order-of-magnitude.

The training team cares about this figure because tokenisation competes with GPU prefetch. Document it in the manifest's `total_token_count_estimate` + `tokenizer_used_for_estimate`.

## What the training team hands back

Reciprocally, you should expect from the training team:

- **A read-rate report per epoch** — bytes/sec actually sustained, per rank. If this is below GPU-saturating rates you have a shard-layout bug (shards too small; too few shards; wrong compression; wrong format for their loader).
- **Any skipped / failed shards** — occasionally a shard fails CRC or decompression. You need the list to re-emit.
- **Loss curves correlated to data slices** — if a training run shows pathological loss on a specific slice, they attach the slice ID to the loss window and send it back. You investigate with mod-104 tools.
- **Benchmark-contamination post-hoc reports** — if the training team discovers contamination after training started (via Shi et al. 2023 Min-K% Prob or Oren 2023 exchangeability tests), they return the finding. See mod-106.

## Shard-size and shard-count discipline

Common failure modes and their causes:

| Symptom | Likely cause |
|---|---|
| GPUs idle, I/O under-saturated | Shards too big; few shards per rank per epoch; use smaller shards or more shards |
| GPUs idle, S3 request rate saturated | Shards too small; too many HTTP GETs; coalesce |
| Poor shuffle quality (the same examples appear in the same minibatch across epochs) | Too few shards; you need many more shards than data-parallel ranks |
| OOM on shard load | Shards too large for the loader's buffer; cap individual shard size |
| Inconsistent sample counts across ranks | Shards have variable sample counts and the loader expects uniform; emit uniform-count shards |

A reasonable default: **shards of 1–4 GB**, **shard count ≥ 100 × data-parallel world size**, **uniform sample count per shard within 5%**.

## Signing and attestations

The manifest is not the slice; it is a document *about* the slice. The two need to be bound. The pattern:

1. Compute SHA-256 of each shard file; include in `per_shard_manifest` (`shard_index.parquet`).
2. Compute SHA-256 of `slice_manifest.json` with the per-shard manifest's path included.
3. Sign the resulting bundle with [Sigstore](https://www.sigstore.dev/) (keyless via OIDC in CI, or key-based for off-CI).
4. Emit a SLSA Level-1+ provenance attestation (<https://slsa.dev/spec/>) describing the build (which pipeline code at which git SHA produced this slice).

Downstream consumers verify signature + attestation before reading shards. [`mod-109-dataset-versioning-lineage-reproducibility`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) covers signing in depth. For this chapter, the point is that the hand-off **includes** the signature bundle and the attestation path; without them the hand-off is ambiguous.

## Summary

- The hand-off to `training-pipeline-engineer` (level 35) is three artifacts: **shards**, **slice manifest**, **dataset card**. All three ship together.
- Learn the four shard formats — **WebDataset** (POSIX tar, streaming), **MosaicML StreamingDataset** (sample-addressable, deterministic shuffle), **Hugging Face Parquet + Arrow** (columnar, Hub-compatible), **Ray Data streaming** — and pick with the training team based on their cluster's I/O.
- The slice manifest schema includes: scope + selection predicate, acquire-stage evidence, every operator with version + config + in/out counts, output shard layout, governance references, and Sigstore / SLSA attestations.
- Shard discipline: 1–4 GB each, ≥100× data-parallel world size in count, uniform sample count. Common I/O failure modes map directly to shard-shape errors.
- Sign and attest the slice. The manifest and the shards are bound by SHA-256 + Sigstore signature; SLSA provenance names the pipeline code SHA.
- Expect the training team to hand back read-rate reports, failed-shard lists, loss-vs-slice correlations, and post-hoc contamination reports. Your pipeline must be re-runnable and re-emittable in response.

That is where this module ends. The next step in the pipeline — reading these shards on a training cluster, doing high-throughput tokenisation, coordinating shuffle across tens of thousands of GPU-ranks — is the `training-pipeline-engineer` curriculum at level 35.
