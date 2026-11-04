# Chapter 01 — The Dataset-Side Monitoring Contract

## Why this chapter exists

The pipeline you own produces training slices on a cadence. Once the slice is signed and released ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)) the clock starts: upstream sources change, crawlers catch new TLDs, licensing posture shifts, a new source family is admitted, a regex in a filter is edited. **Every refresh of the slice is a point at which a dataset-side regression can be introduced** — and if it is introduced silently, the first signal that reaches the business is usually a model-output regression weeks later, with the training-data pipeline nowhere in the incident narrative.

The job of this module is to make the dataset-side signal arrive **first, with a route, with an owner, and with the version-pin of the slice the signal belongs to**. The deliverable is a **dataset monitoring contract** — a per-pipeline specification that enumerates each metric, its cadence, its threshold, its owner, and its alert route. This chapter defines the contract, draws the lines that separate it from adjacent monitoring surfaces, and gives you the schema you will populate for the rest of the module.

## The three distinctions you must hold in your head

The word "monitoring" is overloaded. Four different teams in the org ship things called "monitoring" and all four show up in incident reviews. Before you can write a dataset-side contract that owners will sign, you must be able to name what it is *not*.

### 1. Dataset-side monitoring is not model-output monitoring

Model-output monitoring watches the model's *behaviour* in production: response quality, LLM-as-judge scores, offline evals replayed on fresh traffic, A/B harness metrics, user-satisfaction signals, toxicity / safety classifiers over outputs. That is the craft of the two peer roles at level 30 who sit one step downstream of you:

- `ai-eval-engineer` (peer, L30) — builds the online eval loop: judges, test harnesses, routing of production traffic through eval before it reaches users.
- `model-evaluation-engineer` (peer, L30) — authors offline eval suites and the methodology for scoring them.

Dataset-side monitoring is **upstream**: it watches the training slice, not the inference stream. When a dataset-side alarm and a model-output alarm fire in the same week, the version-pin from Chapter 09 is what lets both sides correlate. But the artifacts are different: your deliverable is a signed `dataset_monitoring_contract.yaml` tied to a dataset version; theirs is an eval-report tied to a model version.

The boundary is sharp. If you catch yourself writing an alert over `prompt_toxicity_rate_at_inference_time`, you are in the wrong module.

### 2. Dataset-side monitoring is not classical data-warehouse monitoring

The data platform team (often `analytics-engineer` / `data-platform-engineer` titles in the org, not on this ladder) runs classical warehouse-side DQ monitoring: freshness SLAs on BigQuery / Snowflake / Databricks tables, null-rate watches, volume anomaly detection, query-performance alerts, schema-registry gates. That work is production-shaped — it exists because dashboards, reports, and operational tables cannot be stale or wrong without immediate business impact.

Those tools (Monte Carlo, Metaplane, Bigeye, Soda Cloud) are **warehouse-first** (Chapter 04). They will run on your training slice if it is exposed through a warehouse or a Lakehouse backend, but the surface they cover natively is the surface of a warehouse table, not the surface of a training corpus. Specifically they do not express out of the box:

- "Jensen-Shannon divergence of the FastText-detected language histogram between `cc_en.2026-10` and `cc_en.2026-11`."
- "Dedup rate per source family, compared to the 90-day trailing median."
- "Benchmark-overlap n-gram hit count relative to the golden decontamination report."
- "Residual-PII rate per source after the Presidio stage, broken down by detector family."

Chapter 04 is explicit about what you can and cannot reuse from these platforms, and when you should feed them your metrics vs. when you need a parallel ML-aware channel.

The warehouse team's monitoring is not wrong — it is tuned for a different axis. Treat it as a *sibling* contract, not as an alternative to yours. Chapter 09 covers the common evidence slot where both end up attached to the dataset version-pin.

### 3. Dataset-side monitoring is not the orchestrator / platform substrate

