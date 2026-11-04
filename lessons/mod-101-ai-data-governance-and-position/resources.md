# Resources for mod-101 — AI Training-Data Governance Frameworks and the AI Data Engineer Position

All chapters in this module are grounded in primary-source standards and regulation. Prefer the primary sources below over any summary. Citations here are to documents and their publishers; URLs have been included where the publisher maintains a canonical landing page. Verify version and effective-date metadata at read time — frameworks and regulations in this area evolve rapidly.

## NIST AI Risk Management Framework

- **NIST AI 100-1** — Artificial Intelligence Risk Management Framework (AI RMF 1.0), US National Institute of Standards and Technology, January 2023. Publisher page: <https://www.nist.gov/itl/ai-risk-management-framework>.
- **NIST AI 600-1** — Artificial Intelligence Risk Management Framework: Generative Artificial Intelligence Profile, July 2024.
- **NIST AI RMF Playbook** — web-hosted suggested-actions companion to AI 100-1, versioned separately. Hosted via the NIST Trustworthy and Responsible AI Resource Center: <https://airc.nist.gov/>.
- **NIST AI 100-2 E2023** — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations. Useful companion for data-poisoning reasoning in mod-105.

## ISO / IEC standards

Standards are paywalled. The ISO Online Browsing Platform (OBP) exposes titles, scope, forewords, and tables of contents.

- **ISO/IEC 42001:2023** — Artificial intelligence — Management system. OBP: <https://www.iso.org/standard/81230.html>.
- **ISO/IEC 23894:2023** — Artificial intelligence — Guidance on risk management. OBP: <https://www.iso.org/standard/77304.html>.
- **ISO/IEC 25012:2008** — Software engineering — Software product Quality Requirements and Evaluation (SQuaRE) — Data quality model. OBP: <https://www.iso.org/standard/35736.html>.
- **ISO/IEC 5259-1:2024** — Data quality for analytics and machine learning — Part 1: Overview, terminology, and examples.
- **ISO/IEC 5259-2:2024** — Part 2: Data quality measures.
- **ISO/IEC 5259-3:2024** — Part 3: Data quality management requirements and guidelines.
- **ISO/IEC 5259-4:2024** — Part 4: Data quality process framework.
- **ISO/IEC 5259-5:2024** — Part 5: Data quality governance.

<!-- needs-research: confirm final publication year on each 5259 part at read time; some parts were still DIS/FDIS when scoped for this module. -->

## EU regulation

- **Regulation (EU) 2024/1689** — the EU Artificial Intelligence Act. EUR-Lex consolidated text: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>.
- **Directive (EU) 2019/790** — Copyright in the Digital Single Market (the DSM Copyright Directive); Article 4 TDM exception. EUR-Lex: <https://eur-lex.europa.eu/eli/dir/2019/790/oj>.
- **Regulation (EU) 2016/679** — General Data Protection Regulation (GDPR). EUR-Lex: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>.
- **EU AI Office** — practitioner hub for the AI Act including the GPAI Code of Practice and the training-data summary template. European Commission page: <https://digital-strategy.ec.europa.eu/en/policies/ai-office>.
- **European Data Protection Board (EDPB) opinions** — binding and influential guidance on GDPR interpretation. Browse by topic at <https://www.edpb.europa.eu/>. Of particular relevance: EDPB Opinion 28/2024 on certain data-protection aspects related to the processing of personal data in the context of AI models. <!-- needs-research: cite the superseding or related EDPB guidance in force at read time. -->

## US privacy and consumer protection

- **California Consumer Privacy Act (CCPA)** as amended by the **California Privacy Rights Act (CPRA)** — Cal. Civ. Code §1798.100 et seq. California Privacy Protection Agency (CPPA) site: <https://cppa.ca.gov/>. Current and pending CPPA regulations on automated decisionmaking technology are tracked at the same site.
- **Global Privacy Control (GPC)** — browser-level opt-out signal specification: <https://globalprivacycontrol.org/>.

## US sector regimes

- **SR 11-7** — "Guidance on Model Risk Management", Federal Reserve Board, April 4, 2011: <https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm>.
- **OCC 2011-12** — "Supervisory Guidance on Model Risk Management", Office of the Comptroller of the Currency, April 4, 2011: <https://www.occ.gov/news-issuances/bulletins/2011/bulletin-2011-12.html>.
- **FDIC FIL-22-2017** — Supervisory guidance on model risk management (adoption of SR 11-7 shape for FDIC-supervised institutions). <!-- needs-research: verify FDIC current adoption reference. -->
- **FDA Good Machine Learning Practice Guiding Principles** (October 2021, joint FDA / Health Canada / UK MHRA): <https://www.fda.gov/medical-devices/software-medical-device-samd/good-machine-learning-practice-medical-device-development-guiding-principles>.
- **FDA Predetermined Change Control Plans for AI-Enabled Device Software Functions — Final Guidance**, US Food and Drug Administration, December 2024. <!-- needs-research: pin the exact FDA guidance document title and docket at read time. -->

