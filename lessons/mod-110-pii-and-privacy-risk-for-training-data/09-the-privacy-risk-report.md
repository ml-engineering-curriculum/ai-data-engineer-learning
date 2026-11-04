# Chapter 09 — The Privacy-Risk Report as Audit Evidence

## Why this chapter exists

Every previous chapter contributes a block. This chapter assembles them into **the single signed deliverable** — `privacy_risk_report.yaml` — that the rest of the organisation consumes. It is the handoff artifact to:

- **`fine-tuning-engineer`** — sets the DP-SGD recipe and the training mixture.
- **`ai-risk-engineer`** — seeds the attack-simulation harness.
- **Privacy counsel** — closes the Art 6 / 9 / 14 / CCPA / HIPAA analysis.
- **`mod-112` dataset card** — surfaces the policy publicly.
- **`mod-109` lineage** — records the pipeline event with provenance.

If the report does not exist for a slice, the slice is not training-ready. That rule is non-negotiable.

## The schema in full

The report is a single YAML file. Every field is machine-readable; human prose belongs in a separate LIA document that this file references by SHA.

```yaml
version: 1.0.0
report_id: privacy-risk-slice42-2026-11-04-abc1234
authored_by: ai-data-engineer
authored_at: "2026-11-04T18:00:00Z"
pipeline_run_id: 2026-11-04-abc1234

# §1 Slice identity and provenance
slice:
  id: slice-42-web-en-2026-q3
  description: "English-language web crawl Q3 2026, post-dedup, post-scrub"
  size:
    documents: 184_212_443
    tokens: 312_000_000_000
  source_manifest_sha: 7c91…      # from mod-102
  dedup_report_sha: 7f9c…         # from mod-105
  quality_report_sha: 2b4a…       # from mod-104
  jurisdictions:
    - EU (source domains with .eu, .de, .fr TLDs)
    - US (source domains with .com, .us, etc.)
    - unknown (unresolved source domains)

# §2 Detection policy
detection:
  pipeline_stage: post_ingest_separate_stage
  detectors:
    - id: presidio-prod
      presidio_analyzer_version: 2.2.33
      nlp_engine: spacy:en_core_web_lg==3.7.1
      config_sha: 9f3c…
      recognisers:
        - name: EmailRecognizer
          type: pattern
          version: builtin
          threshold: 0.60
        - name: CreditCardRecognizer
          type: pattern+validator
          version: builtin
          threshold: 0.70
        - name: PersonRecognizer
          type: ner
          version: builtin
          threshold: 0.40
        - name: customer_id_recognizer
          type: pattern+context
          version: v3
          threshold: 0.80
          context_triggers: ["customer", "cust", "account", "cx"]
          pattern_sha: a1b2…
        # ... full list
    - id: oracle-dlp-sample
      role: eval_oracle_only
      sampling_rate: 0.0005
      scope: "stratified sample, used for Chapter 04 eval + Chapter 06 drift check"
      findings_sha: 44ee…

# §3 Detection quality on eval set
detection_quality:
  eval_dataset_id: eval-slice42-v3
  eval_dataset_sha: ab2c…
  eval_dataset_size: 1_842 labelled documents, 58_431 labelled spans
  per_entity:
    EMAIL_ADDRESS:
      precision: 0.993
      recall:    0.981
      f1:        0.987
      n_support: 412
      ci95_recall: [0.971, 0.989]
    PERSON:
      precision: 0.87
      recall:    0.74
      f1:        0.80
      n_support: 1_120
      ci95_recall: [0.71, 0.77]
    CREDIT_CARD:
      precision: 1.00
      recall:    0.998
      f1:        0.999
      n_support: 83
      ci95_recall: [0.952, 1.000]
    # ... per entity type
  per_source_recall:
    wikipedia:  {PERSON: 0.91, EMAIL_ADDRESS: 0.99}
    forum:      {PERSON: 0.58, EMAIL_ADDRESS: 0.92}
    news:       {PERSON: 0.88, EMAIL_ADDRESS: 0.96}
    # ... per source

# §4 Redaction policy
redaction:
  policy_sha: 5d8e…
  per_entity:
    EMAIL_ADDRESS:
      primitive: mask
      operator: replace
      new_value: "<EMAIL>"
    PHONE_NUMBER:
      primitive: mask
      operator: replace
      new_value: "<PHONE>"
    CREDIT_CARD:
      primitive: mask
      operator: replace
      new_value: "<CARD>"
    US_SSN:
      primitive: mask
      operator: replace
      new_value: "<ID>"
    PERSON:
      primitive: surrogate
      operator: faker_consistent_within_document
      seed_source: "per-corpus seed from policy file"
    LOCATION:
      primitive: pass_through
      note: "LOCATION tokens at country / city granularity preserved; street-level ADDRESS masked"
    IP_ADDRESS:
      primitive: mask
      operator: replace
      new_value: "<IP>"
    INTERNAL_CUSTOMER_ID:
      primitive: mask
      operator: replace
      new_value: "<CUSTOMER_ID>"

# §5 Residual PII accounting
residual_pii:
  measurement_method: "production detector re-run over scrubbed slice (self-consistency check)"
  per_entity:
    EMAIL_ADDRESS:   {count_before: 11_840_112, count_after: 221, per_million_tokens_after: 0.0007}
    PERSON:          {count_before: 183_000_221, count_after: 47_118, per_million_tokens_after: 0.15}
    CREDIT_CARD:     {count_before: 8_412, count_after: 0, per_million_tokens_after: 0.0}
    US_SSN:          {count_before: 2_193, count_after: 0, per_million_tokens_after: 0.0}
    # ...
  per_source_residual:
    wikipedia:  {EMAIL_ADDRESS: 0.0001, PERSON: 0.08}
    forum:      {EMAIL_ADDRESS: 0.0021, PERSON: 0.42}
    news:       {EMAIL_ADDRESS: 0.0005, PERSON: 0.12}

# §6 Memorisation-risk delta (cites dedup report)
memorisation_risk:
  dedup_report_sha: 7f9c…
  pre_dedup:
    p50_duplicate_count: 1
    p99_duplicate_count: 23
    sequences_above_10x: 18_742
  post_dedup:
    p50_duplicate_count: 1
    p99_duplicate_count: 2
    sequences_above_10x: 173
  delta:
    sequences_above_10x_reduction: 0.9908
    note: "per Kandpal et al. 2022, extraction-attack success rate expected to scale approximately with duplicate-count distribution"

# §7 Quarantine manifest
quarantine:
  rule_set_sha: 6f12…
  rules:
    - id: high_pii_density
      condition: "finding_count_per_1k_tokens > 10"
      documents: 4_281
    - id: any_hipaa_identifier
      condition: "any finding of HIPAA identifier type"
      documents: 12_440
    - id: oracle_production_disagreement_high_confidence
      condition: "oracle_detector_high_confidence_finding not in production_detector_findings"
      documents: 332
  total_documents_quarantined: 17_053
  quarantine_store: "s3://trust-tier/quarantine/slice-42-2026-q3/"
  access_policy_sha: 9b77…

# §8 Data flow and retention
data_flow:
  raw_store: "s3://ingest-tier/slice-42-2026-q3/"
  scrubbed_store: "s3://training-tier/slice-42-2026-q3/"
  quarantine_store: "s3://trust-tier/quarantine/slice-42-2026-q3/"
  access_policy_sha: 9b77…
  retention_policy_sha: 3c19…
  training_release_approval_sha: pending

# §9 Data-subject-request process
dsr:
  procedure_sha: 2f45…
  endpoint: "privacy-requests@example.com"
  technical_workflow:
    - "Caller provides source URL or document identifier."
    - "Pipeline looks up doc_id in source manifest."
    - "If found, flag doc_id for exclusion from the next retraining cycle."
    - "If scrubbing is insufficient per counsel review, escalate to quarantine."
  machine_unlearning_handoff: fine-tuning-engineer

# §10 Lawful-basis evidence fields
lawful_basis:
  gdpr:
    art_6_proposed_basis: "6(1)(f) legitimate interests"
    lia_document_sha: 7a3b…      # counsel-authored LIA
    art_9_special_category_strategy: "quarantine documents containing health / religion / political / biometric / genetic findings"
    art_14_transparency_method: "public dataset card (mod-112) + DSR endpoint"
    art_14_5b_exception_cited: true
  ccpa_cpra:
    categories_of_pi: ["identifiers", "commercial information", "internet activity", "geolocation (coarse)"]
    sensitive_pi_strategy: "quarantine documents containing SPI findings"
    sale_or_sharing_classification: "to be determined by counsel"
    gpc_handling_at_scrape: "crawler respects GPC; see mod-103"
  hipaa:
    applicable: false
    rationale: "corpus is not clinical; HIPAA-identifier findings route to quarantine regardless"

# §11 Signatures and artifact SHAs
signatures:
  report_sha256: "<computed over the above sections>"
  sigstore_bundle_path: "signatures/privacy-risk-slice42-2026-11-04-abc1234.sigstore"
artifact_shas:
  scrubbed_slice:          "4a1b…"
  findings_table:          "2c91…"
  operator_result_log:     "6d33…"
  quarantine_manifest:     "8f22…"
  detection_config:        "9f3c…"
  redaction_config:        "5d8e…"
  eval_dataset:            "ab2c…"
  policy_note:             "fe84…"

# §12 Downstream sign-offs (empty slots filled by downstream owners)
sign_offs:
  privacy_counsel:
    signed: false
    reviewer: null
    signed_at: null
    opinion_document_sha: null
  ai_risk_engineer:
    signed: false
    reviewer: null
    signed_at: null
    attack_simulation_report_sha: null
  fine_tuning_engineer:
    signed: false
    reviewer: null
    signed_at: null
    training_release_approval: null
```

