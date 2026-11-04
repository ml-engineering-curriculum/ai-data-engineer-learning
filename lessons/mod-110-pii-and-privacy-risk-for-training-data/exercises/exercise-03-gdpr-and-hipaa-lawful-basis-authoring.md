# Exercise 03 — GDPR and HIPAA Lawful-Basis Evidence Authoring

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 08 — Lawful basis for training data](../08-lawful-basis-gdpr-ccpa-hipaa.md), [Chapter 09 — The privacy-risk report](../09-the-privacy-risk-report.md).

## Objective

Author the **engineering-evidence package** that counsel needs to form a lawful-basis opinion. Produce three artifacts: a GDPR Art 6(1)(f) evidence package for a hypothetical web-scraped slice, a CCPA/CPRA engineering summary, and a HIPAA Safe Harbor coverage table for a hypothetical clinical slice. Populate §10 of the Chapter 09 privacy-risk report.

**You are not writing a legal opinion.** Any assertion in your deliverables about whether the processing is lawful is out of scope. Your output is the engineering evidence that a counsel-authored LIA or DPIA would cite.

By the end you will have:

- A GDPR Art 6(1)(f) engineering-evidence package with the data-minimisation, safeguards, and purpose-limitation fields filled in.
- A CCPA/CPRA engineering summary with PI / SPI categories, sale-or-sharing analysis fields, and GPC-handling evidence.
- A HIPAA Safe Harbor coverage table mapping each of the 18 identifiers to detector-recogniser coverage and residual-count evidence.
- A data-flow diagram showing raw store / scrubbed store / quarantine store with access-control boundaries.
- A data-subject-request (DSR) technical-workflow description.
- A partial `privacy_risk_report.yaml` with §1–§5 (from Exercises 01–02) plus §7–§10 populated.

## Prerequisites

- Exercises 01 and 02 complete. The partial report, detection policy, and redaction policy are inputs.
- Chapter 08 read. You should be able to name the relevant GDPR Articles (6, 9, 14, 17, 25, 35), identify the two HIPAA de-identification methods, and locate the HHS OCR guidance.
- No cloud dependency. This exercise is paper-and-YAML; the regulatory text is the primary source.

## Problem statement

Author three engineering-evidence packages — one per jurisdiction — in a form a legal reviewer can consume:

1. **GDPR Art 6(1)(f) package** — legitimate-interests fields, data minimisation evidence, safeguards, purpose-limitation statement, Art 9 special-category strategy, Art 14 transparency method (and Art 14(5)(b) exception justification), Art 17 erasure process.
2. **CCPA/CPRA package** — PI / SPI category inventory, sale-or-sharing analysis stub, GPC-handling evidence, deletion-request workflow.
3. **HIPAA Safe Harbor coverage table** — one row per 18-identifier, with detector-recogniser mapping and residual-count citation.

## Starter guidance

### Part A — GDPR Art 6(1)(f) evidence package

1. **Assume a hypothetical slice**: 300B tokens of web-scraped English text from `.com`, `.de`, `.fr`, `.co.uk` TLDs, Q3 2026 crawl. Document the assumption.

2. **Fill the package fields.** In `gdpr_art_6_evidence.yaml`:

```yaml
lawful_basis_candidate: "6(1)(f) legitimate interests"
controller_identity: "Example Org, Inc."
purpose: "pretraining an open-weights English-language generative model"
data_minimisation:
  scrubbed_entity_types: [...]
  residual_pii_rate_per_million_tokens: ...
  quarantine_rules: [...]
  data_subject_specific_data_proactively_excluded: [...]
safeguards:
  detection_recall_floor: {...}
  redaction_primitives: {...}
  deduplication_coordination: "see mod-105 dedup_report_sha ..."
  dp_sgd_handoff: "fine-tuning-engineer will apply ... DP recipe"
  attack_simulation_handoff: "ai-risk-engineer will run ... extraction probes"
purpose_limitation:
  statement: "slice is released only to the training lane; retention policy ...; no secondary use"
  retention_days: ...
data_subject_rights:
  art_17_erasure_workflow_sha: ...
  art_15_access_response_time_days: ...
expectation_of_subjects:
  source_characterisation: "public web with robots.txt respect; see mod-103"
  data_subject_proactive_notice: "dataset card URL ..."
art_9_special_category_strategy:
  detector_coverage: {...}
  quarantine_rules: [...]
art_14_transparency:
  method: "public dataset card at ... + privacy-request endpoint"
  art_14_5b_exception_argument: "individual notice is disproportionate because ... [draft argument, counsel refines]"
```

3. **Justify** each field with a one-paragraph rationale in `gdpr_art_6_evidence.md`. Be explicit that the rationale is the engineering context; the balancing test is counsel's.

4. **Reference the LIA.** Include `lia_document_sha: <placeholder>` and note "to be authored by counsel". Your job is to make the LIA cheap to write by preloading the engineering fields.

### Part B — CCPA / CPRA engineering summary

1. In `ccpa_cpra_summary.yaml`:
   - List **PI categories** processed (per § 1798.140(v)): identifiers, commercial info, internet activity, geolocation (coarse / precise?), etc.
   - List **SPI subset** processed (per § 1798.140(ae)): precise geolocation, account credentials, government IDs, health, biometrics, racial / ethnic origin, religious / philosophical beliefs, union membership, communications content.
   - **Sale-or-sharing analysis stub**: describe the engineering facts (training on California-resident data, if any; model will be released commercially or open-weights) and mark the classification "to be determined by counsel".
   - **GPC handling**: document whether the crawler at ingest respects GPC, referencing [`mod-103`](../../mod-103-web-scale-acquisition-and-curation/README.md). If GPC is not respected at scrape, flag this and recommend coordination with the crawler team.
   - **Deletion workflow**: a short technical description of the pipeline-side response to a CCPA deletion request. Note the Delete Act (SB 362) one-stop implications.

2. One-page rationale in `ccpa_cpra_summary.md`.

### Part C — HIPAA Safe Harbor coverage table

1. **Assume a hypothetical clinical slice**: 2B tokens of de-identified clinical note text from a research dataset (e.g., shaped like MIMIC-IV). Document the assumption and why HIPAA might apply.

2. In `hipaa_safe_harbor.yaml`, author a table with one row per each of the 18 identifiers from 45 CFR § 164.514(b)(2):

```yaml
safe_harbor_coverage:
  - identifier: "(A) Names"
    presidio_entity_type: PERSON
    recall_on_eval: 0.74              # from Exercise 01 §3
    redaction_primitive: surrogate    # from Exercise 02 §4
    residual_count: 47118             # from Exercise 02 §5
    residual_rate_per_million_tokens: 0.15
    quarantine_rule: "docs with > 10 findings/1k tokens"
    coverage_adequacy_note: "English PERSON NER recall is ... — insufficient for Safe Harbor on its own; coordinate with Expert Determination path"
  - identifier: "(B) All geographic subdivisions smaller than a state"
    ...
  # ... 18 rows
```

3. In `hipaa_safe_harbor.md`, discuss:
   - Where Safe Harbor is cleanly supported by the detector (structured identifiers).
   - Where it is weakly supported (names, free-text addresses).
   - Whether Expert Determination is a more realistic path for the slice, and what statistical-risk evidence an expert would need.
   - Explicit disclaimer that the engineering evidence does not constitute a de-identification determination.

### Part D — Data-flow diagram and DSR workflow

1. **Data-flow diagram** in `data_flow.md` (ASCII diagram is fine):
   - Raw ingest store (access: data-engineering lane + security audit).
   - Scrubbed store (access: training lane + data-engineering audit).
   - Quarantine store (access: counsel + security only; no training lane access).
   - Audit log store (write-once, retention policy documented).

