# Job Requirements — AI Data Engineer

**Role level:** 30 (hands-on engineering specialist — ML Engineering family)
**Track:** `ai-data-engineer-learning`
**Research window:** 2026-08-08 → 2026-11-06 (last 90 days)
**Today:** 2026-11-06
**Live postings sampled this cycle:** 29 across 20 employers.

This file documents the posting evidence used to validate the AI Data Engineer curriculum. Raw normalised data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); this cycle's proposed additions (none this cycle — see §"Delta" below) live in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Status — live sample, 29 postings

Sampled 29 live postings this cycle via WebSearch + WebFetch across greenhouse (`job-boards.greenhouse.io`), Ashby (`jobs.ashbyhq.com`), employer career sites, and third-party mirrors (used only when canonical pages render client-side). The sample intentionally spans frontier labs (Anthropic × 3, OpenAI, Google DeepMind, Meta Superintelligence Labs × 2, Mistral × 2, xAI, Reka), hyperscaler managed-AI platforms (Microsoft AI × 5, Apple), data-operations / labelling-focused employers (Scale AI, Snorkel AI × 2, Mercor), model providers (Together AI, Cohere, Stability AI, Character.AI, Perplexity, Anysphere/Cursor), research institutes (Allen Institute for AI), and a specialised data-curation startup (DatologyAI).

**Filter-out applied**: (a) general / BI data-engineer postings without AI/ML scope (dropped — classical DE), (b) Senior / Staff / Principal postings whose responsibility framing sits above level-30 scope (retained only when the bullets map to level-30 IC scope, flagged `seniority_bucket='senior'` or `'supplementary'`), (c) frontier-lab research-scientist postings requiring an ML PhD + ICML/NeurIPS pubs, (d) labeller contractor postings.

**Known extraction gaps** (next-cycle work): OpenAI Pretraining Data (canonical page client-rendered), AWS Bedrock Data for AI, Nvidia NeMo Curator team, additional Google DeepMind Staff SWE Data postings, Hugging Face canonical FineWeb data-engineer posting, Common Crawl, LAION paid-staff roles. Three employers surveyed but surfaced no in-window level-30 role: Groq, AI21 Labs (post-downsize), Fal.ai.

## Delta summary

**No module / exercise / project additions this cycle.** The 12-module curriculum already covers every theme that clears the continuity-bias threshold (≥3 distinct in-window postings AND frequency ≥0.30). Two emerging signals (RL environment authoring, multimodal data pipelines) and one other (data flywheel / eval-to-training feedback loop) have ≥3 posting evidence but fall below 0.30 frequency and can be covered incrementally by the existing modules; they are flagged in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) `requirement_themes` with `_emerging` suffixes so the next cycle can promote them if frequency crosses the bar on a larger sample. See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the empty-delta artifact and rationale.

## Methodology

1. Sourced the equivalent-titles set from the research prompt (AI Data Engineer / Data Engineer, AI / ML Data Engineer / Training Data Engineer / Data Curation Engineer / Synthetic Data Engineer / AI Data Operations Engineer) and widened it to the title shapes that surfaced in-sample (`Pre-training Data Infrastructure Engineer`, `Human Data Engineer`, `SWE, RL Data`, `SWE, Pretraining Data`, `Data Engineer, Data Flywheel`, `SWE, AI Data & Evaluation`, `SWE, GenAI Model Quality`).
2. Ran WebSearch across Greenhouse, Ashby, employer career sites, and third-party mirrors to find candidate URLs; ran WebFetch on each to extract requirement quotes. Where a canonical page rendered client-side, used a mirror (`freehire.me`, `jobs.generalcatalyst.com`, `radical.getro.com`, `careers.mavenventures.com`, `resumehog.com`, `welcometothejungle.com`, `builtinnyc.com`, `themuse.com`) and flagged the posting for canonical-verification next cycle.
3. Applied the four filter-out rules to drop postings that did not map to level-30 IC scope.
4. Computed per-theme frequency as (posting_count / 29). Themes with frequency ≥ 0.30 and existing coverage are left alone; themes below 0.30 are either already covered (no-op) or flagged `_emerging` for the next cycle.
5. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. Deferred coverage per the role hierarchy (see "Requirement themes → curriculum ownership" below).

## Requirement themes → curriculum ownership

