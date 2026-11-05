# Chapter 07 — Peer Coordination: Governance Analyst, Risk Engineer, and Evaluation Engineer

## Why this chapter exists

Chapters 01–06 produced evidence. This chapter positions that evidence in the three handoffs an AI Data Engineer must run cleanly for the sourcing-and-provenance workstream to actually land in a regulator-facing package:

- **`ai-governance-analyst` (level 15)** — authors the Data Protection Impact Assessment (DPIA) and the organisation-level AI governance register. You hand them the per-source evidence; they fold it into the DPIA.
- **`ai-risk-engineer` (peer, level 25)** — owns data-side harm modelling for the ML program. You hand them the source-type catalogue and the risk tags; they decide which harms are material.
- **`ai-evaluation-engineer` (next-up, level 35)** — owns the regulator-facing package that the governance council and (in some contexts) the model trainer sign. You hand them the five-artifact slice documentation set plus the three provenance records.

You do not do these peers' work. You feed them. This chapter defines what "fed cleanly" looks like, with the format of each handoff.

## Handoff 1 — `ai-governance-analyst` and the DPIA

Under GDPR Article 35, a Data Protection Impact Assessment is mandatory where processing is likely to result in a high risk to the rights and freedoms of natural persons. Training a large language model on personal data almost always qualifies. In practice:

- The `ai-governance-analyst` drafts the DPIA; counsel reviews; the Data Protection Officer (where appointed) signs.
- The DPIA has sections mandated by Article 35(7) — a systematic description of the processing operations and purposes, an assessment of necessity and proportionality, an assessment of risks to data subjects, and the measures envisaged to address those risks.
- Each section asks data-pipeline questions whose answers come from this module's artifacts.

### What the governance analyst needs from you

For a DPIA to be defensible, the analyst needs seven artifacts from you, keyed to the slice they are assessing:

1. **The selection matrix** (Chapter 01) — what sources were considered and admitted.
2. **The licence policy** (Chapter 02) — what licences were allowlisted and what signed off.
3. **The ingest decision log** (Chapters 03–04) — what the admit function decided for each item, why, and under which policy version.
4. **The Datasheet + Data Statement + Nutrition Label** (Chapter 05) — the per-source and per-slice self-descriptions.
5. **The PII report** (mod-110) — what personal data was detected, scrubbed, retained.
6. **The retention record** (mod-109) — the retention window and the deletion mechanism.
7. **The ML-BOM and SPDX AI records** (Chapter 06) — the supply-chain receipt.

Package these in a directory under the slice's governance folder:

```
slices/sft-chat-owned-ugc-2026-q3/
├── manifest.yaml
├── DATASHEET.md
├── DATA_STATEMENT.md
├── NUTRITION_LABEL.html
├── README.md                                        # HF card
├── croissant.json
├── mlbom.cyclonedx.json
├── spdx.ai.json
├── c2pa_manifests_index.json
├── ingest_decisions.jsonl
├── pii_report.json
├── retention_record.yaml
├── representativeness_report.json
├── quality_25012.yaml
└── governance/
    ├── selection_matrix.yaml                        # Chapter 01
    ├── license_policy_v2026-11-04.yaml              # Chapter 02 (symlink to current policy)
    ├── crawl_contract.md                            # Chapter 03
    └── dpia_input_bundle.yaml                       # the hand-off index this chapter emits
```

The `dpia_input_bundle.yaml` is the directory-of-record for the governance analyst. A minimal shape:

```yaml
# governance/dpia_input_bundle.yaml
slice_id: sft-chat-owned-ugc-2026-q3
slice_sha: 9a1f3c…
bundle_generated: 2026-10-15T09:00:00Z
bundle_generated_by: ai-data-engineer@example.com
artifacts:
  selection_matrix: governance/selection_matrix.yaml
  license_policy_in_force: policies/data_license_policy_v2026-11-04.yaml
  ingest_decision_log: ingest_decisions.jsonl
  datasheet: DATASHEET.md
  data_statement: DATA_STATEMENT.md
  nutrition_label: NUTRITION_LABEL.html
  pii_report: pii_report.json
  retention_record: retention_record.yaml
  mlbom: mlbom.cyclonedx.json
  spdx_ai: spdx.ai.json
  c2pa_index: c2pa_manifests_index.json
  representativeness: representativeness_report.json
  quality_25012: quality_25012.yaml
signoff_status:
  ai_data_engineer: signed 2026-10-15
  ai_governance_analyst: pending
  counsel: pending
  dpo: pending
```

