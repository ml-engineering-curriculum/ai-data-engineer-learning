# Chapter 07 — Subpopulation Balance Drift

## Why this chapter exists

A slice can pass every overall quality and drift check and still mis-serve a subpopulation. A corpus whose English share drifted from 93% to 97% between refreshes looks fine on average — overall length distributions are fine, overall license mix is fine, overall representativeness against the *reference* is "within band" — while Spanish dropped from 2.5% to 0.4% and Arabic from 1.1% to 0.2%. A model retrained on that slice will degrade on Spanish and Arabic inputs by an amount proportional to the drop. Nobody will notice until the user-satisfaction signal from the two markets arrives six weeks later.

**Subpopulation balance drift** is the monitoring axis that catches this. It is not the same as data drift (Chapter 05): data drift asks *has the distribution shifted overall?*, while balance drift asks *has the composition across the axes that matter shifted?*. The disaggregated counters from [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) Chapter 06 are the inputs; this chapter wires them into the monitoring contract.

## Why the signal matters even when the overall is fine

A rigorous way to say it: *overall distribution* statistics are marginals. They collapse across the axes you care about (language, geography, demographic proxies, licensing posture, source family). Two distributions can have identical marginals and completely different conditional structure.

Two empirical findings to cite in a design review:

- **Buolamwini & Gebru 2018, FAT\* (Gender Shades).** When face-analysis systems were evaluated on *skin-tone × gender* subgroups rather than on the overall population, error rates for darker-skinned women were up to 34 percentage points higher than for lighter-skinned men. The overall-accuracy metric hid the gap. <https://proceedings.mlr.press/v81/buolamwini18a.html>.
- **Dodge et al. 2021, EMNLP (Documenting Large Webtext Corpora).** C4's blocklist-based filtering silently removes text about and by marginalised groups at disproportionate rates. Overall token counts stayed in a plausible range; who was in the corpus did not. <https://aclanthology.org/2021.emnlp-main.98/>.

The NIST AI RMF (AI 100-1, MEASURE 2.11 and related sub-categories) and the EU AI Act (Article 10 §3 — "training, validation and testing data sets shall be relevant, sufficiently representative, and to the best extent possible, free of errors and complete… in view of their intended purpose") are both explicit that representativeness is a measurable, documented property of the training data. Subpopulation-balance monitoring is the way you produce that measurement on cadence rather than once.

## What counts as a subpopulation

The subpopulation axes you monitor are the ones you *disaggregate* on — the same axes mod-104 Chapter 06 taught you to disaggregate quality metrics along. Typical shapes for a text corpus:

- **Language** (ISO 639-1 / FastText lid.176 label).
- **Geographic origin / jurisdiction** (country-code from the source TLD, upstream crawl region, deployment target).
- **Licensing posture** (permissive / restrictive / unknown / opt-out-honoured).
- **Source family** (web crawl / books / code / academic / forums / synthetic).
- **Document register / style** (news / encyclopedic / conversational / technical — often a classifier-produced label).
- **Content-language variety where relevant** (e.g. en-US vs en-IN vs en-NG, if the pipeline resolves that signal).

