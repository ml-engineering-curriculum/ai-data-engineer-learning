# Resources for mod-105 — Deduplication at Trillion-Token Scale

Primary sources first. Prefer the canonical publisher's copy over any summary. Versions, DOI stability, and dataset hosting change — re-verify at read time; `<!-- needs-research: ... -->` notes mark items that are particularly time-sensitive.

## Core references — the three claims of Chapter 01

- **Lee, Ippolito, Nystrom, Zhang, Eck, Callison-Burch, Carlini.** *Deduplicating Training Data Makes Language Models Better.* ACL 2022 (arXiv 2107.06499, 2021). <https://aclanthology.org/2022.acl-long.577/> — the perplexity + sample-efficiency case. Companion tooling: `ExactSubstr` and `NearDup` at <https://github.com/google-research/deduplicate-text-datasets>.
- **Kandpal, Wallace, Raffel.** *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022 (arXiv 2202.06539). <https://arxiv.org/abs/2202.06539> — the memorisation / extraction-attack case.
- **Carlini, Ippolito, Jagielski, Lee, Tramèr, Zhang.** *Quantifying Memorization Across Neural Language Models.* ICLR 2023 (arXiv 2202.07646). <https://arxiv.org/abs/2202.07646> — the memorisation-scales-with-frequency result that Chapter 01 and Chapter 08 §4 build on.
- **Soboleva, Al-Khateeb, Myers, Steeves, Hestness, Dey.** *SlimPajama: A 627B token cleaned and deduplicated version of RedPajama.* Cerebras / Together, 2023. <https://www.cerebras.net/blog/slimpajama-a-627b-token-cleaned-and-deduplicated-version-of-redpajama/>. Dataset: <https://huggingface.co/datasets/cerebras/SlimPajama-627B>. SlimPajama processing tools: <https://github.com/Cerebras/modelzoo/tree/main/modelzoo/transformers/data_processing/slimpajama>. <!-- needs-research: confirm current repository location — Cerebras modelzoo has been reorganised across releases. -->

## Near-duplicate algorithm references

- **Broder, A. Z.** *On the resemblance and containment of documents.* Proceedings of Compression and Complexity of Sequences, 1997. <https://ieeexplore.ieee.org/document/666900> — the MinHash primary source.
- **Charikar, M.** *Similarity Estimation Techniques from Rounding Algorithms.* STOC 2002. <https://dl.acm.org/doi/10.1145/509907.509965> — the SimHash primary source.
- **Manku, Jain, Sarma.** *Detecting Near-Duplicates for Web Crawling.* WWW 2007. <https://dl.acm.org/doi/10.1145/1242572.1242592> — the production application of SimHash at Google web scale.
- **Leskovec, Rajaraman, Ullman.** *Mining of Massive Datasets* (3rd edition). Cambridge University Press, 2020. <http://www.mmds.org/> — chapter 3 remains the clearest textbook treatment of MinHash + LSH and shingling.
- **Rajaraman, Ullman.** *Mining of Massive Datasets* lecture notes and slides (Stanford CS 246): <http://www.mmds.org/#book-contents>.
- **Indyk, Motwani.** *Approximate Nearest Neighbors: Towards Removing the Curse of Dimensionality.* STOC 1998. <https://dl.acm.org/doi/10.1145/276698.276876> — the LSH theoretical primary source.

## Fingerprint algorithms

- **xxHash** — <https://github.com/Cyan4973/xxHash>. 64-bit and 128-bit non-cryptographic hashes.
- **BLAKE3** — <https://github.com/BLAKE3-team/BLAKE3>. Cryptographic hash, 3–5× faster than SHA-256 on modern hardware.
- **MurmurHash3** — <https://github.com/aappleby/smhasher>.
- **NIST FIPS 180-4 (SHA-2 family)** — <https://doi.org/10.6028/NIST.FIPS.180-4>.

## Python libraries covered in this module

- **`datasketch`** — <https://github.com/ekzhu/datasketch>. Docs: <https://ekzhu.github.io/datasketch/>. The reference in-memory MinHash / LSH library.
- **`xxhash` (Python bindings)** — <https://github.com/ifduyue/python-xxhash>.
- **`pybloom-live`** — <https://github.com/joseph-fox/python-bloomfilter>. One of several maintained Python Bloom-filter packages; `pybloom3` and `pyroaring` are reasonable alternatives.
- **`pyroaring`** — <https://github.com/Ezibenroc/PyRoaringBitMap> — integer-set compression, faster than Python `set` for large integer domains.
- **`union-find`** — `scipy.sparse.csgraph.connected_components` is the usual production choice; there are many single-file union-find implementations.

