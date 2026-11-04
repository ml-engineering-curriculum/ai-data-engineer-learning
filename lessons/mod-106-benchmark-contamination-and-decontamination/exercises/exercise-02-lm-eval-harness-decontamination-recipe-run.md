# Exercise 02 — LM Eval Harness Decontamination Recipe Run (vs the Dolma Practice)

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 06 — EleutherAI LM Eval Harness vs the Dolma practice](../06-lm-eval-harness-vs-dolma-practice.md), [Chapter 03 — N-gram overlap detection](../03-n-gram-overlap-detection.md)

## Objective

Run **both** public decontamination recipes — the EleutherAI LM Evaluation Harness recipe and the Dolma-style decontamination — on the same small training-corpus slice against the same pinned eval manifest. Normalise the outputs to a common schema, compute the per-benchmark overlap and disagreement, write a one-page `recipe_comparison.md` that picks a recipe (or a per-surface hybrid) for your lab, and extend `decontamination_log.yaml` with the recipe-choice justification in §1 `toolchain.recipe_choice`.

By the end you will have:

- Two per-benchmark removal manifests — one from the EleutherAI harness, one from Dolma — normalised to the Chapter 03 `removal_manifest.parquet` schema.
- A per-benchmark breakdown: documents flagged by EleutherAI only, by Dolma only, by both, and the per-benchmark removal-rate delta.
- A `recipe_comparison.md` with the picked recipe (or hybrid) and the reasoning.
- The scoring-alignment round-trip test (Chapter 06 §"alignment with the scoring harness") passing against both recipes.

## Prerequisites

