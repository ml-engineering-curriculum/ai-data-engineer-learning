# Chapter 07 — The Curse of Recursion and the Model-Collapse Defence

## Why this chapter exists

Every recipe in this module (Chapters 02–06) produces training data from a model. If that model was trained on synthetic data from an earlier model, and that earlier model was trained on synthetic data from the one before it, the pipeline is *recursive*. Shumailov et al. 2024 showed that recursive training against generated data causes the trained distribution to collapse — the model progressively loses mass in the tails of the real distribution, and the loss is cumulative.

This is not a hypothetical. At current practice, teams routinely:

- Fine-tune a student on a synthetic corpus produced by a teacher that was itself fine-tuned on synthetic data.
- Run iterative DPO where each round's preference pairs are generated against the latest student checkpoint.
- Augment a pretraining mix with synthetic data from a model trained on an earlier pretraining mix.

Each of these is a recursion vector with a measurable collapse risk. This chapter is the defence: how to recognise the vector, how to bound the iteration budget, how to anchor to real data, and how to detect collapse before it ships.

The primary reference: Shumailov, Shumaylov, Zhao, Papernot, Anderson, Gal. *AI models collapse when trained on recursively generated data.* Nature 631 (2024). <https://www.nature.com/articles/s41586-024-07566-y>. Earlier preprint: *The Curse of Recursion: Training on Generated Data Makes Models Forget.* arXiv 2305.17493 (2023). <https://arxiv.org/abs/2305.17493>.

## The failure mode in one picture