The table maps each requirement theme to its primary owner on the role hierarchy, the in-window posting frequency, and the curriculum coverage path.

| # | Theme | Freq | n | Owner role | Coverage |
|---|---|---|---|---|---|
| 1 | Engineering-grade fluency with AI data-governance frameworks (NIST AI RMF + GenAI Profile, ISO/IEC 42001 clause 8, ISO/IEC 23894, ISO/IEC 5259, ISO/IEC 25012, EU AI Act Articles 10 + 53 + Annex XI, GDPR Art 5/6/9/14/22, CCPA/CPRA, EU DSM Art 4) | 0.10 | 3 | `ai-data-engineer` (this) | [`mod-101-ai-data-governance-and-position`](lessons/mod-101-ai-data-governance-and-position) |
| 2 | Data sourcing, licensing, consent, and provenance engineering — license matrix, robots.txt + ai.txt + 'noai' + GPTBot/CCBot/ClaudeBot opt-outs, Datasheets / Data Statements / Dataset Cards (HF, Google, Meta), Croissant, ML-BOM, C2PA | 0.10 | 3 | `ai-data-engineer` | [`mod-102-data-sourcing-licensing-and-provenance`](lessons/mod-102-data-sourcing-licensing-and-provenance) |
| 3 | Web-scale acquisition and curation pipelines (Common Crawl, CCNet, trafilatura/resiliparse, FastText language-id, Dolma / FineWeb / RedPajama / RefinedWeb reproduction) with Data-Juicer / NeMo Curator / Ray Data | **0.52** | **15** | `ai-data-engineer` | [`mod-103-web-scale-acquisition-and-curation`](lessons/mod-103-web-scale-acquisition-and-curation) |
| 4 | Data-quality measurement and audit (Great Expectations / Soda / Deequ / pandera suites aligned to ISO/IEC 25012 and 5259; Dodge-style corpus audit; DataComp / DataComp-LM methodology for data-centric ablations) | **0.41** | **12** | `ai-data-engineer` | [`mod-104-data-quality-measurement-and-audit`](lessons/mod-104-data-quality-measurement-and-audit) |
| 5 | Deduplication at trillion-token scale (exact + MinHash+LSH with datasketch / data-juicer / NeMo Curator / dolma; defending Lee 2021 / Kandpal 2022 memorisation + privacy gains; Nightshade / adversarial-perturbation detection) | 0.03 | 1 | `ai-data-engineer` | [`mod-105-deduplication-at-scale`](lessons/mod-105-deduplication-at-scale) |
| 6 | Benchmark contamination detection and decontamination (contamination firewall against MMLU/GSM8K/HumanEval/HELM/BigBench/LiveBench; defending Min-K% Prob + exchangeability detectors; lm-eval-harness recipes) | 0.03 | 1 | `ai-data-engineer` | [`mod-106-benchmark-contamination-and-decontamination`](lessons/mod-106-benchmark-contamination-and-decontamination) |
| 7 | Labelling operations and annotation programs (Label Studio / Argilla / Prodigy / Doccano; vendor contracting Scale AI / Surge AI / Appen / Sama / iMerit / Toloka; IAA with Krippendorff α; Settles active learning; Snorkel weak supervision; RLHF / DPO preference labelling) | 0.24 | 7 | `ai-data-engineer` | [`mod-107-labeling-operations-and-annotation-programs`](lessons/mod-107-labeling-operations-and-annotation-programs) |
| 8 | Synthetic data generation at production scale (Self-Instruct, Evol-Instruct, UltraChat, teacher-student distillation, preference-pair synthesis; defending Shumailov 2024 'Curse of Recursion'; privacy-preserving tabular synth with SDV / Gretel / MOSTLY AI) | 0.21 | 6 | `ai-data-engineer` | [`mod-108-synthetic-data-generation-at-production-scale`](lessons/mod-108-synthetic-data-generation-at-production-scale) |
| 9 | Dataset versioning, lineage, and reproducibility (DVC / lakeFS / Pachyderm; Delta Lake / Iceberg for lakehouses; OpenLineage / Marquez / DataHub / Unity Catalog; Croissant + CycloneDX ML-BOM + SPDX AI profile + SLSA + Sigstore signing) | 0.14 | 4 | `ai-data-engineer` | [`mod-109-dataset-versioning-lineage-reproducibility`](lessons/mod-109-dataset-versioning-lineage-reproducibility) |
| 10 | PII detection, redaction, and privacy risk for training data (Presidio / scrubadub / Google DLP / AWS Comprehend PII; redaction primitives; GDPR Art 6+9+14 + HIPAA lawful basis; defending Carlini 2020/2022 extraction + Kandpal 2022 dedup-privacy linkage) | 0.07 | 2 | `ai-data-engineer` | [`mod-110-pii-and-privacy-risk-for-training-data`](lessons/mod-110-pii-and-privacy-risk-for-training-data) |
| 11 | Data-quality monitoring and regression detection in production (data drift / schema drift / population drift; whylogs / Evidently / Monte Carlo / Soda Cloud; regression fixtures that gate re-release) | 0.21 | 6 | `ai-data-engineer` | [`mod-111-data-quality-monitoring-and-regression`](lessons/mod-111-data-quality-monitoring-and-regression) |
| 12 | Cross-functional data program, dataset-card authoring, and communication (HF / Google / Meta dataset cards; Datasheets end-to-end; EU AI Act Article 53 + Annex XI training-data summary for GPAI; model-card data-slice authoring; data council brief) | 0.10 | 3 | `ai-data-engineer` | [`mod-112-cross-functional-data-program-and-communication`](lessons/mod-112-cross-functional-data-program-and-communication) |
| 13 | Python / PyTorch / Hugging Face Datasets / pandas / SQL / Parquet / Arrow fluency; distributed data processing (Spark / Ray / Flink / Beam) | 0.52 | 15 (prereq) | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 14 | dbt / SQL warehouses / analytics engineering (BigQuery / Snowflake / Redshift / ClickHouse; Airflow / Dagster / Prefect orchestration) | 0.17 | 5 (prereq) | `ml-engineer` (level 20) | Prereq — covered by ml-engineer level-20; AI-data-engineer plugs into that substrate |
| 15 | AI-system inventory maintenance, first-draft impact / data-protection impact assessments, framework-crosswalk legwork | 0.00 | 0 | `ai-governance-analyst` (level 15) | Assumed as familiarity; linked out |
| 16 | MLOps automation and CI/CD-for-ML (Airflow / Dagster / Prefect / Argo pipelines, model registry, deployment gates) | 0.17 | 5 (peer) | `ai-infra-mlops` (level 25) | mod-111 documents the integration contract; deep orchestrator platform engineering linked out |
| 17 | Harm modelling, adversarial-ML engineering, red-team engineering, guardrail engineering | 0.00 | 0 (peer) | `ai-risk-engineer` (level 25) | mod-105 covers data-side defences at working depth; deep adversarial-ML craft linked out |
| 18 | SFT dataset design depth (dataset-mix cards, preference-pair curation), PEFT / quantisation / serving depth | 0.00 | 0 (peer) | `fine-tuning-engineer` (level 30) | mod-107 + mod-108 hand off to that peer; depth linked out |
| 19 | Retrieval-side document ingestion and chunking / embeddings / vector DBs / hybrid retrieval depth | 0.00 | 0 (peer) | `rag-engineer` (level 30) | mod-103 supplies the clean corpus; retrieval contract depth linked out |
| 20 | Benchmark construction, judge-vs-human calibration, gold-standard eval-set labelling methodology depth | 0.03 | 1 (peer) | `model-evaluation-engineer` (level 30) | mod-107 covers production labelling ops; benchmark-construction depth linked out |
| 21 | Tokeniser training depth (SentencePiece / byte-level BPE on domain-adapted corpus, vocab design) | 0.03 | 1 (peer) | `nlp-engineer` (level 30) | mod-103 ends at pretokenisation corpus; tokeniser-training step linked out |
| 22 | Training-cluster-side ingestion depth (WebDataset / Mosaic Streaming / Ray Data at shard scale, tokenisation throughput, I/O pipelines) | 0.00 | 0 (next-up) | `training-pipeline-engineer` (level 35) | mod-109 hand-off contract; cluster execution linked out |
| 23 | Self-service ML platform engineering (feature store, dataset registry, lineage-platform runtime) | 0.00 | 0 (peer) | `ai-infra-ml-platform` (level 30) | mod-109 plugs into the platform; platform engineering linked out |
| 24 | Deep ML / AI security at platform scale (model-extraction defence, judge supply-chain, eval-set exfiltration prevention) | 0.00 | 0 (next-up) | `ai-infra-security` (level 35) | Working depth in mod-105 and mod-110; platform-scale depth linked out |
| 25 | Release-assurance / audit-trail / regulator-facing methodology depth | 0.00 | 0 (next-up) | `ai-evaluation-engineer` (level 35, Governance family) | Interface in mod-112; depth linked out |
| 26 | Foundation-model data-mixture strategy at org scale | 0.03 | 1 (next-up) | `staff-ml-engineer` (level 40) | mod-104 + mod-112 supply the per-source datasets; mixture strategy linked out |

