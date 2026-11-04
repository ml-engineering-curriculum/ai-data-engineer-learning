# mod-110 — PII Detection, Redaction, and Privacy Risk for Training Data

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 12 hours

## What this module is about

[`mod-105`](../mod-105-deduplication-at-scale/README.md) cleaned a slice on the duplication axis. This module cleans it on the **confidentiality** axis — PII detection, redaction, and the privacy-risk accounting that must travel with the slice. The deliverable at the end of the module is a signed `privacy_risk_report.yaml` that, for each slice, captures:

1. **Detection policy** — which recognisers (regex / dictionary / ML) at which thresholds, in which pipeline stage.
2. **Redaction policy** — which primitive (mask / tokenise / surrogate / FPE / hash) per entity type, and the training-utility consequence of each choice.
3. **Residual PII accounting** — what the detector found, what survived after redaction, and the precision / recall of the detector on an authored eval set.
4. **Memorisation-risk delta** — the Kandpal-2022 frequency-to-memorisation estimate, consumed directly from the paired `dedup_report.yaml`.
5. **Lawful-basis evidence package** — the engineering evidence that counsel needs for GDPR Art 6 / 9 / 14, CCPA / CPRA, and HIPAA de-identification decisions. We do not produce the legal opinion; we produce the evidence.
6. **Audit evidence SHAs and a Sigstore signature** — every intermediate artifact hashed; the report itself signed.

By the end you will be able to hand a privacy counsel or an [`ai-risk-engineer`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) peer a `privacy_risk_report.yaml` and have them know — within five minutes of reading it — what PII policy you ran, what survived, and how much memorisation surface is left for the training team to defend against.

## Why pipeline-side PII work is a load-bearing control

Three peer-reviewed findings anchor the framing. Chapter 01 defends them; this list is the summary:

- **Carlini et al. 2020 / USENIX Security 2021** (*Extracting Training Data from Large Language Models*) — real PII is extractable from production-scale LMs with black-box queries. The attack is a measured engineering phenomenon, not a theoretical worry.
- **Carlini et al. 2022 / ICLR 2023** (*Quantifying Memorization Across Neural Language Models*) — memorisation scales log-linearly with duplicate count, with model size, and with prompt length. The pipeline controls the first of the three.
- **Kandpal et al. 2022 / ICML 2022** (*Deduplicating Training Data Mitigates Privacy Risks*) — deduplication alone drops extraction-attack success rates by roughly an order of magnitude. Dedup (owned in [`mod-105`](../mod-105-deduplication-at-scale/README.md)) is a privacy control, not just a quality control.

The data-pipeline-side defences are: **scrub what you can detect, dedup aggressively, quarantine high-risk documents, and emit the audit evidence**. Model-side defences such as DP-SGD, output filters, and attack-simulation tooling are explicitly out of scope — see [Downstream handoffs](#downstream-handoffs).

## Learning objectives

- Engineer PII detection + redaction pipelines with **Microsoft Presidio** (recognisers, context-aware analysis), **scrubadub**, **Google Cloud Sensitive Data Protection (DLP)**, and **AWS Comprehend PII** — combining regex, dictionary, and ML-based recognisers; integrate into the ingest, dedup, and publish stages of the pipeline.
- Choose the right redaction primitive per use case — **masking**, **tokenisation**, **surrogate generation**, **format-preserving redaction**, **hashing** — knowing which preserves training utility and which breaks it.
- Reason about lawful basis for training-data use — **GDPR** Articles 6 (lawful basis), 9 (special categories), 14 (transparency when data not collected from subject); **CCPA / CPRA** 'sale or sharing' obligations; **HIPAA** de-identification (Safe Harbor + Expert Determination) for clinical data — routing legal opinion to counsel and packaging the engineering evidence here.
- Defend against the **Carlini et al. 2020** 'Extracting Training Data' attack and the **Carlini et al. 2022** 'Quantifying Memorization' failure mode — pipeline-side controls (PII scrub, dedup per [`mod-105`](../mod-105-deduplication-at-scale/README.md), high-risk-document quarantine) that reduce memorisation risk.
- Reason about the **Kandpal et al. 2022** result — deduplication is a load-bearing privacy control. Coordinate with [`mod-105`](../mod-105-deduplication-at-scale/README.md) on the pipeline wiring.
- Hand off DP-SGD training depth to `fine-tuning-engineer` (peer, level 30) and `ai-risk-engineer` (peer, level 25). This module covers the data-pipeline-side defences only.

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-why-pii-scrubbing-is-a-pipeline-privacy-control.md) | Why PII scrubbing is a pipeline privacy control | The defence (Carlini / Kandpal) |
| [02](02-detection-with-microsoft-presidio.md) | Detection with Microsoft Presidio — Analyzer, recognisers, context | Objective 1 (Presidio depth) |
| [03](03-detection-with-scrubadub-and-cloud-dlp.md) | Detection with scrubadub, Google DLP, AWS Comprehend PII | Objective 1 (toolchain breadth) |
| [04](04-combining-recogniser-families-and-evaluation.md) | Combining regex, dictionary, and ML recognisers — and evaluating them | Objective 1 (ensemble + eval) |
| [05](05-redaction-primitives-and-training-utility.md) | Redaction primitives — mask, tokenise, surrogate, FPE, hash | Objective 2 |
| [06](06-pipeline-integration-ingest-dedup-publish.md) | Pipeline integration — ingest, dedup, publish | Objective 1 (pipeline wiring) + Objective 5 (Kandpal) |
| [07](07-memorisation-attacks-and-pipeline-defences.md) | Memorisation attacks and pipeline-side defences | Objective 4 + Objective 6 (handoff) |
| [08](08-lawful-basis-gdpr-ccpa-hipaa.md) | Lawful basis for training data — GDPR, CCPA / CPRA, HIPAA | Objective 3 |
| [09](09-the-privacy-risk-report.md) | The privacy-risk report as audit evidence | The deliverable |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-presidio-pii-pipeline-at-corpus-scale.md) | Presidio PII pipeline at corpus scale | 2h |
| [02](exercises/exercise-02-redaction-primitive-choice-per-use-case.md) | Redaction primitive choice per use case | 2h |
| [03](exercises/exercise-03-gdpr-and-hipaa-lawful-basis-authoring.md) | GDPR and HIPAA lawful-basis evidence authoring | 3h |
| [04](exercises/exercise-04-carlini-style-extraction-probe-on-pretrain-slice.md) | Carlini-style extraction probe on a pretrain slice | 3h |
| [05](exercises/exercise-05-privacy-risk-report-for-one-corpus-slice.md) | Privacy-risk report for one corpus slice | 1h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md). The ISO/IEC 25012 *confidentiality* characteristic and the ISO/IEC 27701 privacy-information-management vocabulary are the frames this module uses.
- You have completed [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md). Lawful-basis reasoning (Chapter 08) builds on the sourcing provenance from that module.
- You have completed [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md). The privacy-risk report sits alongside the quality report and shares the per-slice identity / provenance fields.
- You have completed [`mod-105`](../mod-105-deduplication-at-scale/README.md). The memorisation-risk delta (Chapter 09 §4) consumes the duplicate-frequency distribution from the dedup report.
- Python 3.11+ with: `presidio-analyzer`, `presidio-anonymizer`, `spacy` (and `en_core_web_lg` or `en_core_web_trf`), `scrubadub`, `pyarrow`, `pandas`, `pyyaml`. Exercise 01 additionally needs the `transformers` recogniser (`presidio-analyzer[transformers]`). Exercise 04 needs an HF `transformers` runtime capable of running a small open LM (e.g. a 1B–7B parameter model).
- A cloud account — GCP for DLP, AWS for Comprehend PII — is **optional**. Exercises can be completed with Presidio + scrubadub alone; the cloud-DLP chapters are read-and-reason, not required-to-run.
- You have read Carlini et al. 2020 and Kandpal et al. 2022 at least once. The *why* collapses without them.