## Field-level notes

### §1 — Slice identity and provenance

The report is **per-slice**, not per-corpus. A slice is the unit of audit. Every field in §1 is a pointer to an upstream report — this is an **index** section, not a content section.

### §2 — Detection policy

Every recogniser appears. Every version, every threshold, every context-trigger list. The `config_sha` locks the full configuration, so the YAML is a human-readable summary that a reviewer cross-checks against the hash.

### §3 — Detection quality on eval set

Per-entity precision, recall, F1, support, and a bootstrap 95% CI. Per-source recall is mandatory — the number that matters is the number for the source where the risk is highest.

### §4 — Redaction policy

The Chapter 05 primitive-choice table in machine-readable form. Every choice is defensible; if a reviewer asks "why pass-through LOCATION?", the policy note (§11 `policy_note`) has the answer.

### §5 — Residual PII accounting

The single most-asked-about number. **Count before** is from the detector's initial run; **count after** is from a *re-run* of the detector over the scrubbed slice. The re-run is a **self-consistency check**: a detector that cannot find its own leftovers has a recall problem, not just a redaction problem. Per-source disaggregation is required.

### §6 — Memorisation-risk delta

Cites the dedup report. The numbers live there; this field is a reference, not a recomputation. The Kandpal 2022 interpretation lives in the `note` field so a reviewer can follow the citation.

