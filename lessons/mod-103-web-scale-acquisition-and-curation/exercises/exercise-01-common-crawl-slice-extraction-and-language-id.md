# Exercise 01 — Common Crawl Slice Extraction and Language-ID

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 01 — Common Crawl anatomy](../01-common-crawl-anatomy.md), [Chapter 02 — CCNet-style pipeline end-to-end](../02-ccnet-pipeline-end-to-end.md)
**Companion:** [Chapter 06 — Crawling-ethics in code](../06-crawling-ethics-in-code.md) (required for the acquire-stage check)

## Objective

Pull a small, reproducible slice of one Common Crawl snapshot, extract documents from WARC records, run the CCNet-style language-ID stage on them, and emit a slice manifest that another engineer could use to regenerate the slice. The goal is to physically do every step you read about in Chapters 01 and 02 at a scale small enough to iterate on in minutes.

By the end you will have:
- A DuckDB/Polars query against the Common Crawl columnar index that defines your slice.
- A per-record manifest of `(url, warc_filename, offset, length, observed_sha256)` tuples.
- An acquire-stage opt-out decision attached to each record (even if the result is "allowed" for the entire slice).
- A stream of English documents that passed `fasttext lid.176` at your chosen threshold.
- A slice manifest JSON in the shape from Chapter 07.

## Prerequisites

- You have read Chapters 01, 02, and 06.
- You have Python 3.11+ with `duckdb`, `warcio`, `fasttext-wheel`, `httpx`, and a robots.txt parser of your choice (`reppy` works) installed.
- You have downloaded `lid.176.bin` from <https://fasttext.cc/docs/en/language-identification.html> and know its SHA-256.
- You have approximately 10 GB of free disk and a reliable internet connection. Running inside AWS us-east-1 is faster and cheaper but not required.
- You have reviewed one entry in the current Common Crawl snapshot listing (<https://commoncrawl.org/get-started>) and chosen a snapshot ID (e.g. `CC-MAIN-2024-30`). Pick a snapshot at least two weeks old to avoid in-flight indexing.

## Problem statement

Build a slice named `slice-cc-en-small-v1` consisting of approximately 100,000 English-language HTML documents from a single Common Crawl snapshot, filtered at the acquire stage for AI-use opt-outs and at the content stage for English per `lid.176`.

Your slice must be regenerable from its manifest alone — someone else with S3 read access and your code should be able to produce byte-identical outputs by reading the manifest.

## Starter guidance

1. **Pick a snapshot and a selection predicate.** Something like: TLD in `('com','net','org','edu','gov')`, declared content language starts with `eng`, HTTP status 200, MIME type `text/html`, random sample of 150,000 URLs (leave headroom for drops). Save the predicate as `selection_predicate.sql`.

2. **Query the columnar index.** The Parquet index lives at `s3://commoncrawl/cc-index/table/cc-main/warc/crawl={snapshot}/subset=warc/`. DuckDB with `httpfs` is the shortest path:
   ```sql
   INSTALL httpfs; LOAD httpfs;
   SET s3_region='us-east-1';
   -- unsigned S3 reads (Common Crawl bucket is public)
   SET s3_use_ssl=true;
   ```
   Expect the index scan to pull a non-trivial amount of data (hundreds of MB to a few GB of index metadata). Consider running an initial `LIMIT 1_000_000` to prototype your predicate before letting it scan the whole index.

3. **Group URLs by host and run the Chapter 06 acquire-stage check.** One robots.txt fetch per host; parse for `GPTBot`, `Google-Extended`, `CCBot`, `ClaudeBot`, `PerplexityBot`, `Applebot-Extended` (verify the current agent list), plus optional `ai.txt`. Persist a per-URL decision record with the robots.txt SHA-256 as evidence. Drop URLs whose decision is `allowed=false`.

4. **Range-fetch the WARC records** for the surviving URLs using `warcio.ArchiveIterator` on a byte-range HTTP GET. Compute `observed_sha256` of the response body and add it to the manifest.

5. **Extract text** with any extractor — a simple `BeautifulSoup` → `.get_text()` is sufficient for this exercise; Exercise 02 is where you compare proper extractors. Document the extractor used and its config.

6. **Run FastText `lid.176`** at threshold 0.5 on each document; keep the ones labelled `en`. Log the dropped fraction.

7. **Emit the slice manifest** in the schema from Chapter 07 §"The slice manifest schema" (trim to the fields relevant to this exercise — you're not running MinHash or decontamination yet).

## Deliverables

Working directory (all under version control):

- `selection_predicate.sql` — the DuckDB query against the CC index.
- `code/` — your code. One script per stage is fine; a `data-juicer` or `dolma` config is also fine.
- `manifest/slice_manifest.json` — the slice manifest in the Chapter 07 shape.
- `manifest/per_record_manifest.parquet` — one row per surviving URL with `(url, warc_filename, offset, length, observed_sha256, acquire_decision, language, language_score)`.
- `manifest/acquire_evidence/{host_hash}.robotstxt` — the robots.txt cache keyed by host; each file's SHA-256 matches `acquire_decision.evidence_sha256`.
- `output/slice-00000.jsonl.gz` ... — the English documents that passed filtering, in JSONL with at minimum `{text, url, warc_record_id}` per line. One file is fine for this scale; multi-shard is a stretch goal.
- `README.md` — how to re-run the whole thing from the manifest. Include the SHA-256 of `lid.176.bin` and your Python package versions (`pip freeze > requirements.lock`).

## Acceptance criteria

- The slice is reproducible: another reviewer, starting from `selection_predicate.sql` + your code + the manifest, produces the same per-record SHA-256s. (Note that if the snapshot's underlying data were to change — which it does not for a published snapshot — the manifest would record the divergence via `observed_sha256`.)
- Acquire-stage opt-out decisions exist for every URL in the slice, including evidence SHA-256 and timestamp. At least one `evidence_sha256` is independently reverifiable by re-fetching the host's robots.txt and recomputing the hash.
- Language ID results exist for every document; the slice's final doc count is between 80,000 and 120,000 after filtering.
- The slice manifest validates against the Chapter 07 schema shape (relevant fields only).
- The README contains a `make reproduce` or equivalent that re-emits the per-record manifest from the predicate + code.
- Any claim in the README that could not be verified at authoring time is marked with `<!-- needs-research: ... -->` rather than guessed.

## Stretch goals

- **Replace `lid.176` with `glotlid` (Kargaran 2024, <https://aclanthology.org/2024.findings-acl.47/>)** and compare the two on the same slice. Which languages does each over-predict?
- **Run the full CCNet filter chain** including a KenLM perplexity tier step. You will need to train or download a 5-gram KenLM on cleaned English Wikipedia; `kenlm` ships command-line tools for both.
- **Run the pipeline through `data-juicer` or the `dolma` toolkit** instead of writing operators from scratch. Compare operator counts and throughput.
- **Scale to a Ray cluster** and process ten WARC files in parallel. Measure wall-clock vs. single-node.
- **Compute Dodge-style representativeness statistics** on the pre- vs post-filter population (TLD distribution, document length distribution, host diversity). This previews mod-104.

## What this exercise is not

This exercise is not a benchmark of extraction quality — that is Exercise 02. It is not a MinHash dedup — that is mod-105 / Exercise 03. It is not a full reproduction of any public reference pipeline — that is Exercise 03.

It is the end-to-end mechanical skill: index → URL list → acquire-stage check → range-fetch → extract → language-filter → manifest. Everything else in the module layers on top.
