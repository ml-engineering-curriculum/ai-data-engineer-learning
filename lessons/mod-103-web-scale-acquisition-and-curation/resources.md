# Resources for mod-103 — Web-Scale Data Acquisition and Curation Pipelines

Chapters in this module cite primary sources wherever possible. Prefer the primary source over any summary. Version and publication dates matter — operator libraries and reference datasets evolve, and the current state should be re-verified at read time.

## Common Crawl and WARC

- **Common Crawl Foundation** — snapshots, index, and operational documentation: <https://commoncrawl.org/>. Monthly snapshots live at `s3://commoncrawl/` and the HTTPS mirror `https://data.commoncrawl.org/`.
- **Common Crawl columnar index** — Parquet index documentation and schema: <https://commoncrawl.org/blog/index-to-warc-files-and-urls-in-columnar-format>.
- **CCBot crawling policy** — <https://commoncrawl.org/ccbot>. Relevant for the chain-of-custody discussion in Chapter 06.
- **ISO 28500:2017** — WARC File Format. ISO catalog entry: <https://www.iso.org/standard/68004.html>. The normative reference for all WARC-reading code.
- **`warcio`** — maintained WARC reader for Python (BSD): <https://github.com/webrecorder/warcio>.
- **`cdx_toolkit`** — CDX / columnar index access toolkit: <https://github.com/cocrawler/cdx_toolkit>.

## The CCNet pipeline and language models

- **Wenzek, Lachaux, Conneau, Chaudhary, Guzmán, Joulin, Grave.** *CCNet: Extracting High Quality Monolingual Datasets from Web Crawl Data.* LREC 2020. <https://aclanthology.org/2020.lrec-1.494/>. The canonical CCNet-pipeline reference. Code: <https://github.com/facebookresearch/cc_net>.
- **FastText `lid.176` model** — Facebook AI Research; model download + language-identification docs at <https://fasttext.cc/docs/en/language-identification.html>. The 176-language classifier used across the field.
- **Kargaran, Imani, Yvon, Schütze.** *GlotLID: Language Identification for Low-Resource Languages.* ACL 2024 Findings. <https://aclanthology.org/2024.findings-acl.47/>. Modern alternative for broader language coverage. Repo: <https://github.com/cisnlp/GlotLID>.
- **KenLM language modelling toolkit** — Heafield 2011. Project page: <https://kheafield.com/code/kenlm/>. The perplexity-filter primitive used in CCNet.
- **Heafield, Kenneth.** *KenLM: Faster and Smaller Language Model Queries.* WMT 2011. <https://aclanthology.org/W11-2123/>.

## HTML main-content extraction

- **Barbaresi, Adrien.** *Trafilatura: A Web Scraping Library and Command-Line Tool for Text Discovery and Retrieval.* ACL 2021 System Demonstrations. <https://aclanthology.org/2021.acl-demo.15/>. The trafilatura reference. Repo: <https://github.com/adbar/trafilatura>. Docs: <https://trafilatura.readthedocs.io/>.
- **Resiliparse** — part of the ChatNoir toolchain, Bauhaus-Universität Weimar. Docs: <https://resiliparse.chatnoir.eu/>. Repo: <https://github.com/chatnoir-eu/chatnoir-resiliparse>.
- **Kohlschütter, Fankhauser, Nejdl.** *Boilerplate Detection using Shallow Text Features.* WSDM 2010. <https://dl.acm.org/doi/10.1145/1718487.1718542>. The foundational text-density approach that both modern extractors descend from.
- **ScrapingHub article-extraction-benchmark** — labelled HTML corpus for benchmarking main-content extractors: <https://github.com/scrapinghub/article-extraction-benchmark>.

## Public reference pipelines

