# Exercise 05 — DataComp-LM Style Ablation Mini Run

**Estimated effort:** 3 hours (wall-clock; actual GPU-hours depend on your hardware)
**Chapter dependencies:** [Chapter 07 — Data-centric ablation](../07-datacomp-lm-ablation-methodology.md); [Chapter 08 — Per-slice quality report handoff](../08-per-slice-quality-report-handoff.md) for the output shape

## Objective

Run a leave-one-out data-centric ablation over the filters in your slice at a small reference-model scale, using the DataComp-LM methodology. Produce a per-filter contribution table with eval deltas and KEEP / REMOVE / INVESTIGATE verdicts, and emit the `ablation` block that plugs into the per-slice quality report (Chapter 08).

This is the capstone of mod-104. For the first time in the module you are not just describing the slice — you are generating causal evidence about what in the slice actually matters for a downstream model.

By the end you will have:

- `recipes.yaml` enumerating a baseline recipe and one leave-one-out variant per filter.
- A shards directory per recipe (the slice re-emitted with the variant filter list applied).
- `cells/` with one subdirectory per `(recipe, seed)` cell: model checkpoints (or just the final one), training logs, eval JSON.
- `ablation/contributions.json` and `ablation/contributions.md` with per-filter verdicts.
- Everything reproducible via `make ablation SLICE=<id>`.

## Prerequisites

