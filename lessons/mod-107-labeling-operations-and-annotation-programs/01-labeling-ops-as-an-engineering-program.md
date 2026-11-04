# Chapter 01 — Labeling Operations as an Engineering Program

## Why this chapter exists

A labeling program sits in a strange place in the pipeline. It is simultaneously a **data-engineering stage** (inputs, outputs, SLAs, artifacts), a **human-operations problem** (annotators, reviewers, vendors, timezones, payroll), and a **statistics problem** (inter-annotator agreement, cohort drift, active-learning sampling). When a team treats it as only one of the three, the other two leak quality losses that only show up at the end of a fine-tune, when a reward model is already miscalibrated and a release deadline is a week away.

This chapter sets the engineering frame for the rest of the module. The deliverable at the end is an **annotation program package** — not a pile of CSVs with labels. The package has seven artifacts, every one of which is created by a later chapter:

1. **Labeling guidelines** (Chapter 03) — the written contract.
2. **Tool configuration** (Chapters 02 and 03) — the project files the UI runs on.
3. **Calibration set** (Chapters 03 and 05) — the gold-labeled seed for qualification and drift.
4. **IAA report** (Chapter 04) — Cohen / Fleiss / Krippendorff against task structure.
5. **Active-learning + weak-supervision strategy** (Chapters 06 and 07) — the budget plan.
6. **Preference-labeling SOP** (Chapter 08) — the RLHF / DPO / ORPO protocol.
7. **Vendor package** (Chapter 05) — RFP, SLA, audit sample, arbitration.

Every chapter adds one block to this package. Chapter 01 is the defence of why the package exists.

## The three claims you must be able to defend

Pin these three. They are the ones you will cite when someone argues the labeling step is procurement, not engineering.

### 1. Crowdworker labels do not converge to expert labels by default (Snow et al. 2008)

**Paper:** Snow, O'Connor, Jurafsky, Ng. *Cheap and Fast — But is it Good? Evaluating Non-Expert Annotations for Natural Language Tasks.* EMNLP 2008. <https://aclanthology.org/D08-1027/>.

Key findings, in the shape you should be able to cite:

- Across five NLP tasks, non-expert MTurk workers approach or exceed expert quality **only after aggregation**. A single worker's labels are significantly noisier than a single expert's.
- The aggregation procedure (majority vote, weighted by per-worker reliability estimates) is *the* lever. Raw one-worker-per-item labels on nuanced tasks were below usable quality.
- Qualification (restricting workers to those who pass a calibration task) materially changes the quality distribution.

**Why it matters for your pipeline:** the common failure mode of "we'll just MTurk it" produces labels that read fine on a spot-check and poisoned downstream fine-tunes. Every step in this module — IAA, calibration, consensus, adjudication, vendor QA — is a direct consequence of Snow et al.'s finding that aggregation and qualification are not optional.

### 2. Labels are a modeled process, not a measured one (Ratner et al. 2016)

**Paper:** Ratner, De Sa, Wu, Selsam, Ré. *Data Programming: Creating Large Training Sets, Quickly.* NeurIPS 2016. <https://papers.nips.cc/paper/2016/hash/6523caa5b43756b56eaf8c1e4c77c0be-Abstract.html>.

Key findings:

- Noisy labeling signals (heuristics, external KBs, pretrained models, weak classifiers) can be combined into a **probabilistic training label** without ever observing the true label, as long as you model the agreement structure of the sources.
- The *label model* separates "how much should I trust each signal, and how correlated are they" from the "train a classifier on the result" step.
- When the signals cover a substantial fraction of the data, weak-supervision-amplified labels can **match human-labeled training sets at a fraction of the cost**. Where they can't (high-disagreement classes, ambiguous rubrics), the label model's uncertainty is itself an operational signal.

**Why it matters for your pipeline:** labels are not raw truth. They are a signal that comes with a noise model. Chapter 07 is the practical half; Chapter 04 is the statistical half. If you treat human labels as oracles and ignore their agreement structure, you will under-use them and over-trust them simultaneously.