- Python 3.11+ with `lm-evaluation-harness` (installed from <https://github.com/EleutherAI/lm-evaluation-harness>) and `dolma` (installed from <https://github.com/allenai/dolma>).
- The training-corpus slice and `eval_manifest.v1.yaml` from Exercise 01.
- Read Chapter 06.

**Note on version drift.** Both toolkits move between the time this exercise is written and the time you run it. The decontamination subtree in the LM Eval Harness has been reorganised across releases, and Dolma's filter config format has evolved. **Pin both toolkit git SHAs explicitly** in `decontamination_log.yaml` §1 `toolchain`, and mark the specific subcommand / config paths you used with `<!-- needs-research: ... -->` if you cannot verify them against the current README.

## Problem statement

Run both recipes end-to-end on the Exercise 01 slice. Compare their outputs and justify a recipe-choice decision.

1. **Reproduce Exercise 01 as the control.** Your in-house n-gram firewall's `removal_manifest.parquet` from Exercise 01 is the baseline. The two public recipes will be compared against it and against each other.
2. **Run the EleutherAI harness recipe.** Invoke the harness's decontamination script (per its current README) with the manifest's benchmarks. Capture the raw output, then convert to the common schema.
3. **Run the Dolma recipe.** Configure a Dolma mapper with the decontamination filter against the same manifest. Capture the raw output, then convert to the common schema.
4. **Normalise schemas.** Write a small adapter that converts each toolkit's native output to `removal_manifest.parquet` with the Chapter 03 columns.
5. **Compute the three-way comparison.** Per benchmark, count: EleutherAI only, Dolma only, both, in-house Exercise 01 only, union. Produce a table and a Venn-style per-benchmark summary.
6. **Round-trip test the scoring alignment.** Take 20 eval items, render them as the LM Eval Harness would render them during scoring, inject each verbatim into a synthetic training document, and confirm all 20 are flagged by both recipes. Any miss is a normalisation drift bug and must be documented.
7. **Pick a recipe and update the log.** Write `recipe_comparison.md` with the pick (EleutherAI-only / Dolma-only / hybrid) and extend `decontamination_log.yaml` §1 `toolchain.recipe_choice`.

## Starter guidance

1. **Pin both toolkits.** Clone each at a fixed git SHA. Record the SHA in `decontamination_log.yaml` §1 `toolchain.engine_git_sha` (one per recipe if hybrid).

2. **Adapter design.**
   - EleutherAI's output is a flagged-document list (format per the current README). Convert each row to the Chapter 03 schema with `detector = "n-gram"` and the per-item attribution reconstructed from the hit records.
   - Dolma's output is an attributes column embedded in the sharded output Parquet. Expand the column and convert rows the same way.

3. **Common-manifest check.** Both recipes must consume the *same* `eval_manifest.v1.yaml`. If a recipe only accepts its own config format, translate the manifest once (write `manifest_to_lm_eval.py` and `manifest_to_dolma.py`) and SHA each translated config.

4. **Three-way comparison table.** Build a pandas DataFrame indexed by `(benchmark, training_doc_id)` with boolean columns `flagged_in_house`, `flagged_lm_eval`, `flagged_dolma`. Group and count. Keep the per-row Parquet artifact so later exercises can re-use the comparison.

5. **Round-trip test.** Take 20 random items from MMLU test, GSM8K test, HumanEval. Render each as a plain text document in the format the LM Eval Harness emits during few-shot prompting. For each rendered document, assert that `flagged_lm_eval` is `True` and `flagged_dolma` is `True`. Log any miss with the rendered text, the normalisation SHA, and a short diagnosis.

6. **Write `recipe_comparison.md`.** Three paragraphs:
   - Where the recipes agree (expected overlap).
   - Where they disagree (list the per-benchmark deltas and spot-check one example in each).
   - The pick for your pipeline, with the reasoning. Chapter 06 §"picking a recipe" is the decision framework.

## Deliverables

```
exercise-02/
  code/
    manifest_to_lm_eval.py
    manifest_to_dolma.py
    run_lm_eval_decontam.sh
    run_dolma_decontam.sh
    adapt_lm_eval_output.py
    adapt_dolma_output.py
    three_way_compare.py
    roundtrip_test.py
    emit_log_update.py
  configs/
    lm_eval_decontam.yaml
    dolma_decontam.toml
  results/
    lm_eval_removal_manifest.parquet
    dolma_removal_manifest.parquet
    comparison.parquet              # three-way rows
    comparison_table.md             # human-readable per-benchmark table
    roundtrip_results.json          # per-eval-item pass/fail
    plots/
      per_benchmark_overlap.png
    decontamination_log.yaml        # extended with recipe_choice
    recipe_comparison.md
  README.md
```

## Acceptance criteria

- Both recipes consume the *same* `eval_manifest.v1.yaml` SHA and produce outputs normalised to the Chapter 03 `removal_manifest.parquet` schema.
- The per-benchmark comparison table lists, for each benchmark, four counts: in-house only, LM Eval only, Dolma only, union. Zero-count cells are allowed but must be shown.
- The round-trip test passes **20 / 20** on each recipe. Any miss is documented in `roundtrip_results.json` with the normalisation diagnosis and a path forward.
- `recipe_comparison.md` picks a recipe (or a per-surface hybrid) and defends the pick against the two Chapter 06 §"failure modes" patterns (recipe-disagreement-as-noise, manifest-drift).
- `decontamination_log.yaml` §1 `toolchain.recipe_choice` and `toolchain.engine_git_sha` are populated. If hybrid, both engine SHAs are recorded.
- Both toolkit git SHAs are pinned in the log. "latest" or "main" is a fail.
- Any factual claim in `recipe_comparison.md` about recipe behaviour that could not be verified against the current README is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Add a third benchmark** to the comparison — pick one where the two recipes most strongly disagree (BBH is a common candidate because of short-prompt tasks). Explain the delta.
- **Scoring-side alignment.** Run the LM Eval Harness on a tiny model (any HF model available locally) against MMLU test. Confirm the headline score matches the rendering convention your round-trip test used. If they drift, fix the round-trip test, not the harness.
- **HuggingFace `datatrove` as a third recipe.** `datatrove` ships a MinHash + LSH operator and a decontamination filter. Add it to the comparison. Chapter 06 does not treat it; your work would extend the chapter.
- **Dolma log inspection.** Open the Dolma release's published decontamination log and compare its schema to this chapter's §4. List the differences; adopt any missing fields.

## What this exercise is not

- Not the complete firewall. Only the n-gram detector runs here — exact-prompt and LLM-substring are Exercise 04.
- Not an in-depth benchmark study. The comparison is on a small slice; absolute removal rates are not meaningful at this scale, only the *recipe behavioural differences*.
- Not an endorsement of either recipe. Both work; this exercise is about picking well and documenting the choice.
