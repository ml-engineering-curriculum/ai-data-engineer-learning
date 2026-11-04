# Exercise 04 — The Synthetic-Data Contamination Firewall Contract

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 05 — Where the firewall sits](../05-firewall-placement-across-pipeline-surfaces.md), [Chapter 04 — Exact-prompt and LLM-substring detectors](../04-exact-prompt-and-llm-substring-detectors.md)

## Objective

Author the `synthetic_generator_contract.v1.yaml` that [`mod-108`](../../mod-108-synthetic-data-generation-at-production-scale/README.md)'s generation-side owners must sign before any synthetic data enters training. Run the Chapter 05 §"synthetic-generation vector" firewall against a simulated synthetic-generator output using both the exact-prompt matcher (Chapter 04) and the LLM-substring detector. Produce a spot-check report that would trigger (or not trigger) the contract's escalation threshold.

By the end you will have:

- A `synthetic_generator_contract.v1.yaml` matching the Chapter 05 schema, pinned to a hypothetical generator with a stated certification case (A: certified / B: untrusted).
- A corpus-wide LLM-substring detector run against ~1,000 simulated generator outputs, with per-benchmark flagged-fraction metrics.
- An escalation decision (halt / proceed) grounded in the contract's thresholds.
- `decontamination_log.yaml` §4 `synthetic` populated and §1 `toolchain` extended to reference the contract SHA.

## Prerequisites

- Python 3.11+ with `anthropic` (for the LLM substring detector), `pyarrow`, `pandas`, `pyyaml`, `xxhash`.
- An Anthropic API key (or a local model that can role-play the detector prompt; the exercise is written for Claude Haiku 4.5 but any small instruct-tuned model works at lower quality).
- A simulated generator-output file: ~1,000 synthetic "SFT examples" in the shape your lab actually produces. If you do not have real synthetic data, construct one by sampling from a language model with the Chapter 05 §"the recursive-synthetic-contamination vector" prompt pattern ("generate a hard math problem", "generate a Python function with a docstring"). **Seed the file with 10 verbatim benchmark items and 10 lightly-paraphrased benchmark items** so the firewall has a known ground truth to detect.
- `eval_manifest.v1.yaml` from Exercise 01.
- Read Chapter 05 and Chapter 04.

## Problem statement

Author the contract, run the detectors against a simulated generator output, and decide escalation.

1. **Author the contract.** Fill in `synthetic_generator_contract.v1.yaml`:
   - Pick a case: A (certified-decontaminated) or B (untrusted). Document the reason for the choice.
   - List the benchmarks the generator is certified against (case A) or untrusted against (case B).
   - Set `spot_check_rate` and `escalation_threshold.flagged_fraction_above` explicitly. Defend both numbers.
   - Hash the file; record the SHA.

2. **Prepare the simulated generator output.**
   - If you have real synthetic-data outputs, use them.
   - Otherwise, produce ~1,000 "SFT items" with a small LLM given prompts like: "Generate a hard high-school math problem with a worked solution." or "Write a Python function that solves a common algorithmic task; include a docstring."
   - **Seed 10 verbatim MMLU / GSM8K items** (dropped in unchanged).
   - **Seed 10 lightly-paraphrased items** (one word changed per sentence).
   - **Seed 980 clean synthetic items.**
   - Save the ground-truth labels so you can measure firewall precision / recall.

3. **Run the exact-prompt matcher** (Chapter 04 §"exact-prompt matcher") against the synthetic output using the eval manifest. Expect it to flag the 10 verbatim items. Record precision and recall on the ground truth.

4. **Run the LLM-substring detector** (Chapter 04 §"LLM-based substring detector") against the synthetic output. For case A (certified), run **candidate-only** prefiltering; for case B (untrusted), run **corpus-wide**. Expect it to catch the 10 paraphrased items (and reconfirm the 10 verbatim items). Record precision and recall.

5. **Compute the flagged fraction.** `flagged_fraction = (flagged items) / (total items)`. Compare to the contract's `escalation_threshold.flagged_fraction_above`.

6. **Decide escalation.** If above threshold → halt, do not fold the synthetic data in, open an investigation. If below threshold → proceed, but record the flagged items in the removal manifest so they do not enter training.

7. **Populate the log.**
   - `decontamination_log.yaml` §1 `toolchain` adds the contract path and SHA.
   - `decontamination_log.yaml` §4 `synthetic` records: contract SHA, case, spot-check rate (if case A), flagged-fraction, escalation verdict.

## Starter guidance

1. **Contract authoring.** Chapter 05 §"the firewall contract with the generation-side owner" is the schema. Start from the example YAML in the chapter and specialise. The key editorial choices:
   - `provenance`: is the generator trained in-house, an open-weights model, or an API model? Each has different trust implications.
   - `benchmarks_decontaminated` vs `benchmarks_untrusted`: the split is per-benchmark, not generator-wide. LiveBench and LiveCodeBench, for example, are almost always `untrusted` for any pre-2026 generator.
   - `benchmark_style_prompting`: if the prompts to the generator resemble eval items ("generate a hard math problem"), set this to `true` and promote the LLM-substring detector to `corpus_wide` scope.

