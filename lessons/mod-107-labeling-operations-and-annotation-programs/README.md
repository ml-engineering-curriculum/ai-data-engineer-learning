# mod-107 — Labeling Operations, Annotation Programs, and Preference Labeling

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 16 hours

## What this module is about

[`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) measured quality statically; [`mod-105`](../mod-105-deduplication-at-scale/README.md) and [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) cleaned the corpus. This module stands up the one piece of the pipeline where quality is **created**, not measured: the humans (and vendor labor pools) who produce supervised labels, preference pairs, and RLHF rankings.

A labeling program is not a UI choice. It is a **governed operation** with contracts, calibration sets, inter-annotator agreement thresholds, adjudication workflows, active-learning budget plans, weak-supervision force multipliers, and a labeling-guidelines document that behaves like a specification. The deliverable at the end of the module is an **annotation program package** for a real slice:

1. **Labeling guidelines** — the written contract that binds annotators, reviewers, vendors, and `fine-tuning-engineer` to the same definition of a correct label.
2. **Tool configuration** — Label Studio / Argilla / Prodigy / Doccano project files with review queues, consensus rules, and adjudication hooks pinned.
3. **Calibration set** — a small, gold-labeled seed used to qualify annotators and vendor cohorts and to detect drift on cadence.
4. **IAA report** — Cohen's kappa, Fleiss' kappa, and Krippendorff's alpha against the task structure, with task-appropriate thresholds documented and defended.
5. **Active-learning + weak-supervision strategy** — the budget plan: how many labels, selected how, amplified by what labelling functions, with what expected quality trade-off.
6. **Preference-labeling SOP** — pairwise / Likert / arena protocol for RLHF, DPO, and ORPO consumers, with cohort-drift controls.
7. **Vendor package** (if applicable) — RFP, labeling-guidelines contract, calibration set, per-item QA sample policy, quality SLA, escalation and arbitration path.

By the end you will be able to hand `fine-tuning-engineer` (peer, L30) and `model-evaluation-engineer` (peer, L30) a labeled dataset where every label is traceable to a guideline clause, an annotator identity, an IAA number, and a QA verdict — and where the budget trade-off that produced it is documented.

## Why labeling ops is an engineering discipline, not procurement

Three findings pin the framing:

- **Snow, O'Connor, Jurafsky, Ng 2008** (*Cheap and Fast — But is it Good?*, EMNLP 2008) — crowdworker labels converge to expert labels only when the aggregation, qualification, and consensus procedure is engineered. Hand-rolled MTurk pipelines routinely produce *below-chance* labels on nuanced tasks.
- **Ratner, De Sa, Wu, Selsam, Ré 2016** (*Data Programming: Creating Large Training Sets, Quickly*, NeurIPS 2016) — the quantitative framework for how noisy labeling signals can be combined into a trainable supervision signal. Labeling is a modeled, not merely-measured, process.
- **Ouyang et al. 2022** (*Training language models to follow instructions with human feedback*, NeurIPS 2022) — InstructGPT. The preference-labeling program is an explicit operational input to model training; label-pool divergence directly produces reward-model miscalibration downstream.

Chapter 01 is the defence. Do not skip it.

## Learning objectives

- Run production annotation programs in **Label Studio**, **Argilla**, **Prodigy**, and **Doccano** — project configuration, annotator UI, review queues, consensus, adjudication; know which tool fits which modality (text / image / audio / preference pairs / tool-calling trajectories).
- Contract and audit vendor labelling — **Scale AI**, **Surge AI**, **Appen**, **Sama**, **iMerit**, **Toloka**, **SuperAnnotate** — including the vendor-RFP shape, the written labelling-guidelines contract, the calibration set, the per-item QA sample, the vendor-side quality SLA, and the escalation-and-arbitration process.
- Compute inter-annotator agreement correctly — **Cohen's kappa** (two annotators, two classes), **Fleiss' kappa** (multiple annotators), **Krippendorff's alpha** (arbitrary categories and missing values) — and set realistic agreement thresholds per task difficulty.
- Budget labelling with **active learning** (Settles 2009, *Active Learning Literature Survey*) — pool-based, stream-based, and query-by-committee; prove the budget / quality trade-off on a real task.
- Amplify human labels with **weak supervision** and **data programming** — **Snorkel** (Ratner et al. 2016) for labelling functions, label-model training, and discriminative-model training; know where weak supervision is a force multiplier and where it is a hidden-quality loss.
- Run a **preference-labelling program** for RLHF / DPO / ORPO at operations quality — pairwise comparison UI, Likert / fine-grained rubric, calibration drift between cohorts, chatbot-arena-style protocols — handing off the preference-pair curation depth to `fine-tuning-engineer` (peer, L30).
- Coordinate with `model-evaluation-engineer` (peer, L30) on **gold-eval-set labelling methodology** (judge-vs-human calibration, statistical stopping rules) — this module covers production ops; benchmark-construction depth is linked out.

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-labeling-ops-as-an-engineering-program.md) | Labeling ops as an engineering program | The framing / deliverable |
| [02](02-open-source-annotation-tool-landscape.md) | Open-source annotation tool landscape — Label Studio, Argilla, Prodigy, Doccano | Objective 1 (tool-to-modality mapping) |
| [03](03-project-configuration-review-consensus-adjudication.md) | Project configuration, review queues, consensus, and adjudication | Objective 1 (workflow half) |
| [04](04-inter-annotator-agreement-kappa-and-krippendorff.md) | Inter-annotator agreement — Cohen, Fleiss, Krippendorff | Objective 3 |
| [05](05-vendor-labelling-rfp-calibration-sla-arbitration.md) | Vendor labelling — RFP, calibration set, QA sample, SLA, arbitration | Objective 2 |
| [06](06-active-learning-for-budget-efficiency.md) | Active learning for budget efficiency — Settles 2009 | Objective 4 |
| [07](07-weak-supervision-and-data-programming-snorkel.md) | Weak supervision and data programming — Snorkel | Objective 5 |
| [08](08-preference-labelling-for-rlhf-dpo-orpo.md) | Preference-labelling programs for RLHF, DPO, and ORPO | Objective 6 |
| [09](09-gold-eval-coordination-with-model-evaluation-engineer.md) | Gold-eval-set coordination with `model-evaluation-engineer` | Objective 7 |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-label-studio-and-argilla-project-configuration.md) | Label Studio and Argilla project configuration | 2h |
| [02](exercises/exercise-02-vendor-rfp-and-vendor-audit-authoring.md) | Vendor RFP and vendor-audit authoring | 3h |
| [03](exercises/exercise-03-iaa-computation-with-kappa-and-krippendorff.md) | IAA computation with kappa and Krippendorff | 2h |
| [04](exercises/exercise-04-active-learning-budget-vs-quality-run.md) | Active learning budget-vs-quality run | 3h |
| [05](exercises/exercise-05-snorkel-weak-supervision-amplifier.md) | Snorkel weak-supervision amplifier | 3h |
| [06](exercises/exercise-06-preference-labelling-program-for-dpo-pairs.md) | Preference-labelling program for DPO pairs | 3h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) — the ISO/IEC 25012 vocabulary and the governance framing. Labeling ops is the one stage in the pipeline where ISO *accuracy* and *credibility* are **produced**, not measured.
- You have completed [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) — labeling programs are the mechanism that converts an unlabeled slice into a quality-reported labeled slice.
- You have a slice to label — ideally from [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md) Exercise 01, or any text / image / tool-calling trajectory dataset with at least a few thousand items. Exercise 04 benefits from a small (≤ 50k) classification dataset where ground truth is available on a held-out subset.
- Python 3.11+ with: `label-studio-sdk`, `argilla`, `pandas`, `pyarrow`, `scikit-learn`, `nltk` or `statsmodels` (for Cohen / Fleiss), `krippendorff`, `snorkel`, `small-text` or `modAL` (active learning). Exercise 06 benefits from a locally runnable LLM via `vllm`, `ollama`, or an API key.
- You have read Settles 2009 (*Active Learning Literature Survey*) at least once. The three sampling families pin everything downstream.

## How to use this module

Chapter 01 is the frame. It is what you cite when a product manager says "just ship it to Scale and get labels back". The rest of the module turns that frame into operating procedure.

Chapters 02 and 03 pair with Exercise 01. They are the practical half — stand up a project, wire up review queues, run a small session end-to-end — and they anchor every later tooling decision.

Chapter 04 pairs with Exercise 03. IAA is the single most-often-skipped step in production labeling programs, and the single most common source of silent quality failure later. Do not skip Exercise 03.

Chapter 05 pairs with Exercise 02. The RFP + audit shape is what you deliver to Legal and Procurement; writing it once in training makes the real one much faster.

Chapters 06 and 07 are the two force multipliers. Chapter 06 (active learning) targets the right items; Chapter 07 (weak supervision) amplifies the labels you already have. Exercises 04 and 05 pair with them.

Chapter 08 is the RLHF / DPO / ORPO operations chapter — the piece that `fine-tuning-engineer` cares about most. Exercise 06 pairs with it. Keep the hand-off clean: you own the operations; `fine-tuning-engineer` owns the fine-tune-time curation.

Chapter 09 is a short coordination chapter — the gold-eval-set interface with `model-evaluation-engineer`. It has no paired exercise; `model-evaluation-engineer` owns the benchmark-construction depth.

## Downstream handoffs

- To [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): every label event is a lineage event. The guidelines document, calibration set, IAA report, adjudication transcript, and QA verdict table are first-class provenance records.
- To [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md): IAA, QA-sample pass rate, and cohort-drift metrics are monitored on cadence. A sudden drop is the labeling-program equivalent of a dedup anomaly from `mod-105`.
- To [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md): the labeling-program package feeds the dataset card — annotator population, guideline version, IAA, QA policy, vendor sign-off.
- To `fine-tuning-engineer` (peer, L30): preference pairs, SFT labels, and RLHF rankings land here. The hand-off contract is preference-pair format + cohort metadata + calibration-drift report, not raw clicks.
- To `model-evaluation-engineer` (peer, L30): gold-eval-set labeling methodology (Chapter 09). You own the labeling SOP; they own the benchmark design and the statistical stopping rules.
- To `ai-risk-engineer` (peer, L25): annotator-population PII and safety labeling have shared surface with the privacy-risk report in [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md). You own the labeling policy; they own the risk acceptance.
