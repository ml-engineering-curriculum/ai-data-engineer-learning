# Chapter 08 — Preference Labelling for RLHF, DPO, and ORPO

## Why this chapter exists

Preference labeling is the operational pipeline that produces the pairs and rankings that `fine-tuning-engineer` (peer, L30) consumes to train reward models, DPO policies, and ORPO-style preference-tuned models. It is the labeling program most directly tied to model behaviour; every flaw in the preference program is a flaw in the trained model.

This chapter covers the operational depth of the program: the **pairwise comparison UI**, the **Likert / fine-grained rubric**, the **calibration drift between cohorts**, the **chatbot-arena-style protocols**, and the **hand-off contract** to `fine-tuning-engineer`.

What this chapter is **not**: a reward-model-training chapter. `fine-tuning-engineer` owns the reward-model and DPO curricula. You own the preference-pair pipeline that feeds them.

## The three references that pin the framing

- **Ouyang et al. 2022** — *Training language models to follow instructions with human feedback.* NeurIPS 2022. arXiv 2203.02155. <https://arxiv.org/abs/2203.02155>. The InstructGPT-era baseline for a preference-labeling program's shape.
- **Rafailov et al. 2023** — *Direct Preference Optimization: Your Language Model is Secretly a Reward Model.* NeurIPS 2023. arXiv 2305.18290. <https://arxiv.org/abs/2305.18290>. The DPO method. Downstream consumer of your pairs; shapes the data format they need.
- **Chiang et al. 2024** — *Chatbot Arena: An Open Platform for Evaluating LLMs by Human Preference.* ICML 2024. arXiv 2403.04132. <https://arxiv.org/abs/2403.04132>. The arena-style protocol — pairwise battles, Elo / Bradley-Terry aggregation, open participation.

Also worth citing:

- **Bradley & Terry 1952** — *Rank Analysis of Incomplete Block Designs: I. The Method of Paired Comparisons.* Biometrika. <https://www.jstor.org/stable/2334029>. The statistical model underlying arena-style ranking.
- **Hong, Lee, Thorne 2024** — *ORPO: Monolithic Preference Optimization without Reference Model.* arXiv 2403.07691. <https://arxiv.org/abs/2403.07691>. Same preference-pair input, no SFT prerequisite.
- **Christiano et al. 2017** — *Deep Reinforcement Learning from Human Preferences.* NeurIPS 2017. arXiv 1706.03741. <https://arxiv.org/abs/1706.03741>. The foundational RLHF-from-preferences paper.

## The three preference-collection protocols

### 1. Pairwise comparison — the default

Given a prompt and two candidate responses `(A, B)`, the annotator picks one as better. Rafailov et al. 2023 (DPO) and Christiano et al. 2017 both consume this format.

Pros:
- Fastest per-decision.
- Robust — humans are measurably better at *comparing* than at *scoring*.
- Direct fit to DPO and reward-model training.

Cons:
- Loses magnitude information — "much better" and "slightly better" collapse to the same signal.
- Transitivity failures (A > B, B > C, C > A) are common and must be detected.
- A tie option is operationally necessary; forcing a choice injects noise.

UI shape:

```xml
<!-- Label Studio preference-pair config -->
<View>
  <Text name="prompt" value="$prompt"/>
  <View style="display:flex;">
    <View style="flex:1;padding:10px;"><Text name="response_a" value="$response_a"/></View>
    <View style="flex:1;padding:10px;"><Text name="response_b" value="$response_b"/></View>
  </View>
  <Choices name="winner" toName="prompt" choice="single" showInLine="true">
    <Choice value="a"/>
    <Choice value="b"/>
    <Choice value="tie"/>
    <Choice value="both_bad"/>
  </Choices>
  <TextArea name="rationale" toName="prompt" placeholder="Why?"
            required="true" maxLength="500"/>
</View>
```

Three operational decisions:

- **Position bias.** Annotators have a documented preference for the left-side response. Randomise A / B assignment per item, store the original-source mapping separately. Audit on cadence.
- **Length bias.** Longer responses are preferred even when quality is matched (Singhal et al. 2023, *A Long Way to Go: Investigating Length Correlations in RLHF*, arXiv 2310.03716). Report the length distribution of chosen vs rejected; if the correlation is strong, add a length-matched sample to the audit.
- **Rationale requirement.** Requiring a one-sentence rationale per decision adds 20–40% to per-item cost and 3–5× to downstream debuggability. On high-stakes programs, always required.

### 2. Likert / fine-grained rubric — the InstructGPT shape

Given a prompt and a single response, score it on each of N axes. The Ouyang 2022 InstructGPT-era protocol.

Typical axes:

- **Helpfulness** — does the response address the prompt?
- **Honesty** — are the claims factually correct?
- **Harmlessness** — is the response safe?
- **Instruction-following** — does the response follow the prompt's stated constraints?
- **Tone / style** — is the register appropriate?
- **Completeness** — does it address all parts of the prompt?
- **Reasoning** — if reasoning is required, is it sound?

