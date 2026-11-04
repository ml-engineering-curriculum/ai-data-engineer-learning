# Exercise 01 — Author `dataset_monitoring_contract.yaml` for One Pipeline

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 01 — The dataset-side monitoring contract](../01-the-dataset-side-monitoring-contract.md), [Chapter 09 — Alert routing, version-pin, handoffs](../09-alert-routing-version-pin-handoffs.md)

## Objective

Write a signed, review-ready `dataset_monitoring_contract.yaml` for a real pipeline you can touch. By the end you will have:

- A per-pipeline contract with the preamble (`severity_routing`, `reference_strategy`, `subpopulation_axes`, `handoffs`) authored.
- At least **eight metric rows** covering all four `kind` values — `schema_drift`, `data_drift`, `population_drift`, `quality` — with cadence, threshold, severity, owner, route, and evidence-artifact path per row.
- Each row grounded in either a historical-band reference (not a vibes threshold) or an explicit incident reference.
- At least one row per severity tier (`publish_blocking`, `retrain_blocking`, `advisory`), with routing that matches the preamble.
- A short `README.md` defending the choices — why each threshold is where it is, why each severity is what it is, which upstream artifact each row consumes.

## Prerequisites

- Chapter 01 read; you can recite the three monitoring surfaces this contract is *not* (model-output, classical DWH, substrate).
- Chapter 09 read; you understand how alerts attach to the version-pin and how severity tiers map to routes.
- A pipeline to monitor. Reuse the slice-building pipeline from [`mod-104`](../../mod-104-data-quality-measurement-and-audit/README.md), [`mod-105`](../../mod-105-deduplication-at-scale/README.md), or [`mod-110`](../../mod-110-pii-and-privacy-risk-for-training-data/README.md) — anything that produces a per-slice quality / dedup / privacy report on a cadence.
- A dataset version-pin convention. Any of DVC, lakeFS, Pachyderm, or a plain `snapshots/${sha256}/` directory is fine; the contract will reference the SHA shape.
- Python 3.11+ with `pyyaml` and `jsonschema` (to validate the contract against its schema).

## Problem statement

Author a contract that another engineer could hand to `ai-infra-mlops` tomorrow and have running within a sprint. The contract must:

1. Be a well-formed YAML document that validates against an authored JSON Schema (`contract_schema.json`).
2. Include a preamble naming the severity routing, the reference strategy, the subpopulation axes, and the handoff roles.
3. Enumerate at least eight metric rows spanning all four `kind` values.
4. Attach each row to an upstream artifact (`per_slice_quality.yaml`, `dedup_report.yaml`, `contamination_report.yaml`, `privacy_risk_report.yaml`, `schema/*.yaml`, or `population_balance.yaml` — whichever applies).
5. Name the on-call role, the alert route, and the acknowledgement SLA (inherited from the preamble).
6. Hash-reference the reference-slice SHA and the contract version.

## Starter guidance

1. **Pick the pipeline.** Write one paragraph identifying it: what source(s) it ingests, what slice version-pins it emits, what cadence it runs on, who owns it today. Record in `pipeline_scope.md`.

2. **Author the JSON Schema first.** Draft `contract_schema.json` with required fields per row (`id`, `kind`, `expression`, `cadence`, `threshold`, `severity`, `owner`, `route`, `evidence_artifact`). This forces precision on the shape before you write content.

3. **Author the preamble.** The preamble declares severity routing, reference strategy, subpopulation axes, and handoff roles. Copy the `severity_routing` table from Chapter 09 verbatim unless your org has a different route convention — in which case document the deviation.

4. **Enumerate the metric rows.** Walk each upstream artifact and ask what to monitor:
   - `per_slice_quality.yaml` → at least two `kind: quality` rows (residual-PII rate, completeness coverage per the 5259-2 measures).
   - `dedup_report.yaml` → at least one `kind: quality` row (dedup rate per source family vs. trailing median).
   - `contamination_report.yaml` → at least one publish-blocking `kind: quality` row (benchmark-overlap n-gram hit).
   - Schema contract → at least one `kind: schema_drift` row (column-set equality).
   - `population_balance.yaml` → at least one `kind: population_drift` row (language PSI vs. golden reference).
   - Reference-vs-current distribution comparisons → at least one `kind: data_drift` row (e.g., text-length KS statistic).

