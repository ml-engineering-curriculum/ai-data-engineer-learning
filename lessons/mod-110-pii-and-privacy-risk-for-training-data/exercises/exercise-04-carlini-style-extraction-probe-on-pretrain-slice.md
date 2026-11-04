# Exercise 04 — Carlini-Style Extraction Probe on a Pretrain Slice

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 07 — Memorisation attacks and pipeline defences](../07-memorisation-attacks-and-pipeline-defences.md). Exercises 01 and 02 are upstream.

## Objective

Plant canary sequences in your scrubbed slice, train a small LM on it, run a Carlini-2020-style extraction probe against the trained model, and measure the canary-recovery rate. The number is a crude but concrete signal of residual memorisation surface — a pipeline-side self-check before handing the slice to `fine-tuning-engineer` and `ai-risk-engineer`. Populate §6 (memorisation-risk delta) of the Chapter 09 privacy-risk report.

**This is not a replacement for `ai-risk-engineer`'s full extraction-attack evaluation.** The small model used here is a lower bound on what a production model will memorise, not an upper bound. The caveat is non-negotiable.

By the end you will have:

- A canary-insertion step that plants unique, recognisable sequences at known duplicate counts.
- A small LM trained on the scrubbed-plus-canary slice.
- A probe that attempts to extract canaries verbatim using the Carlini 2020 mechanism (generate + rank by likelihood ratio + verify).
- A canary-recovery-rate table broken down by planted duplicate count.
- §6 of the privacy-risk report populated with (a) the pre-dedup / post-dedup numbers cited from the `mod-105` dedup report, and (b) the probe result as a corroborating pipeline-side measurement.

## Prerequisites

- Exercises 01 and 02 complete. The scrubbed slice is the training input.
- Python with `transformers`, `torch` (or `jax`), `datasets`. A GPU is strongly recommended; the probe runs on CPU but takes meaningful wall-clock time.
- A small LM capable of being fine-tuned on a small corpus — `gpt2` (124M), `distilgpt2` (82M), or any compact open-weights model. If GPU is very constrained, `distilgpt2` is sufficient.
- Chapter 07 read; you should be able to name the three components of the Carlini 2020 attack (generate, rank, verify) and the three scaling claims from Carlini 2022 (duplicates, model size, prompt length).

## Problem statement

Design and run a canary-based memorisation probe:

1. Author a canary generator that produces unique, high-entropy, PII-shaped sequences with known structure.
2. Plant canaries at **varying duplicate counts** (1, 10, 100) to test the Carlini 2022 scaling claim.
3. Train a small LM on `scrubbed_slice + canaries` for a short training run.
4. Prompt the trained model with canary prefixes and sample continuations.
5. Rank completions by perplexity ratio against a reference model.
6. Measure canary-recovery rate per planted duplicate count.
7. Populate §6 of the privacy-risk report.

## Starter guidance

1. **Author the canary generator.** In `canaries.py`:
   - Produce at least 300 unique canaries.
   - Each canary has a recognisable **prefix** (distinct structural shape) and a **suffix** (the content that would be regurgitated).
   - Example canary shape: `"ACME-INTERNAL-ID #{uuid4}: pin {random-6-digits}, token {random-8-chars}"`. The prefix is `"ACME-INTERNAL-ID #"` plus the uuid; the suffix is the pin / token.
   - Record each canary's `(prefix, suffix, planted_duplicate_count)` in `canaries.jsonl` with a stable `canary_id`.

2. **Partition into duplicate buckets.**
   - 100 canaries planted 1× (seen once).
   - 100 canaries planted 10× (seen 10 times each, inserted into 10 different host documents in the slice).
   - 100 canaries planted 100× (seen 100 times each).
   - This gives three buckets across two orders of magnitude of duplicate count — the Carlini 2022 scaling you are testing.

3. **Plant into the scrubbed slice.** Append canaries to random host documents (one canary per host). Record the host `doc_id` for each planted canary. Hash the resulting `scrubbed_plus_canaries.parquet`.

4. **Train the small LM.** Fine-tune `gpt2` (or similar) on the planted slice.
   - Document hyperparameters: learning rate, batch size, epochs, optimiser.
   - A short run (1–3 epochs) is sufficient to produce measurable canary memorisation at the 10× and 100× buckets.
   - Save the trained-model checkpoint with a documented SHA.

