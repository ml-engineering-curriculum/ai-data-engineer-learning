# Exercise 03 — Dodge-Style Corpus Audit on One Slice

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 04 — Dodge-style corpus audit](../04-dodge-style-corpus-audit.md); [Chapter 06 — Representativeness and disaggregated counters](../06-representativeness-and-disaggregated-counters.md) for the counter machinery used in Axis 2

## Objective

Run a Dodge-style corpus audit (Dodge et al. 2021, *Documenting Large Webtext Corpora*) on a web-text slice — ideally the output of [`mod-103` Exercise 01](../../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md). Produce the three audit axes from Chapter 04:

1. **What the filters strip** — per-filter drop rate, marginal drop rate, sampled dropped examples, dropped-set profile.
2. **Who the stripped content represents** — dropped-set profile on language, TLD, length, and (for English text) dialect posterior from a validated classifier.
3. **Benchmark contamination** — exact-match hits of your kept slice against test-split inputs of a small benchmark panel.

The deliverable is a `dodge_audit.md` and `dodge_audit.json` that would plug into the per-slice quality report (Chapter 08) unchanged.

## Prerequisites

- A slice with at least ~1M raw records and a reachable raw population (you can re-fetch dropped records if the raw table is missing). If you came from mod-103 Ex 01, your per-record manifest gives you this for free.
- Pipeline instrumentation that emits one `(record_id, filter_name, reason)` event per dropped record. If your `mod-103` Ex 01 pipeline does not, add the hook before starting this exercise — a side-effect `drop_log.write(...)` in each filter is enough.
- Python 3.11+ with `polars` or `duckdb`, `datasets`, `datasketch` (if you want MinHash hints for contamination), and a dialect classifier pinned to a specific release. For English, the Blodgett et al. 2016 twitteraae classifier (<https://github.com/slanglab/twitteraae>, with vocabulary and count tables under `model/`) is the paper's reference. For other languages, see Chapter 04 §"Dialect analysis".
- A pinned list of benchmark test / dev inputs. Use the Hugging Face `datasets` revisions for GLUE, SuperGLUE, HellaSwag, MMLU, GSM8K, and TriviaQA at a fixed commit. Record the dataset revision in `audit/benchmarks_pinned.yaml`.

## Problem statement

Audit a single pretraining slice along all three Dodge axes. Emit the full audit artifact tree from Chapter 04 §"What a Dodge-style audit report looks like". The audit must:

- Attribute every drop to exactly one filter (the first filter that rejected the record wins).
- Reservoir-sample at least 500 dropped records per filter and profile them.
- Apply a validated dialect classifier to English-labelled samples in both the dropped and kept sets. Publish posteriors in aggregate only — no per-document predictions in the public report.
- Run an exact-string benchmark-contamination check against a pinned panel of at least four benchmarks at a 13-gram-or-greater match length (document your mismatch tolerance explicitly).

## Starter guidance

1. **Instrument the pipeline to emit drop events.** If `mod-103` Ex 01's pipeline does not, add a sink:

   ```python
   drop_sink = JsonlSink("audit/drop_events.jsonl.gz")

   def url_blocklist(doc):
       if should_block(doc["url"]):
           drop_sink.write({"record_id": doc["id"], "filter": "url_blocklist",
                            "reason": "tld_in_blocklist", "at": now()})
           return False
       return True
   ```

2. **Attribute marginal drops.** Each record is dropped by exactly one filter — the first one that rejected it in pipeline order. `marginal_drop_rate[filter] = count(filter) / total_raw`. Record the pipeline order in `audit/pipeline_order.json` so the next revision of the audit is comparable.

3. **Reservoir-sample per filter.** Algorithm R or `random.sample` on the drop-event log stratified by filter. Save the sampled records' raw text to `audit/per_filter/<filter>/sample.jsonl.gz`.

4. **Profile the sample.** For each per-filter sample, compute: TLD distribution, length distribution, language distribution (re-run `lid.176`), and dialect posterior distribution (English only — see §"Dialect analysis" below). Emit `audit/per_filter/<filter>/profile.json`.

5. **Dialect analysis.** On the English-labelled samples (kept and per-filter dropped), run the Blodgett et al. twitteraae classifier. Record:

   - Classifier version (git SHA), training corpus, posterior-threshold convention.
   - Per-sample posterior means with 95% CI.
   - *No per-document labels in the public report.* Store per-document posteriors locally for your own verification but do not publish them.

6. **Benchmark contamination.** Build an index of exact-string benchmark inputs (test/dev splits for four or more benchmarks from your pinned list). For each slice document, compute whether any 13-gram-or-greater substring matches a benchmark input. Emit `audit/contamination/report.json` with `(benchmark, split, version, n_matches, hit_rate, sampled_hits)`.

7. **Assemble the audit report.** Follow the Chapter 04 §"What a Dodge-style audit report looks like" structure. `dodge_audit.md` is the narrative; `dodge_audit.json` is the machine-readable summary.

## Deliverables

```
exercise-03/
  audit/
    pipeline_order.json
    drop_events.jsonl.gz
    per_filter/
      url_blocklist/
        sample.jsonl.gz
        profile.json
      bad_words/
        sample.jsonl.gz
        profile.json
      language_id/
        sample.jsonl.gz
        profile.json
      perplexity/
        sample.jsonl.gz
        profile.json
      minhash_dedup/              # optional at this scope
        sample.jsonl.gz
        profile.json
    dialect_classifier/
      classifier_meta.json        # version, training corpus, threshold convention
      posterior_summary.json      # per-sample posterior means + CI
    contamination/
      benchmarks_pinned.yaml
      report.json
      hit_examples/
        <benchmark>_<split>_<nnn>.json
    dodge_audit.md
    dodge_audit.json
  code/
    emit_drop_events.py
    reservoir_sample.py
    profile_sample.py
    dialect_posterior.py
    contamination_scan.py
    render_audit_md.py
  README.md
```

## Acceptance criteria

- Every surviving record in the slice has a documented chain of custody: either it passed every filter, or there is exactly one filter in `drop_events.jsonl.gz` that rejected it.
- Each `per_filter/<filter>/profile.json` includes TLD, length, language, and (for English-labelled samples) dialect posterior distributions.
- `dialect_classifier/classifier_meta.json` pins the exact model / code revision and documents the limitations (training corpus, generalisation caveats).
- No per-document dialect labels appear in any file under `audit/` that is not strictly internal.
- `contamination/report.json` covers at least four benchmarks pinned by dataset revision, uses 13-gram-or-greater matching, documents the mismatch tolerance, and includes at least one sampled hit example per benchmark that had a hit.
- `dodge_audit.json` validates against the Chapter 04 §"What a Dodge-style audit report looks like" shape (minimal schema).
- `dodge_audit.md` opens with a one-paragraph summary suitable for a dataset card and closes with a prioritised follow-up list.
- Any claim in `README.md` that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Multilingual audit.** If your slice has more than one language, run the Dodge audit per language and emit `audit/per_filter/<filter>/per_language/<lang>/`. Pair with Exercise 04's representativeness audit for a complete multilingual picture.
- **Add a Kreutzer-style human rating** (Chapter 05) on the kept English sample. Compare the automated dialect posteriors to the human `C/CS/CB/WL/NL/NS/P` labels on the same documents.
- **Scale to near-dup contamination.** Replace the exact-match contamination step with a MinHash + LSH against the benchmark panel using `datasketch`. Record the Jaccard threshold and compare hit counts to the exact-match floor. Previews [`mod-106`](../../mod-106-benchmark-contamination-and-decontamination/README.md).
- **Emit a signed audit bundle.** Sign `dodge_audit.json` with Sigstore keyless signing. Chapter 08 §"Signing and chain of custody" is the reference.
- **Run the audit against two filter orderings** and compare the marginal drop rates. The attribution shifts when you move a filter earlier or later in the pipeline; the audit should expose that.

## What this exercise is not

It is not a Kreutzer-style multilingual audit (that is Exercise 04's remit in combination with Chapter 05). It is not a decontamination step — detecting a benchmark hit here does not remove it from the slice; that is [`mod-106`](../../mod-106-benchmark-contamination-and-decontamination/README.md). It is not an ablation — the KEEP / REMOVE verdict on a filter requires Exercise 05.

What it *is*: the shape of an audit you can show to a governance reviewer, a journalist, or a partner model team and have them see the slice's filters, its exclusions, and its benchmark overlaps at a glance.