Three generations of a simple model trained recursively on its own outputs (schematic, drawn from the paper's figures):

```
Generation 0 (trained on real data)
  density ▆▆▆▆▆▆▆▆▆▆       # covers the full distribution including tails
         ◀── tails ──▶

Generation 1 (trained on Gen-0 outputs)
  density  ▆▆▆▆▆▆▆▆          # tails thinned
          ◀── tails ──▶

Generation 2 (trained on Gen-1 outputs)
  density   ▆▆▆▆▆▆            # tails nearly gone
           ◀── tails ──▶

Generation N
  density    ▆▆                # concentrated on the mode
            ◀── tails ──▶
```

Two separable effects contribute (both named in the paper):

- **Early-stage model collapse.** The variance of the generated-data distribution is less than the variance of the real distribution — the model does not perfectly fit the tails. Training generation N+1 on generation N's outputs inherits this truncation; across generations, the truncation compounds.
- **Late-stage model collapse.** The functional approximation itself drifts (the trained model differs from the true data distribution for reasons beyond sampling variance). Across generations, this drift compounds too, eventually producing a model whose distribution has little relationship to the original real distribution.

The paper's key technical claim: collapse happens *under all three* of (a) learned-approximation, (b) functional-approximation, and (c) statistical-sampling error. It is a robust effect, not a quirk of specific architectures.

## Why "just filter" is not enough

A naive response: "we filter our synthetic data for quality, so collapse shouldn't happen". This is partially true and partially wrong.

- **Where it is true.** High-quality filters keep egregious generator failures out of the training mix. If the filter is strong enough to bound the chosen-candidate distribution to a subset of the *real* distribution, you have partial anchoring.
- **Where it is wrong.** Filters select for what the judge values. They do not add back mass that the generator failed to emit. If the generator cannot produce samples from the long tail of the real distribution, no filter can create them. The tails are lost by *omission*, not by *inclusion of bad samples*.

The defence has three levers, in order of importance:

1. **Anchor to real data.** Keep real-data mass in every round of training. The anchor ratio is a release-eligibility parameter.
2. **Bound the iteration count.** Set a hard iteration budget; stop even if the next round looks fine on average metrics.
3. **Measure round-over-round distribution drift.** Make collapse visible before it ships.

## Lever 1 — real-data anchoring

The single most effective defence. Every training mix carries a minimum fraction of real (human-produced, not-generated-by-a-model-in-your-pipeline-history) data.

### The anchor ratio

Define the **anchor ratio** `α ∈ [0, 1]` as the fraction of training tokens drawn from real data (as opposed to synthetic data produced by any model in the current pipeline's recursion history).

- `α = 1.0` — no synthetic data. The baseline; collapse-free.
- `α ≥ 0.5` — majority real. Collapse risk is bounded; a Shumailov-style diagnostic should stay below threshold for several generations.
- `α ∈ [0.2, 0.5]` — synthetic-majority with real anchoring. The pattern most teams operate in. Monitor drift.
- `α < 0.2` — collapse-vulnerable. Iterate at your peril.
- `α = 0` — recursion vector. Shumailov's setup; collapse is the expected outcome.

The anchor ratio is not a constant across all training stages. Pretraining typically runs at very high `α`; SFT and DPO runs can tolerate lower `α` if the student model can still see real data in subsequent stages or evaluations.

### What counts as "real"

The distinction is **not** human-authored-vs-generated. It is **does-the-generator-share-ancestry-with-your-pipeline**.

- Text from a human-curated public corpus (Common Crawl, Wikipedia, licensed books) counts as real.
- Text from a generator trained on only real data (e.g., a GPT-3-era base model used once, before any of your synthetic cycles) counts as real *for this pipeline*, though not in an absolute sense.
- Text from last year's version of the student model counts as **not real** for the recursion diagnostic.
- Text from a teacher that was itself trained on last year's student counts as **not real**.

The anchor ratio is computed against the recursion-relevant definition, not the broader "human-vs-model" distinction.

### Anchor logging

```yaml
# recursion_defence/iteration_log.yaml
pipeline_run: 2026-01-15T14:00:00Z/sft-run-7
generation: 3           # the 3rd recursive iteration against student policy
upstream_models:
  - id: anthropic:claude-3-5-sonnet-20241022  # teacher
    treated_as: real_for_this_pipeline           # independent of our recursion
  - id: internal:llama-base-7b-sft-run4         # student, previous iteration
    treated_as: synthetic_in_recursion
training_mix:
  real_tokens: 1_200_000_000
  synthetic_tokens: 600_000_000
  anchor_ratio: 0.667
  anchor_ratio_source: tokens_weighted
```

## Lever 2 — iteration budget

Set a hard iteration budget. The budget is the maximum number of pipeline cycles where the student model's outputs can feed subsequent generation.

- **Budget `k = 1`.** A single pass: teacher generates, student trains, ship. No recursion.
- **Budget `k = 2`.** Second pass: student v1 generates candidates to pair against teacher candidates; student v2 trains. Common in iterative DPO.
- **Budget `k = 3` or more.** Deep recursion. Stay here only when the anchor ratio is high and drift metrics are controlled.

The budget is enforced at the manifest level: every generation run carries a counter, and the counter blocks further runs past the limit.

### Why 2–3 is the usual cap

Empirically (across the small number of published studies and widely-circulated internal reports), recursive iteration against a student policy shows diminishing returns past round 2–3 even when anchored, and shows measurable collapse past round 3–5 when poorly anchored. Treat 2–3 as the operating default; allow exceeding only with a Chapter 07 §"Lever 3" drift-metric release gate. <!-- needs-research: published empirical bounds on iteration count vary across papers and tasks; avoid committing to a single number. -->

## Lever 3 — round-over-round drift measurement

Collapse must be made visible. Three diagnostics, each tracked across iterations:

### Diagnostic 1 — perplexity on a held-out real-data tail

Pick a held-out real-data split (one that is explicitly tail-heavy — rare topics, long-tail entity mentions, uncommon formats). Compute perplexity under each generation.

- If perplexity on the tail rises faster than perplexity on the head, the model is losing the tail — a collapse signal.
- The relative rise, not the absolute number, is the diagnostic.
- Tracking perplexity on the *real* tail, not the *generated* tail, is critical. The generated tail will look fine to a model that produced it.

### Diagnostic 2 — distribution KL divergence against the generation-0 reference

Where the output distribution is computable (categorical outputs, discrete token distributions over a reference prompt set), compute `KL(gen_0 || gen_N)` at each generation. Rising KL → distribution drift → likely collapse.

For a text generator, this reduces to tracking the per-token categorical output on a fixed reference prompt set across generations.

### Diagnostic 3 — qualitative tail-category loss

Run a fixed, hand-authored prompt catalogue that specifically targets long-tail categories (rare entities, uncommon formats, out-of-training-distribution tasks). Review a sample of outputs per generation. The diagnostic:

- Does the model's output on the long-tail prompts get worse (hallucinations, refusals, format failures) across generations?
- Does the long-tail prompt coverage shrink (the model now refuses or diverts on prompts it previously handled)?

Qualitative is the hardest to automate and the most sensitive to early collapse. Keep a small human review loop in the drift-measurement step.

### Kill-switch thresholds

Pin thresholds in the manifest:

```yaml
# recursion_defence/kill_switch.yaml
thresholds:
  tail_perplexity_rise_pct_per_generation: 5.0    # halt if > 5%
  kl_divergence_cumulative_max: 0.3                # halt if cumulative KL > 0.3
  qualitative_tail_review:
    sample_size: 50
    reviewer_signoff_required: true
    halt_on_review: true
action_on_breach:
  halt_generation: true
  notify: ai-data-engineer-owner@example.com
  rollback_target: previous_iteration
```

Breach = release block. A generation that fails any threshold goes to review; it does not ship.

## The three recursion vectors in current practice

### Vector 1 — iterative DPO

Pattern: student v0 is SFT-trained on real data. For round `k`, generate preference pairs using student v(k-1) and a teacher. Train student v(k) on the pairs. Repeat.

Risk: the student's candidate diversity drops across rounds; the pair space shrinks; the policy narrows.

Defence: cap `k` at 2–3. Include diverse teacher candidates (Chapter 06 §"Candidate diversity"). Keep at least a small fraction of pairs where both candidates are teacher-sourced to anchor outside the student's distribution.

### Vector 2 — synthetic pretraining augmentation

Pattern: a student pretraining mix includes synthetic data from a generator model. If that generator was itself trained on synthetic data, you are at `k = 2`. At `k ≥ 3`, collapse is the expected outcome.

Defence: pretraining `α` must be very high (`> 0.9`). Synthetic data is a *supplement* in pretraining, not a *replacement*. Track generator ancestry; anything with recursion depth `> 1` is a flag.

### Vector 3 — distillation chains

Pattern: a teacher produces demonstrations for a student (Chapter 05). The student is then used as a teacher for a further student. Each hop is a recursion step.

Defence: track the distillation ancestry. A distilled model being used as a teacher is a flag; require Lever-3 diagnostics against a real-data reference before the next distillation step.

## How to run the ablation (Exercise 05)

The paper's experimental setup is reproducible at small scale in a day or two. The exercise structure:

1. Pick a small, well-understood model (e.g., a tiny GPT-2 or Pythia-70M) and a dataset with known tails (e.g., a subset of a Wikipedia revision history with explicit long-tail categories).
2. Train generation 0 on the real data.
3. Sample generation 0 to produce synthetic data; train generation 1 on 100% synthetic data.
4. Repeat for generations 2–5.
5. At each generation, measure Diagnostic 1 (tail perplexity), Diagnostic 2 (KL), Diagnostic 3 (qualitative review).
6. Plot the diagnostics across generations.
7. Repeat the sequence with `α = 0.5` (half real, half synthetic) and `α = 0.9` (90% real). Observe the collapse-rate change.

The ablation produces a run-local demonstration of the Shumailov failure mode and of the anchor-ratio defence. It is the single most convincing diagnostic you can show a stakeholder who argues that "we can just loop the generator forever".

## The defence log

The recursion-defence log is one of the seven artifacts of the program package (Chapter 01 §"The program package"):

```yaml
# recursion_defence/iteration_log.yaml
pipeline: synthetic_program/run-7
generations:
  - iteration: 0
    anchor_ratio: 1.0
    upstream_identity: [real_corpus/mod-103]
    diagnostics:
      tail_perplexity: 12.1
      kl_from_gen_0: 0.0
      qualitative_review: passed
  - iteration: 1
    anchor_ratio: 0.7
    upstream_identity: [real_corpus/mod-103, teacher/claude-3-5-sonnet]
    diagnostics:
      tail_perplexity: 12.4         # +2.5%
      kl_from_gen_0: 0.08
      qualitative_review: passed
  - iteration: 2
    anchor_ratio: 0.6
    upstream_identity: [real_corpus/mod-103, teacher/claude-3-5-sonnet, student/v1]
    diagnostics:
      tail_perplexity: 13.0         # +4.8%
      kl_from_gen_0: 0.18
      qualitative_review: passed
budget:
  max_iteration: 3
  kill_switch_breached: false
```

## Where the defence breaks

### 1. Anchor-ratio drift

Symptom: the anchor ratio is set per-iteration but drifts up over time as new synthetic data is added without proportional real data.

Catch: the iteration log is a release gate. If the anchor ratio falls below the threshold between iterations, the next iteration is blocked.

Fix: budget real-data refresh into the pipeline on the same cadence as synthetic generation.

### 2. "Real" definition drift

Symptom: last year's student model is still in the pipeline as a generator, but the iteration counter was reset after a model rename. The anchor computation misses the ancestry.

Catch: generator ancestry is a parquet column, not a free-text field. A rename is explicit; the lineage is unbroken.

Fix: enforce generator identity tracking through [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md). An un-provenanced generator defaults to "synthetic in recursion".

### 3. Diagnostic suite drift

Symptom: the held-out real-tail split used for Diagnostic 1 is accidentally included in later training data. Perplexity on the "tail" falls because the model memorised it, not because the model learned the tail.

Catch: the held-out tail split is firewall-protected like a benchmark (Chapter 10, [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)).

Fix: include the tail split in the Chapter 10 firewall's eval manifest.

### 4. Kill-switch bypass

Symptom: a release is slipping and the team overrides the kill-switch because the metrics are only slightly over threshold.

Catch: the override is a formal action logged in the iteration log, not a silent flag flip. Sign-off trail is traced to a named reviewer.

Fix: override is permitted but requires explicit sign-off from `ai-data-engineer-owner` and (for a release-scale slice) `ai-risk-engineer`. The override is captured in the dataset card.

## Summary

- Shumailov et al. 2024 is the canonical failure mode. Training recursively on generated data causes cumulative distribution collapse through three error sources (learned-approximation, functional-approximation, statistical-sampling).
- The defence has three levers, in order of importance: **anchor to real data** (anchor ratio `α`), **bound the iteration count** (hard budget), **measure round-over-round drift** (three diagnostics).
- "Real" is pipeline-relative: a model's output does not count as real if the model shares ancestry with your pipeline.
- Three vectors in current practice: iterative DPO, synthetic pretraining augmentation, distillation chains. Each needs the three-lever defence.
- The recursion-defence log is a release gate. Thresholds, kill-switch, and override path are explicit and signed.
- The ablation (Exercise 05) is the single most convincing diagnostic for the stakeholder question "why not just loop forever?"

Chapter 08 picks up with the operator libraries — NeMo Curator and Data-Juicer — that let you run the recipes in Chapters 02–06 at production scale without hand-rolling.
