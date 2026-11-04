# Exercise 04 — Active Learning Budget-vs-Quality Run

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 06 — Active learning for budget efficiency](../06-active-learning-for-budget-efficiency.md)

## Objective

Prove the budget / quality trade-off of active learning on a real task. Produce a plot and a one-page `budget_plan.md` that defends the AL choice to a budget owner. Specifically:

- Run **pool-based uncertainty sampling** (entropy and margin) against a **random baseline** on a small, bounded classification task.
- Report the budget / quality curve on an IID held-out set.
- Diagnose at least one failure mode from Chapter 06 §"When active learning hurts" (cold start, class imbalance, distribution shift) on *your* data — even if it turned out not to matter, you must have looked.
- Write a budget-plan artifact for the program package.

By the end you will have a concrete answer to "how many labels does AL save at matched downstream accuracy", on a dataset you understand.

## Prerequisites

- Chapter 06 read. Settles 2009 §§2, 3 skimmed.
- Python 3.11+ with `scikit-learn`, `numpy`, `pandas`, `matplotlib`. Optionally `small-text` (<https://github.com/webis-de/small-text>) or `modAL` (<https://github.com/modAL-python/modAL>) for the AL loop plumbing.
- A **classification** dataset with ground-truth labels available on the full pool (so you can simulate AL with perfect "labeling" by looking up ground truth). Recommended:
  - 20Newsgroups (20 classes, moderate size) — comes with `sklearn.datasets.fetch_20newsgroups`.
  - AG News, DBpedia, Yahoo! Answers (large, ≥ 4 classes).
  - Your Exercise 01 slice if you have gold labels.

## Problem statement

Build an active-learning simulation that compares AL against random baseline on a fixed held-out set. You simulate the "labeler" by reading the ground truth; the point is the budget curve, not actually paying someone.

## Starter guidance

### 1. Split the data

```
full_data (N items with ground truth)
  ├── holdout_iid       (20% of N, IID from the full distribution) - NEVER touched during AL
  ├── initial_seed      (e.g. 200 items, random) - the starting labeled set L
  └── pool              (remaining ~80% - 200) - U, the unlabeled pool
```

### 2. The AL loop

```python
def run_al(strategy, seed_items, pool_items, holdout, budget, batch_size=100):
    L = list(seed_items)
    U = list(pool_items)
    history = []
    while len(L) < budget:
        model = train(L)                       # LR or sklearn pipeline on L
        acc = score(model, holdout)
        history.append((len(L), acc))
        probs = model.predict_proba([u.x for u in U])
        if strategy == "random":
            picks = random_sample(U, batch_size)
        elif strategy == "entropy":
            scores = entropy_of(probs)         # argmax
            picks = top_k_by(scores, U, batch_size)
        elif strategy == "margin":
            scores = -margin_of(probs)         # smaller margin -> higher priority
            picks = top_k_by(scores, U, batch_size)
        else:
            raise ValueError(strategy)
        L.extend(picks)
        for p in picks:
            U.remove(p)
    # final evaluation
    model = train(L)
    history.append((len(L), score(model, holdout)))
    return history
```

Run three strategies: `random`, `entropy`, `margin`. Record `(labels_used, holdout_accuracy)` after each round. Use a fixed seed for reproducibility.

### 3. Report the three curves on one plot

```python
import matplotlib.pyplot as plt

for strategy, history in results.items():
    xs, ys = zip(*history)
    plt.plot(xs, ys, label=strategy)
plt.xlabel("Labels consumed")
plt.ylabel("Holdout accuracy")
plt.legend()
plt.savefig("results/budget_curve.png", dpi=150)
```

The AL curves should sit above the random curve for at least a portion of the budget range. If they don't, that is a real result — Chapter 06 §"When active learning hurts" is where you reason about why.

### 4. Diagnose at least one failure mode

Pick one:

- **Cold start** — re-run with a seed of 20 items instead of 200. Does AL's early-round advantage flip?
- **Class imbalance** — compute the per-class counts in the AL-selected labeled set as a function of round. Is a class systematically under-sampled? If yes, add a diversity-aware term or Core-Set and compare.
- **Distribution shift** — hold out a non-IID set (e.g. items from one class only) and plot accuracy on it as a separate curve. Does the AL-selected training set generalise to the held-out distribution?

Write a half-page `diagnosis.md`.

### 5. Author `budget_plan.md`

Use the seven-section skeleton from Chapter 06 §"When to call the budget plan 'done'". Keep it one page. Include:

- The target metric and the task definition.
- The expected labels-to-target from your curve.
- The AL strategy picked and the Settles 2009 reference.
- The fallback (what you'd do if the AL curve flattens below target).

## Deliverables

```
exercise-04/
  data/
    split_info.yaml                  # holdout size, seed size, seed
  code/
    al_loop.py
    strategies.py
    plot.py
  results/
    curves.parquet                   # (strategy, round, labels_used, accuracy, per_class_counts)
    budget_curve.png
  diagnosis.md                       # one failure-mode diagnosis
  budget_plan.md                     # the artifact for the program package
  README.md
```

## Acceptance criteria

- The three strategies (random, entropy, margin) are run with the **same seed** and the **same holdout** set.
- `budget_curve.png` shows all three curves clearly labelled on one plot.
- `curves.parquet` has at least 10 rounds for each strategy.
- `diagnosis.md` names which of the three failure modes you looked at and reports its presence or absence on *your* data.
- `budget_plan.md` includes a plausible "labels-to-target" estimate and a fallback plan.
- Running `python code/al_loop.py && python code/plot.py` reproduces the plot and the parquet deterministically.
- The IID holdout is **not** touched during model selection or training.

## Stretch goals

- **Add Query-by-Committee.** Train 5 bagged LR classifiers, measure vote-entropy as the acquisition. Compare.
- **Add BatchBALD / Core-Set.** Both are 1-page implementations in `small-text`; the lift over entropy on diverse batches is often material.
- **Wire the AL loop into Label Studio** via the ML-backend API so a human could label the picked items in a real UI. This is where simulation becomes production.
- **Replace the "oracle" with LLM-as-labeler.** Call a local LLM on each picked item and treat its output as the label. Measure how far the curve shifts when the "labeler" is itself noisy.
- **Run on a multi-class with imbalance.** A 10-class dataset where one class is 1% prevalence. Report what AL strategy handles it least badly.

## What this exercise is not

- Not a weak-supervision exercise (Exercise 05).
- Not a statistical-stopping-rule exercise (that is `model-evaluation-engineer`'s territory).
- Not a production-labeling exercise — the "labeler" is an oracle for the simulation.

It is a small, honest experiment that gives you the one plot you will show a budget owner when asking for the AL budget on a real program.
