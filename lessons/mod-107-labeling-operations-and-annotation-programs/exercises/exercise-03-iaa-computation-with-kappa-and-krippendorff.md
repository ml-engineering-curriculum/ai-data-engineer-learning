# Exercise 03 — IAA Computation with Kappa and Krippendorff

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 04 — Inter-annotator agreement: Cohen, Fleiss, Krippendorff](../04-inter-annotator-agreement-kappa-and-krippendorff.md)

## Objective

Compute inter-annotator agreement correctly on a real multi-annotator slice, with per-class breakdown, per-annotator breakdown, and a bootstrap confidence interval. Produce the `iaa_report.yaml` artifact for the program package.

By the end you will have:

- A multi-annotator labeled dataset (3–5 annotators on 300+ items minimum).
- `iaa_report.yaml` with Cohen's κ (pairwise), Fleiss' κ, and Krippendorff's α (nominal; ordinal if applicable), plus a bootstrap 95% CI on the primary statistic.
- A per-class and per-annotator breakdown table.
- An `interpretation.md` where you commit to a pre-measured threshold and record your actual accept / reject decision with reasoning.

## Prerequisites

- Chapter 04 read.
- Python 3.11+ with `pandas`, `pyarrow`, `numpy`, `scikit-learn`, `statsmodels`, `krippendorff`.
- A labeled slice with ≥ 3 annotators per item, ≥ 300 items with full overlap (every annotator labeled every item), ≥ 3 classes. You have four options:
  - **(Preferred)** real labels from Exercise 01 if you ran a multi-annotator session.
  - A public multi-annotator dataset — SNLI's annotator-disagreement subset, GoEmotions (multi-rater), or the Conflicting Emotions (CEM) dataset.
  - Synthesise a dataset: pick a public classification dataset, generate 3 "annotators" by perturbing the gold labels at different per-class error rates, verify you can recover the perturbation structure.
  - Run a short crowd study with 3 peers / colleagues on 100 items.

## Problem statement

Compute all three coefficients, break them down per class and per annotator, bootstrap a CI, and interpret the result against a **pre-committed** threshold.

**The pre-commitment matters.** Write the threshold down (in `interpretation.md`) before running the computation. The exercise is not about getting a high number; it is about being honest about what the number means.

## Starter guidance

### 1. Load the data into the standard shape

Krippendorff's alpha wants a `(n_annotators, n_items)` matrix with `np.nan` for missing ratings. Fleiss's wants `(n_items, n_categories)` counts. Build a helper that converts between them.

```python
import numpy as np
import pandas as pd

def to_wide(df, annotator_col, item_col, label_col):
    """Long-format labels -> (n_annotators, n_items) matrix with np.nan for missing."""
    wide = df.pivot(index=annotator_col, columns=item_col, values=label_col)
    return wide

def to_fleiss(wide, categories):
    """(n_annotators, n_items) wide -> (n_items, n_categories) counts."""
    # one col per category, count how many annotators chose each
    counts = np.zeros((wide.shape[1], len(categories)), dtype=int)
    for j, item in enumerate(wide.columns):
        col = wide[item].dropna().values
        for k, c in enumerate(categories):
            counts[j, k] = (col == c).sum()
    return counts
```

### 2. Compute the three statistics

```python
from sklearn.metrics import cohen_kappa_score
from statsmodels.stats.inter_rater import fleiss_kappa
import krippendorff

# Pairwise Cohen's kappa for each annotator pair
def pairwise_cohen(wide):
    out = {}
    anns = wide.index.tolist()
    for i in range(len(anns)):
        for j in range(i + 1, len(anns)):
            both = wide.loc[[anns[i], anns[j]]].dropna(axis=1)
            out[(anns[i], anns[j])] = cohen_kappa_score(both.iloc[0], both.iloc[1])
    return out

# Fleiss
fleiss_counts = to_fleiss(wide, categories=sorted(set(wide.stack())))
fk = fleiss_kappa(fleiss_counts, method="fleiss")

# Krippendorff
reliability = wide.values  # already (n_annotators, n_items)
alpha = krippendorff.alpha(reliability_data=reliability, level_of_measurement="nominal")
```

Report all three. Explain in `interpretation.md` which one is the **primary** statistic for your data (Chapter 04 §"Pick-the-coefficient decision tree").

