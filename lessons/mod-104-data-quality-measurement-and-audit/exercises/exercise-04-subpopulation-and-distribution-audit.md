# Exercise 04 — Subpopulation and Distribution Audit

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 05 — Kreutzer "Quality at a Glance"](../05-kreutzer-quality-at-a-glance.md), [Chapter 06 — Representativeness and disaggregated counters](../06-representativeness-and-disaggregated-counters.md)

## Objective

Measure the slice's **subpopulation balance** and **representativeness vs. a named target distribution** across at least four dimensions — language, source TLD, temporal (crawl or publication date), and document length — using disaggregated counters, not aggregates. Where the slice is multilingual, run a Kreutzer-style human-rating audit on a per-language stratified sample for at least three languages (one each of: high-resource, mid-resource, and — where you can recruit the rater — low-resource).

By the end you will have:

- A `distribution.parquet` with per-cell counts and metrics, computed during a pipeline pass (not post-hoc from the final table).
- A `representativeness.json` with named divergence measures (TVD / JSD / KS / chi-square) against pinned reference distributions.
- A `kreutzer_audit.json` with per-language `pct_correct`, `pct_sensible_in_lang`, error taxonomy, and rater-pool metadata for at least three languages.
- A short `representativeness.md` summary suitable for a dataset card appendix.

## Prerequisites

