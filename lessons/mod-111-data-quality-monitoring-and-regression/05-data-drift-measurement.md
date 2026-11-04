# Chapter 05 — Measuring Data Drift at the Input Level

## Why this chapter exists

"Data drift" in production is a loaded phrase. It covers at least three different statistical concepts, computed with at least five different tests, with at least four different reference-window choices, each of which flips the alert rate by an order of magnitude. If the pipeline fires drift alerts that the on-call rotation ignores, the contract is dead paper.

This chapter is the opinionation chapter. It fixes vocabulary (what we mean by *data drift* vs. *concept drift*), enumerates the tests you actually use (KS, Chi-square, PSI, Jensen-Shannon, Wasserstein / EMD, MMD for embeddings), pins the thresholds you start from and the sample sizes those thresholds require, and defends the reference-window choice. By the end you can take a row of `dataset_monitoring_contract.yaml` that says `kind: data_drift` and fill in the `stattest`, the `threshold`, the `reference`, and the sample-size note with reasons.

## The three kinds of drift (and what we cover here)

The literature is not always consistent, but the three concepts you must separate:

- **Covariate / input / feature drift** — P(X) changes. The distribution of the **input** data shifts. This is what this module monitors.
- **Label / prior drift** — P(Y) changes. The distribution of labels shifts even when inputs do not. In pre-training there is no Y; in supervised fine-tuning you may own this, in reinforcement / preference-tune data this often flips to a peer role.
- **Concept drift** — P(Y | X) changes. The mapping from X to Y shifts. You do **not** measure this from data alone — it requires model predictions. The two peer-L30 eval roles own it.

Chapter 01's distinctions are operational — this is the statistical restatement of the same boundary. *This chapter is about covariate drift*. Everything else routes to `ai-eval-engineer` / `model-evaluation-engineer`.

Rabanser, Günnemann, Lipton (NeurIPS 2019) *Failing Loudly: An Empirical Study of Methods for Detecting Dataset Shift* is the canonical empirical treatment of which drift-detection methods actually catch which shifts. <!-- needs-research: cite the paper's recommendations table at authoring time rather than paraphrasing. --> Treat it as the reference you consult when you argue a threshold in a design review.

## The test catalogue — one test, one column type

The tests the ML-aware tools (`whylogs`, Evidently, Deepchecks) and the general ones expose, with the signature question each answers.

### Kolmogorov–Smirnov (KS)

- **Column type:** continuous numeric.
- **Signature question:** are the two empirical CDFs the same?
- **Returns:** D-statistic (max vertical distance between CDFs) + p-value.
- **Threshold anchor:** standard alpha 0.05 on p-value, but the KS test is notoriously sensitive at large N — see the sample-size section. A thresholded D-statistic (e.g., D ≤ 0.1) is often more stable.
- **Fails on:** categorical columns, zero-variance columns, heavily tied data. Chi-square or JSD instead.

### Chi-square

- **Column type:** categorical, low-to-moderate cardinality (≤ ~50 categories).
- **Signature question:** are the two category-count distributions the same?
- **Returns:** χ² statistic + p-value.
- **Threshold anchor:** alpha 0.05 on p-value. Fails when expected cell counts drop below ~5 — sparse categories are a problem.
- **Fails on:** very high-cardinality fields (TLD, user_id). Use Jensen-Shannon or PSI instead.

### Population Stability Index (PSI)

- **Column type:** both numeric (after binning) and categorical.
- **Signature question:** how much has the probability mass shifted between the reference distribution and the current one?
- **Returns:** a single scalar PSI.
- **Formula:** PSI = Σ (p_cur,i − p_ref,i) · ln(p_cur,i / p_ref,i) over bins i.
- **Threshold anchor (standard credit-risk / FICO heritage):** < 0.1 stable; 0.1–0.25 moderate; > 0.25 significant. <!-- needs-research: verify the FICO / Siddiqi origin of this threshold convention and cite it directly. --> Treat the thresholds as a strong default, not divinely-sent.
- **Fails on:** zero-probability bins (fix with Laplace smoothing or a small-ε floor). Binning choice is load-bearing.