2. **DSR workflow** in `dsr_workflow.md`:
   - Caller endpoint.
   - Technical lookup: how a document is located given a source URL or identifier.
   - Decision tree: redact / exclude / retrain-flag / quarantine.
   - Machine-unlearning handoff to `fine-tuning-engineer`.
   - Service level: time from request to completion.

### Part E — Assembling §7–§10 of the privacy-risk report

Merge the above into the partial `privacy_risk_report.yaml` from Exercise 02. After this exercise §1–§5, §7, §8, §9, §10 are populated. §6 (memorisation-risk delta) and §11 (signatures) are filled in Exercises 04–05.

## Deliverables

```
exercise-03/
  gdpr_art_6_evidence.yaml
  gdpr_art_6_evidence.md
  ccpa_cpra_summary.yaml
  ccpa_cpra_summary.md
  hipaa_safe_harbor.yaml
  hipaa_safe_harbor.md
  data_flow.md                 # ASCII diagram + access-control table
  dsr_workflow.md
  privacy_risk_report.yaml     # §1-§5 (from Ex 01-02) + §7-§10 populated; §6 + §11-§12 placeholders
  README.md                    # scope disclaimer + "how to hand this to counsel" summary
```

## Acceptance criteria

- **Scope disclaimer**: the README leads with "this is engineering evidence, not legal opinion", and each regulation-specific artifact repeats the disclaimer.
- **GDPR package** includes all required fields: Art 6(1)(f) balancing-relevant evidence, Art 9 strategy, Art 14 transparency method with Art 14(5)(b) argument marked as a draft for counsel, Art 17 workflow, Art 25 by-design statement.
- **CCPA/CPRA summary** includes PI category inventory, SPI subset, sale-or-sharing classification *stub* (not conclusion), GPC-handling evidence, deletion workflow.
- **HIPAA Safe Harbor table** has 18 rows (one per § 164.514(b)(2) identifier). Each row has: entity-type mapping (or explicit "not covered"), recall-on-eval citation, redaction primitive, residual count, coverage-adequacy note.
- **Data-flow diagram** names the three stores with their access-control boundaries.
- **DSR workflow** has a documented technical response path with time SLA.
- **No legal conclusions**: nothing in the deliverables asserts that the processing is lawful under any regulation. The language is "evidence for counsel to assess", not "this is lawful".
- Every fact-claim in the deliverables that cannot be verified is marked `<!-- needs-research: ... -->`.
- `privacy_risk_report.yaml` conforms to the Chapter 09 schema for §7–§10.

## Stretch goals

- **Add a second jurisdiction** — Brazil LGPD, UK GDPR, or Singapore PDPA. Author the engineering-evidence summary at the same rigour.
- **CJEU C-184/20 implications**. Write a one-paragraph note on how the ruling (information that *reveals* a special category is itself special-category) affects your Art 9 strategy.
- **Mock an LIA from the engineering evidence**. Draft a counsel-style balancing test using only the engineering fields you produced, then explicitly mark it "mock" and discuss what a real LIA would add. Useful for learning the shape; must not be mistaken for an actual LIA.
- **Expert Determination packet skeleton**. For the HIPAA clinical slice, draft the structure of an Expert Determination packet (statistical-risk assessment, intended recipient's capabilities, re-identification-risk model) — not the content.

## What this exercise is not

- It is **not legal advice**. Nothing you produce is a determination of lawful basis, Safe Harbor compliance, or sale-or-sharing classification.
- It is not the detection exercise (Exercise 01) nor the redaction exercise (Exercise 02).
- It is not the attack-probe exercise (Exercise 04).
- It is not the full-report-assembly exercise (Exercise 05).

It is the regulatory-evidence authoring exercise. Every field is machine-readable so counsel can read and act on the YAML without re-doing your engineering work.
