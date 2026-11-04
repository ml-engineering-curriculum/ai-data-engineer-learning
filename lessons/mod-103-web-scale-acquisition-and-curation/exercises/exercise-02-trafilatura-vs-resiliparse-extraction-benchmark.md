# Exercise 02 — trafilatura vs resiliparse Extraction Benchmark

**Estimated effort:** 2 hours
**Chapter dependency:** [Chapter 03 — Main-content extraction: trafilatura vs resiliparse](../03-boilerplate-extraction-trafilatura-and-resiliparse.md)

## Objective

Benchmark **trafilatura** and **resiliparse** on a slice of HTML pages you control, producing a numeric comparison and a recommendation for a specific downstream pipeline. The goal is to replace "trafilatura is the Python one, resiliparse is the Cython one" with concrete numbers you have measured on your content.

By the end you will have:
- A labelled reference set of ~200 HTML pages with expected main-content text.
- A per-page comparison of the two extractors on ROUGE-L, character-level F1, boilerplate ratio, throughput, and robustness.
- A short decision memo recommending one extractor (or a hybrid) for a stated downstream use case.

## Prerequisites

- You have read Chapter 03 and the trafilatura (Barbaresi 2021, <https://aclanthology.org/2021.acl-demo.15/>) and resiliparse (ChatNoir documentation, <https://resiliparse.chatnoir.eu/>) references.
- You have Python 3.11+ with `trafilatura`, `resiliparse`, `rouge-score` (or an equivalent), `beautifulsoup4`, and `warcio` installed.
- You have completed Exercise 01, or you have another source of ~500 captured HTML pages representative of a slice you care about.
- Optional: access to ScrapingHub's `article-extraction-benchmark` labelled corpus (<https://github.com/scrapinghub/article-extraction-benchmark>) as an additional reference set.

## Problem statement

You are preparing a 500-TB English-web pretraining corpus from Common Crawl. The pipeline will run on a 128-core / 512 GB node with no GPUs. Your job is to decide whether to use trafilatura or resiliparse as the main-content extractor. The decision will stay in effect for at least six months.

You must produce **evidence**, not opinion. The extractor you pick will be pinned in the slice manifest and auditable.

## Starter guidance

1. **Build the reference set.** Three sources:
   - 100 pages from Exercise 01's slice, sampled across TLDs (`.com`, `.net`, `.org`, `.edu`, `.gov`, at minimum).
   - 50 pages from a public labelled corpus (ScrapingHub's benchmark, or a subset of CleanEval <http://cleaneval.sigwac.org.uk/> if still available).
   - 50 pages you hand-pick to stress the extractors: forum threads, product pages, long-scroll news, paywalled pages, malformed HTML, non-English. These are the *interesting* pages that tell you where the extractors fail.

2. **Produce reference extractions for pages without labels.** For the unlabelled 100 from your slice, you need to decide what "main content" is. Options: run a commercial extractor (Diffbot, Readability.js via a headless browser) and treat its output as reference; hand-label 10% of them and treat only those as authoritative; use crowd-sourced labels. Document the reference methodology — the whole benchmark rests on it.

3. **Run both extractors on every page.** For each page record:
   - Extractor output text.
   - Wall-clock extraction time.
   - Peak RSS during extraction (use `resource.getrusage` or equivalent).
   - Did it raise? Did it return empty? Did it return a short string that is clearly a fragment?

4. **Compute metrics per page**, then aggregate:
   - **ROUGE-L** against reference text (recall-oriented).
   - **Character-level F1** against reference (strict).
   - **Boilerplate ratio**: the Jaccard distance between the extractor's output lines and a hand-curated list of boilerplate lines from the reference page's HTML (navigation menu, cookie banners, etc.). Lower is better.
   - **Throughput**: pages/second per core, averaged across all pages, and separately for the longest 10% of pages.
   - **Robustness**: fraction of pages where the extractor returned non-empty, non-fragmentary text.

5. **Break down by page type.** Report metrics separately for: news/article, forum, product, documentation, malformed. The aggregate tells you one number; the breakdown tells you where each extractor wins.

6. **Write the decision memo.** One page, referring to your numbers. Include a recommendation (one extractor, or a hybrid where you route by heuristic), the *reason* in terms of specific numbers, and an explicit "we would re-evaluate this decision if X" trigger.

## Deliverables

- `reference/` — the ~200 HTML pages plus their reference extractions (`page_001.html`, `page_001.reference.txt`).
- `code/` — scripts to run both extractors and compute metrics.
- `results/per_page.parquet` — one row per page × extractor with all measured metrics.
- `results/summary.md` — aggregate + per-type breakdown as a markdown table.
- `decision_memo.md` — one-page decision memo in the voice of the pipeline owner.
- `README.md` — how to re-run the benchmark; pinned package versions.

## Acceptance criteria

- Reference set is at least 200 pages across at least 4 page types with at least 5 TLDs represented.
- Reference-methodology section in the README explicitly states how reference extractions were produced and discusses their limitations (e.g. "Readability.js outputs were accepted as reference without further review — this biases the benchmark toward trafilatura's agreement with Readability").
- Metrics are computed per-page and aggregated with mean + median + 10th percentile; max/min outliers are retained as examples.
- Throughput measurements name the hardware: core model, clock, Python version, library version. "trafilatura is 5× slower" without hardware context is not an acceptable finding.
- Decision memo includes the specific numbers that drove it. "We recommend X because its Y metric exceeded Z by W" is the pattern.
- Extractor versions are pinned (`trafilatura==x.y.z`, `resiliparse==a.b.c`); the comparison is invalidated when versions drift.
- Any inability to install or run a dependency is marked with `<!-- needs-research: ... -->` rather than silently dropped.

## Stretch goals

- **Add a third extractor** — Mozilla's Readability.js via a headless browser, or Diffbot's API, or the WET baseline. The point is to anchor your two primary extractors against a third reference.
- **Ablate trafilatura config**: `favor_precision` vs `favor_recall`, `include_comments` on/off, `include_tables` on/off. Produce a small sweep and report the Pareto frontier of quality vs throughput.
- **Ablate resiliparse config**: `main_content=True` vs `False`, `preserve_formatting` on/off.
- **Measure the downstream effect.** Pick 20 pages, run both extractors, then run the Chapter 02 KenLM perplexity scoring on both outputs. Do the extractor choices change which tier the document lands in?
- **Build a router**: a simple classifier (page length + text-density heuristics) that picks trafilatura or resiliparse per page. Measure whether the router beats either individual extractor.

## What this exercise is not

- It is not a complete evaluation of extraction technology — an industrial benchmark would involve thousands of pages and multiple labelers.
- It is not an argument that one extractor is universally better. The right answer depends on your slice composition and your compute budget.
- It is not a license-independent recommendation — if your pipeline needs the Apache-2.0 license (resiliparse) over GPLv3+ (trafilatura is GPL; verify current license at read time), that constraint overrides the numbers.

The exercise is to *measure*, not to decide in the abstract.