### 3. Per-class and per-annotator breakdowns

- **Per-class:** compute consensus (majority vote), then per-class F1 / precision / recall of each annotator against the consensus. One row per (annotator, class).
- **Per-annotator:** mean agreement with consensus, standard deviation across items, items where this annotator was the sole disagreer.

Save as a parquet table `per_class_per_annotator.parquet`.

### 4. Bootstrap the primary statistic

```python
rng = np.random.default_rng(seed=42)
n_items = reliability.shape[1]
alphas = []
for _ in range(1000):
    idx = rng.integers(0, n_items, size=n_items)
    alphas.append(krippendorff.alpha(reliability_data=reliability[:, idx], level_of_measurement="nominal"))
ci_low, ci_high = np.percentile(alphas, [2.5, 97.5])
```

Report the point estimate and the CI. For N < 300, expect a wide CI; note it.

### 5. Commit the threshold **before** the result and decide

Fill in `interpretation.md`:

```
Target task-conditioned threshold: α ≥ ___ (pick before running)
Reason for the threshold: ___
Observed: α = ___ (CI [___, ___])
Decision: accept / investigate / reject
Reasoning for decision: ___
Follow-up actions: ___
```

If `α < threshold`, pick at least one of the six root-cause categories from Chapter 04 §"What to do when IAA is too low" and write a one-paragraph diagnosis.

### 6. Emit the audit artifact

Write `iaa_report.yaml` with the structure from Chapter 04 §"The IAA report as an audit artifact":

```yaml
slice_id: ...
guideline_version: ...
guideline_sha: ...
annotator_pool_size: ...
items_measured: ...
annotators_per_item: ...
statistic:
  primary: krippendorff_alpha_nominal
  value: ...
  ci_95: [..., ...]
  threshold_set_pre_measure: ...
  decision: ...
secondary:
  fleiss_kappa: ...
  cohen_kappa_pairs:
    - [ann_01, ann_02, ...]
    - ...
per_class_agreement: {...}
per_annotator_agreement_with_consensus: {...}
produced_by: ...
produced_at: ...
```

## Deliverables

```
exercise-03/
  data/
    labels.parquet                       # (annotator_id, item_id, label, cohort_id, timestamp)
  code/
    compute_iaa.py
    bootstrap_ci.py
    emit_report.py
  results/
    per_class_per_annotator.parquet
    iaa_report.yaml
  interpretation.md                      # threshold committed before computation
  README.md
```

## Acceptance criteria

- `interpretation.md` names a threshold and reasoning **before** any numerical result appears in the document.
- `iaa_report.yaml` conforms to the Chapter 04 schema with all three coefficients populated or explicitly marked `null` with a reason.
- Bootstrap CI is at least 1,000 resamples; both endpoints are reported.
- Per-class and per-annotator tables exist and surface at least one actionable signal (a class or an annotator with materially lower agreement) *or* a note explaining why the data is too uniform to show one.
- Running `code/emit_report.py` twice on the same inputs produces byte-identical output (seed the bootstrap).
- `α` and `κ` differences are plausible — if Fleiss's κ and Krippendorff's α nominal differ by > 0.05 on fully-observed nominal data, investigate your preprocessing before trusting the number.
- Any factual claim in `interpretation.md` that cannot be verified is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Ordinal / interval Krippendorff** — if your task has an ordinal structure (Likert 1–5), rerun with `level_of_measurement="ordinal"` and compare. Note the difference and explain it.
- **MACE on crowd data** — install `MACE` (<https://github.com/dirkhovy/MACE>) and run it on the same data. Compare its per-annotator competence estimate with your per-annotator agreement-with-consensus number.
- **Simulation study** — with a synthesised dataset you control, deliberately introduce 1–2 "spam" annotators (random labels). Measure how quickly α drops and whether per-annotator breakdown surfaces the spammers.
- **Report-generation CLI** — make `emit_report.py` a `click` CLI that any downstream team member can run against their own `labels.parquet`.

## What this exercise is not

- It is not an annotator-recruitment exercise (Exercises 01 and 02 cover the ops side).
- It is not a weak-supervision exercise (Exercise 05).
- It is not a benchmark-construction exercise (Chapter 09 and `model-evaluation-engineer`).

It is the one artifact — a signed, honest IAA report — that most distinguishes a mature labeling program from an immature one.
