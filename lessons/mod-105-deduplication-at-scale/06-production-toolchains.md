# Chapter 06 — Production Toolchains: `data-juicer`, `NeMo Curator`, and `dolma dedup`

## Why this chapter exists

`datasketch` (Chapter 04) is the right toolkit for 10^6 – 10^7 documents and for reasoning about the algorithm. At the production scales you will actually operate on — 10^8 – 10^11 documents — three toolchains dominate:

- **`dolma`** (AllenAI) — the dedup component of the Dolma data pipeline.
- **`NeMo Curator`** (NVIDIA) — CUDA / RAPIDS-accelerated near-dup with Dask / Ray.
- **`data-juicer`** (Alibaba) — Ray-based operator framework with MinHash + LSH plus many adjacent operators.

Each is a plausible default. They differ in deployment shape (Spark-style vs Ray vs CUDA), where they run (CPU cluster vs GPU cluster vs single node vs managed cloud), and what the operator surface looks like (declarative config vs Python-code pipeline). This chapter is the practical-engineering comparison so you can pick one deliberately per slice.

## The decision axes

| Axis | Values |
|---|---|
| Deployment | Single-box CPU · Spark cluster · Ray cluster · GPU box · GPU cluster |
| Operator model | YAML / TOML config · Python pipeline code · Spark DataFrame API |
| Accelerator | CPU only · CUDA (RAPIDS / cuDF) |
| Scale sweet spot | 10^7 · 10^8 · 10^9 · 10^10+ |
| Ecosystem fit | Standalone · plugs into a larger curation pipeline |
| Audit-trail support | Fingerprint dump · cluster file · signed manifest |

Measure each candidate against these before adopting.

## `dolma` — AllenAI's dedup component

**Repo:** <https://github.com/allenai/dolma>. **Paper:** Soldaini, Kinney, Bhagia et al., *Dolma: an Open Corpus of Three Trillion Tokens for Language Model Pretraining Research*, ACL 2024. <https://aclanthology.org/2024.acl-long.840/>. **Docs:** see `docs/` and the README in the repo.

### Shape

- CLI and Python operator library. Config-driven: declare the pipeline as a TOML / YAML file, run `dolma mix` / `dolma dedupe`.
- Parallelism model is **parallel Python processes with a Bloom-filter-backed de-duplicator**. Exact dedup via Bloom filters; near-dup via a MinHash-LSH implementation that includes a disk-backed variant for scale.
- Native formats: JSONL + gzipped shards. Spark-free; AllenAI deliberately optimised for a "many boxes, parallel Python" idiom.

### When to pick `dolma`

- You are running on a CPU cluster or a cluster of large boxes and do not want a Spark / Ray dependency.
- You want a *dedup toolchain that is already used in a public pretraining release* — Dolma is the reference dataset.
- You want Bloom-filter-based exact dedup out of the box with tunable false-positive rate (Chapter 02).
- Your corpus is in JSONL + gzip, which is where dolma lives natively.

### What you give up

- Bloom-filter exact dedup has a bounded false-positive rate — acceptable for pretraining, not acceptable for forensic / audit-grade work (you would need to switch to the exact-set mode and pay the memory cost).
- Less flexible operator composition than `data-juicer` for non-dedup operators (language ID, quality filters). Dolma supports them but with less polish.

### Minimal operator shape (paraphrased)

```toml
# dolma-dedup.toml
streams = [
  { name = "my_slice",
    documents = ["s3://.../my_slice/*.jsonl.gz"] }
]

[dedupers.my_slice_doc]
name = "document_dedup"
documents.key = "text"
documents.attribute_name = "doc_dedup"
# bloom filter sizing from expected N and acceptable ε
bloom_filter.size_in_bytes = 1073741824   # 1 GiB
bloom_filter.estimated_doc_count = 1000000000
bloom_filter.desired_false_positive_rate = 0.0001
```

Record this config in the slice artifact; the Bloom-filter parameters are part of your dedup report's provenance (Chapter 08).

<!-- needs-research: re-verify the current dolma CLI command names and TOML field names against the repo at read time — they have changed across releases. -->

## `NeMo Curator` — NVIDIA's GPU-accelerated curator

**Repo:** <https://github.com/NVIDIA/NeMo-Curator>. **Docs:** <https://docs.nvidia.com/nemo-framework/user-guide/latest/datacuration/index.html>. **Blog introduction:** <https://developer.nvidia.com/blog/scale-and-curate-high-quality-datasets-for-llm-training-with-nvidia-nemo-curator/>.

### Shape

- Python pipeline API on top of Dask (`dask-cudf`) and Ray. Operators work on `cuDF` DataFrames; the dedup operator runs on GPU.
- MinHash signature computation is GPU-accelerated; LSH is partitioned across workers. On A100-class hardware, the per-document fingerprint throughput is an order of magnitude faster than CPU `datasketch`.
- Integrates with the rest of the NeMo stack: tokenisers, quality filters, PII redaction, benchmark decontamination.