- **Dodge, Sap, Marasović, Agnew, Ilharco, Groeneveld, Mitchell, Gardner.** *Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus.* EMNLP 2021. <https://aclanthology.org/2021.emnlp-main.98/>. The C4 audit.
- **Raffel, Shazeer, Roberts, Lee, Narang, Matena, Zhou, Li, Liu.** *Exploring the Limits of Transfer Learning with a Unified Text-to-Text Transformer.* JMLR 2020 (introducing T5 and C4). <https://arxiv.org/abs/1910.10683>.
- **Penedo, Malartic, Hesslow, Cojocaru, Cappelli, Alobeidli, Pannier, Almazrouei, Launay.** *The RefinedWeb Dataset for Falcon LLM: Outperforming Curated Corpora with Web Data, and Web Data Only.* NeurIPS 2023. <https://arxiv.org/abs/2306.01116>. Dataset on Hugging Face: <https://huggingface.co/datasets/tiiuae/falcon-refinedweb>.
- **Gao, Biderman, Black, Golding, Hoppe, Foster, Phang, He, Thite, Nabeshima, Presser, Leahy.** *The Pile: An 800GB Dataset of Diverse Text for Language Modeling.* arXiv:2101.00027 (2020). <https://arxiv.org/abs/2101.00027>. <!-- needs-research: confirm current hosting state of The Pile; portions have been removed from public distribution. -->
- **Soldaini, Kinney, Bhagia, Schwenk, Atkinson, Authur, et al.** *Dolma: An Open Corpus of Three Trillion Tokens for Language Model Pretraining Research.* ACL 2024. <https://aclanthology.org/2024.acl-long.840/>. Dataset: <https://huggingface.co/datasets/allenai/dolma>. Toolkit: <https://github.com/allenai/dolma>.
- **Together AI blog.** *RedPajama-Data-v2: An open dataset with 30 trillion tokens for training large language models.* 2023. <https://www.together.ai/blog/redpajama-data-v2>. Repo: <https://github.com/togethercomputer/RedPajama-Data>. Dataset: <https://huggingface.co/datasets/togethercomputer/RedPajama-Data-V2>.
- **Penedo, Kydlíček, allal, Lozhkov, Mitchell, Raffel, Von Werra, Wolf.** *FineWeb: decanting the web for the finest text data at scale.* 2024. Technical blog: <https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1>. Dataset: <https://huggingface.co/datasets/HuggingFaceFW/fineweb>. FineWeb-Edu: <https://huggingface.co/datasets/HuggingFaceFW/fineweb-edu>. Pipeline: <https://github.com/huggingface/datatrove>.
- **Li, Fang, Smyrnis, Ivgi, Jordan, Gadre, Bansal, Guha, Keh, Arora, et al.** *DataComp-LM: In Search of the Next Generation of Training Sets for Language Models.* NeurIPS 2024. <https://arxiv.org/abs/2406.11794>. The ablation methodology cited in Chapter 04.
- **Lee, Ippolito, Nystrom, Zhang, Eck, Callison-Burch, Carlini.** *Deduplicating Training Data Makes Language Models Better.* ACL 2022. <https://aclanthology.org/2022.acl-long.577/>. The core dedup-improves-quality result.
- **Kandpal, Wallace, Raffel.** *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022. <https://proceedings.mlr.press/v162/kandpal22a.html>. The dedup-privacy linkage.

## Operator libraries

- **Alibaba Data-Juicer** — repo <https://github.com/modelscope/data-juicer>. Paper: Chen et al., *Data-Juicer: A One-Stop Data Processing System for Large Language Models*, SIGMOD 2024, <https://arxiv.org/abs/2309.02033>. Operator catalog is maintained in the repo README and the docs site.
- **NVIDIA NeMo Curator** — repo <https://github.com/NVIDIA/NeMo-Curator>. User guide: <https://docs.nvidia.com/nemo/curator/latest/>.
- **AI2 Dolma toolkit** — repo <https://github.com/allenai/dolma>. Covered by the Dolma paper above.
- **Ray Data** — documentation: <https://docs.ray.io/en/latest/data/data.html>.
- **Hugging Face `datatrove`** — repo <https://github.com/huggingface/datatrove>. The FineWeb orchestrator.
- **`datasketch`** — Python MinHash + LSH library: <https://github.com/ekzhu/datasketch>.

## Crawling-ethics contract (operational references)

