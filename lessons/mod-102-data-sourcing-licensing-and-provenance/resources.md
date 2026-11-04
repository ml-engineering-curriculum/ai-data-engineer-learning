# Resources for mod-102 — Data Sourcing, Licensing, Consent, and Provenance Engineering

All chapters in this module are grounded in primary-source standards, regulation, and canonical academic references. Prefer the primary sources below over any summary. URLs are to the publishers' canonical landing pages; verify version and effective-date metadata at read time — the sourcing / licensing / provenance area is one of the most rapidly evolving in the track.

## Licences

### Creative Commons

- **Creative Commons 4.0 licence suite** — texts, FAQs, and legal-code pages: <https://creativecommons.org/licenses/>. Each licence's canonical URI resolves to the deed; append `/legalcode` for the binding text.
- **CC0 1.0 Universal (public-domain dedication)**: <https://creativecommons.org/publicdomain/zero/1.0/>.
- Earlier-version licences (1.0, 2.0, 2.5, 3.0) remain in force where chosen; the deed-level URL for each is maintained by Creative Commons.

### Open-source software licences used as data licences

- **SPDX License List** — canonical short identifiers and references: <https://spdx.org/licenses/>.
- **Open Source Initiative (OSI)** approved licence list: <https://opensource.org/licenses/>.
- **Apache License 2.0**: <https://www.apache.org/licenses/LICENSE-2.0>.
- **MIT License** (SPDX short identifier `MIT`): canonical text on SPDX list.
- **GNU GPL (v2, v3) and AGPL v3**: <https://www.gnu.org/licenses/>.
- **Mozilla Public License 2.0**: <https://www.mozilla.org/en-US/MPL/2.0/>.

## Opt-out signalling and crawler honour contracts

- **RFC 9309 — Robots Exclusion Protocol** (September 2022): <https://www.rfc-editor.org/rfc/rfc9309>.
- **IETF AI Preferences (ai-pref) working group** — charter, mailing list, and current drafts: <https://datatracker.ietf.org/wg/aipref/about/>. <!-- needs-research: verify published RFCs and active I-Ds at read time. -->
- **ai.txt** — community-maintained AI-crawler opt-out convention. Spawning reference: <https://site.spawning.ai/spawning-ai-txt>.
- **llms.txt** — community-maintained LLM-facing content convention; note that it is an *inclusion* signal, not an opt-out: <https://llmstxt.org/>.
- **DeviantArt `noai` meta tag announcement** — original publisher of the `noai` meta convention: <https://www.deviantart.com/team/journal/UPDATE-All-Deviations-Are-Opted-Out-of-AI-Datasets-934500371>. <!-- needs-research: verify current URL / archive at read time. -->

### Operator documentation for AI-specific crawlers

- **OpenAI** — GPTBot and ChatGPT-User opt-out: <https://platform.openai.com/docs/bots>.
- **Anthropic** — ClaudeBot and Claude-User documentation: <https://support.anthropic.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler>. <!-- needs-research: confirm Anthropic's current published crawler reference page. -->
- **Common Crawl Foundation — CCBot**: <https://commoncrawl.org/ccbot>.
- **Google** — Google-Extended and the full crawler list including Googlebot: <https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers>.
- **Apple** — Applebot and Applebot-Extended documentation: <https://support.apple.com/en-us/119829>. <!-- needs-research: verify current URL at read time. -->
- **Perplexity** — PerplexityBot: <https://docs.perplexity.ai/guides/bots>. <!-- needs-research: verify current URL / section. -->
- **Meta** — Meta-ExternalAgent crawler documentation: <https://developers.facebook.com/docs/sharing/webmasters/web-crawlers/>. <!-- needs-research: confirm current token and URL. -->
- **Cloudflare** — AI-bot management and pay-per-crawl: <https://blog.cloudflare.com/perplexity-is-using-stealth-undeclared-crawlers-to-evade-website-no-crawl-directives/> and <https://blog.cloudflare.com/control-content-use-for-ai-training/>. <!-- needs-research: cite the current platform-feature doc pages, not only blog posts, at read time. -->

## EU regulation

- **Regulation (EU) 2024/1689** — the EU Artificial Intelligence Act. EUR-Lex consolidated text: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Relevant articles for this module: Article 53 (training-data documentation, copyright policy), Annex XI, Annex XII.
- **Directive (EU) 2019/790** — Copyright in the Digital Single Market (the DSM Copyright Directive); Articles 3 and 4 TDM. EUR-Lex: <https://eur-lex.europa.eu/eli/dir/2019/790/oj>.
- **EU AI Office** — practitioner hub for the AI Act, including the GPAI Code of Practice and Article 53(1)(d) summary template. European Commission page: <https://digital-strategy.ec.europa.eu/en/policies/ai-office>.
- **GPAI Code of Practice** — successive draft and final versions published by the AI Office. <!-- needs-research: cite the controlling version at read time and the signatory list. -->
- **European Data Protection Board (EDPB)** — opinions on AI-training issues: <https://www.edpb.europa.eu/>. Of particular relevance: EDPB Opinion 28/2024 on certain data-protection aspects related to the processing of personal data in the context of AI models. <!-- needs-research: cite the controlling EDPB guidance in force at read time. -->
- **Regulation (EU) 2016/679** — General Data Protection Regulation (GDPR). EUR-Lex: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. Particularly Articles 5, 6, 9, 14, 22, 35 (DPIA).