The scheduler that runs your pipeline, the observability platform that stores and queries your metrics, the paging routes that fan an alert out to on-call — all of that is **platform substrate** owned by a peer at the level below you, `ai-infra-mlops` (peer, L25). Airflow / Prefect / Dagster / Argo, Prometheus / Grafana / OpenTelemetry, Datadog / New Relic, PagerDuty / Opsgenie, the artifact stores (S3 / GCS / Azure Blob) — none of these are the deliverable of this module.

What you ship is **the metric authoring layer**: *which numbers to compute, at which cadence, with which threshold, routing where, owned by whom*. `ai-infra-mlops` runs the substrate those metrics live on. The handoff shape is in Chapter 09.

A healthy sanity check: if your "monitoring contract" is a Grafana dashboard, you have crossed into their craft. If it is a YAML document of named metrics and thresholds that *their* dashboard could render, you are in your lane.

## The deliverable — `dataset_monitoring_contract.yaml`

The artifact you produce in this module is a signed YAML (or equivalent schema) per pipeline that enumerates the monitored surface of the training slice it produces. The minimal shape:

```yaml
# dataset_monitoring_contract.yaml — mod-111 deliverable
contract:
  id: dmc.cc_en.v3
  slice_pipeline: cc_en.v3                 # the pipeline this contract covers
  authored_at: 2026-11-04
  authored_by: ai-data-engineer@acme
  reviewers:
    - ai-infra-mlops@acme                  # platform substrate owner
    - ai-governance-analyst@acme           # consumer for RMF evidence
  attached_to: slice_version_sha256.*      # every slice version inherits this contract
  version: 3

metrics:
  - id: m.schema.column_set.v1
    kind: schema_drift                     # schema | data_drift | population_drift | quality | residual_pii
    expression: >
      column_set(current) == column_set(reference)
      AND dtype_map(current) == dtype_map(reference)
    reference: cc_en.v3.ref_2026-09-15     # the pinned reference snapshot
    cadence: on_slice_build                # on_slice_build | daily | weekly | ad_hoc
    threshold:
      must_be: equal                       # explicit, no fuzziness on schema
    severity: publish_blocking             # publish_blocking | retrain_blocking | advisory
    owner: ai-data-engineer@acme
    route: pagerduty:ai-data-eng
    evidence_artifact: schema_diff.json
    rmf_subcategory: "MANAGE 4.1"
    ai_act_article: "72 (post-market monitoring)"

  - id: m.drift.language_histogram_jsd.v1
    kind: data_drift
    expression: >
      jensen_shannon_divergence(
        language_histogram(current),
        language_histogram(reference)
      )
    reference: cc_en.v3.ref_2026-09-15
    cadence: on_slice_build
    threshold:
      lte: 0.02
    severity: retrain_blocking
    owner: ai-data-engineer@acme
    route: slack:#ai-data-eng-alerts
    evidence_artifact: drift_report.html   # Evidently report path
    rmf_subcategory: "MEASURE 2.11"

  - id: m.population.lang_balance_psi.v1
    kind: population_drift
    expression: >
      psi(
        slice_counts_by(language),
        reference_counts_by(language)
      )
    reference: cc_en.v3.ref_2026-09-15
    cadence: on_slice_build
    threshold:
      lte: 0.1                             # standard PSI threshold; Chapter 05 defends the choice
    severity: advisory
    owner: ai-data-engineer@acme
    route: slack:#ai-data-eng-alerts

  - id: m.quality.residual_pii_rate.v1
    kind: quality
    expression: residual_pii_count(current) / token_count(current)
    cadence: on_slice_build
    threshold:
      lte: 1e-5
    severity: publish_blocking
    owner: ai-data-engineer@acme
    route: pagerduty:ai-data-eng

regression_fixtures:
  - id: f.lang_misclass_regression_2026-03.v1
    path: fixtures/lang_misclass_regression_2026-03/
    expected_outcome: pass
    on_failure: block_pr                   # fixture is CI-gating

handoffs:
  substrate_owner: ai-infra-mlops@acme
  model_output_owners:
    - ai-eval-engineer@acme
    - model-evaluation-engineer@acme
  governance_consumer: ai-governance-analyst@acme
```

