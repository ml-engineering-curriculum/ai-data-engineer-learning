# mod-111 — Data Quality Monitoring and Regression Detection in Production

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 12 hours

## What this module is about

[`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) taught you to measure the quality of a slice at publish time. [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) taught you to sign and pin the version of that slice. [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md) taught you to attach a privacy-risk report to it. **This module is the layer that watches those artifacts across *refreshes*.**

A slice does not stay correct by itself. Upstream sources change, licensing posture shifts, filters get retuned, enum domains expand, Pandas silently promotes `int64` to `float64`. Every refresh of the pipeline is a point at which a dataset-side regression can be introduced — and if it is introduced silently, the first signal that reaches the business is a model-output regression weeks later, with the training-data pipeline nowhere in the incident narrative.

The deliverable at the end of this module is a signed `dataset_monitoring_contract.yaml` per pipeline that captures:

1. **Which metrics are watched, at which cadence, with which threshold, by which owner, through which route.** Covariate drift, schema drift, population-balance drift, residual-PII rate, benchmark contamination, dedup-rate — one row per monitored observable.
2. **Which severity each metric carries.** `publish_blocking`, `retrain_blocking`, or `advisory` — and the operational commitment each entails.
3. **Which reference each metric compares against.** A pinned golden slice, a rolling window, or both.
4. **Which regression fixtures CI-gate the pipeline.** Every past incident becomes a fixture; the fixture library is a cumulative asset.
5. **Which substrate runs the metrics** — `ai-infra-mlops` as the platform owner for the scheduler, metric store, alert router, and dashboards. **You** are the metric-authoring layer.
6. **Which handoff boundaries sit on top** — model-output monitoring to `ai-eval-engineer` / `model-evaluation-engineer`; governance synthesis to `ai-governance-analyst`.

The contract is signed and attached to the dataset version-pin so a retraining decision has a defensible dataset-side justification.

## Why dataset-side monitoring is distinct from the three adjacent surfaces

Four different teams in the org ship "monitoring" and all four show up in incident reviews. Chapter 01 draws the lines. The one-sentence summary of each:

- **Model-output monitoring** (`ai-eval-engineer` + `model-evaluation-engineer`, both L30). Watches the trained model's behaviour in production — judges, offline evals, A/B harness. Boundary: anything downstream of training.
- **Classical data-warehouse monitoring** (the data platform team, not on this ladder). Watches warehouse-table freshness, volume, business-table schema. Warehouse-first by design.
- **Monitoring substrate** (`ai-infra-mlops`, L25 peer). Watches the monitoring system: scheduler uptime, alert-router SLAs, dashboard performance. The platform the metrics live on.
- **Dataset-side monitoring** (this module). The metric-authoring layer attached to the training slice's version-pin.

Chapter 04 is explicit about where the general warehouse-first platforms (Monte Carlo, Soda Cloud, Metaplane, Bigeye) fit — as warehouse-side watchers, as alerting substrate, and as lineage consumers — and about where the AI-pipeline surface carries obligations those platforms do not cover natively.

## Learning objectives

- Design the **dataset-side monitoring contract** — which metric feeds which alert, at what cadence, routing to which owner — distinct from model-output monitoring, and distinct from classical data-warehouse quality monitoring.
- Use **whylogs (WhyLabs)**, **Evidently AI**, **Monte Carlo**, **Soda Cloud**, **Metaplane**, and **Bigeye** as the working toolset — knowing where their coverage overlaps and where the data-pipeline-for-AI slice adds obligations the general data-observability tools do not handle out of the box.
- Measure **data drift** (distribution shift at the input level), **schema drift** (fields or dtypes change), and **population drift** (subpopulation balance shifts) on recurring data refreshes; attach alerts to the dataset version-pin so a model re-training decision has a defensible dataset-side justification.
- Author **regression fixtures** — minimal reproducible slices that re-run on every pipeline change, with a pass/fail check — turning dataset-quality incidents into permanent guards.
- Hand off the orchestrator / scheduler / observability platform engineering to [`ai-infra-mlops`](https://github.com/ml-engineering-curriculum) (peer, L25); this module covers the dataset-metric authoring layer, not the platform substrate.
- Hand off the model-output monitoring and application-eval online loop to [`ai-eval-engineer`](https://github.com/ml-engineering-curriculum/ai-eval-engineer-learning) and [`model-evaluation-engineer`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peers, L30).

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-the-dataset-side-monitoring-contract.md) | The dataset-side monitoring contract | Objective 1 (the deliverable + the distinctions) |
| [02](02-whylogs-and-whylabs.md) | whylogs and WhyLabs — per-slice approximate profiles | Objective 2 (whylogs, WhyLabs) |
| [03](03-evidently-ai.md) | Evidently AI — reports, test suites, reference-vs-current | Objective 2 (Evidently) |
| [04](04-monte-carlo-soda-metaplane-bigeye.md) | Monte Carlo, Soda Cloud, Metaplane, Bigeye — where general tools fit and where they don't | Objective 2 (general-tool landscape) |
| [05](05-data-drift-measurement.md) | Measuring data drift at the input level | Objective 3 (data drift) |
| [06](06-schema-drift-and-rollback.md) | Schema drift and the rollback workflow | Objective 3 (schema drift) |
| [07](07-subpopulation-balance-drift.md) | Subpopulation balance drift | Objective 3 (population drift) |
| [08](08-regression-fixtures.md) | Regression fixtures — turning incidents into permanent guards | Objective 4 |
| [09](09-alert-routing-version-pin-handoffs.md) | Alert routing, version-pin attachment, and role boundaries | Objectives 5, 6 (handoffs) |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-dataset-monitoring-contract-for-one-pipeline.md) | Author `dataset_monitoring_contract.yaml` for one pipeline | 2h |
| [02](exercises/exercise-02-whylogs-and-evidently-side-by-side-on-one-dataset.md) | whylogs and Evidently side-by-side on one dataset | 3h |
| [03](exercises/exercise-03-schema-drift-alert-and-rollback-workflow.md) | Schema-drift alert and rollback workflow | 2h |
| [04](exercises/exercise-04-subpopulation-balance-drift-detection.md) | Subpopulation balance drift detection | 2h |
| [05](exercises/exercise-05-regression-fixture-authoring-for-one-pipeline.md) | Regression-fixture authoring for one pipeline | 2h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md). The NIST AI RMF MEASURE / MANAGE sub-categories and the EU AI Act Article 72 post-market-monitoring hook are the governance frames this module slots into.
- You have completed [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md). The per-slice quality report is the point-in-time version of what mod-111 monitors on cadence; chapters here build on the Great Expectations / Soda Core / pandera idiom from mod-104 Chapter 02.
- You have completed [`mod-105`](../mod-105-deduplication-at-scale/README.md), [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md), and [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md). Several example monitors in this module watch outputs from those modules (dedup rate, benchmark contamination hits, residual-PII rate).
- You have completed [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md). Chapter 09's version-pin attachment is Chapter-02/03/08 of mod-109 in action.
- Python 3.11+ with: `whylogs`, `evidently`, `pandera`, `great_expectations`, `pyarrow`, `pandas`, `scipy`, `pyyaml`, `pytest`. Exercise 02 additionally needs `sentence-transformers` for the embedding-drift stretch goal.
- Accounts for at least one SaaS data-observability platform (WhyLabs, Evidently Cloud, Monte Carlo, Soda Cloud, Metaplane, or Bigeye) are **optional** — exercises can be completed entirely on OSS runtimes.
- A slice to monitor. Reuse the mod-104 / mod-105 output if you have it; otherwise any Parquet / JSONL dataset with at least 3 refreshes and at least one categorical column (e.g. `language`, `source_family`).

## How to use this module

Read Chapter 01 first and keep the `dataset_monitoring_contract.yaml` schema in front of you for the rest of the module — every chapter adds detail to one of its fields.

Chapters 02 and 03 together cover the ML-aware tools. Pair them with Exercise 02 — running `whylogs` and Evidently side-by-side on the same slice is where the tools' complementarity becomes concrete.

Chapter 04 is a field guide to the general data-observability platforms. Read it to know what *not* to re-author and to know where the warehouse-first tools stop and your work begins.

Chapters 05–07 are the three drift axes. Pair Chapter 05 with Exercise 02; pair Chapter 06 with Exercise 03 (the rollback workflow is best learned by actually running it); pair Chapter 07 with Exercise 04.

Chapter 08 is the regression-fixture craft. The habit — one fixture per resolved incident — is the one that compounds over the lifetime of the pipeline. Pair it with Exercise 05.

Chapter 09 is the operational layer. The three handoff boundaries it draws are the ones you walk every design review with for the rest of your tenure.

## Downstream handoffs

- To [`ai-infra-mlops`](https://github.com/ml-engineering-curriculum) (peer, L25): `dataset_monitoring_contract.yaml` + the `severity_routing:` preamble. They build the scheduler, metric store, alert-manager rules, routing to PagerDuty / Slack, PR-comment integration, sidecar append pipeline, and the signing of the sidecar. See Chapter 09.
- To [`ai-eval-engineer`](https://github.com/ml-engineering-curriculum/ai-eval-engineer-learning) and [`model-evaluation-engineer`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peers, L30): the version-pin of a slice that passed publish-blocking severities, with its monitoring artifacts and the post-publish sidecar. They own the model-output monitoring, judges, offline evals, A/B harness — they *consume* your trusted slice, they do not re-derive dataset-side signals. See Chapter 09.
- To `ai-governance-analyst` (L15): periodic exports of the monitoring state keyed by pipeline id and version-pins, for NIST AI RMF MEASURE / MANAGE evidence and EU AI Act Article 72 post-market-monitoring reports. See Chapter 09.
- To [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): monitoring artifacts are first-class items in the slice snapshot; alert annotations append to a sidecar keyed by the version-pin SHA. Signed with Sigstore.
- To [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md): the dataset card consumes the monitoring artifacts at publish-time and the sidecar alerts after publish; the retrainability-status field is driven by them.

## Upstream dependencies

- From [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md): `per_slice_quality.yaml` — the point-in-time measure record whose rows become monitored metrics.
- From [`mod-105`](../mod-105-deduplication-at-scale/README.md): `dedup_report.yaml` — duplicate-count distribution per source family, monitored as `kind: quality`.
- From [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md): `contamination_report.yaml` — n-gram overlap and Min-K% Prob results, monitored as publish-blocking quality.
- From [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): the dataset version-pin SHA — the key under which every monitoring artifact is filed.
- From [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md): `privacy_risk_report.yaml` — residual-PII rate monitored as publish-blocking.
