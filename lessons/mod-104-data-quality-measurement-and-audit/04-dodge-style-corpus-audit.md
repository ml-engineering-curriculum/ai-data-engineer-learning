# Chapter 04 — Dodge-Style Corpus Audit: What Your Filters Strip, Who They Strip, and What They Leak

## Why this chapter exists

The checks from Chapters 02 and 03 answer *"do the rows in the slice conform to the schema and thresholds?"* — a necessary but shallow question. The deeper and more ML-specific question is: *"of the raw population the pipeline started with, what did we drop, and does the kept set look like the population we claim to represent?"* A corpus audit is the discipline that answers it.

The canonical reference is **Dodge, Sap, Marasović, Agnew, Ilharco, Groeneveld, Mitchell, Gardner, *Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus*, EMNLP 2021** (<https://aclanthology.org/2021.emnlp-main.98/>). The paper audits C4 — the text corpus that powers T5 — and establishes three axes every subsequent audit uses:

1. **What filters strip.** Characterise the content excluded by each filter (the C4 "bad words" filter, URL-based filters, length filters).
2. **Who the stripped content represents.** The demographic / topical / provenance characteristics of the excluded set — not just the kept set.
3. **Benchmark contamination.** Does the kept set contain documents from test splits of public benchmarks?

The paper's method is reproducible on your own slice, and the Dodge-style audit is now expected evidence in a dataset card (datasheets, [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) and in any public-interest or regulated release.

This chapter is the how-to. The deliverable is a **per-slice audit report** that covers all three axes with evidence you can point to.

## The audit's three axes

### Axis 1 — what the filters strip

A filter is a boolean function `keep(doc) ∈ {True, False}`. The slice is `{doc : keep(doc)}`. The audit is on the *complement*: `{doc : ¬keep(doc)}`.

For every filter in the pipeline:

- **Drop rate.** Number and percentage of raw records removed.
- **Marginal drop rate.** The drop rate attributable to *this* filter, not to filters upstream. Order-dependent; record the pipeline order.
- **Dropped-example sample.** A reservoir sample (say, 1,000 docs) of exactly what this filter dropped, with reasons. The sample is the human-readable evidence a reviewer actually consumes.
- **Dropped-distribution profile.** TLDs, languages, document lengths, crawl dates of the dropped set — aggregated per-filter, in the same shape as the kept set.

For C4, Dodge et al. show the "bad words" blocklist (originally sourced from the LDNOOBW list, <https://github.com/LDNOOBW/List-of-Dirty-Naughty-Obscene-and-Otherwise-Bad-Words>) removes content well beyond its stated intent — documents discussing sexuality or anatomy non-sexually, documents from minority communities, documents containing medical or legal terminology that triggers the list. Your pipeline's blocklist will do the same thing differently; the audit is how you know.

### Axis 2 — who the stripped content represents

This is the *representativeness-of-the-exclusion* axis and the one most pipelines get wrong. Having measured *what* was dropped, measure *who*:

- **Source-domain demographics**. Which hosts, which TLDs, which geographies (via IP or declared contact info where available) are over-represented in the dropped set vs. the kept set.
- **Topical / content-category skew**. Topic model or classifier labels (medical, legal, dialectal, LGBTQ+-related, minority-language) applied to the dropped sample; compare proportions to the kept set.
- **Dialect / sociolect skew** (for text corpora). Dodge et al. found the C4 blocklist disproportionately drops documents written in African-American English (AAE). This is a canonical example you should assume applies in some form to your pipeline until you measure.
- **Language / script skew** (for multilingual slices). See Chapter 05 on Kreutzer "Quality at a Glance" for the discipline specific to this.

The axis is not an opinion. It is a set of **counters disaggregated by the dimension of interest**, produced by Chapter 06's machinery.

### Axis 3 — benchmark contamination

The third axis asks whether the kept set contains documents from the test or dev splits of public evaluation benchmarks. If it does, downstream reported eval numbers are inflated because the model saw the "test" examples during pretraining.

Dodge et al. check C4 for exact-match overlaps against the input of SuperGLUE, GLUE, LAMBADA, and others; they find non-trivial contamination at the input level (and therefore unknown contamination at the full `(input, label)` level). This chapter introduces the audit; [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) is the deep dive on detection and decontamination. For the audit, the minimum is:

- A fixed list of benchmarks with their test / dev splits and known public URLs where their inputs appear verbatim (many benchmarks are *themselves* scraped from the web).
- A per-slice contamination report with hit counts, hit rates, and URL evidence.

## Methodology — Dodge et al.'s audit, operationalised

Reproduce the paper's shape on your slice. The four steps:

**Step 1. Freeze the pipeline inputs and outputs.** You need the raw population *and* the final slice. If you threw away the raw records after filtering, the audit is impossible. Keep at least the `(url, warc_record_id)` tuple of every raw record — the index is cheap; the text can be re-fetched by byte-range.

**Step 2. Attribute every drop to a specific filter.** Instrument your pipeline so every dropped record emits a `(record_id, filter_name, reason, at)` event. Chapter 06's counter infrastructure consumes these events; the Dodge-style report consumes the same events grouped differently.

**Step 3. Sample the dropped set per filter.** Reservoir-sample 500–2000 dropped records per filter. The sample is the auditable evidence a human reviewer reads. Save the raw record alongside its filter attribution.

**Step 4. Profile the sample.** For each filter's sample:

- Count TLDs, languages, document lengths, crawl dates.
- Apply a topic / dialect classifier (see "Dialect analysis" below) and report the distribution.
- Spot-check 50–100 records by hand and classify the reason into a per-pipeline taxonomy (e.g. "legitimate medical", "legitimate minority-community", "legitimate non-English", "boilerplate", "machine-translated", "duplicate of another doc", "actual low-quality").

Your audit report has one section per filter with a **kept vs. dropped** comparison on each axis.

## Dialect analysis — the hardest part of Axis 2

For English text, Blodgett, Green, O'Connor, *Demographic Dialectal Variation in Social Media: A Case Study of African-American English* (EMNLP 2016, <https://aclanthology.org/D16-1120/>) introduced a probabilistic dialect classifier that assigns per-document posterior probabilities over `{AAE, white-aligned, Hispanic-aligned, Asian-aligned, other}`. Dodge et al. use this classifier to measure the AAE-skew of C4's drops. Code lives at <https://github.com/slanglab/twitteraae> (Twitter-trained; cite methodology when applying to non-Twitter text).

Operational guidance:

- Use the classifier on the **dropped set** and the **kept set** separately. Report posterior means and distributions for each.
- Treat classifier output as a **proxy**, not as a label. It is trained on Twitter and over-predicts on long-form text. Record the classifier version, the training corpus, and the posterior-threshold convention in your audit.
- Do not publish per-document predictions as factual demographic labels. The audit uses the posteriors in aggregate.
- For languages other than English, substitute a dialect classifier for that language if one exists; otherwise, note the gap explicitly in the audit (`<!-- needs-research: no validated dialect classifier for X; audit uses language-ID + script heuristics only -->`).

## Benchmark-contamination audit (preview of mod-106)

The minimum viable contamination audit uses **exact-string matching** on benchmark inputs:

1. Build an index of test-split inputs from a curated list of benchmarks: GLUE / SuperGLUE / LAMBADA / HellaSwag / MMLU / GSM8K / HumanEval / BIG-bench, plus any task-specific evaluations your downstream model team uses. Pin the specific benchmark versions.
2. For each slice document, compute whether it contains any benchmark input as a substring. 13-gram or longer substring matches at 0% error, per the Dodge et al. and later [Brown et al. 2020, GPT-3, Appendix C] (<https://arxiv.org/abs/2005.14165>) conventions. <!-- needs-research: confirm the exact n-gram length and mismatch tolerance used by your downstream training team; 8-gram, 13-gram, and 50-gram conventions all exist. -->
3. Emit a contamination report: `(benchmark, test_split_version, n_matches, hit_rate, example_matches)`.

For a thorough contamination story, see [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md). The Dodge audit at this module's scale uses exact-match as a floor; the deeper near-dup / paraphrase detection lives in mod-106.

## What a Dodge-style audit report looks like

Structure the report as a Markdown document the data-governance reviewer can read top-to-bottom, with structured JSON sidecars for machine consumption:

```
audit/
  dodge_audit.md                          # the readable report
  dodge_audit.json                        # the structured summary
  per_filter/
    url_blocklist/
      sample.jsonl.gz                     # 1000 reservoir-sampled dropped docs
      profile.json                        # TLD, length, dialect distributions
    bad_words/
      sample.jsonl.gz
      profile.json
    language_id/
      sample.jsonl.gz
      profile.json
    ...
  contamination/
    report.json                           # per-benchmark hit counts
    hit_examples/
      superglue_boolq_dev_hit_0001.json
      ...
```

The `dodge_audit.json` is the file Chapter 08 ingests into `per_slice_quality.yaml`. Its minimal schema:

```yaml
dodge_audit:
  slice_id: cc-en-small-v1
  audit_date: 2026-11-04
  raw_population:
    record_count: 1_000_000
    tld_distribution_sha256: 2f4a…
    language_distribution_sha256: 9bc1…
  kept_population:
    record_count: 428_317
  filters:
    - name: url_blocklist
      drop_count: 211_433
      drop_rate: 0.211
      sample_count: 1_000
      sample_sha256: 8a1c…
      profile:
        top_tlds_dropped: [...]
        dialect_posterior_mean: {...}
    - name: bad_words
      drop_count: 102_880
      drop_rate: 0.129
      sample_sha256: ...
      profile:
        ...
    ...
  contamination:
    - benchmark: superglue
      split: dev
      version: "1.0"
      n_matches: 42
      hit_rate: 9.8e-05
      hits_sha256: ...
```

## Tooling — which pieces exist off-the-shelf

- **Dolma toolkit** (<https://github.com/allenai/dolma>) emits per-filter drop counts and samples as part of its pipeline; use those as the Axis-1 primitives if your pipeline is dolma-based.
- **Hugging Face `datatrove`** (<https://github.com/huggingface/datatrove>) similarly produces per-step statistics; the `StatsLogger` is your Axis-1 substrate on a FineWeb-style pipeline.
- **`twitteraae` dialect classifier** (<https://github.com/slanglab/twitteraae>) — the Blodgett et al. classifier. Pin a release and record the SHA.
- **EleutherAI LM Evaluation Harness decontamination recipes** (<https://github.com/EleutherAI/lm-evaluation-harness>) — the current practical reference for the contamination part. [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) covers in depth.
- **Hugging Face `datasets`** and **`lm-eval`** benchmark downloads — the `(benchmark, version)` resolver for contamination-audit inputs.

For your first audit, write a short orchestrator in Python that:

1. Reads the pipeline's drop-event log.
2. Reservoir-samples per-filter.
3. Runs the dialect classifier on each sample.
4. Runs the contamination check across a pinned benchmark list.
5. Emits `dodge_audit.md` and `dodge_audit.json`.

Running this on a 1M-document slice should take under an hour on a workstation.

## Pitfalls

- **Dropping records without logging the reason.** If filters don't emit per-drop events, the audit is a reconstruction exercise. Instrument at the filter, not after the fact.
- **Using the kept set as the "control".** The right comparison for Axis 2 is *dropped vs. raw population* or *dropped vs. kept*. The *kept* set is the one your filters already shaped; treating it as the baseline hides the effect.
- **Treating classifier outputs as ground-truth labels.** Dialect classifiers are noisy; publish posteriors, not labels.
- **Running contamination audit against the latest benchmark version only.** Benchmarks are versioned; the model team may be evaluating against an older split. Audit against *all* versions they will evaluate on, pinned.
- **Omitting benchmarks the model team will add later.** The contamination audit's benchmark list is a living asset. Keep it in the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) so the next audit updates alongside eval decisions.
- **Audit-once syndrome.** The audit is not a one-shot document. Every slice gets one; see [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) for the monitor that re-runs it on cadence.

## Summary

- A Dodge-style audit answers three questions no row-level check can: *what did the filters strip*, *who does the stripped content represent*, *does the kept set contaminate public benchmarks*.
- Instrument the pipeline to emit per-drop `(record_id, filter_name, reason)` events; reservoir-sample the dropped set per filter; profile the sample on TLD, length, language, dialect.
- Use a validated dialect classifier (e.g. Blodgett et al. 2016 for English) for Axis 2. Publish posteriors in aggregate, never per-document labels.
- For Axis 3, exact-match on test-split inputs against a pinned benchmark list is the floor; deeper near-dup detection is [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md).
- The audit is a `dodge_audit.md` plus `dodge_audit.json` plus a per-filter sample directory. The JSON lands in the per-slice quality report (Chapter 08).

Chapter 05 turns to the multilingual case — the Kreutzer "Quality at a Glance" discipline for auditing low-resource and multilingual slices, where the Dodge axes alone do not catch the dominant failure modes.
