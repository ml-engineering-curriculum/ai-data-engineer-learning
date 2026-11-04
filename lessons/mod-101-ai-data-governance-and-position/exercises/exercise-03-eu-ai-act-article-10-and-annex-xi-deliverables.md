# Exercise 03 — EU AI Act Article 10 and Annex XI Deliverables

**Estimated effort:** 2 hours
**Chapter dependency:** [Chapter 03 — The EU AI Act for Training Data](../03-eu-ai-act-articles-10-53-72.md)

## Objective

Draft the EU AI Act data-side deliverables for two scenarios:

1. A **high-risk AI system** under Article 6 / Annex III — specifically, a credit-worthiness scoring model for a consumer-lending product (Annex III, point 5(b) — "access to and enjoyment of essential private services"). Produce the Article 10 artifact set for the training-data slice.
2. A **general-purpose AI model** — a 7B-parameter open-weights chat model trained on a mixed corpus. Produce the Annex XI Section 1 data slice and the skeleton of the Article 53(1)(d) public training-data summary.

The scenarios share many artifacts but differ in audience (regulator-for-the-system-provider vs. general-public) and shape.

## Prerequisites

- You have read Chapter 03.
- You have skimmed EU AI Act Articles 3, 6, 10, 51–56, 72, 73, and Annexes III, XI, XII. The EUR-Lex consolidated text of Regulation (EU) 2024/1689 is the primary source.
- You have at least one dataset-card template at hand (Hugging Face YAML, Google Model Cards Toolkit, or similar).
- You have reviewed the EU AI Office training-data summary template if published at read time; if not, note the gap. <!-- needs-research: pin the current AI Office template URL and version at read time. -->

## Problem statement

Two sub-tasks, parallel structure. Do them in order; the second reuses artifacts from the first.

### Sub-task A — Article 10 package for the credit-scoring slice

Build a folder `art10_credit/` containing:

- **`data_objective_brief.md`** — the intended-use / purpose declaration signed by the model owner and the product lead.
- **`source_manifest.yaml`** — list of data sources, licence tag, consent basis per source. For this exercise make the sources concrete (first-party loan-application data with explicit T&C basis, licensed credit-bureau data under contract, publicly-filed court-judgment data).
- **`preparation_log.yaml`** — every preparation step with parameters (PII-field drop, feature engineering, class balancing).
- **`assumptions_note.md`** — explicit assumptions about what the data represents and what it does not.
- **`sufficiency_statement.md`** — quantity, representativeness, and geographic-/contextual-fit statement scoped to the deployment population.
- **`bias_audit.md`** — representativeness and fairness audit covering at minimum protected-attribute coverage and filter-effect analysis.
- **`gaps_and_followups.md`** — known limitations and planned mitigations.
- **`article_10_cross_reference.md`** — a table mapping each paragraph of Article 10(2)–(5) to the artifact(s) above.

### Sub-task B — Annex XI data slice and Article 53(1)(d) summary skeleton

Build a folder `annex_xi_gpai/` containing:

- **`training_data_characterisation.md`** — narrative description for Annex XI Section 1 covering type and provenance of data, curation methodology, how data was obtained and selected, measures to detect unsuitable sources, measures to detect identifiable biases. Cross-reference the per-slice source manifests.
- **`public_training_data_summary.md`** — a draft of the public summary under Article 53(1)(d) using the AI Office template (or your best-effort skeleton if the template is not yet pinned). Sufficiently detailed to let rightsholders evaluate whether their content appears and whether their opt-out was honoured; not a full enumeration.
- **`copyright_policy.md`** — the Article 53(1)(c) policy covering DSM Article 4(3) reservation handling and the admit function's role (reference Chapter 04).
- **`annex_xi_cross_reference.md`** — mapping each enumerated item in Annex XI Section 1 to the artifact in your package.

## Starter guidance

- The credit-scoring scenario is *high-risk* under Annex III, not a GPAI model. Article 53 does not apply. Do not confuse the two scenarios.
- The public summary under Article 53(1)(d) is a *summary*, not a full corpus listing. Think "categories of sources, approximate proportions, licensing posture per category, opt-out treatment." Use the AI Office template if available.
- Article 10(5) (special-category processing for bias correction) is tempting but narrow. Do not invoke it in this exercise unless your scenario genuinely requires it; if you do, document the legal-basis chain.
- Do not write prose that reads like a legal brief. Write artifacts that an engineer could regenerate.
- Flag anywhere you have to guess with `<!-- needs-research: ... -->`.

## Deliverables

- Two folders `art10_credit/` and `annex_xi_gpai/` with the files listed above.
- **`handoff.md`** — one page in your voice as the AI Data Engineer describing which artifacts you own, which came from `ai-governance-analyst`, and which need legal signoff before publication.

## Acceptance criteria

- Every paragraph of Article 10(2) sub-items (a) through (h), plus Article 10(3) and 10(4), has a cross-reference entry pointing at a concrete artifact in `art10_credit/`.
- The Annex XI Section 1 data-side items (type and provenance of data, curation methodology, how obtained and selected, measures to detect unsuitable data sources, measures to detect identifiable biases) are each covered by an artifact in `annex_xi_gpai/`.
- The public training-data summary is between 400 and 1200 words, organised by *category of content* not *individual source*, with explicit treatment of licensing posture and opt-out handling.
- `handoff.md` cleanly separates "I own this" from "I consume this" from "legal must bless this."
- No invented facts about specific EU AI Office guidance versions — use `<!-- needs-research: ... -->` where the template or AI Office position is time-sensitive.

## Stretch goals

- Extend `art10_credit/` to include a Colorado AI Act developer-documentation packaging (Chapter 05) — the same artifacts tagged for a different regime.
- Produce the Article 53(1)(b) Annex XII deliverable (information for downstream providers) as a trimmed, integration-facing subset of your Annex XI package.
- Write an automation script (any language, pseudocode is fine) that generates `article_10_cross_reference.md` from the slice directory structure, so the mapping is maintained by code.
