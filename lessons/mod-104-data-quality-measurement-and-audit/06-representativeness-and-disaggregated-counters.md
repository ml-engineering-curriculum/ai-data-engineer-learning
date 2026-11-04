# Chapter 06 — Representativeness and Disaggregated Counters

## Why this chapter exists

Aggregates lie. A slice that is "99% consistent" can contain a language that is 42% consistent; a slice that is "nicely distributed across TLDs" can concentrate 90% of its Swahili content on three hosts; a slice with a healthy documents-per-year distribution overall can have a dialectal subset entirely from 2008–2012.

The checks from Chapters 02–03 and the audits from Chapters 04–05 all depend on *disaggregated counters* — the same metric computed per cell along one or more dimensions. The dimensions that matter for an ML training slice are the ones ISO/IEC 5259-2 calls out under **representativeness**, **dataset-level diversity**, and **dataset balance**:

- **Language / dialect / sociolect** — the Dodge-dialect and Kreutzer-language stories live here.
- **Temporal** — document crawl date, publication date where available, event time for time-sensitive data.
- **Source domain / registered domain / TLD** — the host-concentration story.
- **Geography** — inferred country of origin (via WHOIS, TLD heuristics, author metadata); handle with care, document the inference method.
- **Topic / domain of content** — medical / legal / news / code / forum / commerce.
- **Length / format** — document length, token count, markup density.
- **Modality / mixture tag** — real / synthetic / human-labelled / machine-labelled (if applicable to the slice).

This chapter is the plumbing. The deliverable is a **per-slice `distribution.parquet` plus a representativeness report** that the per-slice quality report (Chapter 08) links out to.

## The two distributions you care about

For every dimension, you are comparing *two* distributions:

1. **The slice distribution** — the actual proportion of records in each cell.
2. **The target distribution** — what the slice *should* look like, as derived from one of:
   - The downstream population (if the model will be deployed against, say, English queries from a known user population).
   - A reference corpus you trust (Wikipedia language proportions, CLD3 fingerprint of a representative web sample, your organisation's existing production data).
   - A policy decision (an uplift of under-represented languages, a cap on English, a parity constraint between dialects).

The representativeness measure is the **divergence** between the two:

- Total variation distance (TVD) — simple, symmetric, bounded `[0, 1]`.
- Kullback–Leibler divergence (KL) — asymmetric; use when "direction" matters.
- Jensen–Shannon divergence (JSD) — symmetric smoothed KL, bounded `[0, log 2]`.
- Chi-square — categorical; useful when target counts (not proportions) are known.
- Kolmogorov–Smirnov (KS) — ordinal / continuous dimensions like document length or date.
- Earth-Mover's Distance (Wasserstein) — ordinal with an explicit cost structure.

Pick the one that matches the dimension type. Record *which* you used and the formula source.

## The counter infrastructure

The core abstraction is a *disaggregated counter*: `counter[(dim_1, dim_2, …, dim_k)] -> metric_value`. Produce it during the pipeline run, not as a post-hoc Pandas slice of the final table. Three reasons:

1. **Many metrics are not reducible from the final table alone.** "Pre-filter language distribution" requires the pre-filter table; keeping it is expensive, but keeping the counter is cheap.
2. **Spark-native aggregation is cheap when it piggy-backs on the pipeline's existing passes.** Compute the per-cell counts in the same stage that applies the filter.
3. **Per-record-ID attribution is auditable.** Counters can emit per-cell-example samples (`sample_doc_ids`) that let a human verify the counter without re-querying the full slice.

A working shape for the counter:

```python
# counters.py
from collections import defaultdict
import pyarrow as pa, pyarrow.parquet as pq

class DisaggCounter:
    def __init__(self, dims: list[str], metrics: list[str]):
        self.dims = dims
        self.metrics = metrics
        # cell -> {metric: value}, plus `count` always
        self._cells: dict[tuple, dict[str, float]] = defaultdict(lambda: {"count": 0})

    def observe(self, record: dict, metrics: dict[str, float] | None = None):
        cell = tuple(record.get(d, "__unknown__") for d in self.dims)
        self._cells[cell]["count"] += 1
        if metrics:
            for k, v in metrics.items():
                self._cells[cell][k] = self._cells[cell].get(k, 0.0) + v

    def dump(self, path: str):
        rows = []
        for cell, metrics in self._cells.items():
            row = dict(zip(self.dims, cell)) | metrics
            rows.append(row)
        pq.write_table(pa.Table.from_pylist(rows), path)
```

Use it in the pipeline stage itself, not in a separate post-processor:

```python
cnt_pre  = DisaggCounter(dims=["declared_language","url_host_tld","crawl_year"], metrics=["sum_text_len"])
cnt_post = DisaggCounter(dims=["declared_language","url_host_tld","crawl_year"], metrics=["sum_text_len"])

for doc in raw_records:
    cnt_pre.observe(doc, {"sum_text_len": len(doc["text"])})
    if keep(doc):                     # the filter under audit
        cnt_post.observe(doc, {"sum_text_len": len(doc["text"])})

cnt_pre.dump("audit/counters_pre.parquet")
cnt_post.dump("audit/counters_post.parquet")
```

In Spark, replace the loop with `groupBy(dims).agg(...)` — Deequ's `hasHistogramValues` gets you the counts; a plain Spark aggregation gets you everything else.

## Intersectional cells are where the failures hide

A one-dimensional counter *(language-only)* misses the Swahili-concentrated-on-three-hosts story. A two-dimensional *(language × host)* catches it. A three-dimensional *(language × host × time)* catches the time-concentrated subset of that.

Default to **at least two dimensions**, and **three when any of the dimensions is itself coarse**. Beyond three, you hit the small-cell problem: many cells have zero or one record and the proportion estimate is meaningless. Two mitigations:

- **Bucket coarse dimensions** — crawl date to year-quarter, length to log-scale bucket, TLD to a {gov, edu, org, com, …, other} coarsening.
- **Report cell counts alongside proportions** and suppress cells below a minimum (say, 30 records) in the published report. A suppressed cell is still *counted*; it just is not individually profiled.

## A worked example — language × TLD × year

For an English-dominated multilingual slice:

```
counter dims = (declared_language, url_host_tld, crawl_year)
                + per-cell metrics: count, sum_text_len, mean_language_score, pii_hit_count
```

After the pipeline runs, `counters_post.parquet` has rows like:

```
declared_language  url_host_tld  crawl_year  count  sum_text_len  mean_language_score  pii_hit_count
en                 com           2025        82_341 1_201_032_104 0.991                17
en                 edu           2025         9_207   109_184_332 0.994                 0
sw                 ke            2025           443     2_118_002 0.812                 1
sw                 tz            2024           112       381_204 0.766                 0
om                 et            2023            28        92_104 0.602                 0
...
```

From this file, every representativeness report is a one-line query:

```python
import polars as pl
d = pl.read_parquet("audit/counters_post.parquet")

# Slice distribution by language
slice_lang = (d.groupby("declared_language")
                .agg(pl.sum("count").alias("docs"))
                .with_columns((pl.col("docs") / pl.col("docs").sum()).alias("p_slice")))

# Target distribution (e.g. Wikipedia article counts as reference)
target_lang = pl.read_parquet("reference/wikipedia_article_counts.parquet") \
    .with_columns((pl.col("articles") / pl.col("articles").sum()).alias("p_target"))

# Divergence
joined = slice_lang.join(target_lang, on="declared_language", how="outer")
tvd = 0.5 * (joined["p_slice"] - joined["p_target"]).abs().sum()
```

Record the result as a measure:

```yaml
- measure_id: m.representativeness.language.v1
  characteristic: completeness   # population coverage is a 25012 completeness proxy
  measure_5259_2: representativeness_vs_target_population
  proxy: total_variation_distance_vs_wikipedia_article_counts
  reference_source: "https://dumps.wikimedia.org/.../article_counts-2026-10-01.parquet"
  reference_sha256: 9dd4…
  value: 0.187
  threshold: "<= 0.25"
  pass: true
  disaggregated_by: [declared_language]
  evidence_sha256: 8a1c…
```

## Temporal distribution — the under-measured dimension

Temporal skew is the single most common representativeness bug nobody checks. A pipeline that crawls Common Crawl with no date filter will produce a slice whose crawl-date distribution looks like Common Crawl's — heavily biased toward recent snapshots and, for very old content, toward URLs that have survived (survivorship bias in the content as well). The temporal story is:

- **Document crawl date** — when CC captured the page. Use `crawl_year-month` bucket.
- **Document publication date** — when the page was first published. Harder to obtain; parse from HTML meta tags (`article:published_time`, `datePublished` from JSON-LD, OpenGraph `og:updated_time`) with best-effort fallbacks. Record parse success rate.
- **Event-time distribution** — for news or event-driven content, when the described event occurred. Only possible via NER + temporal expression resolution; usually out of scope for a bulk corpus audit.

Report at least crawl-date distribution; attempt publication-date on a sample; flag the gap otherwise.

For labelled datasets ([`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md)), the temporal dimension is *annotation date*: when was the label assigned? Labels can drift as taxonomy or product definitions change; the audit catches it.

## Geography — handle with care

Geographic representativeness is a legitimate measure but a dangerous one to compute wrong. Three principles:

1. **Never infer per-document geography and store it per-document.** The inference is noisy, and storing it attaches a speculative demographic label to real data. Compute per-cell aggregates and store only those.
2. **Use multiple proxies and report the disagreement.** TLD-based inference (`.ke` → Kenya), WHOIS-registrant-country (where available and legal to query), server-IP geolocation, `og:locale` parsing, and author metadata each give different answers. Report distribution per proxy; do not blend them without a documented method.
3. **Record and respect regional ethics / legal frames.** Scraping geographic metadata is subject to the same opt-outs and licence terms as the content ([`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)); per-user geolocation inferred from content may have GDPR implications ([`mod-101`](../mod-101-ai-data-governance-and-position/README.md), [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)).

## Topic / content domain — a classifier with its own quality bar

Topic coverage is measured with a topic classifier (e.g. a small supervised model trained on a corpus labelled with the ACM / IPTC / DMoz-style hierarchy, or an unsupervised topic model like BERTopic). Treat the classifier's output as an *imperfect proxy* and record its version and evaluation metrics as part of the measure. Report the slice's topic distribution; if a target distribution exists (the downstream use case has a known topic mix), report the divergence.

Caution: topic classification on a 10 TB slice is expensive. Sample before classifying — 100k documents stratified by language gets you enough signal for the representativeness report.

## The representativeness report as an artifact

The report is a Markdown document plus a machine-readable JSON, both referenced from the per-slice quality report (Chapter 08):

```
audit/
  representativeness.md             # human-readable per-dimension summary
  representativeness.json           # the measure records
  distribution.parquet              # per-cell counts and metrics (the primary artifact)
  reference_distributions/
    wikipedia_lang_counts.parquet   # pinned reference distribution for languages
    datacomp_lm_tld.parquet         # pinned reference distribution for TLDs (illustrative)
    ...
```

The `representativeness.json` is a list of measure records:

```yaml
measures:
  - measure_id: m.representativeness.language.v1
    characteristic: completeness
    measure_5259_2: representativeness_vs_target_population
    proxy: total_variation_distance
    dimensions: [declared_language]
    value: 0.187
    threshold: "<= 0.25"
    pass: true
    reference:
      name: wikipedia_article_counts
      sha256: 9dd4…
    evidence:
      cell_count: 142
      suppressed_cell_count_below_threshold: 31
      per_cell_sha256: 1c80…
  - measure_id: m.representativeness.tld.v1
    characteristic: completeness
    measure_5259_2: dataset_level_diversity
    proxy: jensen_shannon_divergence
    dimensions: [url_host_tld]
    value: 0.094
    threshold: "<= 0.20"
    pass: true
    ...
  - measure_id: m.dataset_balance.language_x_tld.v1
    characteristic: consistency
    measure_5259_2: dataset_balance
    proxy: max_cell_concentration
    dimensions: [declared_language, url_host_tld]
    value: 0.63            # 63% of docs in some (lang, tld) cell live on one TLD
    threshold: "<= 0.70"
    pass: true
    disaggregated: true
```

## Interaction with Chapter 02–03 checks

The disaggregated counters produced in this chapter feed back into the row-level tools:

- GE custom expectations can read a per-cell file and assert per-language thresholds (e.g. `pct_sensible_in_lang_sw >= 0.4`).
- Soda Core's `group by` block runs disaggregated checks natively against the backend; use it when the disaggregation is one-dimensional.
- Deequ's `hasHistogramValues` asserts the shape of a categorical distribution directly on the Spark DataFrame, without pre-emitting a counter file.

The right assignment is usually: compute disaggregated counters during the pipeline run; load them into GE / Soda / Deequ for threshold checks at validation time.

## Pitfalls

- **Aggregating *before* disaggregating.** A slice-level mean language-score of 0.99 says nothing about low-resource languages. Always disaggregate first, then aggregate if needed.
- **Choosing a divergence measure after seeing the result.** Pick the metric before you compute, with a threshold. Post-hoc metric shopping is bias.
- **Reporting only the top N cells.** A healthy tail is part of the signal; summarise the tail (long-tail bucket) rather than dropping it.
- **Confusing "slice balance" with "target match".** Dataset balance is intrinsic (is one class 99% of the slice?); representativeness is extrinsic (does the slice look like the target?). Both are useful; they are different measures.
- **Pinning an outdated reference distribution.** Reference distributions have their own age. Record the reference's date and refresh cadence; if the reference is a Wikipedia dump, pin the date.
- **Reporting continuous dimensions as histograms without binning.** Document length "distribution" means nothing without a binning convention. Use log-scale buckets; record the bucket edges.

## Summary

- Aggregates hide subpopulation failures; every meaningful quality measure is disaggregated along the dimensions that matter: language, dialect, source domain, time, length, content topic.
- Emit a per-slice `distribution.parquet` with per-cell counts and metrics from the pipeline stages themselves (counters, not post-hoc). Intersectional cells (≥ 2D) are where failures hide.
- Representativeness is the divergence between the slice distribution and a *named, pinned* target distribution — TVD / JSD / KS / chi-square, picked per dimension, documented per measure.
- Temporal distribution is under-measured; report at minimum crawl-date buckets and attempt publication-date on a sample. Geography is a legitimate but ethically fraught measure; use multiple proxies and store only aggregates.
- Record disaggregated measures in the per-slice quality report (Chapter 08) alongside the `distribution.parquet` SHA-256. The row-level tooling (Chapters 02–03) consumes these counters to assert per-cell thresholds.

Chapter 07 moves from descriptive measures to causal ones: DataComp / DataComp-LM style ablation to prove which filter contributes which eval delta on a downstream model.