The governance analyst reads this bundle, writes the DPIA against it, and pushes back any questions as review comments on the bundle itself. You own the resolution; the analyst does not re-engineer your pipeline to answer them.

### What you ask of the governance analyst

Running the handoff cleanly also means asking cleanly:

- **Lawful basis determinations per source** — which GDPR Article 6 basis applies, with pointer to the LIA / consent store / contract ID. This is the `privacy_basis` cell in the selection matrix.
- **Special-category treatment per source** — filter or 9(2) exception + basis.
- **Jurisdiction posture** — which jurisdictions' regimes apply to this slice and what counsel's position is for each.
- **Retention policy** — the window, triggered by what event.
- **Subject-request mechanism** — how the slice supports Article 15 / 17 / 21 requests. The analyst coordinates with the privacy engineering team on the user-facing side; you own the technical lookup inside the slice.

These are the fields you expect the analyst to populate before you can finalise the slice's documentation.

## Handoff 2 — `ai-risk-engineer` and data-side harm modelling

`ai-risk-engineer` is your peer (level 25 in the Risk family, same tier as you in the Data family). Their job is to enumerate the harms the ML system could cause to users, third parties, and the deployer, assess likelihood and severity, and propose mitigations. Data-side harms are a central input.

### What the risk engineer needs from you

- **Source-type catalogue** — the composition of the slice across the five source types (Chapter 01). The risk engineer uses this to reason about memorisation (owned UGC + licensed partner carry highest), unlicensed content exposure (public crawl + third-party public with weak attribution), hallucinated PII risk (synthetic), inherited contamination (third-party public).
- **Opt-out posture** — how robustly the pipeline honours signals (Chapter 03). The risk engineer uses this to reason about reputational and legal risk from rightsholder complaints.
- **Licence posture** — the allowlist / denylist in force (Chapter 02). The risk engineer uses this to reason about commercial exposure if a licence posture shifts.
- **Inherited risk tags** — per-source flags for PII prevalence, benchmark contamination risk, special-category prevalence, offensive-content prevalence. The risk engineer uses these to model downstream model behaviour before the model is trained.

Package these as a **data-risk register slice** keyed to the slice ID:

```yaml
# governance/data_risk_register.yaml
slice_id: sft-chat-owned-ugc-2026-q3
composition:
  owned-ugc: 0.55
  third-party-public: 0.25
  licensed-partner: 0.15
  synthetic: 0.05
inherited_risks:
  - risk: user-memorisation-of-support-conversations
    prevalence_source: owned-ugc
    mitigation_in_pipeline: mod-105-dedup + mod-110-pii-scrub
    residual_likelihood: low
  - risk: benchmark-contamination
    prevalence_source: third-party-public
    mitigation_in_pipeline: mod-106-decontamination
    residual_likelihood: low
  - risk: licensor-dispute-on-attribution
    prevalence_source: licensed-partner
    mitigation_in_pipeline: per-record-attribution-sidecar
    residual_likelihood: low
  - risk: rightsholder-opt-out-missed-on-upstream-aggregator
    prevalence_source: third-party-public
    mitigation_in_pipeline: upstream-opt-out-revalidation-on-rebuild
    residual_likelihood: medium
```

The risk engineer composes this with the other risk inputs (model-training-side, serving-side, integration-side) and produces the system-level risk register. You then know which data-side risks are elevated enough to get additional mitigation attention in the next rebuild.

### Where peer coordination becomes joint work

Some artifacts are jointly authored:

- **Adversarial-data filtering runs** — mod-105 (dedup / poisoning defence) is a peer-collaboration surface.
- **Red-team dataset design** — the risk engineer specs adversarial prompts; you build the dataset that stresses them.
- **Memorisation evaluations** — you produce the dedup report; the risk engineer runs the memorisation probe (e.g. extractive attacks a la Carlini 2021); the result lands in both your risk register and the model team's evaluation report.

Peer coordination is not handoff-then-forget; it is a continuous shared surface. The data-risk register is the artifact that keeps both of you honest.

## Handoff 3 — `ai-evaluation-engineer` and the regulator-facing package

`ai-evaluation-engineer` (level 35, the next tier up from you) owns the regulator-facing package — the composite artifact a regulator (EU AI Office, FDA, financial-services supervisor, state attorney general) would be shown as the complete evidence of an AI system's documentation. For a GPAI system the package shape is derived from AI Act Article 53, Annex XI, Annex XII and the GPAI Code of Practice.

### What the evaluation engineer needs from you

