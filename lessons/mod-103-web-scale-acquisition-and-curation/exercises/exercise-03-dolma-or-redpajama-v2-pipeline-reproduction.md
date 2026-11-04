# Exercise 03 — Dolma or RedPajama-Data v2 Pipeline Reproduction (Scoped)

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 02 — CCNet-style pipeline](../02-ccnet-pipeline-end-to-end.md), [Chapter 04 — Public reference pipelines](../04-public-reference-pipelines.md), [Chapter 05 — Operator libraries](../05-operator-libraries.md), [Chapter 06 — Crawling-ethics in code](../06-crawling-ethics-in-code.md)

## Objective

Reproduce a **scoped subset** of either Dolma (Soldaini et al. 2024) or RedPajama-Data v2 (Together AI 2023) on a small slice of Common Crawl, using the published toolkit with the published configuration, and produce a comparison report against the published reference.

You are not rebuilding the full corpus. You are running the published operator sequence with the published parameters against a ~10–50 GB slice, to internalise the shape and the quirks.

By the end you will have:
- A chosen reference pipeline (Dolma `cc` or RedPajama-Data v2), a cited paper/blog, and a documented reason for the choice.
- A reproduction run on a small slice using the published toolkit.
- A comparison report: your output statistics vs. the published reference's at matched stages.

## Prerequisites

- You have read Chapters 02, 04, 05, and 06.
- You have completed Exercise 01 (you have a working slice-extraction setup).
- You have installed the toolkit for your chosen pipeline:
  - **Dolma**: `pip install dolma` plus the Dolma Rust CLI from <https://github.com/allenai/dolma>; the project README lists the install path for the Rust binary.
  - **RedPajama-Data v2**: clone <https://github.com/togethercomputer/RedPajama-Data>; the signal computation pipeline lives there.
- You have at least 100 GB of free disk and ideally 16+ CPU cores.
- You have reviewed the published config / recipe for your chosen pipeline and have identified the subset you'll run.

## Problem statement

Pick one of:

**Path A — Dolma reproduction.** Reproduce the Dolma `cc` sub-pipeline on a scoped input. Minimum operator set: URL filter, trafilatura extraction, language ID, Gopher-style quality taggers, NSFW tagger, cross-document MinHash dedup, benchmark decontamination. Match Dolma's published parameters exactly where the toolkit exposes them.

**Path B — RedPajama-Data v2 reproduction.** Reproduce the quality-signal computation on a scoped input. Minimum signal set: the ~20 `rps_doc_*` signals plus `ccnet_language_score`, `ccnet_perplexity`, and MinHash signatures at 0.7 / 0.8 / 0.9 Jaccard. You will emit every input document with its signals attached — not a filtered subset.

Document the choice in your README with a one-paragraph justification.

## Scope

Your input is **two or three WARC files from one Common Crawl snapshot** — roughly 1–3 GB compressed per file, totalling maybe 100,000–300,000 records. Not the full snapshot. Not a full WARC-segment. Enough to exercise every operator with realistic data; small enough to iterate in minutes to hours, not days.

## Starter guidance

1. **Pin your inputs.** Write down the snapshot ID, the specific WARC filenames, their observed SHA-256s, and the exact source-selection predicate. All three must be in the manifest.

2. **Run the Chapter 06 acquire-stage check** before running any downstream operator. Dolma does not enforce AI-use opt-outs by default; you need to layer them yourself. RedPajama-Data v2 ships quality signals for every input document — including opted-out ones — so you still need to filter before processing.

3. **Install and configure the toolkit**:
   - **Path A (Dolma)**: author a `tagging_config.yaml` and `mixing_config.yaml` matching the published recipe. The Dolma repo publishes its OLMo-era configs at `configs/`; use them as your starting point. Pin `dolma` package version in `requirements.lock`.
   - **Path B (RedPajama-Data v2)**: the pipeline is script-driven, not YAML-driven. Follow the README under `pretraining/CC-v2/`. Note that the full pipeline has multiple stages; you can start with the signal-computation stage and skip the top-N-snapshot aggregation.

