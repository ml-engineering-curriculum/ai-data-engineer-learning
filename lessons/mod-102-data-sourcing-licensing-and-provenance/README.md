# mod-102 — Data Sourcing, Licensing, Consent, and Provenance Engineering

**Role:** AI Data Engineer (level 30, ML Engineering family) · **Estimated effort:** 16 hours

## What this module teaches

Every slice you build has to answer three questions at audit time: *Where did this data come from? Under what terms are you allowed to train on it? And can you prove it?* This module turns those three questions into pipeline artifacts. You will learn to catalogue source types and author a defensible selection matrix, encode counsel's licence posture as an allowlist / denylist policy, honour the growing family of AI-training opt-out signals, position the EU DSM Article 4 and US copyright substrate, and produce the Datasheets, Data Statements, Croissant, ML-BOM, SPDX 3.0, and C2PA records that make the pipeline end-to-end traceable. Everything in this module is a precondition for the regulator-facing package that the evaluation engineer (level 35) delivers.

## Learning objectives

- Catalog data-source types — public web crawl, licensed partner corpora, owned user-generated content, third-party datasets (Hugging Face Hub, Kaggle, Zenodo, academic releases), and synthetic — and author a defensible selection matrix per training objective.
- Author a data-licence matrix the ML program will defend under audit — Creative Commons tier mapping (CC-BY, CC-BY-SA, CC-BY-NC, CC-BY-ND, CC0), proprietary licences, commercial-use carve-outs, redistribution restrictions; the allowlist / denylist policy and how it binds the pipeline.
- Honour crawling and training opt-out contracts — robots.txt per RFC 9309 (with Disallow + Crawl-delay semantics), ai.txt / llms.txt / `noai` / `noimageai` meta tags, Cloudflare AI-opt-out, Google-Extended, GPTBot, CCBot, ClaudeBot, PerplexityBot — and watch the IETF ai-pref working group's standards-track evolution.
- Honour EU DSM Copyright Directive Article 4 TDM-exception opt-out signals (rightsholder machine-readable reservation); understand the US Copyright Office position and the open litigation landscape on training-data copyright; route legal opinion out of scope to counsel and package the factual record this role contributes.
- Author per-source Datasheets for Datasets (Gebru et al. 2018), Data Statements for NLP (Bender and Friedman 2018), Dataset Nutrition Labels, and publish Croissant (MLCommons) and Hugging Face Dataset Card YAML metadata so consumer platforms index the dataset correctly.
- Emit a CycloneDX ML-BOM entry and an SPDX 3.0 AI-profile record per sourced dataset so the ML supply chain is end-to-end traceable; layer C2PA Content Credentials and the Content Authenticity Initiative provenance records on multimodal sources where the upstream publisher provides them.
- Coordinate with `ai-governance-analyst` (level 15) on the data-protection impact assessment (DPIA) they draft from this evidence, with `ai-risk-engineer` (peer, level 25) on data-side harm modelling, and with `ai-evaluation-engineer` (next-up, level 35) on the regulator-facing package this evidence feeds into.

## Chapters

1. [Data-Source Types and the Per-Objective Selection Matrix](01-data-source-types-and-selection-matrix.md) — the five canonical source types, the seven matrix columns, and the per-objective blend patterns.
2. [The Data-Licence Matrix and the Allowlist / Denylist Policy](02-license-matrix-and-allowlist-denylist.md) — CC tiers, OSS licences, proprietary contracts, platform ToS, public-domain; the signed policy document and the `LicenseGate`.
3. [Honouring Crawling and Training Opt-Out Contracts](03-crawling-and-training-opt-out-contracts.md) — robots.txt (RFC 9309), ai.txt, llms.txt, `noai` / `noimageai` meta, Cloudflare AI-opt-out, the AI-specific user agents, the IETF ai-pref direction.
4. [EU DSM Article 4 Reservations, US Copyright Posture, and the Legal Deferral Contract](04-dsm-article-4-and-us-copyright-posture.md) — Article 4 TDM + reservations, Article 53(1)(c) copyright policy, US fair-use posture, the deferral contract with counsel.
5. [Dataset Documentation: Datasheets, Data Statements, Nutrition Labels, Croissant, and Hugging Face Cards](05-dataset-documentation-datasheets-and-croissant.md) — the five-artifact per-slice documentation set.
6. [ML-BOM, SPDX AI, and C2PA: End-to-End Supply-Chain Traceability](06-ml-bom-spdx-ai-c2pa-provenance.md) — CycloneDX ML-BOM, SPDX 3.0 Dataset + AI profiles, C2PA Content Credentials.
7. [Peer Coordination: Governance Analyst, Risk Engineer, and Evaluation Engineer](07-peer-coordination-governance-risk-evaluation.md) — the DPIA input bundle, the data-risk register slice, the Article 53(1)(d) summary input.

## Exercises

1. [Data-Source Selection Matrix for One Training Objective](exercises/exercise-01-data-source-selection-matrix-for-one-training-objective.md) — the signed per-objective matrix.
2. [Licence Allowlist and Denylist Policy Authoring](exercises/exercise-02-license-allowlist-and-denylist-policy-authoring.md) — the signed policy YAML and the `LicenseGate` enforcer with tests.
3. [Opt-Out Protocol Honouring robots.txt, ai.txt, llms.txt, and the noai family](exercises/exercise-03-opt-out-protocol-honouring-robots-ai-llms-noai.md) — the evaluator, fixture set, and crawl contract.
4. [Datasheets for Datasets and Data Statement Authoring](exercises/exercise-04-datasheets-for-datasets-and-data-statement-authoring.md) — Datasheet + short-form Data Statement + Nutrition Label.
5. [Croissant and ML-BOM Metadata Emission](exercises/exercise-05-croissant-and-ml-bom-metadata-emission.md) — the five-record machine-readable set (Croissant, HF card, CycloneDX ML-BOM, SPDX 3.0, C2PA index).

## Other assessment surfaces

- `labs/` — long-form hands-on labs (populated in a subsequent content cycle).
- `quizzes/` — knowledge checks (populated in a subsequent content cycle).

## References

See [resources.md](resources.md) for primary sources and companion reading.

## Where this module sits on the track

Mod-102 sits directly on top of mod-101's governance foundation. It compiles the legal postures that mod-101 reads into the pipeline-side artifacts the rest of the track (103 acquisition, 104 quality, 105 dedup, 106 contamination, 107 labelling, 108 synthetic, 109 versioning, 110 PII, 111 monitoring, 112 program) iterates and signs. Every downstream module produces evidence that composes with the selection matrix, the licence policy, the opt-out log, the Datasheet, and the ML-BOM authored here. If you only remember one sentence from mod-102, make it this: *the slice is signed, and its signature stands on this module's artifacts*.
