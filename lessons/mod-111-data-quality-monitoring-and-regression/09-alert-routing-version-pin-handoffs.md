# Chapter 09 — Alert Routing, Version-Pin Attachment, and Role Boundaries

## Why this chapter exists

Chapters 02–08 taught you how to compute the numbers the contract watches. This chapter is the operational chapter — how the number becomes a routed alert, how the alert's evidence is attached to the dataset version-pin so a retraining decision can cite it, and where exactly your work stops and the three adjacent roles take over. By the end you can hand `dataset_monitoring_contract.yaml` across those three handoff lines knowing what you deliver vs. consume on each side.

## The alert's job: evidence, route, severity, timebox

An alert is not a text message. An alert is a structured artifact that must carry everything the triage engineer needs without a second lookup.

```yaml
# alert payload — the shape that leaves your pipeline
alert:
  id: alert.cc_en.v3.m.population.lang_psi.v1.2026-11-04T08:17:00Z
  monitor_id: m.population.lang_psi.v1
  pipeline: cc_en.v3
  slice_version_pin:
    sha256: a1e2b3c4…
    artifact_uri: s3://training-data/cc_en/v3/2026-11-04T08:17:00Z/
    lineage_run_id: openlineage://cc_en.v3/2026-11-04T08:17:00Z
  measured_at: 2026-11-04T08:19:34Z
  measured_value: 0.138
  threshold:
    kind: lte
    value: 0.1
  severity: retrain_blocking
  references:
    reference_slice_sha256: 4d5e6f78…                   # the pinned reference
    reference_slice_uri: s3://training-data/cc_en/v3-ref/2026-09-15/
  evidence_artifacts:
    - evidently_report_json: s3://ops/alerts/alert.cc_en.v3.…/evidently.json
    - population_balance_yaml: s3://ops/alerts/alert.cc_en.v3.…/population_balance.yaml
    - dataset_card_snapshot: s3://ops/alerts/alert.cc_en.v3.…/dataset_card.md
  on_call:
    role: ai-data-engineer
    rotation_url: https://pagerduty.example.com/schedules/…
  recommended_actions:
    - "Open the Evidently report; identify which language cell moved"
    - "Compare filter-stage removal rates (m.population.quality_filter_removal_rate_by_language.v1) to historical band"
    - "Decide: upstream fix, pipeline fix, or accept-and-annotate"
  acknowledgement_timebox:
    severity_window_minutes: 60                            # by severity
  version_pin_annotation_queued: true
```

Three properties to internalise:

1. **The alert is a first-class artifact.** It gets stored (object store, incident-manager db, SIEM) and SHA'd. "What alerts fired on this slice version?" is a question with an exact answer.
2. **The evidence is attached, not linked to ephemeral state.** The Evidently report, the population-balance YAML, a snapshot of the dataset card at alert-time — all pinned to the alert object. If the dataset card is edited in the next hour, the alert still carries its as-of state.
3. **The recommended actions are not aspirational.** They are the steps the triage engineer actually takes; if an engineer wrote them, the shift-level runbook is already half-written.

## Severity tiers mapped to routes and timeboxes

Chapter 01 introduced three severities. Here they get concrete routes and SLAs.

| Severity | Route | Acknowledgement SLA | Resolution SLA | What else fires |
|---|---|---|---|---|
| `publish_blocking` | PagerDuty + Slack `#ai-data-eng-alerts` + inline PR comment (if PR-triggered) | 15 min | 4 h | Publish job already exited non-zero; slice is in `quarantine/`. |
| `retrain_blocking` | Slack `#ai-data-eng-alerts` (page only during business hours) + dataset-card annotation | 4 h (business) | 2 business days | Dataset card gains an "unresolved monitoring alert" banner. |
| `advisory` | Slack `#ai-data-eng-alerts` + weekly digest email | next business day | weekly review | Monitoring digest aggregates N per week. |

The table itself lives in the contract's preamble so new monitors inherit the routing from their severity, not from an ad-hoc author decision:

```yaml
# contract preamble
severity_routing:
  publish_blocking:
    routes: [pagerduty:ai-data-eng, slack:#ai-data-eng-alerts, pr_comment]
    ack_minutes: 15
    resolve_hours: 4
  retrain_blocking:
    routes: [slack:#ai-data-eng-alerts, dataset_card_banner]
    ack_hours: 4
    resolve_business_days: 2
  advisory:
    routes: [slack:#ai-data-eng-alerts, weekly_digest]
    ack_business_days: 1
```

That is also the artifact the `ai-infra-mlops` substrate owner consumes — they implement the routing against the contract, not against individual monitor IDs.

## Attaching alerts to the dataset version-pin