2. **Seeded synthetic output.**
   - To get 10 verbatim items: sample 10 from MMLU / GSM8K and inject the full prompt + canonical answer as a synthetic "SFT item" row.
   - To get 10 paraphrased: for each sampled item, change one word per sentence while preserving meaning. Document the paraphrase method.
   - To get 980 clean synthetic: prompt a small model with generation templates, collect outputs, SHA them to confirm they are unique.

3. **Exact-prompt matcher.** Reuse the hash set from Chapter 04's exact-prompt pseudo-code. Normalise the synthetic item's prompt field; hash; look up. Record hits.

4. **LLM-substring detector.** Use Claude Haiku 4.5 with the Chapter 04 system prompt. For each `(synthetic_item, candidate_eval_item)` pair selected by prefiltering (case A) or exhaustively (case B), call the model. Record the three-way verdict. For ground truth, you know which 20 items are leaks; measure precision and recall.

5. **Prompt caching.** The system prompt is identical across ~1,000 calls. Enable prompt caching on the system block — see the Anthropic prompt-caching docs. Compare cost with and without.

6. **Escalation decision.** Honour the threshold. If the contract says `flagged_fraction_above: 0.001` and your measured fraction is 0.004, the decision is halt. Write the decision into the log *without* retuning the threshold.

## Deliverables

```
exercise-04/
  code/
    author_contract.py
    build_synthetic_output.py       # produces the 1,000-item file with ground truth
    exact_prompt_scan.py
    llm_substring_scan.py
    decide_escalation.py
    emit_log_update.py
  configs/
    synthetic_generator_contract.v1.yaml
    llm_detector_system_prompt.txt
    llm_detector_system_prompt.sha256
  data/
    synthetic_output.jsonl           # 1,000 items
    ground_truth_labels.jsonl        # per-item: clean | verbatim_leak | paraphrased_leak
  results/
    exact_prompt_hits.parquet
    llm_substring_hits.parquet
    precision_recall.json
    escalation_decision.yaml
    decontamination_log.yaml         # extended with §1 contract SHA and §4 synthetic
  README.md
```

## Acceptance criteria

- `synthetic_generator_contract.v1.yaml` is complete per the Chapter 05 schema: generator identity, case A/B, benchmark list, prompting profile, firewall plan, spot-check rate, escalation threshold, signature. The contract SHA is recorded in the log.
- The synthetic output file has **≥ 1,000 items** with a ground-truth label per item (clean / verbatim / paraphrased).
- The **exact-prompt matcher catches ≥ 9 of 10** verbatim items. If it catches fewer, the normalisation is drifting — diagnose and fix, do not lower the threshold.
- The **LLM-substring detector catches ≥ 7 of 10** paraphrased items (Chapter 04 does not promise 10/10 on light paraphrase; 7 is a defensible floor at Haiku scale, and you may need to raise or lower based on the paraphrase aggressiveness).
- `precision_recall.json` records both detectors' per-class precision and recall.
- The escalation decision follows the contract *verbatim*. If the flagged fraction is above threshold, the escalation verdict is `halt`; post-hoc threshold adjustment fails the criterion.
- `decontamination_log.yaml` §4 `synthetic` block is fully populated.
- The LLM detector system prompt is SHA'd and the SHA referenced in the log. Any change to the prompt is a new detector version.

## Stretch goals

- **Double-code the LLM detector.** Run a 1% sample against a secondary model (Claude Sonnet 4.6, or a different system prompt). Compute Cohen's κ. Record it in the log's §3 `llm_substring.double_coding` block. Chapter 04 §"agreement and double-coding" is the reference.
- **Ablate prompt-caching cost.** Run the detector with and without prompt caching. Report the per-item cost delta and the end-to-end cost delta over 1,000 items. Put the numbers in the log's `notes` section.
- **Aggressive paraphrase.** Re-produce the 10 paraphrased items with a stronger paraphrase (sentence re-ordering, synonym substitution across the whole item). Re-run the LLM detector. Report the recall drop.
- **Case-A spot-check tuning.** Vary the spot-check rate from 0.5% to 5%. For each rate, compute the probability of catching at least one leak given a 0.1% true leak rate. Pick a rate that catches ≥ 99% of leaks of that size. Record the calculation in the contract.

## What this exercise is not

- Not the generation side of the synthetic data pipeline. That work lives in [`mod-108`](../../mod-108-synthetic-data-generation-at-production-scale/README.md).
- Not a production detector. The 1,000-item scale is for the exercise; a real synthetic-data firewall runs against millions of items with the same contract and the same detectors.
- Not a substitute for the full-firewall log. §4 `synthetic` is one row; Exercise 05 integrates it with the rest.