### Jensen-Shannon divergence (JSD)

- **Column type:** categorical (any cardinality) or binned numeric.
- **Signature question:** how much do two discrete distributions diverge, symmetrically and bounded?
- **Returns:** scalar JSD ∈ [0, 1] (when using log₂).
- **Threshold anchor:** 0.1 is a common moderate-drift threshold; 0.02 is a tight one. Choose based on the sample size and the historical noise band on the metric.
- **Fails on:** very high-cardinality when both distributions are long-tailed sparse — be careful with the smoothing choice.

### Wasserstein / Earth Mover's Distance (EMD)

- **Column type:** continuous numeric or ordinal.
- **Signature question:** what is the minimum cost to transport the reference distribution to the current one?
- **Returns:** scalar with the units of the underlying column.
- **Threshold anchor:** column-specific — the threshold is in the column's units. Pin a historical band per column.
- **Fails on:** categorical, unordered columns.

### Maximum Mean Discrepancy (MMD)

- **Column type:** high-dimensional (embeddings).
- **Signature question:** do two samples come from the same distribution, measured in a kernel-based feature space?
- **Returns:** MMD² statistic + permutation p-value.
- **Threshold anchor:** permutation-test p-value < 0.05, with a reasonable permutation count (≥ 1,000).
- **Fails on:** tiny samples; MMD is sample-hungry. Use for embedding drift only when you have a few thousand embeddings per side.

### Default picks per column archetype

| Column archetype | Default stattest | Backup when default fails |
|---|---|---|
| Continuous numeric (text_length, token_count) | KS or Wasserstein | PSI (binned) |
| Low-cardinality categorical (language, boolean) | Chi-square or JSD | — |
| High-cardinality categorical (url_host_domain, TLD) | JSD (with smoothing) or PSI (with aggregated bins) | top-K + "other" bucket + Chi-square |
| Ordinal (document-quality bucket) | Wasserstein | Chi-square |
| Embedding (512-d sentence embedding) | MMD or energy distance | mean-cosine-shift as a weak proxy |

Evidently's default stattest chooser follows a similar logic; <!-- needs-research: confirm the exact Evidently default-stattest selection rule and cardinality-threshold at the pinned version. --> override explicitly in the `ColumnDriftMetric(column_name=…, stattest=…)` call so the choice is in the diff.

## Sample size and why your p-value is misleading

The KS and chi-square tests are *frequentist significance tests*. At large N the p-value collapses to zero on **any** real-world distribution difference, because the test is detecting arbitrarily small effects with arbitrarily high power. On a 10 M-row training slice almost any column will fail KS at alpha 0.05 — and the alert is meaningless.

Three mitigations:

1. **Monitor the effect size, not the p-value.** The D-statistic in KS, the PSI scalar, the JSD scalar, the Wasserstein distance — these are *magnitudes*, not p-values. They do not collapse at large N. Prefer them in the contract.
2. **Downsample to a fixed reference N.** Sample N rows (say 50 k) from each side; run the test on the sample; repeat K times and record the median and the IQR. The test then measures a *real-size* drift, not a statistical-power drift.
3. **Compute a bootstrap CI.** For any effect-size statistic (D, PSI, JSD), resample with replacement B = 1,000 times to get a 95% CI. Alert on the CI's lower bound crossing the threshold, not on the point estimate.

Record the choice — "we use JSD with 50 k downsample and 1,000-boot CI" — in the `statistics_note` section of the contract preamble. Future-you re-running the metric needs to know the sampling discipline.

## Choosing the reference window

The reference is *the pinned reference slice* — the one Evidently and `whylogs` compare `current` against. Three reasonable choices:

- **Fixed golden reference.** A hand-chosen, hand-reviewed slice version. Example: `cc_en.v3.ref_2026-09-15` — the last slice before the production model was trained. Use when a specific slice is the training baseline you want to defend.
  - Strength: comparisons stay stable over time; you detect long-run drift.
  - Weakness: the reference ages; noise in the reference becomes a false-alert source.

- **Rolling window reference.** Trailing N days / N slice versions. Example: median of the last 4 weekly slices.
  - Strength: tracks the pipeline's recent-normal; low false-alert rate.
  - Weakness: *slow drift* across the rolling window is invisible. If every slice drifts 1% relative to the last one, after a quarter the pipeline is 13% off the fixed golden — and you never alerted.

- **Previous-slice reference.** `v_n−1` compared to `v_n`.
  - Strength: catches sudden changes between versions.
  - Weakness: cannot detect slow cumulative drift at all.

The production shape that catches both failure modes: **a two-track reference**.

```yaml
# contract preamble
reference_strategy:
  - name: golden
    slice: cc_en.v3.ref_2026-09-15
    use_for: [slow_drift_detection, retrain_decision]
  - name: rolling_4w
    strategy: median_of_last_4_weekly_slices
    use_for: [sudden_drift_detection, publish_gate]
```

Each monitored row then names which reference it compares against:

```yaml
- id: m.drift.language_jsd.v1
  reference: golden
  threshold: {lte: 0.02}
  severity: retrain_blocking

- id: m.drift.language_jsd_wow.v1
  reference: rolling_4w
  threshold: {lte: 0.05}
  severity: advisory
```

Two tracks per metric is overhead; pick the subset of columns that deserve both (anything retrain-blocking).

## Embedding drift — when and how

For a text training corpus, scalar descriptors (length, char-type ratio, OOV rate) cover most of the drift surface. But sometimes the semantic content shifts without a descriptor moving — a Reddit-style forum gets dominated by a new topic; a legal-document source shifts jurisdictions; a licensed feed changes genre mix. These shifts register in **sentence embeddings** and only in sentence embeddings.

The embedding-drift pipeline:

1. Pick a frozen embedding model. A small multilingual sentence-transformer (e.g., `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`) is a reasonable default. **The model must be fixed across runs** — a different model is a different reference.
2. Compute embeddings on a stratified sample (10 k / 50 k per side, depending on budget).
3. Run MMD (or energy distance) between the two embedding sets. Report the statistic + permutation-test p-value.
4. For diagnosis, run PCA (or UMAP for exploration only — not for monitoring) and show the shift visually.

Evidently has `EmbeddingsDriftMetric` for exactly this; `whylogs` requires rolling your own. The alert threshold is empirical — pin a historical embedding-drift MMD band for the corpus and alert on exceedance.

Costs:

- **GPU hours per run.** Embedding 50 k documents on a modern GPU is minutes, not hours; it is a real line item.
- **Model pinning.** The embedding model becomes part of the `reference_profile` SHA chain.
- **False confidence on semantics.** An embedding-drift test sees *what the embedding model sees*. Multilingual blind spots, domain-specific semantic failures, and adversarial inputs can defeat it.

Add embedding drift to the contract only when a stakeholder has asked for a semantic-shift signal; descriptor drift is usually enough.

## Alerting discipline — three failure modes to avoid

Drift monitors fire often. The three ways the on-call rotation starts ignoring them:

1. **Threshold whipsaw.** Thresholds set too tight produce near-daily alerts that are within the noise band. Mitigation: pin a historical band per metric (compute the metric on 30 historical slice pairs; take the 95th percentile; alert at twice that).
2. **Reference ageing.** A fixed golden reference from a year ago looks "drifted" because the whole world moved on. Mitigation: refresh the golden reference on an explicit cadence (quarterly, or on each retraining). Record the refresh in the contract's change log.
3. **Alert fatigue cascades.** Three correlated metrics (language JSD, TLD JSD, text-length KS) all fire on the same incident. Mitigation: *de-duplicate* alerts at the substrate layer — one incident per slice version, with the full metric list attached — rather than three pages.

