# Chapter 04 — Inter-Annotator Agreement: Cohen, Fleiss, Krippendorff

## Why this chapter exists

Inter-annotator agreement (IAA) is the primary quantitative signal that your labeling program is producing a usable label. It is also the statistic most frequently reported incorrectly in the wild — either because the wrong coefficient was picked for the task structure, or because a raw-percentage-agreement number was reported as if it were kappa.

Three coefficients cover almost every production labeling task:

- **Cohen's kappa** (Cohen 1960) — exactly two annotators, nominal labels, every item labeled by both.
- **Fleiss' kappa** (Fleiss 1971) — three or more annotators, nominal labels, every item labeled by the same number of annotators (not necessarily the same annotators across items).
- **Krippendorff's alpha** (Krippendorff 1980 / 2004) — any number of annotators, any level of measurement (nominal / ordinal / interval / ratio), **missing data permitted**. The most general coefficient; also the hardest to compute by hand.

This chapter explains when to use each, how to compute each correctly, and — most importantly — what threshold to accept. Threshold-picking is the operational decision; the formula is the easy part.

## Chance-corrected agreement, in one paragraph

Raw percentage agreement is useless. On a binary task where both annotators mark 90% of items as "positive" by random coincidence, their raw agreement is 81% — large and meaningless. **Chance-corrected** agreement subtracts the agreement you would expect if both annotators were labeling at random, given their per-class rates:

```
κ = (P_observed - P_chance) / (1 - P_chance)
```

- `P_observed` is what you measured.
- `P_chance` is what you would have measured if both annotators were randomly choosing labels at their observed per-class rates.
- `κ = 1` → perfect agreement. `κ = 0` → agreement exactly at chance. `κ < 0` → worse than chance (annotators disagree systematically).

The three coefficients differ in how they compute `P_chance` and in what measurement levels they handle. The chance-correction idea is common to all three.

## Cohen's kappa — two annotators, nominal labels

**Reference:** Cohen, J. *A Coefficient of Agreement for Nominal Scales.* Educational and Psychological Measurement, 1960. <https://doi.org/10.1177/001316446002000104>.

Cohen's kappa is the right coefficient when:

- Exactly **two** annotators label every item.
- Labels are **nominal** (no ordering).
- You can tolerate the chance-model assumption that each annotator uses labels independently at their own observed marginal rates.

In `scikit-learn`:

```python
from sklearn.metrics import cohen_kappa_score

# y1, y2 are arrays of labels from annotator A and B on the same N items
k = cohen_kappa_score(y1, y2)
```

### When Cohen's kappa misleads — the prevalence paradox

Cohen's kappa is **low on imbalanced tasks even when agreement is high**, because the chance-agreement baseline is dominated by the majority class. A binary toxicity-labeling task where 2% of items are toxic can show raw agreement of 99% and `κ ≈ 0.4` — not because annotators are disagreeing, but because the baseline chance agreement is also near 99%.

Report both `κ` and the per-class F1 or confusion matrix for imbalanced tasks. A single kappa number without the per-class breakdown is a sign that the statistic is being reported, not understood. See Feinstein and Cicchetti 1990 (*High agreement but low kappa*) for the classical write-up.

## Fleiss' kappa — multiple annotators, nominal labels

**Reference:** Fleiss, J. L. *Measuring Nominal Scale Agreement Among Many Raters.* Psychological Bulletin, 1971. <https://doi.org/10.1037/h0031619>.

Fleiss' kappa generalises Cohen to `n ≥ 3` annotators, with a subtle twist: **it allows different annotators to label different items**, as long as every item receives the same *number* of annotators. (Cohen requires the same two annotators on every item.) This fits the crowdsourced-pool setting well.

In Python with `statsmodels`:

```python
import numpy as np
from statsmodels.stats.inter_rater import fleiss_kappa, aggregate_raters

# Each row: one annotator's label for each item
# ratings shape: (n_items, n_annotators) with the label chosen by each annotator per item
table, categories = aggregate_raters(ratings)   # returns (n_items, n_categories) counts
k = fleiss_kappa(table, method="fleiss")
```

`aggregate_raters` converts from a "which annotator picked which label" matrix into a "how many annotators picked each label per item" count matrix — Fleiss's formulation requires the latter.

Fleiss' kappa has the same prevalence sensitivity as Cohen's, and the same remedy: report per-class breakdown alongside.

## Krippendorff's alpha — the general case

**Reference:** Krippendorff, K. *Content Analysis: An Introduction to Its Methodology.* 1980; 4th edition 2018. SAGE. <https://us.sagepub.com/en-us/nam/content-analysis/book258450>. Also: Hayes & Krippendorff 2007, *Answering the Call for a Standard Reliability Measure for Coding Data.* Communication Methods and Measures. <https://doi.org/10.1080/19312450709336664>.

