# Resources for mod-110 — PII Detection, Redaction, and Privacy Risk for Training Data

Primary sources first. Prefer the canonical publisher's copy over any summary. Versions, DOI stability, regulatory text, and dataset hosting change — re-verify at read time; `<!-- needs-research: ... -->` notes mark items that are particularly time-sensitive.

## Memorisation and extraction attacks — the Chapter 01 / 07 citations

- **Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, Roberts, Brown, Song, Erlingsson, Oprea, Raffel.** *Extracting Training Data from Large Language Models.* USENIX Security 2021 (arXiv 2012.07805). <https://www.usenix.org/conference/usenixsecurity21/presentation/carlini-extracting>. The founding paper for pipeline-side PII-leakage reasoning.
- **Carlini, Ippolito, Jagielski, Lee, Tramèr, Zhang.** *Quantifying Memorization Across Neural Language Models.* ICLR 2023 (arXiv 2202.07646). <https://arxiv.org/abs/2202.07646>. The memorisation-scales-with-duplicates / model-size / prompt-length result.
- **Kandpal, Wallace, Raffel.** *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022 (arXiv 2202.06539). <https://arxiv.org/abs/2202.06539>. The dedup-is-a-privacy-control result consumed in Chapter 06.
- **Nasr, Carlini, Hayase, Jagielski, Cooper, Ippolito, Choquette-Choo, Wallace, Tramèr, Lee.** *Scalable Extraction of Training Data from (Production) Language Models.* arXiv 2311.17035, 2023. <https://arxiv.org/abs/2311.17035>. The follow-up at production-model scale.
- **Shokri, Stronati, Song, Shmatikov.** *Membership Inference Attacks Against Machine Learning Models.* IEEE S&P 2017. <https://ieeexplore.ieee.org/document/7958568>. Foundational membership-inference paper.
- **Carlini, Chien, Nasr, Song, Terzis, Tramèr.** *Membership Inference Attacks From First Principles.* IEEE S&P 2022. <https://ieeexplore.ieee.org/document/9833649>.
- **Lukas, Salem, Sim, Tople, Wutschitz, Zanella-Béguelin.** *Analyzing Leakage of Personally Identifiable Information in Language Models.* IEEE S&P 2023 (arXiv 2302.00539). <https://arxiv.org/abs/2302.00539>.

## Detection toolchains

- **Microsoft Presidio** — <https://microsoft.github.io/presidio/>. Repository: <https://github.com/microsoft/presidio>. Documentation covers the Analyzer, Anonymizer, custom recognisers, context enhancement, and the Transformers-NER integration. The current list of supported entities is at <https://microsoft.github.io/presidio/supported_entities/>; re-check at authoring time.
- **scrubadub** — <https://scrubadub.readthedocs.io/>. Repository: <https://github.com/LeapBeyond/scrubadub>. Plain-Python regex-first PII scrubber; good for lightweight jobs, limited by its NER backbone.
- **Google Cloud Sensitive Data Protection (DLP)** — product page: <https://cloud.google.com/sensitive-data-protection>. API docs: <https://cloud.google.com/sensitive-data-protection/docs>. InfoTypes reference: <https://cloud.google.com/sensitive-data-protection/docs/infotypes-reference>. Transformations: <https://cloud.google.com/sensitive-data-protection/docs/transformations-reference>. <!-- needs-research: the product was rebranded from "Cloud DLP" to "Sensitive Data Protection" in 2023; check that URLs and API names are current. -->
- **AWS Comprehend PII** — detection API `DetectPiiEntities`: <https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html>. Redaction API `ContainsPiiEntities` and real-time endpoints: <https://docs.aws.amazon.com/comprehend/latest/APIReference/API_DetectPiiEntities.html>. Supported entity list: <https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html#how-pii-types>.
- **Amazon Macie** — <https://docs.aws.amazon.com/macie/latest/user/data-classification.html>. S3-oriented sensitive-data discovery; useful for static-corpus audits.
- **Azure AI Language — PII detection** — <https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview>. The managed sibling to Presidio.

## Redaction primitives and format-preserving encryption

