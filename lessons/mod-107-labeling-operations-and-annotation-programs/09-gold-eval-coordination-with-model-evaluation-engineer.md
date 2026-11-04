# Chapter 09 — Gold-Eval-Set Coordination with `model-evaluation-engineer`

## Why this chapter exists

The seventh learning objective in this module is a **coordination** objective, not a construction one. `model-evaluation-engineer` (peer, L30) owns benchmark design, judge-vs-human calibration, statistical stopping rules, and the end-to-end evaluation report. **This role owns the labeling operations that produce the gold labels those benchmarks consume** — guidelines, calibration set, consensus, adjudication, IAA, vendor contract, cohort stability.

This chapter is short on purpose. It names the hand-off interface, pins the two shared artifacts, and lists the eight decisions that are owned by whom. If you are looking for benchmark-design depth — statistical power analysis, judge calibration curves, task-space coverage — read `model-evaluation-engineer`'s material. If you are looking for labeling-program depth — that is Chapters 01 through 08.

## The hand-off interface, one page

```
AI Data Engineer (you)                    Model Evaluation Engineer (peer)
--------------------------                -------------------------------
Owns the labeling operation               Owns the benchmark design

- Guidelines document                     - Task-space coverage
- Calibration set                         - Item selection / pool sampling
- Annotator pool management               - Statistical stopping rules
- Consensus / adjudication workflow       - Judge-vs-human calibration
- IAA computation                         - Evaluation reporting
- Vendor contract + audit                 - Benchmark versioning
- Cohort stability                        - Leaderboard / scoring

              shared, co-authored:
              ------------------
              - gold eval-set labeling SOP
              - gold eval-set IAA report
              - judge-vs-human agreement study plan
```

The two shared artifacts are the only ones with a joint signature. Everything else sits on one side or the other with a clear owner.

## The two shared artifacts

### 1. Gold eval-set labeling SOP

A gold evaluation set is a labeling program whose output is treated as the ground truth that other labelings (including LLM-as-judge outputs) are measured against. Three things distinguish it from a training-data labeling program:

- **Higher consensus** — typically 5+ annotators per item, with full adjudication of any disagreement.
- **Higher IAA target** — often the hardest achievable α on the task, because the gold set's noise is the ceiling on every downstream metric's precision.
- **Domain-expert annotators** — the pool is usually smaller and more expert than a vendor pool. For high-stakes benchmarks (medical, legal, safety), expert-only labeling is the norm.

The gold eval-set labeling SOP borrows from the preference SOP (Chapter 08) and the project configuration (Chapter 03), with these specific sections:

```
gold_eval_sop.md
1. Benchmark scope (link to model-evaluation-engineer's task-space document)
2. Expert annotator qualification
3. Item presentation protocol (how prompts are delivered, context length, tool access)
4. Consensus rule (typically 5-of-5 unanimous or majority-of-N with required adjudication)
5. Adjudication arbiter (named individual, benchmark-specific SME)
6. IAA target (higher than training-data programs)
7. Re-labeling policy for benchmark versioning
8. Blind-release policy (how the benchmark is released: full, held-out, private)
```

### 2. Gold eval-set IAA report

The same `iaa_report.yaml` from Chapter 04, with two additions:

- **Expert-vs-expert IAA** — the headline. On a true gold set, expert annotators should agree substantially; disagreement indicates genuine item ambiguity.
- **Item-level agreement** — items where the experts disagreed, by item. Items with low cross-expert agreement are candidates for exclusion from the benchmark (they are poorly specified) or for explicit flagging as "known-contested".

Joint signatories: you (program owner) and `model-evaluation-engineer`.

## The judge-vs-human calibration study

LLM-as-judge is a cost-effective substitute for human labeling at evaluation time, *if* it has been calibrated against human labels on your task. The calibration study is **jointly owned** — you produce the human gold labels, `model-evaluation-engineer` runs the judge and compares.

