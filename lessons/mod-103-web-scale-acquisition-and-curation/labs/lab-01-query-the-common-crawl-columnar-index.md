# Lab 01 — Query the Common Crawl Columnar Index and Emit a Selection Predicate

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 01 — Common Crawl anatomy](../01-common-crawl-anatomy.md). [Chapter 06](../06-crawling-ethics-in-code.md) supplies the "group by host" consequence.
**Relationship to the exercises:** [Exercise 01](../exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md) is the full end-to-end: index → acquire check → range-fetch → extract → language-filter → manifest. This lab is the *first step of that path, in isolation*: author a reproducible selection predicate against the Parquet index, produce a URL list with the `(warc_filename, offset, length)` triples the rest of the pipeline consumes, and inspect its host distribution. You stop before any robots check or WARC fetch — those belong to Exercise 01.

## Objective

By the end of this lab you will have, for one snapshot:

1. A `selection_predicate.sql` file — a DuckDB query against the Common Crawl columnar Parquet index that defines a reproducible slice.
2. A `url_candidates.parquet` file — one row per candidate URL with at minimum `(url, url_host_registered_domain, warc_filename, warc_record_offset, warc_record_length, content_mime_type, content_languages, fetch_status)`.
3. A short `slice_scope.md` that records the snapshot ID, row count, estimated byte budget for the subsequent range fetches, and the top-20 hosts in the slice (so you know what the Chapter 06 acquire-stage check will actually be *checking against*).

The point: the slice is defined by **predicate + per-record manifest** (Chapter 01 §"The slice"), and the predicate is the single artifact that makes the whole downstream pipeline reproducible. This lab makes that one file tangible.

## Prerequisites

- You have read Chapter 01 and skimmed Chapter 06.
- You have Python 3.11+ and `duckdb>=0.10` installed (`pip install duckdb`). The lab works on a workstation; no AWS account required.
- You have ~2 GB of free disk for DuckDB's local cache of index metadata and for the output Parquet.
- You have a reliable internet connection; the Common Crawl Parquet index at `s3://commoncrawl/cc-index/table/cc-main/warc/` is served over anonymous HTTPS and reachable from anywhere. If you are outside AWS us-east-1 the query will be slower but still work.
- You have reviewed the current Common Crawl snapshot listing at <https://commoncrawl.org/get-started> and chosen a snapshot ID at least two weeks old (a very fresh snapshot may still be indexing). This lab uses `CC-MAIN-2024-30` as a worked example; substitute a current snapshot if you prefer.

## Step 1 — Create a scratch directory and smoke-test DuckDB against the index

```bash
mkdir -p lab01-cc-index-query
cd lab01-cc-index-query
```

Open a DuckDB shell (`duckdb` CLI, or `python -c "import duckdb; con = duckdb.connect()"`) and run the smallest possible query to verify you can reach the index without pulling a whole snapshot's metadata:

```sql
INSTALL httpfs;
LOAD httpfs;
SET s3_region='us-east-1';
-- Common Crawl is a public bucket; anonymous reads suffice.
-- For https-only reads, duckdb will transparently use HTTPS when it cannot
-- find AWS credentials. No S3 account is required for this lab.

SELECT COUNT(*) AS row_count
FROM read_parquet(
  's3://commoncrawl/cc-index/table/cc-main/warc/crawl=CC-MAIN-2024-30/subset=warc/part-00000-*.parquet'
);
```

You should see a row count for *one* index partition file — a few hundred thousand to a few million rows depending on the partition. If DuckDB errors on S3 access, swap the URI for the HTTPS mirror:

```sql
FROM read_parquet(
  'https://data.commoncrawl.org/cc-index/table/cc-main/warc/crawl=CC-MAIN-2024-30/subset=warc/part-00000-*.parquet'
)
```

**Checkpoint:** the single-partition `COUNT(*)` returns a number greater than zero within ~30 seconds. If it hangs for minutes, your httpfs is routing badly — try the HTTPS mirror.

## Step 2 — Inspect the index schema

Chapter 01 §"The index — why you almost never grep a WARC" lists the columns you will care about. Verify them against *your chosen snapshot's* partition; schemas evolve across years.

```sql
DESCRIBE SELECT * FROM read_parquet(
  's3://commoncrawl/cc-index/table/cc-main/warc/crawl=CC-MAIN-2024-30/subset=warc/part-00000-*.parquet'
) LIMIT 0;
```

You should see, at minimum, these columns in the output:

- `url`, `url_host_name`, `url_host_registered_domain`, `url_host_tld`
- `url_path`, `url_query`
- `warc_filename`, `warc_record_offset`, `warc_record_length`
- `fetch_status`, `content_mime_type`, `content_mime_detected`, `content_charset`, `content_languages`
- `crawl`

If any of those columns are missing or renamed in your snapshot, note it in `slice_scope.md`. Re-verify and prefer the names that are actually present. Mark any surprising divergence with `<!-- needs-research: ... -->` and move on with the columns that *do* exist.

**Checkpoint:** `DESCRIBE` returns a column list containing at least `url`, `warc_filename`, `warc_record_offset`, `warc_record_length`, `fetch_status`, `content_mime_type`, and `content_languages`.

## Step 3 — Author the selection predicate

Create `selection_predicate.sql`. Keep it saveable, reviewable, and executable without edits — this file *is* the slice definition.

```sql
-- selection_predicate.sql — mod-103 lab-01 reproducible slice
-- Snapshot: CC-MAIN-2024-30 (substitute your chosen snapshot).
-- Objective: ~100,000 English HTML captures for downstream CCNet-style processing.
-- Predicate shape per Chapter 01 §"The slice — what 'pulling a slice' actually means".

INSTALL httpfs;
LOAD httpfs;
SET s3_region='us-east-1';

COPY (
  SELECT
    url,
    url_host_registered_domain,
    url_host_tld,
    warc_filename,
    warc_record_offset,
    warc_record_length,
    content_mime_type,
    content_languages,
    fetch_status
  FROM read_parquet(
    's3://commoncrawl/cc-index/table/cc-main/warc/crawl=CC-MAIN-2024-30/subset=warc/*.parquet'
  )
  WHERE fetch_status = 200
    AND content_mime_type = 'text/html'
    AND content_languages LIKE 'eng%'
    AND url_host_tld IN ('com', 'net', 'org', 'edu', 'gov')
  USING SAMPLE 150000 ROWS
) TO 'url_candidates.parquet' (FORMAT PARQUET);
```

Points to notice:

- `USING SAMPLE 150000 ROWS` keeps the output size bounded and leaves headroom for drops at later stages (robots opt-outs, language-id disagreement, extractor failures). Exercise 01 asks for ~100,000 surviving documents; aim higher upstream.
- `content_languages LIKE 'eng%'` is the index's declared-language hint — a crawl-time guess, *not* a post-extraction classification. CCNet's FastText `lid.176` is what actually decides the final slice; this predicate just prunes the obvious non-matches cheaply.
- The predicate includes `warc_record_offset` and `warc_record_length`. Together with `warc_filename` these three columns are what makes the range-fetch work; keep them in the output or the downstream range fetch cannot locate the record.
- The predicate is TLD-restricted. This is a *selection* choice that biases your slice and must be documented in `slice_scope.md`. Dolma, RefinedWeb, and FineWeb each make different TLD calls; yours is yours.