### Emerging (below 0.30 — monitor for next cycle, do NOT add content this cycle)

| # | Theme | Freq | n | Current coverage | Suggested next step if it crosses 0.30 |
|---|---|---|---|---|---|
| E-1 | RL environment authoring for post-training — sandboxing, graders, task coverage, reward-hacking QA, agentic task environments | 0.21 | 6 | Partially covered in mod-107 (RLHF/DPO preference labelling) and mod-108 (preference-pair synthesis). Deep RL craft (environment authoring itself) remains `fine-tuning-engineer` scope. | Add an exercise to mod-107 or mod-108 — "wire a RLHF/DPO preference task into an executable grader harness with a reward-hacking QA fixture"; not a new module. |
| E-2 | Multimodal (text / image / audio / video) training-data pipelines — from acquisition through curation, dedup, quality measurement | 0.21 | 6 | mod-103 covers web-scale acquisition with text-first primitives; multimodal is implicit via Data-Juicer / NeMo Curator operators (both multimodal-capable). | Add an exercise to mod-103 — "image-text pair curation shape (WebDataset tar vs parquet-with-blob), video shard sampling, per-modality quality signal"; not a new module. |
| E-3 | Data flywheel / eval-to-training feedback loops — close the signal loop from production eval and user signals back into training-data refresh | 0.17 | 5 | mod-111 covers data-quality monitoring and regression detection; the "eval-to-training feedback loop" framing is adjacent but not an explicit learning objective yet. | Add an exercise to mod-111 — "close the data flywheel: wire evaluation telemetry into a dataset-mixture refresh pin + regression fixture"; not a new module. |

