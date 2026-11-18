# Quiz 01 — Web-Scale Acquisition and Curation

**Module:** [mod-103 — Web-Scale Data Acquisition and Curation Pipelines](../README.md)
**Format:** 12 multiple-choice questions, one correct answer per question.
**Suggested time:** 25 minutes.
**Coverage:** Chapters 01–07.

These questions test whether you can *use* the module's operators — pick the right Common Crawl file family for a job, reason about why CCNet dedups paragraphs within a shard instead of documents across the snapshot, read a reference pipeline's design choices, pick an operator library for a slice, place the ethics check at the correct stage, and hand off cleanly to the training team — not whether you can recite the chapter wording. Answer key with references is at the bottom.

---

## Q1. WARC vs WAT vs WET for one slice

You are building a pretraining slice and need three things on the same captures: (a) the raw HTML so you can run your own extractor, (b) the outbound-link graph for a URL-reputation filter, (c) a quick plain-text baseline you can diff against your extractor's output. According to Chapter 01, how should the three file families map to these three needs?

- **A.** Use WARC for all three — it is the superset format.
- **B.** Use WET for all three — it is the fastest to iterate on.
- **C.** Use **WARC** to run your own extractor, **WAT** for the link graph without parsing HTML, and **WET** as the plain-text baseline; join across the three with `WARC-Target-URI`.
- **D.** Use the columnar index for (a) and (b); use WAT for (c).

---

## Q2. What "pulling a slice" actually means

Chapter 01 defines a Common Crawl **slice** as a reproducible artifact. A reviewer asks what the slice *is*. Which answer matches the chapter?

- **A.** A directory of extracted text files on your local disk, one per URL.
- **B.** A `(source snapshot id, selection predicate, per-record manifest with observed_sha256, extraction config)` quadruple — anyone with S3 read access can regenerate the text from the manifest; the manifest is the primary deliverable.
- **C.** The full set of WARC files for the chosen snapshot, redistributed under your own bucket.
- **D.** The DuckDB query alone — the per-record manifest is optional because the query is deterministic.

---

## Q3. Why CCNet dedups paragraphs within a shard, not documents globally

Chapter 02 explains that CCNet's first operator is paragraph-level SHA-1 dedup **within a shard**, not a global document-level dedup. Which statement matches the chapter's reasoning?

- **A.** Within-shard paragraph dedup is a cheap first pass that bounds memory to one shard's hash set, catches intra-shard template repetition (banners, boilerplate) before the expensive LM scoring runs, and is explicitly a complement to — not a replacement for — the global document-level MinHash stage that belongs to mod-105.
- **B.** Global document MinHash was already run upstream in Common Crawl's WARC pipeline, so CCNet skips it.
- **C.** Paragraph-level hashing is cryptographically safer than document-level hashing, and that is the reason CCNet picked it.
- **D.** Running dedup globally would drop too many documents and reduce the slice; within-shard is a quality decision, not a memory decision.

---

## Q4. KenLM perplexity tiering — percentile vs fixed thresholds

Chapter 02 is explicit about how to turn KenLM perplexity scores into head/middle/tail tiers. What does CCNet actually do?

- **A.** Head is perplexity < 100, middle is 100–500, tail is > 500 — fixed thresholds reused across every snapshot.
- **B.** Head and tail are each the top 1% of scores; everything else is middle.
- **C.** The three tiers are **percentile cuts on the slice itself** (e.g. 33/67). Fixed perplexity thresholds travel badly across snapshots because the distribution shifts; percentile cuts are robust. Downstream recipes decide whether to oversample head, drop tail, or mix.
- **D.** CCNet drops the tail entirely before emitting shards.

---

## Q5. The four documented omissions of CCNet

Chapter 02 names four things CCNet's published pipeline does **not** do that every modern pipeline bolts on. Which option lists those four omissions correctly?

