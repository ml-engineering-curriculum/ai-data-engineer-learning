# Chapter 07 — Weak Supervision and Data Programming (Snorkel)

## Why this chapter exists

Active learning (Chapter 06) targets the next human label. **Weak supervision** amplifies the labels you already have, by combining many noisy signals — heuristics, regexes, pretrained-model outputs, distant-supervision labels, user-feedback signals — into a single probabilistic training label. When it works, it converts a program with 5k hand-labeled items into a model trained on 500k weakly-labeled items without materially lowering downstream accuracy. When it fails silently, it produces a training set that looks large and is hidden-noisy.

This chapter is the operational chapter for weak supervision at labeling-program scale, framed around Snorkel and the data-programming literature. It covers:

- The data-programming framework (Ratner et al. 2016).
- The three Snorkel artifacts: **labeling functions**, the **label model**, the **discriminative model**.
- Where weak supervision is a force multiplier, and where it is a hidden-quality loss.
- Alternatives (rule mining, programmatic weak supervision, LLM-as-labeler).
- The weak-supervision block in the program package.

## The one reference you must have read

**Paper:** Ratner, De Sa, Wu, Selsam, Ré. *Data Programming: Creating Large Training Sets, Quickly.* NeurIPS 2016. <https://papers.nips.cc/paper/2016/hash/6523caa5b43756b56eaf8c1e4c77c0be-Abstract.html>.

Companion Snorkel papers to be aware of:

- **Snorkel system paper:** Ratner, Bach, Ehrenberg, Fries, Wu, Ré. *Snorkel: Rapid Training Data Creation with Weak Supervision.* VLDB 2018 (arXiv 1711.10160). <https://arxiv.org/abs/1711.10160>.
- **Snorkel MeTaL / multi-task extension:** Ratner, Hancock, Dunnmon, Sala, Pandey, Ré. *Training Complex Models with Multi-Task Weak Supervision.* AAAI 2019. <https://arxiv.org/abs/1810.02840>.
- **Snorkel (current open-source project):** <https://github.com/snorkel-team/snorkel>.

Read the 2016 paper sections 2 (the setting), 3 (the generative model), and 5 (empirical evaluation) before Exercise 05.

## The data-programming framework in one page

The setting:

- You have unlabeled data `X = {x_1, ..., x_N}` and want training labels `Y`.
- You have a labeling budget that covers a small gold set `L_gold` (say, 1–5k items) but not the full `N` (say, 100k–1M items).
- You can author `m` **labeling functions** (LFs) `λ_1, ..., λ_m : X → {-1, +1, abstain}` (or `{0, 1, ..., K, abstain}` in the multi-class case). Each LF is a cheap, noisy, programmatic labeler.
- Each `λ_j` has:
  - **Coverage** `c_j = P(λ_j(x) ≠ abstain)` — how often it fires.
  - **Accuracy** `a_j = P(λ_j(x) = y | λ_j(x) ≠ abstain)` — how often it is right when it fires.
  - **Correlation structure** with other `λ_k` — whether it is independent of other LFs or correlated.

The **label model** infers the latent label `Y` and the LF parameters `(a_j, c_j, correlations)` **without observing `Y` directly**, by modeling the joint distribution of LF outputs. The output is a soft label `P(Y | λ_1(x), ..., λ_m(x))` per item.

The **discriminative model** is trained on `(x, soft_label)` pairs. It generalises beyond the LF signal — it picks up features of `x` that the LFs could not easily encode.

Three things to take away:

