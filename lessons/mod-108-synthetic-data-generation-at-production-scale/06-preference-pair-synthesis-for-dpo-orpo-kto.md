# Chapter 06 — Preference-Pair Synthesis for DPO, ORPO, KTO, and RLHF

## Why this chapter exists

DPO (Rafailov et al. 2023), ORPO (Hong et al. 2024), KTO (Ethayarajh et al. 2024), and classical RLHF (Ouyang et al. 2022) all consume **paired or signed preference signal** — `(prompt, chosen, rejected)` for DPO and ORPO, `(prompt, response, label)` for KTO, `(prompt, response, scalar_reward)` for RLHF — as the training input that teaches a policy *which* of two outputs is better. The quality of that signal is set in this chapter, before the training loop ever runs.

The pattern is not new. [`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md) Chapter 08 covered the *human* preference-labelling programme — pairwise comparison UIs, Likert rubrics, arena protocols. This chapter covers the *synthetic* preference-pair programme: generate N candidates with a model, score them with a (model or human) judge, pair or label them, hand off to `fine-tuning-engineer`.

The hand-off is sharp. **You own the pair-synthesis operation and the judge infrastructure. `fine-tuning-engineer` owns the training-time curation.** The hand-off contract is the preference-pair artifact: the pairs themselves, the judge verdicts, the pair-balance report, and the difficulty-stratification bucket counts.

Three papers pin the operational shape:

- Rafailov, Sharma, Mitchell, Ermon, Manning, Finn. *Direct Preference Optimization: Your Language Model is Secretly a Reward Model.* arXiv 2305.18290 (NeurIPS 2023). <https://arxiv.org/abs/2305.18290>. DPO's pair-synthesis operation.
- Hong, Lee, Thorne. *ORPO: Monolithic Preference Optimization without Reference Model.* arXiv 2403.07691 (EMNLP 2024). <https://arxiv.org/abs/2403.07691>. Same pair shape, different loss (no reference model needed).
- Ethayarajh, Xu, Muennighoff, Jurafsky, Kiela. *KTO: Model Alignment as Prospect Theoretic Optimization.* arXiv 2402.01306. <https://arxiv.org/abs/2402.01306>. Unpaired binary-labelled data.

## The recipe at a glance

```
prompt_pool  ─►  candidate_generator (N draws per prompt)  ─►  N candidates
                                                                    │
                                                                    ▼
                                                         judge_model_scoring
                                                                    │
                                                                    ▼
                                                         pair_construction
                                                                    │
                                                                    ▼
                                                     difficulty_stratification
                                                                    │
                                                                    ▼
                                                           pair_balance_report
                                                                    │
                                                                    ▼
                                                     preference_pairs.parquet
```

Three operational knobs set the signal quality:

1. **Candidate diversity.** Generating N candidates at the *same* temperature with the same prompt produces N near-identical responses — the judge has nothing to distinguish. Candidate diversity must be engineered.
2. **Judge fidelity.** The judge is the proxy for the preference signal. A noisy judge produces noisy pairs; a biased judge produces biased pairs.
3. **Pair construction policy.** How you pick which two candidates to pair determines the shape of the preference signal — hardest-vs-easiest, best-of-N-vs-worst-of-N, best-vs-median, etc.

Each knob is a chapter section.

## 1. Candidate diversity

Given one prompt, generate N candidates. The *distribution* of those candidates is what the judge then scores. If all N are near-copies, the judge is effectively picking randomly, and the preference signal is noise.

### Diversity mechanisms

Four mechanisms, from weakest to strongest:

- **Sampling temperature.** Draw N candidates at the same prompt with `temperature ∈ [0.7, 1.0]`. Easy; limited diversity.
- **Multi-generator.** Use multiple generators (different models, different sizes, different system prompts). Produces structurally different candidates; cost scales with the number of generators.
- **Perturbed prompting.** Prepend small variations to the system prompt ("Answer concisely." vs "Explain in detail."). Produces candidates that occupy different style regions.
- **Policy-anchored.** Include the *student* model as one of the generators. The student's candidate becomes the "losing" side in pairs where the teacher's candidate wins. This is the pattern used in iterative DPO.

Modern practice uses a mix. A common setup:

- 1 candidate from the student (policy-anchored).
- 2 candidates from a stronger teacher at `temperature=0.7`.
- 2 candidates from the stronger teacher at `temperature=1.0` with a style-varied system prompt.

The mix is a hyper-parameter; log it.

### Candidate-level filters

Before scoring, drop candidates that are clearly unsuitable:

- **Refusal.** The candidate is a refusal. (A refusal may be the *correct* response; retain if that is the operating target.)
- **Empty or malformed.** Zero-length, truncated by `max_tokens`, or structural-parse-fail.
- **Duplicate of a sibling candidate.** Exact or near-duplicate against another candidate for the same prompt. ROUGE-L or embedding cosine.
- **Prompt leak.** Candidate echoes the prompt instead of answering. Short-prompt case.

## 2. Judge-model scoring

The judge produces the preference signal. In the DPO era, there are three judge shapes:

### Shape A — Pairwise comparison judge

Prompt the judge with `(prompt, response_A, response_B)` and ask it to pick a winner. This is the *human* preference-labelling protocol (mod-107 Chapter 08) ported to a model judge.

- Simplest. The output is literally the preference pair the trainer consumes.
- Known position bias: judges favour the first candidate shown. Mitigate by scoring each pair twice with positions swapped; keep only pairs where both runs agree.
- Known length bias: judges favour longer responses. Mitigate by length-matching candidates or by explicitly instructing the judge to ignore length. The bias is stubborn; expect residual effect.

### Shape B — Scalar-score judge

Prompt the judge with `(prompt, response)` and ask for a numeric score (1–5 or 1–10) along one or more axes (helpfulness, correctness, honesty). Pair construction is downstream.

- Flexible: pair-construction policy is separable from scoring.
- Known calibration issues: the raw numeric range is unstable across prompts. Normalise within prompt (score each candidate at the same prompt, rank them, pair from the ranks) rather than globally.
- Multi-axis variants (as in HHH / UltraFeedback) emit a score per axis; pair construction can target specific axes.

### Shape C — Rubric judge

Prompt the judge with `(prompt, response)` and a structured rubric. The judge emits a structured verdict (per-criterion pass/fail or score). Closer to Likert / fine-grained labelling (mod-107 Chapter 08).

- Most informative per call; most expensive; most sensitive to rubric authoring.
- Suited to code or math, where structured criteria (compiles / passes tests / matches format) are more objective than overall "helpfulness".

### Position-bias and length-bias controls

Pin these as filter steps, not as post-hoc caveats:

- **Position swap.** For Shape A, run the judge twice with positions swapped. Only keep pairs where both runs agree. This halves the pair count; the retained pairs are higher-signal.
- **Length matching.** For Shapes A and B, if the length ratio between chosen and rejected is extreme, apply a length-matched judge prompt or drop the pair. A pair where the chosen is 3× the rejected's length carries mixed signal.
- **Judge-vs-reference calibration.** On a sample of pairs, compare the judge's verdict to a human verdict. Report judge-human agreement. If below ~70% on your task, the judge is not usable; swap it.

### Known reference judges

Three reference judges that are often (re-)used:

- **GPT-4-class model as judge.** The default in many DPO papers (including the original DPO paper for its AlpacaFarm-style setups). Known biases (length, position) apply.
- **Prometheus / Prometheus 2 (Kim et al. 2024).** Open-weight rubric judges. Released explicitly as judges, with published rubrics and training data.
- **A reward model trained on human preferences.** The classical RLHF route. More expensive to produce; higher fidelity when the preference domain is covered.

## 3. Pair construction

From N candidates and their scores, pick which two to pair. The policy decides what the preference signal teaches.

### Policies

- **Best-vs-worst.** Pair the highest-scoring candidate against the lowest-scoring. Strong, clear signal. Risk: the loser is so bad that the gradient saturates; useful only for early training.
- **Best-vs-median.** Pair the best against a middle-scoring candidate. The "harder" signal — teaches fine-grained preference. Preferred for later-stage DPO.
- **Adjacent-rank.** Pair rank-k against rank-(k+1) across all k. Produces many pairs per prompt; adjacent-rank pairs are the hardest for the policy to learn.
- **Threshold-based.** Any candidate above threshold T is "chosen"; any below is "rejected". Allows unbalanced ratios (useful when candidates are rarely bad).

The paper-typical setup is best-of-N vs worst-of-N for initial DPO; iterative DPO runs tighten to best-vs-median as the policy improves.

### Difficulty stratification

A pair's "difficulty" is the score gap between chosen and rejected (big gap → easy pair; small gap → hard pair). Bucket pairs by score-gap and track bucket counts. The trainer's curriculum often runs easy-first, hard-later.

### Pair balance

Four balance axes to track per corpus:

1. **Judge-score distribution.** The chosen set's judge-score distribution should not be degenerate (e.g., all 5/5). Same for rejected.
2. **Prompt-source balance.** If the prompt pool draws from multiple sources (public datasets, internal logs, synthetic prompts), track the per-source counts.
3. **Task-type balance.** From a pool with task-family tags (code, math, conversation, extraction, etc.), track per-task counts.
4. **Positive-vs-negative balance for KTO.** KTO's unpaired format needs a specific positive-to-negative ratio (the paper's prospect-theoretic weighting assumes balance within a tolerance); track and report.

### Difficulty-stratification bucket report

```yaml
# filters/judge_scores.parquet  (summary rows; the parquet is per-pair)
#
# ---
# difficulty_buckets:
#   gap_0.0_to_0.2:   count: 12_304     # hard pairs
#   gap_0.2_to_0.4:   count: 24_118
#   gap_0.4_to_0.6:   count: 18_742
#   gap_0.6_to_0.8:   count:  9_201
#   gap_0.8_to_1.0:   count:  3_018     # easy pairs
```

The bucket counts are an explicit hand-off artifact. `fine-tuning-engineer` uses them to schedule the curriculum or to subsample the easy buckets (which saturate training early).

## The DPO / ORPO / KTO format differences

All three consume preference signal; the artifact shape differs:

- **DPO / ORPO** — paired. Columns: `prompt`, `chosen`, `rejected`, (optional) `score_chosen`, `score_rejected`, `pair_source`, `difficulty_bucket`.
- **KTO** — unpaired. Columns: `prompt`, `response`, `label ∈ {good, bad}`, (optional) `score`, `source`, `difficulty_bucket`.
- **Classical RLHF** — scalar. Columns: `prompt`, `response`, `reward`, `source`.

Convert between shapes where cheap (paired → unpaired is dropping the pair structure; unpaired → paired requires re-pairing). Hand off the native shape your trainer consumes.

## The preference-pair manifest

```yaml
# recipes/preference_pair_config.yaml
prompt_pool:
  path: outputs/sft_corpus.parquet       # usually reused from Chapters 02-05
  sha256: 9a21...
  item_count: 50000

candidate_generation:
  generators:
    - id: student_policy_checkpoint_42
      temperature: 0.8
      candidates_per_prompt: 1
    - id: anthropic:claude-3-5-sonnet-20241022
      temperature: 0.7
      candidates_per_prompt: 2
    - id: anthropic:claude-3-5-sonnet-20241022
      temperature: 1.0
      system_prompt_variant: concise
      candidates_per_prompt: 2
  candidate_level_filters:
    drop_refusals: true
    sibling_dedup_rouge_l: 0.9
    max_tokens: 2048

judge:
  shape: pairwise               # pairwise | scalar | rubric
  judge_model: anthropic:claude-3-5-sonnet-20241022
  position_swap: true           # keep only agreement-after-swap
  length_match_tolerance: 1.5
  reference_calibration_path: filters/judge_vs_human_calibration.md

pair_construction:
  policy: best_vs_median        # best_vs_worst | best_vs_median | adjacent_rank | threshold
  pairs_per_prompt: 2
  difficulty_buckets:
    - [0.0, 0.2]
    - [0.2, 0.4]
    - [0.4, 0.6]
    - [0.6, 0.8]
    - [0.8, 1.0]

hand_off:
  format: dpo                   # dpo | orpo | kto | rlhf
  split: [train, 0.95] [eval, 0.05]
  recipient: fine-tuning-engineer

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

## The hand-off contract with `fine-tuning-engineer`

Three artifacts hand off:

1. **The preference-pair parquet** — one row per pair, with prompt, chosen, rejected, judge-score fields, difficulty bucket, source. Pin SHA.
2. **The pair-balance report** — the bucket counts above, plus the judge-vs-human calibration result, plus the refusal / malformed / dedup drop counts.
3. **The licence ledger reference** — pointer to `generator/licence_ledger.yaml` for all generators used.

The recipient's responsibility:

- Convert format if their trainer consumes a different shape (DPO ↔ ORPO ↔ KTO). You provide the native shape; they convert if needed.
- Run their own additional training-time filters (e.g., per-token-level diff between chosen and rejected; curriculum scheduling).
- Report back on training-time anomalies that trace to the pair-synthesis step.

The hand-off is logged as a mod-109 lineage event. The pair SHAs travel through the training run and land in the student model's card.

## The RLHF pattern (brief)

Classical RLHF (Ouyang et al. 2022, InstructGPT) uses a two-step: train a reward model on paired human preferences, then PPO-optimise the policy against the reward model. For *synthetic* RLHF, the pair-synthesis step feeds the reward-model-training step, not the policy-training step. The reward model is a fitted judge; the same judge-fidelity concerns apply, one level upstream.

For most 2024-era practice, DPO / ORPO / KTO have displaced classical RLHF in the synthetic-preference use case because they skip the reward-model training step. Chapter 06's operational pattern is therefore pitched at paired-and-unpaired formats first; the RLHF pattern is a generalisation.

## Where preference-pair synthesis breaks

### 1. Judge position bias not controlled

Symptom: chosen responses systematically occupy the "A" position in the judge prompt. The policy learns a surface artifact.

Catch: run every judge call with both position orderings. Count disagreement rate; if it is high, the signal is noise.

Fix: enforce position-swap-agreement in the filter; drop pairs that disagree.

### 2. Length bias

Symptom: chosen responses are, on average, 2× longer than rejected. The policy learns "longer is better" rather than the intended preference.

Catch: report length ratio in the pair-balance report. A length ratio above ~1.5 is a yellow flag; above 2.0 is a red flag.

Fix: length-matched candidate generation (same `max_tokens` across generators) or explicit length-controlled judge prompting.

### 3. Judge-generator overlap

Symptom: the judge and one of the candidate generators are the same model. The judge favours its own candidates.

Catch: document the model identity of every generator and the judge in the manifest. Overlap is a flag.

Fix: use an independent judge where possible. If the judge must overlap, calibrate judge-vs-human agreement and carry the calibration result through the hand-off.

### 4. Reward hacking of the judge

Symptom: candidates develop a stylistic pattern that the judge rates high (e.g., "I'd be happy to help with that!" openings) regardless of content quality. The policy learns the stylistic pattern.

Catch: human review a sample of high-scoring pairs. Do the high scores align with substantive quality?

Fix: rotate judges periodically; blend multiple judges; add human-labelled gold pairs to the training set at a small fraction to anchor the signal.

### 5. Difficulty bucket collapse

Symptom: all pairs fall into one or two buckets. The trainer cannot stratify.

Catch: bucket counts in the pair-balance report. If one bucket has 80% of the pairs, the signal is monotone.

Fix: adjust candidate diversity (more varied generators / temperatures), tighten pair-construction policy (best-vs-worst if too easy; adjacent-rank if too monotone).

## Summary

- Preference-pair synthesis is the operation that sets the DPO / ORPO / KTO / RLHF training signal. The signal quality is set here and cannot be recovered at training time.
- Three operational knobs: **candidate diversity**, **judge fidelity**, **pair construction policy**.
- Pin the generator mix, the judge identity, the position-swap and length-match controls, and the pair-construction policy in the manifest. Overlap between judge and candidate generator is a flag.
- Pair-balance and difficulty-stratification bucket counts are first-class hand-off artifacts. `fine-tuning-engineer` uses them to schedule the training curriculum.
- The hand-off is: paired parquet + balance report + licence-ledger reference. Native shape per target trainer (DPO / ORPO / KTO).
- The failure modes (position bias, length bias, judge-generator overlap, judge reward hacking, bucket collapse) are all knowable from the manifest and the balance report. Make them visible.

Chapter 07 picks up with the Curse of Recursion — the failure mode that bounds how many iterations of synthetic generation against a student checkpoint your pipeline can sustain.