## Production toolchains

- **AllenAI `dolma`** — <https://github.com/allenai/dolma>. Paper: Soldaini, Kinney, Bhagia et al., *Dolma: an Open Corpus of Three Trillion Tokens for Language Model Pretraining Research*, ACL 2024. <https://aclanthology.org/2024.acl-long.840/>. Dataset: <https://huggingface.co/datasets/allenai/dolma>.
- **NVIDIA `NeMo Curator`** — <https://github.com/NVIDIA/NeMo-Curator>. Docs: <https://docs.nvidia.com/nemo-framework/user-guide/latest/datacuration/index.html>.
- **Alibaba `data-juicer`** — <https://github.com/modelscope/data-juicer>. Paper: Chen, Dou, Li, Deng, Shen, et al., *Data-Juicer: A One-Stop Data Processing System for Large Language Models* (arXiv 2309.02033). <https://arxiv.org/abs/2309.02033>.
- **Hugging Face `datatrove`** — <https://github.com/huggingface/datatrove>. The pipeline behind FineWeb. Includes a MinHash + LSH operator and Spark-free parallel execution.

## Public pretraining corpora and their dedup documentation

- **The Pile** — Gao, Biderman, Black et al. 2020. <https://arxiv.org/abs/2101.00027>. <!-- needs-research: confirm current hosting state; portions have been removed from public distribution. -->
- **C4 / mC4** — Raffel et al. 2020 (T5). <https://arxiv.org/abs/1910.10683>. HF mirror: <https://huggingface.co/datasets/allenai/c4>.
- **RefinedWeb** — Penedo, Malartic, Hesslow et al. 2023. <https://arxiv.org/abs/2306.01116>. Dataset: <https://huggingface.co/datasets/tiiuae/falcon-refinedweb>.
- **Dolma** — Soldaini et al. 2024. <https://aclanthology.org/2024.acl-long.840/>.
- **RedPajama-Data v2** — Together AI 2023. <https://www.together.ai/blog/redpajama-data-v2>. Dataset: <https://huggingface.co/datasets/togethercomputer/RedPajama-Data-V2>.
- **FineWeb / FineWeb-Edu** — Penedo, Kydlíček, Allal et al. 2024. <https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1>. <https://huggingface.co/datasets/HuggingFaceFW/fineweb>.
- **SlimPajama-627B** — see above.

## Memorisation and extraction attacks

- **Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, et al.** *Extracting Training Data from Large Language Models.* USENIX Security 2021. <https://www.usenix.org/conference/usenixsecurity21/presentation/carlini-extracting>.
- **Nasr, Carlini, Hayase, Jagielski, Cooper, Ippolito, et al.** *Scalable Extraction of Training Data from (Production) Language Models.* arXiv 2311.17035. <https://arxiv.org/abs/2311.17035>.
- **Shokri, Stronati, Song, Shmatikov.** *Membership Inference Attacks Against Machine Learning Models.* IEEE S&P 2017. <https://ieeexplore.ieee.org/document/7958568> — foundational membership-inference paper.
- **Carlini, Chien, Nasr, Song, Terzis, Tramèr.** *Membership Inference Attacks From First Principles.* IEEE S&P 2022. <https://ieeexplore.ieee.org/document/9833649>.

## Data poisoning and adversarial perturbations

- **Carlini, Jagielski, Choquette-Choo, Paleka, Pearce, Anderson, Terzis, Thomas, Tramèr.** *Poisoning Web-Scale Training Datasets is Practical.* arXiv 2302.10149 / IEEE S&P 2024. <https://arxiv.org/abs/2302.10149> — the paper establishing that practical attacks against web-scraped corpora are economically feasible.
- **Shan, Ding, Passananti, Wu, Zheng, Zhao.** *Nightshade: Prompt-Specific Poisoning Attacks on Text-to-Image Generative Models.* IEEE S&P 2024 (arXiv 2310.13828). <https://arxiv.org/abs/2310.13828>. Project: <https://nightshade.cs.uchicago.edu/>.
- **Shan, Cryan, Wenger, Zheng, Hanocka, Zhao.** *Glaze: Protecting Artists from Style Mimicry by Text-to-Image Models.* USENIX Security 2023 (arXiv 2302.04222). <https://www.usenix.org/conference/usenixsecurity23/presentation/shan>. Project: <https://glaze.cs.uchicago.edu/>.
- **Shafahi, Huang, Najibi, Suciu, Studer, Dumitras, Goldstein.** *Poison Frogs! Targeted Clean-Label Poisoning Attacks on Neural Networks.* NeurIPS 2018. <https://arxiv.org/abs/1804.00792>.
- **Turner, Tsipras, Madry.** *Clean-Label Backdoor Attacks.* 2018. <https://people.csail.mit.edu/madry/lab/cleanlabel.pdf>.

