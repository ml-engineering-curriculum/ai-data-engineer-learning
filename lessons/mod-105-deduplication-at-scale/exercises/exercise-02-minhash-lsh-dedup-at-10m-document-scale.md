# Exercise 02 — MinHash + LSH Deduplication at 10M-Document Scale

**Estimated effort:** 4 hours
**Chapter dependencies:** [Chapter 03 — Near-dup fundamentals: MinHash and SimHash](../03-near-dup-fundamentals-minhash-and-simhash.md), [Chapter 04 — MinHash + LSH at scale: datasketch, bands, rows, and the threshold curve](../04-minhash-lsh-at-scale.md)

## Objective

Stand up an explicit-parameters MinHash + LSH pipeline with `datasketch` on a 10M-document slice. Calibrate the S-curve against a labelled sample, run document-level near-dup, resolve clusters, and extend the Exercise 01 dedup report with §3's `near_dedup_document` section and the beginnings of §4 (memorisation-risk delta).

By the end you will have:

- A notebook / script that plots the S-curve `P(flagged|J) = 1 - (1 - J^r)^b` for at least three `(b, r)` configurations and shows the empirical duplicate-rate on a labelled sample overlaid.
- A MinHash + LSH pipeline with `(num_perm, bands, rows)` passed in *explicitly*, not via threshold-only initialisation.
- A clusters file (one row per cluster, member list) and an updated removal manifest.
- A duplicate-count distribution before and after dedup (the Kandpal-2022-style tail).
- The `dedup_report.yaml` extended with §3 `near_dedup_document` and §4 `memorisation_risk` fields.

## Prerequisites

- Python 3.11+ with `datasketch`, `xxhash`, `networkx` (or `scipy.sparse.csgraph` / `union_find` of your choice), `pandas` / `pyarrow`, `matplotlib`.
- The output of Exercise 01 — the exact-deduped slice. If you skipped Exercise 01, synthesise one: ≥ 1M documents of real-ish text (Wikipedia dump, HF `wikitext`, or a Common Crawl slice) with no byte-identical duplicates but seeded with near-duplicate clusters (light edits, re-ordered paragraphs) at known Jaccard values.
- Chapter 04 read. You should be able to compute `s* ≈ (1/b)^(1/r)` on paper for a given `(b, r)`.

## Problem statement

Design and run a document-level near-dup pipeline that:

1. Shingles each document into word 5-shingles (configurable).
2. Builds a `num_perm=128` MinHash signature per document.
3. Builds an `MinHashLSH` index with **`(b, r)` passed in explicitly**.
4. Queries each signature and verifies candidate pairs against the MinHash Jaccard estimator with a Jaccard threshold you have calibrated against a labelled sample.
5. Union-finds candidate pairs into clusters.
6. Resolves clusters with a documented survivor rule (default: longest surviving document).
7. Emits the `near_dedup_document` block of the Chapter 08 §3 report plus the `memorisation_risk` block of §4.

## Starter guidance

1. **Calibrate the threshold first, not last.**
   - Sample 500 candidate pairs with coarse pre-filtering (e.g. exact-match on the first 40 characters of each doc) to bias the sample toward true near-dups.
   - Estimate ground-truth Jaccard directly on the shingle sets for each pair.
   - Human-label (or heuristic-label) each pair as duplicate / non-duplicate at the operational definition your team agreed on.
   - Plot empirical duplicate rate vs ground-truth Jaccard. Identify the sharp step; that is your operational threshold `J*`.

2. **Plot the S-curve for three candidate `(b, r)` configurations** at `K = 128`. Suggestions: `(16, 8)`, `(10, 13)` (needs `K = 130`), `(20, 7)` (needs `K = 140`). Plot `P(flagged)` vs `J` for each. Pick the configuration whose soft threshold is below `J*` (ensures high recall) by a cushion of about `0.05–0.10`.

3. **Build the signature table.**
   - Use `MinHash(num_perm=128)`.
   - Hash shingles with `xxhash` for stability; `datasketch` lets you pass a `hashfunc`.
   - Write signatures to disk as a Parquet column of fixed-width bytes.

4. **Build the LSH index.** Pass `params=(BANDS, ROWS)` to `MinHashLSH`. Insert all signatures. Query each signature for candidates. Keep pairs where the MinHash Jaccard estimator ≥ `J*`.

5. **Union-find the pairs** into clusters. For 10^7 docs this fits in memory; for 10^8 you would need a shard-and-merge pattern — document that extension.