### §7 — Quarantine manifest

The reason-coded set of documents dropped from the slice. Access to the quarantine store is **strictly tier-separated**; the access policy itself is hashed and referenced.

### §8 — Data flow and retention

Where the data lives, who has access, how long it is kept. The audit fields; counsel will ask.

### §9 — Data-subject-request process

The technical half of the DSR response. Counsel owns the policy; this field describes what the pipeline actually does.

### §10 — Lawful-basis fields

The structured handoff to counsel. Note the `lia_document_sha` and `opinion_document_sha` placeholders — the full legal analysis lives in a separate document, referenced by hash. This field is the structured summary; the document is the full text.

### §11 — Signatures and artifact SHAs

The Sigstore signature covers the whole report. Every referenced artifact has a SHA; the report plus the bundle plus the artifacts form a self-consistent audit set.

### §12 — Downstream sign-offs

Slots for the three peers. The report starts with them empty and is updated (or re-signed with a new version) as sign-offs arrive. The training-release decision is **the training-release approval slot in §8** — set by `fine-tuning-engineer` after sign-off.

## The signature step — Sigstore

The report is signed with Sigstore (`cosign sign-blob`). The signature covers the report file; the referenced artifact SHAs provide integrity for the pointed-to objects.

```
cosign sign-blob privacy_risk_report.yaml \
  --bundle signatures/privacy-risk-slice42-2026-11-04-abc1234.sigstore \
  --identity-token $(gcloud auth print-identity-token)
```

Verification:

```
cosign verify-blob privacy_risk_report.yaml \
  --bundle signatures/privacy-risk-slice42-2026-11-04-abc1234.sigstore \
  --certificate-identity-regexp '.*@example\.com' \
  --certificate-oidc-issuer https://accounts.google.com
```