The dataset version-pin ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) Chapter 02 — DVC; Chapter 03 — lakeFS; Chapter 08 — Sigstore / SLSA) is the key under which every alert and every monitor result is filed. The pin is immutable; the alerts attached to it accrete.

Two write paths:

### 1. The on-slice-build path

At slice-build time the pipeline emits the per-slice monitoring artifacts (`per_slice_quality.yaml`, `drift_report.json`, `population_balance.yaml`, `schema_diff.json`) **inside the same snapshot** as the slice itself. The Sigstore signature covers both the slice bytes and the monitoring artifacts.

```
snapshot:
  slice_uri: s3://training-data/cc_en/v3/2026-11-04T08:17:00Z/
  slice_sha256: a1e2b3c4…
  monitoring_artifacts:
    - path: monitoring/per_slice_quality.yaml
      sha256: …
    - path: monitoring/drift_report.json
      sha256: …
    - path: monitoring/population_balance.yaml
      sha256: …
    - path: monitoring/schema_diff.json
      sha256: …
  signed_by: sigstore://keyless:ci@github
  signature_ref: rekor://…
```

That is the **retrainability-decision package**: any later reader can take the SHA, verify the signature, and see exactly what monitoring looked like at publish-time.

### 2. The post-publish alert path

Alerts that fire *after* publish (daily / weekly cadences, retrospective scheduled monitors, ad-hoc investigations) cannot modify the snapshot — it is immutable. They append to a *sidecar* record keyed by the pin:

```
pin_annotations/${slice_sha256}.jsonl
---
{"ts":"2026-11-04T08:19:34Z","alert_id":"alert.cc_en.v3.m.population.lang_psi.v1.2026-11-04T08:17:00Z","severity":"retrain_blocking","measured_value":0.138}
{"ts":"2026-11-04T10:02:11Z","alert_id":"alert.cc_en.v3.m.drift.text_length_ks.v1.2026-11-04T08:17:00Z","severity":"advisory","measured_value":0.08}
{"ts":"2026-11-05T09:00:00Z","alert_id":"…","severity":"advisory","measured_value":…}
```

The sidecar is append-only, signed per append (or in daily digests), and is the authoritative record of "what did we learn about slice `a1e2b3c4…` after it was published?".

When the model team opens the dataset card for `cc_en.v3.2026-11-04T08:17Z`, the card reads the sidecar and renders:

- **Monitoring at publish-time** — the four artifacts from the snapshot.
- **Post-publish alerts** — the list from `pin_annotations/${slice_sha256}.jsonl`, with their severities and evidence links.

The retraining decision reads that card. "Was the slice we trained on under any outstanding retrain-blocking alerts?" is a direct question with a direct answer.

## The three handoff lines, concretely

Chapter 01 named three adjacent roles. This section is the exact artifact that crosses each handoff.

### To `ai-infra-mlops` (peer, L25) — platform substrate

**What you hand them:** `dataset_monitoring_contract.yaml` + the `severity_routing:` preamble.

**What they build:** the scheduler job that runs the monitors, the metric store that holds the results, the alert-manager rules that fire on threshold breach, the dashboards, the routing to PagerDuty / Slack, the PR-comment integration, the sidecar append pipeline, the Sigstore signing of the sidecar.

**What they hand back:** monitor-run IDs, run logs, alert IDs, and the uptime SLO of the monitoring substrate itself (monitor-run success rate, alert latency).

**Where you share ownership:** when a new monitor is proposed, you co-author the threshold (you own the ML-correctness of it, they own the substrate cost). When the substrate migrates (Grafana → Datadog, PagerDuty → Opsgenie), they do the mechanical move and you review the per-monitor equivalence.

**Anti-pattern:** you author a Grafana dashboard definition. The dashboard is their craft; your deliverable is the contract.

### To `ai-eval-engineer` and `model-evaluation-engineer` (peers, L30) — model-output monitoring

**What you hand them:** the version-pin of a slice that has passed the publish-blocking severities, with its monitoring artifacts and the post-publish sidecar. The training-data side's `retrainability_status: ok | caveated | blocked` field on the dataset card.

**What they build:** the online eval loop, the offline eval suite, the LLM-as-judge harness, the A/B test infrastructure, the model-output monitors (quality, toxicity, safety, latency). They *consume* your trusted slice; they do not re-derive dataset-side signals.

**What they hand back:** when their model-output monitoring fires a regression that *might* be dataset-side in origin, they open a correlation ticket against your slice version-pin. The pin is the lingua franca.

**Where you share ownership:** the correlation investigation. "The model regressed on Spanish inputs; did a dataset-side signal move on Spanish?" is a question that reads both sides. You produce the dataset-side answer; they produce the model-output answer.

