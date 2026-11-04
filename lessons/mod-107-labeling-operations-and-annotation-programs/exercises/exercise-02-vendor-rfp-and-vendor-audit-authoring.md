# Exercise 02 — Vendor RFP and Vendor-Audit Authoring

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 03 — Project configuration, review queues, consensus, and adjudication](../03-project-configuration-review-consensus-adjudication.md), [Chapter 05 — Vendor labelling: RFP, calibration set, QA sample, SLA, arbitration](../05-vendor-labelling-rfp-calibration-sla-arbitration.md)

## Objective

Author a **full labeling-vendor RFP + audit package** for a plausible scenario. Produce every document in the `vendor_package/` block of the program-package tree (Chapter 01):

- `rfp.md` — the RFP document itself.
- `sla.md` — the measurable quality + volume + responsiveness SLA.
- `arbitration.md` — the named escalation and arbitration path.
- `audit_sample_policy.md` — the per-item QA sample policy and pass-rate thresholds.
- `audit_sample.parquet` — a 300-item audit sample against the calibration set from Exercise 01.
- `vendor_comparison.md` — a one-page evaluation matrix of three of the seven in-scope vendors against the RFP.

This is a **writing exercise**. No labeling tool is needed. The deliverables are documents that could be sent to Legal + Procurement, and an audit sample that could be reviewed by a real reviewer pool.

## Prerequisites

- Chapter 05 read.
- Chapter 03's guidelines document and calibration set from Exercise 01.
- Access to each of the seven vendors' public documentation / pricing pages (Scale, Surge, Appen, Sama, iMerit, Toloka, SuperAnnotate). <!-- needs-research: confirm current public URLs at authoring time — vendor pages change frequently. -->
- Plain Markdown; no code required.

## Problem statement

Pick a scenario. One of:

- **A.** 50k-item text classification for a customer-support slice (reuse Exercise 01's slice and guidelines). 4 classes, English-only. Target IAA Krippendorff's α ≥ 0.6.
- **B.** 20k preference-pair labeling for an RLHF reward model. 8-axis Likert rubric. Target α ≥ 0.5 per axis.
- **C.** 10k multilingual safety-category labeling across 5 languages. 7 categories including CSAM-adjacent classes.

Scenario C explicitly exercises the workforce-welfare section (Chapter 05 §"Workforce welfare — operational, not aspirational"). Pick B or C if you want to practise the harder parts.

## Starter guidance

### 1. Sketch the RFP skeleton

Use the eight-section skeleton from Chapter 05 §"The RFP — what to include". Keep each section 150–500 words — short on purpose. Do not inline the guidelines; reference them by SHA.

### 2. Write the SLA with teeth

The SLA has three dimensions (quality, volume, responsiveness). For each:

- Name the **metric** (e.g. "Krippendorff's α against gold calibration").
- Name the **target** (e.g. "≥ 0.60 on each rolling 1,000-item window").
- Name the **measurement procedure** (who, when, from what items).
- Name the **miss consequence**, escalating from written notice to service credit to termination right.

See Chapter 05 §"The quality SLA — writing it with teeth" for the structure.

### 3. Write the arbitration path

Structure per Chapter 05 §"Escalation and arbitration — the named path". Four levels (L0 → L3), each with a deadline, named participants, and a decision rule. Name an arbiter — fictional is fine, but it must be a named role and a named backup.

### 4. Build the audit-sample policy

Three sampling rules from Chapter 05 §"The per-item QA sample":

- Random 10% stratified by class.
- Oversample low-prevalence classes.
- 100% on flagged items (disagreement, PII, harmful content).

Then build `audit_sample.parquet`: 300 items drawn from the calibration set by the three rules above (describe the proportions). Each item should have a column `audit_reason ∈ {random, stratified_low_prev, flagged_pii, flagged_harm, flagged_disagreement}`.

### 5. Evaluate three vendors against the RFP

Pick three of the seven. Write `vendor_comparison.md` as a one-page table with rows for each of the RFP's main sections (volume, modality, platform, workforce transparency, security, commercial) and columns for each vendor. Fill in from public documentation.

**You will not be able to fill every cell from public sources.** Mark unknowns with `?` and note "ask at RFP stage". This honesty is the point: a real procurement spreadsheet has these gaps.

### 6. Pick a winner (with reasoning)

In `vendor_comparison.md`'s final section, recommend one vendor for this scenario. Include three reasons based on your table.

## Deliverables

```
exercise-02/
  rfp.md
  sla.md
  arbitration.md
  audit_sample_policy.md
  audit_sample.parquet
  vendor_comparison.md
  README.md
```

## Acceptance criteria

- `rfp.md` has all eight sections from Chapter 05 §"The RFP — what to include". Each section has concrete, scenario-specific content — no "TBD" placeholders.
- `sla.md` has at least three SLAs (quality, volume, responsiveness), each with metric + target + measurement + consequence.
- `arbitration.md` has four levels (L0–L3) with named deadlines and named roles.
- `audit_sample.parquet` has 300 rows with an `audit_reason` column.
- `audit_sample_policy.md` explains, in prose, how the sample was constructed and what pass-rate thresholds apply at each tier.
- `vendor_comparison.md` evaluates three of {Scale, Surge, Appen, Sama, iMerit, Toloka, SuperAnnotate}.
- Any claim about a vendor's current offering that you could not verify from a cited URL at authoring time is marked `<!-- needs-research: ... -->`. Do **not** invent facts about vendors.

## Stretch goals

- **Scenario C with explicit welfare section.** Write the welfare-attestation clauses into `rfp.md` using the Chapter 05 §"Workforce welfare" skeleton.
- **Write a termination letter.** Short — one page — exercising the "right to terminate on repeated misses" clause from the SLA. This forces you to tighten the SLA language until it actually supports the letter.
- **Draft a DPA addendum.** One page, specific to the vendor you picked. Pulls on any regulated-industry or privacy prerequisites.
- **Run an actual vendor pilot.** For Toloka (which has a self-service model with no minimum), launch a 500-item pilot on the audit sample and measure your real-world agreement. This bleeds into Exercise 03.

## What this exercise is not

- Not a tool-configuration exercise (Exercise 01).
- Not a labeling exercise. You do not label anything in this exercise; you build the contract and audit framework around labeling.
- Not a tutorial on each vendor. Three evaluated in a paragraph each is enough.

The deliverable is the **reusable vendor package**. When the real procurement moment arrives, you tune this template to the live scenario — a half-day of work, not a week.
