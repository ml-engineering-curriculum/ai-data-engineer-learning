# mod-101 — AI Training-Data Governance Frameworks and the AI Data Engineer Position

**Role:** AI Data Engineer (level 30, ML Engineering family) · **Estimated effort:** 12 hours

## What this module teaches

Every governance framework you will meet on this track — NIST AI RMF, ISO/IEC 42001 / 23894 / 5259 / 25012, the EU AI Act, GDPR, CCPA/CPRA, EU DSM Article 4, SR 11-7, FDA GMLP/PCCP, Colorado AI Act — is a specification for the pipeline you own. This module teaches you to read each of them as an engineer: open the document, point at the pipeline artifact that satisfies the clause, and know how that artifact is produced and signed. It closes with a concrete scope diagram of this role on the level ladder so you know what to build, what to consume, and what to refuse.

## Learning objectives

- Read NIST AI RMF 1.0 (AI 100-1) + GenAI Profile (AI 600-1) + the NIST AI RMF Playbook as an engineer — mapping each MEASURE / MANAGE sub-category that lands on training-data integrity, provenance, representativeness, bias, and documentation to a concrete pipeline artifact.
- Read ISO/IEC 42001:2023 clause 8 + Annex A.7 (operational data controls), ISO/IEC 23894:2023 (risk sources that originate in data), ISO/IEC 5259 series (Data quality for analytics and ML — Parts 1 overview, 2 measures, 3 requirements, 4 process, 5 governance), and ISO/IEC 25012 (15-characteristic data-quality model) at a level sufficient to translate every clause into an engineering deliverable.
- Read EU AI Act Articles 10 (data governance for high-risk systems), 53 + Annex XI (training-data documentation for GPAI providers), and 72 (post-market monitoring, including dataset-side signals); position the EU AI Office GPAI Code of Practice.
- Read GDPR Articles 5 / 6 / 9 / 14 / 22; layer CCPA / CPRA 'sale or sharing' obligations and the EU DSM Copyright Directive Article 4 TDM-exception + opt-out as the data-licensing substrate.
- Position SR 11-7 / OCC 2011-12 model-risk management and FDA GMLP + PCCP as sector flavours; position Colorado AI Act (SB 24-205) as the US state-level deployer / developer shape.
- Diagram this role's exact scope on the level ladder — what `ml-engineer` (L20) already owns, what `ai-governance-analyst` (L15) hands you, what peers at L30 do on top, and what `training-pipeline-engineer` (L35) and `staff-ml-engineer` (L40) inherit upward.

## Chapters

1. [NIST AI RMF as a Training-Data Pipeline Spec](01-nist-ai-rmf-as-a-pipeline-spec.md) — reading AI 100-1 + AI 600-1 + Playbook as a pipeline artifact spec.
2. [The ISO Suite as Engineering Deliverables](02-iso-suite-for-training-data.md) — 42001 Annex A.7, 23894 Annex B, 25012, and the 5259 series as deliverables.
3. [The EU AI Act for Training Data](03-eu-ai-act-articles-10-53-72.md) — Articles 10, 53+Annex XI, 72; the GPAI Code of Practice.
4. [The Privacy and Copyright Substrate](04-gdpr-ccpa-and-dsm-article-4.md) — GDPR 5/6/9/14/22, CCPA/CPRA sale-sharing, DSM Article 4 TDM + opt-out, and the pipeline admit function.
5. [Sector and State Regimes](05-sector-and-state-regimes.md) — SR 11-7 / OCC 2011-12, FDA GMLP + PCCP, Colorado AI Act, and brief notes on NYC LL 144.
6. [The AI Data Engineer on the Level Ladder](06-role-scope-on-the-ladder.md) — scope boundaries vs. L15, L20, peer L30 roles, L35, L40.

## Exercises

1. [NIST AI RMF → Data Pipeline Artifact Map](exercises/exercise-01-nist-ai-rmf-to-data-pipeline-artifact-map.md)
2. [ISO/IEC 42001 Clause 8 + Annex A.7 and ISO/IEC 5259 Walk](exercises/exercise-02-iso-42001-clause-8-and-iso-5259-walk.md)
3. [EU AI Act Article 10 and Annex XI Deliverables](exercises/exercise-03-eu-ai-act-article-10-and-annex-xi-deliverables.md)
4. [GDPR / CCPA Lawful Basis and the Admit Function](exercises/exercise-04-gdpr-and-ccpa-lawful-basis-for-training-data.md)
5. [Role Scope and the Deferral Contract](exercises/exercise-05-role-scope-and-deferral-contract.md)

## Other assessment surfaces

- `labs/` — long-form hands-on labs (populated in a subsequent content cycle).
- `quizzes/` — knowledge checks (populated in a subsequent content cycle).

## References

See [resources.md](resources.md) for the primary sources and companion reading the module is grounded in.

## Where this module sits on the track

Mod-101 is the governance foundation that the rest of the track builds on. Every subsequent module — sourcing (102), acquisition (103), quality (104), dedup (105), decontamination (106), labelling (107), synthetic data (108), versioning (109), PII (110), monitoring (111), cross-functional program (112) — produces artifacts whose *shape* is set here. You will come back to mod-101 whenever a stakeholder cites a clause you have not yet translated into your pipeline.