6. **Resolve clusters.** Pick survivor = longest doc per cluster. Write `removed_docs.parquet` extending the Exercise 01 schema.

7. **Compute the duplicate-count distribution** at document level for the pre-dedup and post-dedup slices. "Duplicate count" here is the cluster size each document belongs to; post-dedup every document has duplicate-count 1 by construction, so the pre- and post-distributions are the headline input to the memorisation-risk section.

8. **Extend `dedup_report.yaml`** with:
   - `§3.near_dedup_document.*` — pre/post counts, removal rate, clusters_formed, mean/max cluster size, per-source removal rate.
   - `§4.memorisation_risk.pre_dedup_duplicate_count_distribution.{p50,p90,p99,p99_9}` and post equivalents.
   - `§4.memorisation_risk.methodology = kandpal_2022_frequency_curve`.
   - `§4.memorisation_risk.extraction_risk_delta_estimate` — a one-sentence claim grounded in Kandpal 2022, not fabricated.

9. **Write `s_curve.md`.** A one-page report with the S-curve plot, the labelled-sample calibration plot, and the rationale for your `(b, r)` and `J*` choice.

## Deliverables

```
exercise-02/
  code/
    shingle.py
    build_signatures.py
    build_lsh.py            # explicit (b, r)
    verify_pairs.py         # MinHash Jaccard estimator check
    union_find.py
    cluster_resolve.py
    emit_report.py          # extends Exercise 01's dedup_report.yaml
    plot_s_curve.py
  data/
    calibration_sample.jsonl         # 500 pairs with human labels and ground-truth Jaccard
  results/
    signatures.parquet
    signatures.parquet.sha256
    candidate_pairs.parquet
    verified_pairs.parquet
    clusters.parquet
    clusters.parquet.sha256
    removed_docs.parquet            # extended from Exercise 01
    dedup_report.yaml               # §3 near-dup + §4 fields populated
    plots/
      s_curve.png
      calibration.png
      cluster_size_hist.png
  s_curve.md
  README.md
```

## Acceptance criteria

- The S-curve plot includes at least three `(b, r)` curves at `K = 128` and your chosen configuration is justified in `s_curve.md` with a cushion of `0.05–0.10` between the soft threshold `s*` and your operational Jaccard threshold `J*`.
- `build_lsh.py` passes `params=(BANDS, ROWS)` to `MinHashLSH`; it does **not** rely on threshold-only initialisation.
- The calibration sample file has ≥ 500 pairs with per-pair ground-truth Jaccard and human (or heuristic) labels, and the chosen `J*` sits on or near the empirical step in the duplicate-rate curve.
- Candidate pairs pass through a verification step using the MinHash `jaccard` method; pairs failing verification are not promoted to the clusters file.
- `§3.near_dedup_document.per_source_removal_rate` has ≥ 5 distinct sources; `§3.near_dedup_document.max_cluster_size` is populated and the cluster-size histogram plot matches the reported value.
- `§4.memorisation_risk.extraction_risk_delta_estimate` is a sentence that cites Kandpal 2022 and does not fabricate a precise multiplier.
- Determinism: re-running `build_signatures.py` with the same seed produces bit-identical signatures.
- Any factual claim in `s_curve.md` or the README that could not be verified at authoring time is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Shard the LSH index** into `P = 8` partitions by `xxh(doc_id) mod P`. Measure the effect on wall-clock and on the number of candidate pairs produced (candidate pairs in a sharded index are a strict subset of those in a monolithic index — explain why).
- **Replace `datasketch`'s in-memory store** with a disk-backed store (Redis or `shelve`). Measure the throughput cost.
- **Instrument false positives**: for a sample of candidate pairs, compute ground-truth Jaccard directly on shingle sets (not via MinHash). Compare the empirical FP rate to the S-curve prediction.
- **Compare `MinHashLSH` with `MinHashLSHForest`** on the same slice. Explain why the pretraining convention uses the former even though the latter is sometimes more flexible.
- **Preview SlimPajama's numbers**: replicate its reported ~50% removal rate on a Common Crawl slice; if your rate differs by more than 20 percentage points, document why (threshold, granularity, slice composition).

## What this exercise is not

- It is not a paragraph-dedup exercise (Exercise 03).
- It is not a distributed-toolchain exercise (Exercise 04).
- It is not the complete dedup report — §3 paragraph / sentence rows and §5 adversarial-review fields come in Exercises 03 and 05.
