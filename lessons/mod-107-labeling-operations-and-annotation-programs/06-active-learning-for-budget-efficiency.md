# Chapter 06 — Active Learning for Budget Efficiency

## Why this chapter exists

A labeling budget is finite, and most of a candidate labeling pool is uninformative. **Active learning** is the family of techniques that selects *which* items to label next so that each labeling dollar spent yields the most model improvement per label. For most supervised tasks at mid-size scale, active learning is a 2–10× efficiency lever at matched downstream accuracy, following the Settles 2009 survey literature.

This chapter is the operational chapter for active learning in a labeling program. It covers:

- The three primary query strategies: **pool-based**, **stream-based**, and **query-by-committee**.
- When active learning helps and when it hurts.
- How to wire an active-learning loop into Label Studio, Argilla, or Prodigy.
- How to *prove* the budget / quality trade-off on a real task — Exercise 04 is where you do this.

## The one reference you must have read

**Paper:** Settles, B. *Active Learning Literature Survey.* Technical Report 1648, University of Wisconsin–Madison, 2009. <http://burrsettles.com/pub/settles.activelearning.pdf>.

It remains the field's canonical introduction. Read sections 2 (the problem setting), 3 (pool-based sampling with uncertainty), and 4 (query-by-committee) before Exercise 04. Settles's taxonomy of query strategies is the one you will cite in the budget-plan document.