- **A.** No URL-level filtering, no HTML-level filtering, no benchmark-contamination defence, no PII scrubbing.
- **B.** No language identification, no perplexity filter, no dedup, no documentation.
- **C.** No trafilatura extraction, no Ray execution, no sharding, no signing.
- **D.** No WAT support, no S3 reads, no reproducible manifests, no Hugging Face hub integration.

---

## Q6. Picking trafilatura vs resiliparse on a specific slice

Chapter 03 frames the extractor choice as evidence-driven, not opinion-driven. You are building an English-only news-heavy slice on a CPU-only workstation and quality matters more than throughput. Which choice matches the chapter's biases?

- **A.** Resiliparse — its Cython implementation always wins at CPU-only workloads.
- **B.** Trafilatura — Python, tuned on news-article-shaped pages, quality-biased defaults (and you should flip `favor_precision=True` for a pretraining slice); you accept 3–10× lower per-core throughput in return.
- **C.** Common Crawl WET — it is already extracted, so no runtime cost.
- **D.** Whichever is Apache-2.0, independent of quality.

---

## Q7. The joint C4-plus-RefinedWeb lesson about WET

Chapter 03 summarises the combined lesson from the Dodge et al. 2021 C4 audit and Penedo et al. 2023 RefinedWeb. Which statement captures it?

- **A.** WET is production-grade; both papers endorsed WET as a baseline worth shipping.
- **B.** Don't trust WET for production — C4 inherited the WET extractor's boilerplate (some C4 documents are literally cookie banners or 404s that passed the sentence-count filter) and RefinedWeb closed a measurable quality gap by re-extracting from WARC with trafilatura. Re-extract from WARC, pick an extractor deliberately, and measure.
- **C.** Perplexity filtering is unnecessary once you have RefinedWeb-style URL filtering; drop it.
- **D.** Both pipelines prove that langdetect (not `fasttext lid.176`) is the correct language-id choice.

---

## Q8. Tagger / filter separation — Dolma to RedPajama-v2

Chapter 04 identifies a single architectural idea Dolma crystallises and RedPajama-Data v2 takes further. Which option names it correctly?

- **A.** **Tagger / filter separation** — don't bake quality thresholds into the operator that measures the signal; measure the signal, store it as a per-document attribute, and decide thresholds downstream. Dolma enforces this via its side-car attribute files; RedPajama-Data v2 takes it to the extreme by publishing every document plus ~40 quality signals and applying no filter at all, letting consumers choose.
- **B.** **Cross-snapshot MinHash** — FineWeb-style dedup that operates between snapshots, not within.
- **C.** **Ablation-first methodology** — gating every filter decision on a DataComp-LM-style training ablation.
- **D.** **Mixture-aware upsampling** — reweighting sub-corpora at emit time.

---

## Q9. What FineWeb-Edu replaced and why

Chapter 04 describes FineWeb-Edu's most-cited departure from the CCNet skeleton. Which option matches the chapter?

- **A.** FineWeb-Edu replaced FastText `lid.176` with `glotlid`.
- **B.** FineWeb-Edu replaced MinHash with exact SHA-256 dedup to reduce false positives.
- **C.** FineWeb-Edu replaced the KenLM perplexity filter with a **learned educational-value classifier** (Snowflake Arctic-Embed-m based, trained on LLM-rated labels). The technical blog's ablation shows FineWeb-Edu outperforming a KenLM-tiered FineWeb at matched training-compute on downstream benchmarks.
- **D.** FineWeb-Edu dropped all CC data and used only Wikipedia and ArXiv.

---

## Q10. Picking an operator library for a specific pipeline

You need to run a CCNet-style chain on one node but your single hot operator is MinHash across tens of millions of documents, and the workstation has four A100 GPUs sitting idle. Chapter 05's routing table says to reach for which library *for that bottleneck*?

- **A.** Alibaba Data-Juicer — the 80+ operator catalog gives the most breadth.
- **B.** The Dolma toolkit — the Rust core is fastest on any CPU workload.
- **C.** NVIDIA NeMo Curator — GPU-accelerated cuDF-backed MinHash is typically 10–30× faster per dollar on this workload; its depth-per-operator on dedup is the differentiator even when the rest of your stack stays CPU-first.
- **D.** Ray Data alone — it supplies the runtime and the operators.