- A slice with ≥ 100k documents, ideally multilingual (if you only have English, you may still complete the distribution half; the Kreutzer half degrades to a "representativeness of a single language" exercise, which is a weaker learning outcome — see §"Stretch goals").
- Chapter 06 read. You should be able to write a `DisaggCounter` or an equivalent Spark `groupBy` aggregation.
- Pinned reference distributions:
  - **Language**: Wikipedia article counts per language at a pinned dump date, or another named reference. <!-- needs-research: The Wikipedia Statistics pages publish article counts; pin the exact dump date and URL you retrieve. -->
  - **TLD**: a reference TLD distribution of your choice (e.g. Common Crawl's own TLD distribution for the snapshot, or the DCLM-Baseline TLD proportions). Pin the source and SHA-256.
  - **Temporal**: a target cadence from your pipeline's own spec (e.g. "last 36 months, roughly flat").
  - **Length**: pick a target log-scale bucket distribution; a simple "approximately log-normal between 200 and 20k characters" is acceptable if the slice spec does not define one.
- For the Kreutzer half: at least three L1-speaker raters, one per language. Chapter 05 §"Step 2 — recruit raters" has the pragmatic paths.

## Problem statement

Build a full representativeness report for a slice:

1. Emit `distribution.parquet` with cells `(declared_language, url_host_tld, crawl_year_quarter, text_length_bucket)` and per-cell metrics `(count, sum_text_len, mean_language_score)`.
2. For each dimension, compute the divergence vs. the pinned reference. Use the divergence appropriate to the dimension type (TVD / JSD for categorical, KS for ordinal).
3. For multilingual slices, run a Kreutzer-style audit on at least three languages.
4. Produce `representativeness.md` plus `representativeness.json` plus `kreutzer_audit.json`.

## Starter guidance

1. **Add a `DisaggCounter` to the pipeline.** Chapter 06 §"The counter infrastructure" has the pattern. Emit the counter during the final filter pass so you do not re-scan the slice. If the slice is Spark-native, skip the per-doc counter and use `groupBy(dims).agg(...)` with Deequ's `hasHistogramValues` for the pass/fail threshold check.

2. **Pick your bucketing conventions before computing.** Crawl date → year-quarter. Text length → log-scale buckets `(0, 200), (200, 500), (500, 1k), (1k, 2k), (2k, 5k), (5k, 10k), (10k, 20k), (20k, 50k), (50k, ∞)`. TLD → raw TLD for high-frequency; a `__long_tail__` bucket for TLDs below a per-slice threshold. Record all bucket edges in `representativeness.json`.

3. **Compute the divergences.** Choose per-dimension:

   - *Language* (categorical, bounded cells): Total Variation Distance against the Wikipedia reference. Also report the KL divergence in both directions as a sanity check.
   - *TLD* (categorical, long tail): JSD against the pinned TLD reference.
   - *Temporal* (ordinal, time-series): Kolmogorov–Smirnov statistic on the empirical CDFs of the slice and reference.
   - *Length* (ordinal, long tail): KS on the log-length empirical CDF.

   For each, record the threshold and pass/fail. Your thresholds are project-dependent; defaults might be TVD ≤ 0.25, JSD ≤ 0.20, KS ≤ 0.15. Document the choice.

4. **Intersectional cells.** Compute at least one 2D divergence — `(declared_language × url_host_tld)` — and report the maximum cell concentration (`max_cell_concentration = max(cell_count) / total_count_in_language`). Chapter 06 §"Intersectional cells are where the failures hide" is the reference. A healthy slice has no language whose documents are 90%+ concentrated on one host.

5. **Kreutzer audit.**

   - Stratify-sample 50–100 documents per language for your three languages. Record the sampling seed.
   - Give each rater a sheet in their language: `doc_id`, `label ∈ {C, CS, CB, WL, NL, NS, P}`, `if_WL_observed_language`, `notes`.
   - Double-rate at least 20% of each language's sample for Cohen's κ.
   - Compute per-language `pct_correct = P(C ∨ CS ∨ CB)`, `pct_sensible_in_lang = P(C)`, error-taxonomy breakdown, and the binomial CI95 on `pct_correct`.
   - Emit `kreutzer_audit.json` in the Chapter 05 §"Attach to the slice" shape.

6. **Write the summary.** `representativeness.md` opens with a one-paragraph verdict, lists the dimensions with their divergence values and pass/fail, highlights intersectional findings, and lists any language gaps.

## Deliverables

```
exercise-04/
  audit/
    distribution.parquet
    bucket_edges.json
    representativeness.json        # per-dimension divergences
    representativeness.md
    references/
      wikipedia_lang_counts.parquet
      wikipedia_lang_counts.sha256
      tld_reference.parquet
      tld_reference.sha256
      # … others as applicable
    kreutzer/
      sampling_seed.json
      raters.yaml                  # rater proficiency, pay rate, double-rate assignments
      per_language/
        en/
          rating_sheet.parquet
          double_rated_subset.parquet
          agreement_kappa.json
        es/
          ...
        sw/                        # or whichever third language you picked
          ...
      kreutzer_audit.json
  code/
    counters.py
    divergence.py
    kreutzer_runner.py             # tooling-agnostic; wraps Label Studio / Prodigy / Argilla
    render_representativeness_md.py
  README.md
```

## Acceptance criteria

- `distribution.parquet` has cells along at least four dimensions with per-cell counts and at least two per-cell metrics.
- Each dimension has a named divergence (TVD / JSD / KS / chi-square) with a threshold and pass/fail recorded in `representativeness.json`, against a pinned reference with a SHA-256.
- At least one 2D intersectional divergence is reported; the `max_cell_concentration` metric is computed and discussed in `representativeness.md`.
- `kreutzer_audit.json` covers at least three languages with sample size ≥ 50 each. Each language has `pct_correct`, `pct_sensible_in_lang`, error taxonomy, binomial CI95, and a rater-pool metadata block (size, κ on double-rated subset).
- Any language without an accessible rater pool is listed in `kreutzer_audit.gaps[]` with a reason; the dataset card for the slice does not claim release-readiness in that language.
- Rater pay rates and proficiency attestations are in `raters.yaml`.
- `representativeness.md` opens with a one-paragraph verdict and ends with a prioritised follow-up list.
- Any claim in `README.md` that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Compute a 3D divergence** `(language × TLD × crawl_year_quarter)`. Find the smallest cell with ≥ 1% of the slice and discuss what the slice looks like inside it.
- **Reference-distribution sweep.** Rerun the divergence computation against *two* reference distributions per dimension (Wikipedia and CC's own TLD distribution). Report both and discuss which is appropriate for your slice's intended use.
- **English-only fallback.** If you only have an English slice, run a sociolect audit by crawl-date bucket instead of a Kreutzer language audit — sample 50 documents per bucket, hand-rate on `{formal prose, informal prose, technical, user-generated, boilerplate}`, and report the shift across time. The methodology is Kreutzer-shaped even without multiple languages.
- **Hook the audit into CI.** Add a `justfile` target that runs `verify_representativeness.py --slice <id> --fail-on-threshold-breach`. The slice release gates on it.
- **Sign the artifacts.** Sigstore-sign `representativeness.json` and `kreutzer_audit.json`. Chapter 08 §"Signing and chain of custody" is the reference.

## What this exercise is not

It is not a decontamination audit (Exercise 03 / [`mod-106`](../../mod-106-benchmark-contamination-and-decontamination/README.md)) and it is not an ablation (Exercise 05). It is the descriptive half of the slice's quality picture: *what does the slice look like, and does its shape match what we claimed it would*.
