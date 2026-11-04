# Chapter 03 — Main-Content Extraction: trafilatura vs resiliparse

## Why this chapter exists

The second-highest-leverage decision in a web-pretraining pipeline (after dedup) is the **extractor** — the operator that reads raw HTML and returns "the main text", stripping navigation, menus, ads, cookie banners, footer link farms, and inline JavaScript. Choose it poorly and the perplexity filter from Chapter 02 is scoring *boilerplate* as "quality text"; the tiered shards you emit from Chapter 02 are full of menu items; your downstream fine-tune model will learn to generate cookie banners when primed with the wrong context.

This chapter is the extractor chapter. It covers:

1. What the extraction problem actually is, and why "just strip HTML tags" doesn't work.
2. The two production-grade open extractors you will see in any modern pipeline: [**trafilatura**](https://trafilatura.readthedocs.io/) (Barbaresi 2021, ACL) and [**resiliparse**](https://resiliparse.chatnoir.eu/) (Bevendorff et al., ChatNoir project).
3. How C4 (Dodge et al. 2021) and RefinedWeb (Penedo et al. 2023) each handled extraction and what they measured.
4. How to actually benchmark the two on a slice you control — which is Exercise 02.

## The problem

Any typical web page is 10–30% main content and 70–90% chrome: masthead, navigation, sidebar, footer, inline recommendations, cookie banners, GDPR dialogs, newsletter opt-ins, author bio boxes, "related articles", comment sections of variable quality, JavaScript, inlined CSS. A naive `BeautifulSoup`-strip-the-tags extractor keeps all of that. Common Crawl's WET extractor (Apache Tika + Nutch) does slightly better but still passes through menu text and footer boilerplate at a rate that is measurable in downstream perplexity.

The extractor's job is **not** HTML cleanup. It is **main-content segmentation** — identifying the sub-tree(s) of the DOM that carry the author-written prose and returning only those. Historically this has been done with:

- **Boilerpipe-style heuristics** (Kohlschütter et al. 2010) — text-density features (ratio of anchor text to plain text) per DOM node, simple classifier on top.
- **Readability-style heuristics** — Mozilla's Readability.js / arc90's readability.js, focused on the single-article common case.
- **Visual-layout features** — Diffbot and similar commercial extractors render the DOM and reason about geometry.
- **Learned models** — a few research extractors that train a classifier over DOM features; not generally what you use at pretraining scale because inference cost is high.

`trafilatura` and `resiliparse` both fall in the "heuristic + light learned model" camp and are both designed to run at web-pretraining throughput — tens to hundreds of pages per second per core on commodity hardware.

## trafilatura

**Trafilatura** (Barbaresi, A. 2021, "Trafilatura: A Web Scraping Library and Command-Line Tool for Text Discovery and Retrieval", ACL System Demonstrations, <https://aclanthology.org/2021.acl-demo.15/>) is the extractor most commonly cited in open-pretraining corpus papers. Dolma, FineWeb, and RefinedWeb all use it either directly or as a benchmark reference.

What it does:

- Parses HTML with `lxml`, isolates the probable main-content sub-tree using a hand-crafted set of CSS-selector and text-density rules trained against a reference corpus.
- Returns plain text, Markdown, XML-TEI, or JSON-with-metadata.
- Extracts side-channel metadata (publication date, author, canonical URL, categories, tags) from microdata / `<meta>` tags / HTML heuristics.
- Includes built-in language detection, duplicate detection, and short-text filtering — several *operator responsibilities* folded into one library.

CLI and library pattern:

```python
import trafilatura

html = open("example.html", "rb").read()
text = trafilatura.extract(
    html,
    include_comments=False,
    include_tables=True,
    deduplicate=False,              # do dedup in your pipeline, not here
    favor_recall=False,             # tighter extraction = cleaner text
    output_format="txt",
)
```

Trafilatura's biases:

- Tuned on news-article-shaped pages. It is excellent on long-form prose and merely adequate on forums, product pages, and short pages.
- Default settings are **recall-biased** — i.e. favour keeping text over dropping it. For pretraining you often want `favor_precision=True`.
- Its own evaluation (Barbaresi 2021) and third-party benchmarks (ScrapingHub's `article-extraction-benchmark`, <https://github.com/scrapinghub/article-extraction-benchmark>) show it leading or tying the field on English news, with slightly less of a lead on non-English content.
- Python-only. Throughput is CPU-bound; on a single core expect ~50–200 pages/second depending on page complexity.

## resiliparse

**Resiliparse** (part of the ChatNoir research toolchain from Bauhaus-Universität Weimar; see <https://resiliparse.chatnoir.eu/> and the ChatNoir publications) is a Cython-based HTML processing library that includes a main-content extractor. It is designed for very high throughput and robustness on malformed HTML — the ChatNoir team crawls at IR-scale and resiliparse is their production extractor.

What it does differently:

- **C/Cython implementation with a Python API.** Significantly higher throughput than trafilatura on the same hardware — commonly 3–10× depending on page size and flags.
- Uses a **character- and layout-feature based** approach to boilerplate detection: text-density, link-density, tag-type features at the node level, with aggressive handling of pathological HTML.
- Deliberately minimalist on metadata extraction — it will not give you dates and authors the way trafilatura will. For pretraining you often do not care; for curated news corpora you might.
- Returns plain text with structural preservation (paragraph breaks) and optional formatting hints.

Pattern:

```python
from resiliparse.extract.html2text import extract_plain_text
from resiliparse.parse.html import HTMLTree

tree = HTMLTree.parse(html.decode("utf-8", errors="replace"))
text = extract_plain_text(
    tree,
    main_content=True,                  # ask the boilerplate remover to run
    alt_texts=False,
    preserve_formatting=False,
)
```

Resiliparse's biases:

- Optimised for throughput; the main-content classifier is simpler than trafilatura's and sometimes keeps more boilerplate on complex layouts.
- Excellent on malformed HTML — crashes less often on broken pages.
- Thinner metadata story — you will pair it with your own `<head>` scraping if you need canonical URLs or publication dates.

## What C4 did (Dodge et al. 2021)

The Colossal Clean Crawled Corpus (C4) was introduced with T5 (Raffel et al. 2020, <https://arxiv.org/abs/1910.10683>) and audited post-hoc by Dodge et al. in **"Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus"** (EMNLP 2021, <https://aclanthology.org/2021.emnlp-main.98/>). The pipeline:

- Starts from Common Crawl **WET** files — i.e. CC's own text extraction, not a re-run.
- Applies heuristic text filters: drop lines under 3 words, drop pages without a terminating-punctuation sentence, drop pages with less than 5 sentences, drop pages containing bad words from a published list (impacting demographic representation — Dodge measured this), drop pages matching a Javascript-notice template, drop pages with lorem ipsum, langdetect to English.
- No HTML-level extraction. No perplexity filter. No MinHash dedup — only whole-document exact dedup on three-sentence spans.

Dodge et al.'s audit matters for two reasons:

1. **They measured what the filters strip** — demographic composition shifts, documents from specific marginalised communities dropped at rates well above average. This is where "representativeness audit" (mod-104) enters the field as a load-bearing control.
2. **They showed that C4 inherits the WET extractor's boilerplate** — some C4 documents are literally cookie banners or 404 pages that passed the sentence-count filter. This was a strong practical argument for re-extracting from WARC.

## What RefinedWeb did (Penedo et al. 2023)

**RefinedWeb** (Penedo, Malartic, Hesslow, Cojocaru, Cappelli, Alobeidli, Pannier, Almazrouei, Launay, "The RefinedWeb Dataset for Falcon LLM: Outperforming Curated Corpora with Web Data, and Web Data Only", NeurIPS 2023, <https://arxiv.org/abs/2306.01116>) is the explicit answer to "can web-only curation match or beat curated corpora". The pipeline:

- Starts from Common Crawl **WARC**, not WET. Re-extracts with trafilatura.
- Aggressive URL-level filtering (adult-content lists, suspect-TLD heuristics).
- **Document-level** filters: language (`fasttext lid.176`, English), length, repetition (fraction of lines starting with the same characters), symbol-to-word ratio, "Internet marks" density (hashtags, URLs, email addresses).
- Two-stage dedup: exact SHA-1 after normalisation, then **MinHash near-duplicate** at document granularity with 9000 permutations in 450 buckets (b=450, r=20), Jaccard threshold roughly 0.8.
- No perplexity / KenLM filter. The RefinedWeb authors explicitly argue that heavy filtering hurts.

Key takeaways:

- **Re-extracting from WARC with trafilatura closed a measurable quality gap vs WET** — the authors report this in Section 3 of the paper.
- **Filter aggressiveness is a tunable, not a given.** RefinedWeb drops ~90% of input documents and still produces more training-useful tokens per unit of input than C4.
- **Extraction is upstream of every subsequent filter.** If trafilatura leaves 20% boilerplate, the next ten operators are each 20% less effective.

The combined C4+RefinedWeb lesson for your own pipeline: **don't trust WET for production**. Re-extract from WARC. Pick trafilatura or resiliparse deliberately. Measure.

## Benchmarking the two

The right way to compare extractors on your slice (which is Exercise 02):

1. **Sample 200–500 captures** from one WARC file. Sample across TLDs and content types using the index — don't accidentally benchmark on only news pages.
2. **Hand-label the main content** on each page, or use a reference extraction from Diffbot / Readability / a commercial service you trust. The labels are "what text should the extractor have returned". This is finicky work; ScrapingHub's `article-extraction-benchmark` publishes labelled HTMLs you can reuse.
3. **Run both extractors on each page**. Capture: output text, wall-clock, peak memory.
4. **Compute**:
   - **ROUGE-L against the reference text** — a coarse measure of recall.
   - **Character-level F1** against the reference text — stricter.
   - **Boilerplate ratio** — fraction of extracted text that consists of your reference's "boilerplate lines". You need this signal separately from F1 because an extractor that includes extra non-main-content paragraphs can still have high F1 if the main content is present.
   - **Throughput** — pages/second per core on representative hardware.
   - **Robustness** — fraction of pages where the extractor raised, returned empty, or timed out.

Expected result: on English news, trafilatura has the quality edge; resiliparse has a throughput edge of 3–10×. On non-news layouts (forums, product pages) the quality gap narrows; on malformed HTML resiliparse's robustness edge grows. The right *production* call is almost always a function of your slice composition and your compute budget — benchmark on *your* content, not on a generic corpus.

## Reasoning about the extractor's downstream effect

The extractor sits early in the pipeline. Its errors propagate:

- **Extra boilerplate** inflates document length, biases perplexity scoring (banner text has low perplexity), survives naive dedup (banners vary slightly per page), and poisons tokenisation statistics.
- **Missing content** removes signal; your effective slice is smaller than the shard count suggests.
- **Encoding mistakes** (mis-decoded charsets, mangled Unicode) propagate to every downstream operator and are almost impossible to repair later.

A reasonable **extractor QA checklist** for a slice:

- [ ] Tested on at least 5 TLDs / layouts representative of the slice.
- [ ] Reference labels or reference extractions available for ~500 pages.
- [ ] Numeric comparison (ROUGE-L, char-F1, boilerplate-ratio, throughput, robustness) logged per extractor version.
- [ ] Extractor version pinned in the slice manifest (`trafilatura==1.9.0` is not the same slice as `trafilatura==1.12.0`).
- [ ] Encoding and language-attribute handling verified on a non-English sub-sample.

This checklist feeds mod-104's data-quality audit framework directly.

## Where WET still earns its keep

Three legitimate uses of WET in a production pipeline:

1. **Fast iteration during pipeline development** — you don't want to re-extract from WARC every time you tweak a downstream filter. Run the pipeline on WET during development; swap to WARC + trafilatura or resiliparse for the production run.
2. **Baseline benchmarking** — "our extractor beats WET by X%" is a legible claim for a dataset card.
3. **Metadata-light tasks** — if you are building a language-id training corpus or a text-density baseline, WET is adequate.

Every other production use re-extracts from WARC.

## Summary

- Main-content extraction is a load-bearing operator; its errors propagate to every downstream filter.
- The two production-grade open extractors are **trafilatura** (Python, quality-biased, Barbaresi 2021) and **resiliparse** (Cython, throughput-biased, part of ChatNoir).
- **C4** (Dodge et al. 2021 audit) ran on WET and inherited its boilerplate; its filter set had measurable demographic effects.
- **RefinedWeb** (Penedo et al. 2023) re-extracted from WARC with trafilatura and ran aggressive document-level filters + MinHash dedup; Falcon-LLM was trained on it.
- Benchmark the two on **your slice** with labelled references; log ROUGE-L, char-F1, boilerplate ratio, throughput, and robustness per extractor version.
- WET is for iteration and baselines; WARC + a real extractor is for production.
- Pin the extractor version in the slice manifest alongside its parameters.

Chapter 04 steps up one level: given you can extract, filter, and dedup, what do the published reference pipelines actually chain, and how do their choices differ from the CCNet skeleton?