---

## Q11. Why the opt-out check belongs at acquire, not publish

Chapter 06 is the "three-moment principle" chapter. The product manager proposes checking AI-use opt-outs only at the final emit stage to save fetches. Which reply matches the chapter?

- **A.** Acceptable — if the slice never leaves your organisation, acquire-time checking is unnecessary.
- **B.** Wrong — the contract must be enforced at acquire, not publish. You will still have *processed* the opted-out content through every intermediate operator (language ID, perplexity, dedup, decontamination), the upstream statistics (dedup clusters, quality-tier distributions) are polluted by the opted-out population, and the audit trail reads "we acquired it and then dropped it later" (an admission) instead of "we never acquired the document" (a record). Legal exposure under regimes like EU DSM Article 4 and GDPR Article 6 attaches to processing, not just publication.
- **C.** Wrong, but only because fetching robots.txt at publish time is slower than at acquire time.
- **D.** Correct — Common Crawl's CCBot already handled robots at crawl time, so no acquire-time check is needed for CC-derived slices.

---

## Q12. The hand-off bundle to `training-pipeline-engineer`

Chapter 07 says the hand-off to the level-35 training-pipeline role is *three* artifacts that must ship together. Which option names them and the non-negotiable property?

- **A.** Shards, a `requirements.txt`, and a Slack channel — the training team re-runs your pipeline.
- **B.** **Shards + slice manifest + dataset card** — all three ship together; losing any one breaks the hand-off. The manifest names every operator with version and config, acquire-stage evidence, output shard layout, Sigstore signature, and SLSA attestation; shard discipline is 1–4 GB per shard, shard count ≥ 100× data-parallel world size, uniform sample count within ~5%.
- **C.** A WebDataset tar, a ZIP of the trafilatura cache, and a Jupyter notebook.
- **D.** Just the shards — the manifest and the dataset card are governance artifacts, not engineering artifacts.

---

## Answer key

**Q1 — C.** Chapter 01 §"WARC vs WAT vs WET — the operational trade-off" says exactly this: "**WAT for selection, WARC for extraction, WET for a baseline you benchmark against.**" The chapter also notes `WARC-Target-URI` as the join key across all three file families. A and B over-commit to one format; D confuses the index (metadata about where records live on disk) with WAT (per-record HTTP/HTML metadata).

**Q2 — B.** Chapter 01 §"The slice — what 'pulling a slice' actually means" is explicit: a slice is the source snapshot ID + a boolean selection predicate (saved as SQL/JSON) + the per-record manifest with `observed_sha256` (your-computed body hash, not something CC publishes) + the extraction / language-id / dedup config applied downstream. "A slice is not 'a directory of text files'. … Treat the manifest as the primary deliverable." A, C, D each miss the manifest-as-deliverable framing.

**Q3 — A.** Chapter 02 §"Operator 1: shard and hash-dedup paragraphs" names the two reasons directly: within-shard dedup bounds the `seen` set to one shard's memory footprint ("CCNet's 5 GB shards hold on the order of one million paragraphs — enough that intra-shard template repetition is caught, small enough that the `seen` set fits comfortably in RAM"), and the per-shard pass is deliberately an inexpensive precursor to the global document-level MinHash, which is the subject of mod-105. B, C, D are each a plausible-sounding non-reason.

**Q4 — C.** Chapter 02 §"Operator 3: KenLM perplexity scoring" says exactly: "**Head / middle / tail bin boundaries are percentile cuts on the slice itself**, not fixed perplexity thresholds. Fixed thresholds travel badly across snapshots; percentiles are robust." The chapter also notes CCNet's choice is to **keep all three tiers but tag them** — downstream recipes decide. A and B pick fixed thresholds; D is a downstream re-implementation choice, not CCNet's.