Krippendorff's alpha is the general-purpose coefficient of choice when:

- **Different annotators label different numbers of items** (i.e. a sparse rater × item matrix).
- Labels can be **nominal, ordinal, interval, or ratio** — the choice of *difference function* controls the measurement level.
- You need a **single coefficient** that is comparable across tasks with different structure.

The alpha is defined as:

```
α = 1 - (D_observed / D_expected)
```

where `D` is a weighted disagreement computed with a task-appropriate difference function. For nominal labels, `delta(c1, c2) = 0 if c1 == c2 else 1`; for ordinal, `delta` is a function of the distance between category ranks; for interval, `delta = (c1 - c2)^2`. This is why Krippendorff's alpha applies to Likert rubric labeling (ordinal) and preference-strength labels (interval) without changing the statistic.

In Python with the `krippendorff` package:

```python
import krippendorff
import numpy as np

# reliability_data shape: (n_annotators, n_items); use np.nan for missing values
alpha_nominal = krippendorff.alpha(reliability_data=data, level_of_measurement="nominal")
alpha_ordinal = krippendorff.alpha(reliability_data=data, level_of_measurement="ordinal")
alpha_interval = krippendorff.alpha(reliability_data=data, level_of_measurement="interval")
```

Krippendorff's alpha reduces to Scott's π (1955) for two raters on nominal data, and is numerically close to Fleiss' kappa in the fully-observed-nominal case. The practical reason to prefer it is **missing data**: real labeling programs have annotators who skip items, drop out, or label different subsets of a stream.

## Pick-the-coefficient decision tree

```
How many annotators per item?
├── 2 fixed annotators, same pair on all items
│   ├── Nominal labels → Cohen's kappa
│   └── Ordinal / interval → Krippendorff's alpha
├── 3+ annotators, every item gets the same count
│   ├── Nominal, no missing data → Fleiss' kappa
│   └── Otherwise → Krippendorff's alpha
└── Variable number of annotators per item, missing data common
    └── Krippendorff's alpha (always)
```

For a production program using Argilla's `TaskDistribution(min_submitted=3)` with a floating vendor pool, **Krippendorff's alpha is the default**. For a tightly-controlled in-house two-annotator study, **Cohen's kappa** remains the clearest choice. Report both when transitioning from pilot (two annotators) to production (three or more).

## Thresholds — what counts as "good enough" agreement

Landis and Koch 1977 published a widely-cited (and widely-criticised) qualitative scale for kappa values:

| κ range | Landis-Koch label |
|---|---|
| < 0.00 | Poor |
| 0.00–0.20 | Slight |
| 0.21–0.40 | Fair |
| 0.41–0.60 | Moderate |
| 0.61–0.80 | Substantial |
| 0.81–1.00 | Almost perfect |

**Do not use this table as a target.** It was offered as a loose qualitative guide, not an operational threshold. The right target is **task-specific** and falls into roughly three regimes:

- **High-stakes, bounded task** (binary safety classification, fine-grained medical NER): target `κ or α ≥ 0.75`. Below that, the label signal is not strong enough to train on.
- **General NLP classification** (sentiment, topic, intent): target `κ or α ≥ 0.6`. This is where the Snow 2008 era landed for crowdsourced tasks after aggregation.
- **Open-ended rubric / preference** (RLHF helpfulness rubric, essay grading): target `α ≥ 0.4` for the hardest axes and `α ≥ 0.6` for the easiest. Preference-pair tasks are known-noisy; Ouyang 2022 reports IAA numbers in roughly this range on InstructGPT helpfulness and harmlessness rubrics. <!-- needs-research: cite the specific IAA numbers Ouyang 2022 §3.5 reports for the InstructGPT labeler pool from the paper at authoring time. -->

Three rules for picking a threshold:

1. **Pick it before you measure.** Writing down `α ≥ 0.6` in the guidelines before the first labeling session prevents the post-hoc rationalisation of a low number.
2. **Pick it from the task, not from "best practice".** A target of 0.8 on an open-ended essay-grading rubric is unrealistic; a target of 0.5 on a binary safety classification is dangerous.
3. **Treat it as a gate, not a KPI.** Below threshold → investigate (guidelines, calibration set, annotator population). Above threshold → ship, do not keep pushing.

## What to do when IAA is too low

The common root causes, in rough order of frequency:

1. **Guidelines under-specification.** Annotators resolve ambiguity differently. **Fix:** adjudicate a sample of disagreed items, extract the recurring ambiguity patterns, add to the edge-case catalogue, re-run.
2. **Taxonomy overlap.** Two labels are too similar to reliably distinguish. **Fix:** either merge the labels or add explicit disambiguating examples.
3. **Annotator population drift.** A new cohort has different training. **Fix:** re-run calibration on the new cohort before letting them into the production stream.
4. **UI confusion.** Annotators do not understand the fields. **Fix:** screenshot audit — Chapter 03's UI-mechanics guidelines section.
5. **Annotator fatigue.** Agreement drops within a session over time. **Fix:** cap per-session labeling at a modality-appropriate ceiling (text 2–3 hours, image 1–2 hours, video shorter still).
6. **Task is genuinely hard.** Ceiling IAA is simply low; no amount of guideline refinement raises it. **Fix:** report the ceiling to the downstream consumer, use the IAA as the noise estimate in the training setup, and allocate budget to more annotators per item instead of chasing a threshold.

## Per-class breakdown — always report it

A single kappa / alpha number hides which labels are driving disagreement. For every IAA report, emit the per-class agreement as well:

```python
# For a confusion-matrix-style analysis: majority label vs each annotator
from sklearn.metrics import classification_report
# y_consensus: the per-item consensus label (majority vote or adjudicated)
# y_annotator_i: annotator i's labels
report = classification_report(y_consensus, y_annotator_i, output_dict=True)
```

The actionable signal from per-class breakdown:

- One class with low per-class agreement → candidate for taxonomy merge or extra guideline work.
- One annotator with low agreement against consensus → candidate for re-training or removal from pool.
- High aggregate `α` with one class at near-zero agreement → report and act on the class, even though the headline number looks fine.

## The IAA report as an audit artifact

An `iaa_report.yaml` for a slice has, at minimum:

```yaml
slice_id: support-tickets-2026-01
guideline_version: 1.4.2
guideline_sha: sha256:a1b2c3...
annotator_pool_size: 24
items_measured: 1500
annotators_per_item: 3
statistic:
  primary: krippendorff_alpha_nominal
  value: 0.63
  ci_95: [0.60, 0.66]        # bootstrap
  threshold_set_pre_measure: 0.60
  decision: accept
secondary:
  fleiss_kappa: 0.61
per_class_agreement:
  support_request: 0.78
  complaint: 0.55
  feedback: 0.42
  other: 0.69
per_annotator_agreement_with_consensus:
  annotator_01: 0.81
  annotator_02: 0.77
  # ... one per annotator ...
  annotator_19: 0.38        # candidate for re-training
produced_by: program-owner@org.example
produced_at: 2026-11-04T12:00:00Z
```

This is one of the seven artifacts in the program package (Chapter 01). It becomes an input to the dataset card (`mod-112`) and is a lineage event in `mod-109`.

## Confidence intervals and the bootstrap

A single `α = 0.63` without a CI is a weak claim. The standard technique is **bootstrap resampling** of items (with replacement) and recomputing the statistic on each resample. 1,000 resamples, report the 2.5% and 97.5% percentiles:

```python
import numpy as np
import krippendorff

rng = np.random.default_rng(seed=42)
n_items = data.shape[1]
alphas = []
for _ in range(1000):
    idx = rng.integers(0, n_items, size=n_items)
    alphas.append(krippendorff.alpha(reliability_data=data[:, idx], level_of_measurement="nominal"))
ci_95 = (np.percentile(alphas, 2.5), np.percentile(alphas, 97.5))
```

For small items-measured counts (< 300), the CI is wide and the point estimate is noisy. Report the CI and resist over-interpreting the point estimate on small samples.

## What this chapter is **not**

- It is not a tutorial on every possible agreement coefficient. Scott's π, Gwet's AC1 / AC2, Light's kappa, and Conger's weighted kappa all exist and have appropriate niches. The three in this chapter cover 95% of production labeling programs.
- It is not a benchmark-construction chapter. `model-evaluation-engineer` (Chapter 09) uses IAA differently — in service of calibrating judge-vs-human agreement. The statistic is the same; the framing differs.
- It is not an active-learning chapter. Chapter 06 picks up the budget side.

## Summary

- **IAA is a chance-corrected statistic.** Raw percentage agreement is useless.
- **Cohen's kappa** for two annotators + nominal labels.
- **Fleiss' kappa** for three or more annotators + nominal labels + equal annotator counts per item.
- **Krippendorff's alpha** is the general-purpose default — handles missing data, ordinal / interval labels, and variable annotator counts.
- **Pick the threshold task-conditioned and pre-measure.** Landis-Koch is not an operational target.
- **Always report the per-class breakdown and a bootstrap CI.** A single headline number hides where disagreement actually lives.
- The output is an `iaa_report.yaml` artifact — one of the seven in the program package. Exercise 03 produces it.

Chapter 05 picks up the vendor-labelling side — the same statistics and the same calibration set, but now with a managed-service contract on top.
