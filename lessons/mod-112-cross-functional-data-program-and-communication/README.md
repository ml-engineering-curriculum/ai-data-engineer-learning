# mod-112 — Cross-Functional Data Program, Dataset Cards, and Communication

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 12 hours

## What this module is about

Modules 101 through 111 teach you to *build* the training-data pipeline. This module teaches you to **run the program around it** — the cadence on which it reports, the artifacts it ships to stakeholders outside the engineering team, the decisions it routes for sign-off, and the public documents it produces for governance, customers, and regulators.

Running the program is a distinct job from running the pipeline. If you only ship the pipeline, every release is a scramble to answer the same five questions in five different document shapes; the AI governance council signs off blind; the model team pins to a slice without the evidence they need to defend the pin; the EU AI Office waits for a public summary that nobody drafted; the model card goes out with a hand-wavy data section. If you ship the program, your release cycle is pipeline work; everything else renders from the evidence base you maintain.

The module covers five surfaces:

1. **The operating rhythm** — the meetings, the cadences, the decision log, and the briefings that make the program legible to the AI governance council, the data council, and the model/product team.
2. **Dataset cards in three formats** — the Hugging Face Dataset Card for engineers, the Google Data Cards Playbook form for stakeholder reviewers, the Meta/Llama-style disclosure for external audiences — all rendered from one evidence base.
3. **The EU AI Act Article 53 + Annex XI public training-data summary** — the specific regulator-facing document a GPAI provider ships alongside its model release.
4. **Contributions to model cards and system cards** — what to supply, in what shape, keyed to which SHA, when the model team or system team authors a card that cross-references your evidence.
5. **Cross-role coordination and certifications** — the four handoff relationships (governance analyst L15, risk engineer L25, evaluation engineer L35, staff ML engineer L40) and the certifications portfolio (CDMP, CIPP, AIGP, ISO/IEC 42001 LI) that position the role in enterprise hiring.

## Learning objectives

- Own an AI training-data program at team scope across ML / infra / security / governance / legal — cadence, artifacts, decisions, sign-off routing; brief the AI governance council and the data council with a pre-read, one-pager, and decision log.
- Author dataset cards end-to-end in the Hugging Face Dataset Card format, the Google Data Cards Playbook form, and the Meta Data Cards / Llama-style disclosure form — knowing which format fits which audience and how the three cross-reference the underlying Datasheet.
- Draft the EU AI Act Article 53 + Annex XI training-data summary for a general-purpose AI provider — the exact public-facing document a GPAI releases alongside its model to satisfy the Code of Practice and Annex XI obligations.
- Contribute the data slice to a model card (Mitchell et al. 2019 / Hugging Face model card format) and the dataset slice to a system card — covering data sources, licensing, collection, representativeness, known biases, PII scrub, contamination log, and the lineage-traced snapshot hash.
- Coordinate with `ai-governance-analyst` (L15) on the inventory / impact-assessment legwork they draft on top of this evidence, with `ai-risk-engineer` (peer, L25) on data-side harm modelling and the risk register slice, with `ai-evaluation-engineer` (next-up, L35, Governance family) on the release-assurance package this evidence feeds into, and with `staff-ml-engineer` (L40) on the foundation-model data-mixture strategy that consumes per-source datasets this role publishes.
- Position adjacent certifications — DAMA CDMP (data management), IAPP CIPP/E or CIPP/US (privacy), IAPP AIGP (AI governance), ISO/IEC 42001 lead-implementer — and know which are load-bearing in enterprise hiring for data-engineering-for-AI roles.

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-running-the-training-data-program.md) | Running the training-data program at team scope | Objective 1 (operating rhythm) |
| [02](02-briefing-the-councils.md) | Briefing the AI governance council and the data council | Objective 1 (pre-read, one-pager, decision log) |
| [03](03-dataset-card-format-landscape.md) | The dataset-card format landscape and the underlying Datasheet | Objective 2 (format picking) |
| [04](04-hugging-face-dataset-card.md) | The Hugging Face Dataset Card end-to-end | Objective 2 (HF format) |
| [05](05-google-data-cards-playbook.md) | The Google Data Cards Playbook form | Objective 2 (Google format) |
| [06](06-meta-data-cards-and-llama-style-disclosure.md) | Meta Data Cards and Llama-style disclosure | Objective 2 (Meta format) |
| [07](07-eu-ai-act-annex-xi-public-summary.md) | The EU AI Act Article 53 + Annex XI public training-data summary | Objective 3 |
| [08](08-model-card-and-system-card-data-slices.md) | The model-card data slice and the system-card dataset slice | Objective 4 |
| [09](09-cross-role-coordination.md) | Cross-role coordination — governance analyst, risk engineer, evaluation engineer, staff ML engineer | Objective 5 |
| [10](10-adjacent-certifications.md) | Adjacent certifications — CDMP, CIPP, AIGP, ISO/IEC 42001 LI | Objective 6 |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-data-program-charter-for-one-team.md) | Data program charter for one team | 2h |
| [02](exercises/exercise-02-dataset-card-across-three-formats.md) | Dataset card across three formats, rendered from one evidence base | 3h |
| [03](exercises/exercise-03-eu-ai-act-annex-xi-training-data-summary-draft.md) | EU AI Act Annex XI training-data summary draft | 2h |
| [04](exercises/exercise-04-model-card-data-slice-authoring.md) | Model-card data slice and system-card dataset slice authoring | 2h |
| [05](exercises/exercise-05-certifications-portfolio-planner.md) | Certifications portfolio planner | 1h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md). The EU AI Act Articles 10 / 53 / 72, the NIST AI RMF, the ISO/IEC 42001 and 23894 frames, GDPR / DSM Art 4, and the role-ladder context are all referenced from this module without re-derivation.
- You have completed at least one of [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md), [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md), [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md), [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md). Each supplies an artifact the dataset cards and the Article 53 summary consume (quality report, lineage trace, privacy risk report, monitoring state).
- You have a slice to document. Reuse the output of an earlier module's exercise — any Parquet / JSONL slice with a `snapshot.yaml` or `source_manifest.yaml` will do. If you have nothing ready, Exercise 02's starter guidance walks through constructing a minimal evidence base from scratch.
- Python 3.11+ with: `pyyaml`, `jsonschema`, `jinja2` (for the multi-format rendering pattern), `huggingface_hub` (for HF card validation). No special hardware or accounts required.
- Optional but recommended: a repository where you can commit the five documents and the evidence base as versioned artifacts (private GitHub, GitLab, or Gitea are all fine).

