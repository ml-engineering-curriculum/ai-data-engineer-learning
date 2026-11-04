# Chapter 04 — Public Reference Pipelines: Dolma, RedPajama-Data v2, FineWeb, The Pile

## Why this chapter exists

You will not design a Common Crawl curation pipeline from first principles. You will design one by looking at the four or five big public references, picking the one whose shape fits your constraints, and diverging from it with documented reasons. The field has settled on a short list of canonical pipelines; each publishes code, intermediate artifacts, and a paper describing choices and ablations. You should be able to read any of them and know within an hour which operators they chose, which thresholds they tuned, and what evidence they presented for those choices.

This chapter walks you through four:

1. **The Pile** (Gao et al. 2020) — the original large-scale English open corpus; a mixture-of-sources pipeline that framed most of the later work.
2. **Dolma** (Soldaini et al. 2024) — AI2's permissively-sourced open pretraining corpus and the Dolma toolkit; pragmatic, extensively documented, reproducible.
3. **RedPajama-Data v2** (Together AI 2023) — a Common-Crawl-only corpus that ships *every document* with its full set of per-document quality signals, letting downstream users apply their own filter thresholds.
4. **FineWeb / FineWeb-Edu** (Penedo et al. 2024) — the Hugging Face pipeline; introduces learned classifiers in place of KenLM perplexity for educational-content filtering.

Each one is a variation on the CCNet-style skeleton from Chapter 02. Reading them side by side builds the mental model fastest.

## The Pile (Gao et al. 2020)

**Reference**: Gao, Biderman, Black, Golding, Hoppe, Foster, Phang, He, Thite, Nabeshima, Presser, Leahy. *The Pile: An 800GB Dataset of Diverse Text for Language Modeling.* arXiv:2101.00027. <https://arxiv.org/abs/2101.00027>. Dataset originally hosted at <https://pile.eleuther.ai/> (hosting has rotated; verify live availability).

What it is:

- **~825 GB of English text across 22 sub-corpora**, each with its own licensing and sourcing story. Common Crawl is one input, not the dominant one. Others: PubMed Central, ArXiv, StackExchange, PhilPapers, GitHub, FreeLaw, OpenWebText2, Books3, DM Mathematics, YouTubeSubtitles, Wikipedia, HackerNews, NIH, USPTO, Enron, EuroParl, PubMed Abstracts, Ubuntu-IRC, OpenSubtitles, EuroParl, BookCorpus2.
- Each sub-corpus has its own ingest and cleaning code; the overall pipeline is **mixture-aware**: upsample high-quality low-volume sources (PubMed, ArXiv) and downsample Common Crawl relative to its raw size.
- Dedup is per-source, exact (SHA-256), no cross-source MinHash dedup.
- No learned quality filter; no perplexity filter. Each sub-corpus gets source-specific filtering.

What it taught the field:

- **Diversity-of-sources matters.** The ablation in Gao et al. §6 showed that mixing non-Common-Crawl sources improved downstream performance at fixed training tokens.
- **Per-source code is unavoidable.** Common Crawl, PubMed XML, and StackExchange dumps need wildly different ingesters.
- **Licensing bit back.** Books3 was pulled from distribution after copyright disputes (see <https://commoncrawl.org/blog/announcing-removal-of-books3> and reporting around DMCA takedowns — verify current licensing state). This is the foundational "web-scale corpus meets copyright" case study and is why mod-102 exists.

What it did not do:

- No cross-source MinHash dedup (not that era's standard).
- No benchmark decontamination (same).
- No per-document quality signals.

The Pile is a useful mental baseline. In 2026 you will not build a Pile-style corpus unless you specifically need a diverse multi-source English corpus and have the per-source budget; most teams start from a CC-centric corpus and layer additional sources on top (which is basically what Dolma does).

## Dolma (Soldaini et al. 2024)

**Reference**: Soldaini, Kinney, Bhagia, Schwenk, Atkinson, Authur, Bogin, Chandu, Dumas, Elazar, Hofmann, Jha, Kumar, Lucy, Lyu, Lambert, Magnusson, Morrison, Muennighoff, Naik, Nam, Peters, Ravichander, Richardson, Shen, Strubell, Subramani, Tafjord, Walsh, Zettlemoyer, Smith, Hajishirzi, Beltagy, Groeneveld, Dodge, Lo. *Dolma: An Open Corpus of Three Trillion Tokens for Language Model Pretraining Research.* ACL 2024. <https://aclanthology.org/2024.acl-long.840/>. Code: <https://github.com/allenai/dolma>. Dataset: <https://huggingface.co/datasets/allenai/dolma>.

What it is:

- **~3 trillion tokens** across six sub-corpora: Common Crawl, The Stack (code), C4, Reddit, PeS2o (scientific papers), Wikipedia & Wikibooks, Project Gutenberg.
- A pipeline **shipped as a toolkit** — [`dolma`](https://github.com/allenai/dolma) is a reusable Rust-+-Python library implementing the operators AI2 ran, with the configs they ran them with.
- Per-source filtering + cross-source MinHash dedup (document-level, 128 permutations, 9 bands / 13 rows) + benchmark decontamination.
- Explicit **taggers** (operators that annotate documents with quality signals) and **filters** (operators that drop documents based on taggers). This separation is the single most durable design idea in the paper: taggers run once, filters can be re-run with different thresholds.

What makes it a good model to study:

1. **Reproducibility is central.** The paper, the dataset, the toolkit, and the configs are all published. You can download the raw Common Crawl slices Dolma used, run `dolma` with their configs, and get the same output.
2. **Tagger/filter separation.** This is the architectural pattern you should internalise — don't bake quality thresholds into the operator that measures the signal. Measure the signal, store it, and decide thresholds downstream. RedPajama-Data v2 (below) takes this to the extreme.
3. **Dedup + decontamination are first-class.** MinHash dedup is a named stage with its own config; benchmark decontamination against ~20 benchmarks is a named stage with its own config. The ad hoc "we dropped matches of some benchmarks" of earlier corpora is replaced by an explicit, verifiable step.
4. **The data-sheet story is tight.** Dolma ships with a Datasheet for Datasets and a per-source documentation page. For a licensure/governance audience this is the exemplar of "what to hand them".

Dolma's specific operator chain, roughly:

```
Per source (CC / Stack / C4 / Reddit / PeS2o / Wiki / Gutenberg):
  acquire → URL / host filter → content extract (trafilatura on CC) → language ID (cld3 / lid)
  → heuristic quality taggers (length, repetition, stopword ratio, ...) → filter by taggers
  → NSFW tagger → PII tagger
Cross-source:
  → MinHash dedup (128 perms, 9 bands × 13 rows)
  → benchmark decontamination (n-gram match, ~20 benchmarks)
  → emit shards + manifest + dataset card
```

What you should cite Dolma for in your own pipeline: tagger/filter separation, MinHash parameter choice, decontamination benchmark list, documentation shape.

## RedPajama-Data v2 (Together AI 2023)

**Reference**: Together AI blog, "RedPajama-Data-v2: An open dataset with 30 trillion tokens for training large language models." <https://www.together.ai/blog/redpajama-data-v2>. Code: <https://github.com/togethercomputer/RedPajama-Data>. Dataset: <https://huggingface.co/datasets/togethercomputer/RedPajama-Data-V2>.

What it is:

- **~30 trillion tokens** of Common Crawl text across five languages (English, German, French, Spanish, Italian) from 84 snapshots.
- Every document ships with **~40 pre-computed quality signals** — length statistics, language-id scores, repetition statistics, perplexity, domain features, dedup near-duplicate cluster IDs, and benchmark-contamination flags.
- The dataset is **unfiltered**. Users apply their own filter thresholds against the quality signals at consumption time, producing their own subset.

What makes it a design standout:

- **The quality signal is the deliverable.** Everyone else ships a filtered corpus; RedPajama v2 ships every document plus the evidence you would need to filter it yourself. This is a radically different packaging model and it is extraordinarily useful for research — you can ablate thresholds without re-running the expensive signal computation.
- **It operationalises Dolma's tagger/filter separation** to its logical conclusion: taggers are fully published, filters are not applied.
- **It is CC-only.** No StackExchange, no Wikipedia, no code. If your corpus needs those you layer them on top.

Specific signals published per document (non-exhaustive):

- `ccnet_language_score`, `ccnet_perplexity`, `ccnet_bucket` (head/middle/tail)
- `rps_doc_frac_lines_end_with_ellipsis`, `rps_doc_frac_lines_ending_with_terminal_punctuation_mark`
- `rps_doc_mean_word_length`, `rps_doc_num_sentences`, `rps_doc_frac_no_alph_words`
- `rps_doc_frac_unique_words`, `rps_doc_frac_repeat_lines`
- `minhash_signature_0.7`, `minhash_signature_0.8`, `minhash_signature_0.9` (at multiple Jaccard thresholds)
- Benchmark contamination flags per benchmark

The exhaustive list is in the dataset card. The point is that **40 signals × 30 trillion tokens** is a lot of information; the packaging choice is defensible because the signals are cheaper to compute once than to re-run when a user wants to tune thresholds.

What you should cite RedPajama-Data v2 for: the quality-signal-per-document packaging model, the specific signal list as a starter set for your own tagger inventory, the multi-threshold MinHash signature as a way to defer the final dedup decision to the consumer.

## FineWeb and FineWeb-Edu (Penedo et al. 2024)

**Reference**: Penedo, Kydlíček, allal, Lozhkov, Mitchell, Raffel, Von Werra, Wolf. *FineWeb: decanting the web for the finest text data at scale.* 2024. <https://huggingface.co/datasets/HuggingFaceFW/fineweb>. Technical blog: <https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1>. Code: <https://github.com/huggingface/datatrove>.

What it is:

- **FineWeb**: ~15 trillion tokens of English CC text, every 2013–2024 snapshot processed with the `datatrove` pipeline.
- **FineWeb-Edu**: a filtered subset of FineWeb, ~1.3 trillion tokens, retained by a **learned educational-value classifier** trained on LLM-scored examples.
- Processing pipeline is orchestrated by [`datatrove`](https://github.com/huggingface/datatrove), a Hugging Face open-source library for running curation at CC scale on Slurm or Ray.

What is new:

1. **URL filter is first-class.** FineWeb's URL-level filtering is more aggressive than earlier pipelines — large allow/deny lists derived from UT1 lists plus heuristics on registered domain reputation.
2. **MinHash dedup *between* CC snapshots**, not just within. Earlier pipelines deduped within a snapshot; FineWeb dedups across all historical snapshots, which removes a large amount of "same page recrawled" content.
3. **Learned filter replaces perplexity.** FineWeb-Edu uses a Snowflake Arctic-Embed-m-based classifier trained on LLM-rated "educational value" labels. This is the direct descendant of the KenLM filter and a documented step away from it; the ablation in the technical blog shows FineWeb-Edu outperforming a KenLM-tiered FineWeb at matched training-compute on downstream benchmarks.
4. **Ablation-first methodology.** Every filter in FineWeb was gated on an ablation — train a small model on data with the filter, compare on fixed benchmarks, decide. This is the discipline from the DataComp-LM paper (Li et al. 2024, <https://arxiv.org/abs/2406.11794>) applied at pretraining scale.

What you should cite FineWeb for: the datatrove orchestrator, the cross-snapshot MinHash dedup, the learned-classifier filter in place of KenLM, the ablation-first methodology.

## Side-by-side

| | The Pile | Dolma | RedPajama-Data v2 | FineWeb / FineWeb-Edu |
|---|---|---|---|---|
| Year | 2020 | 2024 | 2023 | 2024 |
| Source scope | 22 sub-corpora | 6 sub-corpora | CC-only (5 languages) | CC-only (English) |
| HTML extractor | per-source | trafilatura on CC | ccnet (WET-based) + custom | custom datatrove extractor |
| Language ID | langdetect / per-source | cld3 | fasttext lid.176 | fasttext lid.176, retrained |
| Quality filter | heuristics per source | tagger/filter separation | all signals exposed, no filter applied | heuristics + learned FineWeb-Edu classifier |
| Perplexity | none | none | KenLM, exposed as signal | replaced by learned classifier |
| Dedup | per-source exact | MinHash 128 perms, 9×13, in-source then cross | MinHash signatures at 0.7/0.8/0.9, consumer applies | MinHash across all snapshots |
| Decontamination | none | n-gram match ~20 benchmarks | published as signal | ablation-driven |
| Toolkit | per-source code | `dolma` (Rust+Py) toolkit | `RedPajama-Data` scripts | `datatrove` orchestrator |
| Who runs it in prod | historical reference | AI2, OLMo | research consumers | HF, downstream fine-tunes |
| Primary citation | Gao 2020 | Soldaini 2024 | Together 2023 blog | Penedo 2024 blog/dataset |

## What to actually reproduce

Exercise 03 asks you to reproduce a **scoped subset** of one of these. Not the whole thing — that is nation-state compute budgets. The reproduction target is:

- Pick one slice of ~100 GB raw WARC input (two or three WARC files from one CC snapshot).
- Pick one of Dolma or RedPajama-Data v2. (FineWeb is a stretch option; The Pile is not recommended because its Common-Crawl portion is a small part of what makes it interesting.)
- Run the full published operator sequence on that slice, with the published parameters, using the published toolkit.
- Produce a **slice manifest** and a **comparison report**: your slice's output statistics (doc count, token count, duplicate fraction, language distribution) vs. the published reference's.

The goal is not to beat the reference. The goal is that when you later work on a production pretraining corpus you have personally run the canonical sequence with the canonical parameters and you know the quirks (where trafilatura eats memory on complex HTML, where MinHash takes longer than expected, which CC snapshots are smaller than you'd think).

## Which pipeline should you adopt in your own work?

A pragmatic decision table:

| If you want... | Start from... | Why |
|---|---|---|
| A reproducible CC-based English pretraining corpus with minimal custom code | Dolma + Dolma toolkit | Most operator-complete, best docs, written to be reused |
| To let downstream users pick their own filter thresholds | RedPajama-Data v2 approach | Signal-per-document, consumer-applies-filters |
| A high-quality, educational-value-biased slice | FineWeb-Edu + datatrove | Learned classifier is the current state of the art for this |
| A diverse multi-source English corpus | The Pile sources list + a modern CC pipeline on top | You still want modern CC curation; Pile's value is the non-CC sources list |
| A multilingual corpus | RedPajama-Data v2 + language-specific perplexity models | v2 ships 5 languages with signals; extendable |

In each case cite the reference and record your divergences in the slice manifest.

## Summary

- The four public reference pipelines — **The Pile**, **Dolma**, **RedPajama-Data v2**, **FineWeb / FineWeb-Edu** — share a common skeleton (acquire → extract → language-id → quality filter → dedup → decontaminate) but diverge on which operators are first-class, which parameters they used, and how the output is packaged.
- **Dolma** is the most pragmatic to study: toolkit + configs + paper + dataset card, with a clean tagger/filter separation.
- **RedPajama-Data v2** operationalises tagger/filter separation to the extreme — ship every document with all signals, defer filtering to the consumer.
- **FineWeb-Edu** demonstrates replacing KenLM perplexity with a learned classifier and running the pipeline with datatrove-driven ablations.
- **The Pile** remains the reference for *diverse, multi-source* corpus construction and is the foundational case study for licensing issues in training corpora.
- Exercise 03 reproduces a scoped subset of Dolma or RedPajama-Data v2 on a ~100 GB slice, producing a manifest and a comparison report.

Chapter 05 steps out of the canonical pipelines and into the operator libraries that make running them tractable — Data-Juicer, NeMo Curator, the Dolma toolkit itself, and Ray Data.