A single-slice package is what the evaluation engineer consumes. For a model trained on N slices, they compose N packages into one. Your deliverable per slice is the **slice evidence pack**:

- The seven artifacts from the DPIA bundle above.
- The CycloneDX ML-BOM and SPDX 3.0 AI records.
- The C2PA manifests index.
- The quality-measure log against ISO/IEC 25012 and 5259-2 (mod-104).
- The deduplication report (mod-105).
- The benchmark-decontamination report (mod-106).
- The lineage graph (mod-109).

The evaluation engineer's package wraps these with:

- The model card (not your authorship).
- The model-side ML-BOM / SPDX AI.
- Safety-testing results.
- The AI Act Article 53(1)(d) summary of training content (the "sufficiently detailed summary" the AI Office provides a template for).
- The Code of Practice compliance evidence.

### The AI Act Article 53(1)(d) summary

Under Article 53(1)(d), GPAI model providers must publish a sufficiently detailed summary about the content used for training. The EU AI Office published its *Explanatory Notice and Template for the Public Summary of Training Content* on 24 July 2025 (entered into force 2 August 2025; models placed on the market before that date have until 2 August 2027 to comply) — see the AI Office pages under <https://digital-strategy.ec.europa.eu/en/policies/ai-office>. The summary's data-side inputs are your selection matrix, your source-type catalogue, your opt-out posture, and the aggregate statistics from the Datasheet.

Your deliverable to the evaluation engineer is a **summary input YAML** that maps your slice evidence to the template fields:

```yaml
# governance/article_53_1_d_summary_input.yaml
slice_id: sft-chat-owned-ugc-2026-q3
source_breakdown:
  public_web_crawl: 0.00
  licensed_partner_corpora: 0.15
  owned_ugc: 0.55
  third_party_public_datasets: 0.25
  synthetic: 0.05
licensed_partner_corpora_counterparties:
  - name: licensed-qa-vendor-x
    dla_id: DLA-2026-04
third_party_public_datasets:
  - name: ultrachat-tulu-mix
    license: mixed-upstream
opt_out_signals_honoured:
  - robots-txt
  - ai-txt
  - meta-noai
  - c2pa-do-not-train
  - well-known-ai-preferences  # if applicable at read time
opt_out_point_of_contact: rightsholder-concerns@example.com
dsm_article_4_policy_pointer: policies/copyright_policy_v2026-11-04.md
copyright_policy_gpai_code_signatory: true | false  # organisational choice
```

The evaluation engineer merges this input across all N slices and renders the summary.

### The cadence

- **Per slice rebuild** — you regenerate the evidence pack. The rebuild is triggered by data refresh, policy update, or opt-out revalidation.
- **Per model training run** — the evaluation engineer composes the model's package from the constituent slices.
- **Per regulatory inquiry** — the evaluation engineer serves the package; you respond to clarifying questions whose answers come from your artifacts.
- **Per Code of Practice reporting cadence** — the governance council serves whatever the Code requires; you supply the data-side evidence.

## What you do not do, in all three handoffs

Codifying the role boundary once more, because the three handoffs tempt you across it:

- You do not author the DPIA (governance analyst's).
- You do not opine on lawful basis or on fair use (counsel's).
- You do not model the harms (risk engineer's).
- You do not decide what the regulator is told (evaluation engineer's; governance council's).
- You do not sign off on the AI Act Article 53(1)(d) summary — you supply the input (evaluation engineer signs; the organisation declares).

What you do: build the artifacts, keep them current, keep them signed, and respond cleanly to the questions each peer raises against them. The role boundary is the reason the handoffs are clean; the artifacts are the reason the handoffs are fast.

## Summary

- Three peer handoffs: `ai-governance-analyst` (DPIA), `ai-risk-engineer` (data-side harm modelling), `ai-evaluation-engineer` (regulator-facing package).
- Each handoff has a defined artifact: `dpia_input_bundle.yaml`, `data_risk_register.yaml`, and the slice evidence pack / `article_53_1_d_summary_input.yaml`.
- The artifacts are generated from the manifest and the module-wide reports; they are not hand-maintained.
- You do not do these peers' work. You feed them. The role boundary stays firm; the artifacts stay current.
- The cadence is triggered by slice rebuild, model training run, regulatory inquiry, and Code of Practice reporting. Your artifacts must be current enough to serve each cadence without archaeology.

Exercise 04 (documentation authoring) and Exercise 05 (metadata emission) between them produce the artifacts this chapter routes. Module `mod-112-cross-functional-data-program-and-communication` later picks up the broader program-coordination surface.
