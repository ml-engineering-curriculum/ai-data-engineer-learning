# Resources for mod-104 — Data Quality Measurement, Representativeness, and Corpus Audit

Chapters in this module cite primary sources wherever possible. Prefer the primary source over any summary. Versions and publication dates matter — standards are revised and toolchains evolve — and the current state should be re-verified at read time.

## ISO standards — data quality for AI/ML

- **ISO/IEC 25012:2008** — *Software engineering — Software product Quality Requirements and Evaluation (SQuaRE) — Data quality model*. The fifteen-characteristic data-quality model this module is built on. ISO catalog: <https://www.iso.org/standard/35736.html>.
- **ISO/IEC 25024:2015** — *Software engineering — Software product Quality Requirements and Evaluation (SQuaRE) — Measurement of data quality*. The older measurement-of-data-quality standard; superseded for AI purposes by ISO/IEC 5259-2. <https://www.iso.org/standard/35749.html>.
- **ISO/IEC 5259-1:2024** — *Artificial intelligence — Data quality for analytics and machine learning (ML) — Part 1: Overview, terminology, and examples*. <https://www.iso.org/standard/81088.html>.
- **ISO/IEC 5259-2:2024** — *Part 2: Data quality measures*. The ML-specific measures catalogue referenced throughout this module; published November 2024. <https://www.iso.org/standard/81860.html>. Measure-family names in this module are paraphrased for pedagogy — re-verify the exact clause names against a purchased copy before citing them in an audit artifact.
- **ISO/IEC 5259-3:2024** — *Part 3: Data quality management requirements and guidelines*. Governance process. <https://www.iso.org/standard/81861.html>.
- **ISO/IEC 5259-4:2024** — *Part 4: Data quality process framework*. Lifecycle framing. <https://www.iso.org/standard/81862.html>.
- **ISO/IEC 5259-5:2025** — *Part 5: Data quality governance framework*. Governance layer; published February 2025. <https://www.iso.org/standard/84150.html>.
- **ISO/IEC 42001:2023** — *Artificial intelligence — Management system*. Clause 8 and Annex A.7 drive the data-engineering evidence. <https://www.iso.org/standard/81230.html>.
- **ISO/IEC 23894:2023** — *Artificial intelligence — Guidance on risk management*. Annex B catalogue of AI-related risk sources. <https://www.iso.org/standard/77304.html>.

## Great Expectations

- **Great Expectations documentation** — <https://docs.greatexpectations.io/>.
- **Great Expectations repository** — <https://github.com/great-expectations/great_expectations>.
- **Expectation catalogue (built-ins)** — browsable at <https://greatexpectations.io/expectations/>.
- Note: the Great Expectations 1.x line reshaped the Data Context API versus 0.x and is the current stable series (1.24.x as of late 2026). Pin the exact release you author against.

## Soda Core

- **Soda Core documentation** — <https://docs.soda.io/soda-core/overview.html>.
- **Soda Core repository** — <https://github.com/sodadata/soda-core>.
- **SodaCL check language reference** — <https://docs.soda.io/soda-cl/soda-cl-overview.html>.

## AWS Deequ

- **AWS Deequ repository** (Scala) — <https://github.com/awslabs/deequ>.
- **PyDeequ repository** (Python wrapper) — <https://github.com/awslabs/python-deequ>.
- **Schelter, Lange, Schmidt, Celikel, Biessmann, Grafberger.** *Automating large-scale data quality verification*. VLDB 2018. <https://www.vldb.org/pvldb/vol11/p1781-schelter.pdf>. The reference paper for Deequ's constraints-plus-metric-repository design.
- **AWS blog introduction** — <https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/>.

## pandera

- **pandera documentation** — <https://pandera.readthedocs.io/>.
- **pandera repository** — <https://github.com/unionai-oss/pandera>.
- **Bantilan, Niels.** *pandera: Statistical Data Validation of Pandas Dataframes*. Proceedings of the 19th Python in Science Conference (SciPy 2020). <https://proceedings.scipy.org/articles/Majora-342d178e-010>. DOI: 10.25080/Majora-342d178e-010.

## Dodge-style corpus audit

