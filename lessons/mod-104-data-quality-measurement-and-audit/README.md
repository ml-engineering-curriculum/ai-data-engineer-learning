# mod-104 — Data Quality Measurement, Representativeness, and Corpus Audit

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 14 hours

## What this module is about

The previous module ([`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md)) was about *authoring the pipeline*. This module is about *measuring what it produces*. The deliverable at the end of the module is one artifact — a per-slice quality report — that captures seven things about a given slice:

1. **Row-level quality checks** — Great Expectations, Soda Core, AWS Deequ, pandera — against the ISO/IEC 25012 fifteen-characteristic model and the ISO/IEC 5259-2 catalogue of ML-specific measures.
2. **A coverage matrix** proving every load-bearing characteristic is implemented by at least one measure.
3. **A Dodge-style audit** (Dodge et al. 2021) — what each filter strips, who the stripped content represents, and what the kept set contaminates.
4. **A Kreutzer-style multilingual audit** (Kreutzer et al. 2022) — human L1-speaker ratings per language.
5. **A representativeness report** — disaggregated counters across language, dialect, source, time, length, and divergences vs. a named target distribution.
6. **A data-centric ablation** (DataComp / DataComp-LM methodology) — fixed model, variable data, measured eval deltas.
7. **A signed, structured report** that plugs directly into the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) and the mixture-strategy work at `staff-ml-engineer` (level 40).

By the end you will be able to hand a model-team engineer a `per_slice_quality.yaml` and have them know — within five minutes of reading it — whether the slice is safe to train on and which of its filters earn their place.

## Learning objectives

- Author Great Expectations suites, Soda Core checks, AWS Deequ (Spark) rules, and pandera dataframe schemas — each exercising the ISO/IEC 25012 quality characteristics (accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability) and the ISO/IEC 5259-2 *quality measures for AI/ML* catalogue.
- Run a Dodge-style corpus audit (Dodge et al. 2021 *Documenting Large Webtext Corpora*) on a sourced slice — surfacing what filters strip, demographic representation of the filtered-out content, and contamination against public benchmarks; apply the Kreutzer et al. 2022 *Quality at a Glance* methodology to multilingual slices.
- Measure representativeness and population coverage at the corpus level — subpopulation balance, language / dialect / sociolect distribution, temporal distribution, domain distribution — using disaggregated counters, not aggregates alone.
- Use DataComp (Gadre et al. 2023) and DataComp-LM (Li et al. 2024) methodology to run a data-centric ablation on your pipeline — fixed model, variable data — to prove which filter contributes which eval delta.
- Emit a per-slice quality report that plugs directly into the dataset card from [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md) and the data-mixture-strategy work at `staff-ml-engineer` (higher, level 40).

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-iso-25012-and-5259-as-measurement-spec.md) | ISO/IEC 25012 and 5259-2 as the measurement spec your pipeline implements | Objective 1 (the measurement vocabulary) |
| [02](02-great-expectations-and-soda-core.md) | Great Expectations and Soda Core — row-level checks against the characteristic spec | Objective 1 (tool half 1) |
| [03](03-deequ-and-pandera.md) | AWS Deequ and pandera — scale-first and contract-first checks | Objective 1 (tool half 2) |
| [04](04-dodge-style-corpus-audit.md) | Dodge-style corpus audit — what filters strip, who, benchmark contamination | Objective 2 (audit) |
| [05](05-kreutzer-quality-at-a-glance.md) | Kreutzer "Quality at a Glance" — multilingual slice auditing | Objective 2 (multilingual) |
| [06](06-representativeness-and-disaggregated-counters.md) | Representativeness and disaggregated counters | Objective 3 |
| [07](07-datacomp-lm-ablation-methodology.md) | Data-centric ablation — DataComp and DataComp-LM methodology | Objective 4 |
| [08](08-per-slice-quality-report-handoff.md) | The per-slice quality report — handoff to dataset card and mixture strategy | Objective 5 |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-great-expectations-suite-for-one-slice.md) | Great Expectations suite for one slice | 2h |
| [02](exercises/exercise-02-iso-25012-and-5259-characteristic-coverage-check.md) | ISO/IEC 25012 and 5259-2 characteristic coverage check | 2h |
| [03](exercises/exercise-03-dodge-style-corpus-audit-on-one-slice.md) | Dodge-style corpus audit on one slice | 3h |
| [04](exercises/exercise-04-subpopulation-and-distribution-audit.md) | Subpopulation and distribution audit | 3h |
| [05](exercises/exercise-05-datacomp-lm-style-ablation-mini-run.md) | DataComp-LM style ablation mini-run | 3h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md). In particular, [`mod-101` chapter 02](../mod-101-ai-data-governance-and-position/02-iso-suite-for-training-data.md) has internalised the ISO/IEC 25012, 5259, 42001 vocabulary; you can speak about "A.7.4 quality evidence" without needing a lookup. This module is where you implement that vocabulary in code.
- You have completed [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md) and [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md). You have a real slice to audit — ideally the output of [`mod-103` Exercise 01](../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md). If not, any Parquet / JSONL dataset with ≥ 100k records works for Exercises 01–04; Exercise 05 is more comfortable at ≥ 1B tokens of post-filter text.
- Python 3.11+ with the DQ toolchain: `great_expectations`, `soda-core`, `pydeequ`, `pandera`. Spark 3.5+ for Deequ at scale.
- For Exercise 05 (ablation), a GPU workstation — minimum viable 1×A100 at 100M-parameter model scale; comfortable 8×A100 at DCLM small 160M scale.
- You have read a Dodge audit before (Dodge et al. 2021 EMNLP) and a Kreutzer audit before (Kreutzer et al. 2022 TACL). Both are open-access.

## How to use this module

Read chapter 01 before any other. It fixes the vocabulary — "characteristic", "measure", "proxy", "evidence SHA" — that every subsequent chapter and every exercise uses. The common failure mode of this module is to jump to the tools (02–03) before the vocabulary is pinned, which produces a pile of checks nobody can audit.

Then chapters 02–03 together (the four tools) and Exercise 01–02 together. The exercises reuse the same slice; do not jump between slices.

Chapters 04–06 are the audit half. Exercises 03 and 04 pair with these chapters. If your slice is English-only you can skip the Kreutzer-audit portion of Exercise 04; document that choice.

Chapter 07 and Exercise 05 are the capstone. The ablation is the most expensive deliverable in the module — plan its compute budget before you start.

Chapter 08 and the per-slice quality report ties everything together. If you have gone through the exercises in order, Chapter 08 is the hour of assembly that produces the single artifact you will actually hand off.

## Downstream handoffs

- To [`mod-105`](../mod-105-deduplication-at-scale/README.md): deduplication's own quality evidence (near-dup rate, dedup-removed documents' profile) slots into this module's representativeness report. Expect to run a Chapter-06-style audit on the dedup step's outputs.
- To [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md): the deeper contamination work. Chapter 04's exact-match contamination is the floor; mod-106 is the near-dup / paraphrase / Min-K% Prob world.
- To [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md): synthetic data has its own representativeness-vs-real audit. The ISO 5259-2 measure catalogue applies; synthetic-vs-real mixture ratio is a measure in the same shape.
- To [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): the per-slice quality report is a first-class provenance artifact. Lineage events point at it; signed manifests include it.
- To [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md): every measure emitted in this module is monitored on cadence there. The diff-stability of `per_slice_quality.yaml` is what makes that monitor's job tractable.
- To [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md): the dataset card consumes the summary numbers and audit verdicts from `per_slice_quality.yaml`.
- To [`staff-ml-engineer` (level 40)](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning): the ablation contributions table is the direct input to mixture-strategy work. Without a quality report in this shape, the slice's mixture weight defaults to zero.