Reference: **Zheng et al. 2023** — *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.* NeurIPS 2023. arXiv 2306.05685. <https://arxiv.org/abs/2306.05685>. The paper and its position-bias / verbosity-bias / self-enhancement-bias analyses are the operational basis for judge calibration.

Study shape:

1. Select 500–2,000 items from the benchmark pool.
2. Collect `k ≥ 3` human labels per item under the gold eval-set SOP.
3. Run the candidate LLM judge(s) on the same items, with the same rubric.
4. Compute human-vs-judge agreement (Krippendorff's α or Cohen's κ treating the judge as one rater and the human consensus as the other).
5. Report per-axis agreement, per-prompt-class agreement, and the biases from Zheng et al. (position bias, length bias, self-preference, judge-vs-reference-model).
6. Decide: use the judge, use it with a correction factor, or do not use it.

The output is a `judge_calibration_report.yaml` with the same shape as `iaa_report.yaml` (Chapter 04), plus bias measurements and the decision.

## The eight shared decisions and who owns each

| Decision | Owner | Co-signer |
|---|---|---|
| What task-space the benchmark covers | model-evaluation-engineer | — |
| What item pool to sample from | model-evaluation-engineer | — |
| What qualifies an expert annotator | **you** | model-evaluation-engineer |
| Guidelines + rubric for the gold set | **you** | model-evaluation-engineer |
| Consensus and adjudication rule | **you** | model-evaluation-engineer |
| IAA target and threshold | model-evaluation-engineer | **you** |
| Statistical stopping rule (sample size, CI) | model-evaluation-engineer | — |
| Judge-vs-human calibration decision | model-evaluation-engineer | **you** |

The pattern: you own the labeling-operations side of every decision; `model-evaluation-engineer` owns the benchmark-statistics side. Shared decisions are co-signed.

## Statistical stopping rules — scope boundary

`model-evaluation-engineer`'s territory. The short summary of the interface:

- A benchmark reports a metric with a confidence interval.
- The width of the CI is a function of (i) items labeled, (ii) per-item label noise, (iii) model-output noise.
- The labeling program contributes (ii): the lower the per-item label noise (higher IAA on the gold set), the tighter the CI at matched item count, the smaller the sample needed to detect a given effect.
- The statistical stopping rule says "stop labeling when the CI is narrow enough to decide".

You contribute to this by:

- Reporting the gold-set IAA (Chapter 04).
- Keeping the labeling throughput on a predictable cadence so `model-evaluation-engineer` can forecast remaining time to the stopping point.
- Preserving all raw per-annotator labels (not just the consensus) so `model-evaluation-engineer` can model annotator noise in the CI calculation.

For depth on power analysis and sequential stopping for benchmark construction, defer to `model-evaluation-engineer`.

## What this chapter is **not**

- Not a benchmark-design chapter.
- Not a statistical power chapter.
- Not an LLM-as-judge framework chapter.

All three belong to `model-evaluation-engineer`.

## Summary

- Gold-eval-set labeling is **this role's operational responsibility** with a **co-signed hand-off** to `model-evaluation-engineer`.
- The two shared artifacts are the **gold eval-set labeling SOP** and the **gold eval-set IAA report**.
- The **judge-vs-human calibration study** is joint — you produce the gold labels, `model-evaluation-engineer` runs the judge comparison. Zheng 2023 is the reference.
- Eight joint decisions, four owned by you (annotator qualification, guidelines, consensus, labeling SOP), four by `model-evaluation-engineer` (task-space, item pool, stopping rule, judge calibration decision).
- There is **no paired exercise** for this chapter. The hand-off is exercised in `model-evaluation-engineer`'s material and in the `project-102-labelled-task-dataset-with-full-ops` and `project-103-regulated-use-case-training-data-package` projects at the end of the track.

This closes the module. The deliverable is the annotation program package from Chapter 01 — guidelines, tool configuration, calibration set, IAA report, budget plan, weak-supervision block, preference SOP, vendor package — and it is the artifact `fine-tuning-engineer` and `model-evaluation-engineer` consume as a precondition for their work.
