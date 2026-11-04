# Chapter 08 — Lawful Basis for Training Data: GDPR, CCPA / CPRA, HIPAA

## Why this chapter exists — and what it is not

This chapter is **regulatory literacy for data engineers**. It is the vocabulary you need to:

- Package the engineering evidence that counsel needs to form a lawful-basis opinion.
- Route the right questions to the right people (counsel, DPO, risk).
- Explain the pipeline's role to a non-technical stakeholder without over-reaching.

**It is not legal advice**, and nothing you do in this module constitutes practising law. Reading this chapter does not make you a lawyer. The division of labour is strict and deliberate:

- **Counsel / DPO decides** the lawful basis for processing, the DPIA outcome, the de-identification method, and the release.
- **You** produce the engineering evidence they need to decide: detection policy, redaction policy, quarantine manifest, residual-PII accounting, data-flow diagrams, audit SHAs.

Chapter 09 pins the evidence-package schema. Exercise 03 is the hands-on authoring. This chapter is the context.

## GDPR — the EU baseline

**Primary source:** Regulation (EU) 2016/679, consolidated at <https://eur-lex.europa.eu/eli/reg/2016/679/oj>.

### Article 6 — lawfulness of processing

Article 6(1) enumerates six lawful bases. For training-data processing the realistic ones are:

- **Art 6(1)(a) — consent.** The data subject has given consent to the specific processing. Rarely practical for web-scraped corpora; useful for user-provided fine-tuning data (e.g., a product's own users opting into training).
- **Art 6(1)(b) — contract.** Processing necessary for a contract with the data subject. Usually not applicable to training.
- **Art 6(1)(c) — legal obligation.** Processing required by law. Not usually applicable.
- **Art 6(1)(f) — legitimate interests.** The controller has a legitimate interest that is not overridden by the data subject's rights. This is the **most common basis** claimed for web-scraped training data; it requires a **balancing test** (the "LIA" — legitimate interests assessment) that counsel documents.

The engineering evidence counsel needs for an Art 6(1)(f) analysis:

- Which categories of data are being processed? (PII entity types × source domains.)
- How much data minimisation is being done? (Scrub rates per entity type.)
- What safeguards reduce risk to the data subject? (Dedup, quarantine, DP-SGD handoff.)
- Can the processing be substituted with less-invasive processing? (Synthetic data, consented data.)
- What are the data subject's reasonable expectations? (Public-web data vs. private-forum data.)

Package the answers as structured fields in the privacy-risk report (Chapter 09 §5).

### Article 9 — special categories of personal data

Article 9(1) prohibits processing of **special-category data** by default:

- Racial or ethnic origin
- Political opinions
- Religious or philosophical beliefs
- Trade-union membership
- Genetic data
- Biometric data (for identification)
- Health data
- Data concerning sex life or sexual orientation

Article 9(2) lists specific exceptions (explicit consent, substantial public interest, scientific-research, data manifestly made public by the subject, etc.).

The engineering evidence for Art 9:

- Have you identified special-category entities in your detector? (Health, religion, political, biometric, genetic.)
- What is the detector's recall on those entities? (Chapter 04's eval numbers.)
- Are there whole documents / sources that are categorically special-category? (Medical forums, patient records, political commentary — treat these as quarantine candidates or rely on counsel-negotiated bases.)

The **CJEU C-184/20 (OT v Vyriausioji tarnybinės etikos komisija)** judgment widens the scope of what counts as Art 9 data — information that *reveals* a special category (e.g., a public campaign-finance disclosure that reveals political opinion) is itself special-category. Counsel will apply this; your evidence must be rich enough to support it.

### Article 14 — transparency when data is not collected from the subject

Article 14 requires the controller to inform data subjects that their data is being processed, including:

- The identity of the controller.
- The purposes of processing.
- The categories of personal data.
- The source of the personal data.
- The data subject's rights (access, erasure, restriction).

For web-scraped training data this is **structurally hard**: the controller does not know who the data subjects are. **Art 14(5)(b)** provides an exception where providing the information "proves impossible or would involve a disproportionate effort". Recital 62 explains this with reference to archival / research / statistical purposes.

Even where the exception applies, the controller must **take appropriate measures to protect the data subject's rights**, which typically means a public privacy notice, a public dataset card, and an accessible rights-of-the-subject process.

The engineering evidence for Art 14:

- The dataset card content ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) — public documentation of what was processed.
- The data-subject-request (DSR) handler — a documented process for a subject who contacts the controller. We provide the technical side (how to find a document by its source URL and either redact or exclude from future training); counsel provides the policy side.
- The source-provenance record ([`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)) — which URLs, which licences.

### Article 17 — the right to erasure ("right to be forgotten")

Article 17 gives the data subject the right to have their personal data erased under specific conditions. For training data this is operationally hard: once a model is trained, "erasing" a specific record is a research-grade problem (machine unlearning). The pipeline-side answer is:

- **Erase from the corpus store**: future training runs will not include the record.
- **Flag for retraining consideration**: the aggregate backlog of erasure requests may justify a retraining cycle.
- **Forward to model-team**: machine-unlearning work is model-side.

Record the pipeline-side side of the process in the privacy-risk report. This is a field counsel will ask about.

### Article 25 and 35 — by-design and DPIA

- **Article 25** mandates data protection by design and by default. The pipeline is the by-design implementation. The privacy-risk report is the evidence.
- **Article 35** mandates a Data Protection Impact Assessment (DPIA) for high-risk processing. Training large language models on web-scraped personal data is a canonical high-risk processing case. The DPIA is counsel-authored; your contribution is the technical-evidence section.

### The EDPB opinion on AI model training

The European Data Protection Board published **Opinion 28/2024 on certain data protection aspects related to the processing of personal data in the context of AI models** (<https://www.edpb.europa.eu/system/files/2024-12/edpb_opinion_202428_ai-models_en.pdf>). <!-- needs-research: an EDPB opinion on AI-model training was published December 2024; re-check for subsequent opinions / guidelines. --> This is the current authoritative read on how legitimate-interests balances for LLM training. Counsel cites it; you should read it so you can have the conversation.

## CCPA / CPRA — the California baseline

**Primary source:** Cal. Civ. Code §§ 1798.100 – 1798.199.100 (CCPA as amended by CPRA, 2020). Regulations at <https://cppa.ca.gov/regulations/>.

### "Personal information" and "sensitive personal information"

CCPA's definition of "personal information" is broad — anything that identifies, relates to, describes, or is reasonably capable of being associated with a consumer. Training data drawn from websites that include California-resident posts is in scope.

"Sensitive personal information" (SPI) was added by CPRA and includes government IDs, financial account info, precise geolocation, racial / ethnic origin, religious or philosophical beliefs, genetic / biometric data, health data, sexual-orientation data, content of certain communications. The SPI → Art 9 analogy is imperfect but useful.

### "Sale or sharing" obligations

CCPA / CPRA require the business to disclose and provide an opt-out from the "sale or sharing" of personal information. **"Sharing"** was added by CPRA specifically to cover cross-context behavioural advertising; **"sale"** is defined broadly as any disclosure for monetary or other valuable consideration.

The question for training-data use is: **does using California-resident data to train a commercial model count as a "sale" or "sharing"?** The current regulatory conversation is active and not fully settled; counsel will give you the answer.

The engineering evidence the CCPA / CPRA analysis needs:

- Can you identify California-resident data in the corpus? (Often: no, because source IP / geolocation is lost at scrape time.)
- Do you honour the Global Privacy Control (GPC) signal at collection? (This is a crawler-side concern, [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md); the privacy-risk report cites it.)
- Have you flagged SPI entities in your detection? (The SPI → entity-type mapping is a field in the detection policy.)

### The California Delete Act (SB 362)

The Delete Act (<https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=202320240SB362>) adds a one-stop deletion mechanism, centralising the opt-out / deletion process. The technical implications at the training-data level are indirect but worth knowing: if a data subject exercises the one-stop delete, the business receives a delete request it must honour. Your pipeline's data-subject-request handler is the technical side of that.

## HIPAA — the US health baseline

**Primary source:** HIPAA Privacy Rule, 45 CFR Part 160 and Subparts A, C, E of Part 164.

### De-identification — § 164.514(b)

HIPAA treats **de-identified** health information as out of scope of the Privacy Rule. The regulation provides two methods for de-identification:

#### Safe Harbor method — § 164.514(b)(2)

Remove 18 specific identifiers:

1. Names.
2. Geographic subdivisions smaller than a state (street address, city, county, ZIP code).
3. Dates (other than year) directly related to an individual.
4. Telephone numbers.
5. Fax numbers.
6. Email addresses.
7. Social security numbers.
8. Medical record numbers.
9. Health plan beneficiary numbers.
10. Account numbers.
11. Certificate / licence numbers.
12. Vehicle identifiers and serial numbers, including licence plate numbers.
13. Device identifiers and serial numbers.
14. Web Universal Resource Locators (URLs).
15. Internet Protocol (IP) addresses.
16. Biometric identifiers, including finger and voice prints.
17. Full-face photographic images and any comparable images.
18. Any other unique identifying number, characteristic, or code.

**Plus** the controller must not have actual knowledge that the remaining information could be used alone or in combination to identify an individual.

This is a **mechanical list**. Your detector's entity-type coverage maps to it. The engineering evidence is a per-identifier detection-coverage table.

#### Expert Determination method — § 164.514(b)(1)

A statistical / scientific expert determines, by generally-accepted methods, that the risk of re-identification is "very small". This is a qualitative method that requires an authoritative expert and documentation of the methodology.

The engineering evidence for Expert Determination:

- The dataset's detailed shape (entity counts, record counts, quasi-identifier distribution).
- The de-identification techniques applied (Chapter 05 primitives).
- The anticipated recipient's capabilities.
- The re-identification-risk model's output (k-anonymity, l-diversity, δ-presence — Google DLP's Risk Analysis APIs are useful here, Chapter 03).

**HHS OCR Guidance Regarding Methods for De-identification** (<https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html>) is the authoritative operational reference.

### Pipeline implication

For a corpus with clinical text, the Safe Harbor method is the mechanical baseline; Expert Determination is the alternative for cases where Safe Harbor is too destructive (e.g., some retained dates are needed for longitudinal signal). In both cases, the de-identification is a pipeline step; the audit evidence is the privacy-risk report augmented with a Safe Harbor coverage table or an Expert Determination packet.

## The engineering-evidence package — what counsel needs from you

Chapter 09 pins the schema; the package shape is:

- **§1 Slice identity and provenance** — slice ID, source manifest (from [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)), jurisdictional tags, pipeline run ID.
- **§2 Detection policy** — recogniser list with versions, thresholds, context-trigger words, NLP engine, config SHA.
- **§3 Detection quality on the eval set** — per-entity-type precision / recall / F1, per-source recall, CI.
- **§4 Redaction policy** — per-entity-type primitive choice with parameters.
- **§5 Residual PII accounting** — post-redaction finding counts, per entity type and per source.
- **§6 Memorisation-risk delta** — cites the `mod-105` dedup report by SHA.
- **§7 Quarantine manifest** — documents routed to quarantine, with reason codes.
- **§8 Data-flow and retention** — where the raw store is, who has access, retention policy.
- **§9 DSR process** — how a data-subject request is handled technically.
- **§10 Lawful-basis fields** — the structured fields counsel fills in (or references a separate LIA document from).
- **§11 Signatures and SHAs** — Sigstore signature, artifact SHAs.

Package means **machine-readable**. Counsel reads the YAML and the human-written LIA; the engineering fields are not prose. This is a correctness guarantee against drift.

## Routing — the common conversations

### "Can we use this corpus?"

Not your question. Yours is: "here is the detection policy, redaction policy, residual PII, quarantine manifest, source provenance, and data-flow diagram. Counsel: can we use this?"

### "Is this GDPR-compliant?"

Not a sentence you say. Yours is: "here is the engineering-evidence package for an Art 6(1)(f) analysis, including the data-minimisation and safeguards shape. Counsel will decide if the LIA tips the balancing test."

### "We can just skip the HIPAA check — it's public data."

A flag. Public availability does not override HIPAA for PHI. Document your detector coverage against the 18 identifiers and let counsel decide.

### "Can we release the quarantine bucket for analytics?"

Not your decision. The quarantine manifest exists exactly because counsel has to decide per-document.

### "Our source's ToS lets us train on this."

A relevant fact but not a full lawful basis under GDPR — ToS is not a magic wand that overrides Art 6. Route to counsel.

## Non-EU, non-US regulations — the quick map

- **Brazil LGPD** — closely modelled on GDPR. Art 7 lawful bases, Art 11 sensitive data. <https://www.gov.br/anpd/>.
- **UK GDPR + DPA 2018** — post-Brexit; ICO guidance at <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/>.
- **Canada PIPEDA / provincial laws (Quebec Law 25, etc.)** — consent-centric framework.
- **Singapore PDPA** — regulator: PDPC. <https://www.pdpc.gov.sg/>.
- **South Korea PIPA** — strict consent regime.
- **Japan APPI** — amended for international alignment with GDPR.
- **China PIPL** — cross-border transfer restrictions are the dominant concern.
- **India DPDPA 2023** — new; implementation rules evolving.

The pattern: every jurisdiction has an analogue to Art 6 lawful basis and an analogue to sensitive data. The engineering evidence package is reusable; the LIA / counsel analysis is jurisdiction-specific.

## Summary

- **You do not form legal opinions.** You package the engineering evidence counsel needs: detection policy, redaction policy, residual PII, quarantine, provenance, data flow.
- **GDPR**: Article 6 lawful basis (usually 6(1)(f) legitimate interests), Article 9 special categories, Article 14 transparency (and the 14(5)(b) exception), Article 17 erasure, Article 25 by-design, Article 35 DPIA. **EDPB Opinion 28/2024** is the current authoritative read on AI-model training.
- **CCPA / CPRA**: broad "personal information" scope, SPI subset, "sale or sharing" obligations, GPC signal handling, Delete Act implications.
- **HIPAA**: § 164.514(b) de-identification via **Safe Harbor** (18 identifiers) or **Expert Determination** (statistical risk). For clinical corpora the pipeline implements one of these methods.
- **The engineering-evidence package** has eleven sections; Chapter 09 pins the schema; Exercise 03 is the hands-on authoring.
- **Other jurisdictions** reuse the pattern: lawful-basis analogue + sensitive-data analogue. The engineering evidence is portable; the legal analysis is per-jurisdiction.

Chapter 09 assembles all of this into the signed `privacy_risk_report.yaml` — the single deliverable that the rest of the organisation consumes.