Read the schema once. The rest of the module fills it in. By the end:

- Chapters 02–04 cover the tools that *compute* the metric expressions.
- Chapters 05–07 cover what to put in `kind: data_drift | schema_drift | population_drift` for a training slice, specifically.
- Chapter 08 covers the `regression_fixtures:` block.
- Chapter 09 covers the `route:`, `severity:`, `owner:`, and version-pin attachment — the operational layer.

## Why this contract is the deliverable (not a dashboard)

Three reasons the contract is the signed artifact, not the Grafana board:

1. **Auditability.** NIST AI RMF MANAGE 4 and MEASURE 3 sub-categories, EU AI Act Article 72 (post-market monitoring, including *dataset-side signals* for high-risk and GPAI systems), and ISO/IEC 5259-4 (data-quality process) all want to see the specification of monitoring, not screenshots of the implementation. A dashboard drifts; a signed contract is pinned to a version.
2. **Portability.** `ai-infra-mlops` may switch the substrate from Grafana to Datadog to internal-X over the module's lifetime. The contract survives those migrations; a dashboard definition does not.
3. **Review discipline.** A YAML document can be code-reviewed line by line. A dashboard cannot — reviewers skim the layout and approve. If you want governance stakeholders to actually read what you monitor, the contract is a document, not a picture.

## How this contract relates to artifacts from earlier modules

Mod-111 does not invent the metrics it monitors; it watches metrics that have been produced since mod-104. Specifically:

| Earlier module | Artifact | What the monitor contract says about it |
|---|---|---|
| [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) `per_slice_quality.yaml` | Per-slice 25012 / 5259 measure record | Every row with a `threshold:` becomes a monitored metric; the metric id is reused. |
| [`mod-105`](../mod-105-deduplication-at-scale/README.md) `dedup_report.yaml` | Duplicate-count distribution | Monitored as `kind: quality` (dedup-rate per source family vs. trailing median). |
| [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) `contamination_report.yaml` | n-gram overlap, Min-K% Prob results | Monitored as `kind: quality` with publish-blocking severity. |
| [`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md) label quality | IAA, per-labeller accuracy, task throughput | Monitored as `kind: quality`; sudden drop routes to labelling-ops owner. |
| [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) `snapshot.yaml` | Dataset version-pin | The `attached_to:` field — every alert carries the version-pin as its primary key. |
| [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md) `privacy_risk_report.yaml` | Residual-PII counts, detector precision/recall | Monitored as `kind: quality` with publish-blocking severity on residual-PII rate; detector-recall drop is an advisory. |

The `dataset_monitoring_contract.yaml` is the *moving-average over time* of those point-in-time reports. A single slice gets a report; a *pipeline* gets a contract.

## Cadence options and why each exists

Three cadences cover every monitored metric in this module. Keep the names stable; they feed into the substrate owner's scheduler config.

- **`on_slice_build`** — runs every time the pipeline produces a new slice version. Correct for schema drift, data drift against reference, residual-PII rate, benchmark contamination. Blocks the publish if the severity is publish-blocking.
- **`daily` / `weekly`** — recurring probes against the current *published* slice, independent of pipeline activity. Correct for e.g. labeller IAA roll-ups, upstream-licence revalidation, fairness-slice dashboards. Fires an advisory alert on breach.
- **`ad_hoc`** — triggered from a PR, an incident, or a manual investigation. Correct for regression fixtures (Chapter 08) and for incident-response probes.

Anything you cannot fit into one of these three cadences is probably a model-output monitor — route it to `ai-eval-engineer`.

## Severity levels and what each commits you to

Three severities, three operational semantics. Owners agree to these before the contract is signed.

| Severity | Meaning | Who is on the hook when it fires |
|---|---|---|
| `publish_blocking` | The slice does not leave the pipeline until this clears. The publish job exits non-zero. | You (as the slice owner) and `ai-infra-mlops` (to confirm the publish gate wired correctly). |
| `retrain_blocking` | The slice may publish, but the model team is told it is not retrain-ready without acknowledgement. The dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) is annotated. | You and the model team (as the retraining customer). |
| `advisory` | The signal is recorded on the version-pin and in the digest; it does not block any workflow. | You, to triage within a weekly review cadence. |

Over-escalation is the common failure mode. Every `publish_blocking` metric is an on-call commitment — only schema drift, residual-PII rate, benchmark-contamination hits, and licence-revocation signals belong there. Everything else is `retrain_blocking` or `advisory`.

## A worked example — authoring one line of the contract

Work through this one line:

```yaml
- id: m.drift.language_histogram_jsd.v1
  kind: data_drift
  expression: jensen_shannon_divergence(language_histogram(current), language_histogram(reference))
  reference: cc_en.v3.ref_2026-09-15
  cadence: on_slice_build
  threshold:
    lte: 0.02
  severity: retrain_blocking
  owner: ai-data-engineer@acme
  route: slack:#ai-data-eng-alerts
```

What that means in practice:

- **Who computes it.** The pipeline itself, at slice-build time, invoking an Evidently `ColumnDriftMetric` (Chapter 03) over the `language` column against a pinned reference profile.
- **Where the number lives.** As a field in the per-slice quality report (`per_slice_quality.yaml`) and as a time-series point in the substrate owner's metric store.
- **What fires the alert.** A scheduled check in the substrate that reads the per-slice report and compares `value` to `threshold.lte`.
- **What the alert carries.** Pipeline id, slice version-pin SHA, metric id, measured value, threshold, link to the Evidently HTML report, link to the dataset card, timestamp.
- **Who gets paged.** Nobody — the `route:` is Slack, not PagerDuty. The severity is retrain-blocking, not publish-blocking.
- **What the on-call engineer does.** Open the Evidently report, identify which language rose or fell, decide whether to roll back the publish or acknowledge-and-continue; annotate the dataset card either way.

That is the shape every line of the contract takes. Everything else in this module is detail on *which* metrics to author, *which* tools to compute them with, and *how* to attach their evidence to the version-pin.

## The role-boundary mnemonic you leave this chapter with

- **Model-output monitoring** — `ai-eval-engineer` + `model-evaluation-engineer`. Boundary: anything downstream of training.
- **Classical DWH monitoring** — the data platform team. Boundary: warehouse-table freshness, volume, business-table schema.
- **Monitoring substrate** — `ai-infra-mlops`. Boundary: scheduler, metric store, alert routing, dashboards.
- **Dataset-side monitoring** — **you**. The metric authoring layer attached to the training slice's version-pin.

When a stakeholder in a design review asks "who monitors this?", that mnemonic is the answer.

## Summary

- The deliverable of this module is `dataset_monitoring_contract.yaml` — a per-pipeline specification of metrics, cadences, thresholds, severities, owners, and routes, attached to the dataset version-pin.
- The contract lives *between* three adjacent monitoring surfaces that are owned by other roles: model-output monitoring (peer L30s), classical DWH monitoring (data platform), and the observability substrate (`ai-infra-mlops` at L25).
- Three cadences cover every metric: `on_slice_build`, `daily`/`weekly`, `ad_hoc`.
- Three severities commit you to operational behaviour: `publish_blocking`, `retrain_blocking`, `advisory`. Over-escalation is the common failure mode.
- The contract is a YAML document, not a dashboard — because it is audited, portable, and code-reviewable.
- Every metric in the contract maps back to an artifact produced in an earlier module; mod-111 does not invent metrics, it watches them on a cadence.

Chapter 02 picks up with `whylogs` — the first of the two primary ML-aware profiling tools you will reach for when computing the metric values the contract watches.