## Posting evidence (29 postings, 20 employers)

Postings are keyed by `id` in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) `postings[]`. The table below lists the 29 postings with employer, exact title, URL, salary (if published), and the primary themes they match.

| # | Employer | Title | URL | Salary | Themes |
|---|---|---|---|---|---|
| 1 | Anthropic | Pre-training Data Infrastructure Engineer | https://job-boards.greenhouse.io/anthropic/jobs/4973067008 | USD 500K-850K | 3, 4, 5, 9 |
| 2 | Anthropic | Data Operations Manager, Human Data | https://job-boards.greenhouse.io/anthropic/jobs/5238460008 | USD 270K-365K | 7, E-1, 4, 12 |
| 3 | Anthropic | Software Engineer, RL Data | https://jobs.generalcatalyst.com/companies/anthropic/jobs/81904229-software-engineer-rl-data | USD 320K-485K | E-1, 7, 4 |
| 4 | Apple | Machine Learning Engineer, Data for Foundation Models | https://www.themuse.com/jobs/apple/machine-learning-engineer-data-for-foundation-models | USD 150K-278K | 3, 4, E-2 |
| 5 | Microsoft AI | Data Engineer, AI Infra | https://microsoft.ai/careers/job/4398696009/data-engineer-ai-infra/ | GBP 94K-162K | 3, 4, E-2 |
| 6 | Microsoft AI | Data Engineer, Data Flywheel | https://microsoft.ai/careers/job/4402316009/data-engineer-data-flywheel/ | USD 143K-275K | 1, 2, 8, 10, E-3, 4 |
| 7 | Microsoft AI | Data Engineer, AI Data Platform | https://microsoft.ai/careers/job/4402634009/data-engineer-ai-data-platform/ | USD 143K-296K | 3, 8, E-2, 9, 6 |
| 8 | Microsoft AI | Technical Program Manager, Pretraining Data | https://microsoft.ai/careers/job/4464760009/technical-program-manager-pretraining-data/ | USD 120K-261K | 12 |
| 9 | Microsoft AI | Researcher - Pretraining Data | https://job-boards.greenhouse.io/microsoftcorporation/jobs/4453591009 | GBP 75K-123K | 3, 4, 1 |
| 10 | xAI | Human Data Engineer | https://careers.mavenventures.com/companies/x-ai/jobs/88951863-human-data-engineer | USD 144K-220K + equity | 7, 4, E-2 |
| 11 | Cohere | Data Engineer, Data Foundations | https://freehire.me/jobs/data-engineer-data-foundations-cohere-d3dex256 | USD 190K-325K | 13, 14 (prereqs) |
| 12 | Mercor | Software Engineer, AI Data & Evaluation | https://jobs.generalcatalyst.com/companies/mercor/jobs/80854015-software-engineer-ai-data-evaluation | USD 130K-500K + equity | 3, 4, 8, E-1, E-3 |
| 13 | Meta (Superintelligence Labs) | Data Engineer, Meta Superintelligence Labs | https://freehire.me/jobs/data-engineer-meta-superintelligence-labs-meta-blhc5mrv | — | 3 |
| 14 | Meta (Superintelligence Labs) | Data Engineer, MSL (Privacy) | https://freehire.me/jobs/data-engineer-meta-superintelligence-labs-privacy-meta-ijuysbj3 | — | 2, 10, 1 |
| 15 | Snorkel AI | Senior/Staff FDE - Synthetic Data Generation | https://job-boards.greenhouse.io/snorkelai/jobs/6167063004 | USD 180K-320K | 8, 7, E-1 |
| 16 | Snorkel AI | Senior / Staff Software Engineer — AI Platform | https://job-boards.greenhouse.io/snorkelai/jobs/6185944004 | — | 11, 8, E-1 |
| 17 | Reka AI | Member of Technical Staff (Data Engineer) | https://radical.getro.com/companies/reka-ai-2/jobs/39435269-member-of-technical-staff-data-engineer | — | 3, 8, E-2 |
| 18 | Mistral AI | Data Engineer, Senior | https://www.welcometothejungle.com/en/companies/mistral-ai/jobs/data-engineer-senior_paris | — | 13, 14 (prereqs) |
| 19 | Mistral AI | Research Engineer, Data Infrastructure | https://resumehog.com/jobs/j/e5/34/research-engineer-data-infrastructure-mistral-ai-f6ecdeb0.html | — | 3, 4 |
| 20 | OpenAI | Data Engineer, Scaling Analytics | https://openai.com/careers/data-engineer-scaling-analytics-san-francisco/ | USD 235K-385K + equity | 13 (prereq) |
| 21 | Google DeepMind | Research Engineer, Gemini Large Scale Pretraining | https://www.google.com/about/careers/applications/jobs/results/142611399284007622-research-engineer-gemini-large-scale-pretraining-deepmind | USD 147K-210K | 3, 4 |
| 22 | Together AI | Data Engineer, Research | https://job-boards.greenhouse.io/togetherai/jobs/4737079007 | USD 160K-220K + equity | 3, 4, 26 |
| 23 | Anysphere (Cursor) | Software Engineer, Pretraining Data | https://cursor.com/careers/software-engineer-pretraining-data | — | 2, 3, 9 |
| 24 | Perplexity AI | AI Software Engineer - Data Platform | https://www.perplexity.ai/hub/careers | USD 200K-280K | 3, E-3 |
| 25 | Stability AI | Staff Software Engineer, Data | https://stability.ai/careers | USD 176K-230K | 4, 1, 14 |
| 26 | DatologyAI | Software Engineer, Data Infrastructure | https://jobs.ashbyhq.com/DatologyAI/7a64144a-c612-4af8-bdf8-cdfff03240cc | USD 180K-250K | 3, 4, 13 |
| 27 | Allen Institute for AI | Senior Software Engineer, Data | https://job-boards.greenhouse.io/thealleninstitute/jobs/7872631 | USD 126K-189K | 3, 9 |
| 28 | Character.AI | Staff Software Engineer - Data Engineer | https://character.ai/careers | USD 150K-250K + equity | E-3, 7 |
| 29 | Scale AI | Software Engineer, GenAI Model Quality | https://scale.com/careers/4441791005 | — | 11, 7, E-1, 4 |