## US state AI law

- **Colorado SB 24-205 (Colorado AI Act)** — signed May 17, 2024, effective February 1, 2026. Text and status: <https://leg.colorado.gov/bills/sb24-205>. Colorado Attorney General implementing rulemaking is the controlling practical guidance. <!-- needs-research: cite the Colorado AG rules in force at read time. -->
- **NYC Local Law 144** on Automated Employment Decision Tools — effective July 5, 2023. NYC Department of Consumer and Worker Protection (DCWP) implementing rules: <https://www.nyc.gov/site/dca/about/automated-employment-decision-tools.page>.

<!-- needs-research: additional US state regimes (Utah AI Policy Act, Illinois HB 3773, Texas DPSA, Virginia CDPA, Connecticut CTDPA, Oregon OCPA, Montana MCDPA, etc.) land primarily on deployers; verify active list at read time. -->

## Opt-out signalling and provenance standards

- **RFC 9309** — Robots Exclusion Protocol. <https://www.rfc-editor.org/rfc/rfc9309>.
- **ai.txt** — community-specified AI-crawler opt-out convention. Primary site: <https://site.spawning.ai/spawning-ai-txt> (publisher-maintained).
- **llms.txt** — community-specified LLM-facing content convention. <https://llmstxt.org/>.
- **IETF AI Preferences (ai-pref) working group** — standards-track evolution of AI opt-out signalling. <https://datatracker.ietf.org/wg/aipref/about/>. <!-- needs-research: verify published RFCs, drafts, and status at read time. -->
- **C2PA** — Coalition for Content Provenance and Authenticity, Content Credentials specification: <https://c2pa.org/specifications/>.

## Documentation templates and SBOM standards

- **Datasheets for Datasets** — Gebru et al., 2018. arXiv:1803.09010. <https://arxiv.org/abs/1803.09010>.
- **Data Statements for NLP** — Bender and Friedman, 2018. TACL. <https://aclanthology.org/Q18-1041/>.
- **The Dataset Nutrition Label** — Holland et al., 2018. arXiv:1805.03677. <https://arxiv.org/abs/1805.03677>.
- **Model Cards for Model Reporting** — Mitchell et al., 2019. FAT* '19. <https://arxiv.org/abs/1810.03993>.
- **Hugging Face Dataset Card** — card format and YAML metadata schema. <https://huggingface.co/docs/hub/datasets-cards>.
- **Croissant** — MLCommons metadata format for machine-learning datasets. <https://mlcommons.org/working-groups/data/croissant/>.
- **CycloneDX** — SBOM / ML-BOM specification maintained by OWASP: <https://cyclonedx.org/>.
- **SPDX** — Software Package Data Exchange ISO/IEC 5962:2021 standard with an AI profile in development: <https://spdx.dev/>.
- **SLSA** — Supply-chain Levels for Software Artifacts: <https://slsa.dev/>.
- **Sigstore** — signing, verification, and provenance for software artifacts: <https://www.sigstore.dev/>.

## Companion academic references cited in this module

- Dodge, Sap, Marasović, Agnew, Ilharco, Groeneveld, Mitchell, Gardner. *Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus.* EMNLP 2021. <https://aclanthology.org/2021.emnlp-main.98/>.
- Lee, Ippolito, Nystrom, Zhang, Eck, Callison-Burch, Carlini. *Deduplicating Training Data Makes Language Models Better.* ACL 2022. <https://aclanthology.org/2022.acl-long.577/>.
- Kandpal, Wallace, Raffel. *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022. <https://proceedings.mlr.press/v162/kandpal22a.html>.
- Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, Roberts, Brown, Song, Erlingsson, Oprea, Raffel. *Extracting Training Data from Large Language Models.* USENIX Security 2021. <https://www.usenix.org/conference/usenixsecurity21/presentation/carlini-extracting>.
- Shi, Xu, Wang, Choi, Zettlemoyer. *Detecting Pretraining Data from Large Language Models.* ICLR 2024. <https://arxiv.org/abs/2310.16789>.
- Oren, Meister, Chatterji, Ladhak, Hashimoto. *Proving Test Set Contamination in Black Box Language Models.* ICLR 2024. <https://arxiv.org/abs/2310.17623>.

## Practitioner reading

Opinion and practitioner pieces are useful but not authoritative. Prefer them for *why* an obligation exists; prefer primary sources for *what* it says.

- UK Information Commissioner's Office (ICO) guidance on AI and data protection: <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/>.
- CNIL (France) guidance on AI and personal data: <https://www.cnil.fr/en/artificial-intelligence>.
- EDPB opinions on AI topics (see "US privacy and consumer protection" above for US state links; EU counterparts live at <https://www.edpb.europa.eu/>).

---

When citing any resource in a dataset card, governance pack, or regulator-facing document, use the document title, publisher, and version date — not a summary URL. If a resource is time-sensitive (effective dates, draft-vs-final guidance, signatory lists), add a `<!-- needs-research: ... -->` note so a future pass can verify the current state.