- **Dodge, Sap, Marasović, Agnew, Ilharco, Groeneveld, Mitchell, Gardner.** *Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus*. EMNLP 2021. <https://aclanthology.org/2021.emnlp-main.98/>. The canonical reference for the three-axis audit.
- **Raffel, Shazeer, Roberts, Lee, Narang, Matena, Zhou, Li, Liu.** *Exploring the Limits of Transfer Learning with a Unified Text-to-Text Transformer* (T5 and the C4 corpus). JMLR 2020. <https://arxiv.org/abs/1910.10683>.
- **LDNOOBW list of profane English words** — <https://github.com/LDNOOBW/List-of-Dirty-Naughty-Obscene-and-Otherwise-Bad-Words>. The blocklist analysed in the Dodge audit.
- **Blodgett, Green, O'Connor.** *Demographic Dialectal Variation in Social Media: A Case Study of African-American English*. EMNLP 2016. <https://aclanthology.org/D16-1120/>. The dialect classifier used in Dodge et al.'s Axis-2 analysis. Repo: <https://github.com/slanglab/twitteraae>.

## Kreutzer "Quality at a Glance"

- **Kreutzer, Caswell, Wang, et al.** *Quality at a Glance: An Audit of Web-Crawled Multilingual Datasets*. TACL 2022. <https://aclanthology.org/2022.tacl-1.4/>. The multilingual audit methodology operationalised in Chapter 05.
- **Kargaran, Imani, Yvon, Schütze.** *GlotLID: Language Identification for Low-Resource Languages*. ACL 2024 Findings. <https://aclanthology.org/2024.findings-acl.47/>. Modern alternative to FastText `lid.176`. Repo: <https://github.com/cisnlp/GlotLID>.
- **FastText `lid.176`** — Facebook AI Research. Model + docs: <https://fasttext.cc/docs/en/language-identification.html>.

## Representativeness and disaggregated counters

- **Blodgett, Barocas, Daumé, Wallach.** *Language (Technology) is Power: A Critical Survey of "Bias" in NLP*. ACL 2020. <https://aclanthology.org/2020.acl-main.485/>. The underlying critique that motivates disaggregation.
- **Bender, Friedman.** *Data Statements for Natural Language Processing*. TACL 2018. <https://aclanthology.org/Q18-1041/>. The structured demographic / subpopulation documentation that per-slice reports inherit.
- **Gebru, Morgenstern, Vecchione, Vaughan, Wallach, Daumé, Crawford.** *Datasheets for Datasets*. CACM 2021 / arXiv:1803.09010. <https://arxiv.org/abs/1803.09010>. The dataset-card format the per-slice quality report feeds.
- **Holland, Hosny, Newman, Joseph, Chmielinski.** *The Dataset Nutrition Label*. arXiv:1805.03677. <https://arxiv.org/abs/1805.03677>. An early representativeness-report precursor.
- **Mitchell, Wu, Zaldivar, Barnes, Vasserman, Hutchinson, Spitzer, Raji, Gebru.** *Model Cards for Model Reporting*. FAT* 2019. <https://arxiv.org/abs/1810.03993>. Not a data-side artifact but informs the measurement section of model-facing docs.
- **Hugging Face `datasets` with the `evaluate` / `datasets-sql` / `datasets-card-creator` ecosystem** — <https://huggingface.co/docs/datasets>. Card frontmatter and data-measurements block: <https://huggingface.co/docs/hub/datasets-cards>.

## DataComp / DataComp-LM

- **Gadre, Ilharco, Fang, Hayase, Smyrnis, Nguyen, Marten, Wortsman, Ghosh, Zhang, et al.** *DataComp: In Search of the Next Generation of Multimodal Datasets*. NeurIPS 2023. <https://arxiv.org/abs/2304.14108>. The benchmark + methodology.
- **Li, Fang, Smyrnis, Ivgi, Jordan, Gadre, Bansal, Guha, Keh, Arora, et al.** *DataComp-LM: In Search of the Next Generation of Training Sets for Language Models*. NeurIPS 2024. <https://arxiv.org/abs/2406.11794>. The LM extension.
- **DataComp-LM reference repository** — <https://github.com/mlfoundations/dclm>.
- **EleutherAI LM Evaluation Harness** — <https://github.com/EleutherAI/lm-evaluation-harness>. Standard eval runner for data-centric ablations.
- **HELM (Holistic Evaluation of Language Models)** — <https://crfm.stanford.edu/helm/>. Alternative / complement evaluation suite.
- **BIG-bench** — <https://github.com/google/BIG-bench>.
- **litgpt** — <https://github.com/Lightning-AI/litgpt>. Minimal training codebase suitable for ablations at smaller scales.
- **nanoGPT** — <https://github.com/karpathy/nanoGPT>. Smaller-still reference training code.
- **MosaicML Composer / LLM Foundry** — <https://github.com/mosaicml/composer>, <https://github.com/mosaicml/llm-foundry>.