## Salary evidence

19 of 29 postings publish a salary range. Analysis in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) `salary_summary`.

| Geography / seniority | n | Representative band | Notes |
|---|---|---|---|
| US level-30 IC | 11 | **USD 160K-325K base** | Frontier-lab outliers (Anthropic Pre-training Data Infra at 500-850K; Anthropic SWE RL Data at 320-485K; OpenAI at 235-385K + equity) push the top of the band; the mode sits around 160-280K. Microsoft AI, Together AI, Perplexity, Cohere cluster in 160-325K. |
| US level-30 "senior-staff-flavoured" IC | 3 | **USD 150K-320K + equity** | Snorkel Sr/Staff FDE (180-320K), Stability Staff SWE (176-230K), Character.AI Staff SWE (150-250K + equity). Responsibility framing maps to level-30 IC; "Staff" title but not org-leadership scope. |
| EU-based level-30 IC | 3 | **GBP 75K-162K base** | Microsoft AI London/Zurich leads the band at £94K-£162K. Microsoft Researcher Pretraining Data (London) at £75K-£123K is the lower anchor. DeepMind Gemini at USD 147K-210K (UK-weighted). |
| Supplementary (program / operations) | 2 | USD 120K-365K | Anthropic Data Ops Manager (270-365K) is a program-leadership / ops-crossover role, not a pure IC. Microsoft TPM Pretraining Data (120-261K) is a program-management role; retained as supplementary for coverage mapping only. |

