# Exercise 05 — Snorkel Weak-Supervision Amplifier

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 07 — Weak supervision and data programming (Snorkel)](../07-weak-supervision-and-data-programming-snorkel.md)

## Objective

Build a weak-supervision labeling pipeline on a real classification slice: author 10–20 labeling functions, train a Snorkel label model, train a discriminative classifier on the resulting soft labels, and prove (or disprove) that it meaningfully beats a gold-only baseline. Emit the `weak_supervision/` block of the program package.

This is a production-shaped exercise: you measure, you honestly compare against the baseline, and you are willing to conclude "weak supervision is not pulling its weight on this task" if that's what the data shows.

By the end you will have:

- `labelling_functions.py` with 10–20 versioned LFs and per-LF tests.
- `lf_analysis_summary.yaml` with coverage, overlap, conflict, per-LF gold accuracy.
- A trained Snorkel label model and a trained discriminative model.
- `budget_delta.md` — the one-page summary the budget owner reads.

## Prerequisites

- Chapter 07 read. Ratner et al. 2016 §§2, 3, 5 skimmed.
- Python 3.11+ with `snorkel`, `scikit-learn`, `pandas`, `pyarrow`, `numpy`. Optionally `cleanlab` for the stretch.
- A multi-class classification dataset where the ground truth is known on the full pool, so you can measure LF accuracy. Same options as Exercise 04 (20Newsgroups, AG News, Yahoo! Answers, DBpedia, or your own slice).
- Ideally **not** the same dataset you used for Exercise 04 — covering two distinct tasks is a better learning outcome. If reuse is necessary, say so in `README.md`.

## Problem statement

Set up a weak-supervision pipeline that treats a small gold-labeled seed (`L_gold`, 500–2,000 items) as the only "human" labels, and amplifies to `N_weak` ≥ 50× `|L_gold|` weakly-labeled items via Snorkel. Compare against a gold-only baseline trained on only `L_gold`.

## Starter guidance

### 1. Split the data

```
full_data (N items)
  ├── gold_labeled     (e.g. 1500 items, random)     - L_gold
  ├── pool             (remaining ~N - 1500)         - U, unlabeled from the pipeline's POV
  └── test             (held out IID from the full)  - for both classifiers' final evaluation
```

You have ground truth on the whole pool for the exercise, but your code must not use pool-side ground truth to train either classifier.

### 2. Author 10–20 labeling functions

Author them in `labelling_functions.py`. A good mix:

- **3–5 lexicon LFs** — class-indicative keywords, from your own reading of the data.
- **2–3 regex / pattern LFs** — more precise than lexicons.
- **2–3 heuristic LFs** — length, structure, presence of specific entities.
- **1–2 distant-supervision LFs** — use a pretrained classifier (TF-IDF + logistic regression on L_gold works) as an LF.
- **(stretch) 1 LLM-as-labeler LF** — call a local small LLM with a brief prompt. Treat as *one* LF, not the ground truth.

Each LF has a docstring and at least one `assert` test case (one positive, one negative). Keep them simple — one short function each.

### 3. Apply LFs and run `LFAnalysis`

```python
from snorkel.labeling import PandasLFApplier, LFAnalysis

applier = PandasLFApplier(lfs=lfs)
L_train = applier.apply(df=pool_df)

analysis = LFAnalysis(L=L_train, lfs=lfs).lf_summary(Y=pool_df["y_gold"].values)
```

Save `analysis` as `lf_analysis_summary.yaml`. Every LF should show coverage, overlap, conflict, polarity, correct / incorrect / abstain counts, and emp-accuracy.

Resist the temptation to tune LFs against the LF analysis until the numbers look pretty. Record your first-authored LF set *before* you iterate.

### 4. Train the Snorkel label model

```python
from snorkel.labeling.model import LabelModel

label_model = LabelModel(cardinality=NUM_CLASSES, verbose=True)
label_model.fit(L_train=L_train, n_epochs=500, lr=0.01, log_freq=100, seed=42)
soft_labels = label_model.predict_proba(L_train)            # (n_items, K)
hard_labels = label_model.predict(L_train, tie_break_policy="abstain")
```