The contract's `severity:` fields do most of the work on (3). Most drift monitors should be `advisory`, not `retrain_blocking`. Retrain-blocking belongs to a small set of metrics whose breach *actually* changes the retraining decision.

## A worked example: the `language` column

Walk through one column from the bottom up. The slice is `cc_en.v3`; the column is `language` (two-letter ISO code from FastText lid.176).

**Step 1 — pick the test.** The column is categorical with ~180 possible values but 95%+ mass on `en`. Default chi-square would fail on sparse cells. JSD with Laplace smoothing is the right pick.

**Step 2 — pick the reference.** The retraining decision is anchored to the language mix the production model saw. Fixed golden reference it is: `cc_en.v3.ref_2026-09-15`.

**Step 3 — pick the sample.** 50 k documents per side, stratified by TLD so rare languages are over-sampled; 1,000-bootstrap CI on JSD.

**Step 4 — pick the threshold.** Compute JSD on 30 historical slice pairs. The 95th percentile of the historical band is 0.008. Alert threshold 0.02 is 2.5× the noise band — tight but not hair-triggered.

**Step 5 — pick the severity.** A language-mix shift is retrain-relevant (the model's training objective is anchored to the language distribution it saw). `retrain_blocking`, Slack-routed.

**Step 6 — write the contract row.**

```yaml
- id: m.drift.language_jsd.v1
  kind: data_drift
  tool: evidently
  tool_version: 0.5.1
  stattest: jensenshannon
  stattest_options:
    smoothing: laplace
    alpha: 1e-6
  column: language
  reference: golden                               # defined in preamble
  cadence: on_slice_build
  sampling:
    n: 50000
    strata: url_host_tld
    bootstrap_reps: 1000
  threshold:
    lte: 0.02                                     # JSD, log2 base
    note: "2.5x the 95th-percentile historical-band of 0.008"
  severity: retrain_blocking
  owner: ai-data-engineer@acme
  route: slack:#ai-data-eng-alerts
  evidence_artifact: evidently/reports/${slice_version}.json
```

Every field was a decision with a reason. That is the shape of a reviewable monitoring contract.

## Where drift measurement ties back to earlier modules

- **[`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md)**: the per-slice quality report emits the distributions the drift test consumes. The 5259-2 statistical-quality measures *are* the drift inputs.
- **[`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)**: benchmark-overlap n-gram hits are a drift surface (new overlaps appearing is a regression). Monitored under `kind: quality`, not `kind: data_drift`, because the reference is the eval set, not a historical slice.
- **[`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)**: the `reference:` field in every drift row is a dataset version-pin. The pin is produced by mod-109; this module *watches* it.

## Summary

- Monitor *covariate* drift (P(X) shift). Concept drift and label drift route to the eval roles.
- Test choice follows column type: KS / Wasserstein for continuous; Chi-square / JSD / PSI for categorical; MMD for embeddings.
- At training-corpus scale, prefer *effect-size* statistics (D, PSI, JSD, Wasserstein) over p-values; the p-value collapses to zero and the alert becomes meaningless.
- Use a two-track reference: a pinned golden reference for retrain-decision signals, and a rolling window for sudden-shift detection. Each metric names which reference it compares against.
- Sample-size discipline: downsample to a fixed N, bootstrap the statistic, alert on the CI's lower bound.
- Embedding drift (MMD on a frozen embedding model) adds a semantic axis, at GPU cost; add it when descriptors aren't enough.
- Guard against threshold whipsaw, reference ageing, and alert-fatigue cascades. Most drift signals should be advisory.
- Every row of the contract names the test, the threshold, the reference, the sample size, and the severity — each with a reason.

Chapter 06 moves from *distribution* drift to *schema* drift — the sibling signal, cheaper to compute, more operationally brittle, and the one whose default response is rollback.