**Headline band for the next cycle to reference**: USD 160,000-325,000 base for US level-30 IC AI Data Engineer, with frontier-lab outliers running USD 320K-850K inclusive of equity. The USD 500K-850K Anthropic band is a frontier-lab + distributed-systems premium, not a norm.

## Ownership map — quick reference for next cycle

When backfilling postings, use this ownership decision to keep the curriculum from drifting into peer territory:

- **ML Engineer** (level 20, `ml-engineer-learning`) — owns classical data-engineering fundamentals (pandas, SQL, Parquet, basic DVC, feature-store consumer tier) and the classical-ML stack. **This AI Data Engineer assumes those and does not re-teach them.** Prereq frequency this cycle: 0.52 for distributed data processing; 0.17 for dbt / warehousing.
- **AI Governance Analyst** (level 15, `ai-governance-analyst-learning`) — owns operational analyst work. **This role consumes those artifacts.**
- **AI Risk Engineer** (peer, level 25, `ai-risk-engineer-learning`) — owns the engineering craft of AI risk. **This role supplies the dataset provenance, contamination report, PII-scrub log, and dataset card that peer consumes.**
- **AI Infra MLOps** (peer, level 25, `ai-infra-mlops-learning`) — owns MLOps automation and orchestrator platforms. **This role authors the data-pipeline shape; that peer runs the scheduler.**
- **AI Data Engineer** (this track, level 30, ML Engineering family) — owns the engineering craft of the training / evaluation data pipeline (sourcing, licensing, curation, dedup, decontamination, labelling ops, synthetic generation, versioning + lineage, PII scrub, quality monitoring, dataset-card authoring).
- **Fine-Tuning Engineer** (peer, level 30, `fine-tuning-engineer-learning`) — owns SFT dataset design, preference-pair curation, DPO/ORPO pair generation, and deep RL craft (environment authoring itself). **This role supplies the raw corpus + data-pipeline for RL environments; that peer owns the environment shape.**
- **RAG Engineer** (peer, level 30, `rag-engineer-learning`) — owns retrieval-side chunking / embeddings / vector DBs / hybrid retrieval.
- **Model Evaluation Engineer** (peer, level 30, `model-evaluation-engineer-learning`) — owns gold-standard eval-set labelling methodology, judge-vs-human calibration, canary-set design.
- **NLP Engineer** (peer, level 30, `nlp-engineer-learning`) — owns tokeniser training depth.
- **AI Eval Engineer** (peer, level 30, AI Engineering family, `ai-eval-engineer-learning`) — owns application-trace / judge / online-eval depth.
- **AI Infra ML Platform** (peer, level 30, `ai-infra-ml-platform-learning`) — owns self-service ML platform.
- **Training Pipeline Engineer** (next-up, level 35, `training-pipeline-engineer-learning`) — owns cluster-side ingestion.
- **AI Infra Security** (next-up, level 35, `ai-infra-security-learning`) — owns deep ML / AI security at platform scale.
- **AI Evaluation Engineer** (next-up, level 35, Governance family, `ai-evaluation-engineer-learning`) — owns release-assurance methodology.
- **Staff ML Engineer** (next-up, level 40, `staff-ml-engineer-learning`) — owns foundation-model data-mixture strategy.
- **Principal ML Engineer** (level 48) / **Head of AI Governance** (level 60) / **Chief AI Officer** (level 70) — leadership scope.