1. Weak supervision **does not require per-LF accuracy estimates up front** — the label model infers them from LF co-firing patterns.
2. Weak supervision is **robust to individual LF errors** — as long as no single LF dominates coverage.
3. Weak supervision degrades gracefully when LFs are correlated — but requires an **extended generative model** (Snorkel's "dependency structure") to handle strong correlation without bias.

## What a labeling function looks like

A labeling function is a Python callable that returns a label or `ABSTAIN`:

```python
from snorkel.labeling import labeling_function
from snorkel.labeling import LabelModel, PandasLFApplier

ABSTAIN = -1
COMPLAINT = 0
SUPPORT = 1
FEEDBACK = 2

@labeling_function()
def lf_mentions_refund(record):
    return COMPLAINT if "refund" in record.text.lower() else ABSTAIN

@labeling_function()
def lf_mentions_password(record):
    return SUPPORT if any(w in record.text.lower() for w in ["password", "login", "reset"]) else ABSTAIN

@labeling_function()
def lf_pretrained_sentiment_negative(record):
    # distant supervision from a pretrained sentiment classifier
    return COMPLAINT if sentiment(record.text) < -0.5 else ABSTAIN

@labeling_function()
def lf_regex_praise(record):
    import re
    return FEEDBACK if re.search(r"\b(love|great|thank(s|\syou))\b", record.text.lower()) else ABSTAIN
```

LFs can call anything — regexes, lexicons, pretrained models, external APIs, heuristics, rule lists. A reasonable production LF set has 15–50 functions at moderate coverage (10–40% each) and moderate accuracy (0.6–0.9 each). More LFs is not better: adding redundant / correlated LFs biases the label model if the dependency structure is not modeled.

## The three Snorkel artifacts in the program package

### 1. Labeling functions (`labeling_functions.py`)

A Python module with the full LF set, versioned. Every LF has:

- A docstring explaining the signal it encodes.
- A test case or two (positive and negative example) so regressions are obvious.
- A `# source:` comment if it encodes a distant-supervision source (pretrained model version, KB table, LLM-as-labeler prompt + model).

### 2. Label model output (`label_model_outputs.parquet`)

For each item `x`, the label model emits:

- `soft_label` — `P(Y = c | λ(x))` for each class `c`.
- `hard_label` — argmax of `soft_label` with a tie-break rule.
- `label_model_confidence` — max soft-label probability (useful for filtering / curriculum).
- `lf_coverage_at_item` — how many LFs fired on this item.

The output table is itself a lineage event — produced by a specific `labeling_functions.py` version against a specific label-model configuration.

### 3. Discriminative model

The discriminative model is trained on `(x, soft_label)` pairs. In most production setups it is the final target model — a text classifier for the classification task you were going to train anyway. The important detail is that it is trained on soft labels, not hard labels, so a cross-entropy loss over the full predictive distribution is appropriate.

```python
from snorkel.labeling.model import LabelModel

lf_applier = PandasLFApplier(lfs=[lf_mentions_refund, lf_mentions_password,
                                   lf_pretrained_sentiment_negative, lf_regex_praise])
L_train = lf_applier.apply(df=train_df)   # shape: (n_items, n_lfs)

label_model = LabelModel(cardinality=3, verbose=True)
label_model.fit(L_train=L_train, n_epochs=500, log_freq=100, seed=42)
probs = label_model.predict_proba(L_train)   # (n_items, n_classes)
```

## Diagnostics every weak-supervision program should emit

Before trusting a weakly-supervised training set, measure:

- **Per-LF coverage.** Is any LF dominating? Any LF covering < 2% is nearly noise.
- **Per-LF accuracy on gold.** Use `L_gold` to estimate per-LF accuracy *against known labels*. Disagreement between the label model's inferred accuracy and the gold-measured accuracy is a signal that LFs are correlated in a way the label model does not model.
- **LF overlap and conflict matrices.** `snorkel.labeling.LFAnalysis` produces these natively. High overlap + low conflict → correlated LFs (bias risk). Low overlap → missing coverage.
- **Label model vs majority vote baseline.** Train a simple MV baseline and a label-model-based classifier on the same `L_train`. Compare downstream accuracy. If MV wins, drop the label model or redesign LFs.
- **Discriminative-model accuracy on gold.** The final gate: train on weak labels, evaluate on `L_gold`. This is the number you report in the program package.

Snorkel's `LFAnalysis` surfaces coverage / overlap / conflict in two lines:

```python
from snorkel.labeling import LFAnalysis
summary = LFAnalysis(L=L_train, lfs=lfs).lf_summary(Y=y_gold)
```

Treat the summary as part of the weak-supervision artifact, alongside the LF module.

## Where weak supervision wins — and where it loses

Weak supervision is a **force multiplier** when:

- Labels are easy to approximate with heuristics, lexicons, or pretrained models (named entities, document-level topics, obvious sentiment, domain-specific regex patterns).
- Classes are sufficiently separable that a 15–50-LF set gives > 70% coverage with moderate accuracy.
- The downstream model has a lot of representational capacity — it can learn to denoise the label distribution.
- You have a **non-trivial gold set** `L_gold` (1k+ items) to validate the label model and the final discriminative model. Without `L_gold`, you have no ground truth and cannot catch silent failures.

Weak supervision is a **hidden-quality loss** when:

- Labels require subtle human judgement (ambiguous sentiment, pragmatic inference, long-document reasoning, open-ended rubric scoring).
- LFs are **strongly correlated** (all three rely on the same lexicon). The label model treats them as independent evidence and under-corrects. Use `LFAnalysis.lf_dependencies()` and Snorkel's extended generative model.
- One class has **no cheap heuristic** — all LFs cover other classes. The under-covered class is systematically under-represented in training.
- The gold set is drawn from a different distribution than the weakly-labeled data (distribution shift). The discriminative model generalises away from the gold distribution.

The decision gate is: **train a small supervised baseline on gold-only `L_gold` and compare its test accuracy to the weakly-supervised model's test accuracy on the same test set**. If the weakly-supervised model does not meaningfully beat the gold-only baseline, the weak supervision is not pulling its weight — more labels, not more LFs, is the answer.

## Alternatives in the weak-supervision landscape

Snorkel is the reference; the space is larger:

- **Snorkel / `snorkel.ai`** — the open-source project (<https://github.com/snorkel-team/snorkel>) and the commercial Snorkel Flow platform descended from it.
- **Skweak** — <https://github.com/NorskRegnesentral/skweak>. Weak supervision specialised for sequence labeling (NER, span tasks); the Snorkel label-model approach does not natively handle structured outputs, Skweak does.
- **Cleanlab** — <https://github.com/cleanlab/cleanlab>. Related but distinct problem — finding and filtering **label noise** in an already-labeled dataset. Confident-learning-based. Pair with weak supervision: generate weak labels with Snorkel, then clean them with Cleanlab.
- **LLM-as-labeler** — in 2024–2025 production, calling a frontier LLM as a labeler is a common and powerful "LF". Treat it as *one* LF in a Snorkel pipeline, not as the whole labeling stack — the LLM has correlated errors with other LLM-based LFs and has a non-trivial cost per item. The Zheng et al. 2023 *LLM-as-a-Judge* framing (arXiv 2306.05685, <https://arxiv.org/abs/2306.05685>) is the primary reference.
- **Rule mining / classical weak supervision** — pre-Snorkel techniques: KnowItAll, DeepDive, distant supervision for relation extraction. Mintz, Bills, Snow, Jurafsky 2009 (*Distant supervision for relation extraction without labeled data*, ACL-IJCNLP 2009) is the historical primary. <https://aclanthology.org/P09-1113/>.

## The weak-supervision block in the program package

```
weak_supervision/
  labelling_functions.py            # the LF module, versioned
  labelling_functions.py.sha256
  lf_analysis_summary.yaml          # coverage, overlap, conflict, per-LF gold accuracy
  label_model_config.yaml           # cardinality, epochs, lr, dependency structure
  label_model_outputs.parquet       # soft labels + hard labels + confidence
  weak_discriminative_model/        # trained on soft labels
    model.safetensors
    metrics.yaml                    # gold-set evaluation, baseline comparison
  budget_delta.md                   # how many labels this replaced, at what accuracy cost
```

`budget_delta.md` is the most important of these for the budget owner. Its structure:

```
budget_delta.md:
  gold_labels_used:     1500
  weak_labels_produced: 120000
  effective_training_set_size: 120000  (with soft labels)
  baseline:
    gold_only_classifier_test_accuracy: 0.76 (CI 0.73, 0.78)
  weak_supervision:
    weakly_supervised_classifier_test_accuracy: 0.82 (CI 0.80, 0.84)
  delta: +0.06 test accuracy at ~equivalent human cost
  labels_saved_at_matched_accuracy: ~40000  (estimated from AL budget curve)
```

The delta is what the budget owner reads. The CI is what distinguishes a program that *measured* a result from one that *hoped* for a result.

## When to **not** use weak supervision

Skip weak supervision when:

- The labeling program is small (≤ 10k items). The LF-authoring overhead doesn't pay back under that size.
- Labels are expensive-per-item because they require genuine human judgement (long-form preference, essay grading, medical diagnosis). LFs will not approximate these cheaply.
- The downstream user demands provenance to a named annotator for every label (regulated settings). Weakly-supervised labels do not satisfy that requirement; disclose explicitly in the dataset card.

The right boundary is: weak supervision scales a labeling program; it does not replace a labeling program.

## What this chapter is **not**

- Not a label-noise-cleaning chapter. That is Cleanlab's problem space; mention it for completeness only.
- Not an LLM-as-judge chapter. Chapter 09 and `model-evaluation-engineer` own the judge-calibration problem.
- Not a generative-data chapter. Synthetic data generation is [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md).

## Summary

- **Weak supervision** combines many noisy, programmatic labeling signals (LFs) into a single probabilistic training label via a **label model** that infers LF accuracies and correlations from LF co-firing patterns.
- **Three artifacts:** labeling functions (code), label model outputs (soft labels), discriminative model (trained on soft labels).
- **Diagnostics are mandatory:** per-LF coverage, overlap, conflict, per-LF gold accuracy, label-model-vs-majority-vote comparison, discriminative-model-vs-gold-only-baseline comparison.
- **Force multiplier** when labels are heuristic-tractable and gold `L_gold` exists; **hidden-quality loss** when LFs are correlated, when a class has no cheap heuristic, or when gold is distribution-shifted from the weak labels.
- Pair with **active learning** (Chapter 06): AL targets human budget at the hard items; weak supervision scales the easy items.
- The deliverable is the `weak_supervision/` block of the program package, with `budget_delta.md` as the number the budget owner reads.

Chapter 08 picks up preference labeling for RLHF / DPO / ORPO — the operations chapter for the preference-pair pipeline `fine-tuning-engineer` consumes.