### When to pick NeMo Curator

- You have GPU cluster capacity (your team is a model-training team; this is often the case for the role).
- Slice size is 10^9 – 10^11 documents. The GPU acceleration dominates the economics here.
- You want **fuzzy dedup and exact dedup from the same operator library**, with first-class integration with PII and contamination operators.
- You are already using NeMo for training — the operator-graph dedup slots cleanly into the data-prep stage of the NeMo pipeline.

### What you give up

- GPU dependency. Without GPUs, NeMo Curator is a strict pessimisation over `dolma`.
- Rapidly-evolving API. Pin the version aggressively and check CHANGELOG entries between releases.

### Minimal pipeline shape (paraphrased)

```python
from nemo_curator import ScoreFilter, Sequential
from nemo_curator.modules.fuzzy_dedup import FuzzyDuplicates, FuzzyDuplicatesConfig
from nemo_curator.datasets import DocumentDataset

config = FuzzyDuplicatesConfig(
    cache_dir="./nemo_cache",
    num_hashes=128,
    num_buckets=16,         # <-- this is "bands" (Chapter 04)
    hashes_per_bucket=8,    # <-- this is "rows per band"
    jaccard_threshold=0.8,
    char_ngrams=24,         # character-level n-gram width
    use_64bit_hash=False,
)

ds = DocumentDataset.read_parquet("s3://.../my_slice/*.parquet")
dedup = FuzzyDuplicates(config=config)
dup_pairs = dedup(ds)
```

<!-- needs-research: cross-check API names and default values against the current NeMo-Curator release at read time; `num_buckets` and `hashes_per_bucket` are the current (as of 2024) names. -->

## `data-juicer` — Alibaba's operator framework

**Repo:** <https://github.com/modelscope/data-juicer>. **Paper:** Chen, Dou, Li, Deng, Shen, et al., *Data-Juicer: A One-Stop Data Processing System for Large Language Models*, SIGMOD 2024 companion, arXiv 2309.02033. <https://arxiv.org/abs/2309.02033>. **Docs:** <https://data-juicer.readthedocs.io/>.

### Shape

- Ray-based execution engine with a rich catalogue of ~100 operators spanning text / image / multimodal.
- Dedup operators: `document_deduplicator` (exact, MD5), `document_minhash_deduplicator` (MinHash + LSH), `document_simhash_deduplicator` (SimHash), `image_deduplicator`, `video_deduplicator`.
- Config-driven pipelines (`config.yaml`) plus Python API.

### When to pick data-juicer

- You have a Ray cluster available or want to stand one up.
- You want *multimodal* dedup (image, video) as first-class, not just text.
- You want to compose dedup with a large operator catalogue (language-score filtering, harmful-content filters, etc.) in one pipeline.
- You want **both MinHash and SimHash** operators from a single framework.

### What you give up

- Operator-level APIs evolve quickly; the operator inventory has shifted substantially across releases.
- Spark-shaped teams will find the Ray model unfamiliar; buy-in required.

### Minimal config (paraphrased)

```yaml
# data-juicer-dedup.yaml
project_name: 'my_slice_dedup'
dataset_path: './data/my_slice.jsonl'
export_path: './data/my_slice_dedup.jsonl'
np: 32            # num Ray workers

process:
  - document_deduplicator:                      # exact (Chapter 02)
      lowercase: false
      ignore_non_character: false

  - document_minhash_deduplicator:              # near-dup (Chapter 04)
      tokenization: space
      window_size: 5                            # k-shingle width
      lowercase: true
      ignore_pattern: null
      num_permutations: 128
      jaccard_threshold: 0.8
      num_bands: 16                             # <-- b (Chapter 04)
      num_rows_per_band: 8                      # <-- r (Chapter 04)
      tokenizer_model: null
```

<!-- needs-research: confirm data-juicer operator names and parameter shapes against the current release at read time; the operator registry has been renamed in some releases. -->

## Spark + `datasketch` — the roll-your-own option

For organisations with a mature Spark stack and no appetite for a new framework, **Spark + MinHash in a UDF + a shuffle-by-band reduce** is a well-trodden alternative. The pattern:

1. Shingle in a Spark `map` transformation.
2. Compute the MinHash signature in another `map` — either via a `pandas_udf` with `datasketch` or via Spark MLlib's `MinHashLSH` (be careful: Spark MLlib's `MinHashLSH` uses a different parameterisation — document which).
3. Explode signature into `(band_id, band_hash, doc_id)` tuples, group by `(band_id, band_hash)`, emit candidate pairs.
4. Verify pairs on signatures, union-find for clusters.