**Anti-pattern:** you build an "online eval" of the model's output on a test prompt. That is their craft, not yours.

### To `ai-governance-analyst` (L15) — governance evidence

**What you hand them:** periodic exports of the monitoring state keyed by the pipeline id and the dataset version-pins. For NIST AI RMF evidence packages and EU AI Act Article 72 post-market monitoring reports, they need the monitor list, the historical alert log, and the sidecar records.

**What they build:** the governance-package synthesis — mapping your monitoring evidence to the RMF sub-categories, to the EU AI Act Article 15 (accuracy, robustness) and Article 10 (data governance) obligations, to ISO/IEC 5259-4 (data-quality process) and 42001 Annex A.7.

**What they hand back:** the sub-category identifiers they need you to cite in the contract, plus any audit-driven asks ("we need a 90-day roll-up of language-balance alerts for the Q4 regulator packet").

**Where you share ownership:** the mapping from `rmf_subcategory:` / `ai_act_article:` annotations in the contract to their package. The mapping is theirs; the annotations are yours.

**Anti-pattern:** you write the governance package yourself. That is overreach into their craft and the output is worse than the collaboration shape.

## When the four handoffs collide: the incident war room

Occasionally a pipeline incident touches all four roles at once. The pattern you will see:

1. A model-output regression fires (owned by `ai-eval-engineer`).
2. It correlates with a dataset-side alert on the slice the model was trained on (yours).
3. The root cause is a substrate issue — the monitor that *should* have caught it earlier was mis-scheduled (`ai-infra-mlops`).
4. The regulatory posture requires an Article 72 incident disclosure (`ai-governance-analyst`).

The war-room-shape response:

- **One version-pin anchors the narrative.** Everyone references the same slice SHA. No "which version did you mean?" waste.
- **The dataset card becomes the common page.** Model-output regressions, dataset-side alerts, substrate remediation notes, governance narrative — all appended to the card, each entry signed by its author role.
- **Each role contributes their artifact and nothing else.** You contribute the dataset-side reconstruction (which monitors fired when, why one didn't fire earlier, which version-pin is the remediated replacement). You do not opine on model-output remediation or governance narrative.

A clean war room is the stress test of whether your contract is readable to non-data-engineers. Chapter 01's insistence on the contract being YAML (not a dashboard) pays off here — the governance analyst can read the contract line by line.

## What a "mature" monitoring contract looks like

As a last cross-check, the shape of a contract that has been operated against for a year:

- **Every row has been reviewed.** PR reviewers at both your tier and at `ai-infra-mlops` have signed off on every row addition.
- **Thresholds are grounded in historical-band evidence.** No thresholds set by vibes; every one has a `note:` field citing the 95th-percentile band or the incident that set it.
- **The regression-fixture library has grown.** One fixture per alert that led to a fix; each referenced from the contract.
- **References have been refreshed.** Golden references have been rotated on the pipeline's natural cadence (quarterly, or on each retraining), with the rotation recorded in a changelog.
- **The severity distribution is sane.** Perhaps 3–5 publish-blocking metrics, 10–15 retrain-blocking, 20–40 advisory. Dominant-advisory is the healthy shape.
- **Alert rate is predictable.** The team knows how many retrain-blocking alerts a typical week carries; outliers are discussed in the weekly review.

If the contract looks like that, the rest of the module has landed. If it looks like a dashboard screenshot, start over.

## Summary

- An alert is a structured artifact: monitor id, slice version-pin, measured value, threshold, severity, pinned evidence, recommended actions, acknowledgement timebox.
- Severity tiers map one-for-one to routes and SLAs; the mapping lives in the contract preamble, not per monitor.
- The dataset version-pin is the key under which both publish-time snapshots (immutable, signed) and post-publish alert annotations (append-only, signed) accrete. The retraining decision reads both.
- Three handoff lines: `ai-infra-mlops` owns the substrate (contract → their scheduler / metric store / alerter); `ai-eval-engineer` + `model-evaluation-engineer` own model-output monitoring (consume your trusted slices); `ai-governance-analyst` owns the governance package (consume your annotated contract).
- An incident that touches all four roles anchors on one version-pin and one dataset card. Each role contributes its artifact and nothing else.
- A mature contract is dominant-advisory, grounded in historical bands, paired with a growing fixture library, and reviewed across both your tier and the substrate owner's.

That is the end of the lecture material. The exercises in `exercises/` walk you through authoring each row of the contract, side-by-side `whylogs` + Evidently comparisons, the schema-drift rollback workflow, subpopulation balance monitoring, and regression-fixture authoring — one exercise per load-bearing concept.
