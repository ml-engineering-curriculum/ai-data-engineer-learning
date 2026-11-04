# Exercise 01 — N-Gram-Based Contamination Firewall Authoring

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 02 — The eval surface the firewall defends](../02-the-eval-surface-the-firewall-defends.md), [Chapter 03 — N-gram overlap detection](../03-n-gram-overlap-detection.md)

## Objective

Stand up the primitive of the module: an n-gram overlap firewall that scans a small training-corpus slice against a pinned eval manifest covering at least three of the Chapter 02 benchmarks. Calibrate the n-gram width against a labelled sample, emit a `removal_manifest.parquet` with per-benchmark / per-eval-item attribution, and open the skeleton of `decontamination_log.yaml` (§1, §2, §3, §4 pretrain, §8 artifact SHAs) that the remaining exercises will extend.

By the end you will have:

- An `eval_manifest.v1.yaml` with SHAs for `mmlu`, `gsm8k`, and `humaneval` (at minimum) — the three benchmarks that exercise the three classes of eval-side structure (multi-choice, open-ended answer, code).
- A labelled sample of ~200 candidate (eval item, training span) pairs annotated as `leak` / `incidental-overlap` / `unrelated`.
- A precision / recall sweep over `n ∈ {8, 10, 13, 20, 50}`, a chosen operating point, and a short `n_gram_calibration.md` justifying the pick.
- A streaming-scan n-gram firewall (`firewall.py`) that produces `removal_manifest.parquet` with the Chapter 03 §"attribution" schema.
- The seed of `decontamination_log.yaml` — §1 (identity), §2 (manifest), §3 (n-gram detector only), §4 (pretrain removal counts), §8 (artifact SHAs).

## Prerequisites

- Python 3.11+ with `pyarrow`, `pandas`, `xxhash`, `mmh3`, `pybloom-live` (or similar), `datasets`, `matplotlib`, `pyyaml`.
- A ~100k–1M document training-corpus slice. Acceptable sources: a Common Crawl shard (if you completed [`mod-103` Exercise 01](../../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md)), a HF sample of `allenai/dolma` or `HuggingFaceFW/fineweb`, or a seeded synthetic corpus with known planted leaks. The slice must contain at least some documents you can confirm contain benchmark overlaps (seed if necessary).
- Read Chapter 02 and Chapter 03.

## Problem statement

Build an n-gram overlap firewall that:

1. Loads three pinned benchmarks from Hugging Face (MMLU `test`, GSM8K `test`, HumanEval) with explicit `revision=` pins, computes SHA-256 of the raw files, and writes `eval_manifest.v1.yaml` matching Chapter 02's schema.
2. Builds the Chapter 03 §"streaming scan" shape: an in-memory set of eval-side n-gram 128-bit hashes at your calibrated width.
3. Calibrates the width via the Chapter 03 §"calibration drill" against a 200-pair labelled sample. Document the precision / recall curve.
4. Scans the training-corpus slice document-by-document. For each document, records every n-gram window hit, groups by `(benchmark, eval_item_id)`, and writes a row to `removal_manifest.parquet`.
5. Emits §1, §2, §3 (n-gram only), §4 (pretrain only), §8 of `decontamination_log.yaml`.

## Starter guidance

1. **Pin the eval manifest first.**
   - Use `datasets.load_dataset("cais/mmlu", "all", revision="...")` with an explicit revision hash. Do not use "main" / "latest".
   - Hash the serialised split with SHA-256 and write the SHA into `eval_manifest.v1.yaml`.
   - Pick the test split for each benchmark. For MMLU include both `test` and `validation`.
   - Record the normalisation rule set as a short YAML file; hash it; reference the SHA from the manifest.

2. **Build the calibration sample.**
   - Coarse-prefilter candidate pairs: take any training document containing an 8-token overlap with any eval item. Cap at 1,000 candidates.
   - Randomly sample 200 from the candidates (ensure representation across all three benchmarks — stratify).
   - Label each as `leak` / `incidental-overlap` / `unrelated`. "Incidental overlap" is e.g. a training document that shares `"which of the following is true"` with an MMLU stem but is otherwise unrelated.
   - Save `calibration_sample.jsonl`. Each row: `{pair_id, eval_item_id, training_doc_id, overlap_span, label, annotator}`.

3. **Sweep the width.**
   - For `n ∈ {8, 10, 13, 20, 50}`, compute empirical precision (among flagged pairs, fraction that are `leak`) and recall (among `leak` pairs, fraction that are flagged).
   - Plot precision and recall as functions of `n`.
   - Pick `n` at the smallest value whose precision ≥ 0.98 on the sample. Report both precision and recall at the pick.

4. **Build the eval-side n-gram set.**
   - Apply the normalisation pipeline to every eval item.
   - Tokenise with whitespace (the simplest choice for this exercise; document the choice).
   - For each item, for each sliding window of width `n`, hash to `xxh128` and insert into the set.
   - Keep a sidecar `ngram_to_eval_item.parquet` mapping `ngram_hash -> (benchmark, eval_item_id)` so attribution is possible at scan time.