Run the predicate from Python so you can time it and log row counts (DuckDB's CLI works too; either is fine):

```python
# run_predicate.py
import time
import duckdb

con = duckdb.connect()
con.execute("INSTALL httpfs; LOAD httpfs; SET s3_region='us-east-1';")

start = time.time()
con.execute(open("selection_predicate.sql").read())
elapsed = time.time() - start

rows = con.execute("SELECT COUNT(*) FROM 'url_candidates.parquet'").fetchone()[0]
print(f"predicate wall-clock: {elapsed:.1f}s")
print(f"url_candidates rows: {rows:,}")
```

```bash
python run_predicate.py
```

Expected output (shape — exact numbers depend on the snapshot and sampling):

```
predicate wall-clock: 180.0s
url_candidates rows: 150,000
```

Wall-clock varies enormously by region and network. From outside us-east-1, 3–15 minutes is normal; from an EC2 instance in us-east-1, under a minute is normal. If it exceeds 30 minutes, kill it and rerun against a single partition first (`part-00000-*.parquet`) to confirm your network path.

**Checkpoint:** `url_candidates.parquet` exists and contains between 100,000 and 150,000 rows, each with non-null `warc_filename`, `warc_record_offset`, and `warc_record_length`.

## Step 4 — Estimate the downstream byte budget

The slice is not useful on its own — someone will range-fetch every record. Compute how many bytes that is, so you know whether to run from inside AWS or over the public HTTPS mirror.

```python
# estimate_bytes.py
import duckdb

con = duckdb.connect()
total_bytes = con.execute("""
  SELECT SUM(warc_record_length) AS total_bytes,
         COUNT(DISTINCT warc_filename) AS warc_file_count,
         COUNT(DISTINCT url_host_registered_domain) AS distinct_hosts
  FROM 'url_candidates.parquet'
""").fetchone()

print(f"total range-fetch bytes: {total_bytes[0] / 1e9:.1f} GB")
print(f"distinct WARC files touched: {total_bytes[1]:,}")
print(f"distinct hosts (= robots.txt fetches for Chapter 06): {total_bytes[2]:,}")
```

```bash
python estimate_bytes.py
```

Expected output (shape):

```
total range-fetch bytes: 4.2 GB
distinct WARC files touched: 243
distinct hosts (= robots.txt fetches for Chapter 06): 38,041
```

Two observations to record in `slice_scope.md`:

- **Byte budget.** The total compressed bytes you will pull via range requests. Chapter 01 §"Costs and etiquette": "Use HTTP range requests on the WARC files. **Do not download whole WARC files** unless you genuinely need >50% of their records." The byte-budget number is your evidence that you are not accidentally pulling whole WARCs.
- **Distinct-host count.** This is the *actual* number of robots.txt fetches the Chapter 06 acquire-stage check will make (one per host, not one per URL — see Chapter 06 §"Where this lives in the pipeline" operational note 1). A 100k-URL slice with 38k distinct hosts means 38k robots.txt fetches at acquire time, which is where host-level rate limiting and persistent caching actually matter.

**Checkpoint:** `estimate_bytes.py` prints a total under ~50 GB (if much larger, your predicate is too broad for a lab-scale slice; narrow the TLD or raise the content-type precision), a WARC-file count under ~1,000, and a distinct-host count under ~100,000.

## Step 5 — Compute the top-20 host distribution

A slice's host distribution is the first thing an auditor looks at. A heavy-tailed distribution concentrated on ten hosts is a different slice from one spread across 30,000 — both are legitimate but they mean very different things for the Chapter 06 ethics check (more hosts = more robots.txt fetches) and for downstream mod-104 representativeness.

```python
# top_hosts.py
import duckdb

con = duckdb.connect()
rows = con.execute("""
  SELECT url_host_registered_domain AS host,
         COUNT(*) AS captures
  FROM 'url_candidates.parquet'
  GROUP BY url_host_registered_domain
  ORDER BY captures DESC
  LIMIT 20
""").fetchall()

for host, captures in rows:
    print(f"{captures:>8,}  {host}")
```

```bash
python top_hosts.py > top_hosts.txt
head -5 top_hosts.txt
```

Expected output (shape — exact hosts depend on the snapshot):

```
   1,842  en.wikipedia.org
   1,204  nytimes.com
     940  medium.com
     812  github.com
     644  blogspot.com
```

If the top host carries more than 10% of the slice, note it in `slice_scope.md`: a single-host-dominated slice biases every downstream statistic and is often a mistake unless intentional.

**Checkpoint:** `top_hosts.txt` lists 20 rows; the single top host accounts for less than ~10% of the slice (if not, narrow the predicate in Step 3 — e.g. cap captures per host with a window function — and rerun).

## Step 6 — Write `slice_scope.md`

Create `slice_scope.md` with the following sections, filled in from your actual numbers. Keep it under one page.

```markdown
# slice-cc-en-lab01

## Scope

- Snapshot: CC-MAIN-2024-30
- Predicate file: selection_predicate.sql
- Predicate SHA-256: <output of `sha256sum selection_predicate.sql`>
- Row count: <from Step 3>
- Total range-fetch bytes (compressed): <from Step 4>
- Distinct WARC files touched: <from Step 4>
- Distinct hosts: <from Step 4>
- Top host share: <top host captures / total> %

## Known biases

- TLD-restricted to `.com .net .org .edu .gov` — non-EN-speaking regions and
  country-coded TLDs are excluded by construction. Downstream representativeness
  (mod-104) must call this out.
- `content_languages LIKE 'eng%'` uses the index's declared-language hint, which
  is a crawl-time guess — not a classifier decision. The real language decision
  happens at the Chapter 02 FastText `lid.176` stage.
- `USING SAMPLE 150000 ROWS` is deterministic within a DuckDB version but the
  sampling algorithm is not specified across versions. Pin `duckdb==<your version>`
  in `requirements.lock` if byte-for-byte reproducibility across machines matters.

## Downstream obligations (not done in this lab)

- Chapter 06 acquire-stage opt-out check: ~<distinct_hosts> robots.txt fetches,
  one per host; cache persistently across runs.
- Chapter 01 range-fetch: ~<bytes> GB via HTTP range requests, do NOT download
  whole WARC files.
- Chapter 02 FastText `lid.176` + KenLM perplexity + paragraph SHA dedup.
- Chapter 07 manifest emission binding every operator's config and output.
```

**Checkpoint:** `wc -w slice_scope.md` prints a number between 150 and 350.

## Step 7 — Pin and verify reproducibility

A predicate that produces different results on different runs is not a predicate. Pin the DuckDB version and verify that two consecutive runs produce the same `url_candidates.parquet` by comparing row-count and a sample's SHA-256:

```bash
python -c "import duckdb; print(duckdb.__version__)" > duckdb_version.txt

# First run
python run_predicate.py
python -c "import duckdb; con = duckdb.connect(); print(con.execute(\"SELECT COUNT(*), MD5(STRING_AGG(url, ';' ORDER BY url)) FROM 'url_candidates.parquet'\").fetchone())" > run1.hash

# Second run (delete the output first so DuckDB rewrites)
rm url_candidates.parquet
python run_predicate.py
python -c "import duckdb; con = duckdb.connect(); print(con.execute(\"SELECT COUNT(*), MD5(STRING_AGG(url, ';' ORDER BY url)) FROM 'url_candidates.parquet'\").fetchone())" > run2.hash

diff run1.hash run2.hash && echo "OK: predicate reproducible across runs"
```

Expected output:

```
OK: predicate reproducible across runs
```

If the two runs diverge, the predicate's `USING SAMPLE` is version-dependent on your DuckDB. Replace sampling with a deterministic cut — e.g. `WHERE abs(hash(url)) % 1000 < 7` for a 0.7% random-looking-but-deterministic subset — and rerun.

**Checkpoint:** `diff run1.hash run2.hash` returns no differences and prints `OK: predicate reproducible across runs`.

## What you have at the end

```
lab01-cc-index-query/
├── selection_predicate.sql       # the slice definition
├── url_candidates.parquet        # the per-record manifest (sans body SHA256)
├── slice_scope.md                # the auditor's one-pager
├── top_hosts.txt                 # host-distribution evidence
├── duckdb_version.txt            # reproducibility pin
├── run1.hash, run2.hash          # reproducibility proof
├── run_predicate.py
├── estimate_bytes.py
└── top_hosts.py
```

## Discussion

- The slice is now defined by **two files plus a version pin**: `selection_predicate.sql` + `url_candidates.parquet` + `duckdb_version.txt`. Anyone with those three can regenerate the candidate URL list. `observed_sha256` of each response body is the *last* piece that completes the Chapter 01 slice definition and arrives in Exercise 01's range-fetch step.
- Chapter 01 called out that "the columnar index lets you treat a 90 TB snapshot as a **selectable table**." This lab made that concrete: you queried a public 90 TB dataset from your workstation and emitted a 150k-row candidate list in minutes, without downloading anything near a WARC.
- The distinct-host count is the hinge where Chapter 01 meets Chapter 06. Before this lab the number was abstract; now you have it for your actual slice. When you run Exercise 01, that count is the cardinality of your robots-txt-cache keyset and the gating factor on how aggressively you can parallelise acquire-stage checks.
- `USING SAMPLE` is convenient but a trap for reproducibility; the deterministic-hash alternative is more defensible. For a lab the sample is fine; for a production slice you ship under a dataset card, use the hash.

## If you want to go further (optional)

- Add a predicate that **excludes** hosts on the UT1 blocklists (<https://dsi.ut-capitole.fr/blacklists/>) and measure how many candidates you lose. Compare the host distribution before and after.
- Restrict the predicate to one Common Crawl *segment* instead of a whole snapshot and compare wall-clock. Segments are `segments/1718144497.44/` directory shards inside the snapshot — DuckDB's partition pruning can skip most of the index when you add `AND warc_filename LIKE '%/segments/1718144497.44/%'`.
- Replace `USING SAMPLE` with the deterministic-hash form (`WHERE abs(hash(url)) % 1000 < 7`) and re-run the Step 7 reproducibility check. Record the two approaches' trade-offs in `slice_scope.md`.
- Repeat the query against the *legacy CDX index* (`s3://commoncrawl/cc-index/collections/CC-MAIN-2024-30/indexes/`) via `commoncrawl/cc-index-server` (<https://github.com/commoncrawl/cc-index-server>) and compare throughput against the Parquet query. Chapter 01 §"The index" mentions both; this is the operational comparison.
- Extend `slice_scope.md` into a draft dataset card in the Hugging Face format. The card will be re-opened in Exercise 03 and again at Chapter 07 hand-off time; starting it here means the governance story is in place from the first file.
