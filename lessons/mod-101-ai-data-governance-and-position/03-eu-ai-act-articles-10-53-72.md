# Chapter 03 — The EU AI Act for Training Data: Articles 10, 53+Annex XI, and 72

## Why this chapter exists

The EU AI Act (Regulation (EU) 2024/1689) is the first comprehensive binding law covering AI systems in a major jurisdiction. It entered into force on 1 August 2024, with staggered application dates — the general-purpose AI (GPAI) obligations apply from 2 August 2025, and high-risk-system obligations apply from 2 August 2026. The Act is **horizontal**: it applies to AI systems and GPAI models regardless of sector, and it has extraterritorial reach (providers outside the EU are in scope if their output is used in the EU).

For an AI Data Engineer three Articles matter more than any others:

- **Article 10** — Data and data governance for **high-risk AI systems**. The core obligations on training, validation, and test datasets.
- **Article 53 + Annex XI + Annex XII** — Obligations for **general-purpose AI (GPAI) models**, including the public training-data summary and the technical-documentation package.
- **Article 72** — Post-market monitoring, including dataset-side signals that feed back into pipeline work.

Alongside the binding Act, the **EU AI Office General-Purpose AI Code of Practice** is a voluntary code that anticipates regulator expectations for GPAI providers. The final Code was published on 10 July 2025 and contains three chapters (Transparency, Copyright, Safety and Security); the signatory list and current status are maintained at <https://digital-strategy.ec.europa.eu/en/policies/contents-code-gpai>. Signing is not required, but the Code is the current best available guide for *what the AI Office will accept* in training-data documentation. Verify the signatory list at read time.

This chapter is not legal advice. It teaches you to read these Articles as an engineer so that, when legal counsel and the governance analyst hand you a scope, you know which pipeline artifact answers which paragraph.

## Article 10 — the data-governance Article

Article 10 applies to **high-risk AI systems** as defined in Article 6 and Annex III (biometrics, critical infrastructure, education, employment, essential private and public services, law enforcement, migration/border, administration of justice). A general-purpose AI model is *not itself a high-risk system*, but it can be a *component* of one, and the Article 10 obligations pass through to the system provider.

The Article's paragraphs lay out specific requirements on training, validation, and test data. For an engineer, each paragraph becomes a pipeline obligation:

| Article 10 obligation (paraphrased) | Pipeline obligation it creates |
|---|---|
| Datasets subject to data-governance and management practices appropriate to the intended purpose of the system (Art 10(2)) | A **data-objective brief** signed by the model/product team before sourcing (same artifact as in Chapter 01) |
| Data-collection processes documented (Art 10(2)(b)) | A **source manifest** per slice with collection method, licence, consent basis |
| Relevant data-preparation processing operations documented (Art 10(2)(c)) | A **preparation log** — every operator (filter, dedup, PII scrub, synth, decontam) with parameters and resulting SHA |
| Formulation of relevant assumptions, in particular with respect to the information that the data are supposed to measure and represent (Art 10(2)(d)) | An **assumptions document** on the dataset card — what the slice is a proxy for, and what it is not |
| Assessment of availability, quantity, and suitability of datasets (Art 10(2)(e)) | A **sufficiency statement** in the dataset card — is the slice big enough and representative enough for the intended purpose |
| Examination in view of possible biases that are likely to affect health, safety, fundamental rights, or to lead to discrimination (Art 10(2)(f)) | A **bias audit** — demographic / linguistic / geographic representativeness, filter effects (Dodge-style), fairness evaluation |
| Appropriate measures to detect, prevent, and mitigate possible biases (Art 10(2)(g)) | Mitigation records in the preparation log — reweighting, oversampling, filter adjustments, downstream evals |
| Identification of relevant data gaps or shortcomings and how they can be addressed (Art 10(2)(h)) | A **known-gaps section** on the dataset card with intended follow-up |
| Datasets shall be relevant, sufficiently representative, and to the best extent possible, free of errors and complete in view of the intended purpose (Art 10(3)) | The combination of representativeness audit, quality-measure log, and sufficiency statement — plus an acknowledgement that absolute error-freeness is not attainable at scale |
| Datasets shall take into account the characteristics or elements that are particular to the specific geographical, contextual, behavioural, or functional setting (Art 10(4)) | A **deployment-context section** on the dataset card linking slice composition to deployment setting |
| Processing of special categories of personal data to detect and correct bias is permitted subject to strict conditions (Art 10(5)) | Coordination with legal — you do not initiate Art 10(5) processing on your own; see Chapter 04 on GDPR |

The litmus test for a slice intended for a high-risk use case: if you cannot produce every artifact in the right-hand column within one CLI command, the slice is not Article-10-ready.

## Article 53 — GPAI provider obligations

A **general-purpose AI model** is defined in Article 3 and expanded in Article 51. If you build pretraining corpora for a model that could be sold or offered as a GPAI model, Article 53 applies to your organisation. Article 53(1) imposes four operational obligations on providers:

- **(a)** Draw up and keep up-to-date **technical documentation** for the model including training and testing process — the detail is in **Annex XI**.
- **(b)** Draw up, keep up-to-date, and make available **information and documentation to downstream providers** who intend to integrate the model — the detail is in **Annex XII**.
- **(c)** Put in place a **policy to comply with Union law on copyright and related rights**, including identification and respect for the Article 4(3) TDM reservation under the DSM Copyright Directive — i.e. honour EU opt-outs. See Chapter 04.
- **(d)** Draw up and make publicly available a **sufficiently detailed summary about the content used for training**, according to a template provided by the **AI Office**.

Each obligation is a shipping artifact:

### Annex XI technical documentation (the private package)