- A slice with at least ~1–10B tokens post-filter at the "all filters on" baseline. If yours is smaller, scale down the model and token budget proportionally and record both honestly; the methodology is scale-invariant even if the absolute numbers aren't comparable to published baselines.
- A GPU workstation or cluster. Minimum viable: 1×A100 or 1×H100 running a 100M-parameter model for ~1B tokens — that's a few hours per cell. Comfortable: 8×A100 running the DCLM small 160M for ~8B tokens per cell — order-of-magnitude one workstation-day for a 7-cell ablation.
- A pinned training stack. Three reasonable options:
  - **DCLM** (<https://github.com/mlfoundations/dclm>) — the reference. Use this if your hardware matches the reference configs.
  - **litgpt** (<https://github.com/Lightning-AI/litgpt>) or **nanoGPT** (<https://github.com/karpathy/nanoGPT>) — minimal; good for smaller compute.
  - **LLM Foundry** (<https://github.com/mosaicml/llm-foundry>) with Composer — production-grade determinism, more setup.
  Pin the repo at a specific git SHA.
- A pinned evaluation stack. The EleutherAI LM Evaluation Harness (<https://github.com/EleutherAI/lm-evaluation-harness>) at a specific SHA is the standard. Pick 4–5 tasks from Chapter 07 §"The eval suite": one broad knowledge (MMLU or ARC), one commonsense (HellaSwag), one reading comprehension (TriviaQA or SQuAD), one reasoning (GSM8K or BBH-lite).
- Benchmark decontamination already run on the baseline slice. **Critical** — any eval hit through contamination invalidates the whole ablation. Reuse Exercise 03's contamination scan.

## Problem statement

Run a leave-one-out ablation over the slice's filters. Produce per-cell eval scores with standard errors and a contribution table that calls each filter KEEP, REMOVE, or INVESTIGATE based on the measured deltas.

The ablation must:

- Hold **everything** fixed except the filter being ablated — model architecture, training tokens (not records; tokens), optimiser, LR schedule, seeds, eval suite, environment. Chapter 07 §"What to hold fixed" is the exact list.
- Train for at least ~1B tokens per cell (one can scale down on smaller hardware but document honestly).
- Run at least 1 seed per cell; use 2 seeds on any cell whose average delta is within ~1 standard error of zero.
- Produce the Chapter 07 §"A worked ablation record" YAML as the final artifact.

## Starter guidance

1. **Pick your filter list.** The filters that entered the slice have some pipeline order. For each major filter (say: `url_blocklist`, `bad_words`, `lid`, `perplexity`, `minhash_dedup`, `benchmark_decontam`), you will run one cell with that filter removed. Plus one baseline cell with all filters on. Total cells = 1 + N filters.

   - Do not remove `benchmark_decontam`. Mark it MANDATORY in the contributions table — removing it would contaminate the eval and invalidate every other cell.
   - If a filter is upstream of another (e.g. `lid` removes 60% of docs before `perplexity` sees them), removing `lid` changes `perplexity`'s input distribution. This is expected — the attribution is *per filter in the context of the pipeline*. Document this clearly.

2. **Freeze the training config.** Write `train_config.yaml` once; reference it from every cell. Minimum:

   ```yaml
   model:
     architecture: nanogpt_small    # or dclm_small_160m
     n_layer: 12
     n_head: 12
     d_model: 768
     tokenizer: gpt2                # or whatever your reference uses
     ctx_len: 2048
   optim:
     name: adamw
     lr: 3.0e-4
     betas: [0.9, 0.95]
     weight_decay: 0.1
     warmup_steps: 1000
     schedule: cosine
   training:
     tokens: 1_000_000_000          # scale to your compute
     global_batch_size_tokens: 524_288
     precision: bf16
     seed: 42
   ```

3. **Freeze the eval config.** `eval_config.yaml` with the EleutherAI LM Evaluation Harness SHA, the tasks, the few-shot counts, the decoding settings.

4. **Build each recipe's shards.** For each leave-one-out recipe, re-emit the slice with that filter disabled. The output is a `shards/<recipe_id>/*.jsonl.gz` directory. Record the per-recipe `token_count` and `doc_count`.

5. **Train each cell.** `train_cell.py <recipe_id> <seed>` writes to `cells/<recipe_id>/<seed>/`. Log the training curve (loss per step) to `training.log`; keep the final checkpoint.

6. **Eval each cell.** `eval_cell.py <recipe_id> <seed>` runs the pinned harness on the final checkpoint. Record the per-task accuracies and standard errors.

7. **Compute contributions.** `compute_contributions.py` reads every cell's eval JSON, computes `delta_vs_baseline` per task, aggregates across tasks (average pp delta), and assigns a verdict:

   - `KEEP`: average delta ≤ −0.5pp and consistent sign across tasks (at least 3/4 negative).
   - `REMOVE`: average delta ≥ +0.3pp across tasks, OR delta within noise AND filter is engineering-expensive to maintain.
   - `INVESTIGATE`: within noise band; needs a second seed or a task-specific audit.
   - `MANDATORY`: do not ablate (decontamination).

   These thresholds are starting defaults; document and justify the ones you use.

8. **Emit the ablation block.** Produce `ablation/contributions.yaml` in the Chapter 07 shape. Produce `contributions.md` as the readable summary.

## Deliverables

```
exercise-05/
  recipes.yaml                   # baseline + per-filter leave-one-out
  train_config.yaml
  eval_config.yaml
  shards/
    baseline/*.jsonl.gz
    no_perplexity/*.jsonl.gz
    no_url_blocklist/*.jsonl.gz
    no_bad_words/*.jsonl.gz
    no_lid/*.jsonl.gz
    no_minhash_dedup/*.jsonl.gz
  cells/
    baseline/
      42/
        training.log
        checkpoint_final.pt      # optional to retain
        eval.json
    no_perplexity/
      42/
        training.log
        eval.json
    …
  ablation/
    contributions.yaml           # Chapter 07 §"A worked ablation record" shape
    contributions.md             # readable summary with per-filter verdicts
    eval_evidence_sha256.txt
  code/
    make_shards.py
    train_cell.py
    eval_cell.py
    compute_contributions.py
    render_contributions_md.py
  README.md
```

## Acceptance criteria

- `recipes.yaml` enumerates a baseline plus one cell per major filter; decontamination is listed as MANDATORY and is not ablated.
- Every cell has the same model config, same training tokens, same optimiser, same seeds schedule, and same eval harness SHA. Prove this by running `diff` across the per-cell manifests and including the output in `cells/manifest_diff.txt` — the only differences should be the recipe label, shards path, and output paths.
- Every cell emits an `eval.json` with per-task accuracy and standard error.
- `ablation/contributions.yaml` validates against the Chapter 07 shape, includes `delta_vs_baseline` per task per cell, and assigns a verdict per filter.
- Decontamination evidence (from Exercise 03 or an equivalent) is linked in `README.md` and verified before the first training cell runs.
- At least one cell has 2 seeds — the one with the smallest absolute delta — and the contribution for that cell uses the mean ± SE of the two.
- The README documents scale explicitly: parameter count, training tokens per cell, hardware used, wall-clock per cell, total compute (GPU-hours).
- Any claim in `README.md` that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Two-scale ablation.** Run every cell at two model sizes (e.g. 100M and 300M) and compare contribution signs across scales. Document any sign flips — filter effects can change with scale and this is the most actionable finding you can produce.
- **Mixture-ratio ablation.** Instead of leave-one-out, vary the mixture ratio of the slice against a baseline mix. This previews the staff-ml-engineer level-40 work directly.
- **Composer DAG.** Instead of running cells sequentially, orchestrate them in a DAG runner (Airflow, Dagster, Prefect, or plain `snakemake`) with cell-level caching, so re-running one changed filter does not re-run the whole ablation.
- **Signed ablation artifact.** Sigstore-sign `contributions.yaml`. The level-40 consumer can now rely on the chain of custody from your pipeline to the mixture decision.
- **Monitor hook.** Add a `verify_ablation.py` that re-runs a single cell on a schedule ([`mod-111`](../../mod-111-data-quality-monitoring-and-regression/README.md)) and alerts if the contribution verdict changes. The ablation becomes a monitor, not a one-shot study.

## What this exercise is not

It is not a full DataComp-LM submission. The published DCLM benchmark runs at scales (400M, 1B, 3B, 7B) and compute budgets far beyond a workstation. The methodology is the same; the scale is what you honestly document.

It is not a replacement for human judgement about a filter's purpose. A filter might be MANDATORY not because it improves eval but because it enforces a policy (opt-out compliance, licence, decontamination). The contribution table surfaces the measurable; the KEEP / REMOVE / INVESTIGATE decision is still a human one informed by it.

It is not a one-shot. Any change to a filter or a downstream eval requires a re-run of the affected cells. Treat the ablation as a persistent, versioned artifact of the slice.