5. **Ground every threshold.** Each threshold's `note:` field cites either (a) a historical-band percentile ("2.5× the 95th-percentile of 30 prior slice pairs"), (b) a regulatory / contractual minimum, or (c) a specific incident ID. Vibes-set thresholds fail review.

6. **Pick severities deliberately.** No more than 3–5 rows at `publish_blocking`. The rest divide across `retrain_blocking` and `advisory`. Over-escalation is the common failure mode.

7. **Record the handoff.** The `handoffs:` block names `substrate_owner` (an `ai-infra-mlops` contact), `model_output_owners` (the two eval roles), and `governance_consumer` (the governance analyst).

8. **Validate.** Run `jsonschema` against your contract. Hash the contract; record the SHA in a `contract.sha256` sidecar. Commit both.

9. **Write the defending README.** Three paragraphs: scope, threshold-provenance table, and severity-count summary. The README is what a reviewer reads first.

## Deliverables

```
exercise-01/
  pipeline_scope.md                           # which pipeline, who owns it, what cadence
  contract_schema.json                        # JSON Schema for the contract
  dataset_monitoring_contract.yaml            # the contract itself
  dataset_monitoring_contract.yaml.sha256     # pin the version
  threshold_provenance.md                     # per-row threshold justification table
  README.md                                   # defending README
```

## Acceptance criteria

- `dataset_monitoring_contract.yaml` validates against `contract_schema.json` with `jsonschema` (or an equivalent validator).
- The preamble includes `severity_routing`, `reference_strategy`, `subpopulation_axes`, and `handoffs` blocks.
- At least **eight metric rows** are present, covering all four `kind` values at least once each.
- At least **one row per severity tier** (`publish_blocking`, `retrain_blocking`, `advisory`); no severity tier has more than ~half the rows.
- Every row has: `id`, `kind`, `expression`, `cadence`, `threshold` (with `note`), `severity`, `owner`, `route`, `evidence_artifact`.
- Every threshold's `note` field cites a historical band, a contractual minimum, or an incident ID — not a vibes default.
- Every row names an upstream artifact in the `evidence_artifact` field, keyed off the pipeline's real output paths.
- The `handoffs:` block names a concrete `substrate_owner`, two `model_output_owners`, and one `governance_consumer` — all as role identifiers from the ladder.
- `threshold_provenance.md` has one row per metric with (metric id, threshold value, source of threshold, notes).
- `README.md` is under two pages and contains: pipeline scope (1 paragraph), severity-count summary, three example metric row explanations.
- Any factual claim in the README that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Author the contract JSON Schema as a reusable artifact.** Submit it back as a proposal for a `aicg-dataset-monitoring-contract` convention; it can be linted and versioned like any other schema.
- **Round-trip through `ai-infra-mlops`'s substrate.** Translate three contract rows into concrete monitor definitions in whichever platform your org runs (Prometheus `PrometheusRule`, Datadog monitor JSON, Grafana alert rule, Monte Carlo custom monitor, Soda Cloud check). Demonstrate that round-trip leaves the contract as the source of truth.
- **Review the contract with an actual peer.** Sit with someone in a nearby role (`ai-infra-mlops`, `ai-eval-engineer`, or `ai-governance-analyst`). Record one surprise finding from the conversation in the README.
- **Sign the contract.** Sign `dataset_monitoring_contract.yaml` with Sigstore keyless signing (per [`mod-109`](../../mod-109-dataset-versioning-lineage-reproducibility/README.md) Chapter 08). Record the Rekor entry.

## What this exercise is not

- It is not an implementation exercise — the substrate that *executes* the contract belongs to `ai-infra-mlops`.
- It is not a drift-measurement exercise (Exercise 02).
- It is not a regression-fixture exercise (Exercise 05).

It is the **authoring** of the deliverable this module is organised around. Every subsequent exercise populates one of its rows with concrete computation.