4. **Run the pipeline end-to-end** on your WARCs. Capture:
   - Per-stage wall-clock.
   - Per-stage input count, output count, drop fraction.
   - Peak memory.
   - Any errors / skipped records.

5. **Compute your output statistics** at each matched stage: document count, token count, language distribution, mean document length, duplicate fraction, decontamination hit fraction. These are the columns in the comparison.

6. **Compute the reference statistics.** The published Dolma / RedPajama-Data v2 artifacts report their aggregate statistics in the paper/blog; some of them are also in the dataset card on Hugging Face. Scale appropriate to a comparison (your slice is ~0.001% of the published corpus; compare by *ratios*, not absolutes).

7. **Write the comparison report.** A short table of "stage → your ratio → reference ratio → delta → plausible explanation". Example: "trafilatura drop fraction: 0.033 (ours) vs 0.05 (Dolma paper §3). Difference attributable to our slice being .com-heavy whereas Dolma includes more international TLDs."

## Deliverables

- `README.md` — which path you chose, why, and how to re-run the whole thing.
- `inputs.yaml` — the WARC file list with SHA-256s and the selection predicate.
- `configs/` — the exact pipeline configs (toolkit YAML or script arguments).
- `acquire_evidence/` — Chapter 06 opt-out decision records for every URL in the slice.
- `runs/run_YYYY-MM-DD/` — raw outputs of the pipeline run (shards, side-car attribute files for Dolma, or signal files for RedPajama-Data v2).
- `stats/your_stats.yaml` — per-stage statistics.
- `stats/reference_stats.yaml` — the published reference's statistics at matched stages, with citations.
- `comparison_report.md` — the ratios table and the plausible-explanation column.
- `signatures/` — Sigstore bundle covering the slice manifest and shards (stretch goal — not required but welcomed).

## Acceptance criteria

- The reproduction uses the **published toolkit**, not a from-scratch rewrite. The point is to use the operators the reference pipeline shipped.
- Operator parameters are documented and traceable to the published reference. "We used the Dolma default MinHash config" is acceptable only if the Dolma default is actually what's in the Dolma config repo at your pinned version.
- The acquire-stage opt-out check from Chapter 06 is run *before* any downstream operator sees the content.
- The comparison report includes at least four stage-level ratios with reference citations (paper section or blog URL).
- The "plausible explanation" column for each delta is actually plausible — not "unknown". If the real explanation is "we don't know", document what further investigation would resolve it.
- Pinned toolkit + model versions are in `requirements.lock` / `inputs.yaml`. Changing versions invalidates the comparison.
- Any claim about a published reference pipeline's behaviour that you could not verify against the paper or the code is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Run both paths** (Dolma and RedPajama-Data v2) on the same slice. The two pipelines agree on some statistics (dedup ratios tend to track) and disagree on others (perplexity bucketing is RedPajama-only); the disagreements teach the design-space.
- **Vary one parameter** (MinHash threshold, language-ID threshold, decontamination n-gram size) and measure the downstream effect on output statistics. This is a small DataComp-LM-style ablation (Li et al. 2024, <https://arxiv.org/abs/2406.11794>).
- **Emit a dataset card** for your slice in the Hugging Face format, citing both the reference pipeline and your parameter divergences.
- **Sign the slice** with Sigstore and emit a SLSA Level-1+ provenance attestation (mod-109 preview).
- **Cross-check against FineWeb**: run the datatrove URL filter on the same WARCs and report the overlap with your chosen pipeline's URL filter. Where they disagree is interesting.

## What this exercise is not

- It is not a reproduction of the full published corpus; the published corpora are petabyte-scale and were produced on nation-state compute.
- It is not a judgement on which pipeline is better; both are published and defensible.
- It is not an SFT dataset or an evaluation-set construction task; those are `fine-tuning-engineer` and `model-evaluation-engineer` scope respectively.

The exercise is to run the canonical operator chain against a scoped input with the canonical parameters, and to measure your output against the published reference closely enough that you understand where *your slice choices*, not the pipeline, drive the differences.