## Threat vocabulary — ATLAS, OWASP, NIST

- **MITRE ATLAS** — <https://atlas.mitre.org/>. Techniques of interest in this module: AML.T0019 (Publish Poisoned Datasets), AML.T0020 (Poison Training Data), AML.T0018 (Backdoor ML Model), AML.T0051.001 (LLM Prompt Injection — Training Data Variant).
- **OWASP Machine Learning Security Top 10 (2023)** — <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02:2023 Data Poisoning Attack is the canonical entry.
- **OWASP Top 10 for LLM Applications** — <https://owasp.org/www-project-top-10-for-large-language-model-applications/>. Training Data Poisoning has shifted between LLM03 and LLM04 across editions; cite the current item at read time. <!-- needs-research: verify OWASP LLM Top 10 edition number and the training-data-poisoning item id at authoring time. -->
- **NIST AI RMF** — *Artificial Intelligence Risk Management Framework*. <https://www.nist.gov/itl/ai-risk-management-framework>. AI RMF Playbook: <https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook>.
- **NIST SP 800-218A** — *Secure Software Development Practices for Generative AI and Dual-Use Foundation Models.* <https://csrc.nist.gov/pubs/sp/800/218/a/final>. <!-- needs-research: track SP 800-218A revisions. -->
- **NIST AI 100-2 E2023** — *Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.* <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf>.

## ISO standards referenced

- **ISO/IEC 25012:2008** — *Data quality model.* <https://www.iso.org/standard/35736.html>.
- **ISO/IEC 5259-2:2024** — *AI — Data quality for analytics and machine learning — Part 2: Measures.* <https://www.iso.org/standard/81860.html>. The *similarity / duplication rate* measure family is the relevant 5259-2 entry for this module. <!-- needs-research: confirm 2024 publication state and current measure-name list in the released edition. -->
- **ISO/IEC 42001:2023** — *AI management system.* <https://www.iso.org/standard/81230.html>. Annex A.7 (data for AI) is the audit hook for the dedup report.
- **ISO/IEC 23894:2023** — *AI — Guidance on risk management.* <https://www.iso.org/standard/77304.html>.

## Signing and provenance (previewing mod-109)

- **Sigstore** — <https://www.sigstore.dev/>. `cosign` binary for signing blobs and attestations.
- **SLSA framework** — <https://slsa.dev/spec/>.
- **in-toto attestation framework** — <https://in-toto.io/>.
- **CycloneDX ML-BOM** — <https://cyclonedx.org/capabilities/mlbom/>.
- **MLCommons Croissant metadata** — <https://mlcommons.org/working-groups/data/croissant/>.

## Benchmark decontamination (previewing mod-106)

- **Brown, Mann, Ryder, Subbiah, Kaplan, et al.** *Language Models are Few-Shot Learners* (GPT-3), Appendix C on contamination. arXiv 2005.14165. <https://arxiv.org/abs/2005.14165>.
- **Shi, Xu, Wang, Choi, Zettlemoyer.** *Detecting Pretraining Data from Large Language Models* (Min-K% Prob). ICLR 2024. <https://arxiv.org/abs/2310.16789>.
- **Oren, Meister, Chatterji, Ladhak, Hashimoto.** *Proving Test Set Contamination in Black Box Language Models.* ICLR 2024. <https://arxiv.org/abs/2310.17623>.
- **EleutherAI LM Evaluation Harness decontamination recipes** — <https://github.com/EleutherAI/lm-evaluation-harness>.

## Textbooks and surveys

- **Leskovec, Rajaraman, Ullman.** *Mining of Massive Datasets* (3rd ed.). <http://www.mmds.org/> — the chapter 3 near-dup treatment is the textbook reference.
- **Gionis, Indyk, Motwani.** *Similarity Search in High Dimensions via Hashing.* VLDB 1999. <https://www.vldb.org/conf/1999/P49.pdf> — an early LSH construction.
- **Elmagarmid, Ipeirotis, Verykios.** *Duplicate Record Detection: A Survey.* IEEE TKDE 2007. <https://ieeexplore.ieee.org/document/4016534> — classical duplicate-detection survey predating the LLM era.

---

When citing any resource in a dedup report, dataset card, or governance package, include the document title, publisher, version / date, and (where applicable) the git SHA or Hugging Face revision hash of the toolkit or dataset you actually used. If the resource is time-sensitive (standards drafts, dataset hosting, OWASP edition numbering), add a `<!-- needs-research: ... -->` note so a future pass can verify the current state.