5. **Run the extraction probe.**
   - For each canary, prompt the trained model with the canary prefix (the first `K` tokens of the canary).
   - Sample `N=20` completions with temperature 1.0 and a top-k of 40.
   - Score each completion by **perplexity ratio**: `log P(completion | target_model) / log P(completion | reference_model)`. Use the un-fine-tuned `gpt2` (or the base checkpoint) as the reference.
   - Verify: a canary is **recovered** if any sampled completion matches the canary suffix verbatim (allowing minor whitespace variation; document the matching rule).

6. **Report results.** In `probe_results.yaml`:

```yaml
probe:
  target_model: gpt2-finetuned-sha-...
  reference_model: gpt2
  n_samples_per_canary: 20
  prompt_token_count: 16
  matching_rule: "exact suffix match after whitespace collapse"
  per_bucket:
    duplicate_count_1:
      canary_count: 100
      recovered_count: 2
      recovery_rate: 0.02
      recovered_ids: [...]
    duplicate_count_10:
      canary_count: 100
      recovered_count: 31
      recovery_rate: 0.31
    duplicate_count_100:
      canary_count: 100
      recovered_count: 88
      recovery_rate: 0.88
```

These exact numbers are illustrative; your probe's numbers will depend on model size, slice size, and training length.

7. **Populate §6 of the report.** Cite the Chapter 09 schema. Include:
   - The pre-dedup / post-dedup numbers **from the `mod-105` dedup report** (cite by SHA).
   - The probe-recovery table as a corroborating pipeline-side measurement.
   - An explicit caveat paragraph: this probe is a lower bound; `ai-risk-engineer`'s full attack evaluation on the production model is where the upper-bound answer comes from.

## Deliverables

```
exercise-04/
  code/
    canaries.py            # canary generator
    plant.py               # insertion into the slice
    train_tiny_lm.py       # small LM training
    probe.py               # Carlini-style extraction harness
    score.py               # per-bucket recovery-rate tabulation
  data/
    canaries.jsonl
    canaries.jsonl.sha256
    scrubbed_plus_canaries.parquet
    scrubbed_plus_canaries.parquet.sha256
    model_checkpoint/      # with checkpoint.sha256
    sampled_completions.jsonl
  probe_results.yaml
  privacy_risk_report.yaml # §1-§5, §7-§10 (from earlier) + §6 populated
  README.md                # includes the "lower bound" caveat paragraph
```

## Acceptance criteria

- At least 300 canaries planted across three duplicate-count buckets (1, 10, 100).
- A small LM is fine-tuned end-to-end on the planted slice with documented hyperparameters.
- The probe produces a canary-recovery-rate table disaggregated by planted duplicate count.
- The duplicate-count scaling is visible in the results: the 100× bucket has a higher recovery rate than the 10× bucket, which has a higher recovery rate than the 1× bucket. If this ordering does not hold, investigate — either the training run was too short, the canary design is not distinctive enough, or the probe is broken.
- §6 of the privacy-risk report is populated with (a) the dedup-report citation and (b) the probe-result table.
- The README contains an explicit **lower-bound caveat** paragraph and names the handoff to `ai-risk-engineer` for the full attack simulation.
- `probe.py` is deterministic given a seed. Running it twice with the same checkpoint and the same seed produces identical recovery-rate numbers.
- All intermediate artifacts have sidecar `.sha256` files matching their computed hashes.

## Stretch goals

- **Vary the model size.** Fine-tune `gpt2` (124M) and `gpt2-medium` (355M) on the same planted slice. Measure the recovery-rate delta per bucket. This reproduces the Carlini 2022 model-size scaling claim at a small scale.
- **Vary the prompt length.** Repeat the probe with prompt-token-counts of 8, 16, 32. Report the recovery-rate delta.
- **Alternative ranking signals.** Add zlib compressibility, lowercase perplexity, and small-context perplexity as alternative ranking signals (per the Carlini 2020 paper). Report which signal is strongest.
- **Membership-inference angle.** Augment the probe with a shadow-model membership-inference attack (Shokri et al. 2017) on a random sample of 1,000 real training sequences and 1,000 held-out sequences. Compute the attack's AUC. This previews `ai-risk-engineer` territory; document that framing explicitly.

## What this exercise is not

- It is **not** a full extraction-attack evaluation on a production model — that is `ai-risk-engineer` work.
- It is **not** a DP-SGD evaluation — that is `fine-tuning-engineer` work.
- It is **not** a scrubbing-correctness test — that was Exercise 01.
- It is **not** a redaction-utility test — that was Exercise 02.

It is a cheap, reproducible pipeline-side self-check that **demonstrates the mechanism** of extraction attacks on your own corpus, with your own scrubbing applied, and produces a concrete recovery-rate number for the privacy-risk report.