## US copyright and the open litigation landscape

- **US Copyright Office** — Copyright and Artificial Intelligence report, multi-part: <https://www.copyright.gov/ai/>. Part 1 (Digital Replicas), Part 2 (Copyrightability), Part 3 (Training). <!-- needs-research: cite the latest issued parts and any superseding guidance at read time. -->
- **17 U.S.C. §105** — US Government works and copyright. Cornell Legal Information Institute: <https://www.law.cornell.edu/uscode/text/17/105>.
- **17 U.S.C. §107** — fair-use statute: <https://www.law.cornell.edu/uscode/text/17/107>.
- **PACER / CourtListener** — track the open training-data cases (plaintiffs including The New York Times, Universal Music Group, Getty Images, Concord Music, Ziff Davis, and individual authors, with defendants including OpenAI, Microsoft, Anthropic, Meta, Stability AI, Perplexity, and others). <!-- needs-research: verify case status, docket numbers, and any settlements at read time without opining on the merits. -->

## Other jurisdictions

- **Japan — Copyright Act Article 30-4** (as amended 2018). Japan Agency for Cultural Affairs guidance: <https://www.bunka.go.jp/>. <!-- needs-research: cite canonical English translation and any 2024-2025 guidance on AI training. -->
- **UK — Intellectual Property Office (IPO)** consultation and position on TDM: <https://www.gov.uk/government/organisations/intellectual-property-office>. <!-- needs-research: cite current UK IPO consultation outcome / published position on AI and copyright. -->
- **Singapore — Copyright Act 2021 §243** (computational data analysis): Singapore Attorney-General's Chambers statutes database: <https://sso.agc.gov.sg/Act/CA2021>.
- **South Korea — Copyright Act reforms** — Korea Copyright Commission: <https://www.copyright.or.kr/eng/>. <!-- needs-research: status of pending reforms at read time. -->
- **China — generative-AI measures** (Cyberspace Administration of China and related regulators). <!-- needs-research: cite the current Interim Measures for the Management of Generative AI Services and related rules at read time. -->

## US privacy regimes (reinforced from mod-101)

- **CCPA / CPRA** — California Privacy Rights Act amendments, Cal. Civ. Code §1798.100 et seq. California Privacy Protection Agency: <https://cppa.ca.gov/>.
- **Global Privacy Control (GPC)** specification: <https://globalprivacycontrol.org/>.

## Dataset documentation templates

- **Datasheets for Datasets** — Gebru, Morgenstern, Vecchione, Vaughan, Wallach, Daumé, Crawford. CACM 2021 (originally arXiv:1803.09010 2018): <https://arxiv.org/abs/1803.09010>.
- **Data Statements for NLP** — Bender and Friedman. TACL 2018: <https://aclanthology.org/Q18-1041/>.
- **The Dataset Nutrition Label** — Holland, Hosny, Newman, Joseph, Chmielinski. arXiv:1805.03677 2018: <https://arxiv.org/abs/1805.03677>. Current template set via the Data Nutrition Project: <https://datanutrition.org/>.
- **Model Cards for Model Reporting** — Mitchell, Wu, Zaldivar, Barnes, Vasserman, Hutchinson, Spitzer, Raji, Gebru. FAT* 2019: <https://arxiv.org/abs/1810.03993>.

## Machine-readable metadata formats

- **Croissant** — MLCommons dataset metadata vocabulary. Specification: <https://github.com/mlcommons/croissant>. Working-group page: <https://mlcommons.org/working-groups/data/croissant/>.
- **Hugging Face Dataset Cards** — card specification and YAML schema: <https://huggingface.co/docs/hub/datasets-cards>. Schema reference in <https://github.com/huggingface/hub-docs/blob/main/datasetcard.md>.
- **Schema.org Dataset vocabulary** — <https://schema.org/Dataset>. Croissant builds on schema.org.
- **Google Dataset Search** — consumer of schema.org Dataset + Croissant metadata: <https://datasetsearch.research.google.com/>.

## Supply-chain and provenance standards

- **CycloneDX** — SBOM / ML-BOM specification maintained by OWASP: <https://cyclonedx.org/>. CycloneDX 1.6 JSON schema: <https://cyclonedx.org/docs/1.6/json/>. ML-BOM capabilities page: <https://cyclonedx.org/capabilities/mlbom/>.
- **SPDX 3.0** — <https://spdx.dev/>. SPDX AI and Dataset profiles: <https://spdx.dev/learn/areas-of-interest/ai/>. ISO/IEC 5962:2021 (SPDX 2.2 base): <https://www.iso.org/standard/81870.html>.
- **SLSA** — Supply-chain Levels for Software Artifacts: <https://slsa.dev/>.
- **Sigstore** — signing, verification, and provenance for software artifacts: <https://www.sigstore.dev/>. `cosign` for blob signing.
- **in-toto** — framework for software-supply-chain attestations: <https://in-toto.io/>.