A later textbook update (Settles 2012, *Active Learning*, Morgan & Claypool, <https://doi.org/10.2200/S00429ED1V01Y201207AIM018>) extends the survey with deeper treatment of structured output and cost-sensitive settings; the 2009 technical report is enough for the operational framing here.

## The three query strategies

Every production active-learning loop is a variant of one of three strategies.

### 1. Pool-based sampling

**Setting:** you have a large pool `U` of unlabeled items and a small labeled set `L`. On each round, train a model on `L`, score all items in `U`, pick the top-k most *informative* items, label them, add to `L`, retrain. Repeat.

**Informativeness criteria** (from Settles 2009 §3):

- **Least confidence** — items where the model's most-confident class has lowest confidence: `argmax_x (1 - max_y P(y | x))`.
- **Margin sampling** — items where the top-1 and top-2 class probabilities are closest: `argmin_x (P(y1 | x) - P(y2 | x))`.
- **Entropy** — items where the full predictive distribution has highest entropy: `argmax_x -Σ_y P(y|x) log P(y|x)`.

Least confidence is a special case of entropy for binary tasks; margin sampling is a specialisation that cares about the top-two boundary; entropy generalises to multi-class and is usually the default for 3+ classes.

Pool-based sampling is the right default when:

- You have a fixed pool of candidate items up front.
- Labeling is batched (vendor or in-house with batched throughput).
- Retraining the model between rounds is affordable.

### 2. Stream-based sampling

**Setting:** items arrive one at a time; you decide per item whether to pay to label it or discard it. No large pool exists.

The decision rule is a threshold on the same informativeness score — if `H(P(·|x)) > threshold`, label it; otherwise discard. The threshold is tuned against the budget.

Stream-based sampling is the right default when:

- Data is a live stream (user feedback, inference traces, web crawler output).
- You cannot afford to store or score the full pool.
- Labeling is incremental and cheap per item (in-house annotators, not vendor batches).

### 3. Query-by-committee

**Setting:** train `C` models (or committee members — could be bagged copies, differently-seeded models, differently-initialised models, or structurally different models). On each item, measure **disagreement** among the committee. Items with highest disagreement are labeled next.

Disagreement measures:

- **Vote entropy** — treat each committee member's predicted class as a vote; compute entropy over the vote distribution.
- **KL divergence to the mean** — average per-member predictive distributions; measure each member's KL from the mean; sum.
- **Soft vote entropy** — entropy over the mean predictive distribution.

Query-by-committee is the right default when:

- Single-model uncertainty is poorly calibrated (common with deep models — see the calibration literature).
- You can afford to train C models.
- You need an easily-defensible informativeness measure (committee disagreement is intuitive to a reviewer).

**Reference:** Seung, Opper, Sompolinsky. *Query by Committee.* COLT 1992. <https://dl.acm.org/doi/10.1145/130385.130417>. The primary source.

## When active learning hurts

Settles and later work are explicit that active learning is not a universal win. Three regimes where naive AL **loses** to random sampling:

1. **Early-round cold start.** With `|L| < 50–200` examples (task-dependent), the model is too noisy to produce useful uncertainty estimates. A random initial seed set beats AL-selected items.
2. **Class imbalance.** AL with uncertainty-only queries concentrates sampling on the decision boundary, which is often dominated by the majority class in imbalanced tasks. Minority classes remain under-sampled. **Fix:** stratified or diversity-aware AL (BALD, Core-Set, BatchBALD).
3. **Distribution shift.** AL-selected items are not IID with the eventual deployment distribution. The labeled set becomes skewed toward *informative-for-current-model* items, which are not necessarily representative.

The three regimes map to three operational safeguards:

- Seed `L` with 100–500 random items before the first AL round.
- Monitor class-balance in `L` as a function of round. If a class is under-represented, switch to a diversity or balanced query strategy.
- Keep a **holdout evaluation set that is sampled IID from the deployment distribution**, regardless of how `L` was constructed. Evaluate on this set, not on an AL-selected validation set.

## Diversity-aware extensions — BatchBALD and Core-Set

Two extensions worth naming because they come up in modern DL-era AL work:

- **BALD** (Houlsby, Huszár, Ghahramani 2011) and **BatchBALD** (Kirsch, van Amersfoort, Gal 2019) — Bayesian disagreement objective, with the batch version correcting for redundancy across items picked together. <https://arxiv.org/abs/1906.08158>.
- **Core-Set** (Sener, Savarese 2018) — picks items that maximise coverage of the feature-space of the pool; diversity-first rather than uncertainty-first. <https://arxiv.org/abs/1708.00489>. Particularly useful for the early-round cold start regime.

For a practical production program, start with pool-based entropy or margin sampling. If you observe class imbalance or redundancy in labeled batches, upgrade to BatchBALD or add a Core-Set diversity term. Do not start with the complicated strategy.

## The active-learning loop — the operational shape

```
1. Seed: random sample N_seed items from U, label them, set L = seed.
2. Train model M on L.
3. Score all items in U with M.
4. Pick top-k items by informativeness criterion.
5. Label picked items, add to L, remove from U.
6. Evaluate M on IID held-out set. Record accuracy / F1 / IAA-with-gold.
7. If budget remaining and quality below target, go to step 2.
```

Three operational knobs:

- **Batch size `k`.** Too small → many retraining rounds, operational overhead dominates; annotators wait between rounds. Too large → within-batch redundancy. Practical batches are 100–1000 items.
- **Retraining frequency.** Not every batch needs a full retrain — a warm-start on new items is usually enough for intermediate batches, with a full retrain every 3–5 rounds.
- **Stopping rule.** Either budget-exhausted or quality-threshold-reached. Define both before running; the actual stop is whichever happens first.

## Wiring into the labeling tool

Every production labeling tool supports some form of "pre-populate the queue with model predictions and prioritise by uncertainty":

### Label Studio

Label Studio has a first-class **ML backend** API. Your model implements a small HTTP service that produces `predictions` for each task; Label Studio displays them to the annotator as a starting point and sorts the queue by the prediction's `score` field. See <https://labelstud.io/guide/ml.html>.

```python
# A minimal Label Studio ML backend prediction response
{
    "model_version": "v3",
    "score": 0.42,                 # model uncertainty / informativeness
    "result": [{"from_name": "label", "to_name": "text", "type": "choices",
                "value": {"choices": ["complaint"]}}]
}
```

Annotators label in queue order. If you want hard enforcement of AL order, sort in your backend before posting tasks to Label Studio.

### Argilla

Argilla stores `rg.Record.suggestions` and `rg.Record.metadata` fields with every record. A common pattern: run your model on the pool in a batch job, write a `score` metadata field, and configure the Argilla UI to sort by that field.

```python
records = []
for item in pool:
    probs = model.predict_proba([item["text"]])[0]
    entropy = -float(np.sum(probs * np.log(probs + 1e-12)))
    records.append(rg.Record(
        fields={"text": item["text"]},
        suggestions=[rg.Suggestion(question_name="label", value=str(np.argmax(probs)))],
        metadata={"al_entropy": entropy, "model_version": "v3"},
    ))
dataset.records.log(records)
```

Then `dataset.records.filter(...)` and sort-by-metadata in the UI drives annotator order.

### Prodigy

Prodigy is explicitly built for model-in-the-loop AL. The `textcat.teach` and `ner.teach` recipes implement pool-based uncertainty sampling natively; the annotator sees items in uncertainty order and the model updates after each label. See <https://prodi.gy/docs/active-learning>.

## The budget / quality trade-off — proving it

The operational deliverable of this chapter (and Exercise 04) is a plot that defends the AL choice to a budget owner:

- **X axis:** labels consumed.
- **Y axis:** downstream task metric on an IID held-out set.
- **Lines:** random baseline, pool-based entropy AL, (optional) BatchBALD or Core-Set.

A well-run experiment shows the AL curve reaching a target metric at 20–50% of the labels the random baseline needs. The exact ratio is task- and model-dependent — your number *is* the budget case you can make.

Pitfalls to avoid:

- **Training the baseline with the AL-selected items.** The random baseline must use a random sample of the same size.
- **Reporting accuracy on a non-IID held-out set.** The held-out set must be sampled IID from the deployment distribution, not from `U` at a particular round.
- **Fixing hyperparameters once and running them on every round.** Early-round models are small-data; later rounds might benefit from different hyperparameters. Hold them fixed if you want a clean comparison, but call this out.

## When to call the budget plan "done"

The budget plan is one of the seven artifacts in the program package (Chapter 01). A production budget plan covers:

```
budget_plan.md
  1. Task definition and target metric
  2. Baseline: random sample at 1x, 2x, 4x volume — expected metric
  3. AL strategy picked, with Settles 2009 citation
  4. Expected AL curve (from the pilot on a 5-10k subset)
  5. Budget request: dollars, items, weeks
  6. Stopping rules: budget limit, quality target
  7. Fallback: what to do if AL curve flattens below target
```

The "fallback" clause is often missing. Active learning does not *always* beat random. Having a pre-decided fallback (switch to Core-Set, add a weak-supervision amplifier per Chapter 07, or accept a lower quality bar) means the program does not stall when the AL curve underperforms.

## Where active learning sits relative to weak supervision (Chapter 07)

Active learning and weak supervision are complementary, not competitive:

- **Active learning** chooses which items a human labels — targeted human effort.
- **Weak supervision** amplifies the labels you have — synthetic labels from heuristics and the label model.

The most effective production programs do **both**: use AL to pick the small set of items for which human judgement is highest-value, and use weak supervision to produce a larger training set out of easy-to-heuristic-label items.

Chapter 07 picks up weak supervision. The two chapters together are the full budget-lever answer.

## What this chapter is **not**

- Not an AL theory chapter. Settles 2009 covers the theory; this chapter is the operational shape.
- Not a deep-learning-specific AL chapter. Many modern AL papers target deep architectures with calibration issues; a separate specialist topic that you should pursue on your task.
- Not an evaluation-set-construction chapter. The IAA held-out set for evaluation (Chapter 09) is sampled differently from the AL training stream.

## Summary

- **Active learning** picks the next items to label to maximise model improvement per labeling dollar. Settles 2009 is the one reference you cite.
- Three strategies: **pool-based** (score the pool, pick top-k), **stream-based** (threshold on incoming stream), **query-by-committee** (train C models, measure disagreement).
- Three regimes where AL loses to random: **cold start**, **class imbalance**, **distribution shift**. Seed random, monitor class balance, keep an IID held-out set.
- Diversity-aware extensions — **BatchBALD**, **Core-Set** — fix within-batch redundancy and cold-start problems.
- Wire the AL loop into Label Studio (ML backend), Argilla (metadata + sort), or Prodigy (`teach` recipes).
- Prove the budget / quality trade-off on your own task with an AL-vs-random experiment — Exercise 04 is the one you run.
- Pair with **weak supervision** (Chapter 07) for the full budget-lever story.

Chapter 07 picks up the other half of the budget lever — amplifying labels with heuristics, the Snorkel / data-programming framework.