- **NIST SP 800-38G** — *Recommendation for Block Cipher Modes of Operation: Methods for Format-Preserving Encryption.* <https://csrc.nist.gov/pubs/sp/800/38/g/final>. FF1 (originally FFX) is the canonical FPE construction. <!-- needs-research: NIST has been revisiting SP 800-38G after FF3 attack disclosures; check SP 800-38G Rev. 1 draft status at authoring time. -->
- **Bellare, Rogaway, Spies.** *The FFX Mode of Operation for Format-Preserving Encryption.* 2010. <https://csrc.nist.gov/CSRC/media/Projects/Block-Cipher-Techniques/documents/BCM/proposed-modes/ffx/ffx-spec.pdf>.
- **Durak, Vaudenay.** *Breaking the FF3 Format-Preserving Encryption Standard Over Small Domains.* CRYPTO 2017. <https://link.springer.com/chapter/10.1007/978-3-319-63715-0_24>. Why small-domain FPE is dangerous.
- **Bogdanov, Khovratovich, Rechberger.** *Biclique Cryptanalysis of the Full AES.* ASIACRYPT 2011. <https://eprint.iacr.org/2011/449> — backdrop for AES security claims used by FF1/FF3.
- **NIST SP 800-188** — *Trustworthy Attribute Transformation.* <https://csrc.nist.gov/pubs/sp/800/188/final>. De-identification techniques and risk.
- **ISO/IEC 20889:2018** — *Privacy enhancing data de-identification terminology and classification of techniques.* <https://www.iso.org/standard/69373.html>. Vocabulary used in Chapter 05.
- **NIST IR 8053** — *De-Identification of Personal Information.* <https://csrc.nist.gov/pubs/ir/8053/final>. Non-normative but useful reference.

## Evaluation datasets and benchmarks for PII detection

- **CoNLL-2003 NER dataset** — <https://www.clips.uantwerpen.be/conll2003/ner/>. Not PII-specific but the baseline PERSON / LOCATION / ORGANISATION benchmark most NER backbones report against.
- **OntoNotes 5.0** — <https://catalog.ldc.upenn.edu/LDC2013T19>. Broader entity coverage; 18 entity types.
- **Presidio evaluator & Presidio research** — <https://github.com/microsoft/presidio-research>. Fake-data generation, benchmarking scaffold, model comparison harness.
- **i2b2 de-identification challenges** — <https://www.i2b2.org/NLP/DataSets/>. 2014 and 2006 clinical-text de-identification challenge corpora. Access gated; useful for HIPAA de-ID eval sets.
- **n2c2 NLP Research Dataset** — <https://portal.dbmi.hms.harvard.edu/>. Successor shared-task hub to i2b2. <!-- needs-research: confirm current gatekeeping and dataset hosting at authoring time. -->
- **PubMedQA / MIMIC-III / MIMIC-IV** — <https://physionet.org/>. Credentialed-access clinical corpora; useful once PhysioNet DUA is signed.
- **Pile-CC + C4 PII audits** — community-authored. See the Common Pile <https://huggingface.co/datasets/common-pile/pile_v1_filtered> annotations and the C4 inspection work in: Dodge, Sap, Marasović, et al. *Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus.* EMNLP 2021. <https://aclanthology.org/2021.emnlp-main.98/>.

## GDPR — Chapter 08 primary sources

- **Regulation (EU) 2016/679 (GDPR)** — consolidated text: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. Key articles for this module: Art 4 (definitions), Art 5 (principles — purpose limitation, data minimisation), Art 6 (lawfulness of processing), Art 9 (special categories), Art 13–14 (transparency obligations), Art 17 (right to erasure), Art 25 (data protection by design and by default), Art 35 (DPIA), Art 89 (research exception).
- **European Data Protection Board (EDPB) Guidelines 06/2020 on Article 46(2)(a) and Article 46(3)(b) of Regulation 2016/679 (final)** — <https://edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en>. General landing for the EDPB guideline corpus.
- **EDPB Opinion 28/2024 on certain data protection aspects related to the processing of personal data in the context of AI models** — <https://www.edpb.europa.eu/system/files/2024-12/edpb_opinion_202428_ai-models_en.pdf>. The current authoritative read on legitimate-interests for LLM training. <!-- needs-research: an EDPB opinion on AI-model training was published December 2024; re-check for subsequent opinions / guidelines. -->
- **CJEU Case C-184/20 (OT v Vyriausioji tarnybinės etikos komisija)** — <https://curia.europa.eu/juris/liste.jsf?num=C-184/20>. Clarified the scope of "special category" Art 9 data.
- **ICO (UK) — Guidance on AI and data protection** — <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/guidance-on-ai-and-data-protection/>. Practical regulator interpretation.
- **Article 29 Working Party WP248 rev.01** — *Guidelines on Data Protection Impact Assessment (DPIA).* <https://ec.europa.eu/newsroom/article29/items/611236>.

## CCPA / CPRA — Chapter 08 primary sources

- **California Consumer Privacy Act of 2018 (as amended by CPRA, 2020)** — codified at Cal. Civ. Code §§ 1798.100 – 1798.199.100. <https://cppa.ca.gov/regulations/>.
- **California Privacy Protection Agency (CPPA) regulations** — <https://cppa.ca.gov/regulations/>. Includes the "automated decision-making technology" draft regulations relevant to training-data use.
- **"Sale or sharing" definition and opt-out signals (GPC)** — <https://cppa.ca.gov/faq.html>.
- **California Delete Act (SB 362, 2023)** — <https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=202320240SB362>.

## HIPAA — Chapter 08 primary sources