## How to use this module

Read Chapter 01 first and keep the operating rhythm in front of you for the rest of the module — every later chapter produces an artifact that flows through that rhythm.

Chapter 02 is the authoring craft for council communication; the exercise that depends most on it is Exercise 01 (data-program charter).

Chapters 03–06 are the dataset-card quartet. Read Chapter 03 before 04–06 — the format landscape makes the per-format chapters easier to internalise. Pair with Exercise 02; render the same evidence base into all three formats and read the three outputs side-by-side.

Chapter 07 is the regulatory step. Read it after the dataset-card chapters so the distinction between a Data Card and an Article 53 public summary is sharp. Pair with Exercise 03.

Chapter 08 is the contribution-to-other-roles'-documents chapter. Pair with Exercise 04.

Chapter 09 is the coordination layer. Read it before every major release; use it as a check against whether each of the four downstream handoffs is in good standing.

Chapter 10 is the certifications planner. Pair with Exercise 05; even if you do not act immediately, the exercise forces an explicit career move the next time a hiring decision or role transition is in play.

## Downstream handoffs

- To [`ai-governance-analyst`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning) (L15, AI Governance): dataset cards, lineage attestation, preparation log, decision log, monitoring state. They consume all of these into the inventory entry, the DPIA / impact assessment, the risk register, and the regulator submission. Chapter 09 § 1.
- To `ai-risk-engineer` (peer, L25, AI Governance): threat-surface map, detector precision/recall, synthetic-data provenance graph, lineage. They consume these into the data-side risk register slice and the table-top exercises. Chapter 09 § 2.
- To [`ai-evaluation-engineer`](https://github.com/ml-engineering-curriculum/ai-evaluation-engineer-learning) (L35, AI Governance): the release-assurance data package (dataset cards + contamination report + privacy risk report + bias audit + lineage + monitoring state + retrainability statuses). They consume this into the release-assurance report that the governance council signs off on. Chapter 09 § 3.
- To `staff-ml-engineer` (L40, ML Engineering): the slice catalog (per-slice retrainability + composition + effective-tokens). They consume this into the foundation-model data-mixture strategy for the next-generation training run. Chapter 09 § 4.
- To the governance council and the data council: the monthly pre-read, the release one-pager, the quarterly data-council report. Chapter 02.
- To the public and to the AI Office: the Article 53(1)(d) public training-data summary and the Annex XI technical-documentation package (via legal / governance). Chapter 07.
- To the model team and the system team: the model-card data-slice contribution and the system-card dataset-slice contribution. Chapter 08.

## Upstream dependencies

- From [`mod-101`](../mod-101-ai-data-governance-and-position/README.md): the governance frameworks (NIST AI RMF, EU AI Act, GDPR, DSM Art 4, ISO 42001 / 23894 / 25012 / 5259, sector regimes) that the dataset cards, Article 53 summary, model-card contribution, and risk register slice all cite.
- From [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md): source manifests, licence posture matrix, admission function — Section 3 of the Article 53 summary and the HF card's licensing-information section.
- From [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md): TLD and crawl-process details — Section 2 of the Article 53 summary and the Google card's Provenance section.
- From [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md): `per_slice_quality.yaml` — Validation Types section on the Google card and Section 6 of the Article 53 summary.
- From [`mod-105`](../mod-105-deduplication-at-scale/README.md): `dedup_report.yaml` — Preprocessing sections on all three Data Card formats.
- From [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md): `contamination_report.yaml` — Contamination Log (HF), Known Applications and Benchmarks (Google), Contamination Posture (Meta), Section 6 (Article 53), model-card Section 5.
- From [`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md): annotator-pool composition, IAA, labelling methodology — Annotations and Labeling section on the Google card, § 8 of the model card.
- From [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md): synthetic-data-generator admission log and provenance — Source-class line on all three Data Card formats and § 6.3 of the model card contribution.
- From [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): snapshot SHA, OpenLineage run id, Sigstore signature — the Lineage section on all three Data Card formats and the primary key of the Article 53 package.
- From [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md): `privacy_risk_report.yaml` — Personal/Sensitive sections on all three Data Card formats and Section 5 of the Article 53 summary.
- From [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md): `dataset_monitoring_contract.yaml` and the monitoring-state sidecar — the Retrainability-status field across all card formats and the release-assurance data package handed to the evaluation engineer.