### 3. The preference-labeling program is a model-training input, not a product survey (Ouyang et al. 2022)

**Paper:** Ouyang, Wu, Jiang, Almeida, Wainwright, Mishkin, Zhang, Agarwal, Slama, Ray, Schulman, Hilton, Kelton, Miller, Simens, Askell, Welinder, Christiano, Leike, Lowe. *Training language models to follow instructions with human feedback.* NeurIPS 2022 (arXiv 2203.02155). <https://arxiv.org/abs/2203.02155>.

Key findings:

- InstructGPT's behavioural step change over GPT-3 was driven by a **labeler-population-specific preference distribution**. The labeler pool's demographics, instructions, and training were part of the model definition.
- The paper documents its labeling-program decisions explicitly: labeler recruiting, calibration, instructions, inter-labeler agreement, and disagreement handling. Those are the InstructGPT-era baseline for what a serious preference program looks like.
- Reward-model miscalibration in the preference stage is **not fixable by scaling the RL stage**. Garbage preference pairs produce an aligned-to-garbage model.

**Why it matters for your pipeline:** Chapter 08 is the operational chapter for this. The preference-labeling program is as load-bearing to RLHF as the training-data mixture is to pretraining — this role owns it.

## The labeling-ops framing in ISO terms

As in every other module in this track, the framing lands cleanly on the ISO vocabulary you pinned in [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) and [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md):

| ISO 25012 characteristic | How a labeling program contributes |
|---|---|
| **Accuracy** | The label *is* the accuracy signal. IAA is the proxy for it. |
| **Consistency** | Consistency across annotators and cohorts is the labeling program's core output quality. |
| **Credibility** | A labeled dataset is credible only to the extent that its labeling procedure is. Guidelines, calibration, IAA, and audit evidence build credibility. |
| **Completeness** | Active learning (Chapter 06) is the mechanism for allocating the limited labeling budget across the input distribution so that the labeled subset is informative, not merely sampled. |
| **Traceability** | Every label traces to annotator, guideline version, timestamp, and QA verdict. This is a `mod-109` lineage event. |

**ISO/IEC 5259-3** (*AI — Data quality for analytics and machine learning — Part 3: Data quality management requirements and guidelines*) explicitly scopes the **annotation program** as a managed data-quality activity. The deliverable at the end of this module is the artifact that implements that scope for your slice. <!-- needs-research: confirm the 5259-3 clause numbers that pin annotation-program management; the published edition has reorganised across revisions. -->

## What "labeling" means in this module — and what it doesn't

The word "labeling" covers very different operations depending on the task. Pin these definitions:

- **Classification label** — one of a fixed set of classes per item (sentiment, topic, safety category).
- **Span / entity label** — a structured annotation over a region of the input (NER, relation extraction, slot filling, image bounding boxes, audio timestamps).
- **Preference pair** — given a prompt and two candidate responses, which is better by a stated rubric. This is the RLHF / DPO primary input.
- **Likert / rubric score** — for a single (prompt, response), a numeric score along one or more axes (helpfulness, honesty, harmlessness). The InstructGPT-era protocol.
- **Tool-calling trajectory** — a labeled sequence of (state, tool-call, observation) steps. The 2024-era agent-training supervised signal; Label Studio and Argilla support it with JSON structure; see Chapter 02.
- **Ranking / list-wise** — given a prompt and N responses, produce a full or partial ordering. Arena-style (Chiang et al. 2024) protocol; see Chapter 08.

Every one of these is a labeling program with the same seven artifacts in the package. The **operations** generalise; the **UI configuration** and the **agreement statistic** do not. Chapter 02 handles tool-to-modality mapping; Chapter 04 handles statistic-to-task mapping.

## What the labeling program is **not**

Mark the boundary explicitly — it is often where organisations misfile work:

- It is **not** benchmark construction. `model-evaluation-engineer` (peer, L30) owns benchmark design. You own the labeling SOP that produces the labels the benchmark consumes (Chapter 09).
- It is **not** preference-pair curation for a specific fine-tune. `fine-tuning-engineer` (peer, L30) owns preference-pair selection, filtering, and format conversion for a specific training run. You own the program that produces the raw preference pairs and their cohort metadata (Chapter 08).
- It is **not** PII detection. `ai-risk-engineer` (peer, L25) and [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md) own that. You own the policy for how annotators handle PII they encounter, and the audit trail of exposures.

## The economic shape — and why active learning is a labeling-ops concern

A production labeling program is cost-dominated. Per-item vendor prices are usually in the range of a few cents (simple categorical) to tens of dollars (long-form preference pair with a multi-axis rubric). The absolute numbers depend on vendor, modality, and rubric depth; the point is that **the labeling budget is finite and comparable in magnitude to the training-compute budget for mid-sized fine-tunes**. <!-- needs-research: avoid citing a specific per-item dollar figure; treat the shape qualitatively and point a reader to each vendor's current pricing page at authoring time. -->

Two operational consequences:

1. **Active learning (Chapter 06)** is not a nice-to-have. Choosing *which* 10k items to label out of a 1M-item pool is routinely a 3–10× efficiency lever at matched downstream accuracy, following the Settles 2009 survey literature.
2. **Weak supervision (Chapter 07)** is not a replacement for labels. It is a force multiplier on the labels you can afford — and a signal on which classes are hardest to label, which is itself a budget-planning input.

The budget plan is one of the artifacts in the program package. A labeling operation without a budget plan is a vendor contract with a dollar ceiling, not an engineering program.

## The program package — what this module produces

Here is the artifact tree you will accumulate over Chapters 02–08 and the six exercises:

```
labeling_program/
  guidelines.md                  # chapter 03, exercise 02
  guidelines.sha256              # pin the version
  tool_config/
    label_studio_project.xml     # chapter 02, exercise 01
    argilla_dataset.yml          # chapter 02, exercise 01
    prodigy_recipe.py            # chapter 02
    doccano_project.json         # chapter 02
  calibration_set/
    items.parquet                # chapter 03
    gold_labels.parquet          # chapter 03
    acceptance_threshold.md
  iaa_report.yaml                # chapter 04, exercise 03
  budget_plan.md                 # chapter 06, exercise 04
  weak_supervision/              # chapter 07, exercise 05
    labelling_functions.py
    label_model_outputs.parquet
  preference_sop.md              # chapter 08, exercise 06
  vendor_package/                # chapter 05, exercise 02
    rfp.md
    audit_sample.parquet
    sla.md
    arbitration.md
  program_report.yaml            # the single document that references all of the above
```

Every item in the tree has a chapter and (usually) an exercise that produces it. By the end of the module the tree is a real, signed, auditable deliverable for a real slice.

## Summary

- A labeling program is a **governed engineering operation**, not a procurement activity. Three findings pin this: Snow 2008 (aggregation is the lever), Ratner 2016 (labels are modeled), Ouyang 2022 (preference programs are training inputs).
- It implements ISO/IEC 25012 *accuracy*, *consistency*, *credibility*, *completeness*, and *traceability* characteristics simultaneously, and falls under the ISO/IEC 5259-3 annotation-program management scope.
- The deliverable at the end of the module is a **program package** — guidelines, tool configuration, calibration set, IAA report, budget plan, weak-supervision assets, preference SOP, vendor package — not a labeled CSV.
- The labeling budget is cost-dominated and comparable in magnitude to a training-compute budget. Active learning and weak supervision are the two load-bearing budget levers.
- Boundaries: `fine-tuning-engineer` owns run-time preference curation; `model-evaluation-engineer` owns benchmark design; `ai-risk-engineer` owns the PII and safety risk sign-off. You own the operation that produces the labels they consume.

Chapter 02 is the practical starting point — the open-source tool landscape, tool-to-modality mapping, and the first project you stand up.