Each on a 1–5 or 1–7 Likert scale.

Pros:
- Multi-axis signal — supports aspect-specific reward models and composite preferences.
- Interpretable — downstream users can audit per-axis.

Cons:
- Slow — 30–120 seconds per item vs 10–30 seconds for pairwise.
- **Lower IAA than pairwise** — humans score less consistently than they compare. A rubric item often lands at Krippendorff's α ≈ 0.4–0.6, where pairwise on the same data is α ≈ 0.6–0.8.
- Anchoring and ceiling effects — annotators drift toward the middle or toward the ceiling without calibration.

Argilla's `rg.RatingQuestion` and `rg.MultiLabelQuestion` support this natively:

```python
settings = rg.Settings(
    fields=[rg.TextField(name="prompt"), rg.TextField(name="response")],
    questions=[
        rg.RatingQuestion(name="helpfulness", values=[1, 2, 3, 4, 5]),
        rg.RatingQuestion(name="honesty", values=[1, 2, 3, 4, 5]),
        rg.RatingQuestion(name="harmlessness", values=[1, 2, 3, 4, 5]),
        rg.RatingQuestion(name="instruction_following", values=[1, 2, 3, 4, 5]),
        rg.LabelQuestion(name="overall_pass", labels=["pass", "fail"]),
        rg.TextQuestion(name="failure_mode", required=False),
    ],
    distribution=rg.TaskDistribution(min_submitted=3),
)
```

### 3. Arena / list-wise ranking — Chiang 2024

Given a prompt and N responses (from different models or generations), produce a ranking. Chatbot Arena's protocol:

- Annotators see **two** randomly-chosen responses at a time (anonymised), pick a winner, and the system aggregates pairwise results with a Bradley-Terry model to produce a global ranking.
- Open participation; the Arena uses crowd annotators at scale.
- Per-pair labels are cheap; the volume comes from aggregation.

Pros:
- Scales to many candidate models or many generations.
- Statistically well-grounded — Bradley-Terry 1952, with modern rating-system extensions (Elo, TrueSkill, Plackett-Luce for list-wise).
- Produces a continuous quality signal (Elo-like score) per response or per model.

Cons:
- Requires infrastructure (random pairing, aggregation, rating system).
- Open-pool annotators introduce adversarial and low-quality labels; quality control is heavier.
- Cost scales with the number of pairwise comparisons needed to produce reliable rankings (roughly `O(N log N)` with efficient pair-selection, more with random).

Use arena-style when:
- The downstream consumer needs ranking across many models or many candidate outputs.
- You have a large annotator pool and budget.
- You own, or can host, a Bradley-Terry / Elo aggregation pipeline.

## The hand-off format to `fine-tuning-engineer`

The pair format that lands downstream is the DPO-canonical shape:

```json
{
  "prompt": "...",
  "chosen": "...",
  "rejected": "...",
  "metadata": {
    "annotator_ids": ["ann_03", "ann_17", "ann_42"],
    "consensus_margin": 2,
    "guideline_version": "1.4.2",
    "cohort_id": "cohort_2026_q4_a",
    "generation_models": {"chosen": "modelA-v3", "rejected": "modelB-v1"},
    "position_randomized": true,
    "rubric_scores_chosen": {"helpfulness": 4, "honesty": 5, "harmlessness": 5},
    "rubric_scores_rejected": {"helpfulness": 3, "honesty": 4, "harmlessness": 5},
    "iaa_on_batch": {"statistic": "krippendorff_alpha", "value": 0.68}
  }
}
```

Three things to pin in the hand-off contract:

1. **Only ships pairs where consensus met the threshold.** Ties and both-bad are either dropped or handed off separately with a flag, never silently included.
2. **Each pair carries its cohort_id.** `fine-tuning-engineer` needs to detect cohort-drift effects at training time.
3. **Rubric scores, if collected, ride along with the pair.** They are not aggregated out. Aspect-specific DPO / per-axis reward modeling (Wu et al. 2023, *Fine-Grained Human Feedback*, arXiv 2306.01693) depends on it.

## Cohort drift — the operational problem

Preference labels are **not stationary**. The InstructGPT paper (Ouyang 2022) notes labeler-population dependence explicitly. Over a 6–12 month program:

- Annotators learn — their calibration against the guidelines drifts.
- Pool composition changes — vendors rotate contractors, new cohorts come in.
- The guidelines themselves change (Chapter 03's adjudication feedback loop).
- The model whose outputs you are rating changes — annotators see different kinds of candidates.

All four are real, and each biases the preference dataset in a different direction. Three operational controls:

1. **Cohort IDs on every pair.** A cohort is "the group of annotators labeling against a specific guideline version in a specific calendar window". Make it an explicit metadata field; downstream users can filter or weight by cohort.
2. **Cross-cohort calibration.** On cadence, assign a fixed calibration set (Chapter 03) to every active cohort. Compute cross-cohort agreement (Krippendorff's alpha across cohort pairs). A sudden drop indicates drift.
3. **Pool-stability metrics.** Report annotator-pool joiners / leavers per week, cohort-size histogram, and per-cohort IAA. A pool-stability report is part of the preference program's weekly cadence.

When cross-cohort agreement drops materially (e.g. α delta ≥ 0.10), **pause the hand-off and recalibrate**. Shipping drift-affected pairs to `fine-tuning-engineer` produces a reward model that unlearns stable preferences in favour of recent-cohort preferences. The paper to cite on this risk is Casper et al. 2023 (*Open Problems and Fundamental Limitations of Reinforcement Learning from Human Feedback*, arXiv 2307.15217). <https://arxiv.org/abs/2307.15217>.

## The preference-labeling SOP document

The preference-labeling SOP is one of the seven artifacts in the program package. Its structure:

```
preference_sop.md

1. Scope
   - What model(s) produce the candidates
   - What downstream consumer(s) use the pairs
   - What protocols are used (pairwise, rubric, arena)

2. UI configuration
   - Links to tool_config files
   - Position randomisation policy
   - Tie / both_bad handling

3. Rubric (if used)
   - Each axis defined with examples
   - Scale definition and anchors

4. Annotator pool
   - Qualification process
   - Pool size, cohort policy, rotation cadence

5. Consensus and adjudication
   - Annotators per pair
   - Aggregation rule
   - Adjudication SLA

6. Position / length / self-preference bias audits
   - Measurement cadence and thresholds

7. Cohort-drift controls
   - Cross-cohort calibration cadence
   - Drift thresholds for pause-and-recalibrate

8. Hand-off format to fine-tuning-engineer
   - JSON schema
   - Delivery cadence
   - Cohort metadata contract

9. Known limits
   - What biases the data is known to contain
   - What downstream users must guard against
```

The "Known limits" section is the one that saves the hand-off from being misused. Write it explicitly — "This dataset over-represents long responses; `fine-tuning-engineer` should length-match or down-weight long chosens if training a reward model."

## Biases to measure and report

Every preference-labeling program carries at least these biases. Measure and report them, don't hide them:

- **Position bias** — chosen position (after de-randomisation) frequency. Should be ≈ 50/50 at scale.
- **Length bias** — chosen vs rejected mean token count. Report the ratio and the within-pair length difference distribution.
- **Self-preference / model-identity bias** — if annotators see candidates from identifiable models, they may be biased. Anonymise candidates in the UI; if impossible, measure and disclose.
- **Rubric-axis correlation** — are the axes actually independent? Compute pairwise correlation; high correlation (> 0.8) across axes means the rubric is really one axis masquerading as multiple.
- **Annotator-pool demographic skew** — to the extent the vendor discloses it (Chapter 05), include it. Preference distributions are known to differ across demographics; the trained model inherits the labeler pool's distribution.

## Preference labeling and judge-vs-human calibration

A frequent operational shortcut is to replace human preference labelers with an LLM-as-judge. Zheng et al. 2023 (*Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*, arXiv 2306.05685) establishes that strong LLM judges correlate highly with human preferences on several benchmarks — but:

- Judges inherit their training distribution's biases.
- Judges are not calibrated across prompt styles without explicit calibration.
- Judge-labeled data for training reward models for the same model family introduces a correlated-noise risk.

`model-evaluation-engineer` (Chapter 09) owns the judge-vs-human calibration methodology. For the labeling-ops program's purpose, the rule is: **use LLM-as-judge as one labeling signal (a "labeling function" in the Chapter 07 sense), not as the preference source of truth**, until judge-vs-human calibration on your task is measured and documented.

## What this chapter is **not**

- Not a reward-model-training chapter. `fine-tuning-engineer` owns it.
- Not a DPO / ORPO curriculum chapter. `fine-tuning-engineer` owns training-side pair selection, filtering, and format conversion.
- Not a benchmark-of-models chapter. Chatbot-Arena-as-benchmark is `model-evaluation-engineer`'s territory.

## Summary

- **Preference labeling** is the operations pipeline that produces pairs / rubric scores / rankings for RLHF, DPO, and ORPO. The three primary references: Ouyang 2022, Rafailov 2023, Chiang 2024.
- **Three protocols:** pairwise (default), Likert rubric (InstructGPT-era), arena-style (Bradley-Terry over many pairs).
- **Known biases to measure:** position, length, self-preference, rubric-axis correlation, pool demographic skew. Report them; do not hide them.
- **Cohort drift** is operational and material. Cohort IDs on every pair, cross-cohort calibration cadence, pause-and-recalibrate thresholds.
- **Hand-off contract to `fine-tuning-engineer`** is a DPO-canonical JSON with metadata for cohort, guideline version, rubric scores, IAA, and known limits.
- The deliverable is `preference_sop.md` and the pair stream itself. Exercise 06 is the one you run.

Chapter 09 is the short closing chapter on coordination with `model-evaluation-engineer` for gold-eval-set labeling methodology.