This is close in spirit to what Lee et al. 2021's `NearDup` tool does; a Spark port of that approach is a common in-house choice.

**When to pick this path:** your platform team's answer to every data-pipeline question is "Spark", your SRE / cost team reviews new frameworks with suspicion, and you can afford the engineering time to maintain the custom pipeline. Otherwise prefer one of the three framework options.

## Public pretraining pipelines — what they actually used

A short reference table, for citing in design docs. Verify against each project's public code at read time.

| Dataset | Dedup toolchain | Granularity | Threshold |
|---|---|---|---|
| **The Pile** (Gao et al. 2020) | Custom MinHash + LSH | Document | ~0.87 |
| **C4** (Raffel et al. 2020) | Byte-level exact per URL, then filter | Document | Exact |
| **RefinedWeb** (Penedo et al. 2023) | MinHash + LSH | Document | 0.8 |
| **Dolma** (Soldaini et al. 2024) | `dolma` Bloom + MinHash | Document + paragraph | ~0.8 |
| **SlimPajama** (Cerebras / Together 2023) | MinHash + LSH | Document | ~0.8 |
| **FineWeb** (Penedo et al. 2024) | `datatrove` MinHash | Document | 0.8 |
| **RedPajama-Data v2** (Together 2023) | Dask + MinHash | Document | 0.8 |

<!-- needs-research: confirm each entry's exact settings from the dataset's released code / README at read time; the thresholds here are published in the papers / repos but have shifted in minor-version updates. The table is the shape; the numbers deserve a re-check. -->

Two recurring patterns in that table:

- **Jaccard 0.8** is the field-standard document-level threshold.
- **Document granularity** is the default; paragraph granularity is additive (Dolma) rather than replacement.

## Benchmarking the options

For the exercise in this module you will stand up at least one of these on a 10^7-document slice. A short benchmarking rubric:

- **Throughput.** Documents / second end-to-end (shingle → signature → LSH → cluster).
- **Peak memory.** Per-worker RSS during the LSH build. Compare to your box's memory ceiling.
- **Signature storage.** Final MinHash signature column size on disk.
- **Clustering latency.** Time from "all pairs found" to "all clusters written".
- **Reproducibility.** Does re-running on the same slice with the same config emit bit-identical outputs? Set deterministic hashes; log the seed.
- **Operability.** How much engineering effort per slice to add an operator or change a threshold?

Record these in the dedup report's §"benchmarking".

## Selecting a toolchain — three heuristics

1. **If you have GPUs and your corpus is 10^9+**, start with **NeMo Curator**. The economics change decisively above ~10^9 documents.
2. **If you have a CPU cluster and want a public reference implementation**, start with **dolma**. It is the toolchain used by the Dolma corpus itself, which is a highly defensible provenance story.
3. **If you want multimodal dedup or a rich operator catalogue beyond dedup**, start with **data-juicer**. It is the broadest operator surface of the three.

For a single-box 10^6–10^7-document experiment, `datasketch` is still the right tool. Chapter 04's `MinHashLSH` call suffices.

## What goes into the dedup report at this layer

For each slice you emit:

- `toolchain: dolma | nemo-curator | data-juicer | datasketch | spark+minhash` and its git SHA / package version.
- `config_sha256`: SHA-256 of the exact config file.
- `worker_shape`: `{cpu_cores | gpus, memory_gb, parallelism}`.
- `wall_clock_hours` and `compute_cost` (if measurable).
- `output_fingerprint_sha256`, `output_cluster_file_sha256`, `output_removal_manifest_sha256`.

A dedup step is a signed operation. The report is how Chapter 08 ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)) provenance pipeline knows what happened.

## Summary

- **Three mainstream toolchains** cover production dedup: `dolma` (CPU, Bloom + MinHash, config-driven), `NeMo Curator` (GPU, Dask / Ray, operator API), `data-juicer` (Ray, broad operator catalogue incl. SimHash and multimodal).
- **Spark + `datasketch`** is the roll-your-own route for teams with a mature Spark stack and no appetite for a new framework.
- **Jaccard 0.8, document granularity, MinHash K=128** is the field-standard choice every public pretraining pipeline converges on; paragraph-level is additive.
- **Pick by deployment shape and scale, not familiarity.** GPU cluster → NeMo Curator; CPU cluster + public-reference comfort → dolma; multimodal or operator-breadth needs → data-juicer.
- **Record toolchain, config SHA, versions, and output SHAs** in the dedup report. A toolchain choice is a provenance-affecting decision.

Chapter 07 picks up with the adjacent threat you must also catch at this stage — adversarial perturbations (Nightshade, Glaze) and poisoning signals in web-scraped corpora. Dedup pipelines are often the first (and sometimes the only) stage with the fingerprint infrastructure to spot these.
