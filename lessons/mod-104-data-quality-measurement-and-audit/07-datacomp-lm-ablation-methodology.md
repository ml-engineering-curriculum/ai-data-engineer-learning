# Chapter 07 — Data-Centric Ablation: DataComp and DataComp-LM Methodology on Your Pipeline

## Why this chapter exists

Checks (Chapters 02–03) tell you the slice conforms. Audits (Chapters 04–05) tell you what the pipeline strips and whether the slice is sensible. Counters (Chapter 06) tell you the shape of what you kept. None of them tell you the question that actually matters to the model team: **does this filter earn its place?** A filter that seems principled on inspection can be eval-neutral or eval-negative; a filter that seems hacky can be the difference between a 2% and a 12% improvement on a downstream task.

DataComp (Gadre et al., NeurIPS 2023, <https://arxiv.org/abs/2304.14108>) and its language-model variant DataComp-LM (Li et al., NeurIPS 2024, <https://arxiv.org/abs/2406.11794>) formalise the only reliable way to attribute: **fix the model, vary the data, measure the eval delta.** Everything about the model — architecture, optimiser, hyperparameters, training compute, random seed — is pinned by the benchmark. The only variable is the dataset filtering recipe.

DataComp as a benchmark is run by the LAION/UW/Allen AI/others consortium at very large scale. DataComp-LM extends the shape to language models with reference models at 400M, 1B, 3B, and 7B parameters across multiple compute scales. You are unlikely to run the full benchmark; you *are* going to run the methodology at a smaller scale to make your own filter-contribution claim per slice. This chapter is how.

## The DataComp(-LM) methodology in one picture

```
         +-----------------+
         |  FIXED MODEL    |   architecture, optimiser, hyperparameters, seed,
         |  (reference)    |   training tokens or steps — all pinned
         +--------+--------+
                  |
                  v
         +--------+--------+       +------------------+
         |   VARIABLE      |  -->  |   EVAL SUITE     |  -->  eval_delta_per_recipe
         |   DATA RECIPE   |       | (fixed, pinned)  |
         +-----------------+       +------------------+
```

DataComp-LM's contribution is twofold: (a) the reference model implementations, training scripts, and eval suite are open-source at <https://github.com/mlfoundations/dclm>; (b) the paper's finding — a well-executed data-filtering recipe (DCLM-Baseline) yields gains that compound with scale across the reference-model sizes — demonstrates that the ablation methodology is *useful for decision making*, not just academically interesting.

For your own pipeline, the shape adapts to your budget:

| Scale | What's fixed | What varies | What you learn |
|---|---|---|---|
| **Pipeline-internal mini** | 160M- or 400M-parameter LM trained at a fixed compute budget | One filter toggled on/off at a time | Per-filter contribution to a small eval |
| **Within-scale sweep** | Same model size, several compute budgets | A whole recipe (ordered set of filters) | Does the recipe scale within a size |
| **Scale-up sweep** | Multiple model sizes, each at its compute-optimal point | A whole recipe | Does the recipe gain from scale |
| **Full DataComp-LM submission** | The benchmark's reference models and compute scales | A full-pipeline recipe output | Comparable to published baselines |

This module targets the first row: the pipeline-internal mini ablation per filter. It is what you can run in a day on one GPU workstation and emit as structured evidence in your per-slice quality report.

## What to hold fixed

The core discipline of a data-centric ablation is that *everything that is not the data* is the same across runs. Concretely:

- **Model code** — the exact repo + git SHA. The DCLM repo's `train.py` is a reasonable starting point; alternatives are `nanoGPT` for small-scale, `litgpt` or `megatron-lm` for larger. Pick one and stick with it for the whole ablation.
- **Model config** — architecture (layers, heads, dims), tokenizer, context length, optimiser (AdamW with fixed β's and ε), learning-rate schedule, weight decay, dropout, precision (fp16 / bf16).
- **Training tokens or steps** — the same per run. If one filter produces fewer tokens, either upsample to match (recorded as part of the recipe) or train for the same step count regardless.
- **Random seeds** — a fixed master seed; tokenization seed; sampling seed; init seed. Record all four.
- **Eval suite** — the exact set of tasks, the exact eval harness revision, the same evaluation protocol (few-shot count, decoding temperature, metric aggregation).
- **Compute environment** — the same GPU family, driver version, PyTorch version. Minor versions can change eval numbers by a few tenths of a point.

What varies: one knob at a time. "One knob" can be a filter being on or off, a threshold being 0.5 vs. 0.6, or a mixture ratio being 70/30 vs. 50/50. Change only one knob per paired run.

## The eval suite

Picking the eval suite is as consequential as picking the filter. Guidance:

- **Use a published, versioned suite.** The EleutherAI LM Evaluation Harness (<https://github.com/EleutherAI/lm-evaluation-harness>) is the standard. Pin the git SHA. Alternatives: HELM, BIG-bench, DCLM's own eval set. DCLM-LM reports on a suite that includes commonsense-reasoning, reading comprehension, closed-book QA, world-knowledge, coding, and math benchmarks; replicate as much of it as your compute allows.
- **Cover multiple capability axes.** A single eval (say, MMLU) overweights world-knowledge and underweights reasoning. The minimum is one each of: a broad knowledge eval (MMLU or similar), a commonsense eval (HellaSwag, ARC), a reading-comprehension eval (SQuAD, TriviaQA), and a reasoning eval (GSM8K, BBH). <!-- needs-research: confirm the current recommended set from DCLM-LM at read time; the paper's eval set evolved. -->
- **Decontaminate first.** Every slice that flows into training must have been decontaminated against every eval in the suite. Otherwise the eval delta is noise around data leakage. See [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) and Chapter 04 Axis 3 of this module.
- **Report confidence intervals.** The eval scores have statistical uncertainty — reported standard errors on MMLU-style accuracies are typically 0.5 to 1.5 points at the sample sizes the benchmarks use. A "+0.3%" delta inside the CI is not a filter contribution; it is noise. Published recipes typically run 2–5 seeds per cell and report mean ± SE.

## The per-filter ablation — the smallest useful unit

For a slice with filters `F_1, F_2, …, F_n` (acquire-stage check, URL blocklist, language-id, perplexity, dedup, benchmark decontamination, …), a leave-one-out ablation runs `n + 1` training runs: one with all filters on (the slice as shipped), and one with each filter removed.

Budget realistically:

- A 160M-parameter LM trained on ~8B tokens with the DCLM small reference config takes a few hours on 8×A100 — call it 10 GPU-hours. <!-- needs-research: verify exact wall-clock numbers against DCLM-LM small reference-model reports at read time. -->
- 8 filters × 10 GPU-hours = 80 GPU-hours. One seed per cell — order-of-magnitude one workstation-day.
- Multiple seeds per cell multiply the cost linearly. Start with one seed per cell; add a second seed on cells where the delta is close to the noise floor.

For each cell, record:

- The slice recipe (filters, thresholds, order).
- The model config and git SHA.
- The eval suite revision.
- The eval metric values per task, with standard error.
- The deltas vs. the baseline (all filters on).
- The ratio of training tokens actually used (to detect "filter X dropped so much data that the model is under-trained").

## A worked ablation record

```yaml
ablation:
  slice_id: cc-en-small-v1
  methodology: DataComp-LM (Li et al. 2024)
  reference_implementation: https://github.com/mlfoundations/dclm
  reference_implementation_sha: abc1234
  model:
    architecture: dclm_small_160m
    config_sha256: 8bf1…
    training_tokens: 8_000_000_000
    optimiser: adamw
    lr_schedule: cosine
    seed: 42
  eval_suite:
    repo: EleutherAI/lm-evaluation-harness
    sha: 5a1b2c3
    tasks: [mmlu, hellaswag, arc_challenge, triviaqa, gsm8k]
  cells:
    - recipe_id: baseline
      filters: [url_blocklist, bad_words, lid, perplexity, minhash_dedup, benchmark_decontam]
      tokens_consumed: 8_000_000_000
      eval:
        mmlu: {acc: 0.282, se: 0.004}
        hellaswag: {acc: 0.431, se: 0.006}
        arc_challenge: {acc: 0.236, se: 0.008}
        triviaqa: {acc: 0.107, se: 0.003}
        gsm8k: {acc: 0.021, se: 0.002}
    - recipe_id: no_perplexity
      filters: [url_blocklist, bad_words, lid, minhash_dedup, benchmark_decontam]
      tokens_consumed: 8_000_000_000
      eval:
        mmlu: {acc: 0.271, se: 0.004, delta_vs_baseline: -0.011}
        hellaswag: {acc: 0.418, se: 0.006, delta_vs_baseline: -0.013}
        arc_challenge: {acc: 0.228, se: 0.008, delta_vs_baseline: -0.008}
        triviaqa: {acc: 0.095, se: 0.003, delta_vs_baseline: -0.012}
        gsm8k: {acc: 0.018, se: 0.002, delta_vs_baseline: -0.003}
    - recipe_id: no_url_blocklist
      ...
  contributions:
    - filter: perplexity
      summary: |
        Removing perplexity filter: MMLU -1.1pp (SE 0.4pp, p-value ~0.01),
        HellaSwag -1.3pp (SE 0.6pp), consistent negative deltas across tasks.
        Decision: KEEP.
    - filter: url_blocklist
      summary: |
        Removing URL blocklist: all tasks within ±1 SE of baseline.
        Decision: candidate for simplification.
```

The `contributions` section is the engineering output: for each filter, a short statement of its measured contribution and a recommendation. This is the artifact the model team reads.

## Handing off to mixture strategy

At `staff-ml-engineer` (level 40), the mixture-strategy work takes per-filter contribution reports from many slices and composes a mixture ratio across them. Your contributions table feeds that work directly:

- Each slice's `ablation.contributions` is a vector of per-filter eval deltas.
- Across slices, the staff engineer looks for filter effects that compound or interact (filter X is eval-positive on code slices, eval-neutral on news slices; the mixture decision changes accordingly).
- The mixture weight on a slice is informed by the slice's overall eval delta vs. the mixture baseline — a per-slice ablation makes the slice *comparable*.

Your deliverable makes this comparability possible. Without the pinned methodology, every slice's "quality number" is in different units and the mixture work cannot proceed.

## Pitfalls

- **Comparing across model sizes / hyperparameters.** The whole methodology breaks. Pin everything but the data.
- **Training for different token counts across cells.** A filter that reduces the token budget by 30% and still matches the baseline is not neutral — it's doing more with less or being compensated by the LR schedule. Report tokens consumed; normalise where you can.
- **Running one seed and treating the delta as fact.** The delta is `mean(delta) ± SE`. Report both; call "significant" only what is outside ~2 × SE.
- **Running eval on a contaminated slice.** Any contamination against the eval suite invalidates the whole ablation. Decontaminate first (mod-106) and rerun the audit (Chapter 04 Axis 3).
- **Picking the eval suite after seeing the results.** The eval suite is part of the pipeline's contract. Commit to it before the ablation and record its SHA.
- **Ignoring compute-scale extrapolation.** Filter deltas at 160M can reverse sign at 7B (this is a documented pattern in DataComp-LM's results across scales). Note the scale you ran at; do not claim contributions at a larger scale you didn't measure.
- **Treating the ablation as a one-shot study.** Every change to the filter set requires a re-run of the affected cells. [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) covers how to monitor the ablation on cadence.

## What this module expects — a mini-ablation per slice

For the mod-104 deliverable:

- At least one leave-one-out ablation per *major* filter (the ones you spent engineering time on). "Major" is pipeline-specific; reasonable defaults are: language-id, perplexity, URL blocklist, near-dup, decontamination.
- At a small-model scale appropriate to your compute. 160M on 8B tokens is the DCLM small reference. 110M or 85M models with a smaller token budget are acceptable for the learning objective; record the scale honestly.
- One seed per cell is acceptable for a starting ablation; two seeds per cell where the delta is near the noise floor. Report SE on every cell's eval score.
- Decontamination checked *before* the ablation runs.
- A `contributions` table with a KEEP / REMOVE / INVESTIGATE verdict per filter.

## Tooling

- **DCLM** (<https://github.com/mlfoundations/dclm>) — reference models, training scripts, eval suite, and documented pipeline for the methodology.
- **EleutherAI LM Evaluation Harness** (<https://github.com/EleutherAI/lm-evaluation-harness>) — the standard eval runner. Pin the SHA.
- **litgpt** (<https://github.com/Lightning-AI/litgpt>) and **nanoGPT** (<https://github.com/karpathy/nanoGPT>) — minimal reference training codebases if DCLM's harness is too heavyweight for your compute budget.
- **MosaicML Composer** (<https://github.com/mosaicml/composer>) and **LLM Foundry** (<https://github.com/mosaicml/llm-foundry>) — production-grade training stacks with built-in determinism controls.
- **wandb / mlflow / aim** — experiment-tracking for the ablation cells. The cell's artifacts (eval JSON, training logs, model weights, data manifest) need to be addressable by `(slice_id, recipe_id, seed)`.

## Summary

- Data-centric ablation is the only reliable attribution of a filter's contribution: fix the model, vary the data, measure the eval delta.
- DataComp and DataComp-LM formalise the methodology and provide open reference models, training code, and eval suites.
- At the mod-104 scale, run a leave-one-out ablation per major filter on a small reference model (~160M params, ~8B tokens) and report per-task eval deltas with standard errors.
- Decontaminate before running the ablation; otherwise every eval delta is noise around leakage.
- Emit an `ablation` block in the per-slice quality report with `cells`, `contributions`, and a KEEP / REMOVE / INVESTIGATE verdict per filter. This is the handoff to the mixture-strategy work at `staff-ml-engineer`.
- Treat the ablation as part of the slice's release artifact, not a one-time study. Re-run on filter changes and on cadence.

Chapter 08 assembles the per-slice quality report — the single document that packages every Chapter-02-through-07 output and plugs into the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) and the mixture-strategy work at level 40.