## Benchmark contamination (previewing mod-106)

- **Brown, Mann, Ryder, Subbiah, Kaplan, et al.** *Language Models are Few-Shot Learners* (GPT-3), Appendix C on contamination. arXiv:2005.14165. <https://arxiv.org/abs/2005.14165>.
- **Shi, Xu, Wang, Choi, Zettlemoyer.** *Detecting Pretraining Data from Large Language Models* (Min-K% Prob). ICLR 2024. <https://arxiv.org/abs/2310.16789>.
- **Oren, Meister, Chatterji, Ladhak, Hashimoto.** *Proving Test Set Contamination in Black Box Language Models*. ICLR 2024. <https://arxiv.org/abs/2310.17623>.
- **EleutherAI LM Evaluation Harness decontamination recipes** — <https://github.com/EleutherAI/lm-evaluation-harness>.

## Public pretraining corpora whose audits are canonical reading

- **C4** — introduced in Raffel et al. 2020 (T5). Hugging Face mirror: <https://huggingface.co/datasets/allenai/c4>.
- **The Pile** — Gao, Biderman, Black et al. 2020. <https://arxiv.org/abs/2101.00027>. The full-corpus download from The Eye was taken down under rightsholder pressure; a "pile-uncopyrighted" subset remains on Hugging Face (<https://huggingface.co/datasets/monology/pile-uncopyrighted>), and EleutherAI's successor open corpus is Common Pile (2025).
- **RefinedWeb** — Penedo, Malartic, Hesslow et al. 2023. <https://arxiv.org/abs/2306.01116>. <https://huggingface.co/datasets/tiiuae/falcon-refinedweb>.
- **Dolma** — Soldaini, Kinney, Bhagia et al. 2024. <https://aclanthology.org/2024.acl-long.840/>. <https://huggingface.co/datasets/allenai/dolma>. Toolkit: <https://github.com/allenai/dolma>.
- **RedPajama-Data v2** — Together AI 2023. <https://www.together.ai/blog/redpajama-data-v2>. <https://huggingface.co/datasets/togethercomputer/RedPajama-Data-V2>.
- **FineWeb / FineWeb-Edu** — Penedo, Kydlíček, Allal, et al. 2024. <https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1>. <https://huggingface.co/datasets/HuggingFaceFW/fineweb>. Pipeline: <https://github.com/huggingface/datatrove>.

## Provenance, signing, chain of custody (previewing mod-109)

- **Sigstore** — <https://www.sigstore.dev/>.
- **SLSA framework** — <https://slsa.dev/spec/>.
- **CycloneDX ML-BOM specification** — <https://cyclonedx.org/>.
- **MLCommons Croissant metadata format** — <https://mlcommons.org/working-groups/data/croissant/>.
- **OpenLineage** — <https://openlineage.io/>.

## Human annotation and rating tooling (for Kreutzer-style audits)

- **Label Studio** — <https://labelstud.io/>.
- **Prodigy** — <https://prodi.gy/>.
- **Argilla** — <https://argilla.io/>.
- **Toloka** — <https://toloka.ai/>. (The worker-facing "Tolokers" product is now branded Mindrift following Toloka's 2025 spin-out from Nebius; the enterprise/data-services side still operates as Toloka. Regional availability varies.)
- **Appen** — <https://www.appen.com/>.
- **Amazon Mechanical Turk** — <https://www.mturk.com/>.

## Representativeness and divergence measures (background)

- **Kullback, Leibler.** *On Information and Sufficiency*. Annals of Mathematical Statistics 1951. The KL divergence primary source.
- **Lin, Jianhua.** *Divergence Measures Based on the Shannon Entropy*. IEEE Transactions on Information Theory 1991. The JSD primary source.
- **Rubner, Tomasi, Guibas.** *The Earth Mover's Distance as a Metric for Image Retrieval*. IJCV 2000. The EMD primary source. <https://doi.org/10.1023/A:1026543900054>.
- **SciPy `scipy.stats` reference** — <https://docs.scipy.org/doc/scipy/reference/stats.html>. KS, chi-square, and Wasserstein implementations.

---

When citing any resource in a slice manifest, dataset card, or governance package, include the document title, publisher, version / date, and (where applicable) the git SHA or Hugging Face revision hash of the toolkit or dataset you actually used. If the resource is time-sensitive (standards drafts, dataset hosting, operator catalogs), add a `<!-- needs-research: ... -->` note so a future pass can verify the current state.
