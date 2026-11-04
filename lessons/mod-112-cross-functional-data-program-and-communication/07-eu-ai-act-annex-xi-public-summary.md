# Chapter 07 — The EU AI Act Article 53 + Annex XI Public Training-Data Summary

## Why this chapter exists

Article 53(1)(d) of the EU AI Act (Regulation (EU) 2024/1689) imposes a specific obligation on providers of general-purpose AI (GPAI) models: they must *"draw up and make publicly available a sufficiently detailed summary about the content used for training of the general-purpose AI model, according to a template provided by the AI Office."*

That is a *separate document* from the HF Dataset Card, the Google Data Card, and the Meta/Llama-style Data Card. It has a prescribed template, a prescribed audience (the public, with specific attention to rights-holders), and a prescribed level of detail (*sufficiently detailed* in the regulator's judgement, elaborated in the General-Purpose AI Code of Practice). This chapter is how you, the AI Data Engineer, draft that document.

The private companion document — the **Annex XI technical documentation** — is not public and goes to the AI Office directly (and to downstream providers under the Annex XII package). The data-engineering slice of both lives in the same evidence base, but the two have different audiences and different shipping surfaces. This chapter covers both, with emphasis on the public summary.

## The regulatory landscape in one page

### Which models are in scope

- **GPAI model** — a general-purpose AI model as defined in Article 3 and expanded in Article 51. Characterised by *significant generality* and ability to perform a wide range of distinct tasks. Current examples include large language models and foundation models trained on broad corpora.
- **GPAI model with systemic risk** — a GPAI model that meets the threshold of Article 51 (as of the current text, primarily an indicative compute threshold around 10^25 FLOPs, with other factors the Commission can use to designate specific models). Systemic-risk models carry **additional** Article 55 obligations on top of Article 53.

If your organisation ships a GPAI model, Article 53 applies. If the model is designated systemic-risk, Article 55 applies on top.

### Timeline

Article 53 applies from **2 August 2025**. Article 55 systemic-risk obligations apply from the same date. The AI Office began publishing the official public-summary template in advance of that date to allow providers to prepare. <!-- needs-research: cite the official AI Office page for the template, including version and publication date at the time the student reads this chapter. -->

The **General-Purpose AI Code of Practice** is the voluntary code that specifies how providers interpret Article 53 obligations in practice. Signing the Code is not required by the Act but is the current best available guide to AI Office expectations; signatories commit to following the Code of Practice's specific methods. <!-- needs-research: cite the Code of Practice page and current signatory list at reading time. -->

### Article 53's four obligations (quick recap from mod-101)

- **(a)** Technical documentation of the model — detail in **Annex XI**.
- **(b)** Information for downstream providers — detail in **Annex XII**.
- **(c)** Policy to comply with Union copyright law including the DSM Article 4(3) TDM reservation.
- **(d)** **Public summary of the training content** — per the AI Office template.

This chapter focuses on (d), the public summary, and on (a) the Annex XI package, both of which are data-engineering deliverables.

## The public summary — what the template asks for

The AI Office's public-summary template is published in a structured form (likely a fillable PDF or a web form backed by a JSON Schema). The template version at the time of reading governs the exact field names; the structure below is the content the template consistently asks for.

### Section 1 — Provider and model identification

- Legal name of the provider.
- Model name and version.
- Model identifier (the SHA or version string published alongside the model).
- Model-release date.
- Contact point for rights-holders and other interested parties.

### Section 2 — High-level description of the training data

A narrative paragraph describing the training mix at the source-class level. Shape (parallel to the Meta Data Card's § 1):

> `foundation-model-v4` was trained on approximately 15 trillion tokens of publicly available web data (Common Crawl-derived), licensed datasets under bilateral partnerships, open-source datasets (CC-BY and CC-BY-SA class), code repositories under permissive licences, and internally-generated synthetic data. Training-data collection ran from 2024-08 through 2026-10-23.

Non-marketing. Specific.

### Section 3 — Data sources by class, with licence posture

A table of source classes with:

- Description of the class.
- Approximate share of training tokens.
- Licence posture (public web with robots/TDM-reservation honoured, bilateral licence, open-source SPDX class, synthetic).
- Collection window or cutoff date.

Example:

| Source class | Approx. share | Licence posture | Collection window |
|---|---|---|---|
| Common Crawl-derived web text | ~62% | Public web, robots-TDM reservation + DSM Art 4(3) opt-out honoured | 2024-08 — 2026-10-23 |
| Licensed publisher data | ~14% | Bilateral licence agreements (specific partners under NDA; posture: training rights granted explicitly) | 2024-09 — 2026-09-30 |
| Open-source datasets | ~11% | CC-BY-4.0, CC-BY-SA-4.0, MIT-equivalent; per-dataset manifest available internally | 2024-10 — 2026-08-15 |
| Code repositories | ~8% | Permissive OSS licences (MIT, Apache-2.0, BSD-3, ISC); copyleft repos excluded | 2024-08 — 2026-10-10 |
| Synthetic data | ~5% | Generated by Acme models under internal validation | 2026-07 — 2026-10-01 |

The template may require specific SPDX identifiers for "open-source" classes and may ask for a separate line on "scraped from the public internet" vs. "received under bilateral agreement". Follow the current template's field names exactly.

### Section 4 — Methods used to comply with EU copyright law

The Article 53(1)(c) obligation lives here operationally:

- How the DSM Article 4(3) TDM reservation is detected and honoured (robots.txt parsing, structured-metadata parsing, explicit opt-out lists).
- How EU-opt-out signals propagate through the pipeline (admissions-gate enforcement, re-checks on crawl refresh).
- How rights-holder opt-out requests are received and actioned (named contact point, response SLA).
- How detection errors are handled (false positives: honouring more opt-outs than required; false negatives: incident response when content opted-out is later found in the corpus).

This section is where the data-engineer's Common-Crawl-pipeline enforcement work from mod-102 and mod-103 meets the public obligation. A reviewer reading this section is checking: is there a *process*, is it *enforceable*, and is there *evidence* it was followed?

### Section 5 — PII and sensitive content handling

- PII categories targeted for removal.
- Detector / tool set used.
- Residual-rate against an internal evaluation set.
- Exceptions explicitly invoked (e.g. public-interest exception under Article 10(5)).

Short section. One to two paragraphs plus a detector table.

### Section 6 — Data-quality and known-bias summary

- Quality filters applied (perplexity-based, heuristic, boilerplate removal, dedup, decontamination against eval sets).
- Known biases in the training mix (per-language quality variance, filter-induced effects, source-type imbalances).
- Mitigations and residual acknowledgement.

The honest-about-biases section from the Meta Data Card belongs here in the public summary as well. Chapter 06's § 8 Known Biases content maps approximately 1:1.

### Section 7 — Contact for rights-holders and other requests

- Named contact point (role; may be a mailbox such as `ai-compliance@acme`, not a personal email).
- Expected response SLA.
- The process for a rights-holder to:
  - Verify whether their content was included.
  - Request an opt-out (prospectively).
  - Request remediation (if content was included in violation of posture).

This section is the operational *trust* layer — the public reader checks whether the provider is reachable and responsive. Make the contact point real and the SLA honest.

## What "sufficiently detailed" means in practice

Article 53(1)(d) uses the qualifier "sufficiently detailed" without defining it in the statute. The Code of Practice elaborates. Three practical thresholds:

1. **A rights-holder can plausibly assess whether their content was used.** At the source-class level, with licence posture and TDM-reservation enforcement explained, a rights-holder in the EU can look at the summary and know whether the provider's methodology would have included their content. Full per-URL disclosure is not required, but the process disclosure must be specific enough for the assessment.
2. **An academic or civil-society reviewer can identify the *kinds* of content in the mix.** Not what *specific* data point is present, but whether code, medical text, social-media, child-directed content, licensed news, or other named categories are in the mix.
3. **A regulator can audit the stated process.** Every claim in the public summary must be matched by evidence in the Annex XI private package. A regulator can request the private package and verify.

A summary that fails any of these three is "insufficiently detailed" by the Code of Practice standard. <!-- needs-research: cross-reference the exact Code of Practice language on sufficiency at reading time; signatory commitments may further specify. -->

## The Annex XI private package — what goes to the AI Office

Annex XI, Section 1, lists the minimum content of the GPAI technical documentation. The data-engineering slice of that package (what you author or co-author) is roughly:

- **Data sources and provenance** — detailed list of source classes with per-class provenance, including per-dataset manifests for licensed and open-source classes. May include per-URL lists for licensed packs where contracts permit.
- **Curation methodology** — detailed preparation-log entries for every filter, dedup, decontamination, PII scrub, synthesis, and quality step. Parameter values, reference thresholds, incident log.
- **Measures to detect unsuitable data sources** — the admission function (mod-102), the licence posture matrix, the Article 10(5) special-category handling (if applicable).
- **Measures to detect identifiable biases** — the bias audit, the fairness-slice evaluation, the mitigations applied.
- **Decontamination report** — full `contamination_report.yaml` with per-eval-set measurement and firewall actions.
- **Privacy risk report** — full `privacy_risk_report.yaml` with per-detector precision/recall and residual-rate audit.
- **Lineage attestation** — OpenLineage runs, DataHub / Unity Catalog references, Sigstore signatures with Rekor entries.

For GPAI models with systemic risk (Annex XI Section 2), add:

- Evaluation strategies and internal/external testing results (coordination with `ai-evaluation-engineer` at L35 — see Chapter 09).
- Adversarial / red-team results (coordination with `ai-risk-engineer` at L25 and the model-safety team).
- Energy consumption and computational resources (coordination with `ai-infra-mlops` and `training-pipeline-engineer`).

The data-engineering slice of the Annex XI package is **exactly the evidence base you already have**. The authoring step is arranging it with the Annex XI section headings and cross-references to the public summary.

## How the public summary and the private package are consistent

The regulator's primary audit question is *"does the public summary accurately describe what the Annex XI package contains?"* Three disciplines:

### 1. Single evidence base, two renderings

As in Chapter 03's dataset-card pattern, author one evidence base keyed by Datasheet question number *plus the Annex XI section numbers*. Render the public summary and the Annex XI package from that base with different templates. A fact changed once changes in both.

### 2. Figures in the public summary match figures in the private package

If the public summary says "~62% web text", the Annex XI package shows the exact figure (say, 61.7%). Rounding in the public document is acceptable; contradiction is not.

### 3. Every public assertion has a private citation

The public summary makes claims. The Annex XI package backs each claim with evidence. In the evidence base, every answer carries an `evidence_refs:` list. A reviewer traversing the chain finds the specific `openlineage_run`, the specific `preparation_log` entry, the specific `bias_audit` table.

Unverifiable public assertions are the fastest route to "insufficiently detailed" findings.

## The authoring workflow — who does what

The public summary is a cross-functional deliverable. Who contributes what, at what point in the release cycle:

| Role | Contribution | When |
|---|---|---|
| **AI Data Engineer (you, L30)** | Section 2 (high-level description), Section 3 (source-class table), Section 5 (PII), Section 6 (quality/bias), Annex XI data-engineering evidence slice | Authors the evidence base continuously; drafts the summary at release time |
| **AI Governance Analyst (L15)** | Section 1 (provider/model identification), Section 7 (contact point), consistency check against inventory and DPIA | Pre-release review |
| **Legal counsel** | Section 4 (copyright compliance), licence-posture statements across § 3 | Pre-release legal sign-off |
| **AI Risk Engineer (L25)** | Known-bias and residual-risk framing in § 6 | Pre-release risk review |
| **AI Evaluation Engineer (L35)** | Decontamination firewall evidence and eval-contamination table for § 6 and Annex XI | Pre-release evidence package |
| **Governance council / CAIO** | Approval of the public summary pre-publication | Pre-release sign-off |

The data engineer owns authorship of the data-content-of-the-summary; legal owns the copyright-compliance section; governance owns the identification section and the overall sign-off. Make sure the RACI from Chapter 01 names this explicitly.

## Common authoring mistakes

Five patterns that show up in weak public summaries:

1. **The summary reads as a Data Card.** Authored by a team that confused the two formats, the summary over-indexes on technical characterisation and under-indexes on compliance process (§ 4 Copyright, § 7 Contact for rights-holders).
2. **Missing or vague copyright-compliance section.** This is a specific obligation. "We honour applicable law" is not sufficient; the TDM-reservation enforcement mechanism must be described.
3. **Numbers in the public summary that do not match the Annex XI package.** Even a 2-percentage-point discrepancy between "~60%" public and "58.1%" private is an audit flag.
4. **No real contact point.** A mailbox that is not monitored, or a role that does not exist, destroys trust and becomes an enforcement point.
5. **Stating that no PII was present.** As in the Meta Data Card, false and the reviewer knows it. State the rate.

## Interaction with other formats

The public summary is not a replacement for any of the three Data Card formats. In a typical GPAI release:

- The **HF Dataset Card** ships per internal slice where applicable (a slice that exists as its own artifact on an internal or external Hub).
- The **Google Data Card** ships in the internal governance package (for the design-review and council audience).
- The **Meta / Llama-style Data Card** ships as the public companion to the model card (for the external community and research audience).
- The **Article 53 public summary** ships as the public regulatory document (for rights-holders and the AI Office audience, with a prescribed template).
- The **Annex XI technical documentation** ships to the AI Office directly and to downstream providers under the Annex XII package.

All five render from the same evidence base. A release that ships five documents from five separate authoring efforts has a maintenance debt that compounds; a release that renders five from one evidence base pays the integration cost once and reaps consistency forever.

## Summary

- Article 53(1)(d) of the EU AI Act requires GPAI providers to publish a **sufficiently detailed summary** of training content, using the **AI Office's template**. This is a distinct document from any of the three Data Card formats.
- The public summary has 7 standard sections: provider/model identification, high-level description, source classes with licence posture, methods for copyright compliance (DSM Art 4(3) honouring), PII and sensitive-content handling, data-quality and known-bias summary, contact for rights-holders.
- "Sufficiently detailed" means: a rights-holder can assess whether their content was used; a reviewer can identify the kinds of content in the mix; a regulator can audit the stated process against the private package.
- The **Annex XI technical documentation** is the private companion that backs the public summary. The data-engineering slice is the full evidence base from mod-102 through mod-111.
- Consistency between the public summary and the Annex XI package is audited: single evidence base, matched figures, every public assertion with a private citation.
- Authoring is cross-functional. You author the data content; legal authors the copyright section; governance owns the identification and sign-off.
- The Article 53 summary is one of five artifacts that render from the same evidence base at a GPAI release — the HF card, the Google card, the Meta card, the public summary, and the Annex XI package.

Chapter 08 moves to the other direction — the **data slice of a model card** and the **dataset slice of a system card** — the two documents where the model team and system team cross-reference your evidence as a *contributor*, not as the owner.