- **RFC 9309** — Robots Exclusion Protocol. <https://www.rfc-editor.org/rfc/rfc9309>.
- **IETF AI Preferences Working Group (`aipref`)** — standards-track evolution of AI opt-out signalling. <https://datatracker.ietf.org/wg/aipref/about/>. <!-- needs-research: cite published RFCs and current drafts at read time. -->
- **`ai.txt`** — community-specified AI-crawler opt-out convention. Primary site: <https://site.spawning.ai/spawning-ai-txt>.
- **`llms.txt`** — community-specified LLM-facing content convention. <https://llmstxt.org/>.
- **OpenAI GPTBot documentation** — <https://platform.openai.com/docs/gptbot>. Also lists the current `User-Agent` string and example `robots.txt`.
- **Google-Extended documentation** — <https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers>. Google-Extended `User-Agent` is listed among the published crawlers.
- **Anthropic ClaudeBot** — <https://support.anthropic.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler>. <!-- needs-research: confirm current URL at read time. -->
- **Perplexity PerplexityBot** — <https://docs.perplexity.ai/guides/bots>. <!-- needs-research: confirm current URL at read time. -->
- **Apple Applebot-Extended** — <https://support.apple.com/en-us/119829>. <!-- needs-research: confirm current URL at read time. -->
- **Common Crawl CCBot and its policy** — <https://commoncrawl.org/ccbot>.
- **Cloudflare AI Audit / AI opt-out** — <https://blog.cloudflare.com/declaring-your-aindependence-block-ai-bots-scrapers-and-crawlers-with-a-single-click/>. <!-- needs-research: confirm current Cloudflare product and docs at read time. -->
- **UT1 URL blocklists** — <https://dsi.ut-capitole.fr/blacklists/index_en.php>. The widely-used adult-content and category-based URL lists.
- **`reppy`** — Python robots.txt parser: <https://github.com/seomoz/reppy>.
- **EU Directive (EU) 2019/790 Article 4 (TDM exception)** — EUR-Lex: <https://eur-lex.europa.eu/eli/dir/2019/790/oj>. The rightholder opt-out basis. Covered in depth in mod-102.
- **C2PA Content Credentials specification** — <https://c2pa.org/specifications/>. Primarily image-focused; text support is emerging.

## Hand-off: shard formats and training-pipeline substrate

- **WebDataset** — repo and docs: <https://github.com/webdataset/webdataset>. Aizman, Maltby, Breuel, *High Performance I/O for Large Scale Deep Learning*, arXiv:2001.01858. <https://arxiv.org/abs/2001.01858>.
- **MosaicML Streaming** — repo <https://github.com/mosaicml/streaming>. Docs: <https://docs.mosaicml.com/projects/streaming/en/stable/>.
- **Hugging Face `datasets`** — docs <https://huggingface.co/docs/datasets>. Parquet + Arrow baseline.
- **Apache Arrow** — <https://arrow.apache.org/>.

## Provenance and signing (previewing mod-109)

- **Sigstore** — <https://www.sigstore.dev/>.
- **SLSA framework** — <https://slsa.dev/spec/>.
- **CycloneDX ML-BOM specification** — <https://cyclonedx.org/>.
- **MLCommons Croissant metadata format** — <https://mlcommons.org/working-groups/data/croissant/>.
- **Hugging Face Dataset Card format** — <https://huggingface.co/docs/hub/datasets-cards>.

## Benchmark decontamination (previewing mod-106)

- **EleutherAI LM Evaluation Harness decontamination recipes** — <https://github.com/EleutherAI/lm-evaluation-harness>.
- **Shi, Xu, Wang, Choi, Zettlemoyer.** *Detecting Pretraining Data from Large Language Models.* ICLR 2024. <https://arxiv.org/abs/2310.16789>. The Min-K% Prob detector.
- **Oren, Meister, Chatterji, Ladhak, Hashimoto.** *Proving Test Set Contamination in Black Box Language Models.* ICLR 2024. <https://arxiv.org/abs/2310.17623>.

## Documentation templates (for the dataset card hand-off)

- **Gebru, Morgenstern, Vecchione, Vaughan, Wallach, Daumé, Crawford.** *Datasheets for Datasets.* CACM 2021 / arXiv:1803.09010. <https://arxiv.org/abs/1803.09010>.
- **Bender, Friedman.** *Data Statements for Natural Language Processing.* TACL 2018. <https://aclanthology.org/Q18-1041/>.
- **Holland, Hosny, Newman, Joseph, Chmielinski.** *The Dataset Nutrition Label.* arXiv:1805.03677. <https://arxiv.org/abs/1805.03677>.
- **Mitchell, Wu, Zaldivar, Barnes, Vasserman, Hutchinson, Spitzer, Raji, Gebru.** *Model Cards for Model Reporting.* FAT* 2019. <https://arxiv.org/abs/1810.03993>.

---

When citing any resource in a slice manifest, dataset card, or governance package, include the document title, publisher, version / date, and (where applicable) the git SHA or Hugging Face revision hash of the toolkit or dataset you actually used. If the resource is time-sensitive (standards drafts, dataset hosting, operator catalogs), add a `<!-- needs-research: ... -->` note so a future pass can verify the current state.