- **HIPAA Privacy Rule — 45 CFR Part 160, Subparts A, C, E of Part 164** — <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C>.
- **45 CFR § 164.514(b) — De-identification of protected health information** — Safe Harbor (18 identifiers) and Expert Determination methods. <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514>.
- **HHS OCR — Guidance Regarding Methods for De-identification** — <https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html>. Current operational guidance, including the Expert Determination process and example re-identification-risk thresholds.
- **HITRUST De-Identification Framework** — <https://hitrustalliance.net/>. Industry framework for Expert Determination packaging. <!-- needs-research: confirm current version and gating. -->

## ISO / IEC standards referenced

- **ISO/IEC 25012:2008** — *Data quality model.* <https://www.iso.org/standard/35736.html>. Chapter 01 uses the *confidentiality* characteristic.
- **ISO/IEC 27701:2019** — *Extension to ISO/IEC 27001 and ISO/IEC 27002 for privacy information management.* <https://www.iso.org/standard/71670.html>. The ISMS→PIMS mapping Chapter 06 uses.
- **ISO/IEC 29100:2024** — *Privacy framework.* <https://www.iso.org/standard/85938.html>. <!-- needs-research: confirm 2024 publication state and current Annex content. -->
- **ISO/IEC 20889:2018** — *Privacy enhancing data de-identification terminology and classification of techniques.* <https://www.iso.org/standard/69373.html>.
- **ISO/IEC 5259 series** — *AI — Data quality for analytics and machine learning.* <https://www.iso.org/standard/81859.html>. Part 2 (measures) and Part 5 (coverage, provenance, PII) are the current references. <!-- needs-research: publication state of Parts 3, 4, 5 at authoring time. -->
- **ISO/IEC 42001:2023** — *AI management system.* <https://www.iso.org/standard/81230.html>. Annex A.7.3 (sensitive data handling) is the audit hook.

## NIST references — privacy and AI

- **NIST Privacy Framework** — <https://www.nist.gov/privacy-framework>.
- **NIST AI Risk Management Framework (AI RMF 1.0)** — <https://www.nist.gov/itl/ai-risk-management-framework>. Privacy-relevant sub-categories: MAP 1.5, MEASURE 2.10, MANAGE 2.1.
- **NIST AI 100-2 E2023** — *Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.* <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf>. Chapter on privacy attacks is the relevant section.
- **NIST SP 800-188** — *Trustworthy Attribute Transformation.* <https://csrc.nist.gov/pubs/sp/800/188/final>.
- **NIST SP 800-122** — *Guide to Protecting the Confidentiality of Personally Identifiable Information (PII).* <https://csrc.nist.gov/pubs/sp/800/122/final>.

## Differential privacy — the handoff reading for `fine-tuning-engineer`

The DP-SGD training work is outside this module. These references are the context that lets you have the conversation across the handoff without over-reaching into the other role's craft:

- **Dwork, McSherry, Nissim, Smith.** *Calibrating Noise to Sensitivity in Private Data Analysis.* TCC 2006. <https://link.springer.com/chapter/10.1007/11681878_14>. The foundational DP paper.
- **Abadi, Chu, Goodfellow, McMahan, Mironov, Talwar, Zhang.** *Deep Learning with Differential Privacy.* CCS 2016 (arXiv 1607.00133). <https://arxiv.org/abs/1607.00133>. DP-SGD.
- **Yu, Naik, Backurs, Gopi, Inan, Kamath, Kulkarni, Lee, Manoel, Wutschitz, Zanella-Béguelin, Chen, Jerry.** *Differentially Private Fine-tuning of Language Models.* ICLR 2022. <https://arxiv.org/abs/2110.06500>.
- **Opacus** — <https://opacus.ai/>. PyTorch DP-SGD runtime. The reference library.
- **TensorFlow Privacy** — <https://github.com/tensorflow/privacy>.

## Dataset cards, lineage, and transparency (previews of mod-109 / mod-112)

- **Gebru, Morgenstern, Vecchione, Vaughan, Wallach, Daumé III, Crawford.** *Datasheets for Datasets.* Communications of the ACM 2021 (arXiv 1803.09010). <https://arxiv.org/abs/1803.09010>.
- **Bender, Friedman.** *Data Statements for Natural Language Processing.* TACL 2018. <https://aclanthology.org/Q18-1041/>.
- **Pushkarna, Zaldivar, Kjartansson.** *Data Cards: Purposeful and Transparent Dataset Documentation.* FAccT 2022. <https://arxiv.org/abs/2204.01075>.
- **MLCommons Croissant** — <https://mlcommons.org/working-groups/data/croissant/>.

---

When citing any resource in a privacy-risk report, dataset card, or governance package, include the document title, publisher, version / date, and (where applicable) the git SHA or Hugging Face revision hash of the toolkit or dataset you actually used. Regulatory text is read as of a date — always record the date. If the resource is time-sensitive (EDPB opinions, CPPA regulations, OWASP editions), add a `<!-- needs-research: ... -->` note so a future pass can verify the current state.