5. **Streaming scan.**
   - For each training document in the slice, normalise, tokenise, slide, lookup. Collect all hits.
   - Group hits per `(benchmark, eval_item_id)`. Emit one row per document × benchmark × eval_item_id combination, matching Chapter 03's `removal_manifest.parquet` schema.
   - The `detector` column is literally `"n-gram"` for this exercise; Chapter 04's detectors will add `"exact-prompt"` and `"llm-substring"` rows in later work.

6. **Emit the log skeleton.**
   - Populate `decontamination_log.yaml` §1 (your name, the engine git SHA, a run timestamp, corpus pre/post SHAs). The slice SHA is the SHA of the input Parquet file; the post-firewall SHA is the SHA of the filtered output.
   - Populate §2 from `eval_manifest.v1.yaml`.
   - Populate §3 with the n-gram detector block only — the exact-prompt and LLM-substring blocks are placeholders.
   - Populate §4.pretrain with per-benchmark n-gram counts aggregated from `removal_manifest.parquet`.
   - Populate §8 with SHAs for the manifest, the removal manifest, and the normalisation rules file.

## Deliverables

```
exercise-01/
  code/
    pin_eval_manifest.py
    normalise.py
    build_ngram_set.py
    firewall.py                # streaming scan
    label_calibration.py       # interactive labelling helper
    sweep_n.py
    emit_log.py
  data/
    eval_manifest.v1.yaml
    normalisation_rules.yaml
    calibration_sample.jsonl   # ~200 labelled pairs
  results/
    ngram_to_eval_item.parquet
    removal_manifest.parquet
    removal_manifest.parquet.sha256
    plots/
      precision_recall_vs_n.png
    decontamination_log.yaml   # §1, §2, §3 (n-gram), §4.pretrain, §8
    n_gram_calibration.md
  README.md
```

## Acceptance criteria

- `eval_manifest.v1.yaml` pins each benchmark with an **explicit HF `revision` hash**, the SHA-256 of the serialised split, and the item count. A "main" or "latest" reference fails the criterion.
- `calibration_sample.jsonl` has ≥ 200 pairs across the three benchmarks, with a three-way label field (`leak` / `incidental-overlap` / `unrelated`).
- `plots/precision_recall_vs_n.png` includes data points at `n ∈ {8, 10, 13, 20, 50}` for both precision and recall. `n_gram_calibration.md` cites the specific `n` chosen and defends the pick against the Chapter 03 §"short-prompt edge case" and §"false positive from short common phrases" risks.
- `removal_manifest.parquet` matches the Chapter 03 §"attribution — the removal manifest schema" schema exactly (column names and types).
- **Attribution is non-degenerate.** For a random sample of 10 flagged documents, you can trace each row to the specific eval item and the specific n-gram window that matched. A row with no `eval_item_id` fails the criterion.
- `decontamination_log.yaml` populates §1, §2, §3 (n-gram detector), §4.pretrain, §8. Other sections may be `null` or `TBD` with an explicit marker.
- **Determinism.** Running `firewall.py` twice on the same inputs produces bit-identical `removal_manifest.parquet`. A row count differing by one is a bug.
- **Normalisation round-trip test.** The repo includes a test that renders a random sample of 10 eval items through the normalisation pipeline, injects them into synthetic training documents, and asserts the firewall flags all 10. Any miss is a normalisation bug.
- Any factual claim in `n_gram_calibration.md` or `README.md` that cannot be verified at authoring time is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Add a fourth benchmark.** Pick MATH or BBH and show the per-benchmark precision / recall curve. BBH's short-prompt tasks surface the Chapter 03 §"short-prompt edge case" directly — route those tasks to the exact-prompt matcher placeholder and document the routing.
- **Switch to a suffix-array backend.** Replace the streaming-scan set with Lee et al. 2021's `ExactSubstr` (<https://github.com/google-research/deduplicate-text-datasets>) on the same slice. Compare wall-clock, memory, and attribution fidelity.
- **Plant and recover.** Inject 50 benchmark items verbatim into the training slice (as stand-alone documents). Confirm the firewall flags all 50 and attributes correctly. Then inject 50 *lightly paraphrased* copies (one word changed per sentence). Report how many your n-gram firewall catches. This is the setup for Exercise 03 and Exercise 04.
- **False-positive hunt.** For the 20 highest-removal training documents, hand-inspect to confirm each is a true leak. If any is a false positive, feed it back into the calibration sample and re-sweep.

## What this exercise is not

- Not the full three-detector firewall. The exact-prompt matcher and the LLM substring detector come later.
- Not multi-surface. This exercise targets the pretrain surface only; SFT / preference / synthetic are later.
- Not the Min-K% / Oren audit response. Those arrive in Exercise 03.
- Not the complete decontamination log. This is the seed; later exercises extend it.