Annex XI, Section 1 lists the minimum contents of the GPAI technical-documentation package. The items most relevant to data engineering include:

- Description of the model, including intended use and acceptable use policy.
- Training process description — methodology and techniques, key design choices, the main assumptions.
- Information on the data used — type and provenance of data, curation methodology, how obtained and selected, measures to detect unsuitable data sources, measures to detect identifiable biases.
- Computational resources used.
- Energy consumption.

For GPAI models with **systemic risk** (as classified under Article 51 — currently models trained with cumulative compute greater than 10^25 FLOPs is the primary indicative threshold), Annex XI Section 2 adds requirements including evaluation strategies, internal and external testing, and adversarial testing / red-teaming results.

As the data engineer on a GPAI provider you own the "Information on the data used" slice: the source manifest, the licence matrix, the preparation log, the dedup report, the contamination firewall report, the PII report, the bias audit. Package them into the Annex XI section with cross-references to the dataset cards.

### Annex XII information for downstream providers (the integrator-facing package)

Annex XII lists what downstream providers get. The data-side content overlaps heavily with Annex XI but is scoped for integration — what a product team needs to know about the training data to build on top. In practice this is a subset of your Annex XI package plus the model and dataset cards.

### The public training-data summary (Article 53(1)(d))

The public summary is the headline artifact and the one most visible outside the organisation. The AI Office published the mandatory template and its Explanatory Notice on 24 July 2025 (<https://digital-strategy.ec.europa.eu/en/library/explanatory-notice-and-template-public-summary-training-content-general-purpose-ai-models>); the obligation applies to new GPAI models placed on the market from 2 August 2025 and to pre-existing models by 2 August 2027. The summary must be "sufficiently detailed" — enough to let rights-holders assess whether their content was used and whether their opt-out was honoured — while not requiring the provider to publish a full corpus listing. Confirm the current template revision at read time.

Engineering output: a `training_data_summary.md` (or JSON/CSV in the Office's required format) per model release, built from the source manifests of every slice included in the training mix. This is the mod-112 deliverable; mod-101 teaches you that it exists and who needs it.

## Article 72 — post-market monitoring with dataset-side signals

Article 72 requires providers of high-risk AI systems (and, via Article 55 for GPAI with systemic risk, GPAI providers at that tier) to establish a post-market monitoring system that collects, documents, and analyses data on the performance of the AI system throughout its lifetime.

The data-engineering slice of Article 72:

- **Dataset-side drift signals** — monitoring on the training corpora and on the inference-time inputs that feed back into re-training decisions.
- **Incident signals** — when a downstream serious incident is reported (Article 73), you participate in root-cause analysis if the cause may be in the training data (poisoning, licensing issue, representativeness gap, stale calibration set).
- **Change control** — when you re-curate a slice in response to a post-market signal, the change is logged against the Article 72 plan.

This is primarily the subject of `mod-111-data-quality-monitoring-and-regression`. In this chapter you learn that monitoring is not optional add-on work — it is regulatorily required for high-risk systems.

## The GPAI Code of Practice

The GPAI Code of Practice is drafted by the EU AI Office in consultation with providers, civil society, academia, and member states, as mandated by Article 56 of the Act. Signatories commit to specific measures that *presumptively demonstrate compliance* with the GPAI-provider obligations in Articles 53 and 55. Non-signatories remain bound by the Act itself; they just need to show their own equivalent evidence.

For your purposes:

- The Code is the **best early view of what "sufficiently detailed" means** for the Article 53(1)(d) public summary and what "adequate" means for the Annex XI technical documentation.
- Measures signatories commit to generally include: honouring ai.txt / robots.txt-based opt-outs and DSM Article 4 reservations, publishing the training-data summary in the AI Office template, maintaining incident reporting and systemic-risk assessments.
- You should structure your pipeline so that a decision to sign the Code (or not) later is a configuration change, not a re-plumbing.

The final Code was published on 10 July 2025 at <https://digital-strategy.ec.europa.eu/en/policies/contents-code-gpai>, with the first signatory taskforce meeting held on 30 January 2026. Complementary Commission Guidelines on the scope of GPAI-provider obligations (C(2025) 5045 final, 18 July 2025) sit alongside the Code at <https://digital-strategy.ec.europa.eu/en/policies/guidelines-gpai-providers>. Confirm the signatory roster and any later revisions at read time.

## A reading plan for the Act

1. Read Article 10 in full once. It is short and operational.
2. Read Articles 51–56 as a cluster for GPAI scope; Article 53 is the heart.
3. Read Annexes XI and XII with a highlighter — mark every data-side item.
4. Read Articles 72 and 73 for the post-market monitoring + incident shape.
5. Skim the GPAI Code of Practice when it changes version.

You do not need to memorise definitions or placement in the Act. You need to recognise the clause when legal quotes it at you and know which artifact in your pipeline answers it.

## Summary

- EU AI Act Article 10 is the per-high-risk-system data obligation. Every paragraph maps to a pipeline artifact; produce the artifacts automatically.
- Article 53 + Annex XI + Annex XII + the AI Office template for the public training-data summary are the GPAI-provider deliverables. The data engineer owns the data slice of each.
- Article 72 makes dataset-side post-market monitoring regulatorily required. See mod-111.
- The GPAI Code of Practice is voluntary but is the current best view of what the AI Office will accept. Structure the pipeline so signing is a configuration change.

Exercise 03 walks you through producing the Article 10 artifact set for a sample high-risk use case and drafting the skeleton of an Annex XI package. Chapter 04 adds the privacy and copyright substrate — GDPR, CCPA/CPRA, and the EU DSM Article 4 opt-out.