**Q5 — A.** Chapter 02 §"What CCNet does not do" lists these four omissions in order: no URL-level filtering, no HTML-level filtering, no benchmark-contamination defence, no PII scrubbing. The chapter goes on to say that those four omissions are "the main reason that 'CCNet-style' is a good skeleton, not a finished recipe". B, C, D each name things CCNet *does* do, or confuse the skeleton with implementation details.

**Q6 — B.** Chapter 03 §"trafilatura" says trafilatura is "tuned on news-article-shaped pages. It is excellent on long-form prose" and that "default settings are **recall-biased** — i.e. favour keeping text over dropping it. For pretraining you often want `favor_precision=True`." Throughput is CPU-bound at ~50–200 pages/second per core; resiliparse's 3–10× throughput edge is called out separately. A overstates resiliparse; C is the WET anti-pattern Chapter 03 argues against for production; D is Chapter 03's own caveat near the end but does not drive the quality decision.

**Q7 — B.** Chapter 03 §"What C4 did" and §"What RefinedWeb did" together: C4 "inherits the WET extractor's boilerplate — some C4 documents are literally cookie banners or 404 pages that passed the sentence-count filter"; RefinedWeb "re-extracting from WARC with trafilatura closed a measurable quality gap vs WET". The combined lesson paragraph says: "**don't trust WET for production**. Re-extract from WARC. Pick trafilatura or resiliparse deliberately. Measure."

**Q8 — A.** Chapter 04 §"Dolma" item 2 names **tagger/filter separation** as "the single most durable design idea in the paper: taggers run once, filters can be re-run with different thresholds". Chapter 04 §"RedPajama-Data v2" says RedPajama v2 "operationalises Dolma's tagger/filter separation to its logical conclusion: taggers are fully published, filters are not applied". B, C, D are each real ideas from Chapter 04 but belong to FineWeb (B, C) or The Pile (D).

**Q9 — C.** Chapter 04 §"FineWeb and FineWeb-Edu" new item 3: "**Learned filter replaces perplexity.** FineWeb-Edu uses a Snowflake Arctic-Embed-m-based classifier trained on LLM-rated 'educational value' labels. This is the direct descendant of the KenLM filter and a documented step away from it; the ablation in the technical blog shows FineWeb-Edu outperforming a KenLM-tiered FineWeb at matched training-compute on downstream benchmarks." A, B, D are plausible-sounding but not what the chapter credits FineWeb-Edu with.

**Q10 — C.** Chapter 05 §"How to pick" routing table: "Your bottleneck is MinHash and you have GPUs → NeMo Curator." The §"NVIDIA NeMo Curator" section says GPU-side MinHash is "the operator that makes NeMo Curator worth adopting even if the rest of your stack is CPU-first", with 10–30× speed-ups from cuDF-backed MinHash. A has the breadth story but not the MinHash depth; B is a CPU-first toolkit that does not get you GPU MinHash; D leaves you writing the MinHash yourself.

**Q11 — B.** Chapter 06 §"Why 'check at publish' fails" names all four failures verbatim: you still processed the opted-out content and can't un-process it; upstream statistics (dedup clusters, quality tiers, language histograms) are polluted; legal exposure under EU DSM Article 4 reservations and GDPR Article 6 attaches to processing not publication; the audit trail reads "we acquired it and then dropped it later" (an admission) instead of "we never acquired the document" (a record). Chapter 06 §"The 'my slice is CC, CC already handled robots' fallacy" rebuts D specifically — CCBot handled historical crawl-time robots, but AI-use overlays like GPTBot and Google-Extended largely did not exist pre-2023 and must be re-checked at acquire time.

**Q12 — B.** Chapter 07 §"What you hand off" says exactly: "Three artifacts per slice: **A set of shards** … **A slice manifest** … **A dataset card** … All three ship together; losing any one of them breaks the hand-off." The manifest schema in §"The slice manifest schema" lists operators with version + config + docs in/out, acquire-stage evidence, output shard layout, Sigstore signature bundle, and SLSA attestation path. Shard discipline from §"Shard-size and shard-count discipline" is 1–4 GB each, ≥100× data-parallel world size in count, uniform sample count within ~5%.
