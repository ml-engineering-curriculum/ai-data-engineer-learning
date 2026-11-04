# Chapter 01 — Common Crawl Anatomy: WARC, WAT, WET, and the Index

## Why this chapter exists

Almost every open web-scale pretraining corpus of the last decade — C4, The Pile's `CommonCrawl` subset, OSCAR, mC4, RedPajama, RefinedWeb, FineWeb, Dolma's `cc` subset — starts from one source: a snapshot of [**Common Crawl**](https://commoncrawl.org/). If you are going to work on training data at scale you need to know what a Common Crawl snapshot *is* on disk, how to pull a slice without downloading the whole thing, and what each of the three file families (WARC, WAT, WET) actually gives you.

This chapter is the ground truth. Later chapters assume you can reach into a monthly snapshot and produce a stream of documents you can filter.

## What Common Crawl publishes

Common Crawl is a US 501(c)(3) non-profit that operates a monthly broad web crawl and publishes the output as a public, free, Creative Commons-licensed dataset hosted on Amazon S3 (`s3://commoncrawl/`) and mirrored over HTTPS. Each monthly crawl is called a **snapshot** (also "release", "dump", or "crawl"), named by its archive identifier such as `CC-MAIN-2024-30` (the 30th week-tagged crawl of 2024).

Common Crawl Foundation publishes:

- **The snapshots themselves** at `s3://commoncrawl/crawl-data/{CC-MAIN-YYYY-WW}/`.
- **An index of every URL captured** at `s3://commoncrawl/cc-index/` in Parquet (`cc-index/table/cc-main/warc/`) and in a legacy CDX JSON format (`cc-index/collections/{snapshot}/indexes/`).
- **The crawler itself**, a fork of Apache Nutch, publicly documented at <https://github.com/commoncrawl/nutch>.
- **The robots.txt policy they operate under** — their crawler identifies as `CCBot`, and the published crawling policy is that they respect `robots.txt`, back off on 429/503, and do not retry against hosts that explicitly disallow them. This is relevant to Chapter 06: if you are redistributing documents that originated in a Common Crawl snapshot, you are inheriting CCBot's compliance posture, but the *training-use* opt-outs (GPTBot, Google-Extended, `ai.txt`) are additive and must be re-checked by you.

Each monthly snapshot is roughly 90–100 TB compressed and tens of thousands of files. You will almost never download a whole snapshot. The skill you are learning is **slice extraction**: deciding which files, which URLs, and which byte-ranges you actually need, then pulling only those.

## WARC, WAT, WET — pick the right format for the job

For every page captured, Common Crawl emits three parallel record streams, grouped into gzip-compressed segment files of a few hundred megabytes each.

| File family | Suffix | What's in it | When you use it |
|---|---|---|---|
| **WARC** | `.warc.gz` | The raw HTTP capture: request, response headers, full response body (HTML, images, PDFs, redirects, failures). Format defined by ISO 28500:2017. | When you want to run your own extraction (trafilatura, resiliparse) or need headers / status / MIME-type metadata. |
| **WAT** | `.warc.wat.gz` | A per-record metadata summary in JSON-in-WARC-record form: HTTP headers, HTML `<meta>`, extracted hyperlinks, HTTP response status, declared language, content type. No page body. | When you need to pivot on structure — outbound links, canonical URL, language `hint`, MIME type — without parsing HTML. Also small enough to scan a whole snapshot's metadata fast. |
| **WET** | `.warc.wet.gz` | Plain-text extraction of each HTML response, produced by Common Crawl's own extractor (based on Apache Tika / Nutch). One text blob per captured page. | When you want a quick baseline text stream and are willing to inherit Common Crawl's extractor's boilerplate behaviour. |

The *big* decision early in a pipeline is **WARC-or-WET**. WET is a huge shortcut — you skip HTML parsing and get straight to text — but you also inherit the extractor's weaknesses: menu text, cookie banners, navigation, and footer boilerplate tend to come through. Most recent open pretraining corpora (C4, RefinedWeb, FineWeb) start from WET as a baseline but *re-extract* from WARC when they want tighter main-content boundaries. Dolma's `cc` subset goes further and runs its own `trafilatura`-based extractor on WARC. You will do the WET-vs-WARC comparison explicitly in Chapter 03 and Exercise 02.

The ISO 28500:2017 WARC specification is the normative reference. The record types you will care about are `response` (the HTTP capture), `metadata` (WAT content), and `conversion` (WET content). Every WARC record has a `WARC-Target-URI` header; you can use that as the primary join key across the three file families. Record IDs (`WARC-Record-ID`) are stable within a snapshot and let you re-fetch an individual capture by byte range.

## The index — why you almost never grep a WARC

Each snapshot ships with a **columnar index** (`cc-index/table/cc-main/warc/crawl={snapshot}/subset=warc/`) stored as Parquet, queryable with DuckDB, Polars, Trino, Athena, or Spark. Columns include:

- `url` (the captured URL)
- `url_host_name`, `url_host_registered_domain`, `url_host_tld`
- `url_path`, `url_query`
- `warc_filename`, `warc_record_offset`, `warc_record_length` — the byte range inside the WARC file that holds this capture
- `fetch_status`, `content_mime_type`, `content_mime_detected`, `content_charset`, `content_languages`
- `crawl` (the snapshot identifier)

The columnar index lets you treat a 90 TB snapshot as a **selectable table**:

```sql
-- DuckDB against Common Crawl's Parquet index via httpfs
SELECT url, warc_filename, warc_record_offset, warc_record_length, content_languages
FROM read_parquet('s3://commoncrawl/cc-index/table/cc-main/warc/crawl=CC-MAIN-2024-30/subset=warc/*.parquet')
WHERE url_host_tld = 'de'
  AND content_mime_type = 'text/html'
  AND fetch_status = 200
LIMIT 1000000;
```

Once you have the `(warc_filename, warc_record_offset, warc_record_length)` triple for every URL you want, you fetch each capture with a single HTTP range request against the WARC file on S3. That is the mechanic every modern slice tool (`cc-downloader`, `cdx_toolkit`, the Hugging Face `datatrove` orchestrator) uses under the hood.

A legacy **CDX index** (`cc-index/collections/.../indexes/cluster.idx` and `cdx-*.gz`) is still maintained and is still useful for URL-pattern lookup via `commoncrawl/cc-index-server`, but for analytic selection the Parquet columnar index is the right primitive.

## Reading a WARC record

Once you have a `(file, offset, length)` triple, the lightest-weight reader in Python is [`warcio`](https://github.com/webrecorder/warcio) (BSD, maintained by Webrecorder). The pattern:

```python
import io, requests, warcio

CC_BASE = "https://data.commoncrawl.org/"

def read_record(warc_filename: str, offset: int, length: int) -> dict:
    url = CC_BASE + warc_filename
    headers = {"Range": f"bytes={offset}-{offset + length - 1}"}
    resp = requests.get(url, headers=headers, timeout=30)
    resp.raise_for_status()
    stream = warcio.ArchiveIterator(io.BytesIO(resp.content))
    record = next(stream)  # a single WARC record in a byte-range slice
    return {
        "warc_target_uri": record.rec_headers.get_header("WARC-Target-URI"),
        "http_status": record.http_headers.get_statuscode() if record.http_headers else None,
        "content_type": record.http_headers.get_header("Content-Type") if record.http_headers else None,
        "body": record.content_stream().read(),
    }
```

This is also the pattern that bigger tools wrap. [`datatrove`](https://github.com/huggingface/datatrove) (the Hugging Face pipeline that produced FineWeb), [`cc-downloader`](https://github.com/commoncrawl/cc-downloader), and the Common Crawl team's own examples all reduce to "query index → range-fetch WARCs → iterate records".

An important subtlety: WARCs are gzip-concatenated, one gzip member per record, so a byte-range fetch over a single record's `length` is itself a valid gzip stream. `warcio` handles the concatenation automatically.

## WARC vs WAT vs WET — the operational trade-off

| Dimension | WET | WAT | WARC |
|---|---|---|---|
| File size per snapshot | ~8 TB | ~22 TB | ~90 TB |
| Information per record | Plain text only | Metadata + links | Full HTTP response |
| Your extractor runs? | No (CC's does) | N/A | Yes |
| Boilerplate quality | Baseline (inherits CC) | N/A | Yours to control |
| Right for | Quick baselines, iteration during pipeline development | Link-graph / metadata-only analysis, URL filtering | Production-quality curated corpora |

The right call in practice is often **WAT for selection, WARC for extraction, WET for a baseline you benchmark against**. You will do exactly that split in the exercises.

## The slice — what "pulling a slice" actually means

A **slice** is a reproducible subset of a snapshot defined by:

1. **Source snapshot**: `CC-MAIN-2024-30` or whatever.
2. **Selection predicate**: a boolean on the columnar index (TLD, language hint, content-type, host allow/deny list). Save this as SQL or a serialisable JSON predicate.
3. **Per-record manifest**: for every selected capture, the `(url, warc_filename, offset, length, observed_sha256_of_body)` tuple. The `observed_sha256` is yours to compute, not something Common Crawl publishes; it anchors reproducibility across re-downloads.
4. **A `README.md` or dataset card** that describes the three above and names the extractor / language-id model / dedup threshold applied downstream.

A slice is not "a directory of text files". It is the predicate plus the manifest; anyone with S3 read access can regenerate the text from those two artifacts. Treat the manifest as the primary deliverable. [`mod-109-dataset-versioning-lineage-reproducibility`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) covers how to version this manifest in DVC / lakeFS / Pachyderm; for now, a signed `manifest.parquet` + `selection_predicate.sql` + `extraction_config.json` triple is enough.

## Costs and etiquette

Common Crawl is published **free of charge** on both AWS S3 and over HTTPS (`https://data.commoncrawl.org/`). You pay only for your own egress if you are pulling from outside AWS. If you are inside AWS us-east-1, there is no data transfer cost for the S3 reads — a decisive practical reason that Common Crawl pipelines usually run in us-east-1 or in EC2 in a region with a cached mirror.

Etiquette:

- Use HTTP range requests on the WARC files. **Do not download whole WARC files** unless you genuinely need >50% of their records; you will saturate CC's egress for no reason.
- Identify yourself with a reasonable `User-Agent`. Common Crawl's distribution servers do not enforce it but their ops team appreciates it.
- If you are running a pipeline across tens of thousands of range requests, prefer `s3://` reads from an AWS account to HTTPS from the internet.
- Cache aggressively. Byte ranges are immutable within a snapshot.

## Summary

- Common Crawl publishes monthly web snapshots on S3 (`s3://commoncrawl/`) in three parallel file families: **WARC** (raw HTTP), **WAT** (metadata), **WET** (plain text).
- The ISO 28500:2017 WARC spec is the normative reference; `warcio` is the right Python reader.
- The **columnar Parquet index** (`cc-index/table/cc-main/warc/`) turns a 90 TB snapshot into a selectable table — query it with DuckDB / Polars / Spark, pull the `(warc_filename, offset, length)` triple, range-fetch the records.
- WET is the quickest baseline but inherits CC's extractor's boilerplate; production corpora re-extract from WARC. Chapter 03 benchmarks the alternatives.
- A **slice** is a reproducible `(selection predicate, per-record manifest with SHA256, extraction config)` triple — not a pile of text files.
- Run from us-east-1 if you can, use range requests, cache aggressively, and send a real `User-Agent`.

Chapter 02 picks up the stream of raw records this chapter produces and runs the CCNet-style pipeline on it: language ID, perplexity filter, hash dedup. Exercise 01 is where you first pull a slice end-to-end.