## How to use this module

Read Chapter 01 first. It is the justification you will cite in every design review for the rest of the module.

Chapters 02 and 03 together cover the detection toolchain. Pair Chapter 02 with Exercise 01 — building a corpus-scale Presidio pipeline is the primary engineering deliverable.

Chapter 04 is the evaluation craft. Do not skip the eval-set authoring: PII detectors shipped without a labelled eval set have unknown precision and recall, and unknown-recall detectors do not count as audit evidence.

Chapter 05 pairs with Exercise 02. The "which primitive preserves training utility?" question is the one you will be asked by the model team; write out the answer per entity type.

Chapter 06 is the pipeline-integration layer. It is where the [`mod-105`](../mod-105-deduplication-at-scale/README.md) coordination lives — specifically, the Kandpal 2022 result that dedup is a privacy control is operationalised here.

Chapter 07 is the attack literacy. The pipeline cannot stop everything; this chapter is the honest accounting of what it can and cannot defend against, and where the boundary with `fine-tuning-engineer` / `ai-risk-engineer` sits. Pair it with Exercise 04.

Chapter 08 is the regulatory literacy. Do not skip it. You will not produce legal opinions; you will produce the engineering evidence that counsel needs to form one. Pair it with Exercise 03.

Chapter 09 assembles the report. Exercise 05 is the one-hour assembly against your own slice; by the time you reach it, Exercises 01–04 have each contributed a block.

## Downstream handoffs

- To `fine-tuning-engineer` (peer, level 30): DP-SGD, output filters, and training-side differential privacy are their craft. Our handoff is the `privacy_risk_report.yaml` with residual PII counts, memorisation-risk delta, and high-risk-document quarantine manifest. A slice without a privacy-risk report is not training-ready.
- To `ai-risk-engineer` (peer, level 25): attack-simulation tooling (membership inference, extraction attacks on candidate models) and adversary-mindset threat modelling. We provide the pipeline-side signals (detection coverage, redaction audit trail, quarantine bucket); they own the detectors and the simulation harness.
- To `privacy counsel` (role outside this curriculum): the lawful-basis engineering evidence from Chapter 08 and Exercise 03. We package; they decide.
- To [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): the privacy-risk report is a first-class lineage event. Sigstore signature and artifact SHAs are required.
- To [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md): per-source residual-PII rate and detector-drift metrics are monitored on cadence. A sudden jump is a signal for either regression or a new data source with a different PII distribution.
- To [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md): the dataset card consumes the detection policy, redaction policy, residual PII table, and the counsel sign-off from `privacy_risk_report.yaml`.

## Upstream dependencies

- From [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md): the sourcing provenance (ToS, licence, jurisdiction) for every document. Lawful-basis reasoning is impossible without it.
- From [`mod-105`](../mod-105-deduplication-at-scale/README.md): the duplicate-frequency distribution and the memorisation-risk delta. The privacy-risk report cites `dedup_report.yaml` by SHA.