For labelled data ([`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md)):

- **Labeller cohort** (which labelling team / vendor produced which share of annotations).
- **Task difficulty bucket** if the label workflow records it.

For benchmark / eval slices:

- **Benchmark category** (QA / summarisation / reasoning / safety / math).
- **Difficulty / length bucket**.

You do not monitor demographic categories the pipeline cannot *responsibly* observe. If the pipeline does not infer race, do not estimate it for monitoring — the proxy is worse than the signal. Monitor the axes you already compute.

The contract names its axes explicitly:

```yaml
# contract preamble
subpopulation_axes:
  - name: language
    source: fasttext-lid.176.label
    cardinality_note: "95%+ mass on 20 languages; long tail of 150+"
  - name: url_host_tld
    source: parsed from url
    cardinality_note: "~1200 TLDs; monitoring runs on top-50 + 'other'"
  - name: source_family
    source: pipeline-assigned
    enum: [cc, books, code, arxiv, forums, synthetic]
  - name: license_posture
    source: pipeline-assigned
    enum: [permissive, restrictive, unknown, opt_out_honoured]
```

Every monitored balance metric cites one of these axes. The axes are versioned alongside the schema; changing an axis is a contract bump.

## The measures — population stability, Jensen-Shannon, chi-square

Balance drift is the same mathematical object as the categorical-drift case in Chapter 05, applied to the subpopulation axis.

- **PSI over the axis.** Treat each category on the axis (e.g. each language) as a bin; compute PSI between the reference slice's subpopulation distribution and the current slice's. PSI thresholds inherit from Chapter 05 (< 0.1 stable, 0.1–0.25 moderate, > 0.25 significant).
- **Jensen-Shannon divergence over the axis.** Alternative to PSI; symmetric, bounded, less sensitive to small-count smoothing. Use JSD when the axis has a long tail.
- **Chi-square on the contingency table.** Treat the (axis × slice version) table as contingency; test for independence. Report the χ² statistic + effect size (Cramér's V), not just the p-value — the p-value collapses at corpus scale (Chapter 05).
- **Cell-level alerting.** For axes you care about per-cell (e.g. "did Spanish share drop by more than 25% relative?"), compute per-cell relative deltas and alert on the worst cell.

For a two-dimensional balance surface — e.g. `language × url_host_tld` — you can monitor either:

- The **marginals** independently (easier, misses interactions).
- The **joint** (contingency table of the full cross); JSD / PSI applies but sparsity bites.

Monitoring the marginals plus one or two high-value joints is the pragmatic shape. The ones that matter most are usually (language × source_family) and (language × license_posture).

## Pre-filter vs post-filter: the Dodge frame

Mod-104 Chapter 04 introduced Dodge-style corpus auditing — look at *what the filters strip*, not just at *what the filter output looks like*. Subpopulation balance monitoring reuses that frame on two tracks:

1. **Post-filter vs reference.** Does the final slice's subpopulation balance match the pinned reference? This is the retrain-decision signal.
2. **Filter delta.** For each pipeline filter stage (quality filter, language filter, license filter, PII filter, dedup), *what did that stage remove, disaggregated along each axis?*. A filter whose per-language removal rate drifts over time is often where silent balance drift is born.

The second track is where the signal usually originates. Alerting only on post-filter composition means you miss the *cause* and spend hours identifying which filter moved. Monitor each filter stage's per-axis removal rate directly.

```yaml
# a filter-stage monitor
- id: m.population.quality_filter_removal_rate_by_language.v1
  kind: population_drift
  axis: language
  stage: quality_filter                       # which pipeline stage
  expression: >
    removal_rate_by_axis(
      pre=stage_input,
      post=stage_output,
      axis=language,
    )
  reference: historical_band_last_90d
  threshold:
    per_cell_relative_delta_abs_lte: 0.25     # alert if any language's removal rate moved >25% relative
  severity: advisory
  owner: ai-data-engineer@acme
  route: slack:#ai-data-eng-alerts
```

Over time, the filter-stage monitors become the *root-cause* fingerprinting layer. The post-filter monitor says the balance moved; the per-stage monitors say *where* it moved.

## The `population_balance.yaml` artifact

Each slice version emits a `population_balance.yaml` sidecar. It is the artifact the monitoring contract reads, and the one the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) summarises.

```yaml
# population_balance.yaml — per slice version
slice: cc_en.v3
slice_version: 2026-11-04T08:17:00Z
reference_slice: cc_en.v3.ref_2026-09-15
axes:
  - name: language
    kind: categorical
    distribution:
      en: 0.9472
      es: 0.0041
      fr: 0.0163
      de: 0.0125
      ja: 0.0055
      ar: 0.0019
      zh: 0.0045
      other: 0.0080
    reference_distribution:
      en: 0.9310
      es: 0.0257
      fr: 0.0168
      de: 0.0129
      ja: 0.0057
      ar: 0.0114
      zh: 0.0047
      other: 0.0118
    stats:
      psi: 0.138
      jsd: 0.019
      chi_square_stat: 4172.1
      cramers_v: 0.082
    cell_deltas:                                # relative change per cell
      en: 0.0174
      es: -0.8405
      ar: -0.8333
    flags:
      - level: retrain_blocking
        reason: "es cell delta -84% exceeds threshold (-25%)"
      - level: retrain_blocking
        reason: "ar cell delta -83% exceeds threshold (-25%)"
  - name: source_family
    …
  - name: license_posture
    …
pipeline_stage_removal_rates:
  quality_filter:
    by_language:
      en: 0.182
      es: 0.610
      ar: 0.724
      …
  license_filter:
    …
```

That sidecar is the *input* to the monitoring contract. Rows in `dataset_monitoring_contract.yaml` reference it:

```yaml
- id: m.population.lang_psi.v1
  kind: population_drift
  axis: language
  expression: population_balance[axes.name=language].stats.psi
  reference: golden
  cadence: on_slice_build
  threshold:
    lte: 0.1
  severity: retrain_blocking
  owner: ai-data-engineer@acme
  route: pagerduty:ai-data-eng
  evidence_artifact: population_balance/${slice_version}.yaml
```

## Fairness vs representativeness vs balance — three words, three jobs

The three terms get used interchangeably. Pin the distinctions:

- **Representativeness** is a measurement property: how well the training data represents the deployment / target population. Produced by mod-104's representativeness audit. *A property of a slice.*
- **Balance** is a composition property: the share of each subpopulation in the training data. Produced by the counters here. *A property of a slice.*
- **Fairness** is a model property: how the model's performance varies across subpopulations. Measured by the eval roles on the model's outputs. *A property of a trained model.*

A balanced slice is not automatically a fair model; an unbalanced slice is a strong predictor of an unfair model but not a determinant. You monitor balance because it's the lever you control from the data side; fairness evaluation routes to `ai-eval-engineer` / `model-evaluation-engineer`.

A common failure in incident narratives is to treat a balance-drift alert as *a fairness problem*. It is a *dataset-side input* to the fairness story, one of several. The handoff is covered in Chapter 09.

## When to alert publish-blocking vs retrain-blocking vs advisory

Guidance for picking severity:

- **publish-blocking.** Rare for balance drift. Reserve for cells with a contractual minimum (e.g. "regulated-market slice must contain ≥ 5% jurisdiction-X documents"). Most balance drift is slow and does not justify blocking the publish.
- **retrain-blocking.** The common severity. A material balance shift against the golden reference is the signal the retraining team needs to see on the dataset card before deciding to retrain.
- **advisory.** Appropriate for rolling-window balance drift (balance against last 4 weeks) and for filter-stage per-axis removal-rate drift. Both are triage inputs, not retraining gates.

## Tool notes

- **Evidently.** `ColumnSummaryMetric(column_name=…)` + `ColumnDriftMetric(column_name=…, stattest="psi" or "jensenshannon")` on the axis column gives you the one-dimensional balance story. For two-dimensional, generate a cross-tabulation column and reuse the same metric; or author a custom metric.
- **whylogs.** Frequent-items sketch + `calculate_drift_scores` gives you the categorical-drift story. For per-stage removal rates you compute the counts yourself (filtered-in / filtered-out) and whylogs-profile each side.
- **pandas / Polars.** For the cell-level relative-delta check, hand-rolled dataframe code is usually the shortest path. Keep it in a `population_balance.py` module that emits `population_balance.yaml`.
- **Great Expectations.** `ExpectColumnKLDivergenceToBeLessThan` is useful for PSI-shaped checks against a reference distribution; see [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) Chapter 02.

## A worked example — the Spanish-cell incident

Walk-through to anchor the chapter.

1. Slice `cc_en.v3` on 2026-11-04 completes. `population_balance.yaml` is emitted.
2. Overall data-drift monitor (`m.drift.language_jsd.v1`) measures JSD = 0.019 against the golden reference. Under the threshold of 0.02. **No data-drift alert fires.**
3. Subpopulation balance monitor (`m.population.lang_psi.v1`) measures PSI = 0.138 (moderate). Alert fires at retrain-blocking.
4. Cell-level check catches `es` cell dropped from 2.57% to 0.41% (relative delta -84%, exceeding -25% threshold).
5. Filter-stage monitor shows `quality_filter_removal_rate_by_language` went from 0.41 → 0.61 for Spanish. Root cause is in the quality filter, not in the upstream crawl.
6. Triage: the quality-filter heuristic was tuned against English-anchored proxies; recent crawl shipped more Mexican-Spanish forum text whose length / formatting matched the heuristic's "low-quality" pattern.
7. Decision: fix the quality filter to use a per-language threshold; re-run the slice; annotate the previous version-pin as `rejected: subpopulation_balance_drift(es)`.
8. Regression fixture authored: `f.es_forum_text_quality_filter.v1` is a 500-document Spanish-forum fixture that must pass the quality filter at the same per-language rate as English. Runs on every PR to the quality-filter stage.

The notable thing: the overall data-drift alert *did not* fire. If the pipeline only monitored overall drift, the Spanish regression lands silently.

## Summary

- Subpopulation balance drift is the monitoring axis that catches regressions overall-distribution drift misses.
- The axes you monitor are the ones you disaggregate on in mod-104: language, geography, source family, license posture, register. Only monitor axes the pipeline already observes responsibly.
- Measures are the same as categorical drift (Chapter 05): PSI, JSD, chi-square with effect size (Cramér's V), plus per-cell relative deltas.
- Monitor **both** the final post-filter composition **and** each filter stage's per-axis removal rate. The stage-level monitors root-cause the drift.
- The artifact is `population_balance.yaml`, emitted per slice version. The monitoring contract reads it.
- Keep the vocabulary straight: representativeness, balance, fairness — three different jobs, three different owners.
- Most balance drift is `retrain_blocking` or `advisory`. Reserve `publish_blocking` for contractual-minimum cells.

Chapter 08 moves from "monitor on cadence" to "turn each incident into a permanent guard" — the regression-fixture authoring pattern that keeps resolved incidents resolved.