## C2PA and the Content Authenticity Initiative

- **C2PA** — Coalition for Content Provenance and Authenticity specifications: <https://c2pa.org/specifications/>. The specification versions at read time include the core specification and the extensions for AI training assertions. <!-- needs-research: cite the current spec version and the training-opt-out assertion names at read time. -->
- **Content Authenticity Initiative (CAI)** — Adobe-led practitioner hub: <https://contentauthenticity.org/>.
- **IPTC Photo Metadata Standard** — training-related rights tags that compose with C2PA: <https://iptc.org/standards/photo-metadata/>.
- **c2pa-rs** — official Rust implementation and bindings: <https://github.com/contentauth/c2pa-rs>.

## Headline third-party dataset hosts (referenced in Chapter 01)

- **Hugging Face Hub** — datasets: <https://huggingface.co/datasets>.
- **Kaggle** — datasets: <https://www.kaggle.com/datasets>.
- **Zenodo** — research-data repository hosted by CERN: <https://zenodo.org/>.
- **OpenML** — ML dataset and task catalog: <https://www.openml.org/>.
- **Papers With Code** — datasets index: <https://paperswithcode.com/datasets>.
- **AWS Open Data Registry**: <https://registry.opendata.aws/>.
- **Google Research datasets**: <https://research.google/resources/datasets/>.
- **data.gov** — US federal open data: <https://www.data.gov/>.
- **EU Open Data Portal**: <https://data.europa.eu/>.
- **MLCommons** — benchmark and dataset releases: <https://mlcommons.org/>.
- **BigCode — The Stack**: <https://www.bigcode-project.org/>. Opt-out check: <https://huggingface.co/spaces/bigcode/in-the-stack>.

## Public pretraining corpus documentation (companion reading)

- Soldaini, Kinney, Bhagia, Schwenk, Atkinson, Authur, Bogin, Chandu, Dumas, Elazar, Hofmann, Jha, Kumar, Lucy, Lyu, Lambert, Magnusson, Morrison, Muennighoff, Naik, Nam, Peters, Ravichander, Richardson, Shen, Strubell, Subramani, Tafjord, Walsh, Zettlemoyer, Smith, Hajishirzi, Beltagy, Groeneveld, Dodge, Lo. *Dolma: An Open Corpus of Three Trillion Tokens for Language Model Pretraining Research.* ACL 2024. <https://arxiv.org/abs/2402.00159>.
- Penedo, Kydlíček, Lozhkov, Mitchell, Raffel, Von Werra, Wolf. *The FineWeb Datasets: Decanting the Web for the Finest Text Data at Scale.* 2024. <https://arxiv.org/abs/2406.17557>.
- Gao, Biderman, Black, Golding, Hoppe, Foster, Phang, He, Thite, Nabeshima, Presser, Leahy. *The Pile: An 800GB Dataset of Diverse Text for Language Modeling.* arXiv:2101.00027 2020. <https://arxiv.org/abs/2101.00027>.
- Together AI, RedPajama team. *RedPajama: An Open Dataset for Training Large Language Models.* Dataset card and Technical Blog, 2023–2024. <https://together.ai/blog/redpajama-data-v2>.
- Kocetkov, Li, Allal, Li, Mou, Muñoz Ferrandis, Jernite, Mitchell, Hughes, Wolf, Bahdanau, Werra, de Vries. *The Stack: 3 TB of permissively licensed source code.* 2022. <https://arxiv.org/abs/2211.15533>.

## Canonical adversarial-sourcing incidents (companion reading)

- Thiel. *Identifying and Eliminating CSAM in Generative ML Training Data and Models.* Stanford Internet Observatory, December 2023. <https://purl.stanford.edu/kh752sm9123>. The LAION-5B investigation that led to the dataset's withdrawal and a published remediation plan.
- Longpre, Mahari, Chen, Chen, Obeng-Marnu, Sileo, Brannon, Muennighoff, Khazam, Kabbara, Perisetla, Wu, Shippole, Bollacker, Wu, Villa, Pentland, Hooker. *A Large-Scale Audit of Dataset Licensing and Attribution in AI.* Nature Machine Intelligence 2024. <https://www.nature.com/articles/s42256-024-00878-8>.

---

When citing any resource in a dataset card, governance pack, ML-BOM, or regulator-facing document, use the document title, publisher, and version date — not a summary URL. If a resource is time-sensitive (effective dates, draft-vs-final guidance, signatory lists, licence rulings), add a `<!-- needs-research: ... -->` note so a future pass can verify the current state.
