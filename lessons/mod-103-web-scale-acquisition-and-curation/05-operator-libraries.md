# Chapter 05 — Operator Libraries: Data-Juicer, NeMo Curator, Dolma Toolkit, Ray Data

## Why this chapter exists

The first thing people do when they read Chapter 02 is write their own operators. This is a mistake. The operators that go into a modern web-scale curation pipeline are not hard to write individually — a language ID wrapper, a length filter, a hash dedup — but there are *eighty-plus* of them in a serious pipeline, they all need to run at Common-Crawl throughput, and getting their observability, parallelism, and failure semantics right is an engineering project in its own right. Four open projects have already done that work:

1. **Alibaba [`data-juicer`](https://github.com/modelscope/data-juicer)** — a one-stop operator library with 80+ configurable operators across text, image, video, and audio (Chen et al. 2023, <https://arxiv.org/abs/2309.02033>).
2. **NVIDIA [`NeMo Curator`](https://github.com/NVIDIA/NeMo-Curator)** — GPU-accelerated curation operators, designed to run on Dask / Ray at NeMo scale.
3. **AI2 [`dolma`](https://github.com/allenai/dolma)** — the Dolma team's toolkit (Rust core, Python API) for the exact operators used to build Dolma.
4. **Ray Data** (<https://docs.ray.io/en/latest/data/data.html>) — not a curation library per se but the distributed-dataset primitive on which the other three can run.

This chapter walks through each one with enough depth that you can pick between them for a given slice, and so that you know what parts of the pipeline are **library-supplied** versus **custom-code surface** in your project.

## Why you don't hand-roll

A CCNet-style pipeline needs, at minimum:

- A WARC reader with HTTP-range S3 support.
- HTML extraction (trafilatura or resiliparse).
- URL / host filtering against blocklists.
- Language ID with a pinned model.
- Up to 40 heuristic quality taggers (length, repetition, line termination, stop-word ratio, etc.).
- MinHash + LSH dedup with tunable parameters.
- Benchmark n-gram decontamination against ~20 benchmarks.
- PII detection and redaction.
- Shard writer and manifest emitter.

That is twelve operators at the "class of operation" level; the quality-tagger family alone is forty individual operators. Each operator needs: a config schema, idempotency, a way to report its throughput, a way to report what it dropped and why, a way to recover from partial failure, and a way to be versioned. If you write all of this yourself you have a two-engineer-quarter project before the first slice runs.

Every library below exists because someone did that work already. **Use it.**

## Alibaba Data-Juicer

**Reference**: Chen, Zhang, Zhao, Chen, Ren, Hu, Liu, Fang, Wang, Zhu, Wang, Zhao, Zhou, Tian, Yang. *Data-Juicer: A One-Stop Data Processing System for Large Language Models.* SIGMOD 2024. <https://arxiv.org/abs/2309.02033>. Repo: <https://github.com/modelscope/data-juicer>.

What it is:

- An operator library with 80+ operators (the number keeps growing; verify against the current operator catalog in the repo) across text, image, video, and audio modalities.
- A YAML-configured pipeline runner that composes operators into a sequence with per-operator logging.
- Operator taxonomy: **Mappers** (edit documents), **Filters** (drop documents), **Deduplicators** (within-shard and across-shard), **Selectors** (sample, split).
- Backends: pandas / `dask` for single-node and multi-node, Ray for cluster-scale, Spark optional.

Operator naming convention: `type_object_metric`. Examples:

- `language_id_score_filter` — drop documents whose top-language score is below threshold.
- `perplexity_filter` — drop by perplexity threshold (KenLM-backed).
- `text_length_filter` — length bounds.
- `word_repetition_filter` — fraction of repeated n-grams.
- `special_characters_filter` — fraction of non-alphanumeric characters.
- `clean_html_mapper` — strip HTML.
- `document_deduplicator` — MinHash LSH.
- `url_filter` — host/domain allow/deny list.

The YAML config shape (schematic):

```yaml
# recipe.yaml
project_name: my-slice-v1
dataset_path: data/slice-v1-raw/
export_path: data/slice-v1-curated/
executor_type: default   # or ray

process:
  - url_filter:
      blocklist_path: assets/ut1_blocklist.txt
  - clean_html_mapper: {}
  - language_id_score_filter:
      lang: en
      min_score: 0.5
  - text_length_filter:
      min_len: 100
      max_len: 100000
  - word_repetition_filter:
      rep_len: 10
      max_ratio: 0.3
  - perplexity_filter:
      lang: en
      max_ppl: 2500
  - document_deduplicator:
      num_blocks: 10
      lowercase: true
      ignore_non_character: true
```

Running it:

```bash
python tools/process_data.py --config recipe.yaml
```

The library emits per-operator statistics (count in, count out, bytes processed, wall-clock) and writes the full operator chain to the slice's metadata. This is the "80+ operators of utility" from the module's learning objectives.

**Strengths**:
- Breadth of operators, especially cross-modal.
- YAML-first configs — reproducibility by construction.
- Good documentation with worked examples.
- Active community.

**Caveats**:
- Some operators are thin wrappers that assume a specific external model (FastText, KenLM) is present on disk; you own the model paths.
- Ray backend is well-supported but you're still responsible for cluster provisioning.
- Some multi-modal operators require heavyweight dependencies (OpenCV, decord); keep a text-only install profile if you're doing text-only work.

## NVIDIA NeMo Curator

**Reference**: NVIDIA, "NeMo Curator User Guide." <https://docs.nvidia.com/nemo/curator/latest/>. Repo: <https://github.com/NVIDIA/NeMo-Curator>.

What it is:

- GPU-accelerated curation operators implemented with cuDF, Dask-cuDF, and RAPIDS.
- Designed to run on a multi-GPU node or a cluster via Dask.
- Operator families: data loading (WARC, Common Crawl, JSON, Parquet), language ID (FastText, NGram-based GPU-side), exact + fuzzy dedup (GPU MinHash), quality filtering (configurable score functions), classification-based filtering (domain / quality classifiers), PII redaction.
- Ships with reference pipelines for recreating known corpora (its examples include a Common Crawl + Wikipedia reference).

Key differentiator: **GPU-side MinHash dedup**. For slices large enough that CPU MinHash is bottlenecking on hash computation, cuDF-backed MinHash is often 10–30× faster per dollar of hardware. This is the operator that makes NeMo Curator worth adopting even if the rest of your stack is CPU-first.

Pattern (schematic):

```python
import nemo_curator as nc
from nemo_curator.datasets import DocumentDataset

ds = DocumentDataset.read_json("data/slice-v1-raw/*.jsonl.gz", backend="cudf")

ds = nc.LangDetectModule(target_lang="en").call(ds)
ds = nc.ScoreFilter(
    score_fn=nc.FastTextQualityClassifier(),
    threshold=0.8,
).call(ds)
ds = nc.FuzzyDuplicatesRemover(
    num_hashes=260, num_buckets=20, hashes_per_bucket=13,
).call(ds)

ds.to_parquet("data/slice-v1-curated/")
```

**Strengths**:
- GPU MinHash at scale.
- Interoperable with NeMo training stacks.
- Documented reference pipelines for CC-scale.

**Caveats**:
- GPU-first; CPU-only operation is possible but you're not getting the main value.
- Library surface changes with NeMo releases; pin versions.
- Fewer operators than data-juicer by raw count; the breadth story is weaker but the depth per operator (especially dedup) is strong.

## Dolma Toolkit

**Reference**: Soldaini et al. 2024 (see Chapter 04). Repo: <https://github.com/allenai/dolma>. Documentation: <https://github.com/allenai/dolma/blob/main/docs/>.

What it is:

- The exact toolkit AI2 used to build Dolma. Rust core (`dolma-cli`) for the hot paths (tokenisers, hash dedup, line-by-line taggers), Python API for orchestration and config.
- Opinionated **tagger / filter** separation: taggers emit per-document signals to a side-car file; filters consume those signals and decide what to drop. Signals are reusable across runs.
- Operator families: taggers (language ID, length, repetition, NSFW probability, URL features, PII detection), filters (score thresholds, Boolean combinations), deduplication (exact SHA-256 and MinHash LSH), mixers (combine sources with configurable sampling).

Pattern (schematic, YAML-driven):

```yaml
# tagging_config.yaml
documents:
  - s3://my-bucket/slice-v1-raw/*.jsonl.gz
taggers:
  - cld3_en_doc_v2
  - char_length_with_paragraphs_v1
  - gopher_v1              # the Gopher quality heuristics
  - ft_lang_id_en_doc_v2
  - olmo_pretokenizer_with_paragraphs_v1
processes: 128
```

```yaml
# mixing_config.yaml
streams:
  - name: cc-main-2024-30
    documents:
      - s3://my-bucket/slice-v1-raw/*.jsonl.gz
    attributes:
      - cld3_en_doc_v2
      - gopher_v1
    output:
      path: s3://my-bucket/slice-v1-curated/
      max_size_in_bytes: 4_000_000_000
    filter:
      include:
        - "$@.attributes[?(@.cld3_en_doc_v2[0][0] >= 0.5)]"
      exclude:
        - "$@.attributes[?(@.gopher_v1 contains 'low_quality')]"
```

Run with `dolma tag --config tagging_config.yaml` then `dolma mix --config mixing_config.yaml`.

**Strengths**:
- The actual Dolma operators — if you want the Dolma recipe exactly, this is the shortest path.
- Rust implementations of hot-path operators are fast on commodity CPUs.
- Tagger/filter separation is enforced by the tool's shape, not just a convention.
- Side-car attribute files are a clean lineage story (mod-109).

**Caveats**:
- Operator catalog is smaller than data-juicer.
- Tightly scoped to text (no multi-modal).
- Config DSL (JSONPath-style filter expressions) has a learning curve.

## Ray Data

**Reference**: Ray Data documentation. <https://docs.ray.io/en/latest/data/data.html>.

What it is:

- The distributed-dataset abstraction inside Ray — streaming, auto-parallelising transforms over Arrow-backed batches, with native WARC / JSON / Parquet readers.
- Not a curation library — it supplies **the runtime**, not the operators. You bring the operators (or wrap data-juicer / NeMo Curator / dolma operators as Ray tasks).
- The key features for curation workloads: **streaming execution** (don't materialise whole shards in memory), **pipelined parallelism** across operator stages, **autoscaling** across a Ray cluster, and **resilience** to worker failure.

Pattern:

```python
import ray
from ray.data import read_binary_files

def extract_warc(record_bytes: bytes) -> list[dict]:
    # yields one dict per WARC record in this file-slice
    ...

def extract_text(rec: dict) -> dict:
    rec["text"] = trafilatura.extract(rec["html"])
    return rec

ds = (
    read_binary_files("s3://commoncrawl/crawl-data/CC-MAIN-2024-30/segments/.../warc/*.warc.gz")
    .flat_map(extract_warc)
    .map(extract_text)
    .filter(lambda r: r["text"] and len(r["text"]) > 100)
    .write_parquet("s3://my-bucket/slice-v1-extracted/")
)
```

Ray Data shines when:

- You need to go beyond one machine.
- Operators have heterogeneous costs (GPU extract + CPU filter + GPU MinHash mixed in one DAG).
- You want checkpointing and resumability without writing it yourself.

Ray Data is often the *substrate* for a data-juicer / NeMo Curator run — both libraries have "Ray executor" modes that lift their operator chains onto Ray Data under the hood.

## How to pick

A pragmatic routing table:

| Situation | Reach for |
|---|---|
| You want the exact Dolma recipe with minimal deviation | Dolma toolkit |
| You want maximum operator breadth, text + multi-modal, YAML-first | Data-Juicer |
| Your bottleneck is MinHash and you have GPUs | NeMo Curator |
| You need to scale beyond one node with heterogeneous workers | Ray Data (wrap operators from any of the above) |
| You are doing one-off slice work on a workstation | Dolma toolkit or Data-Juicer with default backend |

It is entirely reasonable (and common) to use **more than one** in a pipeline: Data-Juicer for extraction + taggers, NeMo Curator for GPU MinHash, Ray Data as the glue. Document each handoff in the slice manifest.

## The custom-code surface

Even with a library, you will write some code. Expect to own:

1. **The slice selection predicate** (the Chapter 01 SQL / DuckDB query against the CC index). No library generalises well across selection logic.
2. **Project-specific URL / host allow-lists and deny-lists.** The UT1 lists and similar are starting points; you will maintain a project-specific overlay.
3. **Project-specific tagger thresholds**, documented and version-controlled.
4. **The handoff manifest emitter** (Chapter 07). The slice manifest shape is project-specific; libraries help you produce it but don't define its contents.
5. **Project-specific benchmarks for decontamination** (mod-106). The canonical ~20 benchmarks are a floor; you often have internal eval sets to add.

Everything else — language ID, length filtering, repetition detection, NSFW classification, MinHash, perplexity, benchmark n-gram match — should come from a library. If you find yourself implementing MinHash from scratch, stop and check whether you are solving the right problem.

## Instrumentation and debugging

All four libraries emit per-operator statistics; they do not emit a *comparable* statistics bundle. A durable practice is to wrap each operator in an **in-house decorator** that logs:

- Operator name + version.
- Config hash.
- Count in, count out, drop fraction.
- Wall-clock, peak memory.
- Random sample of 10 dropped records with per-record reasons.

The random sample is the single most valuable instrument for debugging a pipeline. Without it you will spend hours asking "why did this operator drop 40% of my records?" With it you look at ten records and have an answer in five minutes.

## Summary

- Don't hand-roll operators. Four open libraries — **Alibaba Data-Juicer** (80+ ops, YAML-first, multi-modal), **NVIDIA NeMo Curator** (GPU-accelerated, especially MinHash), **AI2 Dolma toolkit** (the Dolma recipe exactly, Rust core, strict tagger/filter separation), and **Ray Data** (the distributed-dataset substrate) — cover the operator surface.
- Pick a library by use case; combine when it helps. Document each handoff in the slice manifest.
- Your custom-code surface is the *selection predicate*, the *project-specific allow/deny lists and thresholds*, the *manifest emitter*, and *project-specific decontamination benchmarks*. The rest comes from a library.
- Wrap every operator in a uniform instrumentation layer that logs config, drop fraction, and a small random sample of dropped records. The sample is the most useful debugging tool in the pipeline.

Chapter 06 picks up a concern that every operator in this chapter touches but none enforces by default — the crawling-ethics contract from mod-102, now at the acquire stage of the pipeline.