[`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) Chapter 08 is the deeper Sigstore / SLSA treatment. The reason the privacy-risk report is signed is the same reason the dedup report is signed: it is **evidence**, and evidence is signed, not just produced.

## The lifecycle of the report

```
┌────────────────────┐                  ┌────────────────────┐
│ ai-data-engineer   │                  │ privacy counsel    │
│ authors §1–§9, §10 │ ───── sign ────▶ │ reviews §10,       │
│ (incomplete),      │    (v1)          │ authors LIA, signs │
│ §11, §12 (empty)   │                  │ off in §12         │
└────────────────────┘                  └────────────────────┘
          │                                       │
          │                                       ▼
          │                             ┌────────────────────┐
          │                             │ ai-risk-engineer   │
          │                             │ runs attack sim,   │
          │                             │ signs off in §12   │
          │                             └────────────────────┘
          │                                       │
          │                                       ▼
          │                             ┌────────────────────┐
          │                             │ fine-tuning-eng    │
          │                             │ sets DP recipe,    │
          │                             │ signs release      │
          │                             └────────────────────┘
          │                                       │
          ▼                                       ▼
   mod-109 lineage                         mod-112 dataset card
```

The report is re-signed when **any** of §1–§10 changes. Re-signing produces a new version in `mod-109`'s versioning substrate. Downstream consumers cite the report by `report_id` *and* SHA, so a stale pointer is detectable.

## Common mistakes

- **Reporting aggregate detection metrics instead of per-entity-type.** Every reviewer who has seen one audit knows to ask for the breakdown. Save them the back-and-forth.
- **Omitting per-source disaggregation.** Average recall is a lie. Low-recall sources are exactly where risk concentrates.
- **Reporting residual PII only from the original detector's findings.** The re-run self-consistency check (§5 `measurement_method`) is the number that matters.
- **Omitting the quarantine manifest.** Counsel often asks about it first. A missing manifest reads as "we quarantined nothing, so all of our documents passed" — which is almost never true.
- **Storing the LIA text inside the privacy-risk report.** Separate documents, referenced by SHA. The report is machine-readable; the LIA is prose.
- **Not re-signing on changes.** Every policy change is a new version. Silent edits of a signed document are an integrity violation.

## Downstream consumers

- [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md) **dataset card** consumes: detection policy summary, redaction policy summary, residual PII table, quarantine summary, lawful-basis header, signed-off-by list.
- [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) **lineage** consumes: the signed report as a first-class lineage event; its SHA is the pointer in the lineage graph.
- [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) **monitoring** consumes: the per-run detector-quality manifest for drift detection.
- **`fine-tuning-engineer`** consumes: the residual PII table, the memorisation-risk delta, the quarantine manifest.
- **`ai-risk-engineer`** consumes: the detection policy, the eval-dataset shape, the quarantine manifest.
- **Privacy counsel** consumes: §10 (lawful-basis fields), §5 (residual PII), §7 (quarantine), §8 (data flow).

## What this chapter is not

- It is not a template-you-can-copy. The schema above is a starting point for your organisation; your legal and risk functions will need fields the schema does not have. Treat the schema as a strong default, not a frozen format.
- It is not a legal instrument. The report is engineering evidence; the lawful-basis opinion is a separate document.
- It is not a one-time artifact. The report is re-authored for every slice release and re-signed whenever its content changes.

## Summary

- `privacy_risk_report.yaml` is the **single signed deliverable** of this module. It is machine-readable, Sigstore-signed, and referenced by SHA from downstream consumers.
- The schema has 12 sections: identity, detection policy, detection quality, redaction policy, residual PII, memorisation-risk delta, quarantine, data flow, DSR process, lawful-basis fields, signatures, downstream sign-offs.
- Every field is **versioned**, **hashed**, and **traceable** back to a Chapter 02–08 upstream artifact.
- The report's **re-run self-consistency check** on residual PII is the number counsel will ask about first; prioritise it.
- The report is **re-signed on every content change**. Silent edits violate the audit model.
- The report is consumed by the dataset card, the lineage substrate, the monitoring harness, and the three peer roles. No slice is training-ready without it.

Exercise 05 is the one-hour assembly of this report against your own slice. By the time you reach it, Exercises 01–04 have each contributed a block.