## External resources for out-of-scope items

Themes with no in-window posting evidence that remain curriculum-adjacent but belong to another track (listed with an authoritative reference so learners can self-serve if they need to cross into that peer's scope):

- **SFT dataset design depth / preference-pair curation** → [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning); reference reading: [DPO (Rafailov et al. 2023)](https://arxiv.org/abs/2305.18290), [UltraFeedback (Cui et al. 2023)](https://arxiv.org/abs/2310.01377).
- **Retrieval-side chunking + embeddings + vector DBs + hybrid retrieval** → [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning); reference reading: [LlamaIndex RAG docs](https://docs.llamaindex.ai/en/stable/), [LangChain Retrieval](https://python.langchain.com/docs/modules/data_connection/).
- **Benchmark construction + judge calibration** → [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning); reference reading: [HELM (Liang et al. 2022)](https://arxiv.org/abs/2211.09110), [Chatbot Arena (Chiang et al. 2024)](https://arxiv.org/abs/2403.04132).
- **Cluster-side ingestion depth (WebDataset / Mosaic Streaming / Ray Data at shard scale)** → [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning); reference reading: [WebDataset](https://github.com/webdataset/webdataset), [Mosaic StreamingDataset](https://docs.mosaicml.com/projects/streaming/en/stable/).
- **Adversarial ML / harm modelling / data poisoning defence at depth** → [`ai-risk-engineer-learning`](https://github.com/ai-governance-curriculum/ai-risk-engineer-learning); reference reading: [MITRE ATLAS](https://atlas.mitre.org/), [OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/).
- **Release-assurance methodology for regulators** → [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning); reference reading: [EU AI Act Article 15](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) + [EU AI Office Code of Practice](https://digital-strategy.ec.europa.eu/en/policies/ai-code-practice).
- **Foundation-model data-mixture strategy** → [`staff-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning); reference reading: [DoReMi (Xie et al. 2023)](https://arxiv.org/abs/2305.10429), [DataComp-LM (Li et al. 2024)](https://arxiv.org/abs/2406.11794).

## Authoritative public references (unchanged from prior cycle — authoritative basis for mod-101 → mod-112)

- **International standards** — [NIST AI RMF 1.0](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf), [NIST AI RMF Playbook](https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook), [NIST AI RMF GenAI Profile](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile), [ISO/IEC 42001:2023](https://www.iso.org/standard/81230.html), [ISO/IEC 23894:2023](https://www.iso.org/standard/77304.html), [ISO/IEC 5259 series](https://www.iso.org/standard/81088.html), [ISO/IEC 25012](https://www.iso.org/standard/35736.html).
- **Regulation and public policy** — [EU AI Act (Regulation 2024/1689)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) with focus on **Articles 10 and 53 + Annex XI**, [EU DSM Copyright Directive Article 4 (TDM exception)](https://eur-lex.europa.eu/eli/dir/2019/790/oj), [GDPR](https://gdpr-info.eu/), [CCPA / CPRA](https://oag.ca.gov/privacy/ccpa), [FDA GMLP + PCCP](https://www.fda.gov/medical-devices/software-medical-device-samd/good-machine-learning-practice-medical-device-development-guiding-principles), [US Federal Reserve SR 11-7](https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm), [Colorado AI Act (SB 24-205)](https://leg.colorado.gov/bills/sb24-205).
- **Dataset documentation standards** — [Datasheets for Datasets (Gebru et al. 2018)](https://arxiv.org/abs/1803.09010), [Data Statements for NLP (Bender & Friedman 2018)](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00041/43452), [Model Cards (Mitchell et al. 2019)](https://arxiv.org/abs/1810.03993), [Hugging Face Dataset Cards](https://huggingface.co/docs/hub/datasets-cards), [MLCommons Croissant](https://mlcommons.org/working-groups/data/croissant/), [CycloneDX ML-BOM](https://cyclonedx.org/capabilities/mlbom/), [C2PA Content Credentials](https://c2pa.org/).
- **Crawling and licensing contracts** — [RFC 9309 Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309), [IETF ai-pref vocabulary draft](https://datatracker.ietf.org/doc/draft-ietf-aipref-vocab/).
- **Reference corpora and papers** — [Common Crawl](https://commoncrawl.org/), [CCNet (Wenzek et al. 2019)](https://arxiv.org/abs/1911.00359), [The Pile (Gao et al. 2020)](https://arxiv.org/abs/2101.00027), [Documenting C4 (Dodge et al. 2021)](https://arxiv.org/abs/2104.08758), [Dolma (Soldaini et al. 2024)](https://arxiv.org/abs/2402.00159), [DataComp-LM (Li et al. 2024)](https://arxiv.org/abs/2406.11794), [FineWeb and FineWeb-Edu](https://huggingface.co/datasets/HuggingFaceFW/fineweb).
- **Extraction and curation tools** — [trafilatura](https://trafilatura.readthedocs.io/), [Alibaba Data-Juicer](https://github.com/modelscope/data-juicer), [NVIDIA NeMo Curator](https://docs.nvidia.com/nemo-framework/user-guide/latest/datacuration/index.html).
- **Deduplication and contamination** — [Lee et al. 2021](https://arxiv.org/abs/2107.06499), [datasketch](https://ekzhu.com/datasketch/), [Shi et al. 2023 Min-K% Prob](https://arxiv.org/abs/2310.16789), [Oren et al. 2023](https://arxiv.org/abs/2310.17623), [EleutherAI LM Eval Harness](https://github.com/EleutherAI/lm-evaluation-harness).
- **Labelling and preference** — [Label Studio](https://labelstud.io/), [Argilla](https://docs.argilla.io/), [Snorkel Data Programming (Ratner 2016)](https://arxiv.org/abs/1605.07723), [Settles 2009 Active Learning Survey](https://minds.wisconsin.edu/handle/1793/60660).
- **Synthetic data** — [Self-Instruct (Wang 2022)](https://arxiv.org/abs/2212.10560), [WizardLM / Evol-Instruct (Xu 2023)](https://arxiv.org/abs/2304.12244), [UltraChat (Ding 2023)](https://arxiv.org/abs/2305.14233), [The Curse of Recursion (Shumailov 2024)](https://arxiv.org/abs/2305.17493).
- **Versioning, lineage, metadata** — [DVC](https://dvc.org/doc), [lakeFS](https://docs.lakefs.io/), [OpenLineage](https://openlineage.io/docs/) + [Marquez](https://marquezproject.ai/docs/), [DataHub](https://datahubproject.io/docs/), [Databricks Unity Catalog](https://docs.unitycatalog.io/), [WebDataset](https://github.com/webdataset/webdataset), [MosaicML StreamingDataset](https://docs.mosaicml.com/projects/streaming/en/stable/).
- **PII and privacy risk** — [Microsoft Presidio](https://microsoft.github.io/presidio/), [Google Cloud DLP](https://cloud.google.com/sensitive-data-protection/docs), [Carlini 2020](https://arxiv.org/abs/2012.07805), [Carlini 2022](https://arxiv.org/abs/2202.07646), [Kandpal 2022](https://arxiv.org/abs/2202.06539).
- **Quality / observability** — [Great Expectations](https://docs.greatexpectations.io/docs/), [Soda Core](https://docs.soda.io/soda-core/overview-main.html), [whylogs](https://whylogs.readthedocs.io/), [Evidently AI](https://docs.evidentlyai.com/).
- **Security framing** — [OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/), [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/), [MITRE ATLAS](https://atlas.mitre.org/).

<!-- needs-research: Next cycle: (1) verify the four mirror-sourced postings (Cursor, Perplexity, DatologyAI, Scale AI) against canonical career-site pages; (2) sample OpenAI Pretraining Data, AWS Bedrock Data for AI, Nvidia NeMo Curator team, Google DeepMind Staff SWE Data (beyond the one captured), Hugging Face FineWeb data-engineer so emerging signals (RL environments, multimodal, data flywheel) can be re-scored on n>29; (3) if any emerging theme crosses 0.30 frequency, add the specific exercise flagged in the Emerging table above to the existing module rather than creating a new module. -->
