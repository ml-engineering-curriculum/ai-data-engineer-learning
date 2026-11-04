# Exercise 04 — Dedup at Scale with NeMo Curator, dolma, or data-juicer

**Estimated effort:** 4 hours
**Chapter dependencies:** [Chapter 06 — Production toolchains: data-juicer, NeMo Curator, dolma dedup](../06-production-toolchains.md), [Chapter 04 — MinHash + LSH at scale](../04-minhash-lsh-at-scale.md)

## Objective

Re-run the Exercise 02 dedup pipeline on the same slice using **one of the production toolchains** (`dolma`, `NeMo Curator`, or `data-juicer`), with equivalent parameters, and compare throughput, memory, correctness (candidate-pair set agreement), and operability against the `datasketch` baseline.

By the end you will have:

- A working production-toolchain pipeline on the same slice as Exercise 02.
- A side-by-side benchmark: throughput, peak memory, wall-clock, output size.
- A correctness comparison: for a random 1,000-pair sample, do both pipelines agree on near-duplicate status?
- A choice statement: for *this* slice, *this* team, *this* deployment, you would ship with toolchain X, and here is why.
- The dedup report extended with Chapter 06 §"What goes into the dedup report at this layer" — toolchain name, version, config SHA, compute shape.

## Prerequisites

- Exercise 02 complete (you need the baseline numbers).
- One of the following stood up and reachable:
  - **`dolma`** on a CPU box with ≥ 32 GiB RAM.
  - **`NeMo Curator`** on a Dask / Ray cluster with at least one GPU (A100-class recommended; small slices work on consumer GPUs).
  - **`data-juicer`** on a Ray cluster of ≥ 4 workers.
- The slice from Exercise 02, converted to the toolchain's expected input format (JSONL + gzip for dolma; Parquet for NeMo Curator; JSONL for data-juicer).
- Chapter 06 read. You can defend your toolchain choice against the two you did not pick.

## Problem statement

Pick one of the three toolchains and:

1. Author the toolchain-specific config / pipeline that reproduces the Exercise 02 setup:
   - K = 128 MinHash permutations
   - `(b, r) = (16, 8)`
   - Jaccard threshold = 0.8
   - Word-level 5-shingles (or the toolchain's closest equivalent — e.g. character n-grams; document any approximation)
2. Run it on the same input slice.
3. Measure throughput (docs/sec), peak memory, wall-clock, output size.
4. For a random 1,000 candidate-pair sample from each pipeline, compute the overlap: `|A ∩ B| / |A ∪ B|` on the pair sets.
5. Produce a `choice.md` that states the recommended toolchain for this slice + team + deployment.
6. Extend `dedup_report.yaml` with the §1 identity updates (new toolchain name, version, config SHA) and the §6 artifact SHAs.

## Starter guidance

### If you pick `dolma`

- Clone <https://github.com/allenai/dolma> and install the CLI.
- Author a TOML config with a `fuzzy_dedup` deduper. Set `num_permutations = 128`, `num_bands = 16`, `rows_per_band = 8`, `jaccard_threshold = 0.8`.
- Run `dolma dedupe --config ./dedup.toml`.
- Consume the attribute outputs and compute removal / cluster stats via a short pandas/pyarrow script.
- Bloom-filter settings for the exact dedup half: match Exercise 01's target `ε = 10^-4` and your expected `N`.

### If you pick `NeMo Curator`

- Install `nemo_curator` from <https://github.com/NVIDIA/NeMo-Curator> (pin a known-good version).
- Author a Python pipeline using `FuzzyDuplicatesConfig(num_hashes=128, num_buckets=16, hashes_per_bucket=8, jaccard_threshold=0.8, char_ngrams=24)`.
- Note: NeMo Curator uses **character n-grams**, not word shingles. Approximation: `char_ngrams = 24` roughly corresponds to 5-word shingles for English prose. Document this approximation in `choice.md`.
- Run on a Dask-CUDA cluster; measure GPU utilisation.

### If you pick `data-juicer`

- Install `data-juicer` from <https://github.com/modelscope/data-juicer>.
- Author a `config.yaml` with `document_minhash_deduplicator` configured per Chapter 06.
- Launch with Ray; run `python tools/process_data.py --config config.yaml`.
- Consume outputs and compute stats.

### Common steps for all three

1. **Freeze versions.** Pin the toolchain version (`pip show`, repo SHA). Record both in the report.
2. **SHA your config.** The config file's SHA-256 is part of the artifact provenance.
3. **Collect compute metrics.** Use `top` / `htop` / `nvidia-smi` for peak RSS / GPU memory, pipeline logs for throughput, `time` for wall-clock.
4. **Export the candidate-pair set.** Each toolchain exposes this differently; you need the raw pair list (or cluster membership from which pairs are derivable) for the correctness comparison.
5. **Correctness comparison.** For a random 1,000-pair sample, compare set membership with Exercise 02's output.
6. **Author `choice.md`.** One page with:
   - Which toolchain you ran, why.
   - Throughput / memory / wall-clock vs Exercise 02 baseline.
   - Correctness agreement (overlap fraction).
   - Operability notes (how hard was it to tune, inspect, re-run).
   - Your recommendation for the hypothetical "this team, this slice" context.

## Deliverables

```
exercise-04/
  toolchain: <dolma | nemo-curator | data-juicer>
  config/
    dedup.<toml|yaml>                # the toolchain config
    dedup.<ext>.sha256
  code/
    run_pipeline.sh                   # launches the toolchain
    collect_metrics.py                # scrapes logs / nvidia-smi / psutil
    export_pairs.py                   # normalises toolchain output to pair list
    compare_pairs.py                  # agreement with Exercise 02
    emit_report.py                    # updates §1 + §6 of dedup_report.yaml
  results/
    pipeline_log.txt
    metrics.json                      # throughput, peak RSS, wall-clock
    candidate_pairs.parquet
    candidate_pairs.parquet.sha256
    pair_agreement.json               # vs Exercise 02
    dedup_report.yaml                 # §1 updated, §6 extended
  choice.md
  README.md
```

## Acceptance criteria

- Pipeline reproduces Exercise 02's semantics to the extent the toolchain supports: K = 128, (b, r) = (16, 8), J = 0.8, document-level. Any deviation (character n-grams vs word shingles, Bloom vs set) is documented.
- `metrics.json` contains at minimum: `docs_per_sec`, `peak_memory_gb`, `wall_clock_hours`, `gpu_memory_gb` (if applicable).
- `pair_agreement.json` reports `jaccard_of_pair_sets` and `intersection_size`, `union_size`. The agreement figure is at least ~0.75 (toolchains will differ; below 0.5 indicates a configuration mismatch, investigate).
- `choice.md` compares your chosen toolchain to the other two by Chapter 06 §"The decision axes" criteria.
- `dedup_report.yaml` §1 fields now reflect the production toolchain (name, version, config SHA, compute shape).
- Any factual claim in `choice.md` or the README that could not be verified at authoring time is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Run the same pipeline on 10× the input volume** (if your infrastructure permits). Does throughput scale linearly? If not, where is the bottleneck?
- **Benchmark two toolchains against each other** on the same slice. Produce a head-to-head memo. Useful when a team is evaluating adoption.
- **Automate the report emitter**: a script that reads the toolchain's native output format and populates the `dedup_report.yaml` fields without hand-authoring.
- **Compare the `FineWeb` `datatrove` pipeline** (<https://github.com/huggingface/datatrove>) as a fourth option. It is the toolchain behind FineWeb and shares shape with dolma.
- **Reproduce SlimPajama's parameters** on a Common Crawl slice. Publish the retention-rate match against the Cerebras / Together blog <!-- needs-research: SlimPajama's exact parameters per the repo at <https://github.com/Cerebras/modelzoo/tree/main/modelzoo/transformers/data_processing/slimpajama>; re-verify at authoring time -->.

## What this exercise is not

- It is not an exhaustive benchmark of all three toolchains. One toolchain is required; two is the stretch goal.
- It is not a full ablation over `(b, r, K, J, k)` — Exercise 02 already explored that surface.
- It is not an adversarial-perturbation exercise (Exercise 05).