Filter out items where the label model abstains (any-row-sum below a threshold or `-1` hard label). Save the surviving `(x, soft_label)` pairs as `label_model_outputs.parquet`.

### 5. Train two downstream classifiers

**Baseline — gold-only:** train on `L_gold` with hard labels. Evaluate on `test`.

**Weakly-supervised:** train on the Snorkel-labeled pool with soft labels (cross-entropy over the full predictive distribution), evaluate on `test`.

Use the same model architecture and hyperparameters for both (logistic regression on TF-IDF is a reasonable default). Report `test_accuracy`, per-class F1, and a bootstrap 95% CI on test accuracy for each.

### 6. Write `budget_delta.md`

Use the structure from Chapter 07 §"The weak-supervision block in the program package":

```
budget_delta.md:
  gold_labels_used:     1500
  weak_labels_produced: ___
  baseline:
    gold_only_classifier_test_accuracy: ___ (CI ___, ___)
  weak_supervision:
    weakly_supervised_classifier_test_accuracy: ___ (CI ___, ___)
  delta: +___ / -___
  decision: amplify / do-not-amplify / pilot-more
  reasoning: ___
```

The exercise is not "beat the baseline". The exercise is "measure and decide honestly". If the weakly-supervised classifier does not beat the baseline, your `decision` is "do-not-amplify" and that is a valid completion.

### 7. Diagnose if it did not work

If the weakly-supervised model underperforms, pick at least one of the Chapter 07 §"Where weak supervision wins — and where it loses" failure modes and diagnose it against your LF analysis. Save as `diagnosis.md`.

## Deliverables

```
exercise-05/
  data/
    split_info.yaml
  code/
    labelling_functions.py
    labelling_functions_test.py        # pytest-style tests for each LF
    apply_lfs.py
    train_label_model.py
    train_discriminative.py
    evaluate.py
  results/
    lf_analysis_summary.yaml
    label_model_outputs.parquet
    baseline_metrics.yaml
    weak_supervised_metrics.yaml
  budget_delta.md
  diagnosis.md                         # required if weak supervision did not beat baseline
  README.md
```

## Acceptance criteria

- `labelling_functions.py` has ≥ 10 LFs, each with a docstring and a test.
- `lf_analysis_summary.yaml` has coverage, overlap, conflict, polarity, and empirical accuracy for every LF.
- No LF has coverage > 85% (unless it is a pretrained classifier, in which case say so explicitly).
- The baseline (gold-only) and weak-supervision classifiers use the **same architecture and hyperparameters**.
- Test-set bootstrap CIs are reported for both classifiers.
- `budget_delta.md` has a committed decision (`amplify` / `do-not-amplify` / `pilot-more`) with reasoning.
- `diagnosis.md` exists if the weak-supervision classifier did not beat the baseline — honest diagnosis is required, not a hidden failure.
- Running `pytest code/labelling_functions_test.py` passes.
- Reproducibility: all seeds are fixed; `python code/apply_lfs.py && python code/train_label_model.py && python code/train_discriminative.py` produces the same metrics on repeat runs.

## Stretch goals

- **LLM-as-labeler LF** — add a Snorkel-style labeling function that calls a local LLM with a short prompt. Measure coverage, accuracy, and (critically) correlation with your other LFs.
- **Snorkel dependency structure** — use `snorkel.labeling.model.DependencyLabelModel` or hand-specify a correlation matrix; measure whether modelling dependencies changes the result.
- **Pair with Cleanlab** — after weak labeling, run `cleanlab` on the weak labels to filter noisy items. Measure the delta.
- **Skweak for a sequence-labeling task** — replicate on NER instead of classification. The library is different; the framework is the same.

## What this exercise is not

- Not a label-noise-cleaning exercise (Cleanlab is a stretch, not the core).
- Not an LLM-as-judge exercise (Chapter 09).
- Not a synthetic-data-generation exercise (that is `mod-108`).

It is the exercise that trains your intuition for **when to use weak supervision and when not to**. The honest `do-not-amplify` conclusion is as valuable as the `+6%` conclusion.
